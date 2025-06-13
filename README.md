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

## Модуль 1: Настройка сетевой инфраструктуры

**Время выполнения**: 1 ч. 00 мин 

### 1. Присвоение имён хостам
```bash
hostnamectl set-hostname hq-rtr.au-team.irpo
```

### 2. Настройка интерфейсов с nmcli
<details>
<summary>Пример для HQ-RTR</summary>
```bash
nmcli con add type ethernet con-name WAN ifname ens18 ip4 172.16.4.2/28 gw4 172.16.4.1
nmcli con mod WAN ipv4.dns 8.8.8.8
nmcli con up WAN

nmcli con add type ethernet con-name LAN ifname ens19 ip4 192.168.100.1/24
nmcli con up LAN
```
</details>

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
```bash
echo "net.ipv4.ip_forward=1" > /etc/sysctl.d/99-forwarding.conf
sysctl -p /etc/sysctl.d/99-forwarding.conf

apt update && apt install -y nftables
# В /etc/nftables.conf добавить:
# chain postrouting { type nat hook postrouting priority 100; policy accept; oifname ens20 masquerade }
systemctl enable --now nftables
```

### 5. GRE-туннель
```bash
# HQ-RTR
ip tunnel add tun0 mode gre remote 172.16.5.2 local 172.16.4.2 ttl 255
ip addr add 10.10.10.1/30 dev tun0
ip link set tun0 up

# BR-RTR
ip tunnel add tun0 mode gre remote 172.16.4.2 local 172.16.5.2 ttl 255
ip addr add 10.10.10.2/30 dev tun0
ip link set tun0 up
```

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

### 7. DHCP-сервер (ISC DHCP)
```bash
apt install -y isc-dhcp-server
# /etc/dhcp/dhcpd.conf: опции домена, подсеть 192.168.100.0/24
# INTERFACESv4="ens19"
systemctl enable --now isc-dhcp-server
```

### 8. DNS-сервер (BIND9)
```bash
apt install -y bind9 bind9-utils
# /etc/bind/named.conf.options: forwarders { 8.8.8.8; 8.8.4.4; }
# /etc/bind/named.conf.local: зоны au-team.irpo и 100.168.192.in-addr.arpa
named-checkconf && named-checkzone au-team.irpo /etc/bind/db.au-team.irpo
systemctl enable --now bind9
```

---

## Модуль 2: Организация сетевого администрирования

**Время выполнения**: 1 ч. 30 мин 

### 1. Samba AD DC (BR-SRV)
```bash
apt install -y samba krb5-config winbind smbclient
samba-tool domain provision --use-rfc2307 --interactive
cp /var/lib/samba/private/krb5.conf /etc/
systemctl unmask samba-ad-dc && systemctl enable --now samba-ad-dc
```

### 2. Вход клиента в домен (HQ-CLI)
```bash
apt install -y realmd sssd sssd-tools adcli packagekit samba-common-bin
realm join --user=administrator au-team.irpo
```

### 3. RAID 5 + NFS (HQ-SRV)
```bash
apt install -y mdadm nfs-kernel-server
mdadm --create /dev/md0 --level=5 --raid-devices=3 /dev/sdb /dev/sdc /dev/sdd
mkfs.ext4 /dev/md0 && echo "/dev/md0 /raid5 ext4 defaults 0 0" >> /etc/fstab
mkdir -p /raid5/nfs && chown nobody:nogroup /raid5/nfs
echo "/raid5/nfs 192.168.100.0/24(rw,sync,no_subtree_check)" >> /etc/exports
exportfs -a && systemctl restart nfs-kernel-server
```

### 4. Chrony NTP
```bash
apt install -y chrony
# /etc/chrony/chrony.conf: local stratum 5, allow 192.168.0.0/16
systemctl restart chrony
# Клиенты: server hq-rtr.au-team.irpo iburst
```

### 5. Ansible (BR-SRV)
```bash
apt install -y ansible
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

ansible all -m ping
```

### 6. Docker + MediaWiki (BR-SRV)
```bash
apt install -y docker.io docker-compose
docker run -d -p 8080:80 --name mediawiki mediawiki
```
