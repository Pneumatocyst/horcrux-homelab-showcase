# Case study: simplifying HORCRUX (September 2026)

## Situation

After a month of fast growth, the lab worked, but it had drifted:

- Two dashboards (a generic start page and a custom one) did overlapping jobs.
- Service folders were split between `/opt/horcrux` and a user's home directory.
- DNS ran from a one-off `docker run` with no Compose file, so its configuration existed only inside Docker.
- A printer VM booted automatically and held memory it rarely used.
- After a host update, two guests didn't come back on their own.
- Monitoring still probed the retired dashboard and missed several real services.

Then the external backup SSD failed.

## Goals

1. One dashboard, one folder layout, one way to operate every service.
2. Dependency-ordered boot, with on-demand guests truly on demand.
3. Monitoring that covers every user-facing service and tells the truth.
4. No surprise downtime, and a rollback for every step.

## Approach

Each move followed the same pattern: **snapshot → stop → move → pin → start → verify → archive**.

| Step | Change | Verification |
|---|---|---|
| Updates | Host and services VM patched, then the host rebooted | Fixed autostart; restored a start-page regression caused by stricter host validation |
| Dashboard consolidation | Retired the generic start page; the custom dashboard (HorcruX Imagined) became the single entry point | Terminal status script repointed; `/healthz` 200 |
| Folder moves | Moved two home-folder projects into `/opt/horcrux/<service>`, with a top-level `name:` so network and volume names didn't change | Containers up; logs clean |
| DNS to Compose | Wrote a Compose file matching the live container (image tag, bindings, volumes). **Copied** the data rather than moving it, kept the old container stopped under a new name, then cut over | Real DNS lookups against the new container; the old one was kept for instant rollback |
| Boot layout | Printer VM set to manual start; order set to services, then AI, then kiosk, then workstation; workstation memory ballooning enabled | About 8 GiB free with everything running |
| Cleanup | Old compose copies, backups, and home-folder leftovers moved into `_archive/`; script ownership fixed | Tidy tree; nothing deleted |
| Rehearsal | Cold reboot of the services VM | 17/17 containers and every endpoint recovered unattended |
| Monitoring | Retired probe replaced; added a real DNS probe, a `401`-means-healthy probe for the token-protected AI bridge, and a maintenance-aware Nextcloud probe over an internal Docker network | Every probe UP except the printer (expected OFF) |
| Dashboards | Grafana gained a service timeline, 7-day uptime tiles, thresholds, clean legends, and branding. Exported to JSON. | Visual review on the kiosk |
| Docs | Updated the sanitized repo, wrote a private state record, and built a printable binder | Automated scan: no private addresses in public files |

## Problems hit, and what fixed them

| Problem | Root cause | Fix / lesson |
|---|---|---|
| `&&` "is not a valid statement separator" | The command was typed in Windows PowerShell instead of the Linux VM | **Check the prompt before every paste**; the binder now opens with a machine/prompt table |
| A move "worked" and then un-did itself | The rollback block was pasted along with the forward steps | Rollback commands now live in a separate message and step |
| `Permission denied` backing up into the archive | The archive folder was root-owned | Use `sudo`, or give the operator ownership of script folders |
| Nextcloud probe returned `400` | Nextcloud rejects untrusted `Host` headers | Send a trusted `Host` header; the probe also catches maintenance mode |
| Probe couldn't reach Nextcloud at all | The backend listens on loopback inside the VM | Attach Blackbox to the app's internal Docker network instead of publishing a port |
| Status tile showed a retired service after the probe changed | The Grafana panel query was hard-coded to that service | Search the dashboard JSON for stale names after any rename |
| PowerShell 7 JSON export failed in a helper tool | `@(List[object])` inside an `[ordered]` literal raised "Argument types do not match" | Copy lists to plain arrays before building objects (fixed upstream in the toolkit) |

## Outcome

- One dashboard, one directory convention, 9 Compose projects, 17 containers.
- Boot order that matches dependencies, and on-demand guests that stay off.
- 13 black-box probes, a service timeline, and 7-day availability tiles.
- A reboot-proven layout, with a rollback path for every change still on disk.
- Clear, honest status on the one real gap: backups wait on replacement hardware.

## What's next

Replace the backup SSD and verify nightly archives, then add an offsite copy, run a full restore rehearsal, and pin image versions.
