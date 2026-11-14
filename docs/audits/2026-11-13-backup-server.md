# Backup server integration and hardware hold — November 13, 2026

**Commissioning is blocked by a confirmed storage read fault. The new PBS copies are untrusted.** Software integration is configured, but the datastore is read-only and its new backup/maintenance schedules are paused. Existing USB jobs remain enabled and unchanged.

## Evidence and outcome

The independent physical PBS 4.2 host has 16 GiB RAM and a 320 GB HDD (about 298 GiB raw). Its datastore shares the OS filesystem; about 252 GiB was initially available. Setup did not format a disk.

All five initial guest transfers completed at 18:18 Central. Selected-data transfer completed at 19:07, with sources totaling about 101.25 GiB. Six groups existed and approximately 113.7 GiB remained available. A CT101 configuration readback succeeded. A small host-configuration archive restored to a private scratch directory; its hostname checksum matched and the restored hypervisor SQLite database passed its integrity check.

Full verification started automatically afterward. PBS then encountered an uncorrectable HDD read, quarantined an unreadable chunk, and failed CT101 filesystem verification. The kernel reported a media read error; SMART pending sectors rose from zero to **eight**. Its overall SMART summary still said PASSED. Reallocated sectors remained zero and historical interface CRC errors remained two. These read errors take precedence over the summary label.

Verification was stopped. The new nightly PBS job was disabled, the datastore placed in read-only maintenance, pruning disabled, and GC/weekly verification unscheduled. No snapshots were deleted and no disk was formatted. This is separate from the repaired November 12 SMART-daemon configuration problem on the compute host.

## Verified integration

- Native Proxmox storage uses a scoped backup identity and pinned TLS certificate.
- Imagined explicitly displays **Disk fault - backups paused**, untrusted-copy status, failed/pending verification, capacity and fresh metadata. Copied groups are not labeled protected.
- Harry reports the disk fault directly from metadata, labeled no AI inference, and cannot call a hardware-held copy healthy.
- Prometheus exposes the hardware hold, verification-failure count and job enablement. Grafana includes a prominent hardware-fault banner.
- Private HTTPS administration uses the existing services VM's Tailscale gateway and a loopback proxy with validated upstream TLS. Remote management depends on that VM; LAN administration remains available. No Funnel was enabled.
- Matching PBS configuration recovery copies were saved in root-only hypervisor and USB folders after explicit approval. They were refreshed to preserve the paused/read-only configuration.
- All 23 existing containers were running, all 15 configured healthchecks passed, and all retained CPU/memory limits remained present.
- Twenty Python regression tests, three frontend health-assessment checks and JavaScript syntax validation passed. The actual live snapshot correctly produced **Needs attention** for the overview while all 15 active services remained up; generic Harry health reads also identified the PBS fault.
- The compute host now reports about 31.2 GiB usable RAM, reflecting the previously planned 32 GB upgrade.

## Policy and current state

| Work | Intended Central schedule | Current state |
| --- | --- | --- |
| Existing USB guest backups, retain five | 21:00 | Enabled, unchanged; latest job OK |
| Existing selected-data USB snapshots, retain 14 dates | 22:30 | Unchanged |
| New PBS guest backups, retain three | 23:00 | Disabled after disk fault |
| New PBS selected data | After guest job | Initial upload completed; untrusted, no further uploads while held |
| PBS prune | 03:00 daily | Disabled |
| PBS garbage collection | 12:00 daily | Unscheduled |
| PBS verification | Sunday 06:00 | Unscheduled after failed initial check |

Selected data includes Nextcloud data, music, the Windows backup directory, application/database exports and hypervisor recovery configuration. Raw movie, TV and download libraries remain excluded. Music and the Windows backup directory were empty at inspection; a valid Windows image chain is not established. A 45 GiB start floor and 40 GiB runtime free-space floor were configured, but do not repair media faults or guarantee future capacity.

## Required next work

1. Replace and validate the faulty storage, preferably providing a larger dedicated datastore. Confirm the physical target before any formatting.
2. Reseed all groups and pass full verification, then rehearse isolated guest/application restores.
3. Only after validation, deliberately clear read-only maintenance and the persistent hardware hold and restore the intended schedules. An older hypervisor recovery snapshot predates the hold; review restored job settings.
4. Validate offsite/disconnected protection, notification delivery, the Windows recovery-image chain and media-SSD SMART coverage.
5. Select a valid PBS update channel after storage replacement. The optional repository/package/Tailscale/power-settings installer was rejected by automatic approval review and did not run.
6. Complete deployment-source reconciliation. The earlier candidate predates PBS integration; operational source was not published by this documentation change.

A separate current dashboard/proxy configuration supplement was initially blocked as an additional sensitive payload. After explicit approval, all 13 files were captured and checksum-verified at 20:20 Central, with matching root-only archive/manifest copies on the hypervisor and its existing USB backup disk. Credential files were excluded. The PBS copy remains deferred under the unchanged read-only hardware hold. The first services-VM seed predates that integration; the existing nightly USB job is scheduled to capture the current live state. Secret values, private addresses, hostnames and raw logs are omitted here.
