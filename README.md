<div align="center">

# Ubuntu 24.04 GNOME Remote Desktop from macOS

Dual RDP modes, Tailscale access, and the macOS Windows App `0x207` fix.

[English](README.md) | [简体中文](README.zh-CN.md)

![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04-E95420)
![GNOME](https://img.shields.io/badge/GNOME-Remote%20Desktop-4A86CF)
![macOS](https://img.shields.io/badge/macOS-Windows%20App-000000)
![RDP](https://img.shields.io/badge/RDP-3389%20%2F%203390-0078D4)
![Tailscale](https://img.shields.io/badge/Tailscale-MagicDNS-6A5ACD)
![License](https://img.shields.io/github/license/ZONGHAOLISOTA/ubuntu-gnome-rdp-macos)

</div>

---

This repository is a reproducible setup guide for using **macOS** to access an **Ubuntu 24.04 GNOME** workstation through Ubuntu's built-in **GNOME Remote Desktop** stack.

It focuses on a practical workstation pattern:

- **Screen sharing / mirror mode** for watching or assisting the physical GNOME session.
- **Remote login / independent session mode** for starting a separate GNOME session through GDM.
- **Tailscale / MagicDNS** for cross-network access without exposing RDP to the public internet.
- **macOS Windows App compatibility fixes**, especially the misleading `0x207` / “password expired” failure.

---

## Relationship to existing guides

There are already scattered posts about Ubuntu 24.04 Remote Desktop and the macOS `0x207` workaround. This guide is **not** claiming first discovery of the `0x207` fix.

The goal is to combine the scattered parts into one end-to-end deployment pattern:

1. user-level GNOME Remote Desktop for screen sharing,
2. system-level GNOME Remote Desktop for remote login,
3. separate RDP ports for the two modes,
4. macOS `.rdp` client-side parameters,
5. Tailscale-based private networking,
6. troubleshooting notes for the actual failure modes.

Useful prior references are listed in [References](#references).

---

## Tested environment

| Component | Tested value / assumption |
|---|---|
| Ubuntu server | Ubuntu 24.04 LTS, GNOME 46+ |
| Remote desktop stack | Built-in `gnome-remote-desktop`, not `xrdp` |
| macOS client | macOS with **Windows App** from the App Store, formerly Microsoft Remote Desktop |
| Network | Tailscale with MagicDNS enabled |
| Hardware profile | Ubuntu workstation / GPU workstation with a physical display available for mirror mode |
| Recommended topology | Keep RDP private through Tailscale; do **not** expose 3389/3390 to the public internet |

Hardware matters mainly for **mirror mode**:

- Mirror mode uses the physical GNOME session, so its resolution is constrained by the monitor / EDID seen by the Ubuntu host.
- Independent remote-login mode negotiates resolution with the client, so it is much better for high-resolution macOS external displays.
- A GPU workstation is a common use case, but the RDP setup itself is not NVIDIA-specific.

---

## Architecture

```text
┌───────────────────────────────────────────────────┐
│ Ubuntu 24.04 workstation                           │
│                                                   │
│  ┌─────────────────────┐   ┌──────────────────┐  │
│  │ user-level grd       │   │ system-level grd │  │
│  │ starts after login   │   │ starts at boot   │  │
│  │ mirrors GNOME        │   │ opens GDM login  │  │
│  │ port 3389            │   │ port 3390        │  │
│  └─────────────────────┘   └──────────────────┘  │
│            ↑                        ↑             │
└────────────┼────────────────────────┼─────────────┘
             │                        │
       Tailscale / MagicDNS private network
             │                        │
┌────────────┼────────────────────────┼─────────────┐
│ macOS client                                      │
│  ┌─────────┴────┐         ┌────────┴──────────┐  │
│  │ mirror.rdp    │         │ independent.rdp   │  │
│  └───────────────┘         └───────────────────┘  │
│     managed by Windows App or opened directly      │
└────────────────────────────────────────────────────┘
```

> Ubuntu's GUI-managed defaults may allocate ports differently when both Desktop Sharing and Remote Login are enabled. This guide intentionally pins screen sharing to `3389` and system-level remote login to `3390` so the two exported `.rdp` files stay stable.

---

## Mode comparison

| Mode | GNOME feature | Behavior | Port in this guide | Best for |
|---|---|---|---|---|
| Mirror mode | Desktop Sharing | macOS sees the physical Ubuntu screen and shares the same pointer | `3389` | Monitoring training, checking dashboards, assisting local work |
| Independent session | Remote Login | macOS connects to GDM and starts a separate GNOME session | `3390` | Primary remote desktop work without relying on the physical monitor |

Key mental model:

- Mirror mode requires a logged-in physical GNOME session.
- Independent mode goes through GDM and can work before the target desktop user is logged in.
- One Linux user cannot safely own two active GNOME graphical sessions at the same time. Use a dedicated remote account if you want the physical session and the remote session to coexist.

---

## 1. Install required packages

```bash
sudo apt update
sudo apt install gnome-remote-desktop winpr-utils
```

`winpr-utils` provides `winpr-makecert`, which is used below to generate a TLS certificate for the system-level RDP service.

---

## 2. Configure mirror mode: user-level Desktop Sharing

This mode mirrors the currently logged-in physical GNOME session.

### 2.1 Enable Desktop Sharing in GNOME Settings

On the physical Ubuntu desktop:

```text
Settings → System → Remote Desktop → Desktop Sharing
```

Turn Desktop Sharing on.

### 2.2 Set user-level RDP credentials

```bash
grdctl rdp set-credentials "remote" "CHANGE_ME_STRONG_PASSWORD"
grdctl rdp enable
systemctl --user restart gnome-remote-desktop.service
```

### 2.3 Verify

```bash
grdctl status
ss -tlnp | grep 3389
```

You should see user-level RDP enabled and listening on port `3389`.

---

## 3. Configure independent mode: system-level Remote Login

This mode starts a new GNOME session through GDM.

### 3.1 Generate a TLS certificate as the `gnome-remote-desktop` user

```bash
sudo -u gnome-remote-desktop sh -c \
  'winpr-makecert -silent -rdp -path ~/.local/share/gnome-remote-desktop tls'
```

The certificate files should be created under:

```text
/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/
```

### 3.2 Configure system-level RDP on port 3390

```bash
sudo grdctl --system rdp set-tls-key \
  /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.key

sudo grdctl --system rdp set-tls-cert \
  /var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.crt

sudo grdctl --system rdp set-credentials "remote" "CHANGE_ME_STRONG_PASSWORD"
sudo grdctl --system rdp set-port 3390
sudo grdctl --system rdp enable
```

### 3.3 Enable GDM and the system service

```bash
sudo systemctl enable --now gdm.service
sudo systemctl enable --now gnome-remote-desktop.service
sudo systemctl daemon-reload
```

### 3.4 Verify

```bash
sudo grdctl --system status
ss -tlnp | grep -E "3389|3390"
```

Expected state:

- `3389`: user-level RDP process for mirror mode.
- `3390`: system-level `gnome-remote-desktop` process for independent remote login.

---

## 4. Optional: create a dedicated remote Linux account

This is recommended if you want the physical local user and the remote user to work simultaneously.

```bash
sudo adduser remote-user
sudo usermod -aG sudo remote-user   # optional; only if remote sudo is needed
```

When the GDM login screen appears in independent mode, select this account instead of your local physical-session account.

---

## 5. Firewall and network access

If UFW is enabled, prefer allowing RDP only on the Tailscale interface:

```bash
sudo ufw allow in on tailscale0 to any port 3389 proto tcp
sudo ufw allow in on tailscale0 to any port 3390 proto tcp
```

Avoid this unless you understand the risk:

```bash
sudo ufw allow 3389/tcp
sudo ufw allow 3390/tcp
```

Do **not** port-forward RDP directly to the public internet. Use Tailscale, another VPN, or an SSH tunnel.

---

## 6. macOS Windows App setup

Install **Windows App** from the Mac App Store.

Create two PC entries or two exported `.rdp` files:

| File | Address | Purpose |
|---|---|---|
| `ubuntu-mirror.rdp` | `your-host.your-tailnet.ts.net` | Mirror physical GNOME session on port `3389` |
| `ubuntu-independent.rdp` | `your-host.your-tailnet.ts.net:3390` | Independent GDM / remote-login session |

Recommended settings:

- Gateway: none.
- Do not connect to an admin session.
- Enable bidirectional clipboard.
- Disable printer redirection.
- Disable smart-card redirection.
- Disable microphone and camera redirection unless you have a known working setup.
- Enable full-screen and smart sizing if desired.

---

## 7. Fix macOS Windows App error `0x207`

The symptom is often misleading: Windows App may show a password-expired or generic `0x207` connection failure.

The important `.rdp` parameter is:

```text
use redirection server name:i:1
```

Procedure:

1. In Windows App, right-click the PC entry and export it as an `.rdp` file.
2. Open the `.rdp` file in a text editor.
3. Change:

   ```text
   use redirection server name:i:0
   ```

   to:

   ```text
   use redirection server name:i:1
   ```

4. Also make sure Linux-unfriendly redirections are disabled:

   ```text
   redirectprinters:i:0
   redirectsmartcards:i:0
   redirectclipboard:i:1
   ```

5. Open the `.rdp` file directly. If needed, re-import it into Windows App after confirming it works.
6. If Windows App keeps using stale credentials, delete the old host entry from macOS Keychain Access and retry.

See [`examples/ubuntu-mirror.rdp`](examples/ubuntu-mirror.rdp) and [`examples/ubuntu-independent.rdp`](examples/ubuntu-independent.rdp).

---

## 8. Usage

### Mirror mode

Prerequisites:

- The target Linux user is already logged into the physical GNOME desktop.
- The physical session is not locked.

Connect to:

```text
your-host.your-tailnet.ts.net
```

This is best treated as a monitoring / assistance mode. The local and remote pointers are the same pointer, so avoid fighting the physical user for control.

### Independent mode

Connect to:

```text
your-host.your-tailnet.ts.net:3390
```

You should see the GDM login screen. Select either:

- a dedicated remote account, recommended; or
- the same local user, in which case GNOME may show **Session Already Running** and ask whether to force-stop the local session.

---

## 9. Troubleshooting

### Timeout

```bash
nc -zv your-host.your-tailnet.ts.net 3389
nc -zv your-host.your-tailnet.ts.net 3390
```

- Timeout: check Tailscale, MagicDNS, firewall rules, and whether both devices are in the same tailnet.
- Connection refused: check whether the relevant `gnome-remote-desktop` service is listening.

### `0x207` still happens

- Confirm `use redirection server name:i:1` is present in the actual `.rdp` file being opened.
- Disable printer and smart-card redirection.
- Delete stale credentials from macOS Keychain Access.
- Reset RDP credentials:

```bash
# user-level
grdctl rdp set-credentials "remote" "NEW_PASSWORD"

# system-level
sudo grdctl --system rdp set-credentials "remote" "NEW_PASSWORD"
```

### Services and logs

```bash
# user-level
systemctl --user status gnome-remote-desktop.service
journalctl --user -u gnome-remote-desktop.service -n 30 --no-pager

# system-level
sudo systemctl status gnome-remote-desktop.service
sudo journalctl -u gnome-remote-desktop.service -n 30 --no-pager
```

Common notes:

- `BIO_new failed for certificate`: check TLS certificate paths and permissions.
- `status=217/USER`: the `gnome-remote-desktop` system user may be missing; try `sudo systemd-sysusers`, then restart.
- `Init TPM credentials failed ... using GKeyFile as fallback`: this is usually expected on machines without TPM-backed storage.

### Mirror mode does not listen on 3389

User-level RDP requires a real GNOME graphical session. If you are only connected over SSH, the user service may exist but the desktop sharing endpoint may not be available.

```bash
echo $DBUS_SESSION_BUS_ADDRESS
loginctl show-user $USER | grep -E "State|Sessions"
```

### Resolution is wrong

- Mirror mode is constrained by the physical monitor / EDID. Use a real high-resolution display or a virtual EDID dongle if needed.
- Independent mode negotiates resolution with the client. Adjust the `.rdp` file or Windows App display settings.

---

## 10. Security and maintenance

- Keep RDP private through Tailscale or another private network.
- Do not publish your real tailnet name, Tailscale IP, Linux username, RDP username/password pair, or private hostnames.
- Store RDP passwords in a password manager, not in this repository.
- Back up these files if you want to reproduce the setup after reinstalling Ubuntu:

```text
/var/lib/gnome-remote-desktop/.local/share/gnome-remote-desktop/tls.{key,crt}
~/.local/share/gnome-remote-desktop/certificates/
```

---

## Quick reference

| Task | Command |
|---|---|
| Check user-level status | `grdctl status` |
| Check system-level status | `sudo grdctl --system status` |
| Change user-level RDP password | `grdctl rdp set-credentials "user" "pass"` |
| Change system-level RDP password | `sudo grdctl --system rdp set-credentials "user" "pass"` |
| Change system-level port | `sudo grdctl --system rdp set-port 3390` |
| Restart user-level service | `systemctl --user restart gnome-remote-desktop.service` |
| Restart system-level service | `sudo systemctl restart gnome-remote-desktop.service` |
| Check listening ports | `ss -tlnp \| grep -E "3389\|3390"` |
| Check Tailscale | `tailscale status` |
| Test connectivity | `nc -zv your-host.your-tailnet.ts.net 3389` |

---

## References

- Ubuntu Desktop documentation: [Share your desktop remotely](https://documentation.ubuntu.com/desktop/en/latest/how-to/share-your-desktop-remotely/)
- Emile M.: [RDP Error Code 0x207 on Mac for Ubuntu 24](https://dev.to/emile1636/rdp-error-code-0x207-on-mac-for-ubuntu-24-d6d)
- Tailscale documentation: [MagicDNS](https://tailscale.com/kb/1081/magicdns)
- FreeRDP / WinPR project: [FreeRDP GitHub repository](https://github.com/FreeRDP/FreeRDP)

---

## Public safety note

This repository intentionally uses placeholder hostnames such as `your-host.your-tailnet.ts.net`. Do not publish your real tailnet name, Tailscale IP, Linux username, RDP username/password pair, or private hostnames.
