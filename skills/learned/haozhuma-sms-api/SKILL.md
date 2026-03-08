---
name: haozhuma-sms-api
description: 豪猪码(haozhuma)短信验证码平台 API 对接指南。当用户提到对接豪猪、豪猪码、haozhuma、接码平台、短信验证码平台对接时使用。包含完整的登录、取号、收码、释放、拉黑等接口文档。
---

# 豪猪码 SMS API 对接

## 服务器地址

| 协议 | 地址 | 
|------|------|
| HTTPS | api.haozhuma.com |
| HTTPS | api.haozhuyun.com |

选择连通性好的线路，建议提供输入框让用户可自行填写服务器地址。

## 对接流程

### 标准流程
1. 调用登录接口获取 token（只需一次，token 固定不变）
2. 调用获取号码接口取号
3. 用取到的号码去目标平台触发发送验证码
4. 每 15 秒轮询获取验证码接口，最多等 3 分钟
5. 收到验证码后使用，或超时则释放/拉黑号码

### 复用号码流程
1. 调用指定号码接口占用号码
2. 调用获取验证码接口读取短信

### 异常处理
- 超过 3 分钟未收到验证码 -> 调用拉黑接口防止再次分配
- 号码不符合要求 -> 调用释放接口

## API 接口

所有接口均支持 GET/POST，基础 URL: `https://{服务器地址}/sms/`

### 1. 登录（获取 token）

```
GET /sms/?api=login&user={用户名}&pass={密码}
```

| 参数 | 必选 | 说明 |
|------|------|------|
| user | 是 | 用户名 |
| pass | 是 | 密码 |

返回: `{"msg":"success","code":0,"token":"xxx"}`

token 为固定值，修改密码后才会变。**不要每次取号都调用登录接口。**

### 2. 获取账户信息

```
GET /sms/?api=getSummary&token={令牌}
```

返回: `{"code":0,"money":"36.00","num":50}` (余额 + 最大区号数量)

### 3. 获取号码

```
GET /sms/?api=getPhone&token={令牌}&sid={项目ID}
```

| 参数 | 必选 | 说明 |
|------|------|------|
| token | 是 | 令牌 |
| sid | 是 | 项目ID |
| isp | 否 | 运营商代码（见下方代码表） |
| Province | 否 | 省份代码（如44=广东） |
| ascription | 否 | 1=虚拟卡, 2=实卡 |
| paragraph | 否 | 只取号段，多选用 `\|` 分隔 |
| exclude | 否 | 排除号段，多选用 `\|` 分隔 |
| uid | 否 | 只取指定对接码的号码 |
| author | 否 | 开发者账号（分成50%） |

返回: `{"code":"0","phone":"手机号","sid":"1000","sp":"移动","phone_gsd":"广东",...}`

### 4. 指定号码（复用场景）

```
GET /sms/?api=getPhone&token={令牌}&sid={项目ID}&phone={号码}
```

与获取号码接口相同 URL，多传 phone 参数即可占用指定号码。

### 5. 获取验证码

```
GET /sms/?api=getMessage&token={令牌}&sid={项目ID}&phone={号码}
```

返回: `{"code":"0","sms":"完整短信内容","yzm":"123456"}`

- `sms`: 完整短信文本
- `yzm`: 系统识别的数字验证码

### 6. 释放号码

释放单个:
```
GET /sms/?api=cancelRecv&token={令牌}&sid={项目ID}&phone={号码}
```

释放全部:
```
GET /sms/?api=cancelAllRecv&token={令牌}
```

### 7. 拉黑号码

```
GET /sms/?api=addBlacklist&token={令牌}&sid={项目ID}&phone={号码}
```

拉黑后该号码不再分配给你。

## 状态码规则

- `code=0` 或 `code=200` 表示成功
- `code=-1` 或其他值表示失败
- `msg` 字段包含中文描述

## 运营商代码表

| 运营商 | isp值 |
|--------|-------|
| 中国移动 | 1 |
| 中国联通 | 5 |
| 中国电信 | 9 |
| 中国广电 | 14 |
| 虚拟运营商 | 16 |

详细子类: 移动数据卡=2, 移动物联网=3, 移动虚商=4, 联通数据卡=6, 联通物联网=7, 联通虚商=8, 电信卫星=10, 电信数据卡=11, 电信物联网=12, 电信虚商=13

## 省份代码表

北京=11, 天津=12, 河北=13, 山西=14, 内蒙古=15, 辽宁=21, 吉林=22, 黑龙江=23, 上海=31, 江苏=32, 浙江=33, 安徽=34, 福建=35, 江西=36, 山东=37, 河南=41, 湖北=42, 湖南=43, 广东=44, 广西=45, 海南=46, 重庆=50, 四川=51, 贵州=52, 云南=53, 西藏=54, 陕西=61, 甘肃=62, 青海=63, 宁夏=64, 新疆=65

## Python 对接模板

```python
import requests
import time

class HaoZhuMa:
    def __init__(self, server, user, password):
        self.base = f"https://{server}/sms/"
        self.token = None
        self.login(user, password)

    def _req(self, params):
        resp = requests.get(self.base, params=params).json()
        if str(resp.get("code")) not in ("0", "200"):
            raise Exception(f"API错误: {resp.get('msg')}")
        return resp

    def login(self, user, password):
        resp = self._req({"api": "login", "user": user, "pass": password})
        self.token = resp["token"]

    def get_phone(self, sid, **kwargs):
        params = {"api": "getPhone", "token": self.token, "sid": sid, **kwargs}
        resp = self._req(params)
        return resp["phone"]

    def get_message(self, sid, phone, timeout=180, interval=15):
        """轮询获取验证码，默认最多等3分钟，每15秒查一次"""
        start = time.time()
        while time.time() - start < timeout:
            try:
                resp = self._req({
                    "api": "getMessage", "token": self.token,
                    "sid": sid, "phone": phone
                })
                return resp.get("yzm"), resp.get("sms")
            except Exception:
                time.sleep(interval)
        return None, None

    def release(self, sid, phone):
        self._req({"api": "cancelRecv", "token": self.token, "sid": sid, "phone": phone})

    def release_all(self):
        self._req({"api": "cancelAllRecv", "token": self.token})

    def blacklist(self, sid, phone):
        self._req({"api": "addBlacklist", "token": self.token, "sid": sid, "phone": phone})

    def assign_phone(self, sid, phone):
        """指定号码（复用场景）"""
        params = {"api": "getPhone", "token": self.token, "sid": sid, "phone": phone}
        return self._req(params)
```

## 注意事项

- 项目名字（sid）选错是收不到码的最常见原因，先用自己手机收一条看短信签名（如 `【4399】` 则项目名为 4399）
- token 获取一次即可，不要重复调用登录接口
- 建议提供服务器地址输入框，方便后续切换线路
