## Настраиваем сервер 
apt-get update - обновление пакетов
## Устанавливаем необходимые компоненты для кеомпиляции

 ```bash linenums="1" 
apt-get install -y git libelf-dev linux-headers-$(uname -r) build-essential pkg-config
 ```
## слонируем репозиторий ядра 


##clear



git clone --single-branch --depth=1 https://github.com/amnezia-vpn/amneziawg-linux-kernel-module.git
git clone --single-branch --depth=1 https://github.com/amnezia-vpn/amneziawg-tools.git

make -C amneziawg-linux-kernel-module/src -j$(nproc)
make -C amneziawg-linux-kernel-module/src install

make -C amneziawg-tools/src -j$(nproc)
make -C amneziawg-tools/src install