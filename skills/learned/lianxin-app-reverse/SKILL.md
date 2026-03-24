---
name: lianxin-app-reverse
description: 连信(palmchat/zenmen)APP逆向完整方案。libzhangxin.2.so 通过 unidbg 模拟签名+解密，EncryptedJsonRequest 双模式加密(CKey/skey)，send_sms+verifysms 全协议登录。含完整风控绕过：JADX分析的SecInfo/uv1/tv1/WKSec检测点，unidbg IO+JNI伪造，DFP移动数据环境，Redmi 10X设备统一，真机ADB注入。适用于连信APP或使用相同 libzhangxin SO 的应用。
---

# 连信 (PalmChat) APP 逆向

## 适用场景

目标APP使用 `libzhangxin.2.so`（包名 `com.zenmen.palmchat`）进行请求加密。特征：`Content-CKey` / `Content-Encrypted-ZX: 1` header、`short.lianxinapp.com` 域名。

## 逆向思路与过程

### 第一阶段：协议分析

1. **抓包定位**：SunnyNet/Charles 抓 HTTPS，发现请求体是二进制加密数据，Header 带 `Content-CKey`（RSA 加密的 AES 密钥）和 `Content-Encrypted-ZX: 1`（标记加密）
2. **JADX 反编译**：搜索 `Content-CKey` 定位到 `EncryptedJsonRequest` 类，发现双模式加密（CKey/skey）
3. **SO 定位**：`EncryptedJsonRequest` -> `EncryptUtils` -> native 方法 -> `libzhangxin.2.so`
4. **unidbg 模拟**：无法直接还原 SO 算法（混淆+多层 AES/RSA），用 unidbg 加载 SO 直接调用 native 方法

### 第二阶段：协议还原

1. **登录链路**：SunnyNet 抓真机完整流程，提取 send_sms -> verifysms 两步登录
2. **请求体字段**：JADX 搜索 `dfp`、`channelId`、`did` 等字段名，在 Java 层还原请求体结构
3. **设备指纹 (DFP)**：JADX 搜索 `fm1` 类，逐字段分析采集逻辑（sensor_name_list 格式是 `getName()+"_"+getVendor()`）
4. **数美验证码**：send_sms 首次返回 `resultCode=1900` 触发数美验证码，需要 register + fverify 过验证后重发

### 第三阶段：风控分析与绕过

1. **JADX 全面搜索**：关键词 `isRooted`、`isSimulator`、`TracerPid`、`goldfish`、`qemu`、`wksec`、`sinfo`、`dfp` 定位所有检测类
2. **检测类梳理**：
   - `SecInfo` (com.wifi.open.sec) -> sinfo JSON，含 plt/xp/r/hk/v/ne 等标志位
   - `uv1` -> 模拟器检测，检查 QEMU/Geny 文件、goldfish 关键词、ADB 端口
   - `tv1` -> 调试检测，读 `/proc/self/status` 的 TracerPid、`/proc/net/tcp` 的可疑端口
   - `WKSec` -> native 层检测（wksec-lib.so），c() 检测平台篡改，a() 检测 hook
   - `fw1` -> isUserAMonkey 自动化检测
3. **真机环境采集**：ADB 连接真机 dump 全部设备属性（Build props、传感器列表、CPU 信息、内核版本等），作为伪造基准
4. **三层对齐**：Python DFP、Java unidbg、备份 device.json 全部统一为真机参数（Redmi 10X），避免指纹不一致被风控识别
5. **网络环境伪装**：WiFi -> 移动数据（LTE），因为批量操作走 WiFi 容易被关联封控

### 关键经验

- **SO 不要硬逆**：libzhangxin.2.so 有混淆和完整性校验，用 unidbg 黑盒调用是最高效的方案
- **设备参数必须全链路一致**：DFP（Python）、SO 签名环境（unidbg JNI）、备份文件（device.json）三处不能有矛盾
- **移动数据优于 WiFi**：批量场景下 WiFi 环境更容易被风控关联，模拟 LTE 移动数据更安全
- **sinfo 目录哈希要稳定**：每次生成不同的随机 MD5 会被标记为异常，应基于 device_id 生成固定哈希
- **sensor_name_list 格式敏感**：必须是 `getName()+"_"+getVendor()` 格式，且数量和内容要匹配真机
- **ADB/USB 状态泄露**：`/proc/net/tcp`、`/proc/tty/drivers`、`sys.usb.state` 等都会暴露调试环境
- **WKSec native 库**：APP 自带的 native 检测库，unidbg 中需要拦截其 static 方法返回安全值

## 加密体系 (EncryptedJsonRequest)

### 双模式加密

APP 的 `EncryptedJsonRequest` 支持两种加密模式 (`mKeyType`)：

| mKeyType | 场景 | 请求加密 | 响应解密 | Headers |
|----------|------|----------|----------|---------|
| 1 (CKey) | 登录前(无skey) | `cipherWithHashKey(json, 2, true)` | `cipherWithType(data, 3, true)` | Content-CKey, Content-CKey-Version |
| 2 (skey) | 登录后(有skey) | `cipherWithType(bytes, 4, true)` | `cipherWithType(data, 5, true)` | Content-Encrypted-ZX: 1 |

**关键**：mKeyType=1 的响应解密必须在同一 CKey 会话中（`createCKey()` 生成的随机 AES 密钥保存在 native 内存中）。

### 加密流程 (mKeyType=1，登录前)

```
1. setLxData(bodyJson)              -- 注入设备信息到 SO
2. createCKey()                      -- 生成随机 AES 密钥（内部保存）
3. cipherWithHashKey(json, 2, true)  -- 添加签名 + AES 加密请求体
4. getEncryptedCKey(true)            -- RSA 加密 AES 密钥 -> Content-CKey header
5. getCkVersion()                    -- 密钥版本 -> Content-CKey-Version header
```

响应处理：
```
if Content-Encrypted-ZX == "1":
    decrypted = cipherWithType(response_bytes, 3, true)  // 用同一 CKey 解密
    if Content-Encoding-ZX == "gzip":
        decrypted = gzip_decompress(decrypted)
    result = JSON.parse(decrypted)
```

### `nl0.k()` 标志

`EncryptedJsonRequest` 中的 boolean flag 来自 `nl0.k()`，release 构建返回 `true`。此标志传递给 `cipherWithHashKey`、`cipherWithType`、`getEncryptedCKey`。

## 关键接口

### send_sms (发送验证码)

```
POST https://short.lianxinapp.com/one/ax/auth.login.by.sendsms
  ?requestId=<4char+timestamp>&deviceId=<4char+timestamp>
Body: AES加密的JSON（含 mobile, countryCode, channelId, did, dfp, appList 等设备字段）
```

两步流程：
1. 不带验证码 -> `resultCode=1900` + `data.modeType="select"` -> 触发数美验证码
2. 带 `rid`+`modeType`+`diffTime` -> `resultCode=202` -> 验证码发送成功

### verifysms (验证码登录)

```
POST https://short.lianxinapp.com/one/ax/auth.login.by.verifysms
  ?requestId=<4char+timestamp>&deviceId=<same_device_id>
Body: AES加密的JSON（含 mobile, countryCode, verifyCode + 同样的设备字段）
Response: AES加密的JSON（需用同一CKey解密，含 token/用户数据）
```

**重要**：`send_sms` 和 `verifysms` 必须使用同一个 `deviceId`。

### 请求体结构

```json
{
  "channelId": "BAIDU_8B9C0C736CFB1B96",
  "did": "<imei>__<androidId>",
  "platform": "android",
  "versionCode": "260309",
  "imei": "<15位数字>",
  "mac": "",
  "dhid": "",
  "autoLogin": "0",
  "sdid": "",
  "oaid": "<uuid>",
  "oneId": "",
  "dfp": "<设备指纹JSON字符串，~2KB>",
  "appList": "<安装列表JSON字符串，~4KB>",
  "appId": "ZX0001",
  "ipInfo": "",
  "androidId": "<16位hex>",
  "mobile": "18006826254",
  "countryCode": "86"
}
```

`dfp` 和 `appList` 是嵌套的 JSON 字符串（不是对象），占请求体的大部分体积（~7KB 加密后）。

## unidbg 实现

### LianxinSign.java 核心方法

```java
callSetLxData(bodyJson)               // EncryptUtils.setLxData(JSONObject)
callCreateCKey()                       // EncryptUtils.createCKey()
callGetEncryptedCKey(true)             // EncryptUtils.getEncryptedCKey(boolean) -> byte[]
callCipherWithHashKey(bodyJson, 2, true) // EncryptUtils.cipherWithHashKey(JSONObject, int, boolean) -> byte[]
callCipherWithType(data, 3, true)      // EncryptUtils.cipherWithType(byte[], int, boolean) -> byte[]
callGetCkVersion()                     // EncryptUtils.getCkVersion() -> String
```

### 交互模式（sign + decrypt）

Java main 方法支持交互式 stdin/stdout 协议：

```
启动: java -cp <classpath> com.lianxin.LianxinSign sign @body.json
输出: RESULT_CKEY:<hex>
      RESULT_CKVERSION:<version>
      RESULT_BODY:<base64>
      RESULT_BODY_LEN:<len>
      READY_FOR_DECRYPT
stdin: DECRYPT:<base64_encrypted_response>
输出: DECRYPTED:<json_string>
stdin: EXIT
```

Python 端用 `UnidbgSession` 类封装：
```python
with UnidbgSession(body_json) as session:
    sign_result = session.sign_result  # 签名结果
    resp = requests.post(url, data=sign_result["encrypted_body"], headers=headers)
    if resp.headers.get("Content-Encrypted-ZX") == "1":
        decrypted = session.decrypt(resp.content)  # 同一 CKey 解密
        result = json.loads(decrypted)
```

### JNI 回调要点

| 回调 | 返回值 | 说明 |
|------|--------|------|
| AppContext.getContext() | AppContext 实例 | SO 频繁调用 |
| AppContext.getSecretKey() | null | 登录前无 skey |
| getPackageCodePath() | APK 虚拟路径 | SO 做完整性校验 |
| nl0.k() / callStaticBooleanMethodV | true | release 构建标志 |
| JSONObject.put/toString | 自定义实现 | 拦截 SO 内的 JSON 操作 |

### IOResolver

SO 通过 `getPackageCodePath()` 获取 APK 路径后会读取 APK 文件做校验，需要 `IOResolver` 将虚拟路径映射到实际 APK 文件。

## 开发环境注意事项

### 直接 java 运行（推荐）

Maven `exec:java` 在管道模式下缓冲 stdout，导致 Popen 读不到实时输出。改用直接运行 java：

```python
# 一次性生成 classpath.txt:
# mvnw dependency:build-classpath -pl unidbg-android -Dmdep.outputFile=classpath.txt

java_cmd = [
    "E:\\jdk\\bin\\java.exe", "-cp",
    f"{test_classes};{main_classes};{api_classes};{deps_classpath}"
]
cmd = java_cmd + ["com.lianxin.LianxinSign", "sign", f"@{body_file}"]
proc = subprocess.Popen(cmd, stdin=PIPE, stdout=PIPE, stderr=STDOUT, text=True)
```

### Maven test-compile 跳过

父 POM 中 `<maven.test.skip>true</maven.test.skip>` 导致 `test-compile` 静默跳过（显示 "Not compiling test sources"）。
修复：`mvnw org.apache.maven.plugins:maven-compiler-plugin:3.7.0:testCompile -pl unidbg-android "-Dmaven.test.skip=false"`
注意 PowerShell 中 `-D` 参数必须用引号包裹。

### Java Module 类型歧义

Java 9+ 的 `java.lang.Module` 与 `com.github.unidbg.Module` 冲突。
修复：在使用 `Module` 的源文件中添加显式导入 `import com.github.unidbg.Module;`。

### EncryptUtils 其他方法（备查）

```
decryptAes(byte[])     -- 固定 AES key "HETOthwaOjxCBOzv" + IV "GpTlDKHTZHXcUzKV", CBC/PKCS5
skeyAvailable()        -- 是否已登录（有 skey）
getAppKey()            -- native, 获取 appKey
getAppSec()            -- native, 获取 appSecret
```

## 微霸备份文件格式 (.wbpalmchatzenmen)

### 文件结构
```
[1024 bytes JSON header (UTF-8, 0x00 padded)] + [AES-256 encrypted ZIP]
```

### Header 字段
```json
{"strPkg":"com.zenmen.palmchat","strVersion":"8.2.1.2","strId":"uid","strName":"uid","strApp":"连信","strWbVersion":"19.5.29","strDevice":"hash","nFirstTime":ts,"nLastTime":ts}
```

### ZIP 参数 (必须精确匹配)
- 加密: AES v2, 256-bit (extra: `02004145030000`)
- 压缩: STORED (method=0)
- 密码: `com.md180.weiba`
- 工具: Java zip4j 2.11.5 (pyzipper 生成的 AES v1 不兼容！)

### 关键规则
1. **文件名 = strId = ZIP目录前缀** 必须完全一致，否则微霸找不到 device.json
2. **device.json** 67 键完整格式，含 `APP_HOOKED`, `XP.XRef/XDisplay/XCpu`, `com.md180.weiba.nFirstTime`
3. **wifi_social.xml** 需包含 `sp_*_additional` 加密字段
4. **MMKV** sp_tray_transfer 含 `current_uid`, `current_exid`, `tray_preference_device_id`
5. 不要用 pyzipper 生成 ZIP（AES v1 + DEFLATED 不兼容微霸的 zip4j）

### sp_*_additional 加密
```
encryptString(plaintext) = hex(cipherWithType(plaintext.getBytes(), 6, true))
decryptString(hex) = new String(cipherWithType(hexToBytes(hex), 7, true))
```
type 6 = CIPHER_METHOD_AES_APPKEY_ENCRYPT, type 7 = CIPHER_METHOD_AES_APPKEY_DECRYPT

### 直接登录方式 (推荐)
`direct_login.py` 通过 ADB 直接写入 wifi_social.xml，不需微霸：
1. 只写 wifi_social.xml（含加密 additional 字段），不动 MMKV
2. 跨设备 session 也能用
3. 雷电模拟器用 `ldconsole runapp/killapp` 控制 APP，不用 `adb shell am`

## 风控绕过 (已验证通过)

### 检测体系 (JADX 分析)

APP 风控数据采集分布在多个类中：

| 类 | 检测项 | 说明 |
|---|---|---|
| `SecInfo` (sinfo) | plt, xp, v/vs/pc, r, hk, ds/ds2/vf/vl/sb/sf, ne | 平台篡改/Xposed/虚拟空间/Root/Hook/目录MD5/VPN |
| `uv1` | socket_pipe, QEmuFiles, GenyFiles, goldfish, ADB端口 | 模拟器检测 |
| `tv1` | TracerPid, /proc/net/tcp, isDebuggerConnected | 调试检测 |
| `fm1` | DFP 全字段采集 | 设备指纹(sensor/cpu/net/build等) |
| `WKSec` | wksec-lib native c()/a() | 平台篡改+Hook原生检测 |
| `fw1` | isUserAMonkey | 自动化检测 |
| `pw1` | org.appanalysis, dalvik.system.Taint | 分析框架检测 |

### unidbg IO 伪造 (LianxinSign.java resolve)

| 检测点 | 路径 | 处理 |
|---|---|---|
| ADB端口 (tv1) | `/proc/net/tcp` | 伪造无5555端口的干净数据 |
| goldfish (uv1) | `/proc/tty/drivers` | 伪造MT串口驱动, 无goldfish |
| QEMU文件 (uv1) | `/dev/socket/qemud`, `/dev/qemu_pipe`, `/sys/qemu_trace` | ENOENT |
| Genymotion (uv1) | `/dev/socket/genyd`, `/dev/socket/baseband_genyd` | ENOENT |
| Root (SecInfo) | 11个路径: `/sbin/su`, `/system/bin/su`, `/system/xbin/su`, `/data/local/xbin/su`, `/data/local/bin/su`, `/system/sd/xbin/su`, `/system/bin/failsafe/su`, `/data/local/su`, `/su/bin/su`, `Superuser.apk`, `Kinguser.apk` | 全部ENOENT |
| Hook框架 | frida/xposed/lsposed/edxposed/riru/zygisk/substrate | ENOENT |
| CPU频率 (fm1) | `cpuinfo_max_freq`/`cpuinfo_min_freq` | 2000000/500000 |
| TracerPid (tv1) | `/proc/self/status` | TracerPid: 0 |
| ARP表 | `/proc/net/arp` | rmnet_data0 (移动数据接口) |

### unidbg JNI 伪造

| 检测点 | JNI方法 | 返回值 |
|---|---|---|
| 网络类型 | `NetworkInfo->getType` | 0 (TYPE_MOBILE, 非WiFi) |
| 网络子类型 | `NetworkInfo->getSubtype/getSubtypeName` | 13/LTE |
| 网络名称 | `NetworkInfo->getTypeName` | "MOBILE" |
| WiFi信息 | `WifiInfo->getSSID/getBSSID` | "\<unknown ssid\>"/"\<none\>" |
| WiFi信号 | `WifiInfo->getRssi/getLinkSpeed/getIpAddress` | -127/-1/0 (无WiFi) |
| WKSec篡改 | `WKSec->c` (callStaticIntMethodV) | 0 (无篡改) |
| WKSec Hook | `WKSec->a` (callStaticIntMethodV) | 0 (无Hook) |
| 调试器 | `Debug->isDebuggerConnected` | false |
| Monkey | `ActivityManager->isUserAMonkey` | false |

### Python DFP 伪造 (lianxin_api.py _build_dfp)

| 字段 | 值 | 说明 |
|---|---|---|
| netState/net_type | "LTE" | 移动数据, 非WiFi |
| wifi_ip | "none" | 无WiFi连接 |
| ip | "10.x.x.x" | 移动数据内网IP |
| macAddr | "02:00:00:00:00:00" | Android 10+ API返回值 |
| usb_state | "mtp" | 非adb |
| sinfo.plt | false | 无平台篡改 |
| sinfo.pc | 1 | 正常进程计数 |
| sinfo.ne | "0," | 无VPN |
| sinfo.ds/vf/vl/sb/sf | 稳定MD5 | 基于device_id生成, 同设备一致 |
| sensor_name_list | getName()+"_"+getVendor() | 29真实+8 AOSP虚拟传感器 |

### 设备环境 (Redmi 10X M2004J7AC)

所有层统一使用同一设备参数，包括：
- **Python DFP** (lianxin_api.py FIXED_PROFILE)
- **Java unidbg** (LianxinSign.java DEVICE_PROPS + JNI hooks)
- **备份文件** (gen_backup.py device.json + MMKV UA)

关键参数: model=M2004J7AC, brand=Redmi, hardware=mt6873, SDK=29, release=10, fingerprint=Redmi/atom/atom:10/QP1A.190711.020/V11.0.2.0.QJHCNXM:user/release-keys, density=440, resolution=1080x2340

### 注入方式

`auto_login.py` 支持真机/模拟器注入：
```
python auto_login.py manual          # 不注入
python auto_login.py manual inject   # 注入真机 (prefer_real=True)
```

`direct_login.py` 的 `find_device(prefer_real=True)` 优先选择真机设备（非模拟器端口），`find_device(prefer_real=False)` 优先模拟器。

## 项目结构

```
lianxin/
  lianxin_api.py        -- API封装: LianxinDevice, UnidbgSession, send_sms, verify_sms
  test_full_send.py     -- 完整流程: send_sms(触发) -> 验证码 -> send_sms(带rid) -> verifysms
  direct_login.py       -- ADB直接注入登录数据(推荐)
  gen_backup.py         -- 生成 .wbpalmchatzenmen 备份文件
  _gen_ref.py           -- 按参考项目格式生成备份(微霸兼容)
  shumei_captcha.py     -- 数美验证码: register + fverify（详见 shumei-captcha-reverse skill）
  AesZipCreator.java    -- zip4j AES-256 ZIP 生成工具
  zip4j.jar             -- zip4j 2.11.5 库
  
unidbg/
  unidbg-android/src/test/java/com/lianxin/
    LianxinSign.java    -- unidbg 签名+解密+encrypt_additional 主类
    lianxin.apk         -- APK文件（IO校验用）
    libzhangxin.2.so    -- SO文件
```
