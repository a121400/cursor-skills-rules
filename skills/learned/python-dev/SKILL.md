---
name: python-dev
description: Python 开发规范。环境兼容性(Python 3.14 + Windows)、项目目录结构(lib/包模式)、依赖版本约束、轻量库选择。创建新 Python 项目、整理目录结构、选择依赖库时使用。
---

# Python 开发规范

## 环境信息

- **Python 3.14**, Windows 10
- 很多 C 扩展包在 3.14 上缺少预编译 wheel，优先选纯 Python 库

## 依赖版本规则

- 不要固定精确版本（`==306`），用范围约束（`>=306`）
- requirements.txt 示例：`pywin32>=306`、`Pillow>=10.0`

## 常见替代方案

| 避免使用 | 替代方案 | 原因 |
|----------|----------|------|
| opencv-python + numpy | Pillow | 3.14 缺 wheel，Pillow 更轻量 |
| pytesseract | rapidocr-onnxruntime | 纯 pip 安装，无需 Tesseract.exe |
| PaddleOCR | rapidocr-onnxruntime | 依赖少，3.14 兼容 |
| playsound (旧版) | pygame.mixer / winsound | 新 Python 兼容问题 |

## 项目目录结构

```
project_name/
  main.py               # 运行入口（可多个）
  config.json            # 配置文件
  data.txt               # 数据文件
  lib/                   # 业务模块包
    __init__.py          # 统一导出
    module_a.py
    bridge.js            # 非 Python 依赖也放 lib/
```

### 规则

1. **根目录只放**：运行脚本（入口）、配置文件、数据文件、输出文件
2. **lib/ 放全部业务模块**：作为 Python 包，带 `__init__.py`
3. **lib 内部用相对 import**：`from .common import xxx`
4. **运行脚本统一从 lib 导入**：`from lib import xxx`
5. **禁止 sys.path hack**：不允许 `sys.path.insert` 跨目录导入
6. **`__init__.py` 统一导出**：把各模块对外接口 re-export
7. **非 Python 文件**（JS/WASM/SO）跟随使用它的模块放在 lib/ 内，用 `Path(__file__).parent` 定位

### __init__.py 写法

```python
from .module_a import FuncA, ClassA
from .module_b import FuncB

__all__ = ["FuncA", "ClassA", "FuncB"]
```

### 运行脚本写法

```python
from lib import FuncA, ClassA, FuncB
from pathlib import Path

_HERE = Path(__file__).resolve().parent
CONFIG_PATH = _HERE / "config.json"
```

## 其他语言环境

- **Go**: 用户也用 Go 开发
- **C++**: Qt6 项目用 QtCreator (`E:\qt\Tools\QtCreator\bin`)
- **C#**: 用于进程注入（如微信 Call 调用）
- **易语言**: 用于部分抢购脚本，与 Python 协作

## 打包注意

- `pyinstaller` 打包后测试无报错才算完成（详见 `pyinstaller-desktop-app` skill）
- 写 Windows 桌面程序时，ctypes + win32api 优先，减少重依赖
