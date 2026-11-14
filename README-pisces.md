# pisces kernel (3.4.113 merged)

Xiaomi Mi 3 TD（pisces，NVIDIA Tegra 4 / T114）内核，供 LineageOS 16.0 使用。

## 分支

| 分支 | 说明 |
|---|---|
| `lineage-16.0-113` | **当前**：3.4.35 厂家树 + 上游 3.4.113 stable 合并 + Android 9 backport |
| `pisces-kk-oss` | 小米开源原始 3.4.35（基线） |

## 合并策略（3.4.35 → 3.4.113）

- **采用上游 3.4.113**：`fs/`、`mm/`、`drivers/usb/core|class`、`security/`、部分 `kernel/`、`sound/`
- **保留厂家实现**（Tegra 强耦合或 API 不兼容）：`net/`、`crypto/`、`sched/`、`time/`、
  `thermal/`、`regmap`、`workqueue`、`8250`、`usb/host`、`usb/storage`、`mmc core`、
  `i2c-tegra`、`ping/ip`
- 冲突处理：3-way merge（base=上游 3.4.35，ours=厂家树，theirs=3.4.113）

## Android 9 backport

| 项目 | 状态 | 说明 |
|---|---|---|
| `getrandom(2)` | ✅ | ARM `__NR 384`，阻塞实现 |
| `memfd_create(2)` | ✅ | ARM `__NR 385`，tmpfs 后端（无 sealing） |
| seccomp `SECCOMP_MODE_FILTER` | ✅ | 自带 classic BPF 解释器，用 `task_pt_regs()` 取寄存器，未改汇编入口 |
| SELinux policy 30 | ❌ | 3.4 内核只认到 26；运行期需跳过策略加载或 backport |
| `mmap_rnd_bits` | ⚠️ | 无该 sysctl，init 需跳过（TWRP 树已验证跳过方案） |

## 编译（独立）

```bash
export CROSS_COMPILE=arm-linux-androideabi-
make ARCH=arm tegra11_pisces_user_defconfig
make ARCH=arm zImage -j16
```

或由 Lineage 构建系统直接编译（`TARGET_KERNEL_SOURCE=kernel/xiaomi/pisces`，
defconfig `tegra11_pisces_user_defconfig`，工具链 `arm-linux-androideabi-4.9`）。

## KernelSU 与 GitHub Actions

- `KernelSU` 为 `backslashxx/KernelSU` 子模块，固定提交
  `41ddc6ee368f8dc7505e10980e78cee8ad1eb3f3`，与 aries 参考移植一致。
- 首次检出后执行 `git submodule update --init --recursive`；
  `drivers/kernelsu` 链接到 `../KernelSU/kernel`。更新子模块时应显式审查并提交新的 gitlink。
- pisces defconfig 内置 KernelSU，使用 ARM syscall table 和旧版 LSM 钩子，
  启用完整 UID 检查，不依赖 kprobes。
- 当前只生成 SELinux enforcing 版本；关闭 `SECURITY_SELINUX_DEVELOP`，
  保持 SELinux 启用并禁止启动参数及运行期关闭 SELinux。
- 手动触发 `.github/workflows/build.yml`，使用 Linaro GCC 5.1.1 和
  `tegra11_pisces_user_defconfig` 编译 `zImage`。
- CI 调用 `.github/scripts/repack_boot.py`，将新 `zImage` 替换进
  `.github/boot.img` 的 kernel 段，保留 ramdisk、cmdline、加载地址、页大小及其他段，
  重新计算 kernel 大小、对齐和 boot SHA-1。原始模板不被覆盖。
  下载产物 `pisces-enforcing-<run number>` 中的
  `boot-pisces-3.4.113-perf-kernelsu-<run number>-selinux-enforcing.img`。

本次移植仅经过静态检查；尚未验证 KernelSU 编译、CI 运行或真机启动。

## 已验证

- TWRP 3.7.0_9 真机启动，root adb、MTP、Format Data 正常
- seccomp：ALLOW 过滤器放行、KILL 过滤器触发 SIGSYS
- 连续多次启动无 Oops/panic

## 已知问题

- `CONFIG_VT` + `console=tty0` 会触发 `con_insert_unipair` Oops（调试内核不要开 VT）
- 无 64-bit binder（32 位用户空间）
- hw_random 不可用（可忽略）
