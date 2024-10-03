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
```

