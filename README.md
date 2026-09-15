vivo S18 (SM7550) 内核提权研究：防御矩阵与攻击面分析
> ⚠️ **本仓库数据存在多处错误**（KASLR slide / physmap 基址 / task_struct 偏移）。
> 请以最新归档为准：**[vivo-S18-SM7550-Kernel-5.15.178-Research-Archive](https://github.com/249707737/vivo-S18-SM7550-Kernel-5.15.178-Research-Archive)**

概述

本项目是对 vivo S18（型号 PD2323，高通 SM7550，Android 16，内核 5.15.178） 的深度黑盒安全研究。研究目标是在 未解锁 Bootloader 的前提下，尝试获取 Root 权限或至少关闭 SELinux。经过大量实验，我们成功绘制了该设备的完整防御矩阵，并验证了多项已知提权路径在物理层和硬件层的失效原因。本报告包含设备信息、攻击尝试记录、防御机制分析以及最终结论，旨在为后续安全研究者提供参考。

免责声明：本研究仅用于安全研究和学术交流。所有测试均在自有设备上进行，未对任何第三方造成影响。请勿将本文内容用于非法目的。

---

设备信息

项目 值
设备型号 vivo S18 (PD2323)
SoC 高通 SM7550 (Snapdragon 7 Gen 3)
内核版本 5.15.178-g0f1e91e908f4-dirty
构建版本 PD2323_A_16.2.9.0.W10
安全补丁级别 2026-05-01
Bootloader 锁定（ro.boot.verifiedbootstate=green）
可用权限 ADB shell（uid=2000）

---

研究目标

· 在未解锁 BL 的设备上实现本地提权（uid=0）。
· 若无法提权，至少将 SELinux 置为 permissive。
· 记录并分析所有遇到的防御机制，为社区提供“负向防御图谱”。

---

尝试的攻击路径与结果

1. GPU 漏洞 (CVE-2025-21479)

· 原理：利用高通 Adreno A7xx 系列 GPU 固件中的 CP_SET_DRAW_STATE 权限校验缺陷，通过 SMMU 绕过获得任意物理内存读写。
· 结果：成功触发漏洞（Cheese PoC 输出 0 0），但后续利用因 高通 SMMU 硬件拦截（Bad address）而失败。物理基址和 KASLR 偏移无法正确匹配。

2. Binder / Futex 传统 UAF

· 原理：利用 Binder（CVE-2023-20938）或 futex（CVE-2026-43499）的内核 UAF 构造任意读写。
· 结果：
  · Binder 因 RANDOM_KMALLOC_CACHES 和专用缓存隔离而失败，堆喷无法跨池。
  · futex 因 SLAB_FREELIST_HARDENED 和 page_poison=on 导致无法控制内存布局。

3. GhostLock (CVE-2026-64560)

· 原理：利用 posix-cpu-timers 的 UAF，配合 timer_create 制造竞态。
· 结果：因目标对象 k_itimer 锁死于 posix_timers_cache（264B），且 add_key 堆喷被 seccomp 拦截，无法实现跨缓存攻击。

4. IonStack (CVE-2026-43499)

· 原理：利用 pselect 竞态泄露 KASLR，并通过 pipe_phys 原语获得任意读写。
· 结果：
  · 成功编译运行，但 slide 阶段因 KernelSnitch 侧信道不可靠 导致设备在 10 秒内硬重启。
  · 手动填写偏移时发现 task_struct 等结构体偏移差异巨大（实际为 0x798 而非 0x820），进一步确认了崩溃原因。
  · 确认内核配置 CONFIG_KASAN_HW_TAGS=y 和 CONFIG_PANIC_ON_OOPS=y 后，判断该路径在当前设备上不可行。

---

防御机制分析（实测确认）

物理层（初始化写死）

· page_poison=on：释放内存后填充 0xaa，阻止 UAF 重写。
· CONFIG_USER_NS 未设置：禁用用户命名空间，堵塞提权入口。
· buildvariant=user：普通用户无 CAP_SYS_ADMIN。
· kpti=0：关闭 KPTI（对攻击者有利，但被其他防护覆盖）。

分配器层

· RANDOM_KMALLOC_CACHES：将相同大小的缓存拆分，公共堆喷无法跨入专用池。
· SLAB_FREELIST_HARDENED：freelist 指针加密，无法精准重分配。
· 专用缓存（如 posix_timers_cache）隔离目标对象。

用户态

· Seccomp 沙箱：拦截了 add_key、io_uring 等高危系统调用。
· SELinux 强制：拒绝 /proc/kallsyms、perf_event_open、dmesg 等。

硬件层

· ARM PAC：函数指针签名校验，控制流劫持会触发 Panic。
· 高通 SMMU：拦截从 EL0 发起的物理页表修改。
· KASAN_HW_TAGS (MTE)：内核内存带硬件标签，导致侧信道泄露的地址失真，后续计算无法匹配。
· PANIC_ON_OOPS：任何内核 OOPS 都会触发硬重启，无试错空间。

---

关键数据（已提取）

符号偏移（基于 vmlinux 和 System.map）

符号 静态地址 用途
_stext 0xffffffc008010000 内核基址
init_task 0xffffffc00aec0cc0 提权目标
root_task_group 0xffffffc00afd8b40 任务组
selinux_state 0xffffffc00b10c710 SELinux 状态
commit_creds 0xffffffc0081ae63c 提权函数
init_cred 0xffffffc00ae79ca8 凭证

KASLR 偏移

· KASLR_SLIDE = 0x2080ae40（多次重启未变）

结构体偏移（通过 BTF 提取）

· TASK_CRED_OFF = 0x798
· TASK_REAL_CRED_OFF = 0x790
· WAITER_TREE_ENTRY_OFF = 0x00
· WAITER_PI_TREE_ENTRY_OFF = 0x18

---

结论

在未解锁 Bootloader 的 vivo S18 上，所有已知的公开提权/关 SELinux 路径均因以下核心原因失败：

1. 硬件级防御（MTE + PAC + SMMU） 使侧信道和物理内存读写不可用。
2. CONFIG_PANIC_ON_OOPS=y 让任何细小错误直接硬重启，彻底扼杀试错空间。
3. 分配器隔离与 Seccomp 堵死了堆喷和系统调用层面的攻击。

因此，当前设备在未解锁状态下不具备通过公开漏洞提权的现实可行性。我们建议：

· 若对 Root 有强烈需求，请申请 vivo 官方解锁 Bootloader（需清空数据，失去保修）。
· 解锁后可通过刷入 Magisk/KernelSU 或自定义内核（关闭 panic_on_oops）实现持久 Root。

---

致谢

本研究参考了大量社区开源项目，包括：

· zhuowei/cheese
· omae-wa-cheese-da
· CVE-2026-43499-Poc-Analysis
· ghostlock-selinux-disabler
· CVE-2026-43499-Neo11Plus

---

联系方式

如有疑问或合作意向，请通过 GitHub Issues 或邮件联系作者249707737z@gmail.com
