???- example "Пример"

    === "log"

        ```yaml linenums="1" hl_lines="2 8"
        "log": {
            "loglevel": "warning"  # (1)!
        }
        ```

        1.  "warning": информация, выводимая при возникновении проблем, не влияющих на нормальную работу, но которые могут повлиять на работу пользователя. Включает всю информацию уровня "error". 
        "error": Xray столкнулся с проблемой, которая не позволяет ему работать нормально, и ее необходимо немедленно решить.

    === "dns"

        ```yaml linenums="1" hl_lines="2 8"
        "dns": { 
            "servers": [ # 
            {
                "address": "https://dns.google/dns-query", # (1)!
                "skipFallback": false # (2)!
            }
            ],
        ```

        1. address — адрес DNS‑over‑HTTPS (DoH) сервера.

        2. **`skipFallback`** управляет тем, будет ли клиент использовать резервные (fallback) DNS‑серверы, если основной сервер не отвечает или работает с ошибками.

            - **`skipFallback: false`**  
            Клиент **не пропускает** fallback‑механику:  
            если основной DNS‑сервер недоступен, даёт сбой, слишком долго отвечает или возвращает ошибку, клиент **может переключиться на резервные DNS‑серверы** (системные, провайдерские или явно заданные как fallback).

            - **`skipFallback: true`**  
            Клиент **пропускает** fallback:  
            он жёстко привязан только к указанному DNS‑серверу и **не будет пытаться использовать другие DNS‑источники**, даже если основной сервер недоступен или работает с ошибками.



    === "inbounds"

        ```yaml linenums="1" hl_lines="2 8"
        "inbounds": [ 
            {
            "tag": "PUBLIPUBLIC_RU_IN",  # (1)!
            "port": 443, 
            "protocol": "vless",
            "settings": {
                "clients": [], 
                "decryption": "none" 
            },
            "sniffing": {                  
                "enabled": true,             
                "destOverride": [            
                "http",                       
                "tls",                        
                "quic"                        
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "reality",
                "realitySettings": {
                "dest": "/dev/shm/nginx.sock",
                "show": false,
                "xver": 1,
                "spiderX": "",
                "shortIds": [
                    "авиа"  # (2)!
                ],
                "privateKey": "#REPLACE_WITH_YOUR_PRIVATE_KEY, GENERATE BELOW IN PANEL ↓",
                "serverNames": [
                    "#REPLACE_WITH_YOUR_SERVER_NAMES, EXAMPLE: example.com"
                ]
                }
            }
            }
        ],
        ```

        1. Имя входящего подключения, по которому движок определяет, какие правила и маршруты к нему применять.
        
        2. Длина — 8 байт, то есть 16 шестнадцатеричных цифр (0-f), может быть меньше 16, ядро автоматически добавит 0 в конец, но количество цифр должно быть четным (потому что один байт состоит из 2 шестнадцатеричных цифр).
        Например, aa1234 будет автоматически дополнено до aa12340000000000, а aaa1234 приведет к ошибке.
        




    === "outbounds"

        ``` json linenums="1" hl_lines="2 3"
        "outbounds": [
            {
            "tag": "DIRECT",
            "protocol": "freedom"
            },
            {
            "tag": "BLOCK",
            "protocol": "blackhole"
            },
            {
            "tag": "VLESS_RU_OUT",
            "protocol": "vless",
            "settings": {
                "vnext": [
                {
                    "port": 443,
                    "users": [
                    {
                        "id": "XXXXXXXXXXXXXXXXXXXXXXXXXXX",
                        "flow": "xtls-rprx-vision",
                        "level": 0,
                        "encryption": "none"
                    }
                    ],
                    "address": "RAZ.cc"
                }
                ]
            },
            "streamSettings": {
                "network": "tcp",
                "security": "reality",
                "realitySettings": {
                "shortId": "XXXXXXXXXXXXXXXXXXX",
                "spiderX": "/",
                "publicKey": "XXXXXXXXXXXXXXXXXXXXXXXX",
                "serverName": "XXX.cc",
                "fingerprint": ""
                }
            }
            }
        ],
        ```
    === "routing"

        ``` json linenums="1" hl_lines="2 3"
        "routing": {
            "rules": [
            {
                "domain": [
                "ads.google.com",
                "doubleclick.net",
                "tracker.example.com",
                "adservice.google.com"
                ],
                "outboundTag": "BLOCK"
            },
            {
                "ip": [
                "geoip:private"
                ],
                "outboundTag": "BLOCK"
            },
            {
                "domain": [
                "geosite:private"
                ],
                "outboundTag": "BLOCK"
            },
            {
                "protocol": [
                "bittorrent"
                ],
                "outboundTag": "BLOCK"
            },
            {
                "ip": [
                "geoip:ru"
                ],
                "outboundTag": "DIRECT"
            },
            {
                "domain": [
                "geosite:category-ru"
                ],
                "outboundTag": "DIRECT"
            },
            {
                "domain": [
                "geosite:youtube",
                "domain:youtube.com",
                "domain:googlevideo.com",
                "domain:*.ytimg.com"
                ],
                "outboundTag": "DIRECT"
            },
            {
                "inboundTag": [
                "PUBLIPUBLIC_RU_IN"
                ],
                "outboundTag": "VLESS_RU_OUT"
            }
            ]
        }
        ```







