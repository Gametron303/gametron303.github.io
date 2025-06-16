**Demo Exam 2025: Пошаговая инструкция**

_Основано на комплексе оценочных материалов Демонстрационного экзамена (КОД 09.02.06-1-2025)

## Содержание

1. [Подготовительный этап](#подготовительный-этап)
2. [Модуль 1: Настройка сетевой инфраструктуры](#модуль-1-настройка-сетевой-инфраструктуры)
3. [Модуль 2: Организация сетевого администрирования](#модуль-2-организация-сетевого-администрирования)
4. [Модуль 3: Эксплуатация объектов сетевой инфраструктуры](#модуль-3-эксплуатация-объектов-сетевой-инфраструктуры)

---

## Подготовительный этап

1. **Платформа и машины**
   - Debian 12 "Bookworm" или аналог на каждой VM.
   - Убедитесь в наличии требуемых сетевых адаптеров:

     | Машина    | Адаптеры                                   |
     |-----------|--------------------------------------------|
     | ISP       | ens18 (k HQ), ens19 (k BR), ens20 (WAN)    |
     | HQ-RTR    | ens18 (WAN), ens19 (LAN)                   |
     | BR-RTR    | ens18 (WAN), ens19 (LAN)                   |
     | HQ-SRV    | ens18 (+ 3 диска 1 ГБ для RAID)            |
     | BR-SRV    | ens18                                     |
     | HQ-CLI    | ens18                                     |

2. **IP-адресация** (игнорируем VLAN 100/200/999 по требованию)

   | Устр-во   | Интерфейс     | IP/Mask          | Шлюз          | Комментарий               |
   |-----------|---------------|------------------|---------------|--------------------------|
   | ISP ens18 | ens18         | 172.16.4.1/28    | -             | связь с HQ               |
   | ISP ens19 | ens19         | 172.16.5.1/28    | -             | связь с BR               |
   | ISP ens20 | ens20         | DHCP             | -             | выход в интернет         |
   | HQ-RTR    | ens18         | 172.16.4.2/28    | 172.16.4.1    |                          |
   | HQ-RTR    | ens19         | 192.168.100.1/24 | -             | LAN HQ                   |
   | BR-RTR    | ens18         | 172.16.5.2/28    | 172.16.5.1    |                          |
   | BR-RTR    | ens19         | 192.168.200.1/24 | -             | LAN BR                   |
   | HQ-SRV    | ens18         | 192.168.100.10/24| 192.168.100.1 |                          |
   | BR-SRV    | ens18         | 192.168.200.10/24| 192.168.200.1 |                          |
   | HQ-CLI    | ens18         | DHCP via HQ-RTR  | 192.168.100.1 |                          |
   | Tunnel    | tun0 (HQ)     | 10.10.10.1/30    | -             |                          |
   | Tunnel    | tun0 (BR)     | 10.10.10.2/30    | -             |                          |

3. **Домен для всех машин**: `au-team.irpo`

---

# 📦 Требуемые пакеты для Демоэкзамена 2025

Перед началом выполнения заданий убедитесь, что все виртуальные машины имеют доступ в интернет и все необходимые пакеты установлены.

---

## ✅ Установить на все машины (общие пакеты)

```bash
apt update && apt install -y \
  network-manager \
  net-tools \
  iproute2 \
  openssh-client \
  openssh-server \
  traceroute \
  curl \
  sudo
```

---

## 📌 Установить на конкретные машины

| Машина     | Назначение                                | Пакеты к установке                                                                 |
|------------|-------------------------------------------|-------------------------------------------------------------------------------------|
| **ISP**    | Интернет-шлюз и маршрутизатор             | `nftables`, `network-manager`                                                      |
| **HQ-RTR** | Основной маршрутизатор, DHCP, GRE, OSPF   | `nftables`, `network-manager`, `isc-dhcp-server`, `frr`, `frr-pythontools`         |
| **BR-RTR** | Резервный маршрутизатор, GRE, OSPF        | `nftables`, `network-manager`, `frr`, `frr-pythontools`                            |
| **HQ-SRV** | Сервер HQ: DNS, NFS, RAID, Chrony         | `bind9`, `bind9-utils`, `nfs-kernel-server`, `mdadm`, `chrony`                     |
| **BR-SRV** | Сервер BR: Samba AD DC, Ansible, Docker   | `samba`, `krb5-config`, `winbind`, `smbclient`, `ansible`, `docker.io`, `docker-compose` |
| **HQ-CLI** | Клиент: вход в домен                      | `realmd`, `sssd`, `sssd-tools`, `adcli`, `packagekit`, `samba-common-bin`          |

---

## Модуль 1: Настройка сетевой инфраструктуры

**Время выполнения**: 1 ч. 00 мин 

### 1. Присвоение имён хостам
```bash
hostnamectl set-hostname hq-rtr.au-team.irpo
```

### 2. Настройка интерфейсов с nmcli
```bash
nmcli con add type ethernet con-name WAN ifname ens18 ip4 172.16.4.2/28 gw4 172.16.4.1
nmcli con mod WAN ipv4.dns 8.8.8.8
nmcli con up WAN

nmcli con add type ethernet con-name LAN ifname ens19 ip4 192.168.100.1/24
nmcli con up LAN
```

### 3. Конфигурация ISP
```bash
nmcli con add type ethernet con-name TO-HQ ifname ens18 ip4 172.16.4.1/28
nmcli con up TO-HQ

nmcli con add type ethernet con-name TO-BR ifname ens19 ip4 172.16.5.1/28
nmcli con up TO-BR

nmcli con add type ethernet con-name WAN-DHCP ifname ens20
nmcli con up WAN-DHCP
```

### 4. Включение IP-форвардинга и NAT (nftables)
Для настройки NAT и включения IP-форвардинга на ваших маршрутизаторах (HQ-RTR и BR-RTR) выполните следующие шаги:

---

## 1. Включаем IP-форвардинг

1. Отредактируйте файл `/etc/sysctl.conf`:
   ```bash
   echo "net.ipv4.ip_forward=1" > /etc/sysctl.conf
   ```
2. Примените настройку:
   ```bash
   sysctl -p
   ```
3. Убедитесь, что forwarding включён:
   ```bash
   sysctl net.ipv4.ip_forward
   # должно быть net.ipv4.ip_forward = 1
   ```

---

# 2. Настройка NAT (MASQUERADE) через iptables

Предположим, ваш «внешний» интерфейс для выхода в Интернет — `enp0s8`. Тогда:

## HQ-RTR

```bash
# Filter таблица
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

iptables -A FORWARD -i enp0s9  -o enp0s8 -j ACCEPT
iptables -A FORWARD -i enp0s10 -o enp0s8 -j ACCEPT
iptables -A FORWARD -i enp0s9  -o enp0s9 -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i enp0s10 -o enp0s10 -m state --state RELATED,ESTABLISHED -j ACCEPT

# NAT таблица
iptables -t nat -P PREROUTING ACCEPT
iptables -t nat -P INPUT ACCEPT
iptables -t nat -P OUTPUT ACCEPT
iptables -t nat -P POSTROUTING ACCEPT

iptables -t nat -A POSTROUTING -s 192.168.1.0/26 -o enp0s8 -j MASQUERADE
iptables -t nat -A POSTROUTING -s 192.168.2.0/26 -o enp0s8 -j MASQUERADE
```

---

## BR-RTR

```bash
# Filter таблица
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT

iptables -A FORWARD -i enp0s9 -o enp0s8 -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -i enp0s9 -o enp0s9 -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT

# NAT таблица
iptables -t nat -P PREROUTING ACCEPT
iptables -t nat -P INPUT ACCEPT
iptables -t nat -P OUTPUT ACCEPT
iptables -t nat -P POSTROUTING ACCEPT

iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o enp0s8 -j MASQUERADE
---

## 3. Сохранение правил iptables на Debian

Чтобы правила автоматически подгружались после перезагрузки:

1. Установите пакет для сохранения правил:
   ```bash
   apt update
   apt install -y iptables-persistent
   ```
2. При установке подтвердите сохранение текущих правил.  
   Если правила появились позже, сохраните вручную:
   ```bash
   netfilter-persistent save
   ```

---

## 4. Проверка

1. Просмотреть текущие NAT-правила:
   ```bash
   iptables -t nat -L -n -v
   ```
2. Просмотреть правила FORWARD:
   ```bash
   iptables -L FORWARD -n -v
   ```
3. Проверить доступность из LAN:
   ```bash
   # на HQ-CLI или любой машине из подсети
   ping -c3 8.8.8.8
   curl -I http://example.com
   ```


### 5. GRE-туннель
 Настройка GRE-туннеля через nmtui

В этом файле приведены точные значения полей для двух маршрутизаторов: HQ-RTR и BR-RTR, по текущей схеме.

---

## 1. HQ-RTR (172.16.4.2 ↔ 172.16.5.2)

1. Запустите `nmtui` → **Edit a connection** → выберите ваш IP-tunnel (например, `gre-tun0`) → **Edit**:

   ```text
   Profile name:  gre-tun0
   Device:        tun0

   ── IP tunnel ────────────────
   Mode:          GRE
   Parent:        ens18
   Local IP:      172.16.4.2
   Remote IP:     172.16.5.2
   MTU:           1400

   ── IPv4 CONFIGURATION ───────
   Method:        Manual
   Addresses:     10.10.10.1/30     # туннельный адрес
   Gateway:       (leave empty)    # шлюз не нужен
   DNS servers:   (leave empty)
   ```

2. Сохраните и **Activate** → `gre-tun0`.

---

## 2. BR-RTR (172.16.5.2 ↔ 172.16.4.2)

1. В `nmtui` → **Edit a connection** → ваш туннель → **Edit**:

   ```text
   Profile name:  gre-tun0
   Device:        tun0

   ── IP tunnel ────────────────
   Mode:          GRE
   Parent:        ens18
   Local IP:      172.16.5.2
   Remote IP:     172.16.4.2
   MTU:           1400

   ── IPv4 CONFIGURATION ───────
   Method:        Manual
   Addresses:     10.10.10.2/30
   Gateway:       (leave empty)
   DNS servers:   (leave empty)
   ```

2. Сохраните и **Activate** → `gre-tun0`.

---

После этого интерфейс `tun0` будет автоматически подниматься с нужными параметрами и настройки сохранятся при перезагрузке NetworkManager.


### 6. OSPF (FRR)
```bash
apt install -y frr frr-pythontools
# /etc/frr/daemons: zebra=yes ospfd=yes
systemctl enable --now frr
vtysh <<EOF
conf t
router ospf
 ospf router-id 1.1.1.1       # HQ (BR: 2.2.2.2)
 network 10.10.10.0/30 area 0
 network 192.168.100.0/24 area 0
 passive-interface ens19
 area 0 authentication message-digest
 interface tun0
  ip ospf message-digest-key 1 md5 MySecretPass
exit
write
EOF
```

### 7. DHCP-сервер (HQ-RTR DHCP)
```bash
apt install -y isc-dhcp-server
authoritative;

option domain-name "au-team.irpo";
option domain-name-servers 192.168.1.2, 8.8.8.8;

subnet 192.168.2.0 netmask 255.255.255.240 {
    range 192.168.2.2 192.168.2.14;
    option routers 192.168.2.1;
    option broadcast-address 192.168.2.15;
    option domain-search "au-team.irpo";
}

systemctl enable --now isc-dhcp-server
reboot
```

### 8. DNS-сервер (BIND9)
# Полная настройка BIND9 и резольвера на HQ-SRV

## 1. Установка BIND

```bash
apt update
apt install -y bind9 bind9-utils
```

---

## 2. Основная конфигурация `/etc/bind/named.conf.options`

Добавьте внутрь блока `options { … };` следующие строки:

```conf
options {
    directory "/var/cache/bind";

    recursion yes;
    allow-query { any; };
    forwarders {
        8.8.8.8;
        8.8.4.4;
    };
    dnssec-validation auto;
};
```

---

## 3. Описание зон в `/etc/bind/named.conf.local`

```conf
# Прямая зона
zone "au-team.irpo" {
    type master;
    file "/etc/bind/db.au-team.irpo";
};

# Обратные зоны
zone "1.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.1";
};

zone "2.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.2";
};

zone "0.168.192.in-addr.arpa" {
    type master;
    file "/etc/bind/db.192.168.0";
};
```

---

## 4. Содержимое файлов зон

### 4.1 `/etc/bind/db.au-team.irpo`

```zone
$TTL 604800
@    IN SOA  hq-srv.au-team.irpo. admin.au-team.irpo. ( 1 604800 86400 2419200 604800 )
;
@      IN NS    hq-srv.au-team.irpo.

; A-записи
hq-rtr  IN A     172.16.4.2
br-rtr  IN A     172.16.5.2
hq-srv  IN A     192.168.1.2
hq-cli  IN A     192.168.2.2
br-srv  IN A     192.168.0.2

; CNAME-записи
moodle  IN CNAME hq-rtr.au-team.irpo.
wiki    IN CNAME hq-rtr.au-team.irpo.
```

### 4.2 `/etc/bind/db.192.168.1`

```zone
$TTL 604800
@    IN SOA  hq-srv.au-team.irpo. admin.au-team.irpo. ( 2 604800 86400 2419200 604800 )
;
@      IN NS    hq-srv.au-team.irpo.

; PTR-записи
2      IN PTR   hq-srv.au-team.irpo.
```

### 4.3 `/etc/bind/db.192.168.2`

```zone
$TTL 604800
@    IN SOA  hq-srv.au-team.irpo. admin.au-team.irpo. ( 3 604800 86400 2419200 604800 )
;
@      IN NS    hq-srv.au-team.irpo.

; PTR-запись
2      IN PTR   hq-cli.au-team.irpo.
```

### 4.4 `/etc/bind/db.192.168.0`

```zone
$TTL 604800
@    IN SOA  hq-srv.au-team.irpo. admin.au-team.irpo. ( 4 604800 86400 2419200 604800 )
;
@      IN NS    hq-srv.au-team.irpo.

; PTR-запись
2      IN PTR   br-srv.au-team.irpo.
```

---

## 5. Проверка и запуск DNS

```bash
# Проверка синтаксиса
named-checkconf
named-checkzone au-team.irpo       /etc/bind/db.au-team.irpo
named-checkzone 1.168.192.in-addr.arpa /etc/bind/db.192.168.1
named-checkzone 2.168.192.in-addr.arpa /etc/bind/db.192.168.2
named-checkzone 0.168.192.in-addr.arpa /etc/bind/db.192.168.0

# Запуск и автозапуск
systemctl enable --now bind9
```

---

## 6. Настройка `resolv.conf`

Добавьте в начало `/etc/resolv.conf`:

```text
search au-team.irpo
nameserver 127.0.0.1
```

---

## 7. Тестирование

```bash
# Локальные запросы к BIND
dig @127.0.0.1 hq-rtr.au-team.irpo A +short
dig @127.0.0.1 -x 192.168.2.2 +short

# Проверка внешней резолюции через форвардер
dig @127.0.0.1 example.com +short

# Проверка ping по имени
ping -c3 hq-rtr.au-team.irpo
```

---

## Модуль 2: Организация сетевого администрирования

## 1. Настройка Samba на BR-SRV
```bash
sudo apt-get update && sudo apt-get install samba samba-common smbclient winbind libnss-winbind libpam-winbind
sudo samba-tool domain provision --realm=HQ.LOCAL --domain=HQ --server-role=dc --dns-backend=SAMBA_INTERNAL --adminpass='StrongAdminP@ssw0rd'
sudo systemctl enable smb nmb winbind
sudo systemctl start smb nmb winbind
sudo samba-tool group add hq
sudo samba-tool user create user1.hq Passw0rd123 --given-name="User One" --surname="HQ"
sudo samba-tool user create user2.hq Passw0rd123 --given-name="User Two" --surname="HQ"
sudo samba-tool user create user3.hq Passw0rd123 --given-name="User Three" --surname="HQ"
sudo samba-tool user create user4.hq Passw0rd123 --given-name="User Four" --surname="HQ"
sudo samba-tool user create user5.hq Passw0rd123 --given-name="User Five" --surname="HQ"
sudo samba-tool group addmembers hq "user1.hq","user2.hq","user3.hq","user4.hq","user5.hq"
while IFS=, read -r login pass fullname; do 
    sudo samba-tool user create "$login" "$pass" --fullname="$fullname"
    sudo samba-tool group addmembers hq "$login"
done < /opt/users.csv
```

## 2. Присоединение HQ-CLI к домену
```bash
sudo apt-get install samba-common realmd sssd adcli krb5-user libnss-winbind libpam-winbind
sudo realm join --user=Administrator HQ.LOCAL
getent passwd user1.hq@HQ.LOCAL
getent group hq@HQ.LOCAL
sudo systemctl restart sssd
sudo visudo  # добавить: %hq ALL=(ALL) NOPASSWD: /usr/bin/cat, /usr/bin/grep, /usr/bin/id
```

## 3. RAID5 и NFS на HQ-SRV
```bash
sudo apt-get install mdadm
sudo mdadm --create --verbose /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm.conf
sudo update-initramfs -u
sudo parted /dev/md0 --script mklabel gpt
sudo parted /dev/md0 --script mkpart primary ext4 0% 100%
sudo mkfs.ext4 /dev/md0p1
sudo mkdir -p /raid5
sudo mount /dev/md0p1 /raid5
# Добавить в /etc/fstab:
# UUID=<UUID> /raid5 ext4 defaults 0 0

sudo apt-get install nfs-kernel-server
sudo mkdir /raid5/nfs
sudo chown nobody:nogroup /raid5/nfs
# В /etc/exports:
/raid5/nfs 192.168.20.0/24(rw,sync,no_subtree_check,no_root_squash)
sudo exportfs -r
exportfs -v
showmount -e localhost

# На HQ-CLI:
sudo apt-get install nfs-common
sudo mkdir -p /mnt/nfs
# В /etc/fstab:
# 192.168.20.10:/raid5/nfs /mnt/nfs nfs defaults,_netdev 0 0
sudo mount -a
touch /mnt/nfs/testfile && echo "NFS OK" > /mnt/nfs/testfile
```

## 4. Chrony
```bash
# На HQ-RTR:
sudo apt-get install chrony
# В /etc/chrony.conf: добавить local stratum 5 и allow 192.168.20.0/24, allow 192.168.10.0/24
sudo systemctl enable chronyd
sudo systemctl restart chronyd

# На клиентах:
sudo apt-get install chrony
# В /etc/chrony.conf: закомментировать pool*, добавить server 192.168.20.1 iburst
sudo systemctl enable chronyd
sudo systemctl restart chronyd
chronyc tracking
chronyc sources
```

## 5. Ansible на BR-SRV
```bash
sudo apt-get install ansible
ssh-keygen -t rsa
ssh-copy-id sshuser@192.168.20.10
ssh-copy-id sshuser@192.168.20.20
ssh-copy-id net_admin@192.168.20.1
ssh-copy-id net_admin@192.168.10.1
# В /etc/ansible/hosts: добавить hq-srv, hq-cli, hq-rtr, br-rtr
# В /etc/ansible/ansible.cfg: host_key_checking = False
ansible all -m ping
```

## 6. Docker MediaWiki на BR-SRV
```bash
sudo apt-get install docker.io docker-compose-plugin
sudo apt-get install docker-compose
# Создать ~/wiki.yml и ~/LocalSettings.php
docker-compose -f wiki.yml up -d
docker-compose ps
docker-compose logs -f
curl http://192.168.10.10:8080
```

## 7. NAT на BR-RTR (Wiki)
```bash
sudo iptables -t nat -A PREROUTING -i eth0 -p tcp --dport 80 -j DNAT --to 192.168.10.10:8080
sudo iptables -t nat -A POSTROUTING -o eth1 -p tcp -d 192.168.10.10 --dport 8080 -j MASQUERADE
# Сохранить правила: iptables-save > /etc/iptables/rules.v4
```

## 8. Moodle на HQ-SRV
```bash
sudo apt-get install apache2 php libapache2-mod-php
sudo apt-get install php-mysql php-xml php-gd php-intl php-curl php-zip php-mbstring
sudo apt-get install mariadb-server
sudo mysql_secure_installation
sudo mysql -u root -p
# В mysql:
CREATE DATABASE moodledb DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'moodle'@'localhost' IDENTIFIED BY 'P@ssw0rd';
GRANT ALL PRIVILEGES ON moodledb.* TO 'moodle'@'localhost';
FLUSH PRIVILEGES;
EXIT;
cd /tmp
wget https://download.moodle.org/download.php/direct/stable401/moodle-4.1.5.tgz -O moodle.tgz
sudo tar -xzvf moodle.tgz -C /var/www/
sudo mv /var/www/moodle /var/www/html/moodle
sudo mkdir /var/moodledata
sudo chown -R www-data:www-data /var/moodledata
sudo chmod 0777 /var/moodledata
sudo chown -R www-data:www-data /var/www/html/moodle
sudo a2ensite moodle.conf
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
```

## 9. Nginx Reverse Proxy на HQ-RTR
```bash
sudo apt-get install nginx
# Создать /etc/nginx/sites-available/moodle.au-team.irpo.conf и wiki.au-team.irpo.conf
sudo ln -s /etc/nginx/sites-available/moodle.au-team.irpo.conf /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/wiki.au-team.irpo.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default
sudo nginx -t && sudo systemctl reload nginx
```

## 10. Яндекс.Браузер на HQ-CLI
```bash
sudo apt-get update
sudo apt-get install yandex-browser-stable
yandex-browser --version
```

