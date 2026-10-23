# Runbook: convert a `docker run` container to Compose (with instant rollback)

Used to move LAN DNS (AdGuard Home) under Compose. DNS is critical, so the design keeps the old container intact until the new one is proven.

## 1. Capture the live definition (read-only)

```bash
sudo docker inspect <container> --format \
 'image={{.Config.Image}}{{"\n"}}ports={{json .HostConfig.PortBindings}}{{"\n"}}restart={{.HostConfig.RestartPolicy.Name}}{{"\n"}}{{range .Mounts}}{{.Type}}: {{.Source}} -> {{.Destination}}{{"\n"}}{{end}}caps={{json .HostConfig.CapAdd}} env={{json .Config.Env}}'
```

## 2. Write the Compose file

Match the image tag, port bindings (including the bind IP), restart policy, and mounts exactly. Use relative paths for the new data location.

```yaml
name: adguard
services:
  adguardhome:
    image: adguard/adguardhome:<pinned-tag>
    container_name: adguardhome        # same name keeps monitoring and scripts working
    restart: unless-stopped
    ports:
      - "<LAN_IP>:53:53/tcp"
      - "<LAN_IP>:53:53/udp"
      - "<LAN_IP>:8080:80/tcp"
    volumes:
      - ./conf:/opt/adguardhome/conf
      - ./work:/opt/adguardhome/work
```

## 3. Cut over (about a minute of DNS downtime)

```bash
sudo docker stop <container> && sudo docker rename <container> <container>-old
sudo cp -a /old/data/conf /old/data/work /opt/<lab>/adguard/      # copy, don't move
cd /opt/<lab>/adguard && sudo docker compose up -d
```

## 4. Verify

```bash
sudo docker ps -a --filter name=adguard --format '{{.Names}} | {{.Status}}'
dig @<LAN_IP> example.com +short
```

## 5. Rollback (only if DNS fails)

```bash
cd /opt/<lab>/adguard && sudo docker compose down
sudo docker rename <container>-old <container> && sudo docker start <container>
```

## 6. Clean up after a few stable days

Remove `<container>-old` and archive the original data folder.

> If you copy the data and then roll back, the copy goes stale. Delete it and copy again right before the next attempt.
