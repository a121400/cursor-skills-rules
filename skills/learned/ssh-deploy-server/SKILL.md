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

## Nginx 反代配置（端口未在腾讯云安全组放行时）

服务器 42.193.177.63 的腾讯云安全组只放行了有限端口（80/443/5050/8080 等），新端口不一定能外部访问。**优先用 nginx 反代到 80 端口**而非开新端口。

### 关键注意事项

1. **PowerShell 会展开 `$host`/`$remote_addr` 等变量**，不能通过 SSH heredoc 或 Python inline 命令写 nginx 配置。必须用 **paramiko sftp 写文件**（写独立 .py 脚本执行）。

2. **宝塔面板 phpfpm_status.conf 监听 80 端口**且 `server_name 127.0.0.1`，内部 curl 测试时会被它截获。测试用 `-H 'Host: 42.193.177.63'` 或直接从外部请求。

3. **新 server 块必须加 `default_server`**，否则请求可能落到 phpfpm_status 的 server 块。

### 配置模板（paramiko sftp 写入）

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh.connect("42.193.177.63", 22, "root", timeout=10)

conf = (
    "server {\n"
    "    listen 80 default_server;\n"
    "    server_name _;\n"
    "\n"
    "    location /<prefix>/ {\n"
    "        proxy_pass http://127.0.0.1:<PORT>/;\n"
    "        proxy_set_header Host " + "$" + "host;\n"
    "        proxy_set_header X-Real-IP " + "$" + "remote_addr;\n"
    "        proxy_set_header X-Forwarded-For " + "$" + "proxy_add_x_forwarded_for;\n"
    "    }\n"
    "}\n"
)

sftp = ssh.open_sftp()
with sftp.open("/www/server/panel/vhost/nginx/my_app.conf", "w") as f:
    f.write(conf)
sftp.close()

_, out, _ = ssh.exec_command("nginx -t 2>&1")
print(out.read().decode())
_, out, _ = ssh.exec_command("nginx -s reload 2>&1")
print(out.read().decode())
ssh.close()
```

### 前端适配反代路径

前端 fetch 请求的 URL 必须加 BASE 前缀，否则通过 `/prefix/` 反代时 API 打不到：

```javascript
const BASE = window.location.pathname.replace(/\/(admin|index\.html)?$/, '') || '';
fetch(BASE + '/api/submit', { ... })
```

### nginx 配置文件位置

- 宝塔 vhost 目录：`/www/server/panel/vhost/nginx/`
- 主配置末尾 `include /www/server/panel/vhost/nginx/*.conf;`
- 新项目创建 `<项目名>.conf` 即可被自动加载

## 常见问题

| 问题 | 解决 |
|------|------|
| 服务启动失败 exit-code 1 | `journalctl -u <name> -n 30` 看具体报错 |
| FileNotFoundError | 数据文件没上传，创建空文件 `touch /path/file.txt` |
| apt-get not found | RHEL 系用 `dnf` 不是 `apt` |
| PowerShell 中 curl 被劫持 | 用 `Invoke-WebRequest` 代替 |
| 中文路径 PowerShell 报错 | Shell 工具用 `working_directory` 参数 |
| scp 权限拒绝 | 检查 SSH 密钥是否正确添加 |
| 新端口外部访问不了 | 腾讯云安全组没放行，用 nginx 反代到 80 端口 |
| nginx 配置 `$host` 变成空 | PowerShell 展开了变量，用 paramiko sftp 写文件 |
| nginx 反代 404 | 内部测试 Host 是 127.0.0.1 被 phpfpm_status 截获，加 `default_server` 或用外部 Host 测试 |

## 已知服务器

**IP**: `42.193.177.63` | **系统**: OpenCloudOS 9.4 | **SSH**: root 免密 | **宝塔面板**: 8888 端口 (路径 `/tencentcloud`)

**腾讯云安全组已放行端口**: 20, 21, 22, 80, 443, 5050, 8080, 8888, 39000-40000（但 39000-40000 实际也被拦截，只有前面几个确认可用）

**nginx**: `/usr/bin/nginx` v1.28.1，配置目录 `/www/server/panel/vhost/nginx/`

**已安装 Python 包**: Flask, requests, pycryptodome

| 项目 | 本地端口 | Nginx 路径 | 服务器路径 | 服务名 |
|------|---------|-----------|-----------|--------|
| card_server (花小猪) | 5050 | 直连 :5050 | /opt/card_server | card-server |
| hxz_server (花小猪) | 8080 | — | /opt/hxz_server | — |
| xc_server (香菜) | 39088 | /xc/ → :39088 | /opt/xc_server | xc-server |
