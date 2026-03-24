---
name: antfans-sms-sign
description: 鲸探(antfans)验证码签名完整逆向方案。QM 通过 unidbg 离线生成（libAPSE_8.0.0.so SignJNIBridge.getColorInfo），Sign 为纯算法 MD5(secret+"&"+signContent)。已打包 apse-signer.jar，Python 调用即可发送，无需 Frida/设备。
---

# 鲸探验证码签名 - 完整逆向方案

## 最终方案（纯离线，无需 Frida/设备）

- **QM**：unidbg 加载 `libAPSE_8.0.0.so`，调用 `SignJNIBridge.getColorInfo` 生成
- **Sign**：`MD5("d65e27ce3bd3e2eb83d8bb21e452eeb0" + "&" + signContent)`
- **交付**：`apse-signer.jar`（fat JAR ~210MB）+ `apse_signer.py`（Python 封装）

## 签名链路

```
hx = "{毫秒时间戳},{UUID},{random(0-999)}"
Ts = c10to64(currentTimeMillis)  -- 自定义 base64
sha = SHA256(JSON序列化请求体 + hx)
signContent = "Operation-Type=...&Request-Data=base64(body)&Ts=..."

qm = SignJNIBridge.getColorInfo(0, "icYRH1LO1lGJhYj2KSm9RR5r", sha, 1)
     --> unidbg 模拟 libAPSE_8.0.0.so，返回 153 字符 base64

Sign = MD5("d65e27ce3bd3e2eb83d8bb21e452eeb0" + "&" + signContent)
     --> 纯 Python hashlib.md5，32 字符 hex
     --> signType = 0
```

## 项目文件

```
JTT/
  unidbg/unidbg-android/
    target/apse-signer.jar          -- fat JAR（QM 生成器）
    src/test/java/com/apse/APSETest.java  -- unidbg 主类
    src/test/resources/
      antfans.apk                   -- 鲸探 APK（151MB）
      libAPSE_8.0.0.so              -- 安全 SDK SO
      antfans_data/                 -- sc_edge 等设备文件
  sms_sender/
    apse_signer.py                  -- Python 封装（JAR 进程 + Sign 算法）
    send_sms_jar.py                 -- 端到端发送（纯离线）
    blast.py                        -- 高并发批量发送
    sign_server.py                  -- Frida RPC 签名服务（旧方案，备用）
```

## 使用方式

```python
from apse_signer import APSESigner

signer = APSESigner()  # 自动找 JAR
signer.start()

qm = signer.get_qm(sha_hex)           # unidbg QM
sign = APSESigner.compute_sign(sc)     # 纯 Python Sign

signer.close()
```

```bash
# 直接发送
python send_sms_jar.py
```

## JAR 运行方式

```bash
# stdin/stdout 协议
echo "icYRH1LO1lGJhYj2KSm9RR5r|<sha256_hex>|1" | \
  java --enable-native-access=ALL-UNNAMED \
    -Dapse.resources=<资源目录> \
    -jar apse-signer.jar
# 输出: QM_OK|<base64_string>
```

## QM 逆向过程

### 调用链
```
APSign.getColorInfo(bizToken, sha, {mode:"1"})
  --> SignManager.getColorInfo(0, bizToken, sha, modeMap)
  --> SignJNIBridge.getColorInfo(0, bizToken, sha, 1)  [native]
  --> libAPSE_8.0.0.so+0x5c7f0 (sub_5C7F0)
  --> sub_6E1A4 --> sub_6B874 (自定义流密码 VM)
```

### unidbg 实现关键点
1. **JNI_OnLoad**：注册所有 native 方法（SignJNIBridge, EdgeNativeBridge 等）
2. **initSI**：在 hook 安装前调用（SO 完整性校验），返回 0x2D600274
3. **Dobby hook**：安装在 initSI 之后
   - `sub_1B73F0`：hash table NULL guard
   - `__system_property_get`：返回设备属性
   - `gettimeofday`：固定时间
4. **initColorInfo(0, bizToken)**：返回 "9404de03"
5. **getColorInfo(0, bizToken, sha, 1)**：返回 153 字符 base64 = QM

### 数据注入
- `apse_data_bss.bin`：从运行进程 dump 的 data+BSS 段
- Phase 1（data 段）：保留 ELF 加载器的值，不覆写
- Phase 2（BSS 段）：注入运行时数据，重定位 SO 指针
- JVM heap regions：4 个区域共 ~2MB，用于 SCP hash table

## Sign 逆向过程

### 算法确认
`Sign = MD5("d65e27ce3bd3e2eb83d8bb21e452eeb0" + "&" + signContent)`

密钥 `d65e27ce3bd3e2eb83d8bb21e452eeb0` 是 mPaaS 平台分配给该 APP 的 appSecret。

### 验证样本
```
MD5(secret + "&" + "A")  = a632070dc0a75a343ea4a9f36b951651 [OK]
MD5(secret + "&" + "B")  = 3f41af01d151432b6f29a9be142e5034 [OK]
MD5(secret + "&" + "AB") = 699ad3afb9386f05c662a64f3c0df017 [OK]
8/8 samples all match
```

### Sign 的 SCP 保护分析（备忘）
原生 Sign 经过 Edge SCP（安全计算平台）多层保护：
- `EdgeNativeBridge.scpInvokeEvent(-54889831, args)` 进入 SCP VM
- VM 分发器 SO+0xB4168 通过 hash table（SO+0x44E038）查找处理函数
- 处理函数 SO+0xB9DA4 是另一个 VM
- 调用链：sub_29BD54 -> sub_29BE58 -> loc_29C1D4（加密调度）
- GOT 函数表 off_432A08-off_432B20 全部 AES 加密
- 数据表 SO+0x4976E0 有乘法完整性校验
- 设备绑定 sc_edge 文件 + ClassLoader 动态加载

但最终算法等价于简单的 MD5(appSecret + "&" + content)。

### Java 层调用链（mode 2 = antGroup/landun）
```
SecurityUtil.signature(SignRequest)
  --> wc.a.d(signRequest)  [k()=2, landun mode]
  --> wc.a.a(context, signRequest)
  --> SecuritySignUtil.antSign(context, signRequest)
  --> ISafeSignatureModule.signString(97, appkey, content)
  --> SafeSignatureModule.signInternal(97, appkey, null, contentBytes, true)
  --> EdgeNativeBridge.scpInvokeEvent(-54889831, [Integer(97), String, null, byte[], Boolean])
```

### Java 层备选路径（mode 3 = client，纯 Java 可复现）
```
vc.a.k(signRequest)
  --> sign = MD5(appSec + "&" + content).hex()
  --> appSec 来自 manifest meta-data "mpaas_appsec"（本 APP 为空，从 SCP 内部获取）
```

## 技术发现

### APSE 8.0.0 架构
- 不走旧版 sgmain 的 doCommandNative
- SignJNIBridge（走 libAPSE_8.0.0.so）用于 QM
- EdgeNativeBridge（同一个 SO）用于 Sign，通过 SCP VM 执行
- initSI 做 SO 完整性校验，必须在 hook 之前调用

### Frida 发现
- Sign 不依赖 appkey 参数（空字符串和正确 appkey 结果相同）
- Sign 确定性（同输入同输出）
- Sign 不走 Java crypto API（Mac/MessageDigest hook 无输出）
- 所有计算在 native SCP VM 内完成

### unidbg QM vs Sign
- QM：unidbg 完美支持，SignJNIBridge.getColorInfo 直接调用
- Sign：SCP 初始化需要 ClassLoader、sc_edge 文件、加密函数表，unidbg 无法完整复现
- 但 Sign 算法等价于 MD5(appSecret + "&" + content)，无需 unidbg
