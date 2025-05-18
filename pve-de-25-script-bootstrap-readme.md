# Единый скрипт первоначальной настройки сети и установки пакетов по ДЭ ССА за 2025 год (Bootstrap-скрипт)

**Важно:** Эти скрипты предназначены для **первоначальной настройки базовой сетевой связности, установки всех необходимых пакетов для Модулей 1 и 2, и скачивания основного скрипта автоматизации конфигурации**. Все остальные конфигурации из Модуля 1 и Модуля 2 должны выполняться вашим основным скриптом (`pve-de-25-script-auto-deployer.sh`), который теперь будет заниматься только настройкой уже установленных служб.

**Предпосылки:**
*   Все ВМ имеют установленную ОС Альт Linux.
*   Вы работаете от имени пользователя `root`.
*   Сетевые интерфейсы называются `ens18`, `ens19`, `ens20` и т.д. (согласно топологии вашего стенда для начальной загрузки).
*   URL для скачивания вашего основного скрипта автоматизации: `https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-deployer.sh`

---

## **Порядок действий для первоначальной настройки стенда (установка пакетов и скачивание основного скрипта):**

Сначала вам необходимо последовательно выполнить предоставленные ниже начальные ("bootstrap") скрипты на каждой виртуальной машине. **Для этих начальных скриптов порядок выполнения следующий: ISP → BR-RTR → HQ-RTR → BR-SRV → HQ-SRV → HQ-CLI.** Этот порядок обеспечит поэтапную настройку сетевой связности и доступа в Интернет.

**Ключевые изменения в этих bootstrap-скриптах:**
1.  **Установка всех пакетов:** Каждый bootstrap-скрипт теперь включает команды для установки **всех** пакетов, которые потребуются данной виртуальной машине для выполнения **всех** задач из Модуля 1 и Модуля 2. Это делается на этапе, когда гарантирован доступ в интернет через публичные DNS-серверы, что решает проблему "курицы и яйца" с DNS на последующих этапах конфигурации.
2.  **Скачивание основного скрипта:** После установки пакетов скачивается основной скрипт автоматизации (`pve-de-25-script-auto-deployer.sh`) в директорию `/root/`.

**Внимание:** Из-за установки большого количества пакетов выполнение этих bootstrap-скриптов может занять значительное время. Пожалуйста, будьте терпеливы.

---

### 1. Настройка ISP (Интернет-провайдер)

**Сначала выполните эти команды на ISP:**
```bash
#!/bin/bash
set -e

echo "--- Начало настройки ISP (Bootstrap) ---"

echo ">>> Установка имени хоста на isp.au-team.irpo..."
hostnamectl set-hostname isp.au-team.irpo

echo ">>> Настройка сетевого интерфейса ens18 (DHCP)..."
mkdir -p /etc/net/ifaces/ens18; find /etc/net/ifaces/ens18 -mindepth 1 -delete
cat <<'EOF' > /etc/net/ifaces/ens18/options
BOOTPROTO=dhcp
TYPE=eth
EOF

echo ">>> Настройка сетевого интерфейса ens19 (Статический IP: 172.16.4.1/28)..."
mkdir -p /etc/net/ifaces/ens19; find /etc/net/ifaces/ens19 -mindepth 1 -delete
echo 'TYPE=eth' > /etc/net/ifaces/ens19/options
echo '172.16.4.1/28' > /etc/net/ifaces/ens19/ipv4address

echo ">>> Настройка сетевого интерфейса ens20 (Статический IP: 172.16.5.1/28)..."
mkdir -p /etc/net/ifaces/ens20; find /etc/net/ifaces/ens20 -mindepth 1 -delete
echo 'TYPE=eth' > /etc/net/ifaces/ens20/options
echo '172.16.5.1/28' > /etc/net/ifaces/ens20/ipv4address

echo ">>> Включение IP-форвардинга в /etc/net/sysctl.conf..."
if ! grep -q "^net.ipv4.ip_forward[[:space:]]*=[[:space:]]*1" /etc/net/sysctl.conf; then
    if grep -q "^[#[:space:]]*net.ipv4.ip_forward[[:space:]]*=[[:space:]]*0" /etc/net/sysctl.conf; then
        sed -i 's/^[#[:space:]]*net.ipv4.ip_forward[[:space:]]*=[[:space:]]*0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
    elif ! grep -q "^net.ipv4.ip_forward" /etc/net/sysctl.conf; then
        echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
    fi
fi
echo ">>> Применение настройки IP-форвардинга немедленно..."
sysctl -w net.ipv4.ip_forward=1

echo ">>> Перезапуск сетевой службы..."
systemctl restart network
echo ">>> Ожидание стабилизации сети (15 секунд)..."
sleep 15

echo ">>> Обновление списка пакетов и установка необходимых пакетов для ISP (iptables, tzdata, wget)... Это может занять время."
apt-get update && apt-get install -y \
    iptables \
    tzdata \
    wget \
    bc
echo ">>> Установка пакетов для ISP завершена."

echo ">>> Настройка NAT с использованием iptables..."
iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
echo ">>> Сохранение правил iptables в /etc/sysconfig/iptables..."
iptables-save > /etc/sysconfig/iptables
echo ">>> Включение и запуск службы iptables..."
systemctl enable --now iptables
systemctl restart iptables

echo ">>> Проверка связи с интернетом (8.8.8.8)..."
ping -c 3 8.8.8.8 || echo "ПРЕДУПРЕЖДЕНИЕ: Интернет недоступен. Проверьте DHCP на ens18."

echo ">>> Скачивание основного скрипта автоматизации (pve-de-25-script-auto-deployer.sh)..."
wget -O /root/pve-de-25-script-auto-deployer.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-deployer.sh
echo ">>> Предоставление прав на выполнение основному скрипту автоматизации..."
chmod +x /root/pve-de-25-script-auto-deployer.sh

echo ">>> Скачивание скрипта проверки (pve-de-25-script-auto-compliance-checker.sh)..."
wget -O /root/pve-de-25-script-auto-compliance-checker.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-compliance-checker.sh
echo ">>> Предоставление прав на выполнение скрипту проверки..."
chmod +x /root/pve-de-25-script-auto-compliance-checker.sh

echo "--- Bootstrap для ISP завершен. Пакеты установлены, основной скрипт скачан в /root/ ---"
echo "--- Теперь можно приступать к полной настройке стенда с помощью /root/pve-de-25-script-auto-deployer.sh ---"
```

---

### 2. Настройка HQ-RTR (Маршрутизатор головного офиса)

**Сначала выполните эти команды на HQ-RTR:**
```bash
#!/bin/bash
set -e

echo "--- Начало настройки HQ-RTR (Bootstrap) ---"

echo ">>> Установка имени хоста на hq-rtr.au-team.irpo..."
hostnamectl set-hostname hq-rtr.au-team.irpo

echo ">>> Настройка сетевого интерфейса ens18 (WAN - IP: 172.16.4.4/28, GW: 172.16.4.1, DNS)..."
mkdir -p /etc/net/ifaces/ens18; find /etc/net/ifaces/ens18 -mindepth 1 -delete
cat <<'EOF' > /etc/net/ifaces/ens18/options
TYPE=eth
EOF
echo '172.16.4.4/28' > /etc/net/ifaces/ens18/ipv4address
echo 'default via 172.16.4.1' > /etc/net/ifaces/ens18/ipv4route
cat <<'EOF' > /etc/net/ifaces/ens18/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

echo ">>> Настройка базового сетевого интерфейса ens19 (для VLANов)..."
mkdir -p /etc/net/ifaces/ens19; find /etc/net/ifaces/ens19 -mindepth 1 -delete
echo 'TYPE=eth' > /etc/net/ifaces/ens19/options

echo ">>> Настройка VLAN100 на ens19 (IP: 192.168.1.1/26)..."
mkdir -p /etc/net/ifaces/vlan100; find /etc/net/ifaces/vlan100 -mindepth 1 -delete
cat <<'EOF' > /etc/net/ifaces/vlan100/options
TYPE=vlan
HOST=ens19
VID=100
EOF
echo '192.168.1.1/26' > /etc/net/ifaces/vlan100/ipv4address

echo ">>> Настройка VLAN200 на ens19 (IP: 192.168.2.1/28)..."
mkdir -p /etc/net/ifaces/vlan200; find /etc/net/ifaces/vlan200 -mindepth 1 -delete
cat <<'EOF' > /etc/net/ifaces/vlan200/options
TYPE=vlan
HOST=ens19
VID=200
EOF
echo '192.168.2.1/28' > /etc/net/ifaces/vlan200/ipv4address

echo ">>> Настройка VLAN999 на ens19 (IP: 192.168.99.1/29)..."
mkdir -p /etc/net/ifaces/vlan999; find /etc/net/ifaces/vlan999 -mindepth 1 -delete
cat <<'EOF' > /etc/net/ifaces/vlan999/options
TYPE=vlan
HOST=ens19
VID=999
EOF
echo '192.168.99.1/29' > /etc/net/ifaces/vlan999/ipv4address

echo ">>> Включение IP-форвардинга в /etc/net/sysctl.conf..."
if ! grep -q "^net.ipv4.ip_forward[[:space:]]*=[[:space:]]*1" /etc/net/sysctl.conf; then
    if grep -q "^[#[:space:]]*net.ipv4.ip_forward[[:space:]]*=[[:space:]]*0" /etc/net/sysctl.conf; then
        sed -i 's/^[#[:space:]]*net.ipv4.ip_forward[[:space:]]*=[[:space:]]*0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
    elif ! grep -q "^net.ipv4.ip_forward" /etc/net/sysctl.conf; then
        echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
    fi
fi
echo ">>> Применение настройки IP-форвардинга немедленно..."
sysctl -w net.ipv4.ip_forward=1

echo ">>> Перезапуск сетевой службы..."
systemctl restart network
echo ">>> Ожидание стабилизации сети (15 секунд)..."
sleep 15

echo ">>> Обновление списка пакетов и установка необходимых пакетов для HQ-RTR (iptables, frr, dnsmasq, chrony, nginx, wget)... Это может занять время."
apt-get update && apt-get install -y \
    iptables \
    frr \
    dnsmasq \
    chrony \
    nginx \
    wget
echo ">>> Установка пакетов для HQ-RTR завершена."

echo ">>> Настройка NAT с использованием iptables..."
iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
echo ">>> Сохранение правил iptables в /etc/sysconfig/iptables..."
iptables-save > /etc/sysconfig/iptables
echo ">>> Включение и запуск службы iptables..."
systemctl enable --now iptables
systemctl restart iptables

echo ">>> Проверка связи со шлюзом (172.16.4.1)..."
ping -c 3 172.16.4.1 || echo "ПРЕДУПРЕЖДЕНИЕ: Шлюз 172.16.4.1 недоступен. ISP должен быть настроен."
echo ">>> Проверка связи с интернетом (8.8.8.8)..."
ping -c 3 8.8.8.8 || echo "ПРЕДУПРЕЖДЕНИЕ: Интернет недоступен."

echo ">>> Скачивание основного скрипта автоматизации (pve-de-25-script-auto-deployer.sh)..."
wget -O /root/pve-de-25-script-auto-deployer.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-deployer.sh
echo ">>> Предоставление прав на выполнение основному скрипту автоматизации..."
chmod +x /root/pve-de-25-script-auto-deployer.sh

echo ">>> Скачивание скрипта проверки (pve-de-25-script-auto-compliance-checker.sh)..."
wget -O /root/pve-de-25-script-auto-compliance-checker.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-compliance-checker.sh
echo ">>> Предоставление прав на выполнение скрипту проверки..."
chmod +x /root/pve-de-25-script-auto-compliance-checker.sh

echo "--- Bootstrap для HQ-RTR завершен. Пакеты установлены, основной скрипт скачан в /root/ ---"
echo "--- Теперь можно приступать к полной настройке стенда с помощью /root/pve-de-25-script-auto-deploy.sh ---"
```

---

### 3. Настройка BR-RTR (Маршрутизатор филиала)

**Сначала выполните эти команды на BR-RTR:**
```bash
#!/bin/bash
set -e

echo "--- Начало настройки BR-RTR (Bootstrap) ---"

echo ">>> Установка имени хоста на br-rtr.au-team.irpo..."
hostnamectl set-hostname br-rtr.au-team.irpo

echo ">>> Настройка сетевого интерфейса ens18 (WAN - IP: 172.16.5.5/28, GW: 172.16.5.1, DNS)..."
mkdir -p /etc/net/ifaces/ens18; find /etc/net/ifaces/ens18 -mindepth 1 -delete
cat <<'EOF' > /etc/net/ifaces/ens18/options
TYPE=eth
EOF
echo '172.16.5.5/28' > /etc/net/ifaces/ens18/ipv4address
echo 'default via 172.16.5.1' > /etc/net/ifaces/ens18/ipv4route
cat <<'EOF' > /etc/net/ifaces/ens18/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

echo ">>> Настройка сетевого интерфейса ens19 (LAN - IP: 192.168.3.1/27)..."
mkdir -p /etc/net/ifaces/ens19; find /etc/net/ifaces/ens19 -mindepth 1 -delete
echo 'TYPE=eth' > /etc/net/ifaces/ens19/options
echo '192.168.3.1/27' > /etc/net/ifaces/ens19/ipv4address

echo ">>> Включение IP-форвардинга в /etc/net/sysctl.conf..."
if ! grep -q "^net.ipv4.ip_forward[[:space:]]*=[[:space:]]*1" /etc/net/sysctl.conf; then
    if grep -q "^[#[:space:]]*net.ipv4.ip_forward[[:space:]]*=[[:space:]]*0" /etc/net/sysctl.conf; then
        sed -i 's/^[#[:space:]]*net.ipv4.ip_forward[[:space:]]*=[[:space:]]*0/net.ipv4.ip_forward = 1/g' /etc/net/sysctl.conf
    elif ! grep -q "^net.ipv4.ip_forward" /etc/net/sysctl.conf; then
        echo "net.ipv4.ip_forward = 1" >> /etc/net/sysctl.conf
    fi
fi
echo ">>> Применение настройки IP-форвардинга немедленно..."
sysctl -w net.ipv4.ip_forward=1

echo ">>> Перезапуск сетевой службы..."
systemctl restart network
echo ">>> Ожидание стабилизации сети (15 секунд)..."
sleep 15

echo ">>> Обновление списка пакетов и установка необходимых пакетов для BR-RTR (iptables, frr, chrony, wget)... Это может занять время."
apt-get update && apt-get install -y \
    iptables \
    frr \
    chrony \
    wget
echo ">>> Установка пакетов для BR-RTR завершена."

echo ">>> Настройка NAT с использованием iptables..."
iptables -t nat -F POSTROUTING
iptables -t nat -A POSTROUTING -o ens18 -j MASQUERADE
iptables -P INPUT ACCEPT
iptables -P FORWARD ACCEPT
iptables -P OUTPUT ACCEPT
echo ">>> Сохранение правил iptables в /etc/sysconfig/iptables..."
iptables-save > /etc/sysconfig/iptables
echo ">>> Включение и запуск службы iptables..."
systemctl enable --now iptables
systemctl restart iptables

echo ">>> Проверка связи со шлюзом (172.16.5.1)..."
ping -c 3 172.16.5.1 || echo "ПРЕДУПРЕЖДЕНИЕ: Шлюз 172.16.5.1 недоступен. ISP должен быть настроен."
echo ">>> Проверка связи с интернетом (8.8.8.8)..."
ping -c 3 8.8.8.8 || echo "ПРЕДУПРЕЖДЕНИЕ: Интернет недоступен."

echo ">>> Скачивание основного скрипта автоматизации (pve-de-25-script-auto-deploy.sh)..."
wget -O /root/pve-de-25-script-auto-deploy.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-deploy.sh
echo ">>> Предоставление прав на выполнение основному скрипту автоматизации..."
chmod +x /root/pve-de-25-script-auto-deploy.sh

echo ">>> Скачивание скрипта проверки (pve-de-25-script-auto-compliance-checker.sh)..."
wget -O /root/pve-de-25-script-auto-compliance-checker.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-compliance-checker.sh
echo ">>> Предоставление прав на выполнение скрипту проверки..."
chmod +x /root/pve-de-25-script-auto-compliance-checker.sh

echo "--- Bootstrap для BR-RTR завершен. Пакеты установлены, основной скрипт скачан в /root/ ---"
echo "--- Теперь можно приступать к полной настройке стенда с помощью /root/pve-de-25-script-auto-deploy.sh ---"
```

---

### 4. Настройка HQ-SRV (Сервер головного офиса)

**Сначала выполните эти команды на HQ-SRV:**
```bash
#!/bin/bash
set -e

echo "--- Начало настройки HQ-SRV (Bootstrap) ---"

echo ">>> Установка имени хоста на hq-srv.au-team.irpo..."
hostnamectl set-hostname hq-srv.au-team.irpo

echo ">>> Настройка сетевого интерфейса ens18 (IP: 192.168.1.10/26, GW: 192.168.1.1, DNS)..."
mkdir -p /etc/net/ifaces/ens18; find /etc/net/ifaces/ens18 -mindepth 1 -delete
cat <<'EOF' > /etc/net/ifaces/ens18/options
TYPE=eth
EOF
echo '192.168.1.10/26' > /etc/net/ifaces/ens18/ipv4address
echo 'default via 192.168.1.1' > /etc/net/ifaces/ens18/ipv4route
cat <<'EOF' > /etc/net/ifaces/ens18/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

echo ">>> Перезапуск сетевой службы..."
systemctl restart network
echo ">>> Ожидание стабилизации сети (10 секунд)..."
sleep 10

echo ">>> Обновление списка пакетов и установка необходимых пакетов для HQ-SRV... Это может занять ОЧЕНЬ много времени."
apt-get update && apt-get install -y \
    dnsmasq \
    chrony \
    mdadm \
    e2fsprogs \
    nfs-utils \
    apache2 \
    mariadb-server \
    php8.2 \
    apache2-mod_php8.2 \
    php8.2-gd \
    php8.2-curl \
    php8.2-intl \
    php8.2-mysqli \
    php8.2-xml \
    php8.2-xmlrpc \
    php8.2-zip \
    php8.2-soap \
    php8.2-mbstring \
    php8.2-opcache \
    php8.2-json \
    php8.2-ldap \
    php8.2-xmlreader \
    php8.2-fileinfo \
    php8.2-sodium \
    unzip \
    curl \
    wget
echo ">>> Установка пакетов для HQ-SRV завершена."

echo ">>> Проверка связи со шлюзом (192.168.1.1)..."
ping -c 3 192.168.1.1 || echo "ПРЕДУПРЕЖДЕНИЕ: Шлюз 192.168.1.1 недоступен. HQ-RTR должен быть настроен."
echo ">>> Проверка связи с интернетом (8.8.8.8)..."
ping -c 3 8.8.8.8 || echo "ПРЕДУПРЕЖДЕНИЕ: Интернет недоступен."

echo ">>> Скачивание основного скрипта автоматизации (pve-de-25-script-auto-deploy.sh)..."
wget -O /root/pve-de-25-script-auto-deploy.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-deploy.sh
echo ">>> Предоставление прав на выполнение основному скрипту автоматизации..."
chmod +x /root/pve-de-25-script-auto-deploy.sh

echo ">>> Скачивание скрипта проверки (pve-de-25-script-auto-compliance-checker.sh)..."
wget -O /root/pve-de-25-script-auto-compliance-checker.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-compliance-checker.sh
echo ">>> Предоставление прав на выполнение скрипту проверки..."
chmod +x /root/pve-de-25-script-auto-compliance-checker.sh

echo "--- Bootstrap для HQ-SRV завершен. Пакеты установлены, основной скрипт скачан в /root/ ---"
echo "--- Теперь можно приступать к полной настройке стенда с помощью /root/pve-de-25-script-auto-deploy.sh ---"
```

---

### 5. Настройка BR-SRV (Сервер филиала)

**Сначала выполните эти команды на BR-SRV:**
```bash
#!/bin/bash
set -e

echo "--- Начало настройки BR-SRV (Bootstrap) ---"

echo ">>> Установка имени хоста на br-srv.au-team.irpo..."
hostnamectl set-hostname br-srv.au-team.irpo

echo ">>> Настройка сетевого интерфейса ens18 (IP: 192.168.3.10/27, GW: 192.168.3.1, DNS)..."
mkdir -p /etc/net/ifaces/ens18; find /etc/net/ifaces/ens18 -mindepth 1 -delete
cat <<'EOF' > /etc/net/ifaces/ens18/options
TYPE=eth
EOF
echo '192.168.3.10/27' > /etc/net/ifaces/ens18/ipv4address
echo 'default via 192.168.3.1' > /etc/net/ifaces/ens18/ipv4route
cat <<'EOF' > /etc/net/ifaces/ens18/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

echo ">>> Перезапуск сетевой службы..."
systemctl restart network
echo ">>> Ожидание стабилизации сети (10 секунд)..."
sleep 10

echo ">>> Обновление списка пакетов и установка необходимых пакетов для BR-SRV... Это может занять ОЧЕНЬ много времени."
apt-get update && apt-get install -y \
    chrony \
    task-samba-dc \
    krb5-clients \
    krb5-workstation \
    bind-utils \
    openssh-clients \
    ansible \
    docker-engine \
    docker-compose \
    wget
echo ">>> Установка пакетов для BR-SRV завершена."

echo ">>> Проверка связи со шлюзом (192.168.3.1)..."
ping -c 3 192.168.3.1 || echo "ПРЕДУПРЕЖДЕНИЕ: Шлюз 192.168.3.1 недоступен. BR-RTR должен быть настроен."
echo ">>> Проверка связи с интернетом (8.8.8.8)..."
ping -c 3 8.8.8.8 || echo "ПРЕДУПРЕЖДЕНИЕ: Интернет недоступен."

echo ">>> Скачивание основного скрипта автоматизации (pve-de-25-script-auto-deploy.sh)..."
wget -O /root/pve-de-25-script-auto-deploy.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-deploy.sh
echo ">>> Предоставление прав на выполнение основному скрипту автоматизации..."
chmod +x /root/pve-de-25-script-auto-deploy.sh

echo ">>> Скачивание скрипта проверки (pve-de-25-script-auto-compliance-checker.sh)..."
wget -O /root/pve-de-25-script-auto-compliance-checker.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-compliance-checker.sh
echo ">>> Предоставление прав на выполнение скрипту проверки..."
chmod +x /root/pve-de-25-script-auto-compliance-checker.sh

echo "--- Bootstrap для BR-SRV завершен. Пакеты установлены, основной скрипт скачан в /root/ ---"
echo "--- Теперь можно приступать к полной настройке стенда с помощью /root/pve-de-25-script-auto-deploy.sh ---"
```

---

### 6. Настройка HQ-CLI (Клиентская машина головного офиса)

**Сначала выполните эти команды на HQ-CLI:**
```bash
#!/bin/bash
set -e

echo "--- Начало настройки HQ-CLI (Bootstrap) ---"

echo ">>> Установка имени хоста на hq-cli.au-team.irpo..."
hostnamectl set-hostname hq-cli.au-team.irpo

echo ">>> Настройка сетевого интерфейса ens18 (IP: 192.168.2.10/28, GW: 192.168.2.1, DNS)..."
mkdir -p /etc/net/ifaces/ens18; find /etc/net/ifaces/ens18 -mindepth 1 -delete
cat <<'EOF' > /etc/net/ifaces/ens18/options
TYPE=eth
EOF
echo '192.168.2.10/28' > /etc/net/ifaces/ens18/ipv4address
echo 'default via 192.168.2.1' > /etc/net/ifaces/ens18/ipv4route
cat <<'EOF' > /etc/net/ifaces/ens18/resolv.conf
nameserver 8.8.8.8
nameserver 1.1.1.1
EOF

echo "--- Машина будет перезагружена. ---"
reboot
```

```bash
#!/bin/bash
set -e

echo "--- Завершение настройки HQ-CLI (Bootstrap) ---"

echo ">>> Обновление списка пакетов и установка необходимых пакетов для HQ-CLI (chrony, nfs-utils, openssh-server, curl, wget)... Это может занять время."
apt-get update && apt-get install -y \
    chrony \
    nfs-utils \
    openssh-server \
    curl \
    wget
echo ">>> Установка пакетов apt для HQ-CLI завершена."

echo ">>> Проверка связи со шлюзом (192.168.2.1)..."
ping -c 3 192.168.2.1 || echo "ПРЕДУПРЕЖДЕНИЕ: Шлюз 192.168.2.1 недоступен. HQ-RTR должен быть настроен."
echo ">>> Проверка связи с интернетом (8.8.8.8)..."
ping -c 3 8.8.8.8 || echo "ПРЕДУПРЕЖДЕНИЕ: Интернет недоступен."

echo ">>> Скачивание основного скрипта автоматизации (pve-de-25-script-auto-deploy.sh)..."
wget -O /root/pve-de-25-script-auto-deploy.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-deploy.sh
echo ">>> Предоставление прав на выполнение основному скрипту автоматизации..."
chmod +x /root/pve-de-25-script-auto-deploy.sh

echo ">>> Скачивание скрипта проверки (pve-de-25-script-auto-compliance-checker.sh)..."
wget -O /root/pve-de-25-script-auto-compliance-checker.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-compliance-checker.sh
echo ">>> Предоставление прав на выполнение скрипту проверки..."
chmod +x /root/pve-de-25-script-auto-compliance-checker.sh

echo "--- Bootstrap для HQ-CLI завершен. Пакеты установлены, основной скрипт скачан в /root/ ---"
echo "--- Теперь можно приступать к полной настройке стенда с помощью /root/pve-de-25-script-auto-deploy.sh ---"
```

--- 

### 7. Минимальный универсальный bootstrap-скрипт для Модуля 2 (Скачивание скриптов)

Этот раздел описывает минимальные bootstrap-скрипты, предназначенные для первоначальной подготовки виртуальных машин стенда Модуля 2. Стенд для Модуля 2 является полностью независимым от стенда Модуля 1 и он уже имеет базовую преднастроенную сетевую связность и доступ в Интернет для скачивания файлов.

**Выполните на соответствующей ВМ Модуля 2:**
```bash
#!/bin/bash
set -e

echo "--- Начало минимального bootstrap для Модуля 2 (скачивание/обновление скриптов) ---"

echo ">>> Обновление списка пакетов и установка wget (если необходимо)..."
apt-get update && apt-get install -y wget

echo ">>> Скачивание основного скрипта автоматизации (pve-de-25-script-auto-deploy.sh)..."
wget -O /root/pve-de-25-script-auto-deploy.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-deploy.sh
echo ">>> Предоставление прав на выполнение основному скрипту автоматизации..."
chmod +x /root/pve-de-25-script-auto-deploy.sh

echo ">>> Скачивание скрипта проверки (pve-de-25-script-auto-compliance-checker.sh)..."
wget -O /root/pve-de-25-script-auto-compliance-checker.sh https://raw.githubusercontent.com/imKeim/PVE-DE-25-Scripts/refs/heads/main/pve-de-25-script-auto-compliance-checker.sh
echo ">>> Предоставление прав на выполнение скрипту проверки..."
chmod +x /root/pve-de-25-script-auto-compliance-checker.sh

echo "--- Минимальный Bootstrap для Модуля 2 завершен. Скрипты скачаны/обновлены в /root/ ---"
```

---

## **После выполнения этих Bootstrap-скриптов на каждой машине:**

1.  На всех машинах должна быть настроена базовая сетевая связность.
2.  Все машины должны иметь доступ в Интернет (при условии, что ISP получает адрес по DHCP и его NAT работает).
3.  **Все необходимые пакеты** для выполнения заданий Модуля должны быть **установлены** на соответствующих машинах.
4.  На всех машинах в каталоге `/root/` должен находиться исполняемый файл `pve-de-25-script-auto-deploy.sh`.

**Теперь вы можете приступать к полной настройке стенда, используя скачанный скрипт `/root/pve-de-25-script-auto-deploy.sh`.**
Скрипт `pve-de-25-script-auto-deploy.sh` является интерактивным и содержит меню для выбора конфигурации Модуля 1 или Модуля 2 для текущей машины. **Поскольку все пакеты уже установлены, основной скрипт будет заниматься только конфигурацией служб.**

### **Разъяснение проблемы "курицы и яйца" с DNS и её решение:**

Ранее существовала потенциальная проблема: если бы основной скрипт `pve-de-25-script-auto-deploy.sh` пытался устанавливать пакеты *после* того, как DNS-клиент машины был перенастроен на внутренний, еще не полностью сконфигурированный DNS-сервер (например, `HQ-SRV`), то машина могла бы потерять доступ к репозиториям пакетов.

**Решение:** Перенос установки **всех** необходимых пакетов на этап выполнения bootstrap-скриптов (как сделано выше) решает эту проблему. На этом этапе все машины используют временные публичные DNS-серверы (например, `8.8.8.8`), что гарантирует доступ к репозиториям для скачивания пакетов. Когда вы будете запускать основной скрипт `pve-de-25-script-auto-deploy.sh`, он будет конфигурировать уже установленные службы, и момент переключения DNS-клиентов на внутренний сервер `HQ-SRV` не повлияет на возможность установки пакетов.

### **Порядок запуска основного скрипта `pve-de-25-script-auto-deploy.sh` для полной настройки Модуля 1:**

Следуйте этому порядку, последовательно заходя на каждую ВМ, запуская `/root/pve-de-25-script-auto-deploy.sh`, выбирая в его главном меню опцию **"Модуль 1: Настройка сетевой инфраструктуры"**, и затем в подменю Модуля 1 выбирая соответствующую машину.

1.  **ISP:**
    *   Зайдите на ВМ `ISP`. Запустите скрипт, выберите "Модуль 1" -> "ISP".
    *   *Действия скрипта:* Настроит IP, NAT, часовой пояс. Пакеты уже установлены.

2.  **HQ-SRV:**
    *   Зайдите на ВМ `HQ-SRV`. Запустите скрипт, выберите "Модуль 1" -> "HQ-SRV".
    *   *Действия скрипта:* Настроит IP, пользователя `sshuser`, SSH-сервер. Затем настроит `dnsmasq` как DNS-сервер. После этого DNS-клиент самого `HQ-SRV` будет переключен на `127.0.0.1`. Настроит часовой пояс. Пакеты уже установлены.

3.  **BR-SRV:**
    *   Зайдите на ВМ `BR-SRV`. Запустите скрипт, выберите "Модуль 1" -> "BR-SRV".
    *   *Действия скрипта:* Настроит IP, пользователя `sshuser`, SSH-сервер. Затем DNS-клиент `BR-SRV` будет переключен на использование `HQ-SRV` (192.168.1.10). Настроит часовой пояс. Пакеты уже установлены.

4.  **HQ-RTR:**
    *   Зайдите на ВМ `HQ-RTR`. Запустите скрипт, выберите "Модуль 1" -> "HQ-RTR".
    *   *Действия скрипта:* Настроит IP-адреса (WAN, LAN VLANs – **помните про фиксированные ID 100/200 для HQ-SRV/HQ-CLI и вариативный ID для сети Управления!**), пользователя `net_admin`, NAT/MSS Clamping. Затем DNS-клиент `HQ-RTR` будет переключен на использование `HQ-SRV` (192.168.1.10). После этого настроит GRE-туннель, OSPF, DHCP-сервер (`dnsmasq`). Настроит часовой пояс. Пакеты уже установлены.

5.  **BR-RTR:**
    *   Зайдите на ВМ `BR-RTR`. Запустите скрипт, выберите "Модуль 1" -> "BR-RTR".
    *   *Действия скрипта:* Настроит IP-адреса, пользователя `net_admin`, NAT/MSS Clamping. Затем DNS-клиент `BR-RTR` будет переключен на использование `HQ-SRV` (192.168.1.10). После этого настроит GRE-туннель, OSPF. Настроит часовой пояс. Пакеты уже установлены.

6.  **HQ-CLI (Этап 1 - до перезагрузки, если требуется по логике скрипта `auto-deploy`):**
    *   Зайдите на ВМ `HQ-CLI`. Запустите скрипт, выберите "Модуль 1" -> "HQ-CLI".
    *   *Действия скрипта:* Настроит FQDN. Если скрипт `auto-deploy` для `HQ-CLI` (Модуль 1) реализует двухэтапную настройку с перезагрузкой, он выполнит первый этап и инициирует перезагрузку. Пакеты уже установлены.

7.  **HQ-CLI (Этап 2 - после перезагрузки, если она была):**
    *   Если `HQ-CLI` перезагружался, снова войдите на ВМ `HQ-CLI` как `root`.
    *   Запустите: `/root/pve-de-25-script-auto-deploy.sh`
    *   В главном меню выберите "Модуль 1", затем в меню Модуля 1 выберите "HQ-CLI".
    *   *Действия скрипта:* Окончательная настройка DHCP-клиента (он получит IP и DNS от `HQ-RTR`). Настроит часовой пояс.

**После завершения всех шагов Модуля 1, переходите к настройке Модуля 2.**

---

### **Порядок запуска основного скрипта `pve-de-25-script-auto-deploy.sh` для полной настройки Модуля 2:**

При работе с преднастроенным стендом Модуля 2, следуйте этому порядку для настройки служб Модуля 2. На каждой ВМ (где это требуется) заходите через консоль, запускайте `/root/pve-de-25-script-auto-deploy.sh` и выбирайте в его главном меню опцию **"Модуль 2: Организация сетевого администрирования ОС"**. Затем в подменю Модуля 2 выберите соответствующую машину для конфигурации.

**Важно для Модуля 2:**
*   Предполагается, что используется **преднастроенный стенд Модуля 2**, где IP-адресация и базовые сетевые службы уже настроены (согласно Таблице 3 из `readme.md`). Имена интерфейсов на этом стенде могут отличаться. Скрипт `pve-de-25-script-auto-deploy.sh` должен это учитывать.
*   **Все необходимые пакеты уже установлены** на этапе bootstrap. Скрипт `pve-de-25-script-auto-deploy.sh` будет заниматься **только конфигурацией уже установленных служб.**
*   **Перезагрузки:** Машина `HQ-CLI` потребует перезагрузки после ввода в домен. Скрипт `pve-de-25-script-auto-deploy.sh` должен обработать это: после перезагрузки вы снова запускаете скрипт на `HQ-CLI` и выбираете ту же опцию, а скрипт продолжает настройку со второго этапа.
*   **Ручные действия:** Некоторые шаги, такие как веб-установка Moodle/MediaWiki или копирование SSH-ключей/файлов между машинами, потребуют вашего вмешательства. Скрипт предоставит инструкции.

### **Особенности последовательности настройки BR-SRV и HQ-CLI (Проблема "курицы и яйца" с `ssh-copy-id`):**

**Важно!** При настройке `BR-SRV` возникает ситуация с `ssh-copy-id` для копирования SSH-ключей на управляемые узлы, включая `HQ-CLI`. Однако SSH-сервер на `HQ-CLI` включается только при настройке самого `HQ-CLI`, что создаёт проблему типа "курица и яйцо":

1.   На `BR-SRV` нельзя выполнить команду `ssh-copy-id` для `HQ-CLI`, пока на `HQ-CLI` не настроен SSH-сервер.
2.   Настройка `HQ-CLI` происходит только после настройки `BR-SRV`.

**Решение:** Настройка `BR-SRV` и `HQ-CLI` выполняется в два этапа (два прогона скрипта) в следующем порядке:

1.   Запустите скрипт на `BR-SRV` (первый прогон) — скрипт выполнит первую часть настройки и завершится, показав инструкции для `ssh-copy-id`.
2.   Запустите скрипт на `HQ-CLI` (первый прогон) — скрипт настроит HQ-CLI, включая SSH-сервер, и перезагрузит машину.
3.   После перезагрузки `HQ-CLI` вернитесь к `BR-SRV` и запустите скрипт снова (второй прогон) — теперь можно выполнить `ssh-copy-id` на `HQ-CLI` и завершить настройку.
4.   Если требуется, вернитесь к `HQ-CLI` и запустите скрипт снова (второй прогон после перезагрузки) для завершения настройки.

**Рекомендуемый порядок настройки машин для Модуля 2:**

1.  **HQ-RTR:**
    *   Запустите скрипт, выберите "Модуль 2" -> "HQ-RTR".
    *   *Действия скрипта:* Настроит `chrony` (NTP-сервер), DNAT для SSH на `HQ-SRV`, Nginx (обратный прокси для Moodle и Wiki).

2.  **HQ-SRV:**
    *   Запустите скрипт, выберите "Модуль 2" -> "HQ-SRV".
    *   *Действия скрипта:* Настроит `chrony` (NTP-клиент), изменит порт SSH (например, на 2024), настроит условную DNS-пересылку на `BR-SRV`, настроит RAID и NFS, установит и настроит Moodle (включая инструкции для веб-установки и коррекцию `config.php`).

3.  **BR-RTR:**
    *   Запустите скрипт, выберите "Модуль 2" -> "BR-RTR".
    *   *Действия скрипта:* Настроит `chrony` (NTP-клиент), настроит DNAT для Wiki и SSH на `BR-SRV`.

4.  **BR-SRV (Этап 1 - до ручного `ssh-copy-id`):**
    *   Запустите скрипт, выберите "Модуль 2" -> "BR-SRV".
    *   *Действия скрипта:* Настроит `chrony` (NTP-клиент), изменит порт SSH (например, на 2024), настроит Samba AD, подготовит настройки для Ansible и сгенерирует SSH-ключи. Скрипт выведет инструкции для выполнения ssh-copy-id на все машины (включая HQ-CLI), но в данный момент HQ-CLI ещё не имеет запущенного SSH-сервера. **Скрипт создаст флаг и завершится.**

5.  **HQ-CLI (Этап 1 - до перезагрузки):**
    *   Запустите скрипт, выберите "Модуль 2" -> "HQ-CLI".
    *   *Действия скрипта:* Настроит `chrony` (NTP-клиент), проверит Яндекс.Браузер, **включит SSH-сервер**, **введет машину в домен** и **инициирует перезагрузку**.

6.  **BR-SRV (Этап 2 - Развертывание MediaWiki, ожидание настройки):**
    *   После настройки и перезагрузки `HQ-CLI` вернитесь на `BR-SRV`.
    *   Запустите скрипт, выберите "Модуль 2" -> "BR-SRV".
    *   *\*Действия скрипта:\** Обнаружит флаг и продолжит настройку. **Развернет MediaWiki и MariaDB в Docker.** По завершении отобразит URL для доступа (`http://<IP>:8080`), данные для подключения к БД и **инструкции для первоначальной веб-установки MediaWiki**. Скрипт **остановится и будет ожидать** ваших дальнейших действий.
    *   *\*Ваши действия:\** **Не нажимая Enter в консоли**, откройте указанный URL в браузере, проведите веб-установку MediaWiki. В конце установки **скачайте файл `LocalSettings.php`**. Теперь переходите к настройке `HQ-CLI`.

7.  **HQ-CLI (Этап 2 - Перенос конфигурации MediaWiki):**
    *   Запустите скрипт, выберите "Модуль 2" -> "HQ-CLI".
    *   *\*Действия скрипта:\** Выполнит стандартную настройку `HQ-CLI` (домашние каталоги, sudo, NFS и т.д.).
    *   *\*Ваши действия:\** Используя `HQ-CLI`, **скопируйте скачанный ранее файл `LocalSettings.php`** на `BR-SRV` в каталог `/home/sshuser/`. (Сначала перенесите файл на HQ-CLI, затем используйте `scp` для копирования на BR-SRV).

8.  **BR-SRV (Этап 3 - Завершение настройки MediaWiki):**
    *   **Вернитесь к консоли `BR-SRV`**, которая ожидает с шага 6.
    *   Убедитесь, что файл `LocalSettings.php` скопирован на `BR-SRV` (в `/home/sshuser/`).
    *   **Нажмите Enter** в консоли `BR-SRV`.
    *   *\*Действия скрипта:\** Продолжит выполнение, используя скопированный `LocalSettings.php`, и завершит настройку MediaWiki на сервере.

9.  **ISP:**
    *   Для Модуля 2 на ВМ `ISP` действия не требуются.

## Итоги

**По завершении всех шагов Модуля 1 и Модуля 2 ваш стенд должен быть полностью настроен согласно заданию.** Не забывайте сверяться с основным файлом `readme.md` для деталей каждого шага и требований к отчётности.