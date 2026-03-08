---
name: windows-dev
description: Windows 开发环境配置和常用操作指南。用于 Windows 系统开发、环境配置、服务管理、注册表操作、进程管理等任务。包含 PowerShell 和 CMD 命令参考。
---

# Windows 开发指南

## 环境配置

### 环境变量

**查看环境变量:**
```powershell
# 查看所有环境变量
Get-ChildItem Env:

# 查看特定变量
$env:PATH
$env:USERPROFILE
```

**设置环境变量:**
```powershell
# 临时设置（当前会话）
$env:MY_VAR = "value"

# 永久设置（用户级别）
[Environment]::SetEnvironmentVariable("MY_VAR", "value", "User")

# 永久设置（系统级别，需管理员权限）
[Environment]::SetEnvironmentVariable("MY_VAR", "value", "Machine")
```

**添加到 PATH:**
```powershell
# 用户 PATH
$userPath = [Environment]::GetEnvironmentVariable("PATH", "User")
[Environment]::SetEnvironmentVariable("PATH", "$userPath;C:\NewPath", "User")
```

## 进程管理

**查看进程:**
```powershell
# 列出所有进程
Get-Process

# 按名称过滤
Get-Process -Name "node"

# 查看端口占用
netstat -ano | findstr :8080
Get-NetTCPConnection -LocalPort 8080
```

**终止进程:**
```powershell
# 按名称终止
Stop-Process -Name "node" -Force

# 按 PID 终止
Stop-Process -Id 12345 -Force

# 终止占用端口的进程
$pid = (Get-NetTCPConnection -LocalPort 8080).OwningProcess
Stop-Process -Id $pid -Force
```

## 服务管理

```powershell
# 列出所有服务
Get-Service

# 启动/停止/重启服务
Start-Service -Name "ServiceName"
Stop-Service -Name "ServiceName"
Restart-Service -Name "ServiceName"

# 设置服务启动类型
Set-Service -Name "ServiceName" -StartupType Automatic
```

## 文件操作

**常用命令:**
```powershell
# 复制文件（强制覆盖）
Copy-Item -Path "source" -Destination "dest" -Force

# 递归复制目录
Copy-Item -Path "source\*" -Destination "dest" -Recurse -Force

# 删除文件/目录
Remove-Item -Path "path" -Recurse -Force

# 创建目录
New-Item -ItemType Directory -Path "path" -Force

# 查找文件
Get-ChildItem -Path "C:\" -Filter "*.txt" -Recurse
```

## 注册表操作

```powershell
# 读取注册表项
Get-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion"

# 设置注册表值
Set-ItemProperty -Path "HKCU:\Software\MyApp" -Name "Setting" -Value "Value"

# 创建注册表项
New-Item -Path "HKCU:\Software\MyApp" -Force

# 删除注册表项
Remove-Item -Path "HKCU:\Software\MyApp" -Recurse
```

## 网络操作

```powershell
# 测试连接
Test-NetConnection -ComputerName "example.com" -Port 443

# DNS 查询
Resolve-DnsName "example.com"

# 防火墙规则
New-NetFirewallRule -DisplayName "Allow Port 8080" -Direction Inbound -LocalPort 8080 -Protocol TCP -Action Allow

# 查看网络适配器
Get-NetAdapter
```

## 常用开发工具路径

| 工具 | 默认路径 |
|------|----------|
| Node.js | `C:\Program Files\nodejs` |
| Python | `C:\Users\{user}\AppData\Local\Programs\Python` |
| Git | `C:\Program Files\Git` |
| VS Code | `C:\Users\{user}\AppData\Local\Programs\Microsoft VS Code` |
| npm 缓存 | `C:\Users\{user}\AppData\Roaming\npm-cache` |
| pip 缓存 | `C:\Users\{user}\AppData\Local\pip\cache` |

## 开发服务器

**常用端口:**
- 3000: React/Next.js 开发服务器
- 5173: Vite 开发服务器
- 8080: 通用 Web 服务器
- 8000: Django/FastAPI

**启动开发服务器后台运行:**
```powershell
# 使用 Start-Process 后台运行
Start-Process -FilePath "node" -ArgumentList "server.js" -WindowStyle Hidden

# 使用 Start-Job 运行
Start-Job -ScriptBlock { cd "C:\project"; npm run dev }
```

## 编码问题处理

```powershell
# 设置控制台编码为 UTF-8
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
chcp 65001

# PowerShell 输出编码
$OutputEncoding = [System.Text.Encoding]::UTF8
```

## 常见问题解决

### 端口被占用
```powershell
# 查找并终止
$proc = Get-NetTCPConnection -LocalPort 8080 -ErrorAction SilentlyContinue
if ($proc) { Stop-Process -Id $proc.OwningProcess -Force }
```

### 权限不足
```powershell
# 以管理员身份运行 PowerShell
Start-Process powershell -Verb runAs
```

### 执行策略
```powershell
# 查看当前策略
Get-ExecutionPolicy

# 设置策略允许脚本运行
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```
