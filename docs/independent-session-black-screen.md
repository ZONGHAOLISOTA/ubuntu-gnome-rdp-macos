# Independent session black screen troubleshooting

This note covers a common failure mode for GNOME Remote Desktop remote-login sessions.

## Symptoms

- Screen-sharing / mirror mode works, or used to work.
- The system-level RDP endpoint on port `3390` reaches GDM.
- After logging in, the macOS RDP window becomes black.
- The physical session may be kicked back to GDM, or an old session may remain stuck in `closing` state.

## One common cause

One common cause is using the same Linux account for both the physical or mirror session and the independent remote-login session.

GNOME may need to tear down the old graphical session before starting the new one. On workstation-class machines, that handover can race with the old session teardown and fail to initialize the new headless rendering path. The result can be a black RDP screen rather than a clean `Session Already Running` prompt.

## Recommended fix

Create a dedicated Linux account for independent remote login and select that account at the GDM screen. Do not use the same account that is already logged into the physical or mirror session.

Example:

```bash
sudo adduser remote-user
```

## Diagnostics

```bash
loginctl list-sessions
loginctl show-user your-local-user | grep -E "State|Sessions"
journalctl --user -u gnome-remote-desktop.service -n 80 --no-pager
```

If an old session is stuck in `closing`, terminate that specific session after confirming it is safe:

```bash
loginctl terminate-session <session-id>
```

## GPU workstation reboot safety

If the machine uses NVIDIA drivers with OEM or rapidly moving kernels, verify that the kernel you will boot into has a matching NVIDIA module before rebooting.

```bash
uname -r
ls /lib/modules
find /lib/modules/$(uname -r) -name 'nvidia.ko*'
```

For an installed but not-yet-running kernel, replace the kernel version explicitly:

```bash
find /lib/modules/<kernel-version> -name 'nvidia.ko*'
```

If the module is missing, install the matching NVIDIA kernel-module package for that kernel before rebooting, or select a known-good kernel from GRUB.
