---
name: reverse-forum-search
description: 逆向工程时自动检索看雪/吾爱破解论坛案例：生成关键词 -> 搜索 -> 拉取详情 -> 汇总可复用的逆向思路。依赖逆向论坛搜索 MCP（reverse-forum-search）。
---

## 适用场景

- 逆向某目标（网站/JS、Android、小程序、PC）时，需快速在**看雪**或**吾爱破解**论坛找类似案例与分析思路。
- 已启用 `reverse-forum-search` MCP，工具：`reverse_forum_search`、`reverse_forum_thread_detail`；看雪回复解锁用 `kanxue_reverse_reply`。

## 论坛与工具

| 论坛     | MCP 参数   | 搜索/详情工具                     | 回复解锁     |
|----------|------------|-----------------------------------|--------------|
| 看雪     | forum=kanxue | reverse_forum_search / reverse_forum_thread_detail | kanxue_reverse_reply |
| 吾爱破解 | forum=52pojie | reverse_forum_search / reverse_forum_thread_detail | 不支持       |

## 触发方式

- **显式触发**：消息以 `【看雪搜索】`、`【吾爱搜索】` 或 `[逆向搜索]` 开头时，执行一次搜索；前两者可指定 forum（kanxue / 52pojie）。
- **隐式触发**：描述中同时出现 2 个以上要素时触发一次搜索：
  - 平台词：安卓 / APK / so / dex / frida / unidbg / Web / JS / 小程序 / Windows / exe / dll
  - 关键点：登录 / sign / token / AES / RSA / HMAC / 混淆 / 验证码 / 抓包 / hook / 加固 / 脱壳

## 执行流程（必须按顺序）

1. **抽取检索意图与论坛**
   - 从描述抽取平台与关键词；若无显式指定，默认 forum=kanxue；若出现「吾爱」「52pojie」可设 forum=52pojie。
   - 组合 1~3 条 query（短、信息密度高），例如：
     - Android：`安卓 逆向 frida 检测 绕过 登录 加解密 AES`
     - Web：`网站 登录 sign 参数 js 加密 混淆 逆向`
     - 小程序：`小程序 wx.login 参数 签名 AES 解密`

2. **调用论坛搜索**
   - 对每条 query 调用 `reverse_forum_search`：
     - `forum`: `"kanxue"` 或 `"52pojie"`
     - `query`: 关键词
     - `limit`: 8~12
     - `sort`: 优先 `"replies"`，偏新内容用 `"time"`

3. **挑选代表性帖子**
   - 优先：标题含「原创」「逆向」「加密」「脱壳」「绕过」「frida」等；回复数高；与目标关键词重叠高。
   - 选 1~3 帖进入详情。

4. **抓取帖子详情**
   - 对选中帖子调用 `reverse_forum_thread_detail`（同一 `forum`）：
     - `threadId` 或 `url`、`maxReplies`: 5~10（默认 8）
   - **仅看雪**：若正文有「回复可见」或明显截断，调用 `kanxue_reverse_reply`（threadId、message、fetchAfterReply: true）后用返回内容替换。

5. **输出「逆向思路总结」**
   - 须包含：可参考帖子列表（标题 + URL）、可复用步骤（抓包入口、关键函数、hook 点、加解密路径）、工具链建议、可能的坑。

## 预算与限流

- 每轮最多：搜索 3 次（3 条 query）、详情 3 篇、看雪回复解锁 2 篇。
- 同一 query 10 分钟内不重复搜索（除非用户要求刷新）。
