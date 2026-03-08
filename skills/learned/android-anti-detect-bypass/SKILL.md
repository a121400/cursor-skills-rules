---
name: android-anti-detect-bypass
description: 安卓加固/模拟器/root/Frida 检测绕过。百度加固(Sagittarius)、多进程检测、SO 最小补丁、Frida hook 顺序、child-gating、syscall 绕过、MuMu 属性伪装。适用于 i茅台等带百度加固/网易安全 SDK 的 APP 在 MuMu 上运行。
---

# 安卓反检测绕过（提炼自 i茅台等实战）

## 检测分层与对策

| 类型 | 常见实现 | 绕过思路 |
|------|----------|----------|
| 反调试 | Java Debug.isDebuggerConnected、Native ptrace/TracerPid | Java hook 返回 false；Frida hook ptrace/fgets 过滤 TracerPid |
| 崩溃自杀 | SO 内 strb 写 0xDEADCAB1 触发 SIGSEGV | SO 仅补丁该指令（nop 或 ldp+ret），勿整函数改 ret |
| 进程终止 | exit/abort/kill/tgkill/raise + 直接 syscall | Frida replace 为 spinForever；拦截 syscall(SYS_kill/SYS_exit_group/SYS_exit) |
| 模拟器 | /proc/cpuinfo、ELF、Build 属性 | cpuinfo 伪造 ARM64；__system_property_get 全面伪装；MuMu 应用级属性配置 |
| Root/Hook | access/fopen su 路径、maps 扫 frida、strstr 关键字、connect 27042 | Frida 拦截返回 -1/NULL；read/fgets 过滤；strstr 返回 NULL；connect 返回 -1 |
| 多进程 | 独立进程(:cache)跑检测 | Frida child-gating，子进程同样注入 bypass 脚本 |

## SO 补丁原则（Houdini 环境）

- 只补丁导致崩溃的**单条/两条指令**（如 strb + bl → ldp+ret，或 strb→nop），保留函数前后文。
- **禁止**整函数入口改成 `ret`：会导致 Houdini 转译缓存错乱、Scudo 堆损坏、abort 卡死。
- 部署：adb push 补丁后 so 到 `/data/app/<path>/lib/arm64/` 并 chmod。

## Frida 运行时要点

- **不要**使用 `Process.setExceptionHandler`，与 Houdini 信号处理冲突，易造成无限 access-violation。
- Hook 安装顺序建议：exit/_exit/abort → kill/tgkill/raise → syscall → __cxa_finalize → __system_property_get → open/read/fgets（cpuinfo、maps、tcp）→ access/fopen → strstr → connect。
- 属性伪装需覆盖：ro.product.cpu.abi/abilist、ro.dalvik.vm.native.bridge=0、ro.enable.native.bridge.exec=0、设备/SoC/Build 系列；清除 ro.kernel.qemu；cpuinfo 返回 ARM64 特征。
- Java hook 需**延迟**（如 setTimeout 5s）再 `Java.perform`，避免 ART 未就绪报错。

## MuMu 应用级伪装

- Java 层 Build 在 Zygote fork 时已缓存，**resetprop 无效**。用 MuMu 自带「应用级设备属性」：在 `/data/system/etc/mumu-configs/app-device-channel-prop.config` 绑定包名到 config 文件，在 device-prop-configs 下写完整 ro.product.*/ro.build.*（如三星 S22 配置）。

## 多进程覆盖

- 检测若在 `:cache` 等子进程：`session.enable_child_gating()`，在 `child-added` 里 attach 子进程、注入同一 bypass 脚本后 resume。

## 一键启动流程

设备发现 → 启动 frida-server → adb forward tcp:27042 tcp:27042 → 部署补丁 so → spawn 目标包名 → attach 并 load 脚本 → 短暂 sleep → resume。

## 常见坑

- 直接 syscall 绕过 libc：需 hook syscall() 并识别 SYS_kill/SYS_exit_group/SYS_exit 等编号。
- __cxa_finalize 需 replace 为 no-op，否则 atexit 里检测线程退出时会崩。
- Magisk 模块在 post-fs-data 里改 ro.product.cpu.abi 可能让 x86 模拟器无法启动，恢复需 VMDK 级处理。
