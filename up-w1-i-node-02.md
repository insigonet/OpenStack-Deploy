## Содержание:
1. [Подготовка сервера w1-i-node-02](#подготовка-сервера-w1-i-node-02)
    - [Настройка sudo без пароля](#настройка-sudo-без-пароля)
    - [Обновление системы и установка необходимых пакетов](#обновление-системы-и-установка-необходимых-пакетов)
    - [Настройка временной зоны и установка Chrony](#настройка-временной-зоны-и-установка-chrony)
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
12. [Финальная настройка сервера](#финальная-настройка-сервера)

---

### Подготовка сервера w1-i-node-02

#### Настройка sudo без пароля

```bash
# Настраиваем sudo без пароля
echo "%sudo ALL=(ALL:ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/group_sudo_nopasswd

# Устанавливаем правильные права на файл
sudo chmod 0440 /etc/sudoers.d/group_sudo_nopasswd

# Проверяем конфигурацию sudoers
sudo visudo -c
```

---

#### Обновление системы и установка необходимых пакетов

```bash
# Обновляем систему
sudo apt update && sudo apt upgrade -y

# Если требуется перезагрузка, выполняем её
[ -f /var/run/reboot-required ] && sudo systemctl reboot

# Устанавливаем необходимые пакеты
sudo apt install -y git python3-dev libffi-dev gcc libssl-dev
```

---

#### Настройка временной зоны и установка Chrony

```bash
# Настраиваем временную зону
sudo timedatectl set-timezone Europe/Kiev

# Устанавливаем Chrony для синхронизации времени
sudo apt install -y chrony
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

# Устанавливаемс права на 01-cloud-init.yaml
sudo chmod 600 /etc/netplan/01-cloud-init.yaml

# Применяем настройки сети
sudo netplan apply
```

---

#### Настройка файла hosts

```bash
# Настраиваем файл hosts
sudo bash -c 'cat << EOF > /etc/hosts
127.0.0.1 localhost
10.64.92.102 w1-i-node-02
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
```

```bash
# Устанавливаем зависимости Ansible Galaxy
kolla-ansible install-deps
```

```bash
# Генерируем пароли
kolla-genpwd
```

---

### Генерация SSH ключей и настройка доступа

```bash
# Генерируем SSH ключ
ssh-keygen -t rsa

# Настраиваем доступ по SSH на сервере
ssh-copy-id -i ~/.ssh/id_rsa.pub master@w1-i-node-02
ssh master@w1-i-node-02 'echo "SSH доступ настроен"'
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
w1-i-node-02

[network]
w1-i-node-02

[compute]
w1-i-node-02

[monitoring]
w1-i-node-02

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
workaround_ansible_issue_8743: yes
kolla_base_distro: "ubuntu"
openstack_release: "2023.2"
kolla_internal_vip_address: "10.64.92.110"
kolla_internal_fqdn: "{{ kolla_internal_vip_address }}"
kolla_external_vip_address: "{{ kolla_internal_vip_address }}"
kolla_external_fqdn: "{{ kolla_external_vip_address }}"
network_interface: "ens20f0.2059"
kolla_external_vip_interface: "{{ network_interface }}"
api_interface: "{{ network_interface }}"
swift_storage_interface: "{{ network_interface }}"
swift_replication_interface: "{{ swift_storage_interface }}"
tunnel_interface: "{{ network_interface }}"
dns_interface: "{{ network_interface }}"
octavia_network_interface: "{{ api_interface }}"
neutron_external_interface: "ens20f1.2924"
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
```

---

### Развертывание OpenStack

```bash
# Активируем виртуальное окружение
source ~/venv/bin/activate

# Проверяем доступность узлов
ansible -i /etc/kolla/inventory all -m ping

# Подготавливаем узел
kolla-ansible -i /etc/kolla/inventory bootstrap-servers

# Проверяем конфигурацию перед развертыванием
kolla-ansible -i /etc/kolla/inventory prechecks

# Развертываем OpenStack
kolla-ansible -i /etc/kolla/inventory deploy

# Выполняем постразвертывание
kolla-ansible -i /etc/kolla/inventory post-deploy
```

---

### Установка OpenStack CLI

```bash
# Устанавливаем OpenStack CLI
pip install python-openstackclient -c https://releases.openstack.org/constraints/upper/2023.2
```

---

### Создание сети и настроек OpenStack

```bash
# Активируем виртуальное окружение и загружаем переменные среды
source ~/venv/bin/activate && source /etc/kolla/admin-openrc.sh
```

```bash
# Создаем внешнюю сеть
openstack network create --share --external --provider-network-type flat \
  --provider-physical-network physnet1 external_network
```

```bash
# Создаем подсеть для внешней сети
openstack subnet create --network external_network \
  --subnet-range 77.88.230.192/27 \
  --gateway 77.88.230.193 \
  --dns-nameserver 8.8.8.8 \
  --dhcp \
  --allocation-pool start=77.88.230.202,end=77.88.230.222 \
  external_subnet
```

```bash
# Устанавливаем MTU для внешней сети
openstack network set --mtu 1500 external_network
```

---

### Создание flavor в OpenStack

```bash
# Создаем базовые flavor для виртуальных машин
openstack flavor create --id 101 --vcpus 2 --ram 4096 --disk 40 2vCPU_4RAM_40SSD
openstack flavor create --id 102 --vcpus 4 --ram 8192 --disk 80 4vCPU_8RAM_80SSD

# Пример создания flavor с дополнительными свойствами
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
  --property 'quota:disk_total_iops_sec'='5000' \
  --property 'quota:vif_inbound_average'='12500' \
  --property 'quota:vif_outbound_average'='12500'

openstack flavor create \
  --id 202 \
  --vcpus 4 \
  --ram 8192 \
  --disk 80 \
  x86_vm_standart \
  --property ':architecture'='x86_architecture' \
  --property ':category'='general_purpose' \
  --property 'hw:mem_page_size'='any' \
  --property 'hw:numa_nodes'='1' \
  --property 'quota:disk_total_iops_sec'='10000' \
  --property 'quota:vif_inbound_average'='125000' \
  --property 'quota:vif_outbound_average'='125000'
```

---

### Настройка security group

```bash
# Разрешаем весь входящий трафик для security group по умолчанию
openstack security group rule create --protocol any --ingress default

# Проверяем текущие правила security group
openstack security group show default
```

---

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

Cервер `w1-i-node-02` подготовлен для работы как основной сервер OpenStack и сервер деплоя для дальнейшего добавления узлов в кластер.
