# Operations

## Change workflow

Every change follows the same loop:

1. **Verify live state.** Never trust old notes.
2. **Snapshot** the affected guest (`qm snapshot <id> pre_<change>_<date>`).
3. **Back up** the file being changed, next to it or in `_archive/`.
4. **Make one change.**
5. **Validate** it with the tool's own checker (`docker compose config -q`, `promtool check config`).
6. **Verify** the service and its dependents (terminal deck, Grafana, direct checks).
7. **Record** the result, and sanitize anything that goes public.

Rollback commands are prepared ahead of time but kept as a separate step, so they're never pasted along with the forward change.

## Maintenance schedule

| Frequency | Task |
| --- | --- |
| Daily | Glance at the Grafana command center (timeline and 7-day uptime) |
| Weekly | Run the terminal control deck; review container restarts, failed units, disk capacity |
| Monthly | Update the host and every guest; review snapshots, storage, Tailscale devices, documentation drift |
| Quarterly | Restore one guest into an unused VM ID; restore one application dataset in isolation |
| After changes | Reboot rehearsal where relevant; refresh documentation; export the Grafana dashboard |

## Snapshots

Snapshots are rollback points on the same disk. They are **not** backups. Each guest keeps a small set of named known-good points (fresh baseline, stable core, post-update). Change-specific snapshots are deleted once a reboot rehearsal proves the change.

## Backup strategy

| Tier | Target | Status |
| --- | --- | --- |
| Guest archives | Nightly `vzdump` of all guests to a dedicated external SSD, `keep-last=7`, with a mount guard so a missing disk can't fill the system drive | **Degraded:** the backup SSD failed in September 2026; replacement pending |
| Configuration | Compose files and configs live inside the services VM, so guest archives cover them | Depends on tier 1 |
| Media | External media SSD, not included in guest archives | Needs an offsite/full copy |
| Offsite | Not yet in place | Top resilience gap |

The mount guard worked as designed when the drive failed: Proxmox marked the storage inactive instead of silently writing backups onto the root filesystem.

## Recovery rehearsals

| Date | Exercise | Result |
| --- | --- | --- |
| 2026-09-13 | Restored three guests from verified archives while renumbering guest IDs | All services validated |
| 2026-09-25 | Services VM cold reboot after the reorganization | 17/17 containers and every endpoint recovered unattended; DNS verified |

## Open work

- [ ] Replace the failed backup SSD, rebuild guarded backup storage, verify a full nightly run
- [ ] Offsite / full-scope backup, including media
- [ ] Full restore rehearsal into an unused VM ID
- [ ] Repair the secondary status monitor and reconcile its checks
- [ ] Update the remaining guests
- [ ] Pin floating container image tags to tested versions
- [ ] UPS with graceful shutdown
- [ ] 32 GB memory upgrade
