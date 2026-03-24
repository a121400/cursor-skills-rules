---
name: pyinstaller-desktop-app
description: PyInstaller 打包 EXE。原生 DLL 打包(tls_client)、sys.frozen 路径修复(_MEIPASS)、PyQt5 打包、进程退出清理。当用户要求「打包」「编译EXE」「发给客户」时使用。
---

# PyInstaller 桌面应用打包

将 Python + PyQt5 桌面程序打包为无需 Python 环境的独立 EXE。

## 基本命令

```bash
pyinstaller --noconfirm --onefile --windowed --name "AppName" main.py
```

| 参数 | 作用 |
|---|---|
| `--onefile` | 单文件 EXE，解压到临时目录运行 |
| `--windowed` | 无控制台窗口 (GUI 应用必须) |
| `--console` | 带控制台 (调试用，可看报错) |
| `--clean` | 清除缓存重新构建 |
| `--name` | 输出文件名 |

## 含原生 DLL 的库打包 (tls_client)

tls_client 依赖 Go 编译的 `tls-client-64.dll`，必须手动带入:

```bash
pyinstaller --onefile --windowed --name "App" \
  --add-data "E:\python311\Lib\site-packages\tls_client\dependencies\tls-client-64.dll;tls_client\dependencies" \
  --hidden-import tls_client \
  --hidden-import tls_client.cffi \
  --hidden-import tls_client.sessions \
  --hidden-import tls_client.cookies \
  --hidden-import tls_client.response \
  --hidden-import tls_client.settings \
  --hidden-import tls_client.structures \
  --hidden-import tls_client.exceptions \
  main.py
```

### 查找 DLL 路径

```python
python -c "import tls_client.cffi; print(tls_client.cffi.library)"
# => <CDLL 'E:\...\tls_client\dependencies\tls-client-64.dll'>
```

### add-data 格式

Windows: `--add-data "源路径;目标相对路径"` (分号)
Linux/Mac: `--add-data "源路径:目标相对路径"` (冒号)

## frozen 路径修复 (关键坑)

PyInstaller 打包后 `__file__` 指向临时解压目录 (`_MEIPASS`)，导致:
- 日志文件写到临时目录 (用户找不到)
- Excel/文件导出到临时目录
- 配置文件读取失败

### 修复方案

```python
import sys
from pathlib import Path

if getattr(sys, 'frozen', False):
    _HERE = Path(sys.executable).resolve().parent  # EXE 所在目录
else:
    _HERE = Path(__file__).resolve().parent  # 源码目录
```

所有用到 `_HERE` 的文件都要加这个判断:
- `main.py` (日志文件路径)
- `gui.py` (文件对话框默认目录、导出路径)
- 任何读写本地文件的模块

## PyQt5 打包要点

- PyInstaller 自动检测 PyQt5 依赖，一般不需要额外 hidden-import
- `--windowed` 必须加，否则 GUI 后面会跟一个黑色控制台窗口
- 调试时用 `--console` 替代 `--windowed` 看报错

## 调试打包后的问题

1. **EXE 双击闪退**: 用 `--console` 模式重新打包，看控制台报错
2. **DLL 找不到**: 检查 `--add-data` 的目标路径是否匹配库的加载路径
3. **文件导出到临时目录**: 检查是否做了 `sys.frozen` 路径修复
4. **界面空白/卡死**: 通常是线程内异常未捕获，加 try/except 和 logging

## 进程退出

PyQt5 + 多线程应用关闭时可能残留后台进程:

```python
import os
from lib.checker import request_global_stop

def main():
    app = QApplication(sys.argv)
    win = MainWindow()
    win.show()
    code = app.exec_()
    request_global_stop()  # 通知所有线程停止
    os._exit(code)         # 强制退出，不等线程
```

线程内的 `time.sleep` 改为可中断版本:

```python
import threading

_stop_event = threading.Event()

def _interruptible_sleep(seconds):
    end = time.monotonic() + seconds
    while time.monotonic() < end:
        if _stop_event.is_set():
            return
        time.sleep(min(0.5, end - time.monotonic()))
```

## 典型产物

| 文件 | 大小 | 说明 |
|---|---|---|
| `dist/App.exe` | 30-70 MB | 单文件 EXE (含 Python 运行时 + 依赖) |
| `build/` | - | 构建缓存，可删除 |
| `App.spec` | - | PyInstaller 配置，可删除 |

`.gitignore` 中排除: `build/`, `dist/`, `*.spec`

Last updated: 2026-03-21
