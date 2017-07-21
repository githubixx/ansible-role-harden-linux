ansible-role-harden-linux
=========================

This Ansible playbook is used in [Kubernetes the not so hard way with Ansible (at Scaleway) - Part 2 - Harden the instances](https://www.tauceti.blog/post/kubernetes-the-not-so-hard-way-with-ansible-at-scaleway-part-2/).

Ansible role for hardening Linux (targeting Ubuntu 16.04 mainly) with the following features:

- Change root password
- Add a regular/deploy user used for administration (e.g. for Ansible or login via SSH)
- Adjust APT update intervals
- Setup ufw firewall and allow only SSH access by default (add more ports/networks if you like)
- Adjust security related sysctl settings (/proc filesystem)
- Change SSH default port (if requested)
- Disable password authentication
- Disable root login
- Disable PermitTunnel

Role Variables
--------------

The following variables don't have defaults. You need to specify them either in `group_vars` or `host_vars`. E.g. if this settings should be used only for one specific host create a file for that host called like the FQDN of that host (e.g `host_vars/your-server.example.tld`) and put the variables with the correct values there. If you want to apply this variables to a host group create a file `group_vars/your-group.yml` e.g. Replace `your-group` with the host group name which you created in the Ansible `hosts` file (do not confuse with /etc/hosts...). `common_deploy_user_public_keys` loads all the public SSH key files specifed in the list from your local hard disk. So at least you need to specify:

```
common_root_password: your_encrypted_password_here

common_deploy_user: deploy
common_deploy_user_password: your_encrypted_password_here
common_deploy_user_home: /home/deploy
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

Defaults for some system variables (sysctl.conf / proc filesystem). This settings are recommendations from Google which they use for their Google Compute Cloud OS images (see [Building Custom Operating Systems](https://cloud.google.com/compute/docs/images/building-custom-os) and [Configuring Imported Images](https://cloud.google.com/compute/docs/images/configuring-imported-images)). If you are happy with this settings you don't have to do anything:

```
sysctl_settings:
  "net.ipv4.tcp_syncookies": 1                    # Enable syn flood protection
  "net.ipv4.conf.all.accept_source_route": 0      # Ignore source-routed packets
  "net.ipv6.conf.all.accept_source_route": 0      # IPv6 - Ignore ICMP redirects
  "net.ipv4.conf.default.accept_source_route": 0  # Ignore source-routed packets
  "net.ipv4.conf.all.accept_redirects": 0         # Ignore ICMP redirects
  "net.ipv6.conf.all.accept_redirects": 0         # IPv6 - Ignore ICMP redirects
  "net.ipv4.conf.default.accept_redirects": 0     # Ignore ICMP redirects
  "net.ipv4.conf.all.secure_redirects": 1         # Ignore ICMP redirects from non-GW hosts
  "net.ipv4.conf.default.secure_redirects": 1     # Ignore ICMP redirects from non-GW hosts
  "net.ipv4.ip_forward": 0                        # Do not allow traffic between networks or act as a router
  "net.ipv6.conf.all.forwarding": 0               # IPv6 - Do not allow traffic between networks or act as a router
  "net.ipv4.conf.all.send_redirects": 0           # Don't allow traffic between networks or act as a router
  "net.ipv4.conf.default.send_redirects": 0       # Don't allow traffic between networks or act as a router
  "net.ipv4.conf.all.rp_filter": 1                # Reverse path filtering - IP spoofing protection
  "net.ipv4.conf.default.rp_filter": 1            # Reverse path filtering - IP spoofing protection
  "net.ipv4.icmp_echo_ignore_broadcasts": 1       # Ignore ICMP broadcasts to avoid participating in Smurf attacks
  "net.ipv4.icmp_ignore_bogus_error_responses": 1 # Ignore bad ICMP errors
  "net.ipv4.conf.all.log_martians": 1             # Log spoofed, source-routed, and redirect packets
  "net.ipv4.conf.default.log_martians": 1         # Log spoofed, source-routed, and redirect packets
  "net.ipv4.tcp_rfc1337": 1                       # Implement RFC 1337 fix
  "kernel.randomize_va_space": 2                  # Randomize addresses of mmap base, heap, stack and VDSO page
  "fs.protected_hardlinks": 1                     # Provide protection from ToCToU races
  "fs.protected_symlinks": 1                      # Provide protection from ToCToU races
  "kernel.kptr_restrict": 1                       # Make locating kernel addresses more difficult
  "kernel.perf_event_paranoid": 2                 # Set perf only available to root
  "kernel.modules_disabled": 1                    # Disable CAP_SYS_MODULE which allows for loading and unloading of kernel modules. This feature is deprecated in the linux kernel.
```

You can override settings in `sysctl_settings` dictionary or add additional settings by creating a variable `sysctl_settings_user`. E.g. for Kubernetes networking we need to enable IP forwarding so we define:

```
sysctl_settings_user:
  "net.ipv4.ip_forward": 1
  "net.ipv6.conf.all.forwarding": 1
```

`sysctl_settings` and `sysctl_settings_user` will be merged and settings in `sysctl_settings_user` will override settings in `sysctl_settings` if they have the same name/key.

Allow SSH by default (deny everything else). If you change the default SSH port below also add the new SSH port here otherwise you would lock out yourself:
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

If you cloned the role via `git` e.g. the role name will differ e.g `ansible-role-harden-linux`. You can rename it to `harden-linux` e.g. and include that one.

License
-------

GNU GENERAL PUBLIC LICENSE Version 3

Author Information
------------------

[www.tauceti.blog](https://www.tauceti.blog)
