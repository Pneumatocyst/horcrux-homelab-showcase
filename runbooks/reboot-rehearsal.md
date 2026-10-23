# Runbook: services VM reboot rehearsal

Proves that every container and endpoint comes back without a human after a restart. Run it after any structural change.

## Before

- Pick a quiet window. LAN DNS is down for about 2 minutes.
- Record the baseline: the terminal status deck and Grafana service timeline should both be green.

## Steps

```bash
sudo reboot
# wait ~2 minutes, reconnect over SSH
/opt/<lab>/scripts/<status-deck>.sh          # expect N/N containers running and every endpoint UP
dig @<LAN_IP> example.com +short             # DNS answering
```

From another device, load a website (proves DNS end to end) and open the dashboard.

## Pass criteria

- Container count matches the baseline.
- Every HTTP/DNS probe returns UP within 5 minutes.
- No containers are restarting in a loop (`docker ps` shows a steady `Up X minutes`).

## After

- Delete the change-specific snapshots now that the change is proven.
- Record the result in the operations log.
