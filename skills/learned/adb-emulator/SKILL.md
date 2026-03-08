---
name: adb-emulator
description: ADB 与模拟器对接模式。MuMu 模拟器连接、ADB 配置、进程数据拦截、Frida 移动端注入。从 jiguang/fenximook/11111/666 等项目提取。
---

# ADB 与模拟器对接 (Auto-Learned)

用户常用 MuMu 模拟器 + ADB 进行移动应用分析和自动化。

## ADB 安装与配置

```powershell
# Windows 下载并配置 ADB 全局环境
# 1. 下载 platform-tools
Invoke-WebRequest -Uri "https://dl.google.com/android/repository/platform-tools-latest-windows.zip" -OutFile "$env:TEMP\platform-tools.zip"
Expand-Archive "$env:TEMP\platform-tools.zip" -DestinationPath "C:\adb" -Force

# 2. 添加到 PATH
[Environment]::SetEnvironmentVariable("PATH", "$env:PATH;C:\adb\platform-tools", "User")
```

## MuMu 模拟器连接

```python
import subprocess

def connect_mumu():
    """连接 MuMu 模拟器 (默认端口 7555)"""
    result = subprocess.run(['adb', 'connect', '127.0.0.1:7555'], capture_output=True, text=True)
    print(result.stdout)
    return 'connected' in result.stdout

def list_devices():
    result = subprocess.run(['adb', 'devices'], capture_output=True, text=True)
    return result.stdout
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

## 注意事项

- MuMu 默认 ADB 端口 7555，网易 MuMu 安装路径 `C:\Program Files\Netease\MuMu`
- `MuMuVMMHeadless.exe` 可直接连接其 TCP 流量
- WSL2 / Docker Desktop 可能与模拟器冲突
- ADB 包装脚本记录时注意编码问题
