---
name: powershell
description: PowerShell 脚本编写指南和最佳实践。用于编写 PowerShell 脚本、自动化任务、系统管理。包含语法参考、常用模式、错误处理和调试技巧。
---

# PowerShell 脚本指南

## 基础语法

### 变量

```powershell
# 变量声明
$name = "value"
$number = 42
$array = @(1, 2, 3)
$hash = @{ key = "value"; key2 = "value2" }

# 类型声明
[string]$text = "hello"
[int]$count = 10
[array]$items = @()
[hashtable]$map = @{}
```

### 字符串操作

```powershell
# 字符串拼接
$result = "Hello, $name!"
$result = "Path: $($env:PATH)"

# Here-String（多行字符串）
$multiline = @"
第一行
第二行
变量: $name
"@

# 字符串方法
$str.ToUpper()
$str.ToLower()
$str.Replace("old", "new")
$str.Split(",")
$str.Trim()
```

### 条件语句

```powershell
if ($condition) {
    # 代码
} elseif ($other) {
    # 代码
} else {
    # 代码
}

# Switch
switch ($value) {
    "option1" { "处理选项1" }
    "option2" { "处理选项2" }
    default { "默认处理" }
}
```

### 循环

```powershell
# ForEach
foreach ($item in $collection) {
    Write-Host $item
}

# For
for ($i = 0; $i -lt 10; $i++) {
    Write-Host $i
}

# While
while ($condition) {
    # 代码
}

# Pipeline ForEach
$collection | ForEach-Object {
    Write-Host $_
}
```

## 函数

### 基本函数

```powershell
function Get-Greeting {
    param(
        [Parameter(Mandatory=$true)]
        [string]$Name,
        
        [Parameter(Mandatory=$false)]
        [string]$Title = "先生/女士"
    )
    
    return "您好，$Title $Name！"
}

# 调用
Get-Greeting -Name "张三" -Title "先生"
```

### 高级函数

```powershell
function Process-Data {
    [CmdletBinding()]
    param(
        [Parameter(Mandatory=$true, ValueFromPipeline=$true)]
        [string[]]$InputData,
        
        [Parameter()]
        [switch]$Verbose
    )
    
    begin {
        Write-Host "开始处理..."
    }
    
    process {
        foreach ($item in $InputData) {
            # 处理每个输入项
            Write-Output $item.ToUpper()
        }
    }
    
    end {
        Write-Host "处理完成"
    }
}
```

## 错误处理

```powershell
# Try-Catch-Finally
try {
    # 可能出错的代码
    $result = Get-Content "不存在的文件.txt" -ErrorAction Stop
}
catch [System.IO.FileNotFoundException] {
    Write-Host "文件未找到: $($_.Exception.Message)"
}
catch {
    Write-Host "发生错误: $($_.Exception.Message)"
}
finally {
    Write-Host "清理操作"
}

# ErrorAction 参数
# -ErrorAction Stop        # 停止执行
# -ErrorAction Continue    # 继续执行（默认）
# -ErrorAction SilentlyContinue  # 静默继续
# -ErrorAction Ignore      # 完全忽略
```

## 文件操作

```powershell
# 读取文件
$content = Get-Content -Path "file.txt"
$content = Get-Content -Path "file.txt" -Raw  # 读取为单个字符串
$content = Get-Content -Path "file.txt" -Encoding UTF8

# 写入文件
Set-Content -Path "file.txt" -Value "内容"
Add-Content -Path "file.txt" -Value "追加内容"
Out-File -FilePath "file.txt" -InputObject "内容" -Encoding UTF8

# JSON 操作
$json = Get-Content "data.json" | ConvertFrom-Json
$obj | ConvertTo-Json -Depth 10 | Set-Content "output.json"

# CSV 操作
$csv = Import-Csv "data.csv"
$data | Export-Csv "output.csv" -NoTypeInformation
```

## 常用 Cmdlet

### 对象操作

```powershell
# Select-Object - 选择属性
Get-Process | Select-Object Name, CPU, Memory

# Where-Object - 过滤
Get-Process | Where-Object { $_.CPU -gt 10 }

# Sort-Object - 排序
Get-Process | Sort-Object -Property CPU -Descending

# Group-Object - 分组
Get-Process | Group-Object -Property Name

# Measure-Object - 统计
Get-Process | Measure-Object -Property CPU -Sum -Average -Maximum
```

### 输出格式

```powershell
# 表格输出
Get-Process | Format-Table -AutoSize

# 列表输出
Get-Process | Format-List

# 网格视图
Get-Process | Out-GridView
```

## 调试技巧

```powershell
# 设置断点
Set-PSBreakpoint -Script "script.ps1" -Line 10

# 调试输出
Write-Debug "调试信息"
Write-Verbose "详细信息"

# 启用调试输出
$DebugPreference = "Continue"
$VerbosePreference = "Continue"

# 单步执行
Set-PSDebug -Step
```

## 实用脚本模式

### 日志函数

```powershell
function Write-Log {
    param(
        [string]$Message,
        [ValidateSet("INFO", "WARN", "ERROR")]
        [string]$Level = "INFO"
    )
    
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $logMessage = "[$timestamp] [$Level] $Message"
    
    switch ($Level) {
        "INFO"  { Write-Host $logMessage -ForegroundColor White }
        "WARN"  { Write-Host $logMessage -ForegroundColor Yellow }
        "ERROR" { Write-Host $logMessage -ForegroundColor Red }
    }
    
    Add-Content -Path "app.log" -Value $logMessage
}
```

### 配置文件加载

```powershell
function Import-Config {
    param([string]$Path)
    
    if (Test-Path $Path) {
        return Get-Content $Path | ConvertFrom-Json
    }
    else {
        throw "配置文件不存在: $Path"
    }
}
```

### 进度显示

```powershell
$items = 1..100
$total = $items.Count
$current = 0

foreach ($item in $items) {
    $current++
    $percent = ($current / $total) * 100
    Write-Progress -Activity "处理中" -Status "$current / $total" -PercentComplete $percent
    
    # 处理逻辑
    Start-Sleep -Milliseconds 50
}
```

## 最佳实践

1. **使用动词-名词命名规范**: `Get-Data`, `Set-Config`, `Remove-Item`
2. **始终使用 `-ErrorAction` 控制错误行为**
3. **使用 `[CmdletBinding()]` 支持通用参数**
4. **避免使用别名，使用完整 Cmdlet 名称**
5. **使用 UTF-8 编码处理中文**
6. **使用 `$PSScriptRoot` 获取脚本目录**
7. **检查管理员权限**: 
   ```powershell
   if (-NOT ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole] "Administrator")) {
       throw "需要管理员权限"
   }
   ```
