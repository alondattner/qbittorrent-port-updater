# qBittorrent Port Updater

Automatically updates qBittorrent’s listening port to match the forwarded port from **Gluetun + ProtonVPN**.

## Usage

### 1. Create the script

Save as `update-qbit-port.sh`:

```sh
#!/bin/sh

QBT_USER={QBITTORRENT_USERNAME}
QBT_PASS={QBITTORRENT_PASSWORD}
QBT_PORT={QBITTORRENT_PORT}
GLT_USER={GLUETUN_USERNAME}
GLT_PASS={GLUETUN_PASSWORD}

while true; do
  while true; do
    PORT=$(curl -s -u ${GLT_USER}:${GLT_PASS} http://gluetun:8000/v1/openvpn/portforwarded | jq -r .port)
    [ "$PORT" -gt 0 ] 2>/dev/null && break
    sleep 2
  done

  curl -s --retry 30 --retry-delay 5 --data "username=${QBT_USER}&password=${QBT_PASS}" \
    --cookie-jar /tmp/qb-cookies.txt http://gluetun:${QBT_PORT}/api/v2/auth/login

  curl -s --retry 30 --retry-delay 5 \
    --data 'json={"listen_port": "'"$PORT"'"}' \
    --cookie /tmp/qb-cookies.txt \
    http://gluetun:${QBT_PORT}/api/v2/app/setPreferences

  sleep 900
done
````

Make it executable:

```
chmod +x update-qbit-port.sh
```

### 2. Add the Docker Compose service

```yaml
services:
  port-updater:
    image: alpine:latest
    container_name: port-updater
    volumes:
      - ./update-qbit-port.sh:/update-qbit-port.sh
    depends_on:
      - gluetun
      - qbittorrent
    command: >
      sh -c '
        apk add --no-cache curl jq >/dev/null;
        sh /update-qbit-port.sh
      '
    networks:
      default:
        aliases:
          - gluetun
          - qbittorrent
    restart: unless-stopped
```

Place the script in the same folder as your `docker-compose.yml`.

## Done

Start your stack. qBittorrent will always use the correct forwarded port.
