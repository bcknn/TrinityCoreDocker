# Trinity Core
This image will allow you to run your own complete dockerized WoW private server, using the Trinity Core open source MMO framework.  The guide was written to use with Docker Desktop on Windows, so adapt as necessary.

## Initial Setup
Before running your private realm, you need to set up your data files, and initialize your database.

### Docker Networking
Create a network to use with this deployment
```
docker network create kbt_net
```

### Data Setup
First, you need to build data files using the 3.3.5 WoW client.  Store this in a data volume or host location.  Your data will be stored in /data, and the client will be pulled from /wowclient.  You only need to do this once.  Note that due to the nature of the data pulling files, there will be temporary data written to the WoW client directory so it cannot be mounted as read-only.  You will need to find the client yourself, and it will need to be the correct version.
```
docker run --name kbtrinity -it -v <DATA VOLUME>\client:/wowclient -v <DATA VOLUME>:/data -d ghcr.io/bcknn/trinitycore:latest --builddata
```

### Database Setup
You will need to configure a database server to use with this
```
docker run -p 3306:3306 --name kbt_db --network kbt_net -e MYSQL_HOST=kbt_db -e MYSQL_PORT=3306 -e MYSQL_ROOT_PASSWORD=root -v <DATA VOLUME>\kbdb:/var/lib/mysql -d mysql:latest
```

### Database Initialization
Next, you need to initialize your database with the create script which will create the necessary databases, and the non-root user that will be used in the application.  This is also executed only once.  There following environment variables are used:

* MYSQL_PORT: The port for your database (defaults to *3306*)
* MYSQL_HOST: The MySQL hostname (defaults to host.docker.internal)
* MYSQL_USER: The username of the non-root user to be used (defaults to *trinity*)
* MYSQL_PASSWORD: the password of the non-root user to be used (defaults to *trinity*)
* MYSQL\_USER_HOST: The host mask to be allowed for the non-root user (defaults to *%* which is any host)

Example:

```
docker run --rm -e MYSQL_PORT=3306 -e MYSQL_HOST=kbt_db -e MYSQL_ROOT_PASSWORD=root --network kbt_net ghcr.io/bcknn/trinitycore:latest --dbinit
```

Note, that if you are running MySQL 5.7 that you should disable strict mode in your database:
```
mysql -u root -p -e "SET GLOBAL sql_mode = 'NO_ENGINE_SUBSTITUTION';" 
```

### Configuration File Setup

Create a config volume which will store your `worldserver.conf` and `bnetserver.conf` files.  You can copy the distribution's versions from `/server/etc`.  A few quick commands that will accomplish this:

```
docker run -it --rm -v <DATA VOLUME>\config:/config ghcr.io/bcknn/trinitycore:latest cp /server/etc/worldserver.conf.dist /config/worldserver.conf
docker run -it --rm -v <DATA VOLUME>\config:/config ghcr.io/bcknn/trinitycore:latest cp /server/etc/bnetserver.conf.dist /config/authserver.conf
```

The mandatory configurations are as follows:

* In both `worldserver.conf` and `authserver.conf`, change all database references to your MySQL database.
* In `worldserver.conf`, change the `DataDir` variable to /data.

Optionally, you can create your own logs volume.  Set the `LogsDir` variable to `/logs` in both `worldserver.conf` and `authserver.conf` files.

* Finally, update the `realmlist` MySQL table in the `auth` to set the external and internal IP addresses of your realm by setting both to your docker host address.

## Run Your Realm
Run your realm by kicking off two containers, one for the world server and the other for the bnet server.  Use the command line options `--worldserver` and `--bnetserver`, respectively.  Note that for the world server you must enable STDIN and TTY (e.g. `docker run -it`) and for the auth server you must enable TTY.

A complete example Docker Compose file which encapsulates everything (but with persistent volumes that are created externally of the compose file):

```
version: '2'

services:
  kbt_db:
    restart: always
    image: mysql:latest
    environment:
      - MYSQL_ROOT_PASSWORD=root
    volumes:
      - <DATA VOLUME>\kbdb:/var/lib/mysql
  kbt_wrld:
    image: ghcr.io/blockkieran/trinitycore:latest
    command: --worldserver
    tty: true
    stdin_open: true
    ports:
      - '8085:8085'
    volumes:
      - <DATA VOLUME>\data:/data
      - <DATA VOLUME>\config:/config
      - <DATA VOLUME>\logs:/logs 
  kbt_auth:
    image: ghcr.io/blockkieran/trinitycore:latest
    command: --authserver
    tty: true
    ports:
      - '3724:3724'
    volumes:
      - <DATA VOLUME>\data:/data
      - <DATA VOLUME>\config:/config
      - <DATA VOLUME>\logs:/logs

```

### Admin Account Setup
Use `docker attach` to attach into your world server instance, and use the following command to create an account:

```
account create <username> <password>
```

Optionally, you can elevate the user to have GM powers:

```
account set gmlevel <username> 3 -1
```

### WoW Client Setup
Modify your `realmlist.wtf` file inside your `data` directory to the following:
```
set realmlist <DOCKER HOST IP ADDRESS>
```

Additionally, and this may not be necessary but I have changed so much to make this work I have no idea, change your WTF\Config.wtf file
```
SET realmList "<DOCKER HOST IP ADDRESS>"
```

## Caveats
Frank's caveat's: The goal of the docker image was to streamline as much of the setup as possible within reason, but it was not designed to be a complete keyturn solution.  Please refer to [the wiki](https://trinitycore.atlassian.net/wiki/spaces/tc/pages/2130077/Installation+Guide) for step by step details if you run into issues.

Updated: This configuration was sadly abandoned, but so much good work was done that it was possible to get it working easily enough.  It does look like Frank was trying to update it for the latest version of work and not WOTLK, which I have rolled back to make it work.  Most errors you will receive can be solved through the official trinitycore help or basic docker troubleshooting.  I hope that I have covered enough for you to get set up.
