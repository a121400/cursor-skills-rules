---
name: shumei-captcha-reverse
description: 数美(Shumei/fengkongcloud)点选验证码纯协议逆向与绕过。DES多Key加密、register/fverify接口、已知明文反推DES Key、SunnyNet抓包对照、冰拓/ttshitu OCR对接。适用于使用数美验证码的APP或网站（特征：captcha1.fengkongcloud.cn、castatic.fengkongcloud.cn、shumei_captcha CSS类、organization参数）。
---

# 数美 (Shumei) 点选验证码纯协议逆向

## 适用场景

目标站点/APP使用数美验证码（特征：`captcha1.fengkongcloud.cn`、`castatic.fengkongcloud.cn`、`shumei_captcha` CSS类、URL含 `organization` 参数、SDK文件 `captcha-sdk*.min.js`）。

## 核心流程

```
1. register  -> 获取验证码图片、order(待点选文字)、rid、k、l
2. OCR识别   -> 得到点选坐标 [(x,y), ...]
3. fverify   -> DES加密坐标+行为数据提交验证 -> riskLevel=PASS
4. 业务接口  -> 携带 rid 完成登录/发送等操作
```

## 关键接口

### register (获取验证码)

```
GET https://captcha1.fengkongcloud.cn/ca/v1/register
  ?organization=<org_id>
  &model=select          // 点选类型
  &captchaUuid=<uuid>    // 时间戳+随机字符串
  &callback=sm_<ts>      // JSONP回调
  &appId=default&channel=default&sdkver=1.1.3&rversion=1.0.4&lang=zh-cn
  &data=%7B%7D
```

响应（JSONP包裹）:
```json
{
  "detail": {
    "bg": "/crb/select-set-.../image.jpg",  // 背景图路径
    "bg_width": 600, "bg_height": 300,       // 图片尺寸
    "order": ["字", "符", "顺", "序"],        // 待点选的文字
    "rid": "20260323...",                     // 验证ID
    "k": "base64==",                         // DES key种子(8字节)
    "l": 8                                   // key长度
  }
}
```

图片URL: `https://castatic.fengkongcloud.cn{detail.bg}`

### fverify (提交验证)

```
GET https://captcha1.fengkongcloud.cn/ca/v2/fverify?<encrypted_params>
```

成功响应: `{"riskLevel": "PASS"}`
失败响应: `{"riskLevel": "REJECT"}`

## DES加密方案

### 核心发现：每个参数使用不同的DES Key

fverify的参数分为**固定参数**和**动态参数**，每个参数用独立的DES Key加密（ECB模式，ZeroPadding）。

### DES Key反推方法（已知明文攻击）

**关键技巧**：通过SunnyNet抓取APP的真实fverify请求，观察到多个参数值在不同会话间完全相同（说明使用固定key加密固定明文）。

1. 从JS SDK中提取所有8字符hex字符串作为候选key
2. 猜测固定参数的明文（trueWidth=300, trueHeight=150, zh-cn等）
3. 用每个候选key尝试加密猜测明文，与抓包值对比
4. 匹配成功即确认该参数的key

```python
from Crypto.Cipher import DES
import base64

def des_encrypt(key_str, data_str):
    key = key_str.encode('utf-8')[:8].ljust(8, b'\0')
    data = data_str.encode('utf-8')
    pad_len = (8 - len(data) % 8) % 8
    data += b'\0' * pad_len
    cipher = DES.new(key, DES.MODE_ECB)
    return base64.b64encode(cipher.encrypt(data)).decode()

def des_decrypt(key_str, enc_b64):
    key = key_str.encode('utf-8')[:8].ljust(8, b'\0')
    cipher = DES.new(key, DES.MODE_ECB)
    return cipher.decrypt(base64.b64decode(enc_b64)).rstrip(b'\0').decode('utf-8')
```

### 完整参数-Key映射（连信APP organization=3li97Iz8MWYgTGzD1z6H）

固定参数（值跨会话不变，可直接复用加密后的值）:

| 参数 | DES Key | 明文 | 加密值 |
|------|---------|------|--------|
| bt | 2f1b6d76 | 150 | AIPm2h0N5C4= |
| ki | 0cedcaf6 | 300 | 2W+B8Dp97No= |
| ie | 93a38761 | default | R1kuycSws3s= |
| oy | 008e0555 | -1 | RnF0eqKWzLY= |
| zh | 07ca8026 | 1 | ZuRlrCPI/Ys= |
| jt | 40af034e | zh-cn | TxywSL4hZRc= |
| iy | 8df985ee | 11 | 4W6LDMQSMko= |
| hy | 50385fd0 | default | lLEFdltPcnA= |
| uf | 66a0c9e1 | 0 | jyjoWLi+92I= |

动态参数（每次验证不同）:

| 参数 | DES Key | 明文内容 |
|------|---------|----------|
| cr | 1fc71626 | 归一化坐标JSON `[[x_ratio,y_ratio,timestamp],...]` |
| cz | 42abc58a | 同cr，相同的坐标JSON |
| cd | 22275674 | 行为数据值（格式不固定） |

非加密参数: `organization`, `rid`, `captchaUuid`, `sdkver`, `rversion`, `protocol`, `act.os`, `ostype`, `callback`

### 坐标归一化

OCR返回的像素坐标需归一化到0-1范围:
```python
x_ratio = pixel_x / bg_width   # bg_width=600
y_ratio = pixel_y / bg_height  # bg_height=300
```

坐标格式: `[[x_ratio, y_ratio, timestamp_ms], ...]`

## OCR对接

### 冰拓 (bingtop) - captchaType 13152

```python
import base64, requests

resp = requests.post('http://www.bingtop.com/ocr/upload/', data={
    'username': USERNAME,
    'password': PASSWORD,
    'captchaType': 13152,
    'captchaData': base64.b64encode(img_bytes).decode(),
    'subCaptchaData': requests.utils.quote("请依次点选出'字、符、顺、序'"),
}, timeout=45)
# result['data']['recognition'] = "337,53|88,87|456,200|525,147"
coords = [tuple(map(int, c.split(','))) for c in result['data']['recognition'].split('|')]
```

- 直接传原图base64 + 文字描述(subCaptchaData需URL编码)
- 返回像素坐标，用 `|` 分隔
- HTTP端点稳定，HTTPS可能有SSL问题
- 首次请求偶尔返回"请稍后再试"，重试即可

### ttshitu - typeid 21/27

- typeid=27: 原图 + content参数(目标文字)，精确但工人不足时不可用
- typeid=21: 合成图(文字头部+验证码图)，y坐标需减去头部高度
- 合成图方案需PIL拼接，较繁琐，且依赖工人数量

## APP集成流程（两步验证码）

很多APP的验证码流程是两步:

1. **Step 1**: 业务请求(如send_sms)，不带验证码参数 -> 服务端返回特殊码(如`resultCode=1900`)触发验证码
2. **Step 2**: 解决验证码获得rid后，业务请求带上 `rid`+`modeType`+`diffTime` -> 成功

```python
# Step 1: 触发验证码
result1 = send_sms(phone, captcha_result=None)
assert result1['resultCode'] == 1900  # 触发验证码

# Step 2-4: 解决验证码
detail = register_captcha()
coords = solve_captcha_ocr(detail)
fverify_result = fverify(detail, coords)
assert fverify_result['riskLevel'] == 'PASS'

# Step 5: 带rid发送
result2 = send_sms(phone, captcha_result={
    'rid': detail['rid'], 'modeType': 'select', 'diffTime': 0
})
assert result2['resultCode'] == 202  # 成功
```

## SDK特征识别

| 特征 | 说明 |
|------|------|
| `captcha1.fengkongcloud.cn` | API域名 |
| `castatic.fengkongcloud.cn` | 静态资源域名 |
| `organization=` | 组织ID参数 |
| `model=select` | 点选类型 |
| `model=slide` | 滑块类型 |
| `shumei_captcha` | CSS类名前缀 |
| `__smCaptcha` | JS全局对象 |
| `protocol=205` | 协议版本 |

## 注意事项

- `k`字段是DES key种子(base64编码8字节)，但SDK中的key派生逻辑被混淆，实际DES key不是直接从k派生
- 不同organization可能使用不同的key集合，需要重新抓包分析
- 固定参数的加密值可能随SDK版本更新而变化
- `bt`参数的key `2f1b6d76` 是SDK中唯一硬编码可见的key，其余key分散在混淆代码中
- JS SDK中hex字符串(`'[0-9a-f]{8}'`)是候选DES key的主要来源
- `isJsFormat`反调试: SDK检测代码是否被格式化(换行数>2)，若检测到则切换不同key
