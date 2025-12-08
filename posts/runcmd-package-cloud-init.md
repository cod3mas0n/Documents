---
title: runcmd and package order in Cloud-Init
published: true
description: runcmd and package oreder in cloud-init. conflict in a case ?!
tags: 'qemu,virtualization,cloudinit,devops'
id: 3090956
date: '2025-12-08T00:45:58Z'
---

How is it possible to run a command via `runcmd` before installing its related package via `packages` in Cloud-Init? Which one runs first ?

In a Cloud-Init configuration, it may look strange that a command referencing a service can run before the package providing that service is installed. For example:

```yaml
#cloud-config
︙
package_update: true
package_upgrade: true

runcmd:
  - systemctl enable --now qemu-guest-agent.service

packages:
  - qemu-guest-agent
︙
```

In this setup, commonly used when provisioning VMs with Terraform’s [libvirt provider][tf-libvirt-provider] on QEMU/KVM, `runcmd` attempts to enable and start `qemu-guest-agent.service` even though the package providing it (`qemu-guest-agent`) hasn’t been installed yet. The guest agent is often required early so tools like Ansible’s [dynamic inventory][ansible-dynamic-inv] can query host information.

So how can Cloud-Init run `systemctl enable --now qemu-guest-agent.service` before the package exists?

The sentence “The [runcmd][runcmd-doc] module runs very late in the boot process” can be confusing at first glance, because in Cloud-Init’s documented module stages, `runcmd` appears in the [Config stage][boot-stages] while package installation happens in the Final stage.

It looks like `runcmd` runs before packages are installed and it wont work.

However, The `runcmd` module does not execute commands directly:

> The runcmd module only writes the script to be run later. The module that actually runs the script is [scripts_user][scripts-user] in the Final boot stage.

So even though `runcmd` appears in the Config stage, the commands you place under `runcmd` are not executed immediately. They are scheduled to run later, after packages have been installed and other finalization tasks are complete.

By the time the `scripts_user` module runs (in the Final stage), the `qemu-guest-agent` is installed, `qemu-guest-agent.service` exists and `systemctl enable --now` will be succeed.

The runcmd scripts will be written in `/var/lib/cloud/instance/scripts/runcmd` in the instance.

```bash
cat /var/lib/cloud/instance/scripts/runcmd

#!/bin/sh
systemctl enable --now qemu-guest-agent
```

</br>

`cloud-config` Example:

```yaml
#cloud-config
users:
  - name: USERNAME
    groups: users, admin
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
    gecos: "DevOps Engineer"
    passwd: "SHA-512 hashed password" # mkpasswd --method=SHA-512
    shell: /bin/bash
    ssh_authorized_keys:
      - "ssh-ed25519 ..."
      - "ssh-rsa ..."

ssh_pwauth: false

package_update: true
package_upgrade: true

apt:
  # file /etc/apt/apt.conf.d/cloud-init-aptproxy will be written
  http_proxy: http://<proxy-server>:port
  https_proxy: http://<proxy-server>:port

write_files:
  - path: /etc/ssh/sshd_config.d/ssh-hardening.conf
    content: |
      Port 2200
      KbdInteractiveAuthentication no
      ChallengeResponseAuthentication no
      MaxAuthTries 2
      AllowTcpForwarding no
      X11Forwarding no
      AllowAgentForwarding no
      AuthorizedKeysFile .ssh/authorized_keys

runcmd:
  - systemctl enable --now qemu-guest-agent

packages:
  - qemu-guest-agent
  - htop
  - tree

```

[cloud-init]:https://cloudinit.readthedocs.io/en/latest/index.html
[tf-libvirt-provider]:https://registry.terraform.io/providers/dmacvicar/libvirt/latest/docs
[ansible-dynamic-inv]:https://github.com/cod3mas0n/qemu-kvm-terraform/blob/main/dynamic-inventory.py
[runcmd-doc]:https://cloudinit.readthedocs.io/en/latest/reference/modules.html#runcmd
[scripts-user]:https://cloudinit.readthedocs.io/en/latest/reference/modules.html#scripts-user
[boot-stages]:https://cloudinit.readthedocs.io/en/latest/explanation/boot.html
