# Upgrading Major PostgreSQL versions

## How to upgrade major PostgreSQL version

**NOTE**: The process described below I have only tested on small databases
used by `umami` and `vaultwarden`, I cannot guarantee it'll work fine for other
services.

Source:
<https://helgeklein.com/blog/upgrading-postgresql-in-docker-container/>

1. While running the `"old"` major version of PostgreSQL run to get a dump
   of the `"old"` database in your `${HOME}` directory. Example: umami

   ```shell
   $ cd
   $ sudo docker exec -it <POSTGRESQL_CONTAINER_NAME> bash
   # cd
   # pg_dumpall -U <YOURPGUSER> > dump.sql
   # exit
   $ sudo docker cp <POSTGRESQL_CONTAINER_NAME>:/root/dump.sql dump<old>.sql
   ```

2. Backup `~/dump<old>.sql` using whatever method you use to backup your
   databases
3. Stop the docker compose service

   ```shell
   sudo systemctl stop docker-compose@<YOURSERVICE>
   ```

4. Create a new directory for the `"new"` major version of PostgreSQL data.
   Example: version 18 for umami

   ```shell
   sudo mkdir /opt/docker/umami/db/data/18
   ```

5. Update `docker-compose.yml` to use the new PostgreSQL major version and
   mount the database dump in read only mode. Example umami upgrading from
   PostgreSQL 17 to 18

   ```yaml
   db:
     image: postgres:18.0-alpine3.22
     volumes:
       #- <https://github.com/umami-software/umami/blob/master/sql/schema.postgresql.sql>:/docker-entrypoint-initdb.d/schema.postgresql.sql:ro
       - /path/to/dump<old>.sql:/tmp/dump.sql:ro
       - /opt/docker/umami/db/data/18:/var/lib/postgresql/18/docker
   ```

6. `cd` to the directory where `docker-compose.yml` is location and only start
   the database service. Example: umami

   ```shell
   cd /etc/docker/compose/umami
   sudo docker compose up -d db
   ```

7. Confirm the service started without issues. Example: umami

   <!-- markdownlint-disable MD013 -->
   ```shell
   $ sudo docker tail -f logs umami-db-1
   ...
   2025-10-26 17:18:33.851 UTC [1] LOG:  starting PostgreSQL 18.0 on x86_64-pc-linux-musl, compiled by gcc (Alpine 14.2.0) 14.2.0, 64-bit
   ...
   2025-10-26 17:18:34.054 UTC [1] LOG:  database system is ready to accept connections
   ```
   <!-- markdownlint-enable MD013 -->

8. Now import the `"old"` database dump into the `"new"` database

   ```shell
   $ sudo docker exec -it <POSTGRESQL_CONTAINER_NAME> bash
   # psql -U <YOURPGUSER> < /tmp/dump.sql
   ```

9. Stop the database service

   ```shell
   sudo docker compose down db
   ```

10. Start the docker compose service

    ```shell
    sudo systemctl start docker-compose@<YOURSERVICE>
    ```

11. Confirm your service can use the new databases and your data is correct.

## `pg_upgrade`

I tried the command below and did not work because it required the binaries
from PostgreSQL version 17.

There may be another way to make this work but I don't know it

```shell
sudo docker run --rm \
  -e POSTGRES_DB=umami \
  -e POSTGRES_USER=umami \
  -e POSTGRES_PASSWORD=<REDACTED> \
  --user postgres \
  -v /opt/umami/db/data17:/var/lib/postgresql/17/docker \
  -v /opt/umami/db/data18:/var/lib/postgresql/18/docker \
  postgres:18.0-alpine3.22 \
  pg_upgrade \
  --link \
  --check \
  --old-datadir /var/lib/postgresql/17/docker \
  --new-datadir /var/lib/postgresql/18/docker
```
