# OlcRTC Manager Panel + WB Stream: установка

В гайде два пути:

- путь A: собрать два Linux-бинарника локально на Windows через Docker, скопировать на VPS и поставить руками;
- путь B: запустить автоустановщик панели на Ubuntu VPS, он сам соберет и поставит бинарники;
- проверка состояния и настройка OlcBox дальше одинаковые.

## Что нужно

- Для пути A: Windows с Docker Desktop, Git, `ssh` и `scp`.
- Ubuntu VPS с root-доступом, systemd и `amd64/x86_64` архитектурой.
- Живая WB Stream ссылка вида:

```text
https://stream.wb.ru/room/<ROOM_ID>
```

Дальше в командах замени:

```text
SERVER_IP=1.2.3.4
ROOM_ID=019f....
```

`ROOM_ID` это последняя часть ссылки после `/room/`.

## 1A. Путь A: собрать бинарники локально на Windows

В этом пути собираются два разных бинарника:

- `olcrtc-src-temp` -> `olcrtc`: сам туннель/сервер;
- `panel-src-temp` -> `olcrtc-manager`: веб-панель, API и supervisor, который запускает `olcrtc`.

Поэтому клонируются оба репозитория.

PowerShell:

```powershell
mkdir C:\olcrtc-manager-build
cd C:\olcrtc-manager-build

git clone --depth 1 --branch master https://github.com/openlibrecommunity/olcrtc.git olcrtc-src-temp
git clone --depth 1 --branch main https://github.com/BigDaddy3334/olcrtc-manager-panel.git panel-src-temp

New-Item -ItemType Directory -Force .\dist-bin\linux-amd64, .\.go-cache\pkg, .\.go-cache\build | Out-Null

$root = (Resolve-Path .).Path
docker run --rm `
  -v "${root}:/src" `
  -v "${root}/.go-cache/pkg:/go/pkg/mod" `
  -v "${root}/.go-cache/build:/root/.cache/go-build" `
  -w /src `
  golang:1.26-bookworm `
  bash -c "set -eux; export PATH=/usr/local/go/bin:/go/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin; go version; cd /src/olcrtc-src-temp; GOPROXY=https://proxy.golang.org,direct go mod download || GOPROXY=https://goproxy.io,direct go mod download || GOPROXY=direct go mod download; CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags='-s -w' -o /src/dist-bin/linux-amd64/olcrtc ./cmd/olcrtc; cd /src/panel-src-temp; GOPROXY=https://proxy.golang.org,direct go mod download || GOPROXY=https://goproxy.io,direct go mod download || GOPROXY=direct go mod download; CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build -trimpath -ldflags='-s -w' -o /src/dist-bin/linux-amd64/olcrtc-manager ./cmd/olcrtc-manager"
```

Проверка, что бинарники реально Linux amd64 и запускаются:

```powershell
$root = (Resolve-Path .).Path
docker run --rm -v "${root}:/src" -w /src golang:1.26-bookworm bash -c "set -e; /src/dist-bin/linux-amd64/olcrtc -h >/tmp/olcrtc-help.txt 2>&1 || true; /src/dist-bin/linux-amd64/olcrtc-manager -h >/tmp/manager-help.txt 2>&1 || true; head -5 /tmp/olcrtc-help.txt; echo '---'; head -5 /tmp/manager-help.txt"
```

Нормально, если `olcrtc -h` пишет примерно:

```text
usage: olcrtc <config.yaml>
```

А `olcrtc-manager -h` пишет:

```text
Usage of /src/dist-bin/linux-amd64/olcrtc-manager:
  -addr string
  -config string
```

## 1A.2. Скопировать бинарники на VPS

PowerShell:

```powershell
$SERVER_IP = "1.2.3.4"

scp .\dist-bin\linux-amd64\olcrtc .\dist-bin\linux-amd64\olcrtc-manager root@${SERVER_IP}:/tmp/
```

После этого переходи к разделу `2A. Установить панель руками на Ubuntu`.

## 1B. Путь B: автоустановщик панели на Ubuntu VPS

Используй этот путь, если хочешь собирать прямо на VPS. Скрипт сам поставит зависимости, Go, соберет `olcrtc`, соберет `olcrtc-manager`, создаст TLS, panel env и systemd service.

Зайди на сервер:

```powershell
ssh root@1.2.3.4
```

На Ubuntu:

```bash
set -euo pipefail

ROOM_ID="019f...."
PUBLIC_IP="1.2.3.4"

KEY="$(openssl rand -hex 32)"
ADMIN_USER="admin"
ADMIN_PASS="$(openssl rand -hex 16)"

echo "[1/5] stop old bare olcrtc service"
systemctl disable --now olcrtc.service 2>/dev/null || true
rm -f /etc/systemd/system/olcrtc.service
systemctl daemon-reload

echo "[2/5] precreate manager config"
install -d -m 0755 /etc/olcrtc-manager
install -d -m 0755 /var/lib/olcrtc/data

cat > /etc/olcrtc-manager/config.json <<JSON
{
  "version": 1,
  "name": "WB VPS",
  "port": 8888,
  "refresh": "10m",
  "clients": [
    {
      "client-id": "wb",
      "refresh": "5m",
      "locations": [
        {
          "name": "wb-vps",
          "endpoint": {
            "room_id": "${ROOM_ID}",
            "key": "${KEY}"
          },
          "carrier": "wbstream",
          "transport": {
            "type": "vp8channel",
            "payload": {
              "vp8-fps": "30",
              "vp8-batch": "64"
            }
          },
          "link": "direct",
          "data": "/var/lib/olcrtc/data",
          "dns": "8.8.8.8:53"
        }
      ]
    }
  ]
}
JSON

chmod 0600 /etc/olcrtc-manager/config.json

echo "[3/5] reset panel env for known admin credentials"
rm -f /etc/olcrtc-manager/panel.env

echo "[4/5] run panel installer"
curl -fsSL https://raw.githubusercontent.com/BigDaddy3334/olcrtc-manager-panel/main/scripts/install.sh -o /tmp/olcrtc-manager-install.sh

PANEL_PORT=8888 \
PANEL_ADDR=0.0.0.0 \
PANEL_ADMIN_PATH=/admin \
PANEL_CERT_IP="${PUBLIC_IP}" \
PANEL_PUBLIC_HOST="${PUBLIC_IP}" \
PANEL_ADMIN_USER="${ADMIN_USER}" \
PANEL_ADMIN_PASS="${ADMIN_PASS}" \
bash /tmp/olcrtc-manager-install.sh

echo "[5/5] status"
systemctl restart olcrtc-manager
sleep 6
systemctl --no-pager --full status olcrtc-manager | sed -n '1,35p'

echo
echo "Panel URL: https://${PUBLIC_IP}:8888/admin"
echo "Panel user: ${ADMIN_USER}"
echo "Panel pass: ${ADMIN_PASS}"
echo
echo "OlcBox URI:"
echo "olcrtc://wbstream?vp8channel<vp8-batch=64&vp8-fps=30>@${ROOM_ID}#${KEY}\$wb-vps"
```

После этого переходи к разделу `3. Проверить состояние через API панели`.

## 2A. Установить панель руками на Ubuntu

Зайди на сервер:

```powershell
ssh root@1.2.3.4
```

На сервере выполни блок ниже. Замени `ROOM_ID` и `PUBLIC_IP`.

```bash
set -euo pipefail

ROOM_ID="019f...."
PUBLIC_IP="1.2.3.4"

KEY="$(openssl rand -hex 32)"
ADMIN_USER="admin"
ADMIN_PASS="$(openssl rand -hex 16)"

echo "[1/7] stop old services"
systemctl disable --now olcrtc.service 2>/dev/null || true
systemctl stop olcrtc-manager.service 2>/dev/null || true
pkill -TERM -f '/usr/local/bin/olcrtc-manager|/usr/local/bin/olcrtc ' 2>/dev/null || true
sleep 1
rm -f /etc/systemd/system/olcrtc.service
systemctl daemon-reload

echo "[2/7] install runtime packages"
export DEBIAN_FRONTEND=noninteractive
apt-get update
apt-get install -y --no-install-recommends ca-certificates curl iproute2 iptables openssl

echo "[3/7] install binaries"
test -s /tmp/olcrtc
test -s /tmp/olcrtc-manager
install -m 0755 /tmp/olcrtc /usr/local/bin/olcrtc
install -m 0755 /tmp/olcrtc-manager /usr/local/bin/olcrtc-manager
/usr/local/bin/olcrtc-manager -h >/dev/null
/usr/local/bin/olcrtc -h >/dev/null 2>&1 || true

echo "[4/7] write manager config"
install -d -m 0755 /etc/olcrtc-manager
install -d -m 0755 /var/lib/olcrtc/data

cat > /etc/olcrtc-manager/config.json <<JSON
{
  "version": 1,
  "name": "WB VPS",
  "port": 8888,
  "refresh": "10m",
  "clients": [
    {
      "client-id": "wb",
      "refresh": "5m",
      "locations": [
        {
          "name": "wb-vps",
          "endpoint": {
            "room_id": "${ROOM_ID}",
            "key": "${KEY}"
          },
          "carrier": "wbstream",
          "transport": {
            "type": "vp8channel",
            "payload": {
              "vp8-fps": "30",
              "vp8-batch": "64"
            }
          },
          "link": "direct",
          "data": "/var/lib/olcrtc/data",
          "dns": "8.8.8.8:53"
        }
      ]
    }
  ]
}
JSON

chmod 0600 /etc/olcrtc-manager/config.json

echo "[5/7] write panel auth and tls"
openssl req -x509 -nodes -newkey rsa:2048 -sha256 -days 825 \
  -keyout /etc/olcrtc-manager/tls.key \
  -out /etc/olcrtc-manager/tls.crt \
  -subj "/CN=olcrtc-manager" \
  -addext "subjectAltName=IP:${PUBLIC_IP},DNS:localhost" >/dev/null 2>&1

chmod 0600 /etc/olcrtc-manager/tls.key
chmod 0644 /etc/olcrtc-manager/tls.crt

cat > /etc/olcrtc-manager/panel.env <<EOF
OLCRTC_MANAGER_USER='${ADMIN_USER}'
OLCRTC_MANAGER_PASS='${ADMIN_PASS}'
OLCRTC_MANAGER_ADMIN_PATH='/admin'
OLCRTC_MANAGER_TLS_CERT='/etc/olcrtc-manager/tls.crt'
OLCRTC_MANAGER_TLS_KEY='/etc/olcrtc-manager/tls.key'
EOF

chmod 0600 /etc/olcrtc-manager/panel.env

echo "[6/7] install systemd service"
cat > /etc/systemd/system/olcrtc-manager.service <<'EOF'
[Unit]
Description=OlcRTC Manager Panel
Documentation=https://github.com/BigDaddy3334/olcrtc-manager-panel
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment=OLCRTC_PATH=/usr/local/bin/olcrtc
EnvironmentFile=-/etc/olcrtc-manager/panel.env
ExecStart=/usr/local/bin/olcrtc-manager -addr 0.0.0.0 -config /etc/olcrtc-manager/config.json
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=5s
KillSignal=SIGTERM
TimeoutStopSec=10s

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now olcrtc-manager
systemctl restart olcrtc-manager
sleep 6

echo "[7/7] status"
systemctl --no-pager --full status olcrtc-manager | sed -n '1,35p'

echo
echo "Panel URL: https://${PUBLIC_IP}:8888/admin"
echo "Panel user: ${ADMIN_USER}"
echo "Panel pass: ${ADMIN_PASS}"
echo
echo "OlcBox URI:"
echo "olcrtc://wbstream?vp8channel<vp8-batch=64&vp8-fps=30>@${ROOM_ID}#${KEY}\$wb-vps"
```

Открой:

```text
https://SERVER_IP:8888/admin
```

Браузер будет ругаться на self-signed TLS-сертификат. Это нормально, если ставишь без домена и Let's Encrypt.

## 3. Проверить состояние через API панели

На сервере:

```bash
ROOM_ID="019f...."
PASS="$(sed -n "s/^OLCRTC_MANAGER_PASS='\(.*\)'/\1/p" /etc/olcrtc-manager/panel.env)"

curl -sk -u "admin:${PASS}" "https://127.0.0.1:8888/api/state"
curl -sk -u "admin:${PASS}" "https://127.0.0.1:8888/api/logs/?client_id=wb&room_id=${ROOM_ID}&transport=vp8channel"
```

Нормальный старт выглядит так:

```text
status: running
Connecting transport=vp8channel carrier=wbstream ...
vp8channel: KCP started ...
Link connected
Current peers count: 0
```

Если в логах:

```text
status 403: {"code":7,"message":"guests cannot create rooms","details":[]}
```

значит WB-комната уже закрыта или неактивна. Создай новую комнату WB, возьми новый `ROOM_ID`, обнови конфиг панели и перезапусти сервис.

## 4. Обновить комнату WB

Самый простой путь: зайти в панель, открыть клиента `wb`, локацию `wb-vps`, поменять `Room` и нажать restart.

Если делаешь через SSH, не генерируй новый `KEY`, иначе OlcBox надо будет перенастраивать. Перепиши `/etc/olcrtc-manager/config.json` тем же JSON-блоком из раздела `1B` или `2A`, но оставь старый `"key"` и поменяй только:

```bash
ROOM_ID="новый-room-id"
```

После изменения:

```bash
systemctl restart olcrtc-manager
sleep 6
systemctl --no-pager --full status olcrtc-manager | sed -n '1,35p'
```

Потом снова проверь API из раздела 3.

## 5. Что ставить в OlcBox

Самый простой способ: скопировать URI, который вывел установочный блок:

```text
olcrtc://wbstream?vp8channel<vp8-batch=64&vp8-fps=30>@ROOM_ID#KEY$wb-vps
```

Если вводишь руками:

```text
Service: WB Stream
Transport: VP8
FPS: 30
Batch: 64
Room ID: ROOM_ID
Encryption key: KEY
```

## 6. Итоговая схема

В итоге получается такая схема:

- `olcrtc` собирается из `openlibrecommunity/olcrtc` ветки `master`;
- `olcrtc-manager` собирается из `BigDaddy3334/olcrtc-manager-panel` ветки `main`;
- на сервер попадают два бинарника: `/usr/local/bin/olcrtc` и `/usr/local/bin/olcrtc-manager`;
- systemd service `olcrtc-manager.service`;
- TLS на `https://SERVER_IP:8888/admin`;
- WB Stream через `vp8channel`, `vp8-fps=30`, `vp8-batch=64`;
- проверка `api/state` и `api/logs`.
