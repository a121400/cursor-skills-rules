---
name: rush-buy-automation
description: 抢购/秒杀自动化。多线程并发Barrier同步、SOCKS5动态代理池(TTL/限频/预取)、库存监控、纪念币/门票/电商抢购、币安期货HMAC签名、登录超时重连策略。当用户要做定时抢购/秒杀/批量并发请求时使用。
---

# 抢购/秒杀自动化 (Auto-Learned)

用户高频任务：纪念币/纪念钞/商品抢购，多线程并发请求。

## 典型架构

```python
import threading
import time
import requests

class RushBuyEngine:
    def __init__(self, config):
        self.session = requests.Session()
        self.proxy_pool = config.get('proxies', [])
        self.target_time = config['target_time']

    def prepare(self):
        """提前登录、预热连接"""
        self.session.get(self.login_url)
        self.check_inventory()

    def rush(self, thread_count=10):
        """多线程抢购"""
        barrier = threading.Barrier(thread_count)
        threads = []
        for i in range(thread_count):
            t = threading.Thread(target=self._worker, args=(barrier,))
            threads.append(t)
            t.start()
        for t in threads:
            t.join()

    def _worker(self, barrier):
        barrier.wait()  # 同步起跑
        try:
            resp = self.session.post(self.buy_url, json=self.payload)
            print(f"[{threading.current_thread().name}] {resp.status_code}: {resp.text[:200]}")
        except Exception as e:
            print(f"Error: {e}")
```

## 嘉立创/纪念币模式 (hxjlc)

- 模块化：配置项分**纪念币**和**纪念钞**
- 支持查询库存函数
- 隧道代理动态IP (SOCKS5，账号密码认证)
- 身份信息管理（姓名+身份证+手机号+城市）

```python
# 代理配置
PROXY_CONFIG = {
    'url': 'http://proxy-api.example.com/get',
    'params': {'count': 2, 'format': 'socks5', 'auth': True}
}

def get_dynamic_proxy():
    """获取隧道代理动态IP"""
    resp = requests.get(PROXY_CONFIG['url'], params=PROXY_CONFIG['params'])
    ips = resp.text.strip().split('\n')
    return {'https': f'socks5://{user}:{pwd}@{ips[0]}'}
```

- **代理池带过期与限频**：池结构用 VecDeque + entry(fetched_at)，TTL（如 270s）消费时跳过过期；refresh 限频（距上次 <1s 则 sleep）；next() 消费式取队头，available_count() 供 UI。
- **SOCKS5 定时前预取**：若定时点为 10:00，在 start_time - 60s 获取代理，获取完再等到 10:00 开抢；执行中按 IP 有效期（如 5 分钟）周期刷新。

## 桌面客户端抢购 (sxd/sxdcg)

- C++ 或 Python 桌面程序
- Qt6 GUI 界面
- 文件上传 + 库存查询 + 抢购一体
- 二维码加载（注意编码问题）
- 支持多账号管理

## 下单流程逆向 (YLH/yuabn)

```
1. MCP 分析下单接口
2. 提取 open_id / token
3. 逆向验证码模块（滑块图坐标采集）
4. 构建完整下单请求
5. 多线程并发执行
```

## 易语言对接 (ZYB)

用户有时用 Python 生成加密数据，放到易语言程序中提交：
- Python 生成 cookie
- Python 生成提交数据（不提交）
- 易语言负责最终 HTTP 请求
- 调试时注意协议头完整性（Origin, Referer）

## 币安期货 (Binance Futures)

- 标的：ETH/USDT 等，需要分析买入、卖出、仓位监控接口
- 要求 24 小时运行，用 jshook MCP 分析 binance.com 前端接口

### USDS-M 合约模块化实现模式 (eth_monitor)

- 公共行情与私有交易分模块：`data_fetcher.py`（公开行情）、`auth.py`（HMAC-SHA256 签名与鉴权）、`trade.py`（下单）、`position.py`（仓位/账户）、`runner.py`（24 小时循环）
- 所有私有接口统一通过 `auth` 模块做 HMAC-SHA256 签名，追加 `timestamp` 与 `signature`，严格按币安官方 USDS-M REST 文档实现
- API Key / Secret 只从环境变量或本地未入库配置文件读取，禁止写死
- 交易逻辑由 `runner` 驱动：拉行情 -> 计算指标 -> 生成信号 -> 决定下单，同时每轮拉取仓位做强平预警，支持「仅监控不下单」模式
- 主循环需异常捕获与退避重试、`logging` 文件输出、`SIGINT/SIGTERM` 优雅退出

## TLS 指纹绕过

```python
# 检测目标是否有 TLS 指纹检测（如 esunny.vip）
# 使用 sunny 的 DLL 进行 TLS 请求
# 方案: 自定义 TLS 客户端，修改 cipher suites 匹配目标
import ssl
context = ssl.create_default_context()
```

- 易语言和 Python 协作：Python 生成加密数据 + 易语言发请求
- 代理 IP 轮换（隧道代理 SOCKS5）

## 登录/发单超时与重连

- **根因**：ETIMEDOUT/ECONNRESET 多为网关或网络问题，自动重连 + 登录失败后再次 new 连接会叠加成「一直 1/5、2/5 重连」的观感。
- **对策**：设登录/发单总失败次数或总时长上限，超限后前端明确提示并停止重试；crash.log 与 server 日志对比定位问题网关 IP，可做网关黑名单或降低该 IP 重连次数；健康检查超时（如 60s 无数据）触发断线重连时，避免与登录重试叠加成死循环。

## SOCKS5 动态代理 IP 信誉与反爬

### 动态 vs 粘性 (Sticky) 代理

| 模式 | 格式示例 | 特点 |
|---|---|---|
| 动态 | `socks5://user-res-ROW:pass@host:9595` | 每次新连接换 IP |
| 粘性 | `socks5://user-res-ROW-Lsid-{id}-TTL-1800:pass@host:9595` | 同 Lsid 保持 IP 30 分钟 |

### IP 信誉问题 (LinkedIn 实测)

- 住宅代理 IP 也会被目标站点标记，不是只有机房 IP 才被封
- 测试 5 个不同住宅 IP (美国/欧洲)，**全部** 被 LinkedIn 拦截 profile 页 (999)，但 login 页正常 (200)
- 粘性代理: 同一 IP 持续被拦截，重试无意义
- **动态代理**: 每次 re-init 换新 IP，约 60% 的 IP 能通过，配合重试策略可解决

### 动态代理 + 重试的最佳实践

```
1. init session -> 访问基础页拿 cookie
2. 发请求 -> 如果被拦截 (999/403):
   a. 重建 session (动态代理自动换 IP)
   b. 重新拿 cookie
   c. 重试，等待时间递增 (3s, 5s, 7s...)
3. 成功后复用当前 session (keep-alive 保持同 IP)
```

### 多线程 + 动态代理注意事项

- 每个线程独立 session + 独立代理连接
- 粘性代理: 每个线程用不同 Lsid (`base_id + worker_id`)，避免共享 IP
- 动态代理: 线程间天然隔离，无需特殊处理
- `threading.Event` 做全局 stop 信号，`time.sleep` 改为可中断版本 (每 0.5s 检查)

## C# WinForms 抢购 GUI 模式 (JWRush/劲舞团)

### 架构

| 组件 | 职责 |
|------|------|
| DataGridView | 账号列表 + 状态 + 支付链接 |
| ShopClient | HttpClient 封装（代理/Cookie/登录） |
| SemaphoreSlim | 并发账号数控制 |
| CancellationTokenSource per account | 单账号成功/风控即停 |
| ConcurrentQueue<string> | 代理池循环使用 |
| Timer + ConcurrentQueue | 日志批量刷新（50ms 间隔） |

### OAuth2 登录含 JS 重定向

```
POST login API → result_uri
GET result_uri → 302 → checkstatOauthZc?code=xxx&state=00
→ 返回 <script>top.location.href='...&state=1'</script>
→ 必须再 GET 带 state=1 的 URL 才能真正建立 session
```

### 一键登录 + 定时抢购流程

```
一键登录（不走代理）→ 存 cookie 到 Row.Tag
  ↓ 定时模式:
T-5分钟: 自动重新登录刷新 cookie
T-5秒:   等待
T:        提取代理 → SpinWait 精确等待 → 瞬间发请求
  ↓ 非定时模式:
点击开始 → 提取代理 → 直接发请求
```

### 代理管理

- HTTP 代理池 API（如熊猫代理），按 `min(账号×并发×2, 200)` 提取
- `ConcurrentQueue` 循环使用（TryDequeue → Enqueue 放回队尾）
- 503/Timeout 自动换下一个代理：`client.Dispose(); NewClient();`
- C# `WebProxy` 需设 `BypassProxyOnLocal = false` + `ServerCertificateCustomValidationCallback = true`

### 风控关键词即停

```csharp
if (detail.Contains("重复下单") || detail.Contains("频繁") ||
    detail.Contains("明日") || detail.Contains("购买上限") ||
    detail.Contains("已到达"))
    → 该账号 CancellationTokenSource.Cancel()
    → UpdateRowStatus(ri, "风控")

if (detail.Contains("IP被封") || detail.Contains("403"))
    → 全局 _cts.Cancel()（所有账号停止）

if (detail.Contains("未失效"))
    → 有待支付订单，仍尝试提交获取支付链接
```

### 结果推送

- Flask 网页服务（/api/push POST 接收，/ 展示）
- C# 抢购成功时 POST 推送 `{phone, pay_url, order_no, amount}`
- 网页 3 秒自动刷新，暗色主题，点击跳转支付
- Nginx 反代 `/pay/` → Flask 8899

### 支付链接提取

```
1. GET /cart/buy?params → 确认页 HTML（含 orderid + sign）
2. POST /Cart/buyresultsign → 302 → lianpay.bamboogame.cn?...JWT...
   或 AllowAutoRedirect=true 时检查 finalUrl.Contains("lianpay")
3. 支付链接含 au_transaction_order_number + transaction_amount + access=Bearer JWT
```

### 配置持久化

- JSON 文件 `jwrush_config.json` 保存所有设置 + 账号列表
- 窗体关闭自动保存，启动自动加载
- 大区列表异步加载（3秒后恢复选中索引）

### 状态颜色

| 状态 | 颜色 |
|------|------|
| 成功 | 绿色背景 |
| 待支付 | 黄色背景 |
| 风控/IP被封/失败 | 红色背景 |
| 其他 | 白色 |

## C# WinForms 多账号抢购工具 (AntfansHelper/鲸探)

### 架构特点

| 组件 | 职责 |
|------|------|
| SmartThreadPool | 并发下单(替代手动 Thread/Barrier) |
| RelaySigner | 远程中继签名(Xposed TCP 模块, 503 自动重试) |
| ApseSigner | 本地 JAR 签名(unidbg, 随 EXE 启动) |
| ProxyService | 熊猫代理池(ConcurrentQueue, 用完即弃) |
| curl_cffi subprocess | TLS 指纹绕过(mgs-biz 域 WAF 405) |
| accounts.txt | 纯文本持久化 `手机号----sessionId=xxx----userId` |
| TabControl | 启动/定时两个 Tab 页 |

### TLS WAF 绕过模式
mgs-biz.antfans.com 对 .NET HttpWebRequest 返回 405，需 Python subprocess:
```
C# AntfansApi -> TryPostViaCurlCffi() -> python http_helper.py (stdin JSON)
  -> curl_cffi impersonate=chrome120 -> 返回 JSON (stdout)
  -> 失败回退: safari17_0, chrome131_android
```

### 编译注意
- 项目 ToolsVersion=15.0，必须用 VS2022 MSBuild，不能用 .NET Framework 4.0 MSBuild
- `"C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\MSBuild\Current\Bin\MSBuild.exe"`
- NuGet 包: Newtonsoft.Json 13.0.3, SmartThreadPool 2.2.6

## 项目结构偏好

- 配置项独立文件
- 多版本 exe 只保留最新版
- 清理 `__pycache__`、`.bak` 备份文件
- 项目文件夹名用英文

## 景点/门票抢票模式 (天安门城楼)

- 抢票站点如果接口完全明文 JSON（无加密），优先用纯请求还原全流程：验证码获取、登录、可约日期查询、时段库存查询、票种列表、订单提交，而不是引入浏览器自动化。
- 将账号、密码、目标日期、首选时段、票种名称、乘客列表等全部抽到 `config.json` 中（如 `username/password/target_date/prefer_period/ticket_type/passengers[]`），脚本只依赖配置文件，便于多账号切换与复用。
- 数字图片验证码用 `ddddocr` 直接识别，登录流程封装为「获取验证码 → OCR → POST /login/in」的循环，限制最大重试次数，避免无限循环。
- 日期与时段选择模式：先从 `/showDate` 解析所有可约日期列表，再根据 `target_date` 选择 `indate`，从 `/getInventorySession` 查时段库存，如果首选无票再按优先级尝试其他时段。
- 下单阶段通过 `/findSellTicket` 动态获取票种 ID 与价格，构造订单 JSON 后 POST `/order/valid`，统一解析错误码与文案输出，便于后续扩展到其他景点抢票脚本。
