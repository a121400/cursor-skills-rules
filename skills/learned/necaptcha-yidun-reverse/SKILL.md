---
name: necaptcha-yidun-reverse
description: 网易易盾(NECaptcha/Yidun)验证码逆向与纯协议绕过。cb签名(eypt)、WaterMark行为数据、SM4加密、VDF POW、jsdom补环境。适用于使用网易易盾验证码的站点（网易充值中心、知乎、网易邮箱等）。
---

# 网易易盾 (NECaptcha/Yidun) 验证码逆向

## 适用场景
目标站点使用网易易盾验证码（特征：`initNECaptcha`、`webzjcaptcha.reg.163.com`、`.yidun` CSS类、`cstaticdun.126.net` 静态资源）。

## 逆向思路（按顺序）

### 1. 先搜论坛和GitHub
- 搜索 `NECaptcha 补环境`、`易盾纯协议`、`YidunCaptchaBreak`
- GitHub: `mdd666/yidun` - 纯Python易盾加密（B方法=cb, i方法=轨迹加密）
- GitHub: `aidencaptcha/YidunCaptchaBreak` - 完整纯协议方案

### 2. cb 签名生成
- 函数: `eypt(uuid(32))`
- 密钥: `14731382d816714fC59E47De5dA0C871D3F`（硬编码在core.js中）
- 自定义base64字符集: `i/x1XgU0z7k8N+lCpOnPrv6\qu2Gj9HRcwTYZ4bfSJBhaWstAeoMIEQ5mDdVFLKy`
- 填充符: `3`（不是`=`）
- cb与token无绑定关系，可独立生成

### 3. WaterMark SDK (wm.js)
- 初始化: `initCaptchaWatchman({productNumber: 'YD00000710348764'})`
- 在Node.js vm沙箱中可运行
- 关键修复: XHR的responseText必须保留JSONP原始格式（SDK内部用正则从responseText提取数据）
- URL修复: SDK构造的URL可能是 `https:://v3/d` 格式，需要修正为 `https://webzjac.reg.163.com/v3/d`
- 输出: acToken（170字符hex，前缀固定`9ca17ae2e6ff`）

### 4. NECaptcha check data (m/p/ext)
- `m` = `eypt(行为轨迹采样.join(':'))` - 24字符
- `p` = `eypt(store状态数据)` - 40字符
- `ext` = `eypt(xor_encode(data, counter+','+length))` - 28字符
- **服务端会解密验证行为数据合法性**，随机值会被拒绝
- `xor_encode(key, data)` = 自定义base64(逐字符XOR)

### 5. jsdom 补环境方案
- jsdom + canvas npm 包可以加载完整 NECaptcha SDK
- SDK 成功返回 validate token
- **但**: 后端二次验证时拒绝（validate中包含环境指纹）
- 需要补充: WebGL、Canvas指纹、字体检测、AudioContext等

### 6. 服务端验证参数关系
- `cb` 和 `token` **无绑定**（替换cb后仍 error=0）
- `data` 和 `token` **有绑定**（替换data后 error=100）
- `validate` 通过 `capkey`(=capId) 在后端验证

## URS 登录加密（ecard.163.com等网易站点）
- **SM4-ECB** 加密，密钥 `BC60B8B9E4FFEFFA219E5AD77F11F9E2`
- 密钥在 `pp_index_*.js` 中硬编码为 `_sm4pubkey`
- 所有请求体: `{"encParams": "hex_sm4_encrypted_json"}`
- PKID/Product 在登录页面的 JS 配置中

## VDF POW
- `x = (x * x) % mod`，迭代 `t` 次（通常200000）
- 输入: `mod`(hex), `t`, `puzzle`(base64)
- 输出: `{puzzle, spendTime, runTimes, sid, args: {x, t, sign}}`

## 关键教训
1. **先搜后分析** - 论坛/GitHub 搜索可能直接有现成方案
2. **NECaptcha 多层防护**: cb签名 → WM行为上报 → check行为数据 → 后端validate二次验证
3. **jsdom 补环境有限制** - 缺少 WebGL/Canvas 指纹会被后端检测
4. **WM SDK 的 XHR 响应格式** - 必须保留 JSONP 原始格式，SDK 用正则解析
5. **param check error** 可能不是参数格式问题，而是数据内容验证失败
