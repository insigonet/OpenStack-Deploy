## Содержание:
1. [Подготовка сервера w1-i-node-01](#подготовка-сервера-w1-i-node-01)
    - [Настройка sudo без пароля](#настройка-sudo-без-пароля)
    - [Обновление системы и установка необходимых пакетов](#обновление-системы-и-установка-необходимых-пакетов)
    - [Установка и настройка Chrony](#установка-и-настрока-Chrony-для-синхронизации-времени)
    - [Настройка сети](#настройка-сети)
    - [Настройка файла hosts](#настройка-файла-hosts)
2. [Установка виртуального окружения и Kolla-Ansible](#установка-виртуального-окружения-и-kolla-ansible)
3. [Настройка Kolla-Ansible](#настройка-kolla-ansible)
4. [Генерация SSH ключей и настройка доступа](#генерация-ssh-ключей-и-настройка-доступа)
5. [Редактирование файла inventory](#редактирование-файла-inventory)
6. [Редактирование файла globals.yml](#редактирование-файла-globalsyml)
7. [Развертывание OpenStack](#развертывание-openstack)
8. [Установка OpenStack CLI](#установка-openstack-cli)
9. [Создание сети и настроек OpenStack](#создание-сети-и-настроек-openstack)
10. [Создание flavor в OpenStack](#создание-flavor-в-openstack)
11. [Настройка security group](#настройка-security-group)
12. [Назначение квот](#назначение-квот)
13. [Добавляем SSH-ключ](#добавляем-ssh-ключ-для-пользователя-admin)
14. [Финальная настройка сервера](#финальная-настройка-сервера)
15. [Панели управления OpenStack](#панели-управления-openstack)

---

### Подготовка сервера w1-i-node-01

#### Настройка sudo без пароля

```bash
# Настраиваем sudo без пароля
echo "%sudo ALL=(ALL:ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/group_sudo_nopasswd

# Устанавливаем права на файл
sudo chmod 0440 /etc/sudoers.d/group_sudo_nopasswd

# Проверяем конфигурацию sudoers
sudo visudo -c
```

---

#### Обновление системы и установка необходимых пакетов

```bash
# Обновляем систему
sudo apt update && sudo apt upgrade -y

# Проверяем, если требуется перезагрузка, перезагрузит
[ -f /var/run/reboot-required ] && sudo systemctl reboot
```

```bash
# Устанавливаем необходимые пакеты
sudo apt install -y git python3-dev libffi-dev gcc libssl-dev git
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
# Разрешаем синхронизацию времени для всех локальных сетей (частные IP-диапазоны)
allow 10.0.0.0/8
allow 172.16.0.0/12
allow 192.168.0.0/16

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

---

#### Настройка сети

```bash
# Задаем имя хоста
sudo hostnamectl set-hostname w1-i-node-01

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
            - 10.64.92.101/24
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

#### Настройка файла hosts

```bash
# Настраиваем файл hosts
sudo bash -c 'cat << EOF > /etc/hosts
127.0.0.1 localhost
10.64.92.101 w1-i-node-01
10.64.92.110 os2.fiberax.online
EOF'
```

---

### Установка виртуального окружения и Kolla-Ansible

```bash
# Устанавливаем виртуальное окружение и необходимые пакеты
sudo apt install -y python3-venv

# Создаем виртуальное окружение
python3 -m venv ~/venv
source ~/venv/bin/activate

# Обновляем pip и устанавливаем Ansible и Kolla-Ansible
pip install -U pip
pip install 'ansible-core>=2.14,<2.16'
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2023.2
```

---

### Настройка Kolla-Ansible

```bash
# Создаем каталоги для конфигурации Kolla-Ansible
sudo mkdir -p /etc/kolla && sudo chown $USER:$USER /etc/kolla

# Копируем примеры конфигурации в каталог Kolla
cp -r ~/venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla

# Копируем файл инвентаря multinode
cp ~/venv/share/kolla-ansible/ansible/inventory/multinode /etc/kolla/inventory

# Устанавливаем зависимости Ansible Galaxy
kolla-ansible install-deps

# Генерируем пароли
kolla-genpwd
```

---

### Генерация SSH ключей и настройка доступа

```bash
# Генерируем SSH ключ
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Настраиваем доступ по SSH на сервере
ssh-copy-id -i ~/.ssh/id_rsa.pub master@w1-i-node-01

# Проверяем доступ по SSH
ssh master@w1-i-node-01 'echo "SSH доступ настроен"'
```

---

### Редактирование файла inventory

```bash
# Открываем файл inventory для редактирования
nano /etc/kolla/inventory
```

Меняем следующие параметры:

```ini
[control]
w1-i-node-01

[network]
w1-i-node-01

[compute]
w1-i-node-01

[monitoring]
w1-i-node-01

[storage]
Не задаем
```

---

### Редактирование файла globals.yml

```bash
# Открываем globals.yml для редактирования
nano /etc/kolla/globals.yml
```

Добавляем или изменяем следующие параметры:

```yaml
---
workaround_ansible_issue_8743: yes
kolla_base_distro: "ubuntu"
openstack_release: "2023.2"
kolla_internal_vip_address: "10.64.92.110"
kolla_internal_fqdn: "{{ kolla_internal_vip_address }}"
kolla_external_vip_address: "{{ kolla_internal_vip_address }}"
kolla_external_fqdn: "{{ kolla_external_vip_address }}"
network_interface: "vlan2059"
kolla_external_vip_interface: "{{ network_interface }}"
api_interface: "{{ network_interface }}"
swift_storage_interface: "{{ network_interface }}"
swift_replication_interface: "{{ swift_storage_interface }}"
tunnel_interface: "{{ network_interface }}"
dns_interface: "{{ network_interface }}"
octavia_network_interface: "{{ api_interface }}"
neutron_external_interface: "vlan2924"
neutron_plugin_agent: "openvswitch"
openstack_region_name: "pl-warsaw-1"
openstack_logging_debug: "False"
nova_compute_virt_type: "kvm"
enable_nova_ssh: "yes"
enable_haproxy: "yes"
enable_horizon: "yes"
enable_skyline: "yes"
enable_central_logging: "yes"
enable_fluentd: "yes"
enable_cinder: "no"
enable_cinder_backend_lvm: "no"
nova_instance_datadir_volume: "/mnt/md0/nova/"

```

```bash
# Создаем директорию для Ephemeral Storage
sudo mkdir -p /mnt/md0/nova/
```
---

### Развертывание OpenStack

```bash
# Активируем виртуальное окружение
source ~/venv/bin/activate
```

```bash
# Проверяем доступность узлов из файла inventory
ansible -i /etc/kolla/inventory all -m ping
```

```bash
# Подготавливаем узел из файла inventory
kolla-ansible -i /etc/kolla/inventory bootstrap-servers
```

```bash
# Проверяем конфигурацию перед развертыванием
kolla-ansible -i /etc/kolla/inventory prechecks
```

```bash
# Развертываем OpenStack
kolla-ansible -i /etc/kolla/inventory deploy
```

```bash
# Выполняем пост-развертывание
kolla-ansible -i /etc/kolla/inventory post-deploy
```

---

### Установка OpenStack CLI

```bash
# Активируем виртуальное окружение
source ~/venv/bin/activate

# Устанавливаем OpenStack CLI
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2023.2
```

---

### Создание сети и настроек OpenStack

```bash
# Активируем виртуальное окружение и загружаем переменные среды
source ~/venv/bin/activate && source /etc/kolla/admin-openrc.sh

# Создаем внешнюю сеть
openstack network create --share --external --provider-network-type flat \
  --provider-physical-network physnet1 external_network

# Создаем подсеть для внешней сети
openstack subnet create --network external_network \
  --subnet-range 77.88.230.192/27 \
  --gateway 77.88.230.193 \
  --dns-nameserver 8.8.8.8 \
  --dhcp \
  --allocation-pool start=77.88.230.202,end=77.88.230.222 \
  external_subnet

# Устанавливаем MTU для внешней сети
openstack network set --mtu 1500 external_network
```

---

### Создание flavor в OpenStack

```bash
# Активируем виртуальное окружение и загружаем переменные среды
source ~/venv/bin/activate && source /etc/kolla/admin-openrc.sh

# Создаем flavor без дополнительныx свойств
openstack flavor create --id 101 --vcpus 4 --ram 8192 --disk 100 4vCPU_8RAM_80SSD

# Созданем flavor с дополнительными свойствами
openstack flavor create \
  --id 201 \
  --vcpus 2 \
  --ram 4096 \
  --disk 40 \
  x86_vm_start \
  --property ':architecture'='x86_architecture' \
  --property ':category'='general_purpose' \
  --property 'hw:mem_page_size'='any' \
  --property 'hw:numa_nodes'='1' \
  --property 'quota:disk_total_iops_sec'='20000' \
  --property 'quota:vif_inbound_average'='62500' \
  --property 'quota:vif_outbound_average'='62500'

openstack flavor create \
  --id 202 \
  --vcpus 4 \
  --ram 8192 \
  --disk 100 \
  x86_vm_standart \
  --property ':architecture'='x86_architecture' \
  --property ':category'='general_purpose' \
  --property 'hw:mem_page_size'='any' \
  --property 'hw:numa_nodes'='1' \
  --property 'quota:disk_total_iops_sec'='30000' \
  --property 'quota:vif_inbound_average'='125000' \
  --property 'quota:vif_outbound_average'='125000'
```

---

### Настройка security group

```bash
# Активируем виртуальное окружение и загружаем переменные среды
source ~/venv/bin/activate && source /etc/kolla/admin-openrc.sh

# Разрешаем весь входящий трафик для security group по умолчанию
openstack security group rule create --protocol any --ingress default

# Проверяем текущие правила security group
openstack security group show default
```

---

### Назначение квот

```bash
# Активируем виртуальное окружение и загружаем переменные среды
source ~/venv/bin/activate && source /etc/kolla/admin-openrc.sh

# Назначение квот для проекта admin
openstack quota set --cores 100 --ram 102400 --instances 100 \
    --networks 100 --ports 100 --routers 100 --subnets 100 \
    --floating-ips 100 --secgroups 100 --secgroup-rules 100 \
    --server-groups 100 --server-group-members 100 --rbac-policies 100 \
    --key-pairs 100 --injected-file-size 10240 --injected-path-size 255 \
    --injected-files 100 --properties 100 --subnetpools 100 \
    --fixed-ips 100 --force admin && \
    openstack quota show admin
```

### Описание ключевых параметров:

- `--cores 100`: Лимит на количество vCPU.
- `--ram 102400`: Лимит на объем оперативной памяти (в МБ).
- `--instances 100`: Лимит на количество инстансов.
- `--networks 100`: Лимит на количество сетей.
- `--ports 100`: Лимит на количество портов.
- `--routers 100`: Лимит на количество роутеров.
- `--subnets 100`: Лимит на количество подсетей.
- `--floating-ips 100`: Лимит на количество плавающих IP.
- `--secgroups 100`: Лимит на количество групп безопасности.
- `--secgroup-rules 100`: Лимит на количество правил безопасности.
- `--server-groups 100`: Лимит на количество групп серверов.
- `--server-group-members 100`: Лимит на количество членов групп серверов.
- `--rbac-policies 100`: Лимит на количество RBAC политик.
- `--key-pairs 100`: Лимит на количество ключевых пар.
- `--injected-file-size 10240`: Максимальный размер инжектируемого файла (в байтах).
- `--injected-path-size 255`: Максимальная длина пути инжектируемого файла.
- `--injected-files 100`: Лимит на количество инжектируемых файлов.
- `--properties 100`: Лимит на количество пользовательских свойств.
- `--subnetpools 100`: Лимит на количество пулов подсетей.
- `--fixed-ips 100`: Лимит на количество фиксированных IP.

---

### Добавляем SSH-ключ для пользователя admin
```bash
# Активируем виртуальное окружение и загружаем переменные среды
source ~/venv/bin/activate && source /etc/kolla/admin-openrc.sh

# Добавляем ssh ключ.
openstack keypair create --user admin --public-key ~/.ssh/id_rsa.pub master-key
```

### Финальная настройка сервера

```bash
# Создаем группу kolla-admins и даем необходимые права
sudo groupadd kolla-admins
sudo chown -R root:kolla-admins /etc/kolla
sudo chmod -R 770 /etc/kolla
sudo usermod -aG kolla-admins master
```

```bash
# Добавляем пользователя в группу Docker и обновляем группу
sudo usermod -aG docker $USER && newgrp docker
```

```bash
# Проверяем контейнеры на сервере
docker ps
```

---

### Панели управления OpenStack

- [Skyline - http://os2.fiberax.online:9999](http://os2.fiberax.online:9999/)
- [Horizon - http://os2.fiberax.online](http://os2.fiberax.online)
- [OpenSearch - http://os2.fiberax.online:5601](http://os2.fiberax.online:5601/)

```bash
# Получаем пароль Keystone для администратора admin:
grep keystone_admin_password /etc/kolla/passwords.yml

# Получаем пароль для OpenSearch для пользователя opensearch:
grep opensearch_dashboards_password /etc/kolla/passwords.yml
```


---

Cервер `w1-i-node-01` подготовлен для работы как основной сервер OpenStack и сервер деплоя для дальнейшего добавления узлов в кластер.
