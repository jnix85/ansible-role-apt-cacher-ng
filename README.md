# clawduino.apt_cacher_ng

Install and configure an [apt-cacher-ng](https://www.unix-ag.uni-kl.de/~bloch/acng/)
server on Debian-family hosts (Debian, Ubuntu, Armbian, Raspberry Pi OS), announced
over mDNS so clients running [auto-apt-proxy](https://packages.debian.org/auto-apt-proxy)
discover it automatically.

Server only — configuring *clients* to use the cache is out of scope (that's what
`auto-apt-proxy` on the client is for).

## How it configures acng

The distro's `/etc/apt-cacher-ng/acng.conf` is never touched. apt-cacher-ng merges
every `*.conf` in its config directory with last-value-wins semantics, so this role
templates a single drop-in, `/etc/apt-cacher-ng/zz-ansible.conf`, containing only
the settings it manages. Package upgrades never conflict with the role.

The cache lives in `/var/cache/acng` by default (deliberately not the package
default), so the role owns the directory outright. The package's empty
`/var/cache/apt-cacher-ng` is left in place — harmless.

## Role variables

| Variable | Default | Notes |
|---|---|---|
| `apt_cacher_ng_port` | `3142` | upstream default |
| `apt_cacher_ng_bind_address` | `""` | empty = bind all interfaces; set e.g. `192.168.1.10` to restrict |
| `apt_cacher_ng_cache_dir` | `/var/cache/acng` | created with `apt-cacher-ng:apt-cacher-ng` ownership |
| `apt_cacher_ng_ex_threshold` | `4` | days a package survives after vanishing from repo indexes |
| `apt_cacher_ng_extra_settings` | `{}` | free-form acng settings, rendered last (win over the above) |
| `apt_cacher_ng_avahi_announce` | `true` | install avahi-daemon and publish `_apt_proxy._tcp` for auto-apt-proxy |
| `apt_cacher_ng_manage_firewall` | `false` | opt-in ufw allow rule for the cache port |
| `apt_cacher_ng_allowed_networks` | `[]` | ufw rule scope; empty = allow from anywhere |

## Example playbook

```yaml
- hosts: apt_cache_servers
  become: true
  roles:
    - role: clawduino.apt_cacher_ng
      vars:
        apt_cacher_ng_ex_threshold: 14
        apt_cacher_ng_manage_firewall: true
        apt_cacher_ng_allowed_networks:
          - 10.1.0.0/24
```

## Requirements

- Ansible >= 2.15, `community.general` collection (only when
  `apt_cacher_ng_manage_firewall` is enabled):
  `ansible-galaxy collection install -r requirements.yml`

## Development & testing

Dev environment is managed with [uv](https://docs.astral.sh/uv/):

```sh
uv sync
uv run ansible-lint
uv run molecule test      # needs Docker running; Debian 12/13 + Ubuntu 24.04
```

Containers can't exercise ufw or real mDNS multicast, so a real-infra smoke test
lives in `tests/`:

```sh
cp tests/inventory.example tests/inventory   # edit to taste
uv run ansible-playbook -i tests/inventory tests/playbook.yml
```

Then on any client on the same LAN, `apt install auto-apt-proxy` and run
`auto-apt-proxy` — it should print `http://<cache-host>:3142`.

## License

MIT
