---
name: csharp-winforms-patterns
description: C# .NET Framework 4.8 WinForms 开发常用模式。SmartThreadPool 多线程、ConcurrentQueue 代理池、HttpWebRequest 网络请求、跨线程 UI 更新、后台任务、subprocess 调用。当用户用 C# WinForms 开发多线程工具/抢购/批量请求/代理管理时使用。
---

# C# WinForms 常用开发模式

目标框架: .NET Framework 4.8, 编译器: VS2022 MSBuild (ToolsVersion 15.0)

## 一、多线程模式 (SmartThreadPool)

### NuGet 包
```
SmartThreadPool.dll 2.2.6
```

### 批量任务执行
```csharp
private SmartThreadPool _stp;
private volatile bool _cancellationRequested;

private void StartBatch(List<AccountInfo> targets, int threadCount)
{
    _cancellationRequested = false;
    _stp = new SmartThreadPool(new STPStartInfo
    {
        MaxWorkerThreads = threadCount,
        MinWorkerThreads = 1,
    });

    foreach (var acct in targets)
    {
        var a = acct; // 闭包捕获
        _stp.QueueWorkItem(new WorkItemCallback(state =>
        {
            if (_cancellationRequested) return null;
            try { DoWork(a); }
            catch (Exception ex) { SetStatus(a, "失败", ex.Message); }
            return null;
        }), null);
    }

    // 等待线程在后台完成，不阻塞 UI
    new Thread(() =>
    {
        try { _stp.WaitForIdle(); } catch { }
        AppendLog("全部任务完成");
        Invoke(new Action(() => { btnStart.Enabled = true; btnStop.Enabled = false; }));
    }) { IsBackground = true }.Start();
}

private void StopBatch()
{
    _cancellationRequested = true;
    new Thread(() =>
    {
        try { _stp?.Shutdown(true, 3000); _stp?.Dispose(); }
        catch { }
        Invoke(new Action(() => { btnStart.Enabled = true; btnStop.Enabled = false; }));
    }) { IsBackground = true }.Start();
}
```

### 单次后台操作 (非批量)
```csharp
btnAction.Enabled = false;
new Thread(() =>
{
    try
    {
        // 耗时操作...
        AppendLog("完成");
    }
    catch (Exception ex) { AppendLog("异常: " + ex.Message); }
    finally { Invoke(new Action(() => btnAction.Enabled = true)); }
}) { IsBackground = true }.Start();
```

## 二、跨线程 UI 更新

### 日志输出 (RichTextBox)
```csharp
private void AppendLog(string msg)
{
    string line = DateTime.Now.ToString("HH:mm:ss.fff") + " " + msg + "\n";
    if (rtbLog.InvokeRequired)
        rtbLog.BeginInvoke(new Action(() => AppendLogInternal(line)));
    else
        AppendLogInternal(line);
}

private void AppendLogInternal(string line)
{
    if (rtbLog.TextLength > 200000)
        rtbLog.Clear();
    rtbLog.AppendText(line);
}
```

### ListView 刷新
```csharp
private void RefreshList()
{
    if (lvAccounts.InvokeRequired)
    {
        lvAccounts.BeginInvoke(new Action(RefreshListInternal));
        return;
    }
    RefreshListInternal();
}

private void RefreshListInternal()
{
    lvAccounts.BeginUpdate();
    // 修改 Items...
    lvAccounts.EndUpdate();
}
```

### 要点
- 后台线程更新 UI 必须用 `Invoke`/`BeginInvoke`
- `BeginInvoke` 异步不阻塞调用线程（推荐用于日志）
- `Invoke` 同步等待 UI 线程执行完（用于需要返回值或顺序保证时）
- 按钮禁用/启用必须在 UI 线程: `Invoke(new Action(() => btn.Enabled = true))`

## 三、代理池 (ConcurrentQueue)

### 用完即弃模式
```csharp
public class ProxyService
{
    private readonly ConcurrentQueue<WebProxy> _queue = new ConcurrentQueue<WebProxy>();
    public bool Enabled { get; set; }
    public int Count => _queue.Count;

    public int Fetch(string url)
    {
        var req = (HttpWebRequest)WebRequest.Create(url);
        req.Timeout = 15000;
        using (var resp = (HttpWebResponse)req.GetResponse())
        using (var sr = new StreamReader(resp.GetResponseStream(), Encoding.UTF8))
        {
            string text = sr.ReadToEnd().Trim();
            int added = 0;
            foreach (var line in text.Split(new[] { '\r', '\n' }, StringSplitOptions.RemoveEmptyEntries))
            {
                var parts = line.Trim().Split(':');
                int port;
                if (parts.Length >= 2 && int.TryParse(parts[1].Trim(), out port))
                {
                    _queue.Enqueue(new WebProxy(parts[0].Trim(), port));
                    added++;
                }
            }
            return added;
        }
    }

    public WebProxy GetProxy()
    {
        if (!Enabled) return null;
        WebProxy p;
        return _queue.TryDequeue(out p) ? p : null;
    }
}
```

### 在请求中使用
```csharp
var req = (HttpWebRequest)WebRequest.Create(url);
if (_proxy != null && _proxy.Enabled)
{
    var p = _proxy.GetProxy();
    if (p != null) req.Proxy = p;
}
```

### 要点
- `ConcurrentQueue` 线程安全，无需额外加锁
- 用完即弃: `TryDequeue` 取出后不回队列，适合短效代理
- 队列空时走直连（不报错）
- UI 勾选框绑定 `Enabled` 属性控制开关
- 代理 API 一般返回 `IP:PORT` 纯文本，按行解析

## 四、HttpWebRequest 网络请求

### 基础 POST + JSON 解析
```csharp
private HttpWebRequest CreateRequest(string url)
{
    var req = (HttpWebRequest)WebRequest.Create(url);
    req.Method = "POST";
    req.ContentType = "application/json";
    req.Timeout = 20000;
    req.AutomaticDecompression = DecompressionMethods.GZip | DecompressionMethods.Deflate;
    return req;
}

private ApiResult PostAndParse(HttpWebRequest req, string bodyJson)
{
    byte[] data = Encoding.UTF8.GetBytes(bodyJson);
    req.ContentLength = data.Length;
    using (var s = req.GetRequestStream())
        s.Write(data, 0, data.Length);

    try
    {
        using (var resp = (HttpWebResponse)req.GetResponse())
        using (var sr = new StreamReader(resp.GetResponseStream(), Encoding.UTF8))
        {
            string text = sr.ReadToEnd().Trim();
            var result = new ApiResult();
            result.Data = JObject.Parse(text);
            result.BizStatusCode = result.Data.Value<int?>("bizStatusCode") ?? 0;
            result.Success = (result.BizStatusCode == 10000);
            return result;
        }
    }
    catch (WebException ex)
    {
        var result = new ApiResult { Success = false };
        var errResp = ex.Response as HttpWebResponse;
        if (errResp != null)
        {
            using (var sr = new StreamReader(errResp.GetResponseStream(), Encoding.UTF8))
                result.RawBody = sr.ReadToEnd();
            result.ErrorMessage = string.Format("HTTP {0}: {1}",
                (int)errResp.StatusCode, result.RawBody.Substring(0, Math.Min(result.RawBody.Length, 200)));
        }
        else
            result.ErrorMessage = ex.Message;
        return result;
    }
}
```

### 503 自动重试
```csharp
public JObject GetWithRetry(string url, int maxRetry = 3)
{
    Exception lastEx = null;
    for (int i = 0; i < maxRetry; i++)
    {
        try { return DoRequest(url); }
        catch (WebException we)
        {
            lastEx = we;
            var httpResp = we.Response as HttpWebResponse;
            if (httpResp != null && (int)httpResp.StatusCode == 503 && i < maxRetry - 1)
            {
                Thread.Sleep(2000 * (i + 1)); // 递增等待
                continue;
            }
            throw;
        }
    }
    throw lastEx;
}
```

### TLS 绕过 (curl_cffi subprocess)
当目标 WAF 拦截 .NET TLS 指纹 (返回 405) 时，通过 Python subprocess 绕过:
```csharp
private ApiResult TryPostViaCurlCffi(string url, Dictionary<string, string> headers, string bodyJson)
{
    string script = FindHttpHelper(); // http_helper.py
    var input = new JObject { ["url"] = url, ["headers"] = JObject.FromObject(headers),
        ["body"] = bodyJson, ["method"] = "POST" };

    var psi = new ProcessStartInfo("python", script)
    {
        UseShellExecute = false,
        RedirectStandardInput = true,
        RedirectStandardOutput = true,
        RedirectStandardError = true,
        CreateNoWindow = true,
    };
    using (var proc = Process.Start(psi))
    {
        proc.StandardInput.Write(input.ToString(Formatting.None));
        proc.StandardInput.Close();
        string stdout = proc.StandardOutput.ReadToEnd();
        string stderr = proc.StandardError.ReadToEnd();
        proc.WaitForExit(30000);
        // 解析 stdout JSON: {"status": 200, "body": "..."}
    }
}
```

### 要点
- `ServicePointManager.Expect100Continue = false` 放 Program.cs 避免 Expect: 100-continue
- WebException 的 Response 可能为 null (网络不通)，需判断
- `AutomaticDecompression` 自动解 gzip，不需手动
- 超时建议 20 秒，代理场景可适当加长

## 五、账号持久化 (纯文本)

### 格式
```
手机号----sessionId=xxx----userId
```

### 保存
```csharp
private void SaveAccounts()
{
    var lines = new List<string>();
    lock (_accountLock)
    {
        foreach (var a in _accounts)
        {
            if (string.IsNullOrEmpty(a.SessionId)) continue;
            lines.Add(string.Format("{0}----sessionId={1}----{2}", a.Phone, a.SessionId, a.UserId));
        }
    }
    File.WriteAllLines(AccountsFile, lines, Encoding.UTF8);
}
```

### 加载
```csharp
private void LoadAccounts()
{
    if (!File.Exists(AccountsFile)) return;
    var lines = File.ReadAllLines(AccountsFile, Encoding.UTF8);
    foreach (var line in lines)
    {
        var parts = line.Trim().Split(new[] { "----" }, StringSplitOptions.None);
        if (parts.Length < 3) continue;
        string phone = parts[0].Trim();
        string session = parts[1].Trim();
        if (session.StartsWith("sessionId=", StringComparison.OrdinalIgnoreCase))
            session = session.Substring("sessionId=".Length);
        // 构造 AccountInfo...
    }
}
```

## 六、共享状态与锁

```csharp
private readonly List<AccountInfo> _accounts = new List<AccountInfo>();
private readonly object _accountLock = new object();

// 读取
lock (_accountLock)
    targets = _accounts.Where(a => !string.IsNullOrEmpty(a.SessionId)).ToList();

// 写入
lock (_accountLock)
    _accounts.Add(account);
```

### 要点
- `List<T>` 非线程安全，多线程读写必须 `lock`
- `lock` 内只做简单操作（读/写/拷贝），不做耗时操作
- `ConcurrentQueue` / `ConcurrentDictionary` 本身线程安全，无需额外 lock
- `volatile` 用于简单布尔标志（如 `_cancellationRequested`）

## 七、编译与依赖

### MSBuild
```powershell
# 必须用 VS2022 MSBuild (支持 C# 6+ 语法)
& "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\MSBuild\Current\Bin\MSBuild.exe" `
  Solution.sln /p:Configuration=Release /t:Rebuild /v:minimal
```

### NuGet 包管理
```powershell
# 下载 nuget.exe (如果没有)
Invoke-WebRequest "https://dist.nuget.org/win-x86-commandline/latest/nuget.exe" -OutFile nuget.exe
# 还原包
.\nuget.exe restore Solution.sln
```

### 常用包
| 包 | 用途 |
|---|------|
| Newtonsoft.Json 13.0.3 | JSON 序列化/反序列化 |
| SmartThreadPool.dll 2.2.6 | 线程池 |

### 注意
- .NET Framework 4.0 MSBuild 不支持 C# 6+ 语法（auto-property initializer `= "default"` 等）
- PowerShell 中 `&&` 不可用，用 `;` 分隔或分步执行
- `http_helper.py` 等资源文件需在 .csproj 中设 `CopyToOutputDirectory=PreserveNewest`
