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

