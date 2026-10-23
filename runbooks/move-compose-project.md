# Runbook: move a Docker Compose project to a new folder

Use this to relocate a project (for example from a home directory into `/opt/<lab>/<service>`) without renaming its networks or volumes.

## Before you start

- Relative bind mounts (`./data`) move with the folder. Absolute paths do not; check them.
- The Compose **project name** defaults to the folder name. Named volumes and networks are prefixed with it. Pin the old name so nothing is recreated empty.
- Take a VM snapshot on the hypervisor first.

```bash
grep -n "home/\|/opt/\|volumes:\|name:\|build" ~/old-project/compose.yml
grep -rln "old-project" /opt/<lab>/scripts /etc/systemd/system 2>/dev/null   # anything else pointing at the old path?
```

## Steps

```bash
cd ~/old-project && sudo docker compose down && cd ~
sudo mv ~/old-project /opt/<lab>/<service>
cd /opt/<lab>/<service>
sudo mkdir -p /opt/<lab>/_archive/<service>-old
sudo cp compose.yml /opt/<lab>/_archive/<service>-old/compose.yml.orig
sudo sed -i '1i name: old-project' compose.yml      # pin the original project name
head -3 compose.yml
sudo docker compose up -d
sudo docker compose ps
```

## Verify

- Every container is `Up` (and `healthy` where there's a healthcheck).
- The app's health endpoint answers.
- Monitoring shows the service UP.

## Rollback (only if verification fails)

```bash
cd /opt/<lab>/<service> && sudo docker compose down
sudo cp /opt/<lab>/_archive/<service>-old/compose.yml.orig compose.yml
sudo mv /opt/<lab>/<service> ~/old-project && cd ~/old-project && sudo docker compose up -d
```
