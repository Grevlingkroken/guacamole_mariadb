# Guacamole with docker-compose and MariaDB

This is my fork of https://github.com/boschkundendienst/guacamole-docker-compose using MariaDB rather than Postgres. All the documentation can be found at the original repo, so I have
just added what has changed in my repo.


## About Guacamole
Apache Guacamole (incubating) is a clientless remote desktop gateway. It supports standard protocols like VNC, RDP, and SSH. It is called clientless because no plugins or client software are required. Thanks to HTML5, once Guacamole is installed on a server, all you need to access your desktops is a web browser.

It supports RDP, SSH, Telnet and VNC and is the fastest HTML5 gateway I know. Checkout the projects [homepage](https://guacamole.incubator.apache.org/) for more information.

## Prerequisites
You need a working **docker** installation and **docker-compose** running on your machine.

## Quick start
Clone the GIT repository and start guacamole:

~~~bash
git clone "https://github.com/Grevlingkroken/guacamole_mariadb.git"
cd guacamole_mariadb
sudo bash prepare.sh
~~~
I prefer to keep passwords/secrets outside the compose-files, so edit the .env file and set your password for the MariaDB-users root and guacamole like
~~~
GUACAMOLE_PASSWORD=MyGuacuser-pwd
MARIADB_ROOT_PASSWORD=MyRootDB-pwd
~~~

Then run
~~~bash
docker-compose up -d
~~~

Your guacamole server should now be available at `https://ip of your server:8443/`. The default username is `guacadmin` with password `guacadmin`.

## Details
To understand some details let's take a closer look at parts of the `docker-compose.yml` file:

### Networking
The following part of docker-compose.yml will create a network with name `guacnet` in mode `bridged`.
~~~python
...
# networks
# create a network 'guacnet' in mode 'bridged'
networks:
  guacnet:
    driver: bridge
...
~~~

### Services
#### guacd
The following part of docker-compose.yml will create the guacd service. guacd is the heart of Guacamole which dynamically loads support for remote desktop protocols (called "client plugins") and connects them to remote desktops based on instructions received from the web application. The container will be called `guacd` based on the docker image `guacamole/guacd` connected to our previously created network `guacnet`. Additionally we map the 2 local folders `./drive` and `./record` into the container. We can use them later to map user drives and store recordings of sessions.

~~~python
...
services:
  # guacd
  guacd:
    container_name: guacd
    image: guacamole/guacd
    networks:
      guacnet:
    restart: always
    volumes:
    - ./drive:/drive:rw
    - ./record:/record:rw
...
~~~

### MariaDB
The following part of docker-compose.yml will create an instance of MariaDB using the official docker image. For an even smaller footprint you can use an Alpine-based image like yobasystems/alpine-mariadb:10.11.5

~~~python
...
  mariadb:
    container_name: mariadb_guacamole
    image: mariadb:10.11.5
    environment:
      MARIADB_ROOT_PASSWORD: ${MARIADB_ROOT_PASSWORD}
      MYSQL_DATABASE: guacamole_db
      MYSQL_PASSWORD: ${GUACAMOLE_PASSWORD}
      MYSQL_USER: guacamole_user
    networks:
      guacnet:
    restart: always
    volumes:
    - ./init:/docker-entrypoint-initdb.d:z
    - ./data:/var/lib/mysql/:rw
...

~~~
## MariaDB notes
By default MariaDB has a samaple-db with anonymous access enabled. It might be a good idea to enter the container and do some extra work to harden it

~~~bash
docker exec -it mariadb_guacamole /bin/bash
~~~
Then run the following script
~~~bash
/usr/bin/mariadb-secure-installation
~~~
You will be prompted with something like this

~~~python
NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user. If you've just installed MariaDB, and
haven't set the root password yet, you should just press enter here.

Enter current password for root (enter for none): (enter password from the .env file)
OK, successfully used password, moving on...

Setting the root password or using the unix_socket ensures that nobody
can log into the MariaDB root user without the proper authorisation.

You already have your root account protected, so you can safely answer 'n'.

Switch to unix_socket authentication [Y/n] n
 ... skipping.

You already have your root account protected, so you can safely answer 'n'.

Change the root password? [Y/n] n
 ... skipping.

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] Y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.
~~~~~~

### Guacamole
The following part of docker-compose.yml will create an instance of guacamole by using the docker image `guacamole` from docker hub. It is also highly configurable using environment variables. In this setup it is configured to connect to the previously created MariaDDB instance using a username and password and the database `guacamole_db`. Guacamole also has native support for TOTP, so I highly recommend enabling this as soon as you have verified that setup is working properly. Port 8080 is only exposed locally! We will attach an instance of nginx for public facing of it in the next step.

~~~python
...

  # guacamole
  guacamole:
    container_name: guacamole
    depends_on:
    - guacd
    - mariadb
    environment:
      GUACD_HOSTNAME: guacd
      MYSQL_DATABASE: guacamole_db
      MYSQL_HOSTNAME: mariadb
      MYSQL_PASSWORD: ${GUACAMOLE_PASSWORD}
      MYSQL_USER: guacamole_user
      # TOTP_ENABLED: "true" #Enable this when you know the install is working
    image: guacamole/guacamole
    links:
    - guacd
    networks:
      guacnet:
    ports:
    - 8080/tcp
    restart: always
...
~~~

#### nginx
The following part of docker-compose.yml will create an instance of nginx that maps the public port 8443 to the internal port 443. The internal port 443 is then mapped to guacamole using the `./nginx/templates/guacamole.conf.template` file. The container will use the previously generated (`prepare.sh`) self-signed certificate in `./nginx/ssl/` with `./nginx/ssl/self-ssl.key` and `./nginx/ssl/self.cert`.

~~~python
...
  # nginx
  nginx:
   container_name: nginx_guacamole
   restart: always
   image: nginx
   volumes:
   - ./nginx/templates:/etc/nginx/templates:ro
   - ./nginx/ssl/self.cert:/etc/nginx/ssl/self.cert:ro
   - ./nginx/ssl/self-ssl.key:/etc/nginx/ssl/self-ssl.key:ro
   ports:
   - 8443:443
   links:
   - guacamole
   networks:
     guacnet:
...
~~~




## prepare.sh
`prepare.sh` is a small script that creates `./init/initdb.sql` by downloading the docker image `guacamole/guacamole` and start it like this:

~~~bash
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > ./init/initdb.sql
~~~

It creates the necessary database initialization file for MariaDB.

`prepare.sh` also creates the self-signed certificate `./nginx/ssl/self.cert` and the private key `./nginx/ssl/self-ssl.key` which are used
by nginx for https.

## reset.sh
To reset everything to the beginning, just run `./reset.sh`.


**Disclaimer**

Downloading and executing scripts from the internet may harm your computer. Make sure to check the source of the scripts before executing them!
