---
name: antfans-sms-sign
description: 鲸探(antfans)验证码签名逆向方案。蚂蚁安全SDK APSE 8.0.0 签名生成，Frida RPC 调用 APSign.getColorInfo / SecurityUtil.signature，MuMu + Frida 后台签名服务，Python 协议发送。
---

# 鲸探验证码签名 (Auto-Learned)

## 项目概况

目标：Python 纯协议发送鲸探 APP 的短信验证码，无需操作 APP 界面。

核心难点：Sign 和 qm 签名头由蚂蚁安全 SDK (APSE 8.0.0) 的 native .so 生成，
内部有 OLLVM 控制流平坦化 + AVMP 虚拟机 + 字符串加密，无法纯 Python 复现。

最终方案：MuMu 模拟器 + Frida RPC 调用设备上的签名函数。

## 签名链路

```
hx = "{毫秒时间戳},{UUID},{random(0-999)}"
Ts = HexaDecimalConvUtil.c10to64(currentTimeMillis)  -- 自定义 base64
x-device-timestamp = String.valueOf(currentTimeMillis)

qm = APSign.getColorInfo("icYRH1LO1lGJhYj2KSm9RR5r",
       SHA256(排序后的请求体JSON + hx), {mode: "1"})
     --> SignManager.getColorInfo() --> native libAPSE_J.so --> libAPSE_8.0.0.so

Sign = SecurityUtil.signature(appkey="ALIPUB059F038311550_ANDROID",
       signType=0, content="Operation-Type=...&Request-Data=base64(body)&Ts=...")
     --> native SecurityGuard SDK
```

Sign 和 qm 都需要 native SDK 调用，不是标准 MD5/HMAC。

## 项目文件

```
JTT/
  bin/frida-server-arm64   -- frida-server 17.7.3 ARM64 (已内置)
  antfans_base.apk         -- 鲸探 APK
  frida_rpc_sign.js        -- Frida 签名脚本 (轮询文件通信)
  start_service.py         -- 一键启动: 检测模拟器/APP/frida/签名脚本
  send_sms_final.py        -- 发送验证码: python send_sms_final.py <手机号>
  api_sms.md               -- API 文档
  requirements.txt         -- requests>=2.28.0
```

## 使用方式

```bash
# 1. 启动 MuMu 模拟器 (emulator-5556)
# 2. 一行命令发送验证码
python send_sms_final.py 13800138000
```

start_service.py 自动执行：
1. 检测 MuMu 模拟器连接 (emulator-5556)
2. 检测/安装鲸探 APK
3. 检测/推送/启动 frida-server
4. 注入 frida_rpc_sign.js 签名脚本
5. 通过文件通信获取签名 -> 发送 HTTP 请求

## Frida 签名脚本机制

frida_rpc_sign.js 在设备上运行，通过文件轮询通信：
- Python 写入 /data/local/tmp/sign_request.json
- Frida 脚本每 500ms 检查，调用 native 签名函数
- 结果写入 /data/local/tmp/sign_result.json
- Python 读取结果

需要 frida CLI 模式加载（不是 Python API），因为 Frida 17.x 的 Python create_script
不包含 Java bridge。命令：`frida -U -p <PID> -l frida_rpc_sign.js --eternalize`

## 技术发现

### APSE 8.0.0 vs 旧版 sgmain
- 鲸探使用 APSE 8.0.0 (新版)，不走 sgmain 的 doCommandNative
- 直接通过 SignManager/SecurityUtil 的 native 方法签名
- initSI 环境初始化做全地址空间扫描，unidbg 无法通过

### unidbg 尝试结论
- JNI_OnLoad 成功（两个 .so 都能加载）
- RegisterNatives 成功（所有 native 方法注册完毕）
- initSI 在环境检测阶段死循环/崩溃
- Dynarmic 后端：C++ 层 abort() 无法绕过（需重编译 native dll）
- Unicorn2 后端：不崩溃但执行卡死（AVMP 执行 7500 万指令后跳到零地址）
- 内存 dump 恢复：指针重定位问题导致不可行

### 关键 Java 类路径
- j6.k -- hx/qm 生成工具类
- j6.f -- RPC 签名器实现 (implements j6.i)
- h6.a -- CommonInterceptor (RPC 拦截器，设置 hx/qm/x-device-timestamp)
- com.alipay.alipaysecuritysdk.sign.manager.SignManager -- native 签名管理
- com.alipay.alipaysecuritysdk.face.APSign -- native 签名入口
- com.alipay.mobile.common.netsdkextdependapi.security.SecurityUtil -- RPC Sign 入口
