# Vaultwarden

## Overview

Mostly based on:

- [Docker Traefik ModSecurity Setup](https://github.com/dani-garcia/vaultwarden/wiki/Docker---Traefik---ModSecurity-Setup)

## Requirements

- [`fail2ban`](https://github.com/fail2ban/fail2ban)

## How to generate `ADMIN_TOKEN`

```shell
$ openssl rand -base64 48 # this will generate the plain text token
<REDACTED>
```

### How to secure the `ADMIN_TOKEN`

The command below will generate the string to enter as environment variables
`ADMIN_TOKEN`

```shell
$ echo -n '<REDACTED>' | argon2 \
  "$(openssl rand -base64 32)" -e -id -k 19456 -t 2 -p 1 | sed 's#\$#\$\$#g'
```

More details [here](https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page#secure-the-admin_token)

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

## Upgrade PostgreSQL

### Example 16.4 to 17

**Note:** Assumes the machine has enough space for the backup

- Comment all services except PostgreSQL

  ```shell
  sudo -e /etc/docker/compose/vaultwarden/docker-compose.yml
  ```

- Stop all docker compose services and start only PostgreSQL container

  ```shell
  sudo systemctl restart docker-compose@vaultwarden
  ```

- Backup existing database

  ```shell
  sudo docker exec vaultwarden-db pg_dumpall -U vaultwarden > ./dump16.4.sql
  ```

- Stop all docker compose services

  ```shell
  sudo systemctl stop docker-compose@vaultwarden
  ```

- Rename existing PostgreSQL data folder

  ```shell
  sudo mv /opt/docker/vaultwarden/db/data /opt/docker/vaultwarden/db/data16.4
  ```

- Create new PostgreSQL data folder

  ```shell
  sudo mkdir /opt/docker/vaultwarden/db/data
  ```

- Mount `/path/to/dump16.4.sql` as `/tmp/dump.sql` in `docker-compose.yml` and
  upgrade PostgreSQL tag to version `17`

  ```shell
  sudo -e /etc/docker/compose/vaultwarden/docker-compose.yml
  ```

  ```yaml
      volumes:
        - /opt/docker/vaultwarden/db/data:/var/lib/postgresql/data
        - /path/to/dump16.4.sql:/tmp/dump.sql:ro
  ```

- Start only PostgreSQL container

  ```shell
  sudo systemctl restart docker-compose@vaultwarden
  ```

- Import backup

  ```shell
  sudo docker exec -it vaultwarden-db bash
  psql -U vaultwarden -d vaultwarden < /tmp/dump.sql
  ```

- Remove comments from services and comment mount `/path/to/dump16.4.sql` in
  `docker-compose.yml`

  ```shell
  sudo -e /etc/docker/compose/vaultwarden/docker-compose.yml
  ```

  ```yaml
      volumes:
        - /opt/docker/vaultwarden/db/data:/var/lib/postgresql/data
        #- /path/to/dump16.4.sql:/tmp/dump.sql:ro
  ```

- Start all docker compose services

  ```shell
  sudo systemctl restart docker-compose@vaultwarden
  ```
