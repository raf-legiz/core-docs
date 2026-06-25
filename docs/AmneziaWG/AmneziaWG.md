## Настраиваем сервер 
 ```bash
    apt-get update - обновление пакетов
 ```
## Устанавливаем необходимые компоненты для кеомпиляции

 ```bash
    apt-get install -y git libelf-dev linux-headers-$(uname -r) build-essential pkg-config
 ```
## Производим установку ядра amneziaWG
 ```bash
    git clone --single-branch --depth=1 https://github.com/amnezia-vpn/amneziawg-linux-kernel-module.git
 ```
 ```bash
    git clone --single-branch --depth=1 https://github.com/amnezia-vpn/amneziawg-tools.git
 ```
## КОмпиляция 
 ```bash
    make -C amneziawg-linux-kernel-module/src -j$(nproc)
 ```
 ```bash
    make -C amneziawg-linux-kernel-module/src install
 ```

## Установка
 ```bash
    make -C amneziawg-tools/src -j$(nproc)
 ```
 ```bash
    make -C amneziawg-tools/src install
 ```


Поднимаем WG 

Создаем интерфейс AmneziaWG 

```bash
    ip link  add awg0 type amneziawg
```
 Поднимаем его
```bash
    ip link  set awg0 up
```


Добавляем рандомный IP 

```bash
    ip addr add 10.47.0.1/24 dev awg0
```

Проверяем, должно появиться что то аналогичное 
```bash
    ip a
```

```console hl_lines="16-19"
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq state UP group default qlen 1000
    link/ether 52:54:00:07:23:91 brd ff:ff:ff:ff:ff:ff
    altname enp0s3
    inet 31.77.160.162/32 brd 31.77.160.162 scope global ens3
       valid_lft forever preferred_lft forever
    inet6 2a12:bec4:1bb0:2792::2/64 scope global
       valid_lft forever preferred_lft forever
    inet6 fe80::5054:ff:fe07:2391/64 scope link
       valid_lft forever preferred_lft forever
3: awg0: <POINTOPOINT,NOARP,UP,LOWER_UP> mtu 1420 qdisc noqueue state UNKNOWN group default qlen 1000
    link/none
    inet 10.47.0.1/24 scope global awg0
       valid_lft forever preferred_lft forever
```



## СОздаем конфигурационный файл 

```bash
    nano /etc/amnezia/awg0.conf
```

И вставляем сюда следующий код 



=== "Приложение на стороне сервера"
    ```ini
    [Interface]
    PrivateKey = oMnb3bDJuhouU5+YRwK2cfrey+DEQSNX3OH5BvWJt3g=
    ListenPort = 12048

    [Peer]
    PublicKey = Cyxi3ECRBSTVdZXM9CIcGYTVjkQo2WiAzdVryCQhuAM=
    PresharedKey = 2PeZAlcWuRVfqTpNDsdGLejkwNeUNXlyTG7EVSB482E=
    AllowedIPs = 10.47.0.2/32
    ```
    ??? warning "Генерируем ключи"
        Генерируем ключь для PrivateKey, след. командой PrivateKey - генерация
        ```bash 
        awg genkey
        ```
        Генерируем ключь для PresharedKey, след. командой PresharedKey - генерация
        ```bash 
        awg genpsk
        ```
    Применяем данные настройки 
        ```bash 
        awg setconf awg0 /etc/amnezia/awg0.conf
        ```
        

=== "Приложение на стороне клиента"

    ```ini
        [Interface]
        PrivateKey = 4ADcTmeIglwS6O6VmMn8VZmyn5nA5aXN/4g53jAQH2Y=
        Address = 10.47.0.2/24

        [Peer]
        PublicKey = PX6J6bFkdMEsKmqVl5M7sbgiXFHKWyl8rlGLaeUc0Rg=
        PresharedKey = yIw2PtOGI35SczEyb4d4mcViTqIjk0wa4sScaE3ciHE=
        AllowedIPs = 10.47.0.0/24
        Endpoint = domain.com:12413
        PersistentKeepalive = 25
    ```





## запускаем скрипт 

Создаем файл для запуска скрипта 
```bash
nano gen-awg.sh
```


```bash
#!/usr/bin/env bash
set -euo pipefail

rand() {
    local min="$1"
    local max="$2"
    shuf -i "${min}-${max}" -n 1
}

S1=$(rand 0 64)
S2=$(rand 0 64)
S3=$(rand 0 64)
S4=$(rand 0 32)

Jc=$(rand 2 5)
Jmin=$(rand 64 256)
Jmax=$(rand "$Jmin" 1024)

RANGE_SIZE=$(rand 1000000 5000000)

H1_START=$(rand 1000000000 1500000000)
H1_END=$((H1_START + RANGE_SIZE))

H2_START=$((H1_END + 1 + $(rand 1000000 10000000)))
H2_END=$((H2_START + RANGE_SIZE))

H3_START=$((H2_END + 1 + $(rand 1000000 10000000)))
H3_END=$((H3_START + RANGE_SIZE))

H4_START=$((H3_END + 1 + $(rand 1000000 10000000)))
H4_END=$((H4_START + RANGE_SIZE))

cat <<EOF
Jc = $Jc
Jmin = $Jmin
Jmax = $Jmax
S1 = $S1
S2 = $S2
S3 = $S3
S4 = $S4
H1 = $H1_START-$H1_END
H2 = $H2_START-$H2_END
H3 = $H3_START-$H3_END
H4 = $H4_START-$H4_END
EOF

```
Данный файл делаем исполняемым
```bash
chmod u+x gen-awg.sh
```

Запускаем данный файл 

```bash
./gen-awg.sh
```

```ini
Jc = 3
Jmin = 190
Jmax = 662
S1 = 48
S2 = 36
S3 = 3
S4 = 17
H1 = 1185686436-1188048232
H2 = 1193064451-1195426247
H3 = 1202295992-1204657788
H4 = 1210265272-1212627068
```







## Маскируем трафик
```ini
I1 = <b 0x9e5e42a4ba0ee46017c494560800450004fed7d8400080115e7aac1fed098efb9777e90601bb04ea9009cd0000000108c3f815569eb7ce1100404600206ab99afa27c11acded55b3ad92d5ec99497f42c17899cbaaa8e5478ecd41bfc2650ae5ce6aa7294013f8edc1e8c1aa8f1b199d3e0b89783e3ca91d70077bcb3c5d9f2c8d44893bc2cb3cbeca2f8861d0f83a77a8798ff92cd22c366a0fdcac8490abbf1f8a892d7ce4e2514947a23441a82528f689c62e068d9d3b34cd367f221a7d240cf5d6153972147b51f71454ae488929f39467bdc69d00487fda37ad775d388f947d050a444e6667a1ebf1d329cf2f24993f32464a266caf2256b584c2bfa0bab5cc00978ee23bac76b841f4a6b3aec97c49e07892abfb0c4ab7bc5791438f6d0c2411998d3bb85d35d4c807066af9188a372109e480b3f754d348a01a94593a7b3fe84a8674e97f22647f027590fc7279a74a7b00f0ea185bc4350918c1b12ed507c9770a2196c47665cdd27cbaf302e85cecf1967a25cf2340ffc28346442171fd856f0f9a1a4aa7f20cc0c8766e9f45ad4af78189bc92f4da69ba85131fef61ecd5dce938ad02532e658569b462de61e7e0f7a31add60606d2eeec09b396b9622c8d30f7763ac64045eeecf32dbfa725d39cc986b36ed48c4fbc8253f51414a080591829f52c7b28fbafc0d2abc179395093696e658fd6b7f2176b5e8c62c96e327e4bfee90b862560c7d2f293240a3feabdc498463dc092746ad6e97efbeb6191b2fb5f652579e99e59733dcdfe10c22d24a662df9838a38f92b04215a63183967e9bdffb4da898a40c9c60171af2a86d1a22ade12e8ff3a2a8fc82571f6931ad603852d1911ef2c40b7150f24dc9e2e45c5ecedfb7ce42ad917a2f85e17153a6be03a62f640e94555aed16977a87b097627847cef199ff648a3a7b8d6ce4ff0ef62b33959a98fef461b33d66a60fa4dc875bce71c774fb28908acc2846e09f8f948eb496a58b4c6d2497faeb63f0a7d2612c47029e7db2da13d98fe64509a057204e2fcc8c75fc498b03a6d6d929c55fdad0fa374692337ac7c4cd5490200f7027790710e928b2e8137238b5affa4ab4e2f0695522f92185ba73fa0e372308540685addc857d9e50389254b07b0265842bb9926132b8b9c4a2bd9ab0f36294beb4c65332c4f83be46050db3f3bef323033d3316f93e15859a212f19a6c71b1c1d811ca0afe562b1d36bcde0292d96e60777dd0e28b2eefad8d28491a751ab7729471c2190aec26828ea8e745c23c4bd58c01ffbb34378c5fac2d70c7441bea28f35d0fa2c5df116247a6db8192c888628e48a00f9c76ee7c3af39fd6d2c4955dce015dda2e75d4caa8f68086fba732e515a0aab77b7c4fdbe5d951ae5eda408a23be1e1773171ceb5a3e76a35dec3988f7c28807ce4bb24438c50b3ac697372b30fc78d1696c56382d06f738859f917b27d2c103e0e5f2bfc9b6ede9888b9fa76a6225d1babb5f5ac62137761984dd7fae78ccce312f5c6d0c8fbfee74fc30f523b3103a19f9db3d3f571e5143c3a96c6985444e7722cedeeb2a36ce70c3ccd850a05e770fc6c6bc17d7bfe126e3db8a087d3130dac6a923a1e87dfe49e5a4ae6be69fb94f700fb61d023f7f1ec2f343f81c0237ed18224447d7446d3a658d4c4bebac630cbb7fbf328a4ac0cfaba7cf672fc7ea71becf3a4dadd054931e95a02f59673e44a4fe5fc93f4c050ee52efb6837645c99f4014b48a0f0cfb0ddfc6351a791486eef21bd60401b778972f2d8c2b0d5b0ce994a6fa20ce0f72cce6ea435714db3e09524fcb93><rc 8><t><r 50>
```