# Security and publication policy

This repository is a **sanitized public view** of a private homelab. It's meant to show design and operations without exposing the environment.

## Never published here

- Credentials, tokens, API keys, cookies, or recovery codes
- Private keys (SSH, TLS, VPN)
- Public or private IP addresses, tailnet names, internal hostnames, or domains
- Usernames, email addresses, serial numbers, MAC addresses, or location details
- Raw logs, database exports, backups, or unreviewed screenshots

Placeholders such as `<LAN_IP>`, `<SERVICE_HOST>`, and `<TRUSTED_DOMAIN>` stand in for private values.

## How it stays clean

- The operational repository is private. This one is curated from it, not a fork, so no private history carries over.
- Before every push: `git diff --staged`, plus a scan for IP-address patterns and personal identifiers.
- Screenshots are recreated or cropped and redacted before they're committed.

## Reporting

If you notice something here that looks sensitive, please open an issue without quoting the sensitive value, and it will be removed and rotated.
