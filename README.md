# Docker iDempiere 7.1

- [Docker Hub](https://hub.docker.com/r/ingeinthub/idempiere)

## Getting Started

```
$ docker swarm init
```

#### Using Docker Commands

iDempiere Docker uses a postgres admin password and user example `POSTGRES_PASSWORD=postgres`, if you use diferent configuration
see [Environment Variables](#environment-variables):

```bash
$ docker run -d --name postgres -p 5432:5432 -e POSTGRES_PASSWORD=postgres postgres:9.6
```

```bash
$ docker run -d --name idempiere -p 8080:8080 --link postgres:postgres ingeinthub/idempiere:7.1
```

Open in the browser: [http://127.0.0.1:8080/webui/](http://127.0.0.1:8080/webui/)

#### Using Docker Stack

Create a `docker-stack.yml` file:

```yaml
version: '3.7'

services:
  idempiere:
    image: ingeinthub/idempiere:7.1
    volumes:
      - idempiere_config:/idempiere/configuration
      - idempiere_plugins:/idempiere/plugins/customs
    environment:
      - TZ=America/Guayaquil
    ports:
      - 8080:8080
      - 8443:8443
      - 12612:12612

  postgres:
    image: postgres:9.6
    volumes:
      - idempiere_data:/var/lib/postgresql/data
    environment:
      - TZ=America/Guayaquil
      - POSTGRES_PASSWORD=postgres
    ports:
      - 5432:5432

volumes:
  idempiere_data:
```

```bash
$ docker stack deploy -c docker-stack.yml idempiere
```

#### Using Makefile to Create from Dockerfile

```
$ make build run
```

Command list:

`make build` creates iDempiere docker image (with labels `idempiere:7.1` and `idempiere:latest`)

`make run` runs iDempiere docker stack (includes `postgres:9.6`)

`make stop` stops the stack

`make log` shows the logs of iDempiere

`make bash` creates a terminal within iDempiere docker image

## Default Accounts

The following users and passwords are part of the initial seed database:

| Usage | User | Password |
| - | - | - |
| System Management | System | System |
| System Management or any role/company | SuperUser | System |
| Sample Client Administration | GardenAdmin | GardenAdmin |
| Sample Client User | GardenUser | GardenUser |

## Environment Variables

| Variable | Default Value | Description |
| - | - | - |
| IDEMPIERE_HOME | /idempiere | Path to iDempiere |
| IDEMPIERE_VERSION | 7.1 | iDempiere Version |
| IDEMPIERE_PORT | 8080 | HTTP port for iDempiere |
| IDEMPIERE_SSL_PORT | 8443 | HTTPS port for iDempiere |
| KEY_STORE_PASS | myPassword | Password for java key store (SSL Certificate) |
| KEY_STORE_ON | idempiere.org | Common Name for SSL Certificate |
| KEY_STORE_OU | iDempiere Docker | Organization Unit for SSL Certificate |
| KEY_STORE_O | iDempiere | Organization for SSL Certificate |
| KEY_STORE_L | iDempiere | Locate/Town for SSL Certificate |
| KEY_STORE_S | CA | State for SSL Certificate |
| KEY_STORE_C | US | Country for SSL Certificate |
| DB_HOST | postgres | Hostname for Postgres database |
| DB_PORT | 5432 | Postgres port |
| DB_NAME | idempiere | iDempiere Database |
| DB_USER | adempiere | User for iDempiere Database |
| DB_PASS | adempiere | Pass for `DB_USER` |
| DB_ADMIN_PASS | postgres | Pass for admin user (`postgres` user) |
| MAIL_HOST | idempiere | Mail server |
| MAIL_USER | info | Mail user |
| MAIL_PASS | info | Mail password |
| MAIL_ADMIN | info@idempiere | Admin e-mail |
| JAVA_HOME | /usr/local/openjdk-11 | Path to java |
| JAVA_OPTS |  | Java execution parameters (e.g. `-Xms` and `-Xmx`) |
| DEBUG_PORT| 4554 | Port for remote debug |
| TELNET_PORT | 12612 | OSGI port for telnet connection |
| MIGRATE_EXISTING_DATABASE | false | Migrate the existing database to latest version |

## Default Ports

| Port | Description |
| - | - |
| 8080 | Default HTTP port for iDempiere |
| 8443 | Default HTTPS port for iDempiere |
| 12612 | Default OSGI port for telnet connection |
| 4554 | Default remote debug port |

## How it works

iDempiere starts looking for an existing database, if it don't exist
iDempiere will create a seed database.

> If the database exists iDempiere won't migrate it, you must do it manually, or using `MIGRATE_EXISTING_DATABASE=true`.

Before starting, it will configure all the settings according to
the `setup.sh` or `console-setup.sh` files. See [docker-entrypoint.sh](docker-entrypoint.sh) file.

> This project has not support for oracle database.

## Volumes

Postgres Data:

```yaml
volumes:
  - idempiere_data:/var/lib/postgresql/data
```

iDempiere Plugins:

```yaml
volumes:
  - idempiere_config:/idempiere/configuration
  - idempiere_plugins:/idempiere/plugins/customs
```

`idempiere_config` saves the plugins configuration and `idempiere_plugins` is
the path to contain the jar plugin.

Other way to share plugins:

```yaml
volumes:
  - idempiere_config:/idempiere/configuration
  - ./plugins:/idempiere/plugins/customs
```

## Run as Debug

Adding `command` yml property:
```yaml
command: idempiere debug
```

## Docker Secrets

As an alternative to passing sensitive information via environment variables,
`_FILE` may be appended to some of the previously listed environment variables,
causing the initialization script to load the values for those variables
from files present in the container. See [Docker Secrets](https://docs.docker.com/engine/swarm/secrets/) and [Docker PosgreSQL](https://hub.docker.com/_/postgres).

#### Variable list:

| Variable | Original variable |
| - | - |
| DB_ADMIN_PASS_FILE | DB_ADMIN_PASS |
| DB_PASS_FILE | DB_PASS |
| MAIL_PASS_FILE | MAIL_PASS |
| KEY_STORE_PASS_FILE | KEY_STORE_PASS |

#### Example:

Create the secret:

```bash
$ printf "postgres" | docker secret create db_admin_pass -
```

Use environment variable in the stack file:

```yaml
version: '3.7'

services:
  idempiere:
    image: idempiere:7.1
    ports:
      - 8080:8080
      - 8443:8443
      - 12612:12612
    environment:
      - TZ=America/Guayaquil
      - DB_ADMIN_PASS_FILE=/run/secrets/db_admin_pass
    secrets:
      - db_admin_pass

  postgres:
    image: postgres:9.6
    volumes:
      - idempiere_data:/var/lib/postgresql/data
    environment:
      - TZ=America/Guayaquil
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_admin_pass
    secrets:
      - db_admin_pass
    ports:
      - 5432:5432

volumes:
  idempiere_data:

secrets:
  db_admin_pass:
    external: true
```