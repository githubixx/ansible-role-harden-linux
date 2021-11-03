ansible-role-harden-linux
=========================

This Ansible role was mainly created for my blog series [Kubernetes the not so hard way with Ansible - Harden the instances](https://www.tauceti.blog/posts/kubernetes-the-not-so-hard-way-with-ansible-harden-the-instances/). But it can be used also standalone of course to harden Linux. It has the following features:

- Change root password
- Add a regular/deploy user used for administration (e.g. for Ansible or login via SSH)
- Adjust APT update intervals
- Setup UFW firewall and allow only SSH access by default (add more rules/allowed networks if you like)
- Adjust security related sysctl settings
- Adjust sshd settings e.g disable sshd password authentication, disable sshd root login and disable sshd PermitTunnel
- Install sshguard and adjust whitelist
- Optional: Install/configure Network Time Synchronization (NTP) e.g. `openntpd`/`ntp`/`systemd-timesyncd`

Versions
--------

I tag every release and try to stay with [semantic versioning](http://semver.org). If you want to use the role I recommend to checkout the latest tag. The master branch is basically development while the tags mark stable releases. But in general I try to keep master in good shape too.

Changelog
---------

see [CHANGELOG.md](https://github.com/githubixx/ansible-role-harden-linux/blob/master/CHANGELOG.md)

Role Variables
--------------

The following variables don't have defaults. You need to specify them either in a file in `group_vars` or `host_vars` directory. E.g. if this settings should be used only for one specific host create a file for that host called like the FQDN of that host (e.g `host_vars/your-server.example.tld`) and put the variables with the correct values there. If you want to apply this variables to a host group create a file `group_vars/your-group.yml` e.g. Replace `your-group` with the host group name which you created in the Ansible `hosts` file (do not confuse with /etc/hosts...). `harden_linux_deploy_user_public_keys` loads all the public SSH key files specified in the list from your local hard disk. So at least you need to specify:

```yaml
harden_linux_root_password: your_encrypted_password_here

harden_linux_deploy_user: deploy
harden_linux_deploy_user_password: your_encrypted_password_here
harden_linux_deploy_user_home: /home/deploy
harden_linux_deploy_user_public_keys:
  - /home/your_user/.ssh/id_rsa.pub
```

With `harden_linux_root_password` and `harden_linux_deploy_user_password` we specify the password for the `root` user and the `deploy` user. Ansible won't encrypt the password for you. How to create an encrypted password is described in the [Ansible FAQs](http://docs.ansible.com/ansible/latest/reference_appendices/faq.html#how-do-i-generate-encrypted-passwords-for-the-user-module). But as Ansible is installed anyways the most easiest way is probably the following command:

```bash
ansible localhost -m debug -a "msg={{ 'mypassword' | password_hash('sha512', 'mysecretsalt') }}"
```

`harden_linux_deploy_user` specifies the user we want to use to login at the remote host. As already mentioned the `harden-linux` role will disable root user login via SSH for a good reason. So a different user is needed. This user will get "sudo" permission which is need for Ansible (and/or yourself of course) to do it's work.

`harden_linux_deploy_user_public_keys` specifies a list of public SSH key files you want to add to `$HOME/.ssh/authorized_keys` of the deploy user on the remote host. If you specify `/home/deploy/.ssh/id_rsa.pub` e.g. as a value here the content of that **local** file will be added to `$HOME/.ssh/authorized_keys` of the deploy user on the remote host.

The following variables below have defaults. So you only need to change them if you need another value for the variable. `harden_linux_optional_packages` (before version `v6.0.0` of this role this variable was called `harden_linux_required_packages`) specifies additional/optional packages to install on the remote host e.g. (by default this variable is not specified):

```yaml
harden_linux_optional_packages:
  - vim
```

The role changes some `sshd` settings by default:

```yaml
harden_linux_sshd_settings:
  "^PasswordAuthentication": "PasswordAuthentication no"  # Disable password authentication
  "^PermitRootLogin": "PermitRootLogin no"                # Disable SSH root login
  "^PermitTunnel": "PermitTunnel no"                      # Disable tun(4) device forwarding
  "^Port ": "Port 22"                                     # Set sshd port
```

Personally I always change the default SSH port as lot of brute force attacks taking place against this port (but of course a port scanner will still be able to figure this out quickly). So if you want to change the port setting you can do so e.g.:

```yaml
harden_linux_sshd_settings_user:
  "^Port ": "Port 22222"
```

(Please notice the whitespace after `^Port`!). The playbook will combine `harden_linux_sshd_settings` and `harden_linux_sshd_settings_user` while the settings in `harden_linux_sshd_settings_user` have preference which means it will override the `^Port ` setting/key in `harden_linux_sshd_settings`. As you may have noticed all the key's in `harden_linux_sshd_settings` and `harden_linux_sshd_settings_user` begin with `^`. That's because it is a regular expression (regex). One of playbook task's will search for a line in `/etc/ssh/sshd_config` e.g. `^Port ` (while the `^` means "a line starting with ...") and replaces the line (if found) with e.g `Port 22222`. This way makes the playbook very flexible for adjusting settings in `sshd_config` (you can basically replace every setting). You'll see this pattern for other tasks too. So everything mentioned here holds true in such cases.

Next some default settings for firewall/iptables. Firewall/iptables rules and settings are managed by [UFW](https://wiki.ubuntu.com/UncomplicatedFirewall):

```yaml
harden_linux_ufw_defaults:
  "^IPV6": 'IPV6=yes'
  "^DEFAULT_INPUT_POLICY": 'DEFAULT_INPUT_POLICY="DROP"'
  "^DEFAULT_OUTPUT_POLICY": 'DEFAULT_OUTPUT_POLICY="ACCEPT"'
  "^DEFAULT_FORWARD_POLICY": 'DEFAULT_FORWARD_POLICY="DROP"'
  "^DEFAULT_APPLICATION_POLICY": 'DEFAULT_APPLICATION_POLICY="SKIP"'
  "^MANAGE_BUILTINS": 'MANAGE_BUILTINS=no'
  "^IPT_SYSCTL": 'IPT_SYSCTL=/etc/ufw/sysctl.conf'
  "^IPT_MODULES": 'IPT_MODULES="nf_conntrack_ftp nf_nat_ftp nf_conntrack_netbios_ns"'
```

This settings are basically changing the values in `/etc/defaults/ufw`. To override one or more of the default settings you can so by specifying the same key (which is a regex) as above e.g. `^DEFAULT_FORWARD_POLICY` and simply assign it the new value:

```yaml
harden_linux_ufw_defaults_user:
  "^DEFAULT_FORWARD_POLICY": 'DEFAULT_FORWARD_POLICY="ACCEPT"'
```

As already mentioned above this playbook will also combine `harden_linux_ufw_defaults` and `harden_linux_ufw_defaults_user` while the settings in `harden_linux_ufw_defaults_user` have preference.

Next we can specify some firewall rules with `harden_linux_ufw_rules`. The default is to allow SSH traffic on port `22` which uses protocol `tcp`:

```yaml
harden_linux_ufw_rules:
  - rule: "allow"
    to_port: "22"
    protocol: "tcp"
```

The following parameters are available with defaults (if any):

```yaml
rule (no default)
interface (default '')
direction (default 'in')
from_ip (default 'any')
to_ip (default 'any')
from_port (default '')
to_port (default '')
protocol (default 'any')
log (default 'false')
delete (default 'false')
```

A [rule](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-rule) can have the values `allow`, `deny`, `limit` and `reject`. [interface](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-interface) specify interface for the rule. The [direction](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-direction) (`in` or `out`) used for the `interface` depends on the value of direction. [from_ip](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-from_ip) specifies the source IP address and [from_port](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-from_port) the source port. [to_ip](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-to_ip) specifies the destination IP address and [to_port](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-to_port) the destination port. [protocol](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-proto) is `any` by default. Possible values are `tcp`, `udp`, `ipv6`, `esp`, `ah`, `gre` and `igmp`. [log](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-log) can either be `false` (default) or `true` and specifies if new connections that matched this rule should be logged. [delete](https://docs.ansible.com/ansible/latest/collections/community/general/ufw_module.html#parameter-delete) specifies if a rule should be deleted. This is important if a previously added rule should be removed. Just removing a rule from `harden_linux_ufw_rules` isn't enough! You must use `delete` to delete that rule.

You can also allow hosts to communicate on specific networks (without port restrictions) e.g.:

```yaml
harden_linux_ufw_allow_networks:
  - "10.3.0.0/24"
  - "10.200.0.0/16"
```

Next harden-linux role also changes some system variables (sysctl.conf / proc filesystem). This settings are recommendations from Google which they use for their Google Compute Cloud OS images (see [Google Cloud - Requirements to build custom images](https://cloud.google.com/compute/docs/images/building-custom-os) and [Configure your imported image for Compute Engine](https://cloud.google.com/compute/docs/images/configuring-imported-images)). These are the default settings (if you are happy with this settings you don't have to do anything but I recommend to verify if they work for your setup):

```yaml
harden_linux_sysctl_settings:
  "net.ipv4.tcp_syncookies": 1                    # Enable syn flood protection
  "net.ipv4.conf.all.accept_source_route": 0      # Ignore source-routed packets
  "net.ipv6.conf.all.accept_source_route": 0      # IPv6 - Ignore ICMP redirects
  "net.ipv4.conf.default.accept_source_route": 0  # Ignore source-routed packets
  "net.ipv6.conf.default.accept_source_route": 0  # IPv6 - Ignore source-routed packets
  "net.ipv4.conf.all.accept_redirects": 0         # Ignore ICMP redirects
  "net.ipv6.conf.all.accept_redirects": 0         # IPv6 - Ignore ICMP redirects
  "net.ipv4.conf.default.accept_redirects": 0     # Ignore ICMP redirects
  "net.ipv6.conf.default.accept_redirects": 0     # IPv6 - Ignore ICMP redirects
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
  "net.ipv4.icmp_echo_ignore_all": 0              # Ignore bad ICMP errors
  "net.ipv4.conf.all.log_martians": 1             # Log spoofed, source-routed, and redirect packets
  "net.ipv4.conf.default.log_martians": 1         # Log spoofed, source-routed, and redirect packets
  "net.ipv4.tcp_rfc1337": 1                       # Implement RFC 1337 fix
  "kernel.randomize_va_space": 2                  # Randomize addresses of mmap base, heap, stack and VDSO page
  "fs.protected_hardlinks": 1                     # Provide protection from ToCToU races
  "fs.protected_symlinks": 1                      # Provide protection from ToCToU races
  "kernel.kptr_restrict": 1                       # Make locating kernel addresses more difficult
  "kernel.perf_event_paranoid": 2                 # Set perf only available to root
```

You can override every single setting e.g. by creating a variable called `harden_linux_sysctl_settings_user`:

```yaml
harden_linux_sysctl_settings_user:
  "net.ipv4.ip_forward": 1
  "net.ipv6.conf.default.forwarding": 1
  "net.ipv6.conf.all.forwarding": 1
```

One of the playbook tasks will combine `harden_linux_sysctl_settings` and `harden_linux_sysctl_settings_user` while again `harden_linux_sysctl_settings_user` settings have preference. Have a look at `defaults/main.yml` file of the role for more information about the settings.

If you want UFW logging enabled set:

```yaml
harden_linux_ufw_logging: 'on'
```

Possible values are `on`,`off`,`low`,`medium`,`high` and `full`.

Next we've the "sshguard" settings. "sshguard" protects from brute force attacks against SSH. To avoid locking out yourself for a while you can add IPs or IP ranges to a whitelist. By default it's basically only "localhost":

```yaml
harden_linux_sshguard_whitelist:
  - "127.0.0.0/8"
  - "::1/128"
```

Also NTP packages can be installed/configured. This is optional. By default I'd recommend to use `systemd-timesyncd`. You can also use `ntp` package. But `openntpd` and `systemd-timesyncd` have the advantage that they don't listen on any ports by default. If you just want to keep the hosts clock in sync this is absolutely sufficient. Having the same time on all your hosts is critical for some services. E.g. for certificate validation, for etcd, databases, cryptography, and so on.

Valid options for `harden_linux_ntp` are:

- openntpd
- ntp
- systemd-timesyncd

`openntpd` and `systemd-timesyncd` have the advantage that they don't listen on any ports by default as already mentioned. If you just want to keep the hosts clock in sync one of those two should do the job. `systemd-timesyncd` is already installed if a distribution uses `systemd` (which is basically true for most Linux OSes nowadays). So no additional packages are needed in this case. To enable `openntpd` set `harden_linux_ntp` accordingly e.g.:

```yaml
harden_linux_ntp: "openntpd"
```

Settings for `openntpd`, `ntpd` or `systemd-timesyncd` (see next paragraph). For further options see man page: `man 5 ntpd.conf` for `ntp` and `openntpd` and `man 5 timesyncd.conf` for `systemd-timesyncd`.

The "key" here is a regular expression of a setting you want to replace and the value is the setting name + the setting value. E.g. we want to replace the line `servers 0.debian.pool.ntp.org` with `servers 1.debian.pool.ntp.org`. The regex (the key) would be `^servers 0` which means:

"search for a line in the configuration file that begins with `server 0` and replace the whole line with `servers 1.debian.pool.ntp.org`". This way every setting in the configuration file can be replaced by something else. Some examples:

```yaml
harden_linux_ntp_settings:
  "^servers 0": "servers 0.debian.pool.ntp.org"
  "^servers 1": "servers 1.debian.pool.ntp.org"
  "^servers 2": "servers 2.debian.pool.ntp.org"
  "^servers 3": "servers 3.debian.pool.ntp.org"
```

Please note: `systemd-timesyncd` comes with reasonable compile time defaults. So there is normally no need to change the configuration (not even the NTP servers). So the following are just examples but you don't really need to specify `harden_linux_ntp_settings` for `systemd-timesyncd`.

For `systemd-timesyncd` the configuration file is a little bit different. In this case a systemd drop-in for `/etc/systemd/timesyncd.conf` will be created at `/etc/systemd/timesyncd.conf.d/`.

Example for Debian:

```yaml
harden_linux_ntp_settings:
  "^#NTP=": "NTP=0.debian.pool.ntp.org 1.debian.pool.ntp.org 2.debian.pool.ntp.org 3.debian.pool.ntp.org"
```

For Ubuntu:

```yaml
harden_linux_ntp_settings:
  "^#NTP=": "NTP=ntp.ubuntu.com"
```

For Archlinux:

```yaml
harden_linux_ntp_settings:
  "^#NTP=": "NTP=0.arch.pool.ntp.org 1.arch.pool.ntp.org 2.arch.pool.ntp.org 3.arch.pool.ntp.org"
```

With `harden_linux_files_to_delete` a list of files can be specified that should be absent on the target host e.g.:

```yaml
harden_linux_files_to_delete:
  - "/root/.pw"
```

Also the package manager caching behavior can be influenced. E.g. for Ubuntu:

```yaml
# Set to "false" if package cache should not be updated
harden_linux_ubuntu_update_cache: true

# Set package cache valid time
harden_linux_ubuntu_cache_valid_time: 3600
```

For Archlinux:

```yaml
# Set to "false" if package cache should not be updated
harden_linux_archlinux_update_cache: true
```

Example Playbook
----------------

If you installed the role via `ansible-galaxy install githubixx.harden-linux` then include the role into your playbook like in this example:

```yaml
- hosts: webservers
  roles:
    - githubixx.harden-linux
```

License
-------

GNU GENERAL PUBLIC LICENSE Version 3

Author Information
------------------

[www.tauceti.blog](https://www.tauceti.blog)
