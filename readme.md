# VPN   
   
## Цели домашнего задания   
   
1. Между двумя виртуалками поднять vpn в режимах:   
   
- tun   
   
- tap   
   
Описать в чём разница, замерить скорость между виртуальными машинами в туннелях, сделать вывод об отличающихся показателях скорости.   
   
2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку.   
   
## 1. VPN   
   
● устанавливаем epel репозиторий и пакет openvpn и iperf3   
```
yum install -y epel-release && yum install -y openvpn iperf3
```
   
● Отключаем SELinux   
```
setenforce 0
```
   
   
### Настройка openvpn сервера:   
   
● создаём файл-ключ   
```
openvpn --genkey --secret /etc/openvpn/static.key
```
   
● создаём конфигурационный файл vpn-сервера   
```
/etc/openvpn/server.conf
```
   
   
```
dev tap   
ifconfig 10.10.10.1 255.255.255.0   
topology subnet   
secret /etc/openvpn/static.key   
comp-lzo   
status /var/log/openvpn-status.log   
log /var/log/openvpn.log   
verb 3
```
   
   
● Создадим service unit для запуска openvpn   
```
vi /etc/systemd/system/openvpn@.service
```
   
● Файл server.conf должен содержать следующий конфиг   
```
[Unit]   
Description=OpenVPN Tunneling Application On %I   
After=network.target   
[Service]   
Type=notify   
PrivateTmp=true   
ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf   
[Install]   
WantedBy=multi-user.target
```
   
   
● Запускаем openvpn сервер и добавляем в автозагрузку.   
```
systemctl start openvpn@server   
systemctl enable openvpn@server
```
   
   
### Настройка openvpn клиента.   
   
● создаём конфигурационный файл клиента   
```
vi /etc/openvpn/server.conf
```
   
● Файл должен содержать следующий конфиг   
```
dev tap   
remote 192.168.56.10   
ifconfig 10.10.10.2 255.255.255.0   
topology subnet   
route 192.168.56.0 255.255.255.0   
secret /etc/openvpn/static.key   
comp-lzo   
status /var/log/openvpn-status.log   
log /var/log/openvpn.log   
verb 3
```
   
   
● На сервер клиента в директорию /etc/openvpn необходимо скопировать   
файл-ключ static.key, который был создан на сервере.   
● Запускаем openvpn клиент и добавляем в автозагрузку   
```
systemctl start openvpn@server   
systemctl enable openvpn@server
```
   
   
### Далее необходимо замерить скорость в туннеле.   
● на openvpn сервере запускаем iperf3 в режиме сервера   
```
iperf3 -s &
```
   
● на openvpn клиенте запускаем iperf3 в режиме клиента и замеряем   
скорость в туннеле   
```
iperf3 -c 10.10.10.1 -t 40 -i 5
```
   
   
Получаем вывод в режиме TAP:   
```
Connecting to host 10.10.10.1, port 5201   
[ 4] local 10.10.10.2 port 45618 connected to 10.10.10.1 port 5201   
[ ID] Interval Transfer Bandwidth Retr Cwnd   
[ 4] 0.00-5.00 sec 33.9 MBytes 56.8 Mbits/sec 29 1002 KBytes   
[ 4] 5.00-10.01 sec 34.6 MBytes 57.9 Mbits/sec 9 1.16 MBytes   
[ 4] 10.01-15.01 sec 38.6 MBytes 64.7 Mbits/sec 2 1.06 MBytes   
[ 4] 15.01-20.00 sec 42.1 MBytes 70.8 Mbits/sec 0 1.07 MBytes   
[ 4] 20.00-25.00 sec 35.9 MBytes 60.3 Mbits/sec 0 1.26 MBytes   
[ 4] 25.00-30.00 sec 32.3 MBytes 54.2 Mbits/sec 14 1.21 MBytes   
[ 4] 30.00-35.00 sec 37.2 MBytes 62.4 Mbits/sec 1 1.03 MBytes   
[ 4] 35.00-40.00 sec 40.8 MBytes 68.4 Mbits/sec 0 1.06 MBytes   
- - - - - - - - - - - - - - - - - - - - - - - - -   
[ ID] Interval Transfer Bandwidth Retr   
[ 4] 0.00-40.00 sec 295 MBytes 61.9 Mbits/sec 55 sender   
[ 4] 0.00-40.00 sec 293 MBytes 61.5 Mbits/sec receiver   
   
iperf Done.
```
   
   
### Повторяем пункты 1-5 для режима работы tun. Конфигарационные файлы сервера и клиента изменятся только в директиве dev. Делаем выводы о режимах, их достоинствах и недостатках.   
   
● Вывод в режиме TUN:   
```
[ 4] local 10.10.10.2 port 45622 connected to 10.10.10.1 port 5201   
[ ID] Interval Transfer Bandwidth Retr Cwnd   
[ 4] 0.00-5.00 sec 21.2 MBytes 35.6 Mbits/sec 22 535 KBytes   
[ 4] 5.00-10.01 sec 23.6 MBytes 39.6 Mbits/sec 4 523 KBytes   
[ 4] 10.01-15.00 sec 35.1 MBytes 58.9 Mbits/sec 190 503 KBytes   
[ 4] 15.00-20.00 sec 37.2 MBytes 62.3 Mbits/sec 65 445 KBytes   
[ 4] 20.00-25.00 sec 36.6 MBytes 61.4 Mbits/sec 0 551 KBytes   
[ 4] 25.00-30.00 sec 36.5 MBytes 61.2 Mbits/sec 22 456 KBytes   
[ 4] 30.00-35.00 sec 37.3 MBytes 62.7 Mbits/sec 0 566 KBytes   
[ 4] 35.00-40.00 sec 36.4 MBytes 61.0 Mbits/sec 0 601 KBytes   
- - - - - - - - - - - - - - - - - - - - - - - - -   
[ ID] Interval Transfer Bandwidth Retr   
[ 4] 0.00-40.00 sec 264 MBytes 55.3 Mbits/sec 303 sender   
[ 4] 0.00-40.00 sec 262 MBytes 55.0 Mbits/sec receiver   
   
iperf Done.
```
   
   
### Вывод:   
Скорость работы тоннеля в режиме TAP немного выше, это связано с уровнями работы: TAP работает на уровне L2, тогда как TUN - на уровне L3.   
   
   
## 2. RAS на базе OpenVPN   
   
Для выполнения данного задания можно воспользоваться Vagrantfile из 1 задания, только убрать 1 ВМ. После запуска отключаем SELinux (setenforce 0) или создаём правило для него   
● Устанавливаем репозиторий EPEL.   
```
yum install -y epel-release
```
   
● Устанавливаем необходимые пакеты.   
```
yum install -y openvpn easy-rsa
```
   
● Переходим в директорию /etc/openvpn/ и инициализируем pki   
```
cd /etc/openvpn/   
/usr/share/easy-rsa/3.0.8/easyrsa init-pki
```
   
● Сгенерируем необходимые ключи и сертификаты для сервера   
```
echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa build-ca nopass   
echo 'rasvpn' | /usr/share/easy-rsa/3.0.8/easyrsa gen-req server nopass   
echo 'yes' | /usr/share/easy-rsa/3.0.8/easyrsa sign-req server server   
/usr/share/easy-rsa/3.0.8/easyrsa gen-dh   
openvpn --genkey --secret ca.key
```
   
   
Версия easy-rsa может отличаться, посмотреть актуальную версию можно так:   
```
rpm -qa | grep easy-rsa   
easy-rsa-3.0.8-1.el8.noarch
```
   
● Сгенерируем сертификаты для клиента.   
```
echo 'client' | /usr/share/easy-rsa/3/easyrsa gen-req client nopass   
echo 'yes' | /usr/share/easy-rsa/3/easyrsa sign-req client client
```
   
● Создадим конфигурационный файл /etc/openvpn/server.conf   
(файл конфигурации показан на слайде #12)   
● Зададим параметр iroute для клиента   
```
echo 'iroute 10.10.10.0 255.255.255.0' > /etc/openvpn/client/client
```
   
● Запускаем openvpn сервер и добавляем его в автозагрузку   
```
systemctl start openvpn@server   
systemctl enable openvpn@server
```
   
● Скопируем следующие файлы сертификатов и ключ для клиента на хостмашину.   
```
/etc/openvpn/pki/ca.crt   
/etc/openvpn/pki/issued/client.crt   
/etc/openvpn/pki/private/client.key
```
   
(файлы рекомендуется расположить в той же директории, что и client.conf)   
● Создадим конфигурационны файл клиента client.conf на хост-машине (файл конфигурации показан на слайде #13).   
   
(Создание юнита рассмотренно на слайде #7)   
Файл конфигурации server.conf >   
```
port 1207   
proto udp   
dev tun   
ca /etc/openvpn/pki/ca.crt   
cert /etc/openvpn/pki/issued/server.crt   
key /etc/openvpn/pki/private/server.key   
dh /etc/openvpn/pki/dh.pem   
server 10.10.10.0 255.255.255.0   
ifconfig-pool-persist ipp.txt   
client-to-client   
client-config-dir /etc/openvpn/client   
keepalive 10 120   
comp-lzo   
persist-key   
persist-tun   
status /var/log/openvpn-status.log   
log /var/log/openvpn.log   
verb 3
```
   
   
Файл конфигурации client.conf >   
```
dev tun   
proto udp   
remote 192.168.56.10 1207   
client   
resolv-retry infinite   
remote-cert-tls server   
ca ./ca.crt   
cert ./client.crt   
key ./client.key   
route 192.168.56.0 255.255.255.0   
persist-key   
persist-tun   
comp-lzo   
verb 3
```
   
   
В этом конфигурационном файле указано, что файлы сертификатов располагаются в директории, где располагается client.conf. При желании можно разместить сертификаты в других директориях и в конфиге скорректировать пути.   
   
● После того, как все готово, подключаемся к openvpn сервер с хост-машины.   
```
sudo openvpn --config client.conf
```
   
● При успешном подключении проверяем пинг по внутреннему IP адресу сервера в туннеле.   
```
ping -c 4 10.10.10.1
PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.051 ms
64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.033 ms
64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.035 ms
64 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=0.046 ms

--- 10.10.10.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 2998ms
rtt min/avg/max/mdev = 0.033/0.041/0.051/0.008 ms

```
   
● Также проверяем командой ip r (netstat -rn) на хостовой машине что сеть туннеля импортирована в таблицу маршрутизации.