# Runbooks

Procedures used in real changes to HORCRUX. Placeholders like `<LAN_IP>` and `<lab>` stand in for private values.

| Runbook | Use it when |
|---|---|
| [Move a Compose project](move-compose-project.md) | Relocating a service folder without renaming its networks or volumes |
| [Convert `docker run` to Compose](convert-docker-run-to-compose.md) | Bringing a hand-started container under Compose with instant rollback |
| [Reboot rehearsal](reboot-rehearsal.md) | Proving the services VM recovers unattended after a change |
| [Add a Blackbox probe](../docs/monitoring.md#safe-change-procedure) | Monitoring a new service |

Every runbook follows the same pattern: capture state, snapshot, make one change, verify, and keep rollback as a separate step.
