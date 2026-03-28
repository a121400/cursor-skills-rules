---
name: android-anti-detect-bypass
description: 安卓反检测绕过技术。Frida hook 顺序/child-gating/syscall 拦截、SO 最小补丁(Houdini)、MuMu 应用级属性伪装、百度加固bypass、模拟器/root/反调试检测绕过、attach 时间窗口速调技巧。当 APP 检测到模拟器/root/Frida/调试器而崩溃或闪退时使用。
---

# 安卓反检测绕过

执行时遵循 **reverse-auto-execute**：实际执行 adb/frida 等命令、反复测试、诊断后重试或与用户交互，不轻易只给方案或暂停。

## 检测分层与对策

| 类型 | 常见实现 | 绕过思路 |
|------|----------|----------|
| 反调试 | Java Debug.isDebuggerConnected、Native ptrace/TracerPid | Java hook 返回 false；Frida hook ptrace/fgets 过滤 TracerPid |
| 崩溃自杀 | SO 内 strb 写 0xDEADCAB1 触发 SIGSEGV | SO 仅补丁该指令（nop 或 ldp+ret），勿整函数改 ret |
| 进程终止 | exit/abort/kill/tgkill/raise + 直接 syscall | Frida replace 为 spinForever；拦截 syscall(SYS_kill/SYS_exit_group/SYS_exit) |
| 模拟器 | /proc/cpuinfo、ELF、Build 属性 | cpuinfo 伪造 ARM64；__system_property_get 全面伪装；MuMu 应用级属性配置 |
| Root/Hook | access/fopen su 路径、maps 扫 frida、strstr 关键字、connect 27042 | Frida 拦截返回 -1/NULL；read/fgets 过滤；strstr 返回 NULL；connect 返回 -1 |
| 多进程 | 独立进程(:cache)跑检测 | Frida child-gating，子进程同样注入 bypass 脚本 |

## Frida 反检测脚本模板

### 核心 bypass（libc 层）

```javascript
'use strict';
function spinForever() { while(true) Thread.sleep(0x7FFFFFFF); }

// 1. 进程终止拦截
["exit", "_exit", "abort"].forEach(name => {
    var addr = Module.findExportByName("libc.so", name);
    if (addr) Interceptor.replace(addr, new NativeCallback(() => {
        console.log("[BYPASS] " + name + "() blocked"); spinForever();
    }, 'void', []));
});
["kill", "tgkill"].forEach(name => {
    var addr = Module.findExportByName("libc.so", name);
    if (addr) Interceptor.replace(addr, new NativeCallback(function() {
        return 0;
    }, 'int', name === "kill" ? ['int','int'] : ['int','int','int']));
});

// 2. strstr 隐藏关键字
var strstr = Module.findExportByName("libc.so", "strstr");
if (strstr) Interceptor.attach(strstr, {
    onEnter(args) {
        if (!args[1].isNull()) {
            var n = args[1].readUtf8String();
            if (n && /frida|xposed|magisk|substrate/.test(n)) this.block = true;
        }
    },
    onLeave(ret) { if (this.block) ret.replace(ptr(0)); }
});

// 3. /proc/net/tcp 屏蔽（Frida 端口扫描）
var openat = Module.findExportByName("libc.so", "openat");
if (openat) Interceptor.attach(openat, {
    onEnter(args) {
        var p = args[1].readUtf8String();
        if (p && p.indexOf("/proc/net/tcp") !== -1) this.block = true;
    },
    onLeave(ret) { if (this.block) ret.replace(ptr(-1)); }
});

// 4. connect 端口拦截
var connect = Module.findExportByName("libc.so", "connect");
if (connect) Interceptor.attach(connect, {
    onEnter(args) {
        if (args[1].readU16() === 2) {
            var port = (args[1].add(2).readU8() << 8) | args[1].add(3).readU8();
            if (port === 27042 || port === 27043) this.block = true;
        }
    },
    onLeave(ret) { if (this.block) ret.replace(ptr(-1)); }
});

// 5. syscall 直接调用拦截 (ARM64)
var syscall = Module.findExportByName("libc.so", "syscall");
if (syscall) Interceptor.attach(syscall, {
    onEnter(args) {
        var nr = args[0].toInt32();
        // __NR_kill=129, __NR_exit=93, __NR_exit_group=94, __NR_tgkill=131
        if ([93, 94, 129, 131].includes(nr)) { this.block = true; }
    },
    onLeave(ret) { if (this.block) ret.replace(ptr(0)); }
});
```

### Hook 安装顺序

exit/_exit/abort → kill/tgkill/raise → syscall → __cxa_finalize → __system_property_get → open/read/fgets → access/fopen → strstr → connect

## Frida attach 时间窗口技巧

APP 有延迟反检测（启动后 2~5 秒才开始扫描）时的最佳实践：

### 模式选择

| 模式 | 适用场景 | 注意事项 |
|------|----------|----------|
| **attach（推荐）** | 检测延迟 ≥2s；需调用已初始化的 Java API | 抢在检测触发前完成 RPC |
| spawn + bypass | 检测在 JNI_OnLoad 或更早 | 需完整 bypass 脚本 |
| spawn + 脚本延迟 | ART 未就绪，Java.perform 报错 | `setTimeout(fn, 3000~5000)` |

### attach 快速 RPC 模式（鲸探实战验证）

```python
session = device.attach(pid)
script = session.create_script(JS_CODE)
script.load()
result = script.exports_sync.get_all(args)  # 单次调用获取所有参数
session.detach()
```

关键点：
1. **合并为单次 RPC**：把多个 Java 调用合并到一个 `getAll` 函数，减少 attach 时间
2. **attach 后立即调用**：不 sleep，直接调 RPC
3. **调用完立即 detach**：减少被检测窗口
4. **失败自动重启 APP**：`am force-stop` → `am start` → sleep 8s → 重新 attach

### frida-server 权限（APatch/Magisk）

```bash
# APatch 设备
adb shell "su 0 /data/local/tmp/frida-server -D"

# Magisk 设备
adb shell "su -c /data/local/tmp/frida-server -D"

# 验证
frida -U -l test.js com.target.app
```

> `need Gadget to attach on jailed Android` = frida-server 未以 root 运行。

## SO 补丁原则（Houdini 环境）

- 只补丁导致崩溃的**单条/两条指令**，保留函数前后文
- **禁止**整函数入口改成 `ret`：Houdini 转译缓存错乱
- 部署：`adb push` 到 `/data/app/<path>/lib/arm64/` 并 chmod

## MuMu 应用级伪装

Java 层 Build 在 Zygote fork 时已缓存，resetprop 无效。用 MuMu 自带「应用级设备属性」：
`/data/system/etc/mumu-configs/app-device-channel-prop.config` 绑定包名到 config。

## 多进程覆盖

```javascript
session.enable_child_gating();
session.on('child-added', child => {
    // attach 子进程，注入同一 bypass 脚本
});
```

## 常见坑

- 直接 syscall 绕过 libc：需 hook `syscall()` 并识别系统调用号
- `__cxa_finalize` 需 replace 为 no-op，否则 atexit 检测线程退出崩溃
- Frida `spawn` 时 `unexpected crash while trying to allocate memory` = 反篡改检测，需加 bypass 脚本
- `ProcessNotFoundError` / `script has been destroyed` = APP 延迟检测杀进程，缩短 attach 时间或加 bypass
- **不要**使用 `Process.setExceptionHandler`，与 Houdini 信号处理冲突
- Java hook 需延迟（setTimeout 3~5s）再 `Java.perform`，避免 ART 未就绪
