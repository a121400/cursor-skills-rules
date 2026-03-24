---
name: ssh-deploy-server
description: SSH 部署项目到远程服务器。paramiko 免密配置、scp 上传、systemd 服务创建、dnf/apt 依赖安装。当用户要求「部署到服务器」「上传到服务器」「同步到服务器」时使用。
---

# SSH 部署项目到远程服务器

## 前置信息（向用户确认）
- 服务器 IP
- SSH 端口（默认 22）
- SSH 用户名（默认 root）
- SSH 密码（或已有免密登录）
- 项目在服务器上的目标路径（默认 /opt/<项目名>）
- 服务监听端口

## 步骤

### 1. 检查本地 SSH 密钥

```powershell
ls ~/.ssh/id_rsa
```

如果不存在，用 paramiko 生成：

```python
import paramiko
from pathlib import Path

key = paramiko.RSAKey.generate(4096)
key_path = Path.home() / ".ssh" / "id_rsa"
key.write_private_key_file(str(key_path))
pub = f"ssh-rsa {key.get_base64()} deploy"
(Path.home() / ".ssh" / "id_rsa.pub").write_text(pub + "\n")
```

### 2. 用 paramiko 连接并配置免密登录

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect(HOST, 22, USER, PASSWORD, timeout=10)

# 添加公钥到服务器
pub_str = open(Path.home() / ".ssh" / "id_rsa.pub").read().strip()
ssh.exec_command("mkdir -p ~/.ssh && chmod 700 ~/.ssh")
ssh.exec_command(f'grep -qF "{pub_str}" ~/.ssh/authorized_keys 2>/dev/null || echo "{pub_str}" >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys')
ssh.close()
```

配置完成后即可 `ssh root@IP` 免密登录。

### 3. 检测服务器系统并安装依赖

先判断包管理器：

```bash
# RHEL/OpenCloudOS/CentOS
ssh root@IP "cat /etc/os-release | head -3"
```

| 系统 | 包管理器 | 安装命令 |
|------|---------|---------|
| OpenCloudOS / CentOS / RHEL | dnf / yum | `dnf install -y nodejs npm python3-pip` |
| Ubuntu / Debian | apt | `apt-get update && apt-get install -y nodejs npm python3-pip` |

Python 依赖用 pip：
```bash
ssh root@IP "pip3 install flask requests ..."
```

### 4. 上传代码（用 scp）

```powershell
# 单文件
scp local_file root@IP:/remote/path/

# 整个目录
scp -r local_dir root@IP:/remote/path/
```

注意事项：
- 不要上传 `__pycache__/`、`.db-shm`、`.db-wal`（除非需要同步数据库）
- Windows PowerShell 中 scp 路径用 `\` 分隔，远程路径用 `/`
- 中文目录名在 PowerShell 中需要用 `working_directory` 参数指定，不要用 `cd`

### 5. 同步数据库和数据文件

```bash
# 先停服务
ssh root@IP "systemctl stop <service-name>"

# 上传 db 文件（.db + .db-shm + .db-wal 一起传）
scp app.db root@IP:/remote/path/
scp app.db-shm root@IP:/remote/path/
scp app.db-wal root@IP:/remote/path/

# 上传数据文件
scp *.txt *.jsonl *.json root@IP:/remote/path/
```

### 6. 创建 systemd 服务

```bash
ssh root@IP "cat > /etc/systemd/system/<name>.service << 'EOF'
[Unit]
Description=My Service
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/<project>
ExecStart=/usr/bin/python3 /opt/<project>/server.py --port <PORT>
Restart=always
RestartSec=5
Environment=PYTHONUNBUFFERED=1

[Install]
WantedBy=multi-user.target
EOF"

ssh root@IP "systemctl daemon-reload && systemctl enable <name> && systemctl start <name>"
```

### 7. 验证

```bash
# 查看服务状态
ssh root@IP "systemctl status <name> --no-pager"

# 查看启动日志（排错用）
ssh root@IP "journalctl -u <name> --no-pager -n 30"

# 外部访问测试（PowerShell）
Invoke-WebRequest -Uri "http://IP:PORT/" -UseBasicParsing -TimeoutSec 10
```

### 8. 常用管理命令

```bash
systemctl start/stop/restart/status <name>   # 服务管理
journalctl -u <name> -f                       # 实时日志
```

## 常见问题

| 问题 | 解决 |
|------|------|
| 服务启动失败 exit-code 1 | `journalctl -u <name> -n 30` 看具体报错 |
| FileNotFoundError | 数据文件没上传，创建空文件 `touch /path/file.txt` |
| apt-get not found | RHEL 系用 `dnf` 不是 `apt` |
| PowerShell 中 curl 被劫持 | 用 `Invoke-WebRequest` 代替 |
| 中文路径 PowerShell 报错 | Shell 工具用 `working_directory` 参数 |
| scp 权限拒绝 | 检查 SSH 密钥是否正确添加 |

## 已知服务器

| 项目 | IP | 端口 | 路径 | 服务名 |
|------|-----|------|------|--------|
| card_server | 42.193.177.63 | 5050 | /opt/card_server | card-server |
