### **Настрока HostBill в разделе Form Fild**
> Во вкладке Advanced прописываем packages в поле Variable name
>
> Во вкладке Advanced включаем Show in cart
>
> При добавлении чекбокса задаеам его имя и в поле Value passed to App указываем имя переменной


```
#cloud-config
ssh_pwauth: true
user: root
password: {$password}
chpasswd: {literal}{expire: False}{/literal}

package_update: true
packages:
  - htop
  - curl
{foreach from=$forms.packages item=package}
{if $package.variable_id == "fio"}
  - fio
{/if}
{/foreach}
```

```
#cloud-config
ssh_pwauth: true
user: root
password: {$password}
chpasswd: {literal}{expire: False}{/literal}

package_update: true
packages:
  - htop
  - curl
{foreach from=$forms.packages item=package}
{if $package.variable_id == "fio"}
  - fio
{/if}
{if $package.variable_id == "fail2ban"}
  - fail2ban
{/if}
{/foreach}

runcmd:
{if $package.variable_id == "fail2ban"}
  - systemctl enable fail2ban
  - systemctl start fail2ban
  - |
    echo "[sshd]" > /etc/fail2ban/jail.local
    echo "enabled = true" >> /etc/fail2ban/jail.local
    echo "port = ssh" >> /etc/fail2ban/jail.local
    echo "logpath = /var/log/auth.log" >> /etc/fail2ban/jail.local
    echo "maxretry = 5" >> /etc/fail2ban/jail.local
    echo "bantime = 900" >> /etc/fail2ban/jail.local
    echo "findtime = 600" >> /etc/fail2ban/jail.local
  - systemctl restart fail2ban
{/if}

{if $package.variable_id == "docker_composer"}
  # Установка Docker
  - curl -fsSL https://get.docker.com -o get-docker.sh
  - sh get-docker.sh
  - systemctl enable docker
  - systemctl start docker
  
  # Установка Docker Compose
  - curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
  - chmod +x /usr/local/bin/docker-compose
  - ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
  
  # Установка Composer
  - curl -sS https://getcomposer.org/installer | php
  - mv composer.phar /usr/local/bin/composer
{/if}

```

