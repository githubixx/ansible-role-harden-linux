ansible-role-harden-linux
=========================

This Ansible playbook is used in [Kubernetes the not so hard way with Ansible (at scaleway) - part 2 - harden the instances](https://www.tauceti.blog/post/kubernetes-the-not-so-hard-way-with-ansible-at-scaleway-part-2/).

Ansible role for hardening Linux (targeting Ubuntu 16.04 mainly) with the following features:

- Change root password
- Add a regular/deploy user used for administration (e.g. for Ansible or login via SSH)
- Adjust APT update intervals
- Setup ufw firewall and allow only SSH access by default (add more ports/networks if you like)
- Adjust sysctl settings (/proc filesystem)
- Change SSH default port (if requested)
- Disable password authentication
- Disable root login
- Disable PermitTunnel

I used this also for my Kubernetes cluster setup (see [Kubernetes the not so hard way with ansible (at scaleway) - part 2 - harden the instances](https://www.tauceti.blog/post/kubernetes-the-not-so-hard-way-with-ansible-at-scaleway-part-2/)).

Role Variables
--------------

The following variables don't have defaults. You need to specify them either in `group_vars` or `host_vars`. E.g. if this settings should be used only for one specific host create a file `host_vars/your-server.example.net.yml` and put the variables with the correct values there. If you want to apply this variables to a host group create a file `group_vars/your-group/root_settings.yml` e.g. Replace `your-group` with the host group name which you created in the Ansible `hosts` file (do not confuse with /etc/hosts...). `common_deploy_user_public_keys` loads all the public SSH key files specifed in the list from your local hard disk. So at least you need to specify:

```
common_root_password: your_encrypted_password_here
common_deploy_user: deploy
common_deploy_user_pw: your_encrypted_password_here 
common_deploy_user_public_keys:
  - /home/your_user/.ssh/id_rsa.pub
```

To create a encrypted password use `python -c 'import crypt; print crypt.crypt("This is my Password", "$1$SomeSalt$")'`. (You may need `python2` instead of `python` in case of Archlinux e.g.).

The following variables below have defaults. So only specify if you need another value for the variable. `common_required_packages` specifies the packages this playbook requires to work:
```
common_required_packages:
  - ufw
  - sshguard
  - unattended-upgrades
```

`common_ssh_port` specifies the port you want to use for SSH. If you change this setting also add the new port to `ufw_allow_ports` list. Otherwise you won't be able to login via SSH anymore:
```
common_ssh_port: 22
```

Defaults for some system variables (sysctl.conf / proc filesystem). This settings are recommendations from Google which they use for their Google Compute Cloud OS images (see [Build a Compute Engine Image from Scratch](https://cloud.google.com/compute/docs/tutorials/building-images)). If you are happy with this settings you don't have to do anything:

```
sysctl_settings:
  - { key: 'net.ipv4.tcp_syncookies',                    value: '1', comment: 'enables syn flood protection'}
  - { key: 'net.ipv4.conf.all.accept_source_route',      value: '0', comment: 'ignores source-routed packets'}
  - { key: 'net.ipv4.conf.default.accept_source_route',  value: '0', comment: 'ignores source-routed packets'}
  - { key: 'net.ipv4.conf.all.accept_redirects',         value: '0', comment: 'ignores ICMP redirects'}
  - { key: 'net.ipv4.conf.default.accept_redirects',     value: '0', comment: 'ignores ICMP redirects'}
  - { key: 'net.ipv4.conf.all.secure_redirects',         value: '1', comment: 'ignores ICMP redirects from non-GW hosts'}
  - { key: 'net.ipv4.conf.default.secure_redirects',     value: '1', comment: 'ignores ICMP redirects from non-GW hosts'}
  - { key: 'net.ipv4.ip_forward',                        value: '0', comment: 'do not allow traffic between networks or act as a router'}
  - { key: 'net.ipv4.conf.all.send_redirects',           value: '0', comment: 'do not allow traffic between networks or act as a router'}
  - { key: 'net.ipv4.conf.default.send_redirects',       value: '0', comment: 'do not allow traffic between networks or act as a router'}
  - { key: 'net.ipv4.conf.all.rp_filter',                value: '1', comment: 'reverse path filtering - IP spoofing protection'}
  - { key: 'net.ipv4.conf.default.rp_filter',            value: '1', comment: 'reverse path filtering - IP spoofing protection'}
  - { key: 'net.ipv4.icmp_echo_ignore_broadcasts',       value: '1', comment: 'ignores ICMP broadcasts to avoid participating in Smurf attacks'}
  - { key: 'net.ipv4.icmp_ignore_bogus_error_responses', value: '1', comment: 'ignores bad ICMP errors'}
  - { key: 'net.ipv4.conf.all.log_martians',             value: '1', comment: 'logs spoofed, source-routed, and redirect packets'}
  - { key: 'net.ipv4.conf.default.log_martians',         value: '1', comment: 'log spoofed, source-routed, and redirect packets'}
  - { key: 'net.ipv4.tcp_rfc1337',                       value: '1', comment: 'implements RFC 1337 fix'}
  - { key: 'kernel.randomize_va_space',                  value: '2', comment: 'randomizes addresses of mmap base, heap, stack and VDSO page'}
  - { key: 'fs.protected_hardlinks',                     value: '1', comment: 'provides protection from ToCToU races'}
  - { key: 'fs.protected_symlinks',                      value: '1', comment: 'provides protection from ToCToU races'}
  - { key: 'kernel.kptr_restrict',                       value: '1', comment: 'makes locating kernel addresses more difficult'}
  - { key: 'kernel.perf_event_paranoid',                 value: '2', comment: 'set perf only available to root'}
```

Allow SSH by default (deny everything else):
```
ufw_allow_ports:
  - 22
```

You can also allow hosts to communicate on specific networks (without port restrictions) e.g.

```
ufw_allow_networks:
  - "10.3.0.0/24"
```

For `ufw_allow_networks` there is no default setting. Nothing to do if you don't need this setting.

Example Playbook
----------------

If you installed the role via `ansible-galaxy install githubixx.harden-linux` then include the role into your playbook like in this example:

```
- hosts: webservers
  roles:
    - githubixx.harden-linux
```

If you cloned the role via `git` e.g. the role name will differ e.g `ansible-role-harden-linux`. You can rename it to `harden-linux` e.g. and include that one. It's just a matter of the name of the role directory.

License
-------

GNU GENERAL PUBLIC LICENSE Version 3

Author Information
------------------

[www.tauceti.blog](https://www.tauceti.blog)
