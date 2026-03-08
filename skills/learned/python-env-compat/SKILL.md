---
name: python-env-compat
description: 用户 Python 环境兼容性规则。Python 3.14 + Windows，依赖版本不要写死，优先用轻量库。
---

# Python 环境兼容性

用户环境: **Python 3.14, Windows 10**

## 依赖版本规则

- 不要固定精确版本（`==306`），用范围约束（`>=306`）
- 很多库在 Python 3.14 上还没有预编译 wheel，优先选择纯 Python 或已适配的库

## 常见替代方案

| 避免使用 | 替代方案 | 原因 |
|----------|----------|------|
| opencv-python + numpy | Pillow | 3.14 缺 wheel，Pillow 更轻量 |
| pytesseract (需装 Tesseract.exe) | rapidocr-onnxruntime | 纯 pip 安装 |

## requirements.txt 写法

```txt
# 用范围约束，不要固定版本
pywin32>=306
Pillow>=10.0
```

## 更多替代方案

| 避免使用 | 替代方案 | 原因 |
|----------|----------|------|
| PaddleOCR | rapidocr-onnxruntime | 依赖少，3.14 兼容 |
| playsound (旧版) | pygame.mixer / winsound | playsound 在新 Python 上有兼容问题 |

## 其他语言环境

- **Go**: 用户也用 Go 开发（Go1 项目 - UI 程序 py+Qt + 验证码识别）
- **C++**: Qt6 项目用 QtCreator (`E:\qt\Tools\QtCreator\bin`)
- **C#**: 用于进程注入（如微信 Call 调用）
- **易语言**: 用于部分抢购脚本，与 Python 协作

## 打包注意

- `pyinstaller` 打包后测试无报错才算完成
- 用户会说"打包测试没问题能够正常运行不报错就行我先睡觉了"
- 需要无人值守完成所有打包和测试

## 注意事项

- `pywin32` 在 3.14 上版本号可能跳到 311+，不要写死 306
- 写 Windows 桌面程序时，ctypes + win32api 优先，减少重依赖
- 新 Python 版本下很多 C 扩展包缺少预编译 wheel，优先选纯 Python 库
- ddddocr (滑块验证码识别) 可能也有 3.14 兼容问题，提前测试
