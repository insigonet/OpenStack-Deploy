## Содержание:
1. [Подготовка сервера w1-i-node-02](#подготовка-сервера-w1-i-node-02)
    - [Настройка sudo без пароля](#настройка-sudo-без-пароля)
    - [Обновление системы и установка необходимых пакетов](#обновление-системы-и-установка-необходимых-пакетов)
    - [Настройка сети](#настройка-сети)
    - [Настройка файла /etc/hosts на всех серверах](#настройка-файла-etchosts-на-всех-серверах)
    - [Установка и настройка Chrony](#Установка-и-настрока-Chrony-для-синхронизации-времени)
    - [Смена пути Ephemeral Storage](#смена-пути-ephemeral-storage)
2. [Настройка SSH ключей](#настройка-ssh-ключей)
3. [Обновление inventory и globals.yml на сервере деплоя](#обновление-inventory-и-globalsyml-на-сервере-деплоя)
4. [Добавление нового сервера в кластер](#добавление-нового-сервера-в-кластер)
5. [Финальная настройка сервера w1-i-node-02](#финальная-настройка-сервера-w1-i-node-02)
---

### Подготовка сервера w1-i-node-02

#### Настройка sudo без пароля

```bash
# Добавляем правило для sudo без пароля
echo "%sudo ALL=(ALL:ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/group_sudo_nopasswd

# Устанавливаем правильные права на файл
sudo chmod 0440 /etc/sudoers.d/group_sudo_nopasswd

# Проверяем конфигурацию файла sudoers
sudo visudo -c
```

---

#### Обновление системы и установка пакетов

```bash
# Обновляем систему
sudo apt update && sudo apt upgrade -y

# Проверяем, если требуется перезагрузка, перезагрузит
[ -f /var/run/reboot-required ] && sudo systemctl reboot
```

---

#### Настройка сети

```bash
# Задаем имя хоста
sudo hostnamectl set-hostname w1-i-node-02

# Создаем файл для отключения сетевой конфигурации cloud-init:
sudo bash -c 'cat << EOF > /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
network: {config: disabled}
EOF'

# Настраиваем сеть через netplan
sudo bash -c 'cat << EOF > /etc/netplan/01-cloud-init.yaml
network:
    ethernets:
        ens20f0: {}
        ens20f1: {}
    version: 2
    vlans:
        vlan2059:
            addresses:
            - 10.64.92.102/24
            id: 2059
            link: ens20f0
            nameservers:
                addresses:
                - 8.8.8.8
                - 1.1.1.1
            routes:
            -   to: default
                via: 10.64.92.254
        vlan2924:
            id: 2924
            link: ens20f1
EOF'

# Удаляем старый конфиг
sudo rm /etc/netplan/50-cloud-init.yaml

# Устанавливаем права на 01-cloud-init.yaml
sudo chmod 600 /etc/netplan/01-cloud-init.yaml

# Применяем настройки сети
# sudo netplan apply

# Перезагружаем
sudo reboot
```

---

### Настройка файла /etc/hosts на всех серверах

```bash
# Настраиваем файл hosts на всех узлах
sudo bash -c 'cat << EOF > /etc/hosts
127.0.0.1 localhost
10.64.92.101 w1-i-node-01
10.64.92.102 w1-i-node-01
10.64.92.110 os2.fiberax.online
EOF'
```

---

#### Установка и настрока Chrony для синхронизации времени
```bash
# Устанавливаем chrony
sudo apt install -y chrony

# Устанавливаем временную зону
sudo timedatectl set-timezone Europe/Kiev

# Настраиваем Chrony для синхронизации с другими узлами
sudo bash -c 'cat << EOF > /etc/chrony/chrony.conf

# Сервера для синхронизации с интернетом
pool ntp.ubuntu.com         iburst maxsources 4
pool 0.pool.ntp.org         iburst maxsources 2
pool 1.pool.ntp.org         iburst maxsources 2
pool 2.pool.ntp.org         iburst maxsources 2

# Используем файлы конфигурации из стандартных директорий
confdir /etc/chrony/conf.d
sourcedir /etc/chrony/sources.d
sourcedir /run/chrony-dhcp

# Файлы для хранения ключей и дрейфа
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
ntsdumpdir /var/lib/chrony

# Синхронизация времени с RTC
rtcsync

# Настройка для корректировки больших отклонений при старте
makestep 1 3

# Логирование
logdir /var/log/chrony

# Запрет на внесение неправильных данных в системные часы
maxupdateskew 100.0
EOF'

# Включаем и запускаем Chrony
sudo systemctl enable chrony && sudo systemctl restart chrony

# Проверяем статус службы
sudo systemctl status chrony

# Проверка состояния источников времени и текущего состояния синхронизации:
chronyc sources && chronyc tracking
```

### Смена пути Ephemeral Storage

```bash
# Создаем директорию для Ephemeral Storage
sudo mkdir -p /mnt/md0/nova/
```

---

### Настройка SSH ключей

#### Добавляем существующие ключи на новый сервер

**Сервер Deploy**
```bash
# Устанавливаем и копируем SSH ключи с w1-i-node-01 на новый узел
ssh-copy-id -i ~/.ssh/id_rsa.pub master@w1-i-node-02
scp -P 22 ~/.ssh/id_rsa* master@w1-i-node-02:~/.ssh/

# Проверяем соединие с w1-i-node-01 на новый узел
ssh master@w1-i-node-02 'echo "SSH доступ настроен"'
```
**Новый сервер**
```bash
# Проверяем обратное соединие с w1-i-node-02 на сервер деплоя
ssh master@w1-i-node-02 'echo "SSH доступ настроен"'
```

#### Обновляем ключи на всех узлах

```bash
# Добавляем SSH ключи всех узлов в known_hosts на всех серверах
ssh-keyscan w1-i-node-02 w1-i-node-02 >> ~/.ssh/known_hosts

# Удаляем дублирующиеся записи
sort -u ~/.ssh/known_hosts -o ~/.ssh/known_hosts
```

---

### Обновление inventory и globals.yml на сервере деплоя

```bash
# Обновляем файл inventory, добавляя новый узел в секции network и compute
nano /etc/kolla/inventory
```

Добавляем новый сервер `w1-i-node-02`:

```ini
[network]
w1-i-node-01
w1-i-node-02

[compute]
w1-i-node-01
w1-i-node-02
```

---

### Добавление нового сервера в кластер на сервере деплоя

```bash
# Активируем виртуальное окружение
source ~/venv/bin/activate
```

```bash
# Проверяем доступность всех узлов
ansible -i /etc/kolla/inventory all -m ping
```

```bash
# Подготавливаем новый узел w1-i-node-02
kolla-ansible -i /etc/kolla/inventory --limit w1-i-node-02 bootstrap-servers
```

```bash
# Проверяем конфигурацию перед развертыванием
kolla-ansible -i /etc/kolla/inventory --limit w1-i-node-02 prechecks
```

```bash
# Развертываем на всех узлах, включая новый сервер
kolla-ansible -i /etc/kolla/inventory deploy
```

```bash
# Выводим список сервисов Compute и агентов сети на сервере deploy
openstack compute service list
openstack network agent list
```

---

### Финальная настройка сервера w1-i-node-02

```bash
# Создаем группу kolla-admins и даем необходимые права
sudo groupadd kolla-admins
sudo chown -R root:kolla-admins /etc/kolla
sudo chmod -R 770 /etc/kolla
sudo usermod -aG kolla-admins master

# Добавляем пользователя в группу Docker и обновляем группу
sudo usermod -aG docker $USER && newgrp docker
```

---

Новый сервер `w1-i-node-02` подготовлен и добавлен в кластер OpenStack.

--- 

