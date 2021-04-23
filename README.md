# Local Postgres with Docker Compose

Run

```
docker-compose down
docker-compose pull
docker compose up -d
```

Go to `localhost:8000`

# Setting up the docker-compose volumes and network

`docker-compose` derives the network name from the directory name containing the
`docker-compose.yml` file. In this example the network name is `docker-compose-postgres_default`.

# Setting up the Postgres Container

The `docker-compose.yml` contains a service named `postgres` defined below.

```yml
services:
  postgres:
    container_name: demo_postgres
    image: "postgres"
    environment:
      POSTGRES_USER: "postgres"
      POSTGRES_PASSWORD: "postgres"
      PGDATA: "/data/postgres"
    volumes:
      - postgres:/data/postgres
      - ./docker_postgres_init.sql:/docker-entrypoint-initdb.d/docker_postgres_init.sql
    ports:
      - "15432:5432"
    restart: unless-stopped
```

To configure the administrative user for the database we set the `POSTGRES_USER` and
`POSTGRES_PASSWORD` environment variables.

Postgres database files are stored in `/data/postgres`.
This directory is mapped to postgres volume via the mapping `postgres:/data/postgres`.

When the postgres container starts it looks for a file called `docker_postgres_init.sql`
which will be executed during start up to configure the database.

```SQL
CREATE DATABASE graphql
    WITH
    OWNER = postgres
    ENCODING = 'UTF8'
    LC_COLLATE = 'en_US.utf8'
    LC_CTYPE = 'en_US.utf8'
    TABLESPACE = pg_default
    CONNECTION LIMIT = -1;
```

The init sql file is mapped to the place where the postgres container expects it to
be via the volume mapping below.

```yaml
volumes:
  - ./docker_postgres_init.sql:/docker-entrypoint-initdb.d/docker_postgres_init.sql
```

In order to make the postgres database running inside the docker container accessible to
applications on the workstation we map the default postgres port `5432` to
`15432` as shown by the docker-compose configuration below.

```yaml
ports:
  - "15432:5432"
```

# Setting up the pgAdmin 4 Container

To set up the pgAdmin container we use the following service in the `docker-compose` file

```yaml
pgadmin:
container_name: demo_pgadmin
image: "dpage/pgadmin4"
environment:
  PGADMIN_DEFAULT_EMAIL: admin@admin.com
  PGADMIN_DEFAULT_PASSWORD: admin
  PGADMIN_CONFIG_SERVER_MODE: "False"
  PGADMIN_CONFIG_MASTER_PASSWORD_REQUIRED: "False"
volumes:
  - pgadmin:/var/lib/pgadmin
  - ./docker_pgadmin_servers.json:/pgadmin4/servers.json
ports:
  - "8000:80"
entrypoint:
  - "/bin/sh"
  - "-c"
  - "/bin/echo 'postgres:5432:*:postgres:password' > /tmp/pgpassfile && chmod 600 /tmp/pgpassfile && /entrypoint.sh"
restart: unless-stopped
```

Since we are running pgAdmin on developer workstation accessible only on localhost we want to
skip pgAdmin authentication. Therefore, we run pgAdmin in desktop mode by setting the
environment variable `PGADMIN_CONFIG_SERVER_MODE: "False"`.

Even though pgAdmin is running in desktop mode we still need to set up an admin
username and password using the environment variables below.

```yaml
environment:
  PGADMIN_DEFAULT_EMAIL: admin@admin.com
  PGADMIN_DEFAULT_PASSWORD: admin
```

Master password can be disabled with the environment variable `PGADMIN_CONFIG_MASTER_PASSWORD_REQUIRED: "False"`.

To avoid having to enter the connection settings we can create a json file with the connection
details that pgAdmin will import into its configuration the first time it starts.

```json
{
  "Servers": {
    "1": {
      "Name": "Docker Compose",
      "Group": "Servers",
      "Port": 5432,
      "Username": "postgres",
      "Host": "postgres",
      "SSLMode": "prefer",
      "MaintenanceDB": "postgres",
      "PassFile": "/tmp/pgpassfile"
    }
  }
}
```

The pgAdmin container looks for connections file at `pgadmin4/servers.json` therefore
we store the configuration file at `docker_pgadmin_servers.json` and map it into
the pgAdmin container using the mapping `./docker_pgadmin_servers.json:/pgadmin4/servers.json`

In order for pgAdmin to store it's state across container restarts we map the location
it stores state `/var/lib/pgadmin` to the docker volume `pgadmin` as shown in the yaml
below.

```yaml
volumes:
  - pgadmin:/var/lib/pgadmin
  - ./docker_pgadmin_servers.json:/pgadmin4/servers.json
```

pgAdmin provides no mechanism to set passwords for the connections in `servers.json` since
database passwords are sensitive and therefore should not be stored locally. This is a
very good policy on pgAdmin's part but given our goal of being able to just land directly
in the UI we have to override the pgAdmin container entrypoint as shown below.

```yaml
entrypoint:
  - "/bin/sh"
  - "-c"
  - "/bin/echo 'postgres:5432:*:postgres:password' > /tmp/pgpassfile && chmod 600 /tmp/pgpassfile && /entrypoint.sh"
```

Docker executes an entrypoint via exec() function call without any shell expansion.
A docker entry point is an array of strings the first is the name of the executable
to run followed by the parameters to pass to the executable. To make formatting easier in the
`docker-compose` file I used the list form as show in the code snippet above.
If we remove the yaml and docker entrypoint formatting the command executed is the one below.

```bash
/bin/sh -c /bin/echo 'postgres:5432:*:postgres:password' > /tmp/pgpassfile && chmod 600 /tmp/pgpassfile && /entrypoint.sh
```

`/bin/sh -c` is used to execute the shell with input from the command line rather than from a file.
The `&&` is used to chain a series of separate commands on a single line. If any command fails
subsequent commands will not be executed. With these advanced shell features explained we can
breakdown the command into individual steps.

```bash
/bin/echo 'postgres:5432:*:postgres:password' > /tmp/pgpassfile
```

The command above will write a postgres password file to `/tmp/pgpassfile` which is referenced
in the `servers.json`

```json
{
  "Servers": {
    "1": {
      "PassFile": "/tmp/pgpassfile"
    }
  }
}
```

A postgres password file has the format `hostname:port:database:username:password` with
the ability to use `*` a wild card match on the first 4 fields. So we write
`postgres:5432:*:postgres:password` into the `/tmp/pgpassfile` notice that we use
`*` for the database portion this means that for all databases pgAdmin will try to use
`postgres` as the username and `password` as the password.

For security reasons pgAdmin will ignore any password files that are not locked down with
unix permissions of 600. Therefore, the next command we execute is `chmod 600 /tmp/pgpassfile`

With the password file written with the correct unix permission we launch the pgAdmin container
entry point by calling `/entrypoint.sh` which on startup will find password file and server
connections that point to the postgres we launched in docker. The net result is that you can
just click around the pgAdmin UI, and you will not have to set up any connections or passwords.
