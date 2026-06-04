# Security Policy

This repository is a public guide for GNOME Remote Desktop configuration. Do not open issues or pull requests that include private infrastructure details.

## Do not publish

- Real Tailscale tailnet names
- Real Tailscale IP addresses
- Private hostnames
- Linux usernames tied to private hosts
- RDP usernames and passwords
- Exported `.rdp` files that contain real `full address` values

Use placeholders such as:

```text
your-host.your-tailnet.ts.net
remote-user
CHANGE_ME_STRONG_PASSWORD
```

## Reporting a sensitive exposure

If you notice that this repository accidentally contains private hostnames, tailnet identifiers, IP addresses, or credentials, please open a private security advisory if available, or contact the repository owner directly instead of posting the sensitive value in a public issue.
