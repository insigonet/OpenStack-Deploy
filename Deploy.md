Конечно! Вот инструкция по установке и настройке OpenStack в формате Markdown (MD), которую вы можете скопировать.

```markdown
# Инструкция по установке и настройке OpenStack с использованием Kolla-Ansible на Hyper-V

## Введение

Эта инструкция описывает процесс создания тестовой среды OpenStack на платформе Hyper-V. OpenStack — это платформа для создания и управления публичными и частными облаками, которая позволяет разворачивать виртуальные машины и управлять сетями и хранилищами.

## Архитектура

**Цель:** Развернуть OpenStack в тестовой среде с использованием Hyper-V в качестве гипервизора, Kolla-Ansible для управления контейнерами, и pfSense для маршрутизации и безопасности.

## Сетевые компоненты

### Виртуальные машины и их роли

1. **Контроллер (w1-node-01)**
   - **ОС:** Ubuntu 22.04.3 LTS
   - **Роли:** Управляет основными сервисами OpenStack, включая API, планировщики, и сетевые функции.
   - **Ресурсы:** 64 GB RAM, 24 vCPU, 500 GB системный диск, 2x1000 GB для RAID.

2. **Узел вычислений (w1-node-02)**
   - **ОС:** Ubuntu 22.04.3 LTS
   - **Роли:** Запускает виртуальные машины и обрабатывает их нагрузку.
   - **Ресурсы:** 64 GB RAM, 24 vCPU, 500 GB системный диск.

3. **Хранилище (w1-node-03)**
   - **ОС:** Ubuntu 22.04.3 LTS
   - **Роли:** Управляет блочными хранилищами для виртуальных машин.
   - **Ресурсы:** 64 GB RAM, 24 vCPU, 2x1000 GB для хранения данных.

### Сетевые интерфейсы

- **eth0**: Внутренняя сеть
  - **IP:** 172.16.15.51 (w1-node-01), 172.16.15.52 (w1-node-02), 172.16.15.53 (w1-node-03)
  - **Назначение:** Взаимодействие между компонентами OpenStack, управление и внутренняя связь.

- **eth1**: Внешняя сеть
  - **IP:** Без IP-адреса, используется для внешнего трафика.
  - **Назначение:** Обеспечивает доступ виртуальных машин к интернету через pfSense.

### pfSense

pfSense используется для управления маршрутизацией и обеспечением безопасности в сети. Виртуальные машины подключаются к pfSense для:

- **pfSense-Local-1:** Управление внутренней сетью 172.16.15.0/24, включая доступ к интернету для узлов OpenStack.
- **pfSense-Local-2:** Управление внешней сетью 10.10.10.0/24, включая NAT и маршрутизацию внешнего трафика виртуальных машин.

### Сетевые топологии

1. **Внутренняя сеть (eth0):** Используется для управления OpenStack, взаимодействия между узлами и передачи данных. Все управляющие и сервисные потоки идут через этот интерфейс.

2. **Внешняя сеть (eth1):** Предназначена для сетевого доступа виртуальных машин и предоставления публичных сервисов. Внешняя связь обеспечивается через pfSense, который действует как шлюз.

## Подготовка к установке OpenStack

### Шаг 1: Создание виртуальной машины

1. **Конфигурация VM на Hyper-V:**
   - **ОС:** Ubuntu 22.04.3 LTS
   - **RAM:** 64 GB
   - **CPU:** 24 vCPU
   - **Диски:** 500 GB системный, 2x1000 GB для RAID

2. **Настройка Hyper-V:**

   ```powershell
   Set-VMProcessor -VMName w1-node-01 -ExposeVirtualizationExtensions $true
   Set-VMNetworkAdapter -VMName "w1-node-01" -MacAddressSpoofing On
   ```

### Шаг 2: Установка и настройка ОС

1. **Обновление системы и установка пакетов:**

   ```bash
   sudo apt update && sudo apt upgrade -y
   sudo apt install -y git python3-dev libffi-dev gcc libssl-dev linux-azure chrony
   ```

2. **Перезагрузка системы:**

   ```bash
   sudo reboot
   ```

3. **Настройка имени хоста и часового пояса:**

   ```bash
   sudo hostnamectl set-hostname w1-node-01
   sudo timedatectl set-timezone Europe/Kiev
   ```

4. **Настройка файла /etc/hosts:**

   ```bash
   sudo bash -c 'cat << EOF > /etc/hosts
   127.0.0.1    localhost
   172.16.15.51 w1-node-01
   172.16.15.52 w1-node-02
   172.16.15.53 w1-node-03
   172.16.15.100 os.unitec.com.ua
   EOF'
   ```

5. **Настройка сети с помощью Netplan:**

   ```bash
   sudo bash -c 'cat << EOF > /etc/netplan/00-installer-config.yaml
   network:
     version: 2
     ethernets:
       eth0:
         addresses: [172.16.15.51/24]
         nameservers: [8.8.8.8, 1.1.1.1]
         routes: [{ to: default, via: 172.16.15.1 }]
       eth1:
         dhcp4: no
         optional: false
   EOF'
   sudo netplan apply
   ```

### Шаг 3: Настройка RAID

1. **Создание RAID 1 во время установки Ubuntu:**
   - **Диски:** /dev/sdb и /dev/sdc
   - **RAID:** md0, смонтирован в `/mnt/md0`

2. **Проверка записи в `/etc/fstab`:**

   ```plaintext
   /dev/disk/by-id/md-uuid-af0bc13f:094d5d1a:e9efc394:27748a70-part1 /mnt/md0 ext4 defaults 0 1
   ```

## Установка Ansible и Kolla-Ansible

### Шаг 4: Установка и настройка Ansible и Kolla-Ansible

1. **Создание виртуального окружения и установка Ansible:**

   ```bash
   sudo apt install -y python3-venv
   python3 -m venv ~/venv
   source ~/venv/bin/activate
   pip install -U pip
   pip install 'ansible-core>=2.14,<2.16'
   ```

2. **Установка Kolla-Ansible:**

   ```bash
   pip install git+https://opendev.org/openstack/kolla-ansible@stable/2023.2
   ```

3. **Подготовка конфигурации Kolla-Ansible:**

   ```bash
   sudo mkdir -p /etc/kolla && sudo chown $USER:$USER /etc/kolla
   cp -r ~/venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
   cp ~/venv/share/kolla-ansible/ansible/inventory/all-in-one ./inventory
   ```

4. **Установка зависимостей Ansible Galaxy:**

   ```bash
   kolla-ansible install-deps
   ```

5. **Генерация паролей:**

   ```bash
   kolla-genpwd
   ```

## Конфигурация Kolla-Ansible

### Шаг 5: Настройка globals.yml

1. **Создание и настройка файла `/etc/kolla/globals.yml`:**

   ```bash
   sudo bash -c 'cat << EOF > /etc/kolla/globals.yml
   ---
   workaround_ansible_issue_8743: yes

   ## Kolla options
   kolla_base_distro: "ubuntu"
   openstack_release: "2023.2"

   ## Network configuration
   network_interface: "eth0"
   kolla_internal_vip_address: "172.16.15.100"
   kolla_external_vip_address: "os.unitec.com.ua"
   api_interface: "eth0"
   storage_interface: "eth0"
   neutron_external_interface: "eth1"

   ## OpenStack configuration
   enable_haproxy: "yes"
   enable_central_logging: "yes"
   enable_horizon: "yes"
   enable_nova_ssh: "yes"
   nova_compute_virt_type: "qemu"

   ## Cinder - Block Storage Options
   enable_cinder: "yes"
   enable_cinder_backend_lvm: "no"
   EOF'
   ```

## Развертывание OpenStack

### Шаг 6: Развертывание OpenStack с использованием Kolla-Ansible

1. **Активизация виртуального окружения**

   ```bash
   source

 ~/venv/bin/activate
   ```

2. **Тестирование подключения Ansible**

   ```bash
   ansible -i ./inventory all -m ping
   ```

3. **Подготовка серверов**

   ```bash
   kolla-ansible -i ./inventory bootstrap-servers
   ```

4. **Добавление пользователя в группу Docker**

   ```bash
   sudo usermod -aG docker $USER
   ```

5. **Предварительные проверки**

   ```bash
   kolla-ansible -i ./inventory prechecks
   ```

6. **Развертывание OpenStack**

   ```bash
   kolla-ansible -i ./inventory deploy
   ```

7. **Пост-развертывание действий**

   ```bash
   kolla-ansible -i ./inventory post-deploy
   ```

## Проверка развертывания

1. **Проверка состояния сервисов**

   ```bash
   docker ps
   ```

2. **Доступ к Horizon**

   - Откройте браузер и перейдите по адресу `http://os.unitec.com.ua` для доступа к веб-интерфейсу Horizon.

3. **Тестирование функциональности**

   - Создайте тестовые виртуальные машины и сети, чтобы убедиться, что OpenStack работает как ожидалось.

## Заключение

Теперь у вас есть развернутая тестовая среда OpenStack, готовая для использования. Следуйте указанным шагам для управления и настройки вашего облака.
```
