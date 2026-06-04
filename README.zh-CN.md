# 从 Mac 远程访问 Ubuntu 24.04 GNOME：双 RDP 模式、Tailscale 与 0x207 修复完整指南

> 使用 Ubuntu 24.04 内置 GNOME Remote Desktop，从 macOS 通过 RDP 访问 Linux workstation。
> 支持「屏幕镜像」和「独立会话」两种模式并存，并通过 Tailscale 安全跨网访问。

英文版见：[README.md](README.md)。

---

## 本文与已有资料的关系

网上已经有一些关于 Ubuntu 24.04 Remote Desktop、macOS Windows App 连接失败，以及 `0x207` workaround 的零散资料。本文不声称首发发现 `0x207` 修复方法；本文的重点是把这些分散的坑整合成一套可复现的 workstation 部署方案：

- 用户级 GNOME Remote Desktop 做屏幕镜像；
- system 级 GNOME Remote Desktop 做 GDM 独立登录；
- 两种模式分配独立端口并保存为两个 `.rdp` 文件；
- 通过 Tailscale / MagicDNS 跨网络访问；
- 汇总 macOS Windows App 的兼容性参数和实际故障排查路径。

---

## 测试环境与硬件说明

| 项目 | 测试值 / 假设 |
|---|---|
| Ubuntu 主机 | Ubuntu 24.04 LTS，GNOME 46+ |
| 远程桌面栈 | Ubuntu 内置 `gnome-remote-desktop`，不是 `xrdp` |
| Mac 客户端 | macOS + App Store 的 Windows App（原 Microsoft Remote Desktop） |
| 网络 | Tailscale + MagicDNS |
| 硬件形态 | Ubuntu workstation / GPU workstation；镜像模式需要物理显示器或 EDID |
| 安全建议 | 只通过 Tailscale / VPN 访问，不要把 3389/3390 暴露到公网 |

硬件最影响的是「屏幕镜像」模式：镜像模式的分辨率取决于 Ubuntu 主机实际看到的物理显示器 / EDID。独立会话模式则由客户端协商分辨率，更适合 Mac 外接高分屏。GPU workstation 是常见使用场景，但本文配置本身不依赖 NVIDIA。

---

## 架构

```text
┌───────────────────────────────────────────────────┐
│ Ubuntu 24.04 workstation                           │
│                                                   │
│  ┌─────────────────────┐   ┌──────────────────┐  │
│  │ user-level grd       │   │ system-level grd │  │
│  │ 用户登录后启动       │   │ 开机即可启动      │  │
│  │ 镜像 GNOME 会话      │   │ 打开 GDM 登录     │  │
│  │ 端口 3389            │   │ 端口 3390         │  │
│  └─────────────────────┘   └──────────────────┘  │
│            ↑                        ↑             │
└────────────┼────────────────────────┼─────────────┘
             │                        │
       Tailscale / MagicDNS 私有网络
             │                        │
┌────────────┼────────────────────────┼─────────────┐
│ macOS client                                      │
│  ┌─────────┴────┐         ┌────────┴──────────┐  │
│  │ mirror.rdp    │         │ independent.rdp   │  │
│  └───────────────┘         └───────────────────┘  │
│     通过 Windows App 管理，或直接双击 .rdp 文件     │
└────────────────────────────────────────────────────┘
```

> Ubuntu GUI 默认设置在 Desktop Sharing 和 Remote Login 同时启用时可能自动分配不同端口。本文为了稳定使用两个导出的 `.rdp` 文件，明确把屏幕镜像固定为 `3389`，把 system 级独立登录固定为 `3390`。

---

## 两种模式对照

| 模式 | GNOME 功能 | 行为 | 本文端口 | 适合场景 |
|---|---|---|---|---|
| 屏幕镜像 | Desktop Sharing | Mac 看到 Ubuntu 物理屏幕，共用同一个鼠标指针 | `3389` | 看训练状态、dashboard、远程协助 |
| 独立会话 | Remote Login | Mac 连接 GDM，开启一个新的 GNOME 会话 | `3390` | 远程主力办公，不依赖物理显示器 |

关键认知：

- 镜像模式需要目标 Linux 用户已经登录物理 GNOME 桌面。
- 独立模式通过 GDM 登录，可以在目标桌面用户未登录前使用。
- 同一个 Linux 用户不适合同时持有两个活跃 GNOME 图形会话。如果想让本地物理会话和远程会话并行，建议创建专用远程账户。

---

## 1. 安装必要包

```bash
sudo apt update
sudo apt install gnome-remote-desktop winpr-utils
```

`winpr-utils` 提供 `winpr-makecert`，后面会用它给 system 级 RDP 服务生成 TLS 证书。

---

## 2. 配置屏幕镜像：用户级 Desktop Sharing

这个模式镜像当前已经登录的物理 GNOME 会话。

### 2.1 在 GNOME Settings 中启用 Desktop Sharing

在 Ubuntu 物理桌面上打开：

```text
Settings → System → Remote Desktop → Desktop Sharing
```

打开 Desktop Sharing。

### 2.2 设置用户级 RDP 凭据

```bash
grdctl rdp set-credentials "remote" "CHANGE_ME_STRONG_PASSWORD"
grdctl rdp enable
systemctl --user restart gnome-remote-desktop.service
```

### 2.3 验证

```bash
grdctl status
ss -tlnp | grep 3389
```

应当能看到用户级 RDP 已启用，并监听 `3389`。

---

## 3. 配置独立会话：system 级 Remote Login

这个模式通过 GDM 开启新的 GNOME 会话。

### 3.1 以 `gnome-remote-desktop` 用户生成 TLS 证书

```bash
sudo -u gnome-remote-desktop sh -c \
  'winpr-makecert -silent -rdp -path ~/.local/share/gnome-remote-desktop tls'
```

证书文件应当生成在：

```text
/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/
```

### 3.2 配置 system 级 RDP，并改到 3390 端口

```bash
sudo grdctl --system rdp set-tls-key \
  /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.key

sudo grdctl --system rdp set-tls-cert \
  /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.crt

sudo grdctl --system rdp set-credentials "remote" "CHANGE_ME_STRONG_PASSWORD"
sudo grdctl --system rdp set-port 3390
sudo grdctl --system rdp enable
```

### 3.3 启用 GDM 和 system 服务

```bash
sudo systemctl enable --now gdm.service
sudo systemctl enable --now gnome-remote-desktop.service
sudo systemctl daemon-reload
```

### 3.4 验证

```bash
sudo grdctl --system status
ss -tlnp | grep -E "3389|3390"
```

预期状态：

- `3389`：用户级 RDP 进程，用于屏幕镜像。
- `3390`：system 级 `gnome-remote-desktop` 进程，用于独立远程登录。

---

## 4. 可选：创建专用远程 Linux 账户

如果希望本地物理用户和远程用户同时工作，推荐新建一个专用远程账户：

```bash
sudo adduser remote-user
sudo usermod -aG sudo remote-user   # 可选：只有需要远程 sudo 时才加
```

独立会话模式进入 GDM 后，选择这个账户登录，而不是正在物理屏幕上使用的账户。

---

## 5. 防火墙与网络访问

如果启用了 UFW，建议只允许 Tailscale 接口访问 RDP：

```bash
sudo ufw allow in on tailscale0 to any port 3389 proto tcp
sudo ufw allow in on tailscale0 to any port 3390 proto tcp
```

不推荐直接全局放行，除非你明确知道风险：

```bash
sudo ufw allow 3389/tcp
sudo ufw allow 3390/tcp
```

不要把 RDP 端口直接暴露到公网。更安全的做法是使用 Tailscale、其他 VPN，或者 SSH tunnel。

---

## 6. macOS Windows App 配置

在 Mac App Store 安装 **Windows App**。

创建两个 PC 条目，或者导出两个 `.rdp` 文件：

| 文件 | 地址 | 用途 |
|---|---|---|
| `ubuntu-mirror.rdp` | `your-host.your-tailnet.ts.net` | 镜像物理 GNOME 会话，默认端口 `3389` |
| `ubuntu-independent.rdp` | `your-host.your-tailnet.ts.net:3390` | 独立 GDM / remote-login 会话 |

推荐设置：

- Gateway：无。
- 不要连接到 admin session。
- 启用双向剪贴板。
- 禁用打印机重定向。
- 禁用智能卡重定向。
- 除非你确认可用，否则禁用麦克风和摄像头重定向。
- 按需启用全屏和 smart sizing。

---

## 7. 修复 macOS Windows App 的 `0x207` 错误

这个错误经常很误导，Windows App 可能显示 password expired 或泛化的 `0x207` 连接失败。

关键 `.rdp` 参数是：

```text
use redirection server name:i:1
```

处理流程：

1. 在 Windows App 中右键目标 PC，导出 `.rdp` 文件。
2. 用文本编辑器打开 `.rdp` 文件。
3. 把：

   ```text
   use redirection server name:i:0
   ```

   改成：

   ```text
   use redirection server name:i:1
   ```

4. 同时确保 Linux 下容易出问题的重定向被禁用：

   ```text
   redirectprinters:i:0
   redirectsmartcards:i:0
   redirectclipboard:i:1
   ```

5. 直接双击 `.rdp` 文件测试。确认可用后，再按需重新导入 Windows App。
6. 如果 Windows App 一直使用旧凭据，在 macOS Keychain Access 中删除对应主机的旧凭据再试。

示例文件见：

- [`examples/ubuntu-mirror.rdp`](examples/ubuntu-mirror.rdp)
- [`examples/ubuntu-independent.rdp`](examples/ubuntu-independent.rdp)

---

## 8. 使用方式

### 屏幕镜像模式

前提：

- 目标 Linux 用户已经登录物理 GNOME 桌面。
- 物理会话没有锁屏。

连接地址：

```text
your-host.your-tailnet.ts.net
```

这个模式适合监控或协助操作。由于本地和远程共享同一个鼠标指针，不建议两边同时操作。

### 独立会话模式

连接地址：

```text
your-host.your-tailnet.ts.net:3390
```

你应当会看到 GDM 登录界面。可以选择：

- 专用远程账户，推荐；或
- 同一个本地用户，此时 GNOME 可能显示 **Session Already Running**，并询问是否 force-stop 本地会话。

---

## 9. 故障排查

### 连接超时

```bash
nc -zv your-host.your-tailnet.ts.net 3389
nc -zv your-host.your-tailnet.ts.net 3390
```

- Timeout：检查 Tailscale、MagicDNS、防火墙，以及两台设备是否在同一个 tailnet。
- Connection refused：检查对应的 `gnome-remote-desktop` 服务是否正在监听。

### 仍然出现 `0x207`

- 确认实际打开的 `.rdp` 文件中有 `use redirection server name:i:1`。
- 禁用打印机和智能卡重定向。
- 删除 macOS Keychain Access 中的旧凭据。
- 重新设置 RDP 凭据：

```bash
# 用户级
grdctl rdp set-credentials "remote" "NEW_PASSWORD"

# system 级
sudo grdctl --system rdp set-credentials "remote" "NEW_PASSWORD"
```

### 服务与日志

```bash
# 用户级
systemctl --user status gnome-remote-desktop.service
journalctl --user -u gnome-remote-desktop.service -n 30 --no-pager

# system 级
sudo systemctl status gnome-remote-desktop.service
sudo journalctl -u gnome-remote-desktop.service -n 30 --no-pager
```

常见现象：

- `BIO_new failed for certificate`：检查 TLS 证书路径和权限。
- `status=217/USER`：可能缺少 `gnome-remote-desktop` system user，可以尝试 `sudo systemd-sysusers` 后重启。
- `Init TPM credentials failed ... using GKeyFile as fallback`：没有 TPM-backed storage 时通常是正常 fallback。

### 镜像模式 3389 不监听

用户级 RDP 需要真实的 GNOME 图形会话。如果只是 SSH 登录，用户服务可能存在，但 Desktop Sharing endpoint 不一定可用。

```bash
echo $DBUS_SESSION_BUS_ADDRESS
loginctl show-user $USER | grep -E "State|Sessions"
```

### 分辨率不对

- 镜像模式受物理显示器 / EDID 限制。需要更高分辨率时，使用真实高分屏或虚拟 EDID dongle。
- 独立会话模式由客户端协商分辨率。可以调整 `.rdp` 文件或 Windows App 的显示设置。

---

## 10. 安全与维护

- 只通过 Tailscale 或其他私有网络访问 RDP。
- 不要公开真实 tailnet 名称、Tailscale IP、Linux 用户名、RDP 用户名 / 密码组合或私有主机名。
- RDP 密码应存入密码管理器，不应写入仓库。
- 如果将来重装 Ubuntu，建议备份：

```text
/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.{key,crt}
~/.local/share/gnome-remote-desktop/certificates/
```

---

## 速查命令

| 任务 | 命令 |
|---|---|
| 查用户级状态 | `grdctl status` |
| 查 system 级状态 | `sudo grdctl --system status` |
| 改用户级 RDP 密码 | `grdctl rdp set-credentials "user" "pass"` |
| 改 system 级 RDP 密码 | `sudo grdctl --system rdp set-credentials "user" "pass"` |
| 改 system 级端口 | `sudo grdctl --system rdp set-port 3390` |
| 重启用户级服务 | `systemctl --user restart gnome-remote-desktop.service` |
| 重启 system 级服务 | `sudo systemctl restart gnome-remote-desktop.service` |
| 看监听端口 | `ss -tlnp \| grep -E "3389\|3390"` |
| 看 Tailscale 状态 | `tailscale status` |
| 测试端口连通 | `nc -zv your-host.your-tailnet.ts.net 3389` |

---

## 参考资料

- Ubuntu Desktop documentation: [Share your desktop remotely](https://documentation.ubuntu.com/desktop/en/latest/how-to/share-your-desktop-remotely/)
- Emile M.: [RDP Error Code 0x207 on Mac for Ubuntu 24](https://dev.to/emile1636/rdp-error-code-0x207-on-mac-for-ubuntu-24-d6d)
- Tailscale documentation: [MagicDNS](https://tailscale.com/kb/1081/magicdns)
- FreeRDP / WinPR project: [FreeRDP GitHub repository](https://github.com/FreeRDP/FreeRDP)

---

## 公开发布安全提醒

本文刻意使用 `your-host.your-tailnet.ts.net` 之类占位符。不要把真实 tailnet 名、Tailscale IP、Linux 用户名、RDP 凭据或私有主机名提交到公开仓库。
