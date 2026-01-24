# Synapse home server for matrix

## `/opt/docker` folder structure

```shell
root@server:~# tree -L 4 /opt/docker/
/opt/docker/
└── synapse
    ├── caddy
    │   ├── Caddyfile
    │   ├── config
    │   └── data
    ├── postgres
    │   ├── data
    │   │   └── 18
    │   │       └── docker
    │   └── runtime.env
    └── synapse
        ├── config
        │   ├── chat.yourdomain.tld.log.config
        │   ├── chat.yourdomain.tld.signing.key
        │   └── homeserver.yaml
        └── data

17 directories, 8 files
```

## How to generate keys

Source: [element installation](https://element-hq.github.io/synapse/latest/setup/installation.html)

Before you can start Synapse, you will need to generate a configuration file.
To do this, run (in your virtualenv, as before):

```shell
cd ~/synapse
python -m synapse.app.homeserver \
    --server-name chat.yourdomain.tld \
    --config-path homeserver.yaml \
    --generate-config \
    --report-stats=[yes|no]
```

substituting an appropriate value for --server-name and choosing whether or not
to report usage statistics (hostname, Synapse version, uptime, total users,
etc.) to the developers via the --report-stats argument.

This command will generate you a config file that you can then customise, but
it will also generate a set of keys for you.
