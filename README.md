# gametron303.github.io
Отлично! Подготовим подробную пошаговую инструкцию по выполнению всех трёх модулей демонстрационного экзамена по специальности "Сетевое и системное администрирование" (КОД 09.02.06-1-2025) на базе ОС Debian.

Мы будем использовать nmcli для управления сетевыми интерфейсами, где это применимо, и современные инструменты, такие как frr для маршрутизации и nftables для межсетевого экрана.

Важное замечание: Согласно вашему требованию, мы полностью игнорируем технологию VLAN. В задании упоминаются VLAN 100, 200 и 999, но мы будем настраивать сети так, как будто их не существует, размещая все хосты офиса HQ в одной общей подсети. Это является адаптацией задания под указанное вами ограничение.

Подготовительный этап: Виртуальные машины и IP-адресация

Прежде чем начать, убедитесь, что у вас есть все необходимые виртуальные машины с установленной Debian (например, Debian 12 "Bookworm") и правильным количеством сетевых адаптеров.

Топология виртуальных машин:

ISP: 3 сетевых адаптера (к интернету, к HQ-RTR, к BR-RTR)

HQ-RTR: 2 сетевых адаптера (к ISP, к локальной сети HQ)

BR-RTR: 2 сетевых адаптера (к ISP, к локальной сети BR)

HQ-SRV: 1 сетевой адаптер (+3 доп. диска по 1 ГБ для RAID в Модуле 2)

BR-SRV: 1 сетевой адаптер

HQ-CLI: 1 сетевой адаптер

План IP-адресации (наше решение для этого туториала):

Устройство	Интерфейс	IP-адрес / Маска	Шлюз	Описание
ISP	ens18 (к HQ)	172.16.4.1/28	-	Сеть для связи с HQ-RTR
	ens19 (к BR)	172.16.5.1/28	-	Сеть для связи с BR-RTR
	ens20 (WAN)	DHCP	-	"Внешний" интернет
HQ-RTR	ens18 (WAN)	172.16.4.2/28	172.16.4.1	Подключение к ISP
	ens19 (LAN)	192.168.100.1/24	-	Локальная сеть офиса HQ
BR-RTR	ens18 (WAN)	172.16.5.2/28	172.16.5.1	Подключение к ISP
	ens19 (LAN)	192.168.200.1/24	-	Локальная сеть офиса BR
HQ-SRV	ens18	192.168.100.10/24	192.168.100.1	Сервер в HQ
BR-SRV	ens18	192.168.200.10/24	192.168.200.1	Сервер в BR
HQ-CLI	ens18	DHCP	192.168.100.1	Клиент в HQ
Туннель	tun0 (HQ-RTR)	10.10.10.1/30	-	GRE-туннель
	tun0 (BR-RTR)	10.10.10.2/30	-	GRE-туннель

Домен для всех машин: au-team.irpo.

Модуль 1: Настройка сетевой инфраструктуры
1. Базовая настройка: имена хостов и статические IP

На каждой машине установите соответствующее имя хоста.

# Пример для HQ-RTR. Выполнить на всех машинах с их именами.
hostnamectl set-hostname hq-rtr.au-team.irpo


Теперь настроим сетевые интерфейсы с помощью nmcli.

[На HQ-RTR]

# Настраиваем WAN интерфейс (ens18)
nmcli con add type ethernet con-name "WAN" ifname ens18 ip4 172.16.4.2/28 gw4 172.16.4.1
nmcli con mod "WAN" ipv4.dns "8.8.8.8"
nmcli con up "WAN"

# Настраиваем LAN интерфейс (ens19)
nmcli con add type ethernet con-name "LAN" ifname ens19 ip4 192.168.100.1/24
nmcli con up "LAN"
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На BR-RTR]

# Настраиваем WAN интерфейс (ens18)
nmcli con add type ethernet con-name "WAN" ifname ens18 ip4 172.16.5.2/28 gw4 172.16.5.1
nmcli con mod "WAN" ipv4.dns "8.8.8.8"
nmcli con up "WAN"

# Настраиваем LAN интерфейс (ens19)
nmcli con add type ethernet con-name "LAN" ifname ens19 ip4 192.168.200.1/24
nmcli con up "LAN"
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На HQ-SRV]

nmcli con add type ethernet con-name "LAN" ifname ens18 ip4 192.168.100.10/24 gw4 192.168.100.1
nmcli con mod "LAN" ipv4.dns "192.168.100.10" # Сам себе DNS
nmcli con up "LAN"
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На BR-SRV]

nmcli con add type ethernet con-name "LAN" ifname ens18 ip4 192.168.200.10/24 gw4 192.168.200.1
nmcli con mod "LAN" ipv4.dns "192.168.100.10" # DNS-сервер в HQ
nmcli con up "LAN"
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На HQ-CLI]
Эта машина будет получать адрес по DHCP, пока ничего не делаем.

2. Настройка ISP

[На ISP]

# Настраиваем интерфейсы к филиалам
nmcli con add type ethernet con-name "TO-HQ" ifname ens18 ip4 172.16.4.1/28
nmcli con up "TO-HQ"

nmcli con add type ethernet con-name "TO-BR" ifname ens19 ip4 172.16.5.1/28
nmcli con up "TO-BR"

# Настраиваем "внешний" интерфейс через DHCP
nmcli con add type ethernet con-name "WAN-DHCP" ifname ens20
nmcli con up "WAN-DHCP"

# Включаем форвардинг и NAT
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf

# Устанавливаем и настраиваем файрвол для NAT
apt update && apt install -y nftables
# В файле /etc/nftables.conf добавляем в конец таблицы 'inet filter' перед '}'
# chain postrouting {
#    type nat hook postrouting priority 100; policy accept;
#    oifname "ens20" masquerade
# }
# После редактирования файла:
systemctl enable --now nftables
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
3. Создание пользователей

[На HQ-SRV и BR-SRV]

# Создаем пользователя sshuser
useradd -m -s /bin/bash sshuser
# Устанавливаем пароль
passwd sshuser # Вводим P@ssw0rd
# Добавляем в группу sudo, чтобы он мог выполнять команды с правами root
usermod -aG sudo sshuser
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На HQ-RTR и BR-RTR]
Здесь мы будем использовать frr для маршрутизации, у которого своя система пользователей. Настроим системного пользователя net_admin для доступа к ОС.

useradd -m -s /bin/bash net_admin
passwd net_admin # Вводим P@$$word
usermod -aG sudo,frr,frrvty net_admin # Добавляем в группы для управления FRR
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
4. Настройка SSH

[На HQ-SRV и BR-SRV]
Редактируем /etc/ssh/sshd_config:

Port 2024
PermitRootLogin no
PasswordAuthentication yes # Оставляем для простоты, в реальной жизни - ключи
AllowUsers sshuser
Banner /etc/issue.net
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
IGNORE_WHEN_COPYING_END

Создаем файл баннера:
echo "Authorized access only" > /etc/issue.net

Перезапускаем SSH:
systemctl restart sshd

5. Настройка GRE-туннеля

[На HQ-RTR]

# Создаем туннель
ip tunnel add tun0 mode gre remote 172.16.5.2 local 172.16.4.2 ttl 255
# Назначаем IP-адрес
ip addr add 10.10.10.1/30 dev tun0
# Поднимаем интерфейс
ip link set tun0 up
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На BR-RTR]

# Создаем туннель
ip tunnel add tun0 mode gre remote 172.16.4.2 local 172.16.5.2 ttl 255
# Назначаем IP-адрес
ip addr add 10.10.10.2/30 dev tun0
# Поднимаем интерфейс
ip link set tun0 up
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Проверяем пингом: ping 10.10.10.2 с HQ-RTR.

6. Динамическая маршрутизация (OSPF с FRR)

[На HQ-RTR и BR-RTR]

# Устанавливаем FRR
apt update
apt install -y frr frr-pythontools

# Включаем демоны OSPF и Zebra в /etc/frr/daemons
# ospfd=yes
# zebra=yes

# Запускаем сервис
systemctl enable --now frr

# Входим в конфигурацию
vtysh

# --- Конфигурация для HQ-RTR в vtysh ---
conf t
router ospf
 ospf router-id 1.1.1.1
 network 10.10.10.0/30 area 0
 network 192.168.100.0/24 area 0
 passive-interface ens19
 area 0 authentication message-digest
 interface tun0
  ip ospf message-digest-key 1 md5 MySecretPass
exit
exit
write
# --- Конец конфигурации HQ-RTR ---


# --- Конфигурация для BR-RTR в vtysh ---
conf t
router ospf
 ospf router-id 2.2.2.2
 network 10.10.10.0/30 area 0
 network 192.168.200.0/24 area 0
 passive-interface ens19
 area 0 authentication message-digest
 interface tun0
  ip ospf message-digest-key 1 md5 MySecretPass
exit
exit
write
# --- Конец конфигурации BR-RTR ---
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Проверяем: на HQ-RTR show ip route должен показать маршрут до 192.168.200.0/24.

7. Настройка NAT

[На HQ-RTR и BR-RTR]

# Включаем форвардинг
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf

# Устанавливаем и настраиваем nftables
apt install -y nftables
# В файле /etc/nftables.conf для HQ-RTR добавляем правило NAT:
# chain postrouting {
#    type nat hook postrouting priority 100; policy accept;
#    oifname "ens18" masquerade
# }
# Для BR-RTR то же самое, oifname тоже будет "ens18".
systemctl enable --now nftables
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
8. DHCP-сервер

[На HQ-RTR]

apt install -y isc-dhcp-server

# Редактируем /etc/dhcp/dhcpd.conf
# Указываем наш домен и DNS
option domain-name "au-team.irpo";
option domain-name-servers 192.168.100.10;

# Базовая конфигурация
default-lease-time 600;
max-lease-time 7200;
ddns-update-style none;
authoritative;

# Описание нашей подсети
subnet 192.168.100.0 netmask 255.255.255.0 {
  range 192.168.100.100 192.168.100.200;
  option routers 192.168.100.1;
  option broadcast-address 192.168.100.255;
}

# Указываем интерфейс для работы в /etc/default/isc-dhcp-server
# INTERFACESv4="ens19"

# Запускаем
systemctl enable --now isc-dhcp-server
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Теперь на HQ-CLI можно запустить dhclient или перезагрузить сеть, и он получит IP.

9. DNS-сервер

[На HQ-SRV]

apt install -y bind9 bind9-utils

# Редактируем /etc/bind/named.conf.options, добавляем forwarders
# forwarders {
#   8.8.8.8;
#   8.8.4.4;
# };

# Редактируем /etc/bind/named.conf.local для добавления наших зон
zone "au-team.irpo" {
    type master;
    file "/etc/bind/db.au-team.irpo";
};

zone "100.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.100";
};

# Создаем файл прямой зоны /etc/bind/db.au-team.irpo
$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. admin.au-team.irpo. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      hq-srv.au-team.irpo.
hq-rtr  IN      A       172.16.4.2
br-rtr  IN      A       172.16.5.2
hq-srv  IN      A       192.168.100.10
br-srv  IN      A       192.168.200.10
hq-cli  IN      A       192.168.100.100 ; Пример IP от DHCP
moodle  IN      CNAME   hq-rtr
wiki    IN      CNAME   hq-rtr

# Создаем файл обратной зоны /etc/bind/db.192.168.100
$TTL    604800
@       IN      SOA     hq-srv.au-team.irpo. admin.au-team.irpo. (
                              1         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      hq-srv.au-team.irpo.
10      IN      PTR     hq-srv.au-team.irpo.
100     IN      PTR     hq-cli.au-team.irpo. ; Пример

# Проверяем и запускаем
named-checkconf
named-checkzone au-team.irpo /etc/bind/db.au-team.irpo
systemctl enable --now bind9
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

Проверяем с HQ-CLI: dig hq-srv.au-team.irpo @192.168.100.10.

Модуль 2: Организация сетевого администрирования
1. Доменный контроллер Samba

[На BR-SRV]

# Устанавливаем пакеты
apt install -y samba krb5-config winbind smbclient

# Останавливаем и отключаем системные службы, которые будут заменены Samba
systemctl stop smbd nmbd winbind
systemctl disable smbd nmbd winbind

# Запускаем подготовку домена
samba-tool domain provision --use-rfc2307 --interactive
# Realm: AU-TEAM.IRPO
# Domain: AU-TEAM
# Server Role: dc
# DNS backend: SAMBA_INTERNAL
# Administrator password: Вводим сложный пароль, например, P@ssw0rd123

# Копируем конфиг Kerberos
cp /var/lib/samba/private/krb5.conf /etc/

# Запускаем Samba как AD DC
systemctl unmask samba-ad-dc
systemctl enable --now samba-ad-dc

# Проверяем
samba-tool domain level show

# Создаем пользователей и группу
samba-tool group add hq
for i in {1..5}; do samba-tool user create user${i}.hq --given-name=User --surname=HQ${i} --mail-address=user${i}@au-team.irpo; samba-tool group addmembers hq user${i}.hq; done
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На HQ-CLI]
Чтобы ввести машину в домен, нужно установить пакеты и настроить realmd.

apt install -y realmd sssd sssd-tools adcli packagekit samba-common-bin
# Редактируем /etc/sssd/sssd.conf и /etc/realmd.conf при необходимости

# Вступаем в домен
realm join --user=administrator au-team.irpo
# Вводим пароль администратора домена
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
2. Файловое хранилище (RAID + NFS)

[На HQ-SRV]

apt install -y mdadm nfs-kernel-server

# Создаем RAID5 из трех дисков (/dev/sdb, /dev/sdc, /dev/sdd)
mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd

# Сохраняем конфигурацию
mdadm --detail --scan >> /etc/mdadm/mdadm.conf

# Форматируем
mkfs.ext4 /dev/md0

# Создаем точку монтирования и монтируем
mkdir /raid5
mount /dev/md0 /raid5

# Настраиваем автомонтирование в /etc/fstab
echo "/dev/md0 /raid5 ext4 defaults 0 0" >> /etc/fstab

# Настраиваем NFS
mkdir /raid5/nfs
chown nobody:nogroup /raid5/nfs
# Добавляем в /etc/exports
echo "/raid5/nfs 192.168.100.0/24(rw,sync,no_subtree_check)" >> /etc/exports

# Экспортируем и перезапускаем
exportfs -a
systemctl restart nfs-kernel-server
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На HQ-CLI]

apt install -y nfs-common
mkdir -p /mnt/nfs
# Добавляем в /etc/fstab для автомонтирования
echo "192.168.100.10:/raid5/nfs /mnt/nfs nfs auto,nofail,noatime,nolock,intr,tcp,actimeo=1800 0 0" >> /etc/fstab
mount -a
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
3. Служба времени (chrony)

[На HQ-RTR] (Сервер)

apt install -y chrony
# Редактируем /etc/chrony/chrony.conf
# Добавляем:
# local stratum 5
# allow 192.168.0.0/16
systemctl restart chrony
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На всех остальных машинах] (Клиенты)

apt install -y chrony
# Редактируем /etc/chrony/chrony.conf
# Комментируем пулы (pool ...) и добавляем:
# server hq-rtr.au-team.irpo iburst
systemctl restart chrony
chronyc sources # Проверка
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
4. Ansible

[На BR-SRV]

apt install -y ansible
mkdir /etc/ansible
# Создаем inventory файл /etc/ansible/hosts
cat <<EOF > /etc/ansible/hosts
[routers]
hq-rtr.au-team.irpo
br-rtr.au-team.irpo

[servers]
hq-srv.au-team.irpo
br-srv.au-team.irpo

[clients]
hq-cli.au-team.irpo

[all:vars]
ansible_user=net_admin
ansible_ssh_pass=P@$$word
ansible_become_pass=P@$$word
EOF
# Для серверов и клиентов ansible_user=sshuser и ansible_ssh_pass=P@ssw0rd

# Проверяем
ansible all -m ping
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
5. Docker и MediaWiki

[На BR-SRV]

apt install -y docker.io docker-compose
systemctl enable --now docker

# Создаем файл docker-compose.yml
cat <<EOF > ~/wiki.yml
version: '3'

services:
  mariadb:
    image: mariadb
    container_name: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: some_root_password
      MYSQL_DATABASE: mediawiki
      MYSQL_USER: wikiuser
      MYSQL_PASSWORD: WikiP@ssw0rd
    volumes:
      - ./mariadb_data:/var/lib/mysql

  wiki:
    image: mediawiki
    container_name: wiki
    restart: always
    ports:
      - "8080:80"
    links:
      - mariadb
    volumes:
      - ./mediawiki_data:/var/www/html/images
EOF

# Запускаем
docker-compose -f ~/wiki.yml up -d
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
6. Nginx как Reverse Proxy

[На HQ-RTR]

apt install -y nginx
# Создаем конфиг /etc/nginx/sites-available/reverse-proxy
cat <<EOF > /etc/nginx/sites-available/reverse-proxy
server {
    listen 80;
    server_name moodle.au-team.irpo;

    location / {
        proxy_pass http://192.168.100.10; # Адрес Moodle
        proxy_set_header Host \$host;
    }
}

server {
    listen 80;
    server_name wiki.au-team.irpo;

    location / {
        proxy_pass http://192.168.200.10:8080; # Адрес Wiki
        proxy_set_header Host \$host;
    }
}
EOF

# Активируем
ln -s /etc/nginx/sites-available/reverse-proxy /etc/nginx/sites-enabled/
systemctl restart nginx
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
Модуль 3: Эксплуатация объектов сетевой инфраструктуры
1. Центр сертификации и HTTPS

[На HQ-SRV]

apt install -y easy-rsa
make-cadir ~/my_ca
cd ~/my_ca

# Редактируем vars, устанавливаем нужные значения
# ./easyrsa init-pki
# ./easyrsa build-ca # Создаем CA
# ./easyrsa gen-req moodle.au-team.irpo nopass # Запрос для Moodle
# ./easyrsa gen-req wiki.au-team.irpo nopass # Запрос для Wiki
# ./easyrsa sign-req server moodle.au-team.irpo # Подписываем
# ./easyrsa sign-req server wiki.au-team.irpo # Подписываем

# Копируем сертификаты и ключи на HQ-RTR
# scp pki/ca.crt net_admin@hq-rtr:/etc/nginx/ssl/
# scp pki/issued/moodle.au-team.irpo.crt net_admin@hq-rtr:/etc/nginx/ssl/
# scp pki/private/moodle.au-team.irpo.key net_admin@hq-rtr:/etc/nginx/ssl/
# и т.д.
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На HQ-RTR]
Обновляем конфиг Nginx /etc/nginx/sites-available/reverse-proxy для работы с HTTPS.

server {
    listen 443 ssl;
    server_name moodle.au-team.irpo;

    ssl_certificate /etc/nginx/ssl/moodle.au-team.irpo.crt;
    ssl_certificate_key /etc/nginx/ssl/moodle.au-team.irpo.key;

    location / {
        proxy_pass http://192.168.100.10;
        proxy_set_header Host $host;
    }
}
# Аналогично для wiki
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Nginx
IGNORE_WHEN_COPYING_END

systemctl restart nginx

2. Защищенный туннель (IPsec)

[На HQ-RTR и BR-RTR]
Мы будем шифровать наш существующий GRE-туннель.

apt install -y strongswan

# Конфиг /etc/ipsec.conf на HQ-RTR
conn gre-tunnel
    type    = transport
    authby  = secret
    left    = 172.16.4.2
    right   = 172.16.5.2
    auto    = start
    protocol= gre

# Конфиг /etc/ipsec.conf на BR-RTR (left и right меняются местами)
conn gre-tunnel
    type    = transport
    authby  = secret
    left    = 172.16.5.2
    right   = 172.16.4.2
    auto    = start
    protocol= gre

# На обеих машинах /etc/ipsec.secrets
172.16.4.2 172.16.5.2 : PSK "MySuperSecretKey"

# Запускаем
systemctl restart strongswan
ipsec status # Проверка
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
3. Межсетевой экран (Firewall)

[На HQ-RTR и BR-RTR]
Используем nftables. Редактируем /etc/nftables.conf.

#!/usr/sbin/nft -f

flush ruleset

table inet filter {
    chain input {
        type filter hook input priority 0;
        
        # Разрешаем established/related
        ct state {established, related} accept

        # Разрешаем loopback
        iif lo accept
        
        # Разрешаем ICMP
        ip protocol icmp accept
        
        # Разрешаем SSH, DNS, HTTP, HTTPS, OSPF, IPsec
        tcp dport {22, 80, 443} accept
        udp dport 53 accept
        ip protocol ospf accept
        udp dport {500, 4500} accept

        # Разрешаем GRE
        ip protocol gre accept

        # Все остальное с WAN-интерфейса дропаем
        iifname "ens18" drop
    }
    
    chain forward {
        type filter hook forward priority 0;
        ct state {established, related} accept

        # Разрешаем трафик из LAN в WAN
        iifname "ens19" oifname "ens18" accept
        # Разрешаем трафик из LAN в туннель
        iifname "ens19" oifname "tun0" accept
        # Разрешаем трафик из туннеля в LAN
        iifname "tun0" oifname "ens19" accept
        
        # Дропаем все остальное
        drop
    }

    chain postrouting {
        type nat hook postrouting priority 100; policy accept;
        oifname "ens18" masquerade
    }
}
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Nft
IGNORE_WHEN_COPYING_END

systemctl restart nftables

4. Сервер логирования и мониторинга

[На HQ-SRV] (Сервер)

# Syslog
apt install -y rsyslog
# В /etc/rsyslog.conf раскомментировать строки для приема логов по UDP/TCP
# module(load="imudp")
# input(type="imudp" port="514")
# module(load="imtcp")
# input(type="imtcp" port="514")
systemctl restart rsyslog

# Мониторинг (Zabbix) - очень кратко
apt install -y zabbix-server-pgsql zabbix-frontend-php zabbix-agent
# Настроить базу данных postgresql, импортировать схему
# Настроить /etc/zabbix/zabbix_server.conf
# Настроить /etc/zabbix/apache.conf
# Запустить и добавить в автозагрузку
systemctl enable --now zabbix-server zabbix-agent apache2
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END

[На всех остальных машинах] (Клиенты логов и агенты мониторинга)

# Отправка логов
echo "*.* @192.168.100.10:514" > /etc/rsyslog.d/50-remote.conf
systemctl restart rsyslog

# Агент Zabbix
apt install -y zabbix-agent
# В /etc/zabbix/zabbix_agentd.conf прописать:
# Server=192.168.100.10
# ServerActive=192.168.100.10
# Hostname=<имя машины>
systemctl enable --now zabbix-agent
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Bash
IGNORE_WHEN_COPYING_END
5. Ansible: инвентаризация и бэкап

[На BR-SRV]
Создаем плейбук для сбора информации ~/inventory.yml:

- name: Gather facts about hosts
  hosts: all
  tasks:
    - name: Create directory for facts
      file:
        path: "/root/PC_INFO"
        state: directory
      delegate_to: localhost
    - name: Save host facts to a file
      copy:
        content: |
          Hostname: {{ ansible_hostname }}
          IP Address: {{ ansible_default_ipv4.address }}
        dest: "/root/PC_INFO/{{ inventory_hostname }}.yml"
      delegate_to: localhost
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Yaml
IGNORE_WHEN_COPYING_END

Запускаем: ansible-playbook ~/inventory.yml

Создаем плейбук для бэкапа конфигурации роутеров ~/backup.yml:

- name: Backup router configurations
  hosts: routers
  tasks:
    - name: Create backup directory
      file:
        path: "/root/NETWORK_INFO/{{ inventory_hostname }}"
        state: directory
      delegate_to: localhost
    - name: Fetch frr.conf
      fetch:
        src: /etc/frr/frr.conf
        dest: /root/NETWORK_INFO/{{ inventory_hostname }}/frr.conf
        flat: yes
    - name: Fetch nftables.conf
      fetch:
        src: /etc/nftables.conf
        dest: /root/NETWORK_INFO/{{ inventory_hostname }}/nftables.conf
        flat: yes
IGNORE_WHEN_COPYING_START
content_copy
download
Use code with caution.
Yaml
IGNORE_WHEN_COPYING_END

Запускаем: ansible-playbook ~/backup.yml
