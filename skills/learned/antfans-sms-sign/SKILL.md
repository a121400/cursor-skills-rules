---
name: antfans-sms-sign
description: 鲸探(antfans)完整协议逆向方案。短信签名(QM/Sign)通过 unidbg 离线生成，下单协议通过 Xposed 中继模块获取动态设备参数。已实现 C# WinForms 多账号抢购工具(AntfansHelper)、Xposed TCP 签名模块、中继服务器心跳保活、curl_cffi TLS 绕过。当用户提到鲸探/antfans/下单/抢购/QM/Sign/APSE/AntfansHelper 时使用。
---

# 鲸探(antfans) 完整协议逆向方案

## 方案总览

| 场景 | 方案 | 关键参数 |
|------|------|----------|
| 短信签名 | unidbg 离线（apse-signer.jar） | QM + Sign |
| 下单协议 | **Xposed 中继模块**(推荐) 或 Frida attach | QM + Sign + x-device-color + x-device-token + x-device-ap-token |
| 多账号抢购 | C# WinForms AntfansHelper | 中继签名 + SmartThreadPool + curl_cffi TLS 绕过 |
| TLS WAF 绕过 | curl_cffi Python subprocess | mgs-biz 域 405 -> chrome120 impersonate |

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

## 三、C# WinForms 多账号抢购工具 (AntfansHelper)

### 项目结构
```
antfans_login/AntfansHelper/AntfansHelper/
  MainForm.cs / MainForm.Designer.cs  -- 主界面(账号管理/查询/下单/定时)
  Models/
    AccountInfo.cs                     -- 账号模型, ParseImportLine() 解析 "手机号----sessionId=xxx----userId"
    DeviceInfo.cs                      -- 设备指纹模型
  Services/
    AntfansApi.cs                      -- API 封装(商品查询/藏品下单/抽卡下单), curl_cffi TLS 绕过
    ApseSigner.cs                      -- 本地 JAR 签名器(QM, 短信场景)
    RelaySigner.cs                     -- 中继签名客户端(503 自动重试 3 次)
    CryptoHelper.cs                    -- Sign/SHA256/C10to64/Hx
    DeviceGenerator.cs                 -- 设备指纹生成
    ProxyService.cs                    -- 熊猫代理池(ConcurrentQueue, 用完即弃)
  http_helper.py                       -- curl_cffi TLS 指纹绕过(chrome120/safari17_0/chrome131_android)
```

### 关键功能
- **账号持久化**: `accounts.txt`, 格式 `手机号----sessionId=xxx----userId`
- **查询**: 商品 Feed 分页 + 抽卡 queryDrawCardMainPage 翻页(BOTH->RIGHT, rightHasNext)
- **下单**: 藏品 nftOrderCreate + 抽卡 createPlanOrder
- **TLS 绕过**: mgs-biz 域被 WAF 拦截(.NET HttpWebRequest 返回 405), 通过 Python subprocess 调用 curl_cffi 绕过
- **签名器随 EXE 启动**: MainForm 构造函数中 StartApseSignerAsync() 后台启动 JAR
- **下单结果**: orderInfo 保存到 `pay_links/{phone}_{orderNo}.txt`
- **编译**: 需要 VS2022 MSBuild (ToolsVersion 15.0), 不能用 .NET Framework 4.0 的 MSBuild

### 抽卡协议 (数字抽赏)

| 操作 | Operation-Type | 关键参数 |
|------|---------------|----------|
| 查询抽赏主页 | queryDrawCardMainPage | direction(BOTH/RIGHT), projectId |
| 查询计划阶段 | queryPlanStage | outRelationId, productCode=JING_TAN_CARD |
| 创建计划订单 | createPlanOrder | outRelationId, productCode=JING_TAN_CARD, quantity=1 |

翻页逻辑: 首次 direction=BOTH, 然后循环检查 rightHasNext, 用最后一个 projectId 作 anchor 发 direction=RIGHT

## 四、Xposed 签名模块 (xposed-signer)

### 项目结构
```
xposed-signer/
  app/src/main/java/com/antfans/xposed/
    SignerModule.java                  -- Xposed 入口, hook antfans 进程
    SignerRelayClient.java             -- TCP 长连接到中继服务器
    SignerSocketServer.java            -- 本地 Socket 服务(备用)
  build.bat                            -- 手动编译脚本(javac+d8+aapt2+apksigner)
```

### 关键设计
- **包名**: `com.antfans.xposed.signer`
- **框架**: Vector (LSPosed 分支, v2.0), Zygisk 模式
- **连接**: TCP 长连接到 `42.193.177.63:443`, JSON Lines 协议
- **心跳**: 收到 `{"cmd":"ping"}` 回复 `{"type":"pong"}`, 防断连
- **自动重连**: `connectLoop()` 断开后 5 秒自动重连
- **缓存**: color/deviceToken/apToken 缓存 5 分钟(CACHE_TTL=300000)
- **预签名**: 支持 prepare 命令批量预签存入服务端缓存池
- **编译安装**: `build.bat` 编译 -> `adb install` -> 需在 LSPosed 中激活 -> 重启鲸探生效
- **签名不兼容**: 重新编译后签名变更需 `adb uninstall` 旧版再 `adb install`

### APSE API 调用 (在 antfans 进程内)
```java
// QM
APSign.getColorInfo(bizToken, sha256hex, {mode:"1"})
// x-device-color
APDeviceColor.getColorLabel()
// x-device-token
APDID.getTokenResult("zorro_antfans").apdidToken
// x-device-ap-token
APDID.getTokenResult("antfans_android").apdidToken
```

## 五、中继服务器 (signer_relay_server.py)

### 架构
```
手机(Xposed) ---TCP:443---> 中继服务器(42.193.177.63) <---HTTP:5052--- PC(C#/Python)
                                      |
                              nginx 反代 /signer -> :5052
```

### 断连修复 (关键)
原始问题: `client.settimeout(120)` 导致 2 分钟无活动即断连

修复方案:
1. **TCP keepalive**: `settimeout(None)` + `SO_KEEPALIVE` + `TCP_KEEPIDLE=30/KEEPINTVL=10/KEEPCNT=6`
2. **应用层心跳**: 每 30 秒发 `{"cmd":"ping"}`, 手机回 `{"type":"pong"}`
3. **C# 客户端重试**: GetAll 遇 503 自动重试 3 次, 间隔 2s/4s/6s

### HTTP API
| 端点 | 方法 | 说明 |
|------|------|------|
| /sign | POST | `{"method":"getAll","sha":"..."}` -> QM+color+tokens |
| /status | GET | `{"phone_connected":true/false}` |
| /prepare | POST | 预签名池 `{"itemId":"...","userId":"...","poolSize":10}` |
| /cached_sign | POST | 从缓存池取预签结果 |
| /cache_status | GET | 缓存池状态 |

### 部署
```bash
scp signer_relay_server.py root@42.193.177.63:/opt/signer_relay/
ssh root@42.193.177.63 "kill <old_pid>; cd /opt/signer_relay && nohup python3 signer_relay_server.py --port 5052 &"
```

## 六、项目文件索引

```
JTT/
  antfans_login/
    AntfansHelper/                     -- C# WinForms 多账号抢购工具
    login_and_extract.py               -- 登录+提取设备参数
    order_demo_relay.py                -- Python 下单参考实现(548行)
    draw_card_order.py                 -- Python 抽卡下单参考(422行)
    account_data.json                  -- 账号配置
    deps/
      relay_signer.py                  -- Python 中继签名客户端
      apse_signer.py                   -- Python JAR 签名封装
  xposed-signer/                       -- Xposed 签名模块源码
  unidbg/unidbg-android/
    target/apse-signer.jar             -- QM 生成器(unidbg fat JAR)
  antfans_rush/
    signer_relay_server.py             -- 中继服务器(部署到 42.193.177.63)
    rush_buy.py                        -- 抢购主脚本
    scripts/
      frida_bypass_detect.js           -- Frida 反检测绕过
```

## 七、Sign 算法

`Sign = MD5("d65e27ce3bd3e2eb83d8bb21e452eeb0" + "&" + signContent)`

密钥是 mPaaS appSecret，纯 Python `hashlib.md5` 计算。signType=0。

## 八、unidbg QM（离线方案，短信场景用）

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

## 九、关键逆向发现

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

### QM 三条生成路径

| 路径 | 调用位置 | sub_681BC | sub_6B874 type | SCP block |
|------|----------|-----------|----------------|-----------|
| A | 0x6E56C | 有(用unk_4A0CA0) | 0x210 | 从 SCP key |
| B（真机实际走） | 0x6FCD4 | 无 | 0x4F0 | 从 sub_6F578 |
| C | 0x6B06C | 有(其他数据) | 0x170 | 可能 |

### SCP 门控检查（已证明不影响 QM）
- 位置：`SO+0x5C934`
- 逻辑：`(dword_44099C ^ dword_4408EC) == 0x167EB108`
- 真机上此检查 **FAIL**（`0x167EB107 ≠ 0x167EB108`，差1），但 QM 仍然 193 字符
- 说明 SCP block 在 Bridge 路径内部生成，不依赖此门控

### IDA 关键地址速查
- `0x5C7F0` — getColorInfo 入口
- `0x5C91C` — 外层 VM 分发器
- `0x5C934` — SCP 门控检查
- `0x5CD1C` — NewStringUTF（QM 输出点）
- `0x6E1A4` — Bridge VM 入口
- `0x6E540` — SCP 插入点（路径 A）
- `0x6F578` — SCP 生成 VM（路径 B，真机走）
- `0x681BC` — SCP key 计算函数
- `0x6B874` — QM 组装函数（Crypto VM）
- `0x1B96A0` — 链表遍历（vtable 循环，需清零 qword_49F0C0）
- `unk_4A0CA0` — SCP key data（运行时全零，非预存值）

### 内存 Dump 技巧（root 真机）
```bash
# /proc/PID/mem 完整 dump（需 root）
adb shell "su -c cat /proc/<PID>/maps" > maps.txt
# 按 segment 逐段 dump

# Frida warm dump（SCP 初始化后）
# 1. attach → 调用 getColorInfo 激活 SCP
# 2. 立即 dump SO data+BSS + heap 页
# 3. 单次 RPC 内完成，避免反检测

# MTE 指针特征
# 高字节 0xA0-0xBF → MTE-tagged heap 指针
# 高字节 0x00 + 地址 >= 0x7000000000 → 47-bit heap VA
# 其他大值 → config data（保留）
```

### unidbg SCP 失败原因（完整实验记录）

**已尝试的 10+ 种方案**：

| 方案 | 结果 | 原因 |
|------|------|------|
| Phase1 + ExtendedBSS 注入 | 挂死 | BSS 残留指针导致链表死循环 |
| SCP Key Only 注入 | 挂死 | scpInitialize 内部 VM 死循环 |
| SCP 门控 patch（SO+0x5C934 强制通过） | VM 崩溃 | 缺少完整 heap 上下文 |
| 二分法定位挂死区 + 跳过 | 仍挂 | 更多隐藏挂死范围 |
| Frida SCP 字节 + unidbg crypto 拼接 | 30831 | 服务器验证原子性 |
| 92/500 个 heap 页 dump | 不够 | SCP VM 访问内存远超 dump |
| 完整进程内存 dump（597MB, 2030段） | Unicorn 溢出→合并后挂死 | MTE 多级指针链重定位失败 |
| TBI handler（UC_HOOK_MEM_*_UNMAPPED） | 性能过慢 | 每次触发 Java callback |
| MTE strip + SO mirror（设备基址镜像） | VM 操作码偏离 | config data 被误杀/链表环 |
| 精确清零链表头 + config 白名单 | 1亿指令仍循环 | Bridge VM 分支值不匹配（蝴蝶效应） |
| warm text+data 全量注入 | 链表死循环 | raw heap 指针保留 |

**技术关键点**：
- SO 有自解密代码段 → 需 Frida dump warm text（4.1MB 解密后代码）
- MTE（Memory Tagging Extension）指针高字节 0xA0-0xBF → Unicorn 不支持 TBI
- Unicorn 映射上限 4096 → 2030 段需合并到 701 个
- VM config data（加密密钥/操作码表）与 heap 指针在值特征上难区分
- data section 0x440000-0x442400 区域是 VM 配置表，不可清零
- **蝴蝶效应**：VM 初始状态差 1 bit → 后续所有操作码转换偏离

**结论**：APSE 8.0.0 的 SCP VM 需要完整 Android 运行时，unidbg 无法模拟

### 替代离线方案

| 方案 | 可行性 | 说明 |
|------|--------|------|
| **Frida RPC**（当前在用） | 已验证 | attach→call→detach <2秒，bizStatusCode=10000 |
| **LSPosed Xposed 模块** | 推荐 | 比 Frida 更隐蔽，持久化 TCP 签名服务 |
| app_process 设备签名 | 可能 | 需要编译 dex，设备上运行 |
| 完整 SCP VM 逆向 | 极难 | 需手工分析 3 层 VM 操作码语义，耗时数周 |

### 风控体系
- 30831 = 操作频繁/设备风控，非频率限制
- 设备指纹三件套（color/device-token/ap-token）任一错误即触发
- QM 的 bizToken/mode 错误也触发 30831
- unidbg Bridge QM（151→193补丁）被服务器拒绝，SCP block 必须真实计算
- x-device-color 每次调用动态生成（非静态缓存）
- 代理 IP 质量影响：住宅代理仍可能被标记，直连优先
