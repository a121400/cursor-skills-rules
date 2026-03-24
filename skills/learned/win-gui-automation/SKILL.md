---
name: win-gui-automation
description: Windows GUI 自动化。pywin32 窗口查找/后台截图(BitBlt)/后台点击(PostMessage)、GDI 资源清理顺序、PyQt5 悬浮窗/全局热键、OCR 集成。当用户要做 Windows 窗口自动化/后台操作/桌面小工具时使用。
---

# Windows GUI 自动化 (Auto-Learned)

用户常做的任务：通过窗口类名找句柄，后台截图，图像识别按钮，后台点击。

## 窗口查找

```python
import win32gui

# 按类名找窗口
hwnd = win32gui.FindWindow("Qt5151QWindowIcon", None)

# 枚举所有窗口找匹配
def find_windows_by_class(class_name):
    results = []
    def callback(hwnd, _):
        if win32gui.GetClassName(hwnd) == class_name:
            results.append(hwnd)
        return True
    win32gui.EnumWindows(callback, None)
    return results
```

## 后台截图 (不激活窗口)

```python
import win32gui, win32ui, win32con
from PIL import Image

def capture_window(hwnd):
    rect = win32gui.GetWindowRect(hwnd)
    w = rect[2] - rect[0]
    h = rect[3] - rect[1]

    hwndDC = win32gui.GetWindowDC(hwnd)
    mfcDC = win32ui.CreateDCFromHandle(hwndDC)
    saveDC = mfcDC.CreateCompatibleDC()
    bitmap = win32ui.CreateBitmap()
    bitmap.CreateCompatibleBitmap(mfcDC, w, h)
    saveDC.SelectObject(bitmap)
    saveDC.BitBlt((0, 0), (w, h), mfcDC, (0, 0), win32con.SRCCOPY)

    bmpinfo = bitmap.GetInfo()
    bmpstr = bitmap.GetBitmapBits(True)
    img = Image.frombuffer('RGB', (bmpinfo['bmWidth'], bmpinfo['bmHeight']), bmpstr, 'raw', 'BGRX', 0, 1)

    # GDI 资源清理顺序很重要，反了会 DeleteDC failed
    saveDC.DeleteDC()
    mfcDC.DeleteDC()
    win32gui.ReleaseDC(hwnd, hwndDC)
    win32gui.DeleteObject(bitmap.GetHandle())

    return img
```

## GDI 资源清理顺序

**必须严格按顺序释放，否则 DeleteDC failed：**
1. `saveDC.DeleteDC()` (兼容DC先删)
2. `mfcDC.DeleteDC()` (窗口DC再删)
3. `win32gui.ReleaseDC(hwnd, hwndDC)` (释放原始DC)
4. `win32gui.DeleteObject(bitmap.GetHandle())` (最后删位图)

## 后台点击 (PostMessage)

```python
import win32api, win32con

def click_background(hwnd, x, y):
    lparam = win32api.MAKELONG(x, y)
    win32gui.PostMessage(hwnd, win32con.WM_LBUTTONDOWN, win32con.MK_LBUTTON, lparam)
    win32gui.PostMessage(hwnd, win32con.WM_LBUTTONUP, 0, lparam)
```

## OCR 选择

| 方案 | 优缺点 |
|------|--------|
| rapidocr-onnxruntime | 纯 pip，推荐，Python 3.14 兼容 |
| pytesseract | 需额外装 Tesseract.exe，不推荐 |
| PaddleOCR | 重，依赖多 |

## 绿色按钮检测 (Pillow)

```python
def has_green_button(img, roi=None):
    if roi:
        img = img.crop(roi)
    pixels = list(img.getdata())
    green_count = sum(1 for r, g, b in pixels if g > 150 and r < 100 and b < 100)
    return green_count > len(pixels) * 0.01
```

## PyQt5 桌面悬浮窗/快捷工具

用户常需要开发 Windows 桌面小工具（浮窗、快捷启动器、目录导航器等）：

```python
from PyQt5.QtWidgets import QWidget, QListWidget, QApplication
from PyQt5.QtCore import Qt

class FloatingPanel(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowFlags(Qt.FramelessWindowHint | Qt.WindowStaysOnTopHint | Qt.Tool)
        self.setAttribute(Qt.WA_TranslucentBackground)
        self.list_widget = QListWidget(self)

    def toggle_visibility(self):
        self.setVisible(not self.isVisible())
```

### 全局热键

```python
# 方案1: pynput (推荐，纯 pip)
from pynput import mouse
listener = mouse.Listener(on_click=on_click)

# 方案2: keyboard 库
import keyboard
keyboard.add_hotkey('ctrl+space', callback)
```

### 注意事项
- 鼠标中键 Hook 需要 `pynput`，不要用 `pyHook`（已废弃）
- 窗口 `Qt.Tool` flag 可以避免在任务栏显示
- 全局热键注册后要确保程序退出时注销

## 音频播放注意事项

- Windows 后台进程播放音频可能无声，需确认音频设备输出
- 使用 `playsound` 或 `pygame.mixer` 时，确保在主线程或正确初始化
- 测试时先用简单的 `winsound.PlaySound` 验证音频通路

## C++/Qt6 桌面程序开发

用户有 Qt6 环境 (`E:\qt\Tools\QtCreator\bin`)，常做 C++ 桌面项目：
- IntraMirror1: C++ 一比一复刻重构
- jiguang: C++ 绘制游戏小地图
- sxdcg: 桌面抢购客户端

```
项目重构模式:
1. 分析现有代码，制定详细计划
2. 多 agent 并行执行独立模块
3. GUI 界面用 Qt6
```

## 微信/进程 Call 注入 (wxx)

```csharp
// C# 调用微信发消息 Call
// 1. CE/x64dbg 附加微信进程
// 2. 找到发消息的 Call 地址
// 3. C# 远程注入调用

// 测试方式: 输入 "test8888xyz" 验证 Call 是否正确
```

## VSIX 逆向 (rz)

- 逆向 VSCode 扩展文件 (.vsix)
- 在 extension.js 中搜索"激活"、"登录"、"校验"字符串
- 定位校验位置，修改后重新打包

## 文件/计划放置偏好

- 生成的计划文件(PLAN.md)、TODO等应放在**当前项目目录**，不要放到 C: 系统目录
- 每次任务创建文件时都在当前工作目录下
