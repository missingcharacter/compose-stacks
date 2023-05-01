# Vaultwarden

## Overview

Mostly based on:

- [Docker Traefik ModSecurity Setup](https://github.com/dani-garcia/vaultwarden/wiki/Docker---Traefik---ModSecurity-Setup)

## Requirements

- [`fail2ban`](https://github.com/fail2ban/fail2ban)

## `fail2ban` setup

The documented `fail2ban` configuration for `vaultwarden` is
[here](https://github.com/dani-garcia/vaultwarden/wiki/Fail2Ban-Setup#debian--ubuntu--raspberry-pi-os)

Below you'll find suggestions for when the containers log to `stdout` and
`stderr`, and `docker` is configured to log to `journald`

```shell
/etc/fail2ban/filter.d/waf.local
/etc/fail2ban/filter.d/vaultwarden-admin.local
/etc/fail2ban/filter.d/vaultwarden.local
/etc/fail2ban/jail.d/waf.local
/etc/fail2ban/jail.d/vaultwarden-admin.local
/etc/fail2ban/jail.d/vaultwarden.local
```

### `/etc/fail2ban/filter.d/waf.local`

```text
[INCLUDES]
before = common.conf

[Definition]
failregex = ^.*\[client <ADDR>\] ModSecurity: Access denied with code 403 .*$
ignoreregex =
```

### `/etc/fail2ban/jail.d/waf.local`

```text
[waf]
enabled = true
port = 80,443
action = iptables-allports[name=waf, chain=FORWARD]
backend = systemd
filter = waf[journalmatch='CONTAINER_NAME=waf']
maxretry = 1
bantime = 14400
findtime = 14400
```

### `/etc/fail2ban/filter.d/vaultwarden-admin.local`

```text
# path_f2b/filter.d/vaultwarden-admin.local
# Source: https://github.com/dani-garcia/vaultwarden/wiki/Fail2Ban-Setup#filter-1

[INCLUDES]
before = common.conf

[Definition]
failregex = ^.*Invalid admin token\. IP: <ADDR>.*$
ignoreregex =
```

### `/etc/fail2ban/jail.d/vaultwarden-admin.local`

```text
# path_f2b/jail.d/vaultwarden-admin.local
# Source: https://github.com/dani-garcia/vaultwarden/wiki/Fail2Ban-Setup#jail-1

[vaultwarden-admin]
enabled = true
port = 80,443
action = iptables-allports[name=vaultwarden-admin, chain=FORWARD]
backend = systemd
filter = vaultwarden[journalmatch='CONTAINER_NAME=vaultwarden']
maxretry = 3
bantime = 14400
findtime = 14400
```

### `/etc/fail2ban/filter.d/vaultwarden.local`

```text
# path_f2b/filter.d/vaultwarden.local
# Source: https://github.com/dani-garcia/vaultwarden/wiki/Fail2Ban-Setup#filter

[INCLUDES]
before = common.conf

[Definition]
failregex = ^.*Username or password is incorrect\. Try again\. IP: <ADDR>\. Username:.*$
ignoreregex =
```

### `/etc/fail2ban/jail.d/vaultwarden.local`

```text
# path_f2b/jail.d/vaultwarden.local
# Source: https://github.com/dani-garcia/vaultwarden/wiki/Fail2Ban-Setup#jail

[vaultwarden]
enabled = true
port = 80,443,8081
action = iptables-allports[name=vaultwarden, chain=FORWARD]
backend = systemd
filter = vaultwarden[journalmatch='CONTAINER_NAME=vaultwarden']
maxretry = 3
bantime = 14400
findtime = 14400
```
