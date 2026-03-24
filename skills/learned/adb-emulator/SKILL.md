---
name: adb-emulator
description: ADB 与模拟器操作。MuMu/雷电连接区分、ADB 配置、截图状态分析、Frida 注入、i茅台 bypass 启动流程。当用户提到 ADB/模拟器/MuMu/雷电/Frida spawn/截图分析模拟器/i茅台过检测/模拟器连接 时使用。
---

# ADB 与模拟器对接 (Auto-Learned)

用户常用 MuMu + 雷电(LDPlayer) 模拟器 + ADB 进行移动应用分析和自动化。

## 模拟器识别与区分

MuMu 和雷电可能共享 CET-AL00 等相同系统镜像，通过 `emulator-xxxx` 和 `127.0.0.1:port` 无法区分。

### 区分方法

| 方法 | MuMu | 雷电(LDPlayer) |
|------|------|----------------|
| `getprop ro.product.model` | 取决于镜像（可能相同） | 取决于镜像（可能相同） |
| ADB serial | `emulator-5556` (多开递增) | `127.0.0.1:port` (端口变化) |
| 专用工具 | 无 | `F:\leidian\LDPlayer9\ldconsole.exe` |
| 可靠区分 | 关掉雷电后剩余的是 MuMu | 用 `ldconsole list2` 确认 |

### 关键教训
- **切勿通过 ADB 的 `am start/force-stop` 操作目标 APP**，可能路由到错误设备
- 雷电必须用 `ldconsole runapp/killapp` 启停应用
- 雷电 ADB 端口在重启后会变（5557/16416/其他），每次需重新扫描
- 两个模拟器同时运行时，标准 `adb` 命令可能被路由到任一设备

### 雷电模拟器操作

```python
LDCONSOLE = r"F:\leidian\LDPlayer9\ldconsole.exe"
LD_ADB = r"F:\leidian\LDPlayer9\adb.exe"

# 列出实例
# ldconsole list2
# 输出: index,name,top_pid,bottom_pid,running,pid,vbox_pid,w,h,dpi

# 启停应用（不走 ADB，保证目标正确）
# ldconsole runapp --index 1 --packagename com.zenmen.palmchat
# ldconsole killapp --index 1 --packagename com.zenmen.palmchat

# 安装应用
# ldconsole installapp --index 1 --filename path/to/apk

# ADB 命令（通过 ldconsole 路由到正确实例）
# ldconsole adb --index 1 --command "shell su -c 'cat /data/...'"

# 文件操作用 LDPlayer 自己的 ADB
# LD_ADB -s 127.0.0.1:PORT push local_file /data/local/tmp/
# ldconsole adb --index 1 --command "shell su -c 'cp /data/local/tmp/... /data/data/...'"
```

### 雷电 ADB 端口扫描

```python
import subprocess
LD_ADB = r"F:\leidian\LDPlayer9\adb.exe"
for port in [5555, 5557, 5559, 16384, 16416, 16448]:
    r = subprocess.run([LD_ADB, "connect", f"127.0.0.1:{port}"],
                       capture_output=True, text=True)
    if "connected" in r.stdout and "cannot" not in r.stdout:
        print(f"LDPlayer at port {port}")
        break
```

## ADB 安装与配置

```powershell
# Windows 下载并配置 ADB 全局环境
Invoke-WebRequest -Uri "https://dl.google.com/android/repository/platform-tools-latest-windows.zip" -OutFile "$env:TEMP\platform-tools.zip"
Expand-Archive "$env:TEMP\platform-tools.zip" -DestinationPath "C:\adb" -Force
[Environment]::SetEnvironmentVariable("PATH", "$env:PATH;C:\adb\platform-tools", "User")
```

## MuMu 模拟器连接

MuMu 多开实例使用 `emulator-5556`, `emulator-5558` 等递增的 serial。

```python
import subprocess

def connect_mumu():
    """连接 MuMu 模拟器 (默认 emulator-5556)"""
    result = subprocess.run(['adb', 'devices'], capture_output=True, text=True)
    for line in result.stdout.splitlines():
        if 'emulator-' in line and 'device' in line:
            return line.split()[0]
    return None
```

## ADB 数据拦截

用户需求：监控某个进程通过 ADB 读取了什么数据

```python
# 创建 ADB 包装脚本记录所有 ADB 命令和数据
import subprocess
import logging

logging.basicConfig(filename='adb_log.txt', level=logging.DEBUG)

def adb_wrapper(cmd):
    """ADB 命令包装器，记录所有输入输出"""
    full_cmd = ['adb'] + cmd
    logging.info(f"CMD: {' '.join(full_cmd)}")
    result = subprocess.run(full_cmd, capture_output=True)
    logging.info(f"OUT: {result.stdout[:500]}")
    return result
```

## Frida + 模拟器

```python
# Frida 连接 MuMu 模拟器中的目标 App
# 需要先 push frida-server 到模拟器
# adb push frida-server /data/local/tmp/
# adb shell chmod 755 /data/local/tmp/frida-server
# adb shell /data/local/tmp/frida-server &
```

## C++ 游戏辅助 (jiguang)

- IDA + CE 附加 MuMu 模拟器进程
- 查找游戏内存数据（坐标、血量）
- 绘制小地图显示人物/野怪位置
- 血量 <= 0 不显示

## scrcpy 远程投屏

```powershell
# 使用 scrcpy 投屏控制模拟器
# 安装路径: E:\Develop\DebugTool\scrcpy-win64-v2.0
scrcpy --serial 127.0.0.1:7555
```

## MuMu 截图与状态分析

需要查看模拟器当前界面、分析目标 App 是否卡在门口时，**先执行截图再分析**，不要等用户自己截图。

```powershell
# 推荐（PNG 到本地文件）
adb -s 127.0.0.1:7555 exec-out screencap -p > "<workspace>/mumu_screen.png"

# 备选（兼容性更好）
adb -s 127.0.0.1:7555 shell screencap -p /sdcard/mumu_screen.png
adb -s 127.0.0.1:7555 pull /sdcard/mumu_screen.png "<workspace>/mumu_screen.png"
```

用 Read 工具读取截图 PNG，根据画面分析状态并给出结论或下一步建议。若 `exec-out screencap -p` 在 Windows 下乱码，改用 shell screencap + pull 方式。

## i茅台 MuMu bypass 启动流程

涉及 i茅台 + MuMu + Frida bypass 时，按以下顺序执行并观察输出：

1. **连接**: `adb connect 127.0.0.1:7555`（失败则 `adb kill-server` 后重试）
2. **确认**: `adb devices`（unauthorized 则提示用户确认弹窗）
3. **端口转发**: `adb -s 127.0.0.1:7555 forward tcp:27042 tcp:27042`
4. **检查 frida-server**: `adb -s 127.0.0.1:7555 shell "ps -A | grep frida"`（无进程则启动）
5. **启动 bypass**: `frida -U -f com.moutai.mall -l <bypass.js> --no-pause`
6. **循环重试**: 任一步失败执行诊断后重试 2~3 次，仍失败向用户请求一条明确操作

技术细节（hook 顺序、MuMu 应用级伪装、child-gating）以 `android-anti-detect-bypass` skill 为准。

## 注意事项

- MuMu 多开以 `emulator-5556/5558/5560` 递增
- 雷电 ADB 端口不固定，重启后需重新扫描连接
- 雷电安装路径: `F:\leidian\LDPlayer9`
- 两个模拟器同时运行时，标准 `adb` 的 `am start` 等命令可能路由到错误设备
- 雷电操作应用务必用 `ldconsole runapp/killapp`，不用 `adb shell am`
- `ldconsole push` 不等于 `adb push`，行为不同（通过 Pictures 目录中转）
- 文件推送用 `LD_ADB -s serial push`，启停用 `ldconsole`
- WSL2 / Docker Desktop 可能与模拟器冲突
