---
name: antfans-sms-sign
description: 鲸探(antfans)完整协议逆向方案。短信签名(QM/Sign)通过 unidbg 离线生成，下单协议需 Frida attach 获取动态设备参数(x-device-color/token/ap-token/QM)。已实现 bizStatusCode=10000 纯协议下单。当用户提到鲸探/antfans/下单/抢购/QM/Sign/APSE 时使用。
---

# 鲸探(antfans) 完整协议逆向方案

## 方案总览

| 场景 | 方案 | 关键参数 |
|------|------|----------|
| 短信签名 | unidbg 离线（apse-signer.jar） | QM + Sign |
| 下单协议 | Frida attach + curl_cffi | QM + Sign + x-device-color + x-device-token + x-device-ap-token |

## 一、签名链路（短信/通用）

```
hx = "{毫秒时间戳},{UUID},{random(0-999)}"
Ts = c10to64(currentTimeMillis)
sha = SHA256(JSON序列化请求体 + uid + hx)
signContent = "Operation-Type=...&Request-Data=base64(body)&Ts=..."

QM = SignJNIBridge.getColorInfo(0, bizToken, sha, mode)
Sign = MD5("d65e27ce3bd3e2eb83d8bb21e452eeb0" + "&" + signContent)
```

### QM bizToken 与 mode 按接口区分

| 接口 | bizToken | mode | SHA256 输入 |
|------|----------|------|------------|
| sendSmsCode（短信） | `"antfans"` | `0` | `JSON(body) + uid + hx` |
| nftOrderCreate（下单） | `"icYRH1LO1lGJhYj2KSm9RR5r"` | `1` | `JSON({"itemId":"xxx"}) + uid + hx` |

> 来源：JADX 分析 `j6.k.a()` — 不同 operationType 对应不同 bizToken/mode/sha 输入。

## 二、下单协议完整方案（Frida 动态参数）

### 动态参数获取（Frida attach）

| 参数 | APSE API | bizTag/参数 |
|------|----------|------------|
| x-device-color | `APDeviceColor.getColorLabel()` | 无参 |
| x-device-token | `APDID.getTokenResult("zorro_antfans").apdidToken` | bizTag=`zorro_antfans` |
| x-device-ap-token | `APDID.getTokenResult("antfans_android").apdidToken` | bizTag=`antfans_android` |
| QM（下单） | `APSign.getColorInfo(bizToken, sha256hex, {mode:"1"})` | bizToken=`icYRH1LO1lGJhYj2KSm9RR5r` |

> **关键发现**：x-device-token 和 x-device-ap-token 用不同 bizTag 生成，JADX `j6.g` 确认。

### 请求构造要点

1. **Cookie 清洁**：仅携带 `sessionId`，不带 `acw_tc`（`acw_tc` 属于 mgs-normal 域）
2. **时间戳同步**：`hx` 和 `x-device-timestamp` 用同一 `now` 值
3. **TRACEID**：格式 `{Did}{Ts}_{RpcId}`
4. **TLS 指纹**：curl_cffi `impersonate="chrome120"`
5. **直连优先**：住宅代理 IP 可能被风控标记，直连成功率更高
6. **body 必含 `quantity: 1`**

### 30831 风控排查清单

| 排查项 | 说明 |
|--------|------|
| QM bizToken/mode | 下单必须用 `icYRH1LO1lGJhYj2KSm9RR5r` / `1` |
| device-token 区分 | `zorro_antfans` vs `antfans_android` 不能混 |
| itemId 正确性 | 用 SunnyNet 抓最新真机下单确认 |
| 代理 IP | 住宅代理仍可能触发 30831，优先测直连 |
| Cookie 多余字段 | mgs-biz 域不应带 mgs-normal 的 acw_tc |
| hx/timestamp 一致性 | 两者必须同源 now 值 |

## 三、项目文件索引

```
JTT/
  unidbg/unidbg-android/
    target/apse-signer.jar            -- QM 生成器（unidbg fat JAR）
    src/test/java/com/apse/APSETest.java
  sms_sender/
    apse_signer.py                    -- Python 封装 JAR（短信场景）
    send_sms_jar.py                   -- 端到端发送
  antfans_rush/
    frida_signer.py                   -- Frida RPC 封装（下单场景）
    order_frida_debug.py              -- 成功下单脚本（bizStatusCode=10000）
    rush_buy.py                       -- 抢购主脚本（代理/TLS/pagets）
    captured_order_reference.json     -- 真机抓包参考
    scripts/
      frida_bypass_detect.js          -- Frida 反检测绕过
      frida_rpc_only.js               -- 纯 RPC（无 bypass）
```

## 四、Sign 算法

`Sign = MD5("d65e27ce3bd3e2eb83d8bb21e452eeb0" + "&" + signContent)`

密钥是 mPaaS appSecret，纯 Python `hashlib.md5` 计算。signType=0。

## 五、unidbg QM（离线方案，短信场景用）

```bash
echo "antfans|<sha256_hex>|0" | java --enable-native-access=ALL-UNNAMED \
  -Dapse.resources=<资源目录> -jar apse-signer.jar
# 输出: QM_OK|<base64_string>
```

```python
from apse_signer import APSESigner
signer = APSESigner()
signer.start()
qm = signer.get_qm(sha_hex)
signer.close()
```

## 六、关键逆向发现

### APSE 8.0.0 SCP VM 架构（深度分析）
- `SignJNIBridge`（QM）和 `EdgeNativeBridge`（Sign SCP）共用 libAPSE_8.0.0.so
- `initSI` 做 SO 完整性校验，必须在 hook 前调用
- QM 有两种模式：
  - **Bridge 路径**：151字符 AQAA_（无 SCP，unidbg 可生成）
  - **SCP 路径**：193字符 AQAB_（含 32B SCP block，每次调用不同）

### SCP 32 字节结构
```
Bytes 0-3:  4b8f67bc (magic, 常量)
Bytes 4-5:  递增计数器
Bytes 6-7:  6935 (常量)
Bytes 8-11: 9d010000 (常量)
Bytes 12-31: 20字节密码学计算结果 (每次不同)
```
- SCP block 由 `sub_6F578`（独立 SCP 生成 VM）计算
- 通过 `sub_6B874`（type=0x4F0）组装到 QM
- 调用链：getColorInfo VM(0x5C91C) → Bridge VM(0x6E1A4) → SCP VM(0x6F578) → QM组装(0x6B874)

### unidbg SCP 失败原因
- SO 内有自解密代码段（需要 warm text dump）
- 三层嵌套 VM 分发器，分支条件依赖 data section 中的密钥/配置表
- MTE（Memory Tagging Extension）指针遍布 heap，Unicorn 不支持 TBI
- 已尝试：warm dump 597MB + TBI handler + SO mirror + MTE strip → VM 操作码序列仍无法重现（蝴蝶效应）
- **结论**：APSE 8.0.0 的 SCP VM 需要完整 Android 运行时，unidbg 无法模拟

### 风控体系
- 30831 = 操作频繁/设备风控，非频率限制
- 设备指纹三件套（color/device-token/ap-token）任一错误即触发
- QM 的 bizToken/mode 错误也触发 30831
- unidbg Bridge QM（151→193补丁）被服务器拒绝，SCP block 必须真实计算
- x-device-color 每次调用动态生成（非静态缓存）
- 代理 IP 质量影响：住宅代理仍可能被标记，直连优先
