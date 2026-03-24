---
name: huaxiaozhu-wsg-sign
description: 花小猪打车(huaxiaozhu/hongyibo)登录协议逆向完整方案。wsgsig(dd06)+wsgenv 通过 unidbg 离线模拟 libdidiwsg.so 生成，cell_encrypted 为 RSA-1024/PKCS1 加密。已实现发送验证码(codeMT)+验证码登录(signInByCode)全流程。适用于滴滴系 WSG SDK v5.67.5 的 APP。
---

# 花小猪打车 WSG 签名 - 完整逆向方案

## 最终方案（unidbg 离线，无需 Frida/设备）

- **wsgsig**: unidbg 加载 `libdidiwsg.so` (arm64)，调用 `nativeSig` 生成 DD06 版签名
- **wsgenv**: unidbg 调用 `nativeCollect` 生成环境加密数据
- **cell_encrypted**: Python RSA-1024/PKCS1Padding 加密手机号
- **secdd-authentication**: 时间戳初始化，响应头更新
- **secdd-challenge**: 固定格式 `1,包名|版本||||0||`

## 签名链路

```
请求签名流程:
1. cell_encrypted = RSA_PKCS1(phone, rsa_public_key.pem)
2. body = "q=" + urlencode(json_params)
3. wsgenv = nativeCollect(host + path)  -- unidbg libdidiwsg.so
4. wsgsig = nativeSig(ctx, timestamp, phone, bodyBytes)  -- unidbg libdidiwsg.so
5. url = baseUrl + "?wsgenv=" + urlencode(wsgenv)
6. headers["wsgsig"] = wsgsig
```

## API 接口

### 基础信息
- **域名**: `epassport.hongyibo.com.cn`
- **APP**: `com.huaxiaozhu.rider` v1.13.8 (花小猪打车)
- **SDK**: WSG v5.67.5 (`libdidiwsg.so`, 3.24MB, arm64-v8a)

### 登录流程 (3步)

1. **gatekeeper** (可选前置校验)
   - POST `/passport/login/v5/gatekeeper?wsgenv=...`
   - 返回 `sec_session_id`

2. **codeMT** (发送验证码)
   - POST `/passport/login/v5/codeMT?wsgenv=...`
   - body: `q={"code_type":0,"cell_encrypted":"...","appid":130000,...}`
   - 返回 `errno:0` + `sec_session_id`

3. **signInByCode** (验证码登录)
   - POST `/passport/login/v5/signInByCode?wsgenv=...`
   - body: `q={"code":"验证码","code_type":1,"cell_encrypted":"...",...}`
   - 返回 `ticket` + `uid` + `cell`

### 请求头
```
User-Agent: com.huaxiaozhu.rider/1.13.8 Rabbit/1.4.12 Carrot/1.5.18
Content-Type: application/x-www-form-urlencoded; charset=utf-8
Productid: 430
CityId: 3
TripCountry: CN
wsgsig: dd06-...
secdd-authentication: <时间戳或服务端token>
secdd-challenge: 1,com.huaxiaozhu.rider|1.0.30||||0||
didi-header-rid: <UUID格式>
didi-header-omgid: <设备ID>
didi-header-ssuuid: <设备UUID>
```

## JADX 分析的核心类

| 类名 | 作用 |
|------|------|
| `com.didi.unifylogin.utils.RsaEncryptUtil` | 手机号 RSA 加密 (assets/rsa_public_key.pem) |
| `com.didi.security.wireless.adapter.SignInterceptor` | wsgsig/wsgenv 注入拦截器 |
| `com.didi.security.wireless.SecurityManager` | prepareSign + doSign 签名入口 |
| `com.didi.security.wireless.SecurityLib` | native 方法声明，加载 libdidiwsg.so |
| `com.didi.security.wireless.adapter.SecurityWrapper` | wsgenv 生成包装 |
| `com.didi.security.wireless.adapter.AuthInterceptor` | secdd-authentication 管理 |
| `com.didichuxing.security.challenge.DiChallenge` | secdd-challenge 挑战应答 |

## RSA 公钥

```
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC3yWkNvyIYZMLLm4BJdt7DaD/3
kxPXkjuvPcsd8aVeoRb4RIFEUZhXCbppEuhGAAgJoaZtMaFEn9pSByQ8V7AaOIZT
qDlSus8R1yXOMsotYG7bTgLbaPMB1wGgrn95woNZzZP9tYZ84oBi8Nm5pofEhZ/W
ImT1HOLVP5EtGG6lbwIDAQAB
```
- 算法: RSA-1024 / PKCS1Padding
- 来源: APK `assets/rsa_public_key.pem`

## unidbg 补环境清单 (20+ JNI 方法)

### 必补方法
| 方法 | 返回值 | 说明 |
|------|--------|------|
| `SecurityLib.safetyApolloGet(ctx, key)` | `""` | Apollo 配置 |
| `SecurityLib.getUserMode()` | `1` (COM) | 用户模式 |
| `SecurityLib.ApolloGetToggle(ctx, key, def)` | `false` | 功能开关 |
| `SecurityLib.checkAppForegroundState()` | `true` | 前台状态 |
| `StatUtils.isNetworkAvailable(ctx)` | `true` | 网络可用 |
| `DAQUtils.getUserId/Phone/Ticket()` | `""` | 用户信息 |
| `DAQUtils.getPackageName/AppVersionName(ctx)` | 包名/版本 | 应用信息 |
| `ClassLoader.loadClass(name)` | 对应 Class | 类加载 |
| `Throwable.getStackTrace()` | 伪造栈 | **反调试栈回溯** |
| `StackTraceElement.toString/getClassName/getMethodName()` | 栈信息 | 反调试校验 |
| `com.didi.security.utils.o.a()` | `DvmLong(0)` | 性能追踪 |

### 栈回溯反调试
nativeSig 调用时会通过 `new Throwable().getStackTrace()` 校验调用栈，必须返回合法的栈帧：
```java
StackTraceElement("com.didi.security.wireless.SecurityLib", "sign", "SecurityLib.java", 100),
StackTraceElement("com.didi.security.wireless.SecurityManager", "doSign", "SecurityManager.java", 200),
StackTraceElement("com.didi.security.wireless.adapter.SignInterceptor", "addSig", "SignInterceptor.java", 300)
```

### unidbg 源码修复
`ARM64SyscallHandler.java` 的 `prctl` 方法需添加 `PR_SET_VMA = 0x10` 支持，否则 nativeSig 会崩溃。

## wsgenv 内部加密流程 (看雪 thread-290055)

```
明文信息收集 → zlib 压缩 → AES-128-CBC 加密 → crc32 校验 → 自定义 Base64 编码
```
- AES Key/IV 从 SO 内部动态生成
- Base64 使用自定义字符表（非标准）

## wsgsig 生成流程 (Java 层可见部分)

```java
// 1. prepareSign: 提取 query 参数 + body hex，逆序排序拼接
String prepared = SecurityManager.prepareSign(url, bodyBytes);
// 2. doSign: 调用 native sign → nativeSig(ctx, timestamp, phone, prepared.getBytes())
String wsgsig = SecurityManager.doSign(prepared);
// 输出格式: dd06-{base64签名数据}
```

## 项目文件

| 文件 | 路径 | 说明 |
|------|------|------|
| unidbg Java 类 | `unidbg/unidbg-android/src/test/java/com/huaxiaozhu/HxzWsgSign.java` | 核心签名类 |
| libdidiwsg.so | `unidbg/unidbg-android/src/test/resources/huaxiaozhu/` | WSG SDK SO |
| base.apk | `unidbg/unidbg-android/src/test/resources/huaxiaozhu/` | 花小猪 APK |
| Python 测试脚本 | `test_hxz_login.py` | 完整登录流程 |
| RSA 公钥 | `huaxiaozhu_extract/rsa_public_key.pem` | 手机号加密公钥 |

## fat JAR 打包方法

### 打包步骤 (基于 maven-assembly-plugin)

1. 在 `unidbg-android/pom.xml` 的 `<build><plugins>` 中添加:
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>3.6.0</version>
    <configuration>
        <descriptors>
            <descriptor>assembly.xml</descriptor>
        </descriptors>
        <finalName>hxz-signer</finalName>
        <archive>
            <manifest>
                <mainClass>com.huaxiaozhu.HxzWsgSign</mainClass>
            </manifest>
        </archive>
    </configuration>
</plugin>
```

2. 创建 `unidbg-android/assembly.xml`:
```xml
<assembly xmlns="http://maven.apache.org/ASSEMBLY/2.1.0">
    <id>fat</id>
    <formats><format>jar</format></formats>
    <includeBaseDirectory>false</includeBaseDirectory>
    <dependencySets>
        <dependencySet><outputDirectory>/</outputDirectory><unpack>true</unpack><scope>runtime</scope></dependencySet>
        <dependencySet><outputDirectory>/</outputDirectory><unpack>true</unpack><scope>test</scope></dependencySet>
    </dependencySets>
    <fileSets>
        <fileSet><directory>${project.build.directory}/classes</directory><outputDirectory>/</outputDirectory></fileSet>
        <fileSet><directory>${project.build.directory}/test-classes</directory><outputDirectory>/</outputDirectory></fileSet>
    </fileSets>
</assembly>
```

3. 执行打包:
```bash
JAVA_HOME=/path/to/jdk8
mvn clean install -DskipTests -Dgpg.skip -Dmaven.javadoc.skip=true
mvn test-compile -pl unidbg-android -Dgpg.skip -Dmaven.test.skip=false -Dmaven.javadoc.skip=true
cd unidbg-android && mvn assembly:single -Dgpg.skip -Dmaven.javadoc.skip=true
# 输出: target/hxz-signer-fat.jar (~128MB)
```

### APK 路径处理
fat JAR 中的 APK 需要从 classpath 提取到临时文件:
```java
File apkFile = new File("unidbg-android/src/test/resources/huaxiaozhu/base.apk");
if (!apkFile.exists()) {
    InputStream is = HxzWsgSign.class.getClassLoader().getResourceAsStream("huaxiaozhu/base.apk");
    if (is != null) {
        apkFile = File.createTempFile("hxz_base", ".apk");
        apkFile.deleteOnExit();
        FileOutputStream fos = new FileOutputStream(apkFile);
        byte[] buf = new byte[8192];
        int n;
        while ((n = is.read(buf)) != -1) fos.write(buf, 0, n);
        fos.close(); is.close();
    }
}
```

### unidbg 源码修复
`ARM64SyscallHandler.java` 的 `prctl` 方法需添加 `PR_SET_VMA = 0x10` case 返回 0，否则 nativeSig 会崩溃。

## 调用方式

### Java JAR (推荐)
```bash
java -jar hxz-signer.jar both {hostPath} {body}
# 输出:
# RESULT_WSGENV:eV60A...
# RESULT_WSGSIG:dd06-...
```

### Python
```python
from test_hxz_login import send_code, login_with_code
send_code("手机号")         # 发送验证码
login_with_code("手机号", "验证码")  # 登录

from hxz_grab_coupon import query_sessions, HxzCouponGrabber
query_sessions(token)       # 查询场次+库存
HxzCouponGrabber(token).grab(sku_id, activity_id, price)  # 抢券
```

### 完整自动化
```bash
python hxz_rush.py login 手机号        # 登录
python hxz_rush.py query              # 查询
python hxz_rush.py rush               # 自动抢购
```

## 相关资料

- **看雪 thread-290055**: libdidiwsg.so nativeCollect 的 SO trace 分析（zlib+AES-128-CBC+crc32+base64）
- **tanglei.cc**: 滴滴 Web 端 dd03 版 wsgsig 的 Python 实现（MD5+自定义base64，仅 Web 端有效）
- **R5网 task/19179**: DD04 wsgsig 算法接口悬赏（¥1500，已完成）

## 随机设备指纹绕过频率限制

### 方案
每次调用 unidbg 时随机生成设备参数（ddfp/suuid/android_id），绕过服务端的设备级别频率限制。

### 实现
在 `HxzWsgSign.java` 构造函数中随机生成：
```java
this.deviceImei = md5Hex("imei_" + UUID.randomUUID()).toLowerCase();
this.deviceDdfp = deviceImei + md5Hex("buildprop_" + seed).toUpperCase();
this.deviceSuuid = md5Hex("1_2_" + deviceImei + "3_" + md5Hex("cpu_" + seed)).toUpperCase();
this.deviceAndroidId = md5Hex("aid_" + seed).substring(0, 16).toLowerCase();
```

在 `callStaticObjectMethod` 中使用：
```java
case "android/provider/Settings$Secure->getString(...)":
    return new StringObject(vm, deviceAndroidId);
```

JAR 输出设备参数供 Python 使用：
```
DEVICE_DDFP:xxx
DEVICE_SUUID:xxx
DEVICE_ANDROID_ID:xxx
```

### 关键发现
- 服务端**不会**跨请求校验设备一致性（发验证码和登录可以用不同的随机设备）
- 服务端只检查**单次请求内部** wsgenv 和请求体参数的一致性
- wsgenv 中的设备指纹由 unidbg JNI mock 的 android_id 等值生成
- 请求体中的 ddfp/suuid 只要格式正确即可（不需要和 wsgenv 完全一致）
- 每次用新设备指纹 = 新设备，不受频率限制

### 风控触发条件
- 同一 ddfp 短时间内发送太多次 → 53001 操作频繁
- 持续触发 53001 后 → 41002 图形验证码
- 图形验证码在模拟器上手动通过后恢复
- 使用随机设备完全绕过以上限制

## 适用范围

此方案适用于所有使用滴滴 WSG SDK v5.67.5 的 APP，包括但不限于：
- 花小猪打车 (com.huaxiaozhu.rider)
- 滴滴出行 (com.sdu.didi.psnger) — 需提取对应版本 SO
- 其他滴滴系 APP

核心逻辑相同，仅 appid、channel、UA 等参数不同。
