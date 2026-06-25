## Настраиваем сервер 
apt-get update - обновление пакетов
## Устанавливаем необходимые компоненты для кеомпиляции

 ```bash
    apt-get install -y git libelf-dev linux-headers-$ (uname -r) build-essential pkg-config
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
