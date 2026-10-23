# Case study: backups, least privilege, and a live kiosk (October 2026)

## Situation

At the end of September the lab was tidy but exposed:

- The external SSD holding nightly guest backups had **failed**.
- The custom dashboard (HorcruX Imagined) could see services but not the hypervisor, so guest state and power still meant opening Proxmox.
- The physical kiosk showed a Grafana dashboard: useful, but static and hard to read from across a room.
- A chat channel kept receiving "down" alerts for a service retired weeks earlier.

## Goals

1. Backups on a dedicated drive again, with the same "never fill the system disk" guard, and visible on the dashboard.
2. Give the dashboard hypervisor visibility without giving it the keys to the hypervisor.
3. A kiosk that shows the lab *working*: live traffic, events, and alerts.
4. Alerts that only fire for things that exist.

## 1. Rebuilding the backup tier

**A correction first.** Earlier notes said there had been no backups since the SSD failed. Checking the job history showed otherwise: the nightly job had been writing to a temporary directory on the media SSD. Those copies sat on the same disk as the media they didn't protect, so they were a stopgap, but they existed. The notes were fixed before anything else changed.

| Step | Detail | Check |
|---|---|---|
| Identify the new disk | A new 2.5" 500 GB drive on USB, identified by model and size before touching it | SMART passed; zero power-on hours |
| Format | ext4 with a label; mounted by UUID | `findmnt` shows the expected device |
| Mount options | `nofail`, `x-systemd.automount`, 10-second device timeout | A missing drive can't stall host boot |
| Proxmox storage | Directory storage re-created with `is_mountpoint` | Storage goes inactive if the drive is missing, so nothing lands on the root filesystem |
| Job | Re-pointed; `keep-last=7` | Manual run of all five guests succeeded, about 40 GB in total |
| Old entry | The failed drive's `fstab` line commented out, with a dated copy of `fstab` | Clean boot |

Seven full sets come to about 280 GB, which fits on the drive with headroom.

**Showing it on the dashboard.** The dashboard's Proxmox token is read-only, and Proxmox requires write-level rights (`Datastore.AllocateSpace`, `VM.Backup`) just to *list* backup volumes. Rather than grant those to a web app, the collector reads vzdump **task logs**, which read-only audit rights can see, and parses each guest's start, finish, and archive size. The storage card now shows each guest's last archive and its age, and an overall state: ok, partial, degraded, stale, or unknown.

The first deploy showed "Unavailable": Proxmox returns retention values as strings, and Python refused to compare a string with an integer. One conversion fixed it.

## 2. Least-privilege hypervisor access

Imagined v2 got its own Proxmox user and a privilege-separated API token:

- **Read-only on everything** (`PVEAuditor` on `/`).
- **Power control on one VM only:** a custom role with `VM.PowerMgmt` on the on-demand 3D-printer VM, plus an allow-list in the app itself.
- **TLS pinned** to the Proxmox certificate's SHA-256 fingerprint instead of turning verification off.

**Gotcha:** a role granted on a deeper path *replaces* the inherited one for that path. With only `VM.PowerMgmt` there, the printer VM vanished from the dashboard. The custom role needs `VM.Audit` too.

Two upstream version quirks also showed up: the media server's newer release rejects the old token header and wants an `Authorization` header instead, and another service now answers a successful login with `204 No Content` rather than a body.

## 3. HorcruX Orbit: a live kiosk

Orbit is a 3D observatory that replaced the Grafana view on the kiosk. It renders guests, services, and the media library as a slowly rotating scene, with traffic shown as pulses along links.

**Design constraints:**

- **No credentials.** Orbit reads the dashboard's cached snapshot every 3 seconds and Prometheus every 2 seconds. It never talks to Proxmox directly.
- **Measured, not invented.** Per-container rates come from cAdvisor; per-guest rates from the host's `tap`/`veth` counters (host receive is guest transmit); uplink from the host NIC. Containers on the host network can't be separated from the VM's own traffic, so they're shown as not measurable.
- **Hardened.** Read-only filesystem and mounts, all capabilities dropped, 160 MB memory and 0.35 CPU limits, no Docker socket, tailnet-only bind.
- **Cheap to render.** Canvas, capped at 30 FPS, paused when hidden.

**Iterations:**

| Version | Change | Lesson |
|---|---|---|
| 1.0 | Observatory and kiosk mode | The kiosk had to join the tailnet, because Orbit binds only there |
| 1.1 | Live per-link traffic from Prometheus | A regex meant to pull a VM ID out of `tap100i0` also matched the bridge `vmbr0`. Anchored it. Default scrape intervals made the view lag, so only the cAdvisor and node-exporter jobs moved to 5 s |
| 1.2 | A typed event console (services up/down, guest state, playback, traffic surges, top talker) and an alert mode that frames the whole screen red or amber | The "top talker" line flapped between two guests. Hysteresis, and excluding the services VM that carries everyone's traffic, settled it. Alerts need two consecutive polls before they show |

**An idea that didn't ship:** ambient console sound. The kiosk's only audio path is the host's HDMI audio, which shares an IOMMU group with the host's own network adapter. Passing it through would have taken the hypervisor off the network, so the kiosk stays silent. The sound build only runs in a browser elsewhere, and only when asked for in the URL.

## 4. The ghost alert

The chat channel kept posting that the old start page was down. The secondary status monitor had no such check. The source was a **Grafana alert rule** still watching the retired probe. It's being retargeted to the dashboard that replaced it, and a second rule that may still watch the failed backup drive is under review.

**Lesson:** "retire a service" now means searching probes, dashboards, status-page checks, **and alert rules**.

## Other changes this month

- **DNS everywhere:** AdGuard became the tailnet's global nameserver, so every tailnet device is filtered at home and away. The services VM and the hypervisor opt out to avoid resolving through themselves. The ISP gateway can't hand out a custom DNS server, so covering non-tailnet devices will mean moving DHCP to AdGuard.
- **Media layout:** downloads and both libraries moved under one parent folder, so hard links work and the download stack can see only that folder.
- **Snapshots:** the thin pool refuses new snapshots above 70% use because the volume group has almost no room to grow. Old snapshots were cleared before each risky change.

## Results

| Before | After |
|---|---|
| Backup SSD failed; temporary copies on the media disk | Dedicated drive, guarded, 5/5 guests backed up, status on the dashboard |
| Dashboard blind to the hypervisor | Read-only view plus one-VM power control, TLS pinned |
| Static Grafana kiosk | Live observatory with an event console and alert mode |
| Alerts for a retired service | Source found; retirement checklist updated |

## Still open

- A run of clean nightly backups, then remove the temporary copies.
- A full restore rehearsal from the new drive into an unused VM ID.
- An offsite copy, including media.
- Tighten one monitoring listener that's currently bound on all interfaces.
