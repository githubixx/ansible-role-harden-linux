Changelog
---------

**v6.1.0**

- **FEATURE**: Support Ubuntu 22.04 (contribution by @lvnilesh)
- add Molecule test for Ubuntu 22.04

**v6.0.0**

This version contains a few **breaking** changes. Please read the changelog carefully:

- **BREAKING**: `harden_linux_required_packages` variable was removed. Please use `harden_linux_optional_packages` instead now. Installation of packages `ufw`, `sshguard`, `sudo` and `unattended-upgrades` (last one Ubuntu only) are now installed in the tasks that setup the packages accordingly. So if you used `harden_linux_required_packages` to install additional packages you basically just need to rename that variable to `harden_linux_optional_packages` but you can remove `ufw`, `sshguard`, `sudo` and `unattended-upgrades` from the list and just keep all the packages you added manually.
- **BREAKING**: The deploy user specified in `harden_linux_deploy_user` will no longer be added to `/etc/sudoers` directly. Instead a file `/etc/sudoers.d/{{ harden_linux_deploy_user }}` will be created. After that file is created the old user entry will be removed from `/etc/sudoers`.
- **BREAKING**: Min. supported Ansible version changed from `2.5` to >= `2.9`
- **FEATURE**: Support Archlinux. Archlinux support doesn't contain "unattended upgrades" as it the case for Ubuntu.
- **FEATURE**: add `harden_linux_files_to_delete` variable. This variable allows to specify a list of files which should be absent on the target host.
- **FEATURE**: add `delete` parameter to UFW `rule`'s. Specifies if a rule should be deleted. This is important if a previously added rule should be removed. Just removing a rule from `harden_linux_ufw_rules` isn't enough! You must use `delete` to delete that rule.
- **FEATURE:** (Ubuntu only): add `harden_linux_ubuntu_update_cache` variable. Set to `false` if package cache should not be updated. Previously package cache was always updated.
- **FEATURE:** (Ubuntu only): add `harden_linux_ubuntu_cache_valid_time` variable. Set package cache valid time (in seconds). Previously it was always `3600` seconds.
- **FEATURE:** (Archlinux only): introduce `harden_linux_archlinux_update_cache` variable. Set to `false` if package cache should not be updated.

**v5.1.0**

- add `systemd-timesyncd` as additional option for `harden_linux_ntp`

**v5.0.0**

- Remove Ubuntu 16.04 support

**v4.1.0**

- Added basic Molecule tests
- updated README about how to generate encrypted passwords

**v4.0.3**

- Updated for Ubuntu 20.04 LTS

**v4.0.1**

- make `harden_linux_ntp` optional (commented in `defaults/main.yml`).

**v4.0.0**

- introduce `harden_linux_ntp` and `harden_linux_ntp_settings` variables. `openntpd` is installed by default now. See README for more information. If `harden_linux_ntp` variable isn't set no ntp service will be installed.

**v3.1.0**

- fix deprecation warning in "install required packages" task
- moved changelog entries to separate file
- make Ansible linter happy

**v3.0.1**

- update README

**v3.0.0**

- Ansible v2.5 needed for Ubuntu 18.04 Bionic Beaver as Python 3 is default there. It *should* work with Ansible >= 2.2 too but who knows ;-) As Ubuntu 18.04 comes with Python 3 support only by default you may adjust your Ansible's `hosts` file. E.g you can add the `ansible_python_interpreter` env. like so: `host.domain.tld ansible_python_interpreter=/usr/bin/python3` (also see http://docs.ansible.com/ansible/latest/reference_appendices/python_3_support.html for more examples)

**v2.1.0**

- support for Ubuntu 18.04 Bionic Beaver
- added `sudo` package to `harden_linux_required_packages`

**v2.0.1**

- fixed deprecation warning while installing aptitude

**v2.0.0**

- major refactoring
- removed `common_ssh_port` (see `harden_linux_sshd_settings` instead)
- all variables that started with `common_` are now starting with the prefix `harden_linux_`. Additionally ALL variables that the role uses are now prefixed with `harden_linux_`. Using a variable name prefix avoids potential collisions with other role/group variables.
- introduced `harden_linux_deploy_user_uid` and `harden_linux_deploy_user_shell`
- single settings in `harden_linux_sysctl_settings` can be overridden by specifing the key/value in `harden_linux_sysctl_settings_user` list (whole list needed to be replaced before this change)
- more documentation added to `defaults/main.yml` (please read it ;-) )
- every setting in hosts `/etc/ssh/sshd_config` config file can now be replaced by using `harden_linux_sshd_settings_user` list. The defaults are specified in `harden_linux_sysctl_settings` and will be merged with `harden_linux_sysctl_settings_user` during run time.
- added variable `harden_linux_sshguard_whitelist` for Sshguard whitelist
- firewall rules can now be added using `harden_linux_ufw_rules` variable

**v1.0.0**

- initial release
