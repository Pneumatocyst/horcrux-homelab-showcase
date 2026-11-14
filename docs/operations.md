# Operations

**November 13 hardware hold:** The independent PBS integration is configured, but its disk returned uncorrectable reads and eight pending sectors during initial verification. PBS is read-only and its new jobs are paused; use the unchanged USB tier for recovery. The host memory upgrade is reflected in a 31.2 GiB live reading. See the [backup-server record](audits/2026-11-13-backup-server.md).

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

Verified November 12, 2026:

| Tier | Scope | Current evidence and limits |
| --- | --- | --- |
| Guest archives | Nightly backup of guests 100, 101, 200, 300, 400; keep last five per guest | Latest job completed for all five; 25 retained archives total about 225 GiB. The stopped experimental guest is excluded. |
| Selected external data | Daily hardlink snapshots; retain 14 dated snapshots | Nextcloud data, music, Windows backup folder, and application/database exports; latest run succeeded. Movies, TV, and downloads are excluded. |
| Storage guard | Dedicated backup drive with mount-point guard | Active and roughly 73% used, about 102 GiB available at audit time. Both backup tiers share this drive. |
| Independent on-site destination | Separate physical PBS host; planned 23:00 guests plus selected data, retain three | Initial transfers completed; disk failed verification. Read-only, jobs paused, copies untrusted. |
| Offsite/disconnected copy | Separate site or disconnected media | Not verified; remains open. |

A successful archive or export does not establish restore consistency. A Windows backup folder does not prove a usable image chain. The hardlink snapshot sizes must not be summed as independent full copies. Imagined now reads a root-published retained-archive inventory, reports all USB generations, and distinguishes PBS jobs so they cannot replace the USB result. The earlier roughly 44 GiB latest-generation estimate is superseded.

## Recovery rehearsals

Historical records describe guest restores in September 2026 and a services-VM reboot rehearsal. This audit did not repeat those exercises. A restore from the current backup drive and an isolated application/database restore remain unverified.

## Open work

- [x] Rebuild guarded guest backup storage and complete recent nightly runs.
- [x] Add selected external-data backups with application/database exports.
- [x] Retire Portainer, Uptime Kuma, and Sentinel from the live stack.
- [ ] Diagnose the host SMART daemon startup timeout; extend disk-health coverage beyond the system NVMe.
- [ ] Restore a guest and an application dataset from the current backup drive into isolation.
- [x] Provision an independent physical on-site PBS destination.
- [ ] Replace faulty PBS storage, reseed all groups and pass full integrity verification before resuming schedules.
- [ ] Add offsite/disconnected protection and decide whether full media needs protection.
- [ ] Reconcile live deployment files and Compose projects with the infrastructure repository.
- [ ] Correct Imagined's retained-archive storage estimate and remove its retired Portainer launcher.
- [ ] Review 12 containers without healthchecks, 10 without memory limits, and 14 without NanoCPU limits. Other CPU quota mechanisms were not audited.
- [ ] Review older wildcard listeners, probe coverage for newer services, and floating image tags.
- [ ] Revalidate historical alert-rule cleanup and update status before making maintenance changes.
- [ ] Consider a UPS and memory upgrade after measuring workload pressure.
