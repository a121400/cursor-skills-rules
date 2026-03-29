---
name: frida-android-rpc
description: Frida Android RPC 实战模式。attach/spawn 选择、Java RPC 导出模式、反检测时间窗口、APDID/APSign/APDeviceColor 等 APSE SDK 调用模式、Python frida 库封装、常见错误排查。当用户需要 Frida hook/RPC 调用 Android APP Java 方法、获取加密参数、动态签名时使用。
---

# Frida Android RPC 实战指南

## 一、运行模式选择

| 模式 | 命令 | 适用场景 | 风险 |
|------|------|----------|------|
| attach | `device.attach(pid)` | APP 已运行，需调用已初始化的 SDK | 检测窗口短 |
| spawn | `device.spawn(pkg)` + `resume()` | 需在初始化前注入 bypass | 部分 APP 反 spawn |

### 推荐策略：attach 优先

大部分安全 SDK（APSE/SecGuard 等）在 APP 启动后才初始化，attach 可直接调用已初始化的 Java 类。如果 APP 有反 Frida 检测但有延迟（2~5s），使用 attach + 快速 RPC + 立即 detach 模式。

## 二、JS RPC 导出模板

### 基础模板

```javascript
rpc.exports = {
    ping: function() { return "pong"; },

    // 单个方法调用
    callMethod: function(arg1) {
        return new Promise(function(resolve) {
            Java.perform(function() {
                try {
                    var Cls = Java.use("com.example.TargetClass");
                    var result = Cls.targetMethod(arg1);
                    resolve(result ? result.toString() : "");
                } catch(e) {
                    resolve("ERR:" + e);
                }
            });
        });
    },

    // 合并多个调用（减少 attach 时间）
    getAll: function(arg1, arg2) {
        return new Promise(function(resolve) {
            Java.perform(function() {
                var r = {};
                try { r.field1 = SomeClass.method1().toString(); }
                catch(e) { r.field1 = ""; }
                try { r.field2 = SomeClass.method2(arg1).toString(); }
                catch(e) { r.field2 = ""; }
                resolve(JSON.stringify(r));
            });
        });
    }
};
```

### Java 类型注意事项

```javascript
// Java String 对象 → JS 字符串
var javaStr = SomeClass.getStringValue();
var jsStr = javaStr ? javaStr.toString() : "";

// Java HashMap 参数
var HashMap = Java.use("java.util.HashMap");
var map = HashMap.$new();
map.put("key", "value");

// Java Enum
var EnumClass = Java.use("com.example.MyEnum");
var val = EnumClass.valueOf("ITEM_A");

// 字段访问（.value 获取 Java 字段值）
var obj = SomeClass.someField.value;

// 注意：Frida 包装的 Java String 字段
// 如果字段类型是 String，用 .value 获取实际值
// tokenResult.apdidToken.value  — 如果 apdidToken 是 String 字段
// tokenResult.apdidToken        — 如果 apdidToken 是 getter 方法
```

## 三、Python 封装模式

### FridaSigner 模式（推荐）

```python
import frida, subprocess, time, json

PACKAGE = "com.target.app"

class FridaSigner:
    def __init__(self):
        self._device = frida.get_usb_device()

    def _ensure_running(self):
        r = subprocess.run(["adb", "shell", "pidof", PACKAGE],
                           capture_output=True, text=True)
        pids = r.stdout.strip().split()
        if pids:
            return int(pids[0])
        subprocess.run(["adb", "shell", "am", "start",
                        f"{PACKAGE}/.MainActivity"], capture_output=True)
        time.sleep(8)
        r = subprocess.run(["adb", "shell", "pidof", PACKAGE],
                           capture_output=True, text=True)
        pids = r.stdout.strip().split()
        return int(pids[0]) if pids else None

    def get_all(self, **kwargs):
        pid = self._ensure_running()
        session = self._device.attach(pid)
        try:
            script = session.create_script(JS_CODE)
            script.load()
            raw = script.exports_sync.get_all(*kwargs.values())
            return json.loads(raw) if raw else {}
        finally:
            try: session.detach()
            except: pass

    def get_all_with_restart(self, **kwargs):
        """重启 APP 获取干净状态"""
        subprocess.run(["adb", "shell", "am", "force-stop", PACKAGE],
                       capture_output=True)
        time.sleep(2)
        subprocess.run(["adb", "shell", "am", "start",
                        f"{PACKAGE}/.MainActivity"], capture_output=True)
        time.sleep(8)
        return self.get_all(**kwargs)
```

### 调用模式

```python
signer = FridaSigner()

# 单次获取所有参数
params = signer.get_all(sha="abc123")
print(params["field1"], params["field2"])

# APP 被杀后自动重启
params = signer.get_all_with_restart(sha="abc123")
```

## 四、APSE SDK 调用速查

蚂蚁安全 SDK（com.alipay.alipaysecuritysdk.face.*）常见 API：

```javascript
// APDeviceColor — 设备颜色标签
var C = Java.use("com.alipay.alipaysecuritysdk.face.APDeviceColor");
var color = C.getColorLabel().toString();

// APDID — 设备 Token（注意不同 bizTag）
var A = Java.use("com.alipay.alipaysecuritysdk.face.APDID");
var tr = A.getTokenResult("bizTag_name");
var token = tr.apdidToken.value.toString();

// APSign — 签名/QM
var S = Java.use("com.alipay.alipaysecuritysdk.face.APSign");
var H = Java.use("java.util.HashMap");
var m = H.$new();
m.put("mode", "1");
var qm = S.getColorInfo("bizToken", sha256hex, m);
```

### JADX 逆向定位技巧

1. 搜索 `getColorLabel` / `getTokenResult` / `getColorInfo` 找调用点
2. 看拦截器（如 `h6.a.preHandle`）找 header 赋值逻辑
3. 追踪 bizToken 来源：通常在接口配置类（如 `j6.k`）中按 operationType 分发
4. 追踪 SHA 输入：通常在同一配置类中定义，注意 body 序列化 + uid + hx 拼接方式

## 五、frida-server 部署

```bash
# 下载（匹配手机架构）
# https://github.com/frida/frida/releases
# frida-server-<version>-android-arm64.xz

# 部署
adb push frida-server /data/local/tmp/
adb shell "chmod 755 /data/local/tmp/frida-server"

# 启动（需 root）
adb shell "su 0 /data/local/tmp/frida-server -D"   # APatch
adb shell "su -c /data/local/tmp/frida-server -D"   # Magisk

# 验证
frida -U -l test.js com.target.app
frida-ps -U | findstr target
```

## 六、常见错误排查

| 错误 | 原因 | 解决 |
|------|------|------|
| `need Gadget to attach on jailed Android` | frida-server 未以 root 运行 | `su 0` 或 `su -c` 启动 |
| `unexpected crash while trying to allocate memory` | APP 反篡改检测 | 加 bypass 脚本（hook exit/kill/syscall） |
| `ProcessNotFoundError` | 进程被杀 | 延迟检测触发，缩短 attach 时间或加 bypass |
| `script has been destroyed` | 进程被杀或 session 断开 | 同上 |
| `Java.use` 返回 Error | 类未加载 | spawn 模式 + setTimeout 延迟，或等 APP 完全启动后 attach |
| RPC 返回空字符串 | SDK 未初始化 | 等 APP 完全启动（sleep 8s）再 attach |
| `TypeError: cannot read property 'value'` | Java 字段访问错误 | 检查是字段(.value)还是方法(直接调用) |

## 七、性能优化

1. **合并 RPC 调用**：一个 `getAll` 取代多次 attach/detach
2. **内联 JS 代码**：Python 文件内嵌 JS 字符串，避免文件 IO
3. **连接复用**：如果没有反检测，保持 session 多次调用
4. **批量参数传递**：一次传入所有需要的参数，减少 round-trip
5. **错误重试**：attach 失败自动重启 APP 重试（最多 3 次）

## 八、进程内存 Dump（unidbg 上下文获取）

当需要将 SDK 的运行时状态 dump 出来给 unidbg 复现时：

### Frida warm dump（推荐）
```javascript
// 在单次 RPC 内完成 warmup + dump，避免反检测
rpc.exports = {
    warmupAndDump: function() {
        return new Promise(function(resolve) {
            Java.perform(function() {
                // 1. 调用目标函数激活 SDK 状态
                var S = Java.use("com.target.SignClass");
                S.init();

                // 2. dump SO data+BSS
                var mod = Process.findModuleByName("libtarget.so");
                // ...读取 data section 写入文件
                resolve("done");
            });
        });
    }
};
```

### root /proc/PID/mem dump
```bash
# 不受 Frida 反检测限制，直接读进程内存
adb shell "su -c cat /proc/<PID>/maps" | grep libtarget
adb shell "su -c dd if=/proc/<PID>/mem bs=1 skip=<offset> count=<size> of=/sdcard/dump.bin"
```

### MTE 指针处理
- Android MTE（Memory Tagging Extension）在高字节添加 tag（0xA0-0xBF）
- unidbg/Unicorn 不支持 TBI → 需剥离 tag：`addr & 0x00FFFFFFFFFFFFFF`
- 区分真 heap 指针 vs config data：heap 指针高字节 0xA0-0xBF 或地址 >= 0x7000000000

## 九、LSPosed/Vector Xposed 替代方案

当 Frida 被反检测杀死时，Xposed 模块更隐蔽且持久：

### 优势
- 在 Zygote fork 时注入，比 Frida attach 更早
- 不会在 /proc/maps 中出现 frida-agent
- 不监听 27042 端口
- 持久化运行，无需每次 attach
- TCP 长连接到远程中继服务器，手机无需 ADB 连 PC

### 模块结构 (xposed-signer)
```
xposed-signer/
  app/src/main/java/com/antfans/xposed/
    SignerModule.java          # IXposedHookLoadPackage 入口
    SignerRelayClient.java     # TCP 长连接到中继服务器(推荐)
    SignerSocketServer.java    # 本地 Socket 服务(备用)
  app/src/main/assets/
    xposed_init                # 入口类声明
  build.bat                    # javac+d8+aapt2+apksigner 编译
```

### 中继模式工作流 (推荐)
1. Xposed 模块 hook `Application.onCreate`，在 antfans 进程内启动 `SignerRelayClient`
2. TCP 长连接到中继服务器 `42.193.177.63:443`，JSON Lines 协议
3. PC 端 HTTP 请求 `POST /signer/sign {"method":"getAll","sha":"..."}` -> 中继转发到手机
4. 手机调用 APSE SDK 生成 QM+color+tokens 返回
5. 心跳保活：服务器每 30s 发 `{"cmd":"ping"}`，手机回 `{"type":"pong"}`
6. 断连自动重连：`connectLoop()` 断开后 5 秒重试

### 编译安装
```bash
cd xposed-signer && build.bat
adb uninstall com.antfans.xposed.signer   # 签名不兼容时需先卸载
adb install build/unsigned.apk
# 在 LSPosed/Vector 管理器中激活模块，勾选 com.antfans.fans
adb shell "su -c 'am force-stop com.antfans.fans'"
adb shell "monkey -p com.antfans.fans -c android.intent.category.LAUNCHER 1"
```

### 常见问题
- **phone_connected=false**: 检查手机网络、重启 APP、看 `adb logcat | grep AntfansSigner`
- **503 服务器不可用**: 中继服务器 TCP 连接断了，已加心跳+重试修复
- **INSTALL_FAILED_UPDATE_INCOMPATIBLE**: 新编译签名不同，需先 uninstall 再 install
- **Xposed 框架**: 当前用 Vector v2.0 (LSPosed 分支)，Zygisk 模式，KernelSU
