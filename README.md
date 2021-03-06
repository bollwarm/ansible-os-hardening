# os-hardening (Ansible Role)

[![Build Status](http://img.shields.io/travis/dev-sec/ansible-os-hardening.svg)][1]
[![Gitter Chat](https://badges.gitter.im/Join%20Chat.svg)][2]
[![Ansible Galaxy](https://img.shields.io/badge/galaxy-os--hardening-660198.svg)][3]

## Description

This roles provides numerous security-related configurations, providing all-round base protection.

It configures:

 * Configures package management e.g. allows only signed packages
 * Remove packages with known issues
 * Configures `pam` and `pam_limits` module
 * Shadow password suite configuration
 * Configures system path permissions
 * Disable core dumps via soft limits
 * Restrict Root Logins to System Console
 * Set SUIDs
 * Configures kernel parameters via sysctl

It will not:

 * Update system packages
 * Install security patches

## Requirements

* Ansible

## Variables

| Name           | Default Value | Description                        |
| -------------- | ------------- | -----------------------------------|
| `os_desktop_enable`| false |  true if this is a desktop system, ie Xorg, KDE/GNOME/Unity/etc|
| `os_env_extra_user_paths`| [] | add additional paths to the user's `PATH` variable (default is empty).|
| `os_env_umask`| 027| set default permissions for new files to `750` |
| `os_auth_pw_max_age`| 60 | maximum password age|
| `os_auth_pw_min_age`| 7 | minimum password age (before allowing any other password change)|
| `os_auth_retries`| 5 | the maximum number of authentication attempts, before the account is locked for some time|
| `os_auth_lockout_time`| 600 | time in seconds that needs to pass, if the account was locked due to too many failed authentication attempts|
| `os_auth_timeout`| 60 | authentication timeout in seconds, so login will exit if this time passes|
| `os_auth_allow_homeless`| false | true if to allow users without home to login|
| `os_auth_pam_passwdqc_enable`| true | true if you want to use strong password checking in PAM using passwdqc|
| `os_auth_pam_passwdqc_options`| "min=disabled,disabled,16,12,8" | set to any option line (as a string) that you want to pass to passwdqc|
| `os_security_users_allow`| [] | list of things, that a user is allowed to do. May contain `change_user`.
| `os_security_kernel_enable_module_loading`| true | true if you want to allowed to change kernel modules once the system is running (eg `modprobe`, `rmmod`)|
| `os_security_kernel_enable_sysrq`| false | sysrq is a 'magical' key combo you can hit which the kernel will respond to regardless of whatever else it is doing, unless it is completely locked up. |
| `os_security_kernel_enable_core_dump`| false | kernel is crashing or otherwise misbehaving and a kernel core dump is created |
| `os_security_suid_sgid_enforce`| true | true if you want to reduce SUID/SGID bits. There is already a list of items which are searched for configured, but you can also add your own|
| `os_security_suid_sgid_blacklist`| [] | a list of paths which should have their SUID/SGID bits removed|
| `os_security_suid_sgid_whitelist`| [] | a list of paths which should not have their SUID/SGID bits altered|
| `os_security_suid_sgid_remove_from_unknown`| false | true if you want to remove SUID/SGID bits from any file, that is not explicitly configured in a `blacklist`. This will make every Ansible-run search through the mounted filesystems looking for SUID/SGID bits that are not configured in the default and user blacklist. If it finds an SUID/SGID bit, it will be removed, unless this file is in your `whitelist`.|
| `os_security_packages_clean'`| true | removes packages with known issues. See section packages.|
| `ufw_manage_defaults` | true | true means apply all settings with `ufw_` prefix|
| `ufw_ipt_sysctl` | '' | by default it disables IPT_SYSCTL in /etc/default/ufw. If you want to overwrite /etc/sysctl.conf values using ufw - set it to your sysctl dictionary, for example `/etc/ufw/sysctl.conf`
| `ufw_default_input_policy` | DROP | set default input policy of ufw to `DROP` |
| `ufw_default_output_policy` | ACCEPT | set default output policy of ufw to `ACCEPT` |
| `ufw_default_forward_policy` | DROP| set default forward policy of ufw to `DROP` |

## Packages

We remove the following packages:

 * xinetd ([NSA](http://www.nsa.gov/ia/_files/os/redhat/rhel5-guide-i731.pdf), Chapter 3.2.1)
 * inetd ([NSA](http://www.nsa.gov/ia/_files/os/redhat/rhel5-guide-i731.pdf), Chapter 3.2.1)
 * tftp-server ([NSA](http://www.nsa.gov/ia/_files/os/redhat/rhel5-guide-i731.pdf), Chapter 3.2.5)
 * ypserv ([NSA](http://www.nsa.gov/ia/_files/os/redhat/rhel5-guide-i731.pdf), Chapter 3.2.4)
 * telnet-server ([NSA](http://www.nsa.gov/ia/_files/os/redhat/rhel5-guide-i731.pdf), Chapter 3.2.2)
 * rsh-server ([NSA](http://www.nsa.gov/ia/_files/os/redhat/rhel5-guide-i731.pdf), Chapter 3.2.3)

## Example Playbook

    - hosts: localhost
      roles:
        - dev-sec.os-hardening


## Changing sysctl variables

If you want to overwrite sysctl-variables, you have to overwrite the *whole* dict, or else only the single overwritten will be actually used.
So for example if you want to change the IPv4 traffic forwarding variable to `1`, you must pass the whole dict like this:

```
    - hosts: localhost
      roles:
        - dev-sec.os-hardening
      vars:
        sysctl_config:
          # Disable IPv4 traffic forwarding.
          net.ipv4.ip_forward: 1

          # Disable IPv6 traffic forwarding.
          net.ipv6.conf.all.forwarding: 0

          # ignore RAs on Ipv6.
          net.ipv6.conf.all.accept_ra: 0
          net.ipv6.conf.default.accept_ra: 0

          # Enable RFC-recommended source validation feature.
          net.ipv4.conf.all.rp_filter: 1
          net.ipv4.conf.default.rp_filter: 1

          # Reduce the surface on SMURF attacks.
          # Make sure to ignore ECHO broadcasts, which are only required in broad network analysis.
          net.ipv4.icmp_echo_ignore_broadcasts: 1

          # There is no reason to accept bogus error responses from ICMP, so ignore them instead.
          net.ipv4.icmp_ignore_bogus_error_responses: 1

          # Limit the amount of traffic the system uses for ICMP.
          net.ipv4.icmp_ratelimit: 100

          # Adjust the ICMP ratelimit to include ping, dst unreachable,
          # source quench, ime exceed, param problem, timestamp reply, information reply
          net.ipv4.icmp_ratemask: 88089

          # Disable IPv6
          net.ipv6.conf.all.disable_ipv6: 1

          # Protect against wrapping sequence numbers at gigabit speeds
          net.ipv4.tcp_timestamps: 0

          # Define restriction level for announcing the local source IP
          net.ipv4.conf.all.arp_ignore: 1

          # Define mode for sending replies in response to
          # received ARP requests that resolve local target IP addresses
          net.ipv4.conf.all.arp_announce: 2

          # RFC 1337 fix F1
          net.ipv4.tcp_rfc1337: 1
```

Alternatively you can change Ansible's [hash-behaviour](https://docs.ansible.com/ansible/intro_configuration.html#hash-behaviour) to `merge`, then you only have to overwrite the single hash you need to. But please be aware that changing the hash-behaviour changes it for all your playbooks and is not recommended by Ansible.

## Local Testing

For local testing you can use vagrant and Virtualbox of VMWare to run tests locally. You will have to install Virtualbox and Vagrant on your system. See [Vagrant Downloads](http://downloads.vagrantup.com/) for a vagrant package suitable for your system. For all our tests we use `test-kitchen`. If you are not familiar with `test-kitchen` please have a look at [their guide](http://kitchen.ci/docs/getting-started).

Next install test-kitchen:

```bash
# Install dependencies
gem install bundler
bundle install

# Fetch tests
bundle exec thor kitchen:fetch-remote-tests

# fast test on one machine
bundle exec kitchen test default-ubuntu-1204

# test on all machines
bundle exec kitchen test

# for development
bundle exec kitchen create default-ubuntu-1204
bundle exec kitchen converge default-ubuntu-1204
```

For more information see [test-kitchen](http://kitchen.ci/docs/getting-started)


## Contributors + Kudos

...

This role is mostly based on guides by:

* [Arch Linux wiki, Sysctl hardening](https://wiki.archlinux.org/index.php/Sysctl)
* [NSA: Guide to the Secure Configuration of Red Hat Enterprise Linux 5](http://www.nsa.gov/ia/_files/os/redhat/rhel5-guide-i731.pdf)
* [Ubuntu Security/Features](https://wiki.ubuntu.com/Security/Features)
* [Deutsche Telekom, Group IT Security, Security Requirements (German)](http://www.telekom.com/static/-/155996/7/technische-sicherheitsanforderungen-si)

Thanks to all of you!
## Contributing

See [contributor guideline](CONTRIBUTING.md).

## License and Author

* Author:: Sebastian Gumprich <sebastian.gumprich@38.de>

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.


[1]: http://travis-ci.org/dev-sec/ansible-os-hardening
[2]: https://gitter.im/dev-sec/general
[3]: https://galaxy.ansible.com/dev-sec/os-hardening
