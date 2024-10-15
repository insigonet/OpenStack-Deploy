# Справочник команд диагностики OpenStack Kolla-Ansible

Этот справочник содержит коллекцию команд для диагностики и управления сервисами OpenStack, развернутыми с помощью Kolla-Ansible. Команды разделены на категории для удобства использования.

---

## Содержание

1. [Общее управление Kolla-Ansible и Docker](#общее-управление-kolla-ansible-и-docker)
2. [Диагностика Neutron и Open vSwitch](#диагностика-neutron-и-open-vswitch)
3. [iptables и сетевые настройки](#iptables-и-сетевые-настройки)
4. [Управление виртуальными машинами и миграцией](#управление-виртуальными-машинами-и-миграцией)
5. [Управление конфигурацией Kolla-Ansible](#управление-конфигурацией-kolla-ansible)
6. [Проверка сервисов OpenStack](#проверка-сервисов-openstack)
7. [Команды Open vSwitch](#команды-open-vswitch)
8. [Команды Neutron](#команды-neutron)
9. [Проверка логов](#проверка-логов)
10. [Диагностика сети](#диагностика-сети)
11. [Информация о системе и мониторинг](#информация-о-системе-и-мониторинг)
12. [Команды Ansible](#команды-ansible)
13. [Прочие команды](#прочие-команды)
14. [Примечания](#примечания)

---

## Общее управление Kolla-Ansible и Docker

```bash
# Развернуть или обновить сервисы OpenStack с помощью Kolla-Ansible
kolla-ansible -i /etc/kolla/inventory deploy

# Применить изменения конфигурации к определенным сервисам (например, Neutron)
kolla-ansible -i /etc/kolla/inventory reconfigure -t neutron

# Проверить статус всех контейнеров Docker
docker ps

# Просмотреть логи контейнера Docker
docker logs <имя_контейнера>

# Перезапустить контейнер Docker
docker restart <имя_контейнера>

# Выполнить команду внутри контейнера Docker
docker exec -it <имя_контейнера> <команда>
```

---

## Диагностика Neutron и Open vSwitch

```bash
# Просмотреть конфигурацию агента Neutron Open vSwitch
docker exec -it neutron_openvswitch_agent cat /etc/neutron/plugins/ml2/openvswitch_agent.ini

# Проверить статус агентов Neutron
openstack network agent list

# Отобразить конфигурацию мостов и портов Open vSwitch
docker exec -it openvswitch_vswitchd ovs-vsctl show

# Список портов на мосту br-int
docker exec -it openvswitch_vswitchd ovs-vsctl list-ports br-int

# Проверить наличие моста br-ex
docker exec -it openvswitch_vswitchd ovs-vsctl br-exists br-ex && echo "br-ex существует" || echo "br-ex отсутствует"

# Проверить, что OVSDB сервер слушает на порту 6640
sudo netstat -tulnp | grep 6640
```

---

## iptables и сетевые настройки

```bash
# Отобразить правила iptables
sudo iptables -L

# Проверить сетевые интерфейсы и их конфигурации
ip a

# Просмотреть сетевые конфигурационные файлы (например, netplan)
sudo cat /etc/netplan/50-cloud-init.yaml

# Проверить таблицу маршрутизации
ip route

# Отобразить сетевые неймспейсы
ip netns list
```

---

## Управление виртуальными машинами и миграцией

```bash
# Показать подробную информацию о виртуальной машине
openstack server show <id_сервера>

# Выполнить живую миграцию виртуальной машины на другой узел
openstack server migrate --live --block-migration <id_сервера>

# Подтвердить или откатить миграцию
openstack server migrate confirm <id_сервера>
openstack server migrate revert <id_сервера>

# Проверить статус виртуальной машины
openstack server status <id_сервера>

# Список всех виртуальных машин
openstack server list
```

---

## Управление конфигурацией Kolla-Ansible

```bash
# Создать директорию для пользовательских конфигураций Kolla-Ansible
sudo mkdir -p /etc/kolla/config/<имя_сервиса>/

# Редактировать пользовательский конфигурационный файл для сервиса (например, Neutron)
sudo nano /etc/kolla/config/neutron/<имя_конфигурационного_файла>.ini

# Пример содержания файла /etc/kolla/config/neutron/openvswitch_agent.ini
[ovs]
bridge_mappings = physnet1:br-ex
```

---

## Проверка сервисов OpenStack

```bash
# Список агентов Neutron и их статусы
openstack network agent list

# Проверить статус гипервизоров
openstack hypervisor list

# Список доступных сетей
openstack network list

# Список доступных подсетей
openstack subnet list

# Список доступных маршрутизаторов
openstack router list

# Список доступных портов
openstack port list
```

---

## Команды Open vSwitch

```bash
# Добавить мост br-ex вручную (если необходимо)
docker exec -it openvswitch_vswitchd ovs-vsctl add-br br-ex

# Добавить порт к мосту br-ex
docker exec -it openvswitch_vswitchd ovs-vsctl add-port br-ex <имя_интерфейса>

# Показать потоки Open vSwitch
docker exec -it openvswitch_vswitchd ovs-ofctl dump-flows br-int

# Удалить мост
docker exec -it openvswitch_vswitchd ovs-vsctl del-br <имя_моста>
```

---

## Команды Neutron

```bash
# Перезапустить агенты Neutron
docker restart neutron_server neutron_openvswitch_agent neutron_l3_agent neutron_dhcp_agent neutron_metadata_agent

# Проверить логи сервера Neutron
docker logs neutron_server

# Проверить логи агента Neutron Open vSwitch
docker logs neutron_openvswitch_agent
```

---

## Проверка логов

```bash
# Просмотреть логи конкретного сервиса OpenStack
docker logs <имя_контейнера_сервиса>

# Следить за логами в реальном времени
docker logs -f <имя_контейнера_сервиса>

# Просмотреть системные логи (если необходимо)
sudo journalctl -u <имя_сервиса>

# Просмотреть логи внутри контейнера (например, логи Neutron)
docker exec -it <имя_контейнера> cat /var/log/kolla/<сервис>/<имя_лога>

# Пример: Просмотреть логи сервера Neutron
docker exec -it neutron_server cat /var/log/kolla/neutron/neutron-server.log
```

---

## Диагностика сети

```bash
# Проверить связь с другим узлом
ping <ip_адрес>

# Проверить, открыт ли порт на удаленном узле
nc -zv <ip_адрес> <номер_порта>

# Трассировка маршрута до удаленного узла
traceroute <ip_адрес>

# Проверить разрешение DNS
nslookup <доменное_имя>

# Отобразить сетевые соединения, таблицы маршрутизации, статистику интерфейсов
netstat -tulpn

# Проверить открытые порты и прослушивающие сервисы
sudo lsof -i -P -n | grep LISTEN

# Отобразить таблицу ARP
arp -a

# Проверить статистику сетевых интерфейсов
ifconfig -a

# Тестировать связь внутри сетевого неймспейса
ip netns exec <неймспейс> ping <ip_адрес>
```

---

## Информация о системе и мониторинг

```bash
# Отобразить информацию о системе
uname -a

# Показать информацию о дистрибутиве
lsb_release -a

# Проверить использование дискового пространства
df -h

# Проверить использование памяти
free -m

# Мониторинг процессов и использования ресурсов
top
htop

# Отобразить время работы системы и нагрузку
uptime

# Отобразить информацию о CPU
lscpu

# Отобразить устройства PCI (полезно для проверки оборудования)
lspci

# Отобразить блочные устройства
lsblk
```

---

## Команды Ansible

```bash
# Выполнить команду на всех узлах в инвентаре
ansible -i /etc/kolla/inventory all -m shell -a 'hostname'

# Выполнить команду на всех вычислительных узлах
ansible -i /etc/kolla/inventory compute -m shell -a 'hostname'

# Проверить доступность всех узлов
ansible -i /etc/kolla/inventory all -m ping

# Собрать факты о всех узлах
ansible -i /etc/kolla/inventory all -m setup

# Выполнить команду для перезапуска сервиса на вычислительных узлах
ansible -i /etc/kolla/inventory compute -m shell -a 'systemctl restart <имя_сервиса>'
```

---

## Прочие команды

```bash
# Проверить текущие альтернативы iptables
sudo update-alternatives --display iptables

# Переключиться на определенную версию iptables (legacy или nft)
sudo update-alternatives --set iptables /usr/sbin/iptables-legacy

# Проверить переменные окружения (например, PATH)
echo $PATH

# Проверить конфигурацию SSH (например, авторизованные ключи)
cat ~/.ssh/authorized_keys

# Отобразить сообщения кольцевого буфера ядра (полезно для аппаратных ошибок)
dmesg | less

# Проверить дерево процессов
pstree

# Список открытых файлов процессом
lsof -p <pid>

# Проверить статус синхронизации времени NTP
timedatectl status

# Показать текущую дату и время
date

# Отобразить системные лимиты
ulimit -a

# Проверить статус SELinux (для CentOS/RHEL)
sestatus
```
