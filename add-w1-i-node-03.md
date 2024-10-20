## Содержание:
1. [Подготовка сервера w1-i-node-03](#подготовка-сервера-w1-i-node-03)
    - [Настройка sudo без пароля](#настройка-sudo-без-пароля)
    - [Обновление системы и установка необходимых пакетов](#обновление-системы-и-установка-необходимых-пакетов)
    - [Настройка сети](#настройка-сети)
    - [Установка и настрока Chrony для синхронизации времени](#установка и настрока Chrony для синхронизации времени)
2. [Настройка SSH ключей](#настройка-ssh-ключей)
3. [Настройка файла /etc/hosts на всех серверах](#настройка-файла-etchosts-на-всех-серверах)
4. [Обновление inventory и globals.yml на сервере деплоя](#обновление-inventory-и-globalsyml-на-сервере-деплоя)
5. [Добавление нового сервера в кластер](#добавление-нового-сервера-в-кластер)
6. [Проверка состояния и финальная настройка](#проверка-состояния-и-финальная-настройка)

---

### Подготовка сервера w1-i-node-03

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

#### Обновление системы и установка необходимых пакетов

```bash
# Обновляем систему
sudo apt update && sudo apt upgrade -y

# Если требуется перезагрузка, выполняем её
[ -f /var/run/reboot-required ] && sudo systemctl reboot
```

---

#### Настройка сети

```bash
# Задаем имя хоста
sudo hostnamectl set-hostname w1-i-node-03

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
            - 10.64.92.103/24
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

# Устанавливаемс права на 01-cloud-init.yaml
sudo chmod 600 /etc/netplan/01-cloud-init.yaml

# Применяем настройки сети
sudo netplan apply
```

---

#### Установка и настрока Chrony для синхронизации времени

```bash
# Устанавливаем необходимые пакеты
sudo apt install -y git chrony

# Настраиваем временную зону
sudo timedatectl set-timezone Europe/Kiev

# Настраиваем Chrony для синхронизации с другими узлами
sudo bash -c 'cat << EOF > /etc/chrony/chrony.conf
server w1-i-node-02 iburst
server pool.ntp.org iburst
EOF'

# Включаем и запускаем Chrony
sudo systemctl enable chrony
sudo systemctl restart chrony

# Проверяем статус службы
sudo systemctl status chrony

# Проверка состояния источников времени и текущего состояния синхронизации:
chronyc sources && chronyc tracking
```

---

### Настройка SSH ключей

#### Добавляем существующие ключи на новый сервер

```bash
# Копируем SSH ключи с w1-i-node-02 на новый узел
ssh-copy-id -i ~/.ssh/id_rsa.pub master@w1-i-node-03

№ Проверяем соединие
ssh master@w1-i-node-03 'echo "SSH доступ настроен"'
```

#### Обновляем ключи на всех узлах

```bash
# Добавляем SSH ключи всех узлов в known_hosts на всех серверах
ssh-keyscan w1-i-node-02 w1-i-node-03 w1-i-node-03 >> ~/.ssh/known_hosts

# Удаляем дублирующиеся записи
sort -u ~/.ssh/known_hosts -o ~/.ssh/known_hosts
```

---

### Настройка файла /etc/hosts на всех серверах

```bash
# Настраиваем файл hosts на всех узлах
sudo bash -c 'cat << EOF > /etc/hosts
127.0.0.1 localhost
10.64.92.102 w1-i-node-02
10.64.92.103 w1-i-node-03
10.64.92.110 os2.fiberax.online
EOF'
```

---

### Обновление inventory и globals.yml на сервере деплоя

```bash
# Обновляем файл inventory, добавляя новый узел в секции network и compute
nano /etc/kolla/inventory
```

Добавляем `w1-i-node-03`:

```ini
[network]
w1-i-node-02
w1-i-node-03

[compute]
w1-i-node-02
w1-i-node-03
```

---

### Добавление нового сервера в кластер

```bash
# Активируем виртуальное окружение
source ~/venv/bin/activate

# Проверяем доступность всех узлов
ansible -i /etc/kolla/inventory all -m ping

# Подготавливаем новый узел w1-i-node-03
kolla-ansible -i /etc/kolla/inventory --limit w1-i-node-03 bootstrap-servers

# Проверяем конфигурацию перед развертыванием
kolla-ansible -i /etc/kolla/inventory --limit w1-i-node-03 prechecks

# Развертываем на всех узлах, включая новый сервер
kolla-ansible -i /etc/kolla/inventory deploy

# Выполняем постразвертывание
kolla-ansible -i /etc/kolla/inventory post-deploy
```

---

### Проверка состояния и финальная настройка сервера w1-i-node-03

```bash
# Добавляем пользователя в группу Docker и обновляем группу
sudo usermod -aG docker $USER && newgrp docker
```

```bash
# Проверяем контейнеры на новом узле
docker ps
```

---

Новый сервер `w1-i-node-03` подготовлен и добавлен в кластер OpenStack.

--- 

