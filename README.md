# Build container providing MariaDB as a service

A build container providing [MariaDB][mariadb] as service. Originally it was
created for building projects using the [Drone][drone] continuous integration
platform but should work in other CI environments, too.

Contrary to the official [MariaDB docker image][docker-mariadb], this image is
based on [Alpine Linux][alpine] version 3.2 / 3.3, and builds upon the
[Gliderlabs Alpine image][docker-gliderlabs].

Tags for MariaDB 5.5 and 10.1 are available.

## What is MariaDB?
MariaDB is a community-developed fork of the MySQL relational database management
system intended to remain free under the GNU GPL. Being a fork of a leading open
source software system, it is notable for being led by the original developers of
MySQL, who forked it due to concerns over its acquisition by Oracle. Contributors
are required to share their copyright with the MariaDB Foundation.

The intent is also to maintain high compatibility with MySQL, ensuring a "drop-in"
replacement capability with library binary equivalency and exact matching with
MySQL APIs and commands. It includes the **XtraDB** storage engine for replacing
**InnoDB**, as well as a new storage engine, **Aria**, that intends to be both a
transactional and non-transactional engine perhaps even included in future
versions of MySQL.

> <https://en.wikipedia.org/wiki/MariaDB>

![MariaDB][mariadb-logo]

## How to use this image

### Start a `mariadb` server instance

```
$ docker run --name some-mariadb -e MYSQL_ROOT_PASSWORD=my-secret-pw -d danielsreichenbach/mariadb:tag
```

... where `some-mariadb` is the name you want to assign to your container,
`my-secret-pw` is the password to be set for the MariaDB root user and `tag` is
the tag specifying the MariaDB version you want. See the tag list on Docker Hub
for available tags.

### Connect to MariaDB from an application in another Docker container

Since MariaDB is intended as a drop-in replacement for MariaDB, it can be used
with many applications. This image exposes the standard MySQL port (3306), so
container linking makes the MariaDB instance available to other application
containers. Start your application container like this in order to link it to
the MariaDB container:

```
$ docker run --name some-app --link some-mariadb:mysql -d application-that-uses-mysql
```

### Connect to MariaDB from the MariaDB command line client

The following command starts another MariaDB container instance and runs the
`mysql` command line client against your original MariaDB container, allowing you
to execute SQL statements against your database instance:

```
$ docker run -it --link some-mariadb:mysql --rm mariadb sh -c 'exec mysql -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD"'
```

... where `some-mariadb` is the name of your original MariaDB container.

More information about the MySQL command line client can be found in the
[MariaDB documentation][mariadb-doc].

### Container shell access and viewing logs

The `docker exec` command allows you to run commands inside a Docker container.
The following command line will give you a sh shell inside your MariaDB
container:

```
$ docker exec -it some-mariadb sh
```

The MariaDB Server log is available through Docker's container log:

```
$ docker logs some-mariadb
```
### Using a custom MariaDB configuration file

The MariaDB startup configuration is specified in the file `/etc/mysql/my.cnf`,
and that file in turn includes any files found in the `/etc/mysql/conf.d`
directory that end with `.cnf`. Settings in files in this directory will augment
and/or override settings in `/etc/mysql/my.cnf`. If you want to use a customized
MariaDB configuration, you can create your alternative configuration file in a
directory on the host machine and then mount that directory location as
`/etc/mysql/conf.d` inside the MariaDB container.

If `/my/custom/config-file.cnf` is the path and name of your custom configuration
file, you can start your MariaDB container like this (note that only the directory
path of the custom config file is used in this command):

```
$ docker run --name some-mariadb -v /my/custom:/etc/mysql/conf.d -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mariadb:tag
```

This will start a new container `some-mariadb` where the MariaDB instance uses
the combined startup settings from `/etc/mysql/my.cnf` and
`/etc/mysql/conf.d/config-file.cnf`, with settings from the latter taking
precedence.

Note that users on host systems with SELinux enabled may see issues with this.
The current workaround is to assign the relevant SELinux policy type to your new
config file so that the container will be allowed to mount it:

```
$ chcon -Rt svirt_sandbox_file_t /my/custom
```

### Environment Variables

When you start the `mariadb` image, you can adjust the configuration of the MariaDB
instance by passing one or more environment variables on the docker run command line.
Do note that none of the variables below will have any effect if you start the container
with a data directory that already contains a database: any pre-existing database will
always be left untouched on container startup.

#### `MYSQL_ROOT_PASSWORD`

This variable is mandatory and specifies the password that will be set for the
MariaDB root superuser account. In the above example, it was set to `my-secret-pw`.

#### `MYSQL_DATABASE`

This variable is optional and allows you to specify the name of a database to be
created on image startup. If a user/password was supplied (see below) then that
user will be granted superuser access (corresponding to `GRANT ALL`) to this database.

#### `MYSQL_USER, MYSQL_PASSWORD`

These variables are optional, used in conjunction to create a new user and to set
that user's password. This user will be granted superuser permissions (see above)
for the database specified by the `MYSQL_DATABASE` variable. Both variables are
required for a user to be created.

Do note that there is no need to use this mechanism to create the root superuser,
that user gets created by default with the password specified by the
`MYSQL_ROOT_PASSWORD` variable.

#### `MYSQL_ALLOW_EMPTY_PASSWORD`

This is an optional variable. Set to yes to allow the container to be started
with a blank password for the root user. NOTE: Setting this variable to yes is
not recommended unless you really know what you are doing, since this will leave
your MariaDB instance completely unprotected, allowing anyone to gain complete
superuser access.

### Initializing a fresh instance

When a container is started for the first time, a new database `mysql` will be
initialized with the provided configuration variables. Furthermore, it will
execute files with extensions `.sh` and `.sql` that are found in
`/docker-entrypoint-initdb.d`. You can easily populate your `mariadb` services
by [mounting a SQL dump into that directory][docker-mount] and provide
[custom images][docker-builder] with contributed data.

### Caveats Where to Store Data

Important note: There are several ways to store data used by applications that
run in Docker containers. We encourage users of the `mariadb` images to
familiarize themselves with the options available, including:

*   Let Docker manage the storage of your database data by writing the database
    files to disk on the host system using its own internal [volume management][docker-volumes].
    This is the default and is easy and fairly transparent to the user. The
    downside is that the files may be hard to locate for tools and applications
    that run directly on the host system, i.e. outside containers.
*   Create a data directory on the host system (outside the container) and [mount][docker-mount]
    this to a directory visible from inside the container. This places the database
    files in a known location on the host system, and makes it easy for tools and
    applications on the host system to access the files. The downside is that the
    user needs to make sure that the directory exists, and that e.g. directory
    permissions and other security mechanisms on the host system are set up correctly.

The Docker documentation is a good starting point for understanding the different
storage options and variations, and there are multiple blogs and forum postings
that discuss and give advice in this area. We will simply show the basic procedure
here for the latter option above:

1.  Create a data directory on a suitable volume on your host system, e.g. `/my/own/datadir`.
2.  Start your `mariadb` container like this:

    ```
    $ docker run --name some-mariadb -v /my/own/datadir:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mariadb:tag
    ```

The `-v /my/own/datadir:/var/lib/mysql` part of the command mounts the `/my/own/datadir`
directory from the underlying host system as `/var/lib/mysql` inside the container,
where MariaDB by default will write its data files.

Note that users on host systems with SELinux enabled may see issues with this.
The current workaround is to assign the relevant SELinux policy type to the new
data directory so that the container will be allowed to access it:

```
$ chcon -Rt svirt_sandbox_file_t /my/own/datadir
```

### No connections until MariaDB init completes

If there is no database initialized when the container starts, then a default
database will be created. While this is the expected behavior, this means that it
will not accept incoming connections until such initialization completes. This
may cause issues when using automation tools, such as `docker-compose`, which
start several containers simultaneously.

### Usage against an existing database

If you start your `mariadb` container instance with a data directory that already
contains a database (specifically, a `mysql` subdirectory), the `$MYSQL_ROOT_PASSWORD`
variable should be omitted from the run command line; it will in any case be
ignored, and the pre-existing database will not be changed in any way.

## Supported Docker versions

This image is officially supported on Docker version 1.9.0.

Support for older versions (down to 1.6) is provided on a best-effort basis.

Please see the [Docker installation documentation][docker-install-docs] for
details on how to upgrade your Docker daemon.

## Usage

The container can be used to compose services in Drone, as shown in the
following example:

```yaml
services:
  mysql:
    image: danielsreichenbach/mariadb:10.1
    environment:
      - MYSQL_DATABASE=test
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
```

This would connect a running MariaDB instance to your container, providing
password-less access, and an empty `test` database.

[drone]:                https://github.com/drone/

[mariadb]:              https://mariadb.org/
[mariadb-logo]:         https://raw.githubusercontent.com/docker-library/docs/master/mariadb/logo.png
[mariadb-doc]:          https://mariadb.com/kb/en/mariadb/documentation/
[docker-mariadb]:       https://hub.docker.com/r/library/mariadb/

[docker-install-docs]:  https://docs.docker.com/installation/
[docker-mount]:         https://docs.docker.com/userguide/dockervolumes/#mount-a-host-file-as-a-data-volume
[docker-builder]:       https://docs.docker.com/reference/builder/
[docker-volumes]:       https://docs.docker.com/userguide/dockervolumes/#adding-a-data-volume

[alpine]:               http://alpinelinux.org/
[docker-gliderlabs]:    https://hub.docker.com/r/gliderlabs/alpine/
