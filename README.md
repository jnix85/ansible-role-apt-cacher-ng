# ansible-role-apt-cacher-ng
[![CI](https://github.com/elnappo/ansible-role-apt-cacher-ng/workflows/CI/badge.svg)](https://github.com/elnappo/ansible-role-apt-cacher-ng/actions?query=workflow%3ACI) [![Ansible Galaxy](https://img.shields.io/badge/galaxy-elnappo.apt--cacher--ng-blue.svg?style=flat)](https://galaxy.ansible.com/elnappo/apt-cacher-ng/)

Simply installs and start apt-cacher-ng on boot. Get more informations about apt-cacher-ng at https://www.unix-ag.uni-kl.de/~bloch/acng/

## Requirements
- Ubuntu or Debian
- Ansible 2.10+ (firewall automation uses the `community.general` and `ansible.posix` collections)

## Role Variables
- `apt_cacher_ng_port` (default `3142`): Port for apt-cacher-ng to listen on.
- `apt_cacher_ng_cache_dir` (default `/var/cache/apt-cacher-ng`): Cache directory location.
- `apt_cacher_ng_config_dir` / `apt_cacher_ng_config_dir_mode`: Directory that holds configuration and its permissions.
- `apt_cacher_ng_config_file` / `apt_cacher_ng_config_owner` / `apt_cacher_ng_config_group` / `apt_cacher_ng_config_mode`: Path and ownership for the rendered configuration file.
- `apt_cacher_ng_config_settings`: Ordered list of key/value objects rendered into `acng.conf`.
- `apt_cacher_ng_extra_settings`: Optional additional key/value objects or raw strings appended to the configuration.
- `apt_cacher_ng_debconf_file`: Debconf configuration file that is removed by this role.
- `apt_cacher_ng_service_name`: Service name managed by the role.
- `apt_cacher_ng_firewall_backends`: List of firewall helpers (`ufw`, `iptables`, `nftables`, `firewalld`) to configure.
- `apt_cacher_ng_firewall_manage_packages`: Install firewall packages before configuring them.
- `apt_cacher_ng_firewall_sources`: Optional list of CIDR sources to restrict access.
- `apt_cacher_ng_firewall_interface`: Optional interface that the firewall rule should apply to.
- `apt_cacher_ng_firewalld_zone`: firewalld zone used for port authorisation.
- `apt_cacher_ng_firewall_comment`: Comment inserted into firewall rules.
- `apt_cacher_ng_nftables_family` / `apt_cacher_ng_nftables_table` / `apt_cacher_ng_nftables_chain`: Location where nftables rules are inserted.
- `apt_cacher_ng_setup_ufw`: Deprecated compatibility flag that maps to `apt_cacher_ng_firewall_backends`.

See `defaults/main.yml` for the authoritative list of variables and defaults.

## Firewall Support
Set `apt_cacher_ng_firewall_backends` to any combination of supported backends to automatically open the apt-cacher-ng port. The role does not touch the local firewall unless this list is set (the legacy `apt_cacher_ng_setup_ufw` flag is still honoured for backwards compatibility). When `apt_cacher_ng_firewall_manage_packages` is `true`, the corresponding firewall packages are installed before the rules are applied.

## Check Mode
The role is safe to run with `ansible-playbook --check`; tasks avoid failing in check mode so you can preview changes confidently.

## Dependencies
This role has no role dependencies. Firewall integrations rely on the `community.general` and `ansible.posix` collections declared in `meta/main.yml`.

## Example Playbook

```yaml
- hosts: servers
  become: true
  roles:
    - role: elnappo.apt_cacher_ng
      apt_cacher_ng_firewall_backends:
        - ufw
      apt_cacher_ng_extra_settings:
        - key: BindAddress
          value: 0.0.0.0
        - "VerboseLog: 1"
```

## Client configuration
### with ansible
Set apt_proxy as a host var

	[host:vars]
	apt_proxy=http://apt.example.com:3142/

**For the whole system:**

```yaml
- name: Set up apt proxy
  template: src=templates/apt_proxy.conf dest=/etc/apt/apt.conf.d/01proxy owner=root group=root mode=0644
    when: ansible_os_family == "Debian" and apt_proxy is defined
```

templates/apt_proxy.conf:

	# {{ ansible_managed }}
	Acquire::http { Proxy "{{ apt_proxy }}"; };
	Acquire::https { Proxy "https://"; };

**Only for one task:**

```yaml
- apt: name=ufw state=installed
  environment:
    http_proxy: "{{ apt_proxy }}"
```

### without ansible
Replace server IP/FQDN!

	$ echo 'Acquire::http { Proxy "http://apt.example.com:3142"; };' > /etc/apt/apt.conf.d/01proxy

## Import localhost cache

	$ echo 'Acquire::http { Proxy "http://localhost:3142"; };' > /etc/apt/apt.conf.d/01proxy
	$ apt-get update
	$ apt-get autoclean
	$ mkdir -p /var/cache/apt-cacher-ng/_import
	$ ln -s /var/cache/apt /var/cache/apt-cacher-ng/_import/apt
	$ wget "http://localhost:3142/acng-report.html?abortOnErrors=aOe&doImport=Start+Import&calcSize=cs&asNeeded=an#bottom"

After the import has finished, you can remove the symlink with:

	$ rm /var/cache/apt-cacher-ng/_import/apt

## License

MIT

## Author Information

elnappo <elnappo@nerdpol.io>
