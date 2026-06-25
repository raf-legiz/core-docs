Устанавливаем обычный домен 

```bash
apt-get install dnsmasq
```
Останавливаем службу dnsmasq

```bash
service dnsmasq stop
```
Чистим дефолтный конфиг.

```bash
> /etc/dnsmasq.conf
```

ОТкрываем данный дефолтный файл
```bash
nano /etc/dnsmasq.conf
```

В данный файл записываем следующию информацию 
```bash
interface=awg0
server=8.8.8.8
cache-size=5000



server=/ipinfo.io/127.0.0.1#5353
```

Запускаем данную службу 

```bash
service dnsmasq start
```




## УСтановим x ray  

Создадим папку 
```bash
mkdir 111
```

перейдем в данную папку

```bash
cd ~/111
```
