---
name: sunny-real-device-lan-proxy
description: 真机通过局域网连接 PC 端 SunnyNet 抓包。自动用 ADB 写入系统全局 HTTP 代理、解析本机内网 IP 与端口对齐、清代理恢复上网、防火墙与证书要点。当用户提到 SunnyNet 抓包、真机、手机代理、局域网 IP、MCP 连手机、http_proxy、网上国网/银行类 App 走代理时使用。
---

# 真机 + PC SunnyNet 局域网抓包（全局工作流）

## 角色分工

| 位置 | 含义 |
|------|------|
| **SunnyNet 跑在 PC** | 监听 `程序工作端口`（常见 2024/2025），抓包主界面在电脑 |
| **手机** | 通过 **HTTP 代理**把流量指到 **电脑的局域网 IP:端口**，不是 `127.0.0.1` |

**易错点**：`127.0.0.1` 在手机上指**手机自己**。只有 SunnyNet **装在本机 Android 上运行**时，手机代理才用 `127.0.0.1:端口`。

## 自动执行顺序（Agent 应实际跑命令，不要只口述）

1. **确认 ADB**：`adb devices`，需 `device` 且已授权。
2. **确认 SunnyNet 已在 PC 启动**，状态栏或设置里可见 **启动成功:端口**（记为 `PORT`）。
3. **取 PC 局域网 IP**（优先与手机同网段）：
   - Windows：`ipconfig`，取当前 Wi‑Fi/以太网的 **IPv4**（常见 `192.168.x.x` / `10.x.x.x`）。
   - 或看 SunnyNet **「当前内网 IP」** 列表，**优先选与普通家用路由器同段的 `192.168.*.*`**。
   - **慎用**：`198.18.*`（虚拟网卡/TUN）、`172.28.*`（WSL/虚拟交换机）——除非确认手机能路由到该地址。
   - 记为 `PC_IP`。
4. **写入手机全局代理**（与手动设 Wi‑Fi 代理等效，可脚本化）：
   ```powershell
   adb shell settings put global http_proxy PC_IP:PORT
   adb shell settings get global http_proxy
   ```
5. **本机防火墙**：允许 **入站 TCP 端口 `PORT`**（否则手机显示网络失败）。可让用户临时关闭防火墙验证。
6. **HTTPS**：手机须信任 **当前 PC SunnyNet 使用的 CA**（用 SunnyNet 内 **证书安装教程** 导出/安装到用户区或系统区）。与仅在手机侧安装的证书可能不是同一文件，以 PC 端为准。
7. **结束抓包或断网排查**时清代理（须清干净，否则部分 App 仍判「有代理」）：
   ```powershell
   adb shell settings delete global http_proxy
   adb shell settings delete global global_http_proxy_host
   adb shell settings delete global global_http_proxy_port
   adb shell settings delete global global_http_proxy_exclusion_list
   adb shell settings delete global global_proxy_pac_url
   ```
   说明：部分系统用 **`http_proxy` 单字段**，部分用 **`global_http_proxy_host` + `global_http_proxy_port`** 分字段，只删其一会导致金融类 App 仍提示「检测到 Wi‑Fi 代理」。
8. **端口不一致**：SunnyNet 改端口后，必须同步改 `http_proxy` 中的 `PORT`，否则全机断网。

## 与「仅 MCP / 仅浏览器」的区别

- **jshook / 浏览器 MCP**：抓的是 **PC 浏览器** 或自动化环境流量。
- **本 skill**：抓 **真实 Android App** 流量，依赖 **手机系统代理 → PC SunnyNet**，必要时再配合根证书。

## Root / 风控类 App（可选）

- 需隐藏 Root 时：Magisk **排除列表** + Shamiko，对目标包名 `magisk --denylist add <包名>`（与代理无关，另线处理）。
- 部分 App **忽略系统 HTTP 代理**，需 SunnyNet **TUN/VPN/模块** 或官方文档中的非代理模式；本 skill 的 `http_proxy` 无法覆盖此类情况。

## 强风控 App 检测「Wi‑Fi/系统代理」

- 金融、政务、电网类 App（如 **网上国网**）常检测 **`http.proxyHost` / `ConnectivityManager` / 全局 `http_proxy`**，一旦开启系统代理即拒绝或限流。
- **对策**：不要用 `settings put global http_proxy` 抓这类 App；改用 **TUN/VPN 层**（SunnyNet **OpenDrive + VPN**、手机 SunnyNet **模块**、或另一台无代理环境抓包）。抓完或不用时 **`settings delete global http_proxy`** 恢复直连。

## 静默安装 APK（减少手机点确认）

- 已 root：`adb push` 到 `/data/local/tmp/` 后 `adb shell su -c "pm install -r -g -t /path.apk"`。
- 用户若在桌面 `Gooooo` 下放有 `install-apk-silent.ps1`，可按该脚本参数调用。

## 自检清单

- [ ] `PC_IP` 与手机在同一二层网络（同 Wi‑Fi 或同网段路由）。
- [ ] `PORT` 与 SunnyNet **程序工作端口**一致。
- [ ] 防火墙放行 `PORT`。
- [ ] HTTPS 已装对应当前 SunnyNet 的 CA。
- [ ] 停抓后已 `settings delete global http_proxy`。
