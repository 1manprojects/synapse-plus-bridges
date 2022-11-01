# Matrix Synapse with Telegram and Signal Bridge
While running my own Synapse server I found it a bit difficult to run everything with multiple bridge services easily. So I created a docker-compose file to run Synapse with Telegram and Signal bridge services with one Database.


## Disclaimer
This Guide is not perfect and not all settings are going to be mentioned. For a complete configuration guide, i recommend you read the synapse docs https://synapse.docs.vertex.link/en/latest/index.html and the Bridge setup docs https://docs.mau.fi/bridges/general/docker-setup.html?bridge=telegram.


## Instructions
In the following, we will be assuming you have a URL for your synapse service ready to go. And the Container will be exposing its default Port 8008.

### Getting Started
First reate a directory to hold all data
`mkdir synapse`
and add the docker-compose.yaml file from this repo into it.
!!!Update the Password in the compose file!!!
Now we will create the necessary directories for the containers.
`cd synapse`
`mkdir db`
`mkdir signal`
`mkdir signald`
`mkdir synapse`
`mkdir telegram`

Now to create the matrix-synapse configuration and set the necessary file permissions run the latest synapse docker container.

Change the SYNAPSE_SERVER_NAME to your own URL.
```
docker run -it --rm \
    -v ./synapse:/data \
    -e SYNAPSE_SERVER_NAME=my.matrix.host \
    -e SYNAPSE_REPORT_STATS=yes \
    matrixdotorg/synapse:latest generate
```
There now should be a "homeserver.yaml" file located in the synapse/synapse folder.

Now run the following command to start the Postgresql container.
`docker-compose up - db`

And create the necesary Databases with the following commands.

`docker exec -it synapse_db_1 bash`
`psql -U postgres`

`CREATE DATABASE signal;`
`CREATE DATABASE telegram;`
```
CREATE DATABASE matrix
 ENCODING 'UTF8'
 LC_COLLATE='C'
 LC_CTYPE='C'
 template=template0;
```

Now exit postgresql and the container

`\q`
`exit`

Now we update the homeserver.yaml configuration. Here we need to update the domain and baseurl to the synapse-URL and set the Database configuration.

```
database:
  name: psycopg2
  args:
    user: postgres
    password: <CHANGE_ME>
    database: matrix
    host: db
    cp_min: 5
    cp_max: 10
```

Now we will create a admin user for Synapse.
First start the Synapse container
`docker-compose up - synapse`
exec into the container
`docker exec -it synapse_synapse_1 bin/bash`
and to create a user run the following
`register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008`

### Telegram Bridge

To create the configurations for Telegram and Signal run the following commands
`docker-compose up mautrix-signal`
`docker-compose up telegram`
`docker-compose down`
This will create a "config.yaml" file in the signal and telegram directory that need to be edited and will stop all containers running.

First, we will edit the Telegram config file. Here you need to change the following lines.

```
homeserver:
    address: https://my.matrix.host
    domain: my.matrix.host
appservice:
    address: http://telegram:29317
    database: postgres://postgres:<CHANGE_ME>@db/telegram
telegram:
    # Get your own API keys at https://my.telegram.org/apps
    api_id: YOUR_API_KEY
    api_hash: API_HASH
permissions:
    '*': relaybot
    web.my.matrix.host: user
    my.matrix.host: full
    '@root:my.matrix.host': admin
```

The following is optional and will allow the bridge bot to function in encrypted rooms.
```
encryption:
    # Allow encryption, work in group chat rooms with e2ee enabled
    allow: true
```

### Signal Bridge

Now to set the Signal config file. Here you need to change the following lines. 

```
homeserver:
    address: https://my.matrix.host
    domain: my.matrix.host
appservice:
    address: http://mautrix-signal:29328
    database: postgres://postgres:<CHANGE_ME>@db/signal
permissions:
    '*': relay
    my.matrix.host: user
    '@admin:my.matrix.host': admin
```
The following is optional and will allow the bridge bot to function in encrypted rooms.
```
encryption:
    # Allow encryption, work in group chat rooms with e2ee enabled
    allow: true
```
### Bridging the container

After editing the configurations we can create a registration file for each bridge service by running the same commands to create the initial configurations.
`docker-compose up mautrix-signal`
`docker-compose up telegram`
`docker-compose down`
Now we can copy the registraion files for telegram and signal to the synapse directory.
`cp signal/registration.yaml synapse/signal_registration.yaml`
`cp telegram/registration.yaml synapse/telegram_registration.yaml`
The path to these files have to be added to the homeserver.yaml file in the synapse directory
```
app_service_config_files:
   - data/signal_registration.yaml
   - data/telegram_registration.yaml
```

Now everything should be setup correctly and we can start the containers with
`docker-compose up -d`

### Final Steps
From here we can set up a reverse proxy with an ssl cert using apache or ngnix. Once that is done you can log in to synapse and create two rooms one for each bridge and invite the bridge bot.