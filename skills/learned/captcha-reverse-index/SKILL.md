---
name: captcha-reverse-index
description: 验证码逆向统一索引。遇到验证码(滑块/点选/WAF cookie)时读此 skill 按特征路由到子 skill。覆盖数美/易盾/腾讯防水墙/顶象/京东云/瑞数。当用户提到验证码、滑块、点选、captcha、WAF cookie 时使用。
---

# 验证码逆向索引

遇到验证码相关任务时，先根据下表特征判断类型，再读取对应子 skill 获取完整方案。

## 特征路由表

| 验证码类型 | 识别特征 | 子 Skill |
|-----------|----------|----------|
| **数美 (Shumei)** | `fengkongcloud.cn`、`castatic.fengkongcloud.cn`、`organization=`、`shumei_captcha` CSS、点选验证 | `shumei-captcha-reverse` |
| **网易易盾 (NECaptcha)** | `initNECaptcha`、`webzjcaptcha.reg.163.com`、`.yidun` CSS、`cstaticdun.126.net` | `necaptcha-yidun-reverse` |
| **腾讯防水墙 (TCaptcha)** | `cap_union_prehandle`、`cap_union_new_verify`、`turing.captcha.gtimg.com`、TDC.js | `tcaptcha-solver` |
| **顶象 (DingXiang)** | `greenseer.js`、`_dx.UA`、背景图乱序切割(32块x12px)、`/api/a` + `/api/v1` | `dingxiang-slider-captcha` |
| **京东云 (JCAP)** | `captcha-api-global.jdcloud.com`、`x-jdcloud-captcha-auth`、`jcap.js` WASM | `jdcloud-captcha-jcap` |
| **瑞数 (Ruishu)** | `$_ts` 变量、`yrXXX` cookie、HTTP 412 拦截、`jhsgvYT0` URL 参数、VMP JS | `ruishu-sdenv-bypass` |

## 通用工具

| 工具 | 用途 |
|------|------|
| ddddocr | 滑块缺口检测 (`slide_match`) |
| OpenCV | Canny 边缘+模板匹配（顶象用） |
| YOLO best.pt | 缺口物体检测（腾讯防水墙用） |
| 冰拓 (bingtop) | 点选 OCR (captchaType 13152) |
| ttshitu | 点选 OCR (typeid 21/27) |

## 未知验证码判断流程

1. **检查域名/URL**：从 Network 抓包看验证码请求域名
2. **检查 JS 加载**：搜索页面加载的验证码 SDK 文件名
3. **检查 CSS 类名**：inspect 验证码弹窗的 class 前缀
4. **无法匹配上表**：可能是自建验证码，按 `js-reverse-practical` 的通用逆向流程分析
