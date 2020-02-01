disqus:
<!-- Disables disqus comment system for this page -->

## Before you install

The following installation guide will document setup with [Docker](https://www.docker.com/) because it's what we at *Team 1533 Triple Strange* and *The Red Alliance* use. It's also important to note that StrangeScout is designed to be run behind a reverse proxy, and as such does **not** have built-in support for HTTPS certificates. We do provide example configurations using the [traefik reverse proxy](https://containo.us/traefik/), but you are more than welcome to use your own reverse proxy, or even fork the code and roll in your own support for HTTPS.

### Dependencies

StrangeScout only has two true dependencies:

- [Docker](https://www.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)

*Docker* is the program that allows services to be run as individual *containers* on a host machine, and *Docker Compose* is a utility that simplifies choreography of multiple docker containers.

You will also need to have a domain pointing at the server you're installing StrangeScout on.

---

First, install Docker following the available installation instructions for your distribution of choice, for example on an x86_64 Ubuntu server you would run the following (as root):

[*official instructions*](https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-using-the-repository)

``` shell
#!/bin/bash

# update package lists
apt-get update

# install packages to add repositories
apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

# add the docker GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

# add the docker x86_64 repository
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

# update package lists
apt-get update

# install docker packages
apt-get install docker-ce docker-ce-cli containerd.io
```

---

Secondly, install Docker Compose following the available installation instructions for your distribution of choice, for example on a *nix server you would run the following (as root):

[*official instructions*](https://docs.docker.com/compose/install/#install-compose)

``` shell
#!/bin/bash

# save the docker-compose script to the bin
curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# make it executable
chmod +x /usr/local/bin/docker-compose
```

---

After both dependencies have been installed, ensure the Docker system service has been started and enabled:

``` shell
systemctl start docker
systemctl enable docker
```

## Setting up the reverse proxy

Now that all dependencies have been installed we can begin setting up traefik for our reverse proxy.

For the sake of organization and due to the way docker-compose names containers, you should make a directory to store the reverse proxy files. Run the following two commands before continuing to setup the reverse proxy:

``` shell
mkdir -v reverse-proxy
cd reverse-proxy
```

### Creating a docker network

The first step to setting up traefik is to create a new docker network:

``` shell
docker network create strangescout_main
```

This will allow for containers on the same network to easily communicate with each other. In our case it allows for the StrangeScout server to communicate with the database in its own separate container.

### Creating the certificate file

Any modern website should be served using the HTTPS protocol. This ensures any data sent to or from the users is encrypted and cannot be read by unauthorized persons. Our reverse proxy, traefik, has built in support for HTTPS, and can automatically update certificates for HTTPS using ACME (Automated Certificate Management Environment). To do so, we need somewhere to store these certificates. Create a file with the following command as root:

``` shell
touch acme.json
```

!!! note
    In order for traefik to store certificates in this file, it must be owned by root. When creating the file, ensure you're running the command as root or using sudo!

In order to work properly, we need to set specific permissions on this file. It must *ONLY* be readable and writable by root. Update the files permissions with the following command as root:

``` shell
chmod 600 acme.json
```

!!! note
    `chmod` is used to update file attributes. In this case, we're setting the file permissions to 600, which corresponds to R/W for the owner, and no permissions for anyone else. Because the file is owned by root, only root can access the file. If these permissions are not set traefik will not be able to get certificates.

### Creating the compose file

Now that our network and ACME certificate file have been created, we can put together the docker-compose file for the reverse proxy. To start, we need to create our compose file:

``` shell
touch docker-compose.yml
```

---

Now that we have our compose file, let's start with some basic options:

``` yaml
# docker-compose.yml

version: '3.3'

services:
    reverse-proxy:
        image: traefik:v2.0
        container_name: reverse-proxy
        hostname: reverse-proxy
        restart: always
```

What we're doing here is start off by stating that we're using version 3.3 of the docker-compose syntax. Next we start our list of services, and define a service with the name `reverse-proxy`. Within the reverse-proxy service we specify that it should use the `traefik:v2.0` image from Docker Hub, that the container name should be `reverse-proxy`, the hostname should be `reverse-proxy`, and the container should always restart if it fails.

!!! note
    the *hostname* key sets friendly names that containers can be accessed by on the docker network we created earlier. While this isn't necessary for the reverse proxy container, it will be useful later when we want to specify the url to access the database at.

---

Next, we'll add some command line options that will be given to traefik when it runs:

``` yaml hl_lines="12 13 14 15 16 17 18 19 20 21 22"
# docker-compose.yml

version: '3.3'

services:
    reverse-proxy:
        image: traefik:v2.0
        container_name: reverse-proxy
        hostname: reverse-proxy
        restart: always

        command:
            - --providers.docker
            
            - --entrypoints.web.address=:80
            - --entrypoints.websecure.address=:443
            - --entrypoints.mongodb.address=:27017

            - --certificatesresolvers.leresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
            - --certificatesresolvers.leresolver.acme.email=<youremail@yourdomain.tld>
            - --certificatesresolvers.leresolver.acme.storage=/acme.json
            - --certificatesresolvers.leresolver.acme.tlschallenge=true
```

- `--providers.docker` tells traefik we have services running via docker
- `--entrypoints.web.address=:80`: Sets the port on the entrypoint named `web` to port 80
- `--entrypoints.websecure.address=:443` sets the port on the entrypoint named `websecure` to port 443
- `--entrypoints.mongodb.address=:27017` sets the port on the entrypoint named `mongodb` to port 27017
- `--certificatesresolvers.leresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory` sets the URL of the ca server to use for ACME certificates
- `--certificatesresolvers.leresolver.acme.email=<youremail@yourdomain.tld>` sets the email used when acquiring certificates
- `--certificatesresolvers.leresolver.acme.storage=/acme.json` tells traefik where to store certificates
- `--certificatesresolvers.leresolver.acme.tlschallenge=true` tells traefik what method to use to get certificates

!!! note
    Entrypoints are points where network traffic can enter traefik to be routed. The `web` entrypoint will be used for unencrypted HTTP traffic (we'll be setting it up to automatically redirect to HTTPS as well), the `websecure` entrypoint for encrypted HTTPS traffic, and the `mongodb` entrypoint for direct access to our database.

---

Next we need to open the ports that we defined in our traefik entrypoints:

``` yaml hl_lines="24 25 26 27"
# docker-compose.yml

version: '3.3'

services:
    reverse-proxy:
        image: traefik:v2.0
        container_name: reverse-proxy
        hostname: reverse-proxy
        restart: always

        command:
            - --providers.docker
            
            - --entrypoints.web.address=:80
            - --entrypoints.websecure.address=:443
            - --entrypoints.mongodb.address=:27017

            - --certificatesresolvers.leresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
            - --certificatesresolvers.leresolver.acme.email=<youremail@yourdomain.tld>
            - --certificatesresolvers.leresolver.acme.storage=/acme.json
            - --certificatesresolvers.leresolver.acme.tlschallenge=true

        ports:
            - '80:80'
            - '443:443'
            - '27017:27017'
```

Each port definition has two parts, separated by a `:` colon. The first part describes the port to map to on the host machine, while the second part is the port to bind to within the container. For example, `80:80` means that requests to port 80 on the host machine will route to port 80 within the container. In our case we are routing ports 80, 443, and 27017 on the host machine to those same ports within the traefik container.

!!! note
    Port definitions can also be more specific, including a specific interface they should listen to. For example, `127.0.0.1:80:80` would only map traffic on port 80 from localhost to the container.

---

The next step is to setup some volumes:

``` yaml hl_lines="29 30 31"
# docker-compose.yml

version: '3.3'

services:
    reverse-proxy:
        image: traefik:v2.0
        container_name: reverse-proxy
        hostname: reverse-proxy
        restart: always

        command:
            - --providers.docker
            
            - --entrypoints.web.address=:80
            - --entrypoints.websecure.address=:443
            - --entrypoints.mongodb.address=:27017

            - --certificatesresolvers.leresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
            - --certificatesresolvers.leresolver.acme.email=<youremail@yourdomain.tld>
            - --certificatesresolvers.leresolver.acme.storage=/acme.json
            - --certificatesresolvers.leresolver.acme.tlschallenge=true

        ports:
            - '80:80'
            - '443:443'
            - '27017:27017'

        volumes:
            - "./acme.json:/acme.json"
            - "/var/run/docker.sock:/var/run/docker.sock:ro"
```

Here we are defining two volumes for the container.

- `"./acme.json:/acme.json"` mounts our ACME certificate file to `/acme.json` in the container, which is where we've told traefik to look for it
- `"/var/run/docker.sock:/var/run/docker.sock:ro"` mounts our host machine's docker socket read-only inside the container

Mounting the docker socket in the container allows traefik to find out which containers it needs to connect to and what traffic to route to them.

---

Next we need to set some labels on the container, which will be used to configure some additional features:

``` yaml hl_lines="33 34 35 36 37 38 39 40"
# docker-compose.yml

version: '3.3'

services:
    reverse-proxy:
        image: traefik:v2.0
        container_name: reverse-proxy
        hostname: reverse-proxy
        restart: always

        command:
            - --providers.docker
            
            - --entrypoints.web.address=:80
            - --entrypoints.websecure.address=:443
            - --entrypoints.mongodb.address=:27017

            - --certificatesresolvers.leresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
            - --certificatesresolvers.leresolver.acme.email=<youremail@yourdomain.tld>
            - --certificatesresolvers.leresolver.acme.storage=/acme.json
            - --certificatesresolvers.leresolver.acme.tlschallenge=true

        ports:
            - '80:80'
            - '443:443'
            - '27017:27017'

        volumes:
            - "./acme.json:/acme.json"
            - "/var/run/docker.sock:/var/run/docker.sock:ro"

        labels:
            - traefik.http.middlewares.compress.compress=true

            - traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https

            - traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)
            - traefik.http.routers.redirs.entrypoints=web
            - traefik.http.routers.redirs.middlewares=redirect-to-https
```

- `traefik.http.middlewares.compress.compress=true` creates a new traefik middleware called *compress* that compresses traffic
- `traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https` creates a middleware that redirects traffic to HTTPS
- ``traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)`` creates a router that matches to all domains
- `traefik.http.routers.redirs.entrypoints=web` sets the router to listen on the HTTP *web* entrypoint
- `traefik.http.routers.redirs.middlewares=redirect-to-https` enables the HTTPS redirection middleware

Essentially, these labels enable compression on web traffic and automatically redirect any HTTP traffic to HTTPS.

---

Finally, we can add our docker network:

``` yaml hl_lines="42 43 44 45 46 47"
# docker-compose.yml

version: '3.3'

services:
    reverse-proxy:
        image: traefik:v2.0
        container_name: reverse-proxy
        hostname: reverse-proxy
        restart: always

        command:
            - --providers.docker
            
            - --entrypoints.web.address=:80
            - --entrypoints.websecure.address=:443
            - --entrypoints.mongodb.address=:27017

            - --certificatesresolvers.leresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory
            - --certificatesresolvers.leresolver.acme.email=<youremail@yourdomain.tld>
            - --certificatesresolvers.leresolver.acme.storage=/acme.json
            - --certificatesresolvers.leresolver.acme.tlschallenge=true

        ports:
            - '80:80'
            - '443:443'
            - '27017:27017'

        volumes:
            - "./acme.json:/acme.json"
            - "/var/run/docker.sock:/var/run/docker.sock:ro"

        labels:
            - traefik.http.middlewares.compress.compress=true

            - traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https

            - traefik.http.routers.redirs.rule=hostregexp(`{host:.+}`)
            - traefik.http.routers.redirs.entrypoints=web
            - traefik.http.routers.redirs.middlewares=redirect-to-https

        networks:
            - strangescout_main

networks:
    strangescout_main:
        external: true
```

There are two parts to adding the network. First, we tell our reverse-proxy service to attach to the `strangescout_main` network, then we define the network in the top level of the compose file.

### Starting the reverse proxy

To start the reverse proxy you only need to run one command:

``` shell
docker-compose up -d
```

As long as you're in the same directory as the `docker-compose.yml` file, it will automatically be read and the service will start up and fork to the background thanks to the `-d` flag!

## Setting up the database

After we've setup the reverse proxy, we'll need a database to store our scouting data. In the case of StrangeScout we use [MongoDB](https://www.mongodb.com/).

As with the reverse proxy, we'll once again create a directory to store the database files:

``` shell
mkdir -v database
cd database
```

---

We also need persistent storage for the database, so we'll create a directory for that as well:

``` shell
mkdir mongodb_data
```

### Creating the compose file

To start, we need to create our compose file:

``` shell
touch docker-compose.yml
```

---

The compose file for the database will start very similarly to that of the reverse proxy:

``` yaml
# docker-compose.yml

version: '3.3'

services:
    mongodb:
        image: mongo:4.2.2
        container_name: mongodb
        hostname: mongodb
        restart: always
```

---

Next, we need to mount our data directory into the container:

``` yaml hl_lines="12 13"
# docker-compose.yml

version: '3.3'

services:
    mongodb:
        image: mongo:4.2.2
        container_name: mongodb
        hostname: mongodb
        restart: always

        volumes:
            - "./mongodb_data:/data/db"
```

---

Next we need to setup the root database account:

``` yaml hl_lines="15 16 17"
# docker-compose.yml

version: '3.3'

services:
    mongodb:
        image: mongo:4.2.2
        container_name: mongodb
        hostname: mongodb
        restart: always

        volumes:
            - "./mongodb_data:/data/db"

        environment:
            MONGO_INITDB_ROOT_USERNAME: root
            MONGO_INITDB_ROOT_PASSWORD: example-password
```

- `MONGO_INITDB_ROOT_USERNAME: root` sets the database's root account username to `root`
- `MONGO_INITDB_ROOT_PASSWORD: example-password` sets the account password to `example-password`

By convention you'll probably want to leave the username as root. Make sure the password is something secure because this account will have full read/write access to the entire database!

---

Now we need to set some labels to tell our reverse proxy how to route database traffic:

``` yaml hl_lines="19 20 21"
# docker-compose.yml

version: '3.3'

services:
    mongodb:
        image: mongo:4.2.2
        container_name: mongodb
        hostname: mongodb
        restart: always

        volumes:
            - "./mongodb_data:/data/db"

        environment:
            MONGO_INITDB_ROOT_USERNAME: root
            MONGO_INITDB_ROOT_PASSWORD: example-password

        labels:
            - traefik.tcp.routers.mongodb.entrypoints=mongodb
            - traefik.tcp.routers.mongodb.rule=HostSNI(`*`)
```

- `traefik.tcp.routers.mongodb.entrypoints=mongodb` tells traefik to put this container on a TCP router with the `mongodb` entrypoint we setup that allows traffic on MongoDB's port 27017
- ``traefik.tcp.routers.mongodb.rule=HostSNI(`*`)`` tells traefik to match all domains, which is required for traefik TCP services

---

Finally, we attach the docker network to the container:

``` yaml hl_lines="23 24 25 26 27 28"
# docker-compose.yml

version: '3.3'

services:
    mongodb:
        image: mongo:4.2.2
        container_name: mongodb
        hostname: mongodb
        restart: always

        volumes:
            - "./mongodb_data:/data/db"

        environment:
            MONGO_INITDB_ROOT_USERNAME: root
            MONGO_INITDB_ROOT_PASSWORD: example-password

        labels:
            - traefik.tcp.routers.mongodb.entrypoints=mongodb
            - traefik.tcp.routers.mongodb.rule=HostSNI(`*`)

        networks:
            - strangescout_main

networks:
    strangescout_main:
        external: true
```

### Starting the database

The database will be started with the exact same command as the reverse proxy:

``` shell
docker-compose up -d
```

## Building StrangeScout

While we do provide prebuilt docker images, you may decide to build StrangeScout yourself. If you want to use prebuilt images you can skip this step.

### Cloning the repository

The first step to build any project is to clone its repository. StrangeScout has separate repositories for the frontend, backend, and deploy scripts. Since we're trying to build the final product, we want to clone the build repository. We also need to make sure we've cloned the submodules:

``` shell
git clone https://github.com/the-red-alliance/strangescout-deploy.git
cd strangescout-deploy
git submodule sync --recursive
git submodule update --init --force --recursive
```

### Building the image

Thanks to the premade Dockerfile, building the image is as simple as one command:

``` shell
docker image build -t strangescout .
```

Once the command completes, you'll have a docker image tagged `strangescout`!

## Setting up StrangeScout

Once again we'll start by creating a self contained directory:

``` shell
mkdir -v strangescout
cd strangescout
```

### Creating the compose file

The StrangeScout service itself doesn't require any persistent volumes or other local files, so all we have to do is create a compose file:

``` shell
touch docker-compose.yml
```

And start editing:

``` yaml
# docker-compose.yml

version: '3.3'

services:
    strangescout:
        image: theredalliance/strangescout:latest
        container_name: strangescout
        hostname: strangescout
        restart: always
```

We once again start the same way as the database and reverse proxy services. Here we're using the latest prebuilt image, but if you built the image yourself you should put whatever you tagged it as for the `image` key. A list of prebuilt image tags can be found here: [https://hub.docker.com/r/theredalliance/strangescout/tags](https://hub.docker.com/r/theredalliance/strangescout/tags)

---

Next we need to set some environment variables:

``` yaml hl_lines="12 13 14 15 16 17 18"
# docker-compose.yml

version: '3.3'

services:
    strangescout:
        image: theredalliance/strangescout:latest
        container_name: strangescout
        hostname: strangescout
        restart: always

        environment:
            - SECRET=jwt-secret
            - "DB_URL=mongodb://root:db-password@mongodb:27017/dbname?authSource=admin"
            - DB_DEBUG=true
            - ADMIN_EMAIL=email@yourdomain.tld
            - ADMIN_PASSWORD=supersecretpassword
            - TBA_KEY=xxx
```

- `SECRET=jwt-secret` sets the secret used to encrypt the tokens used for authenticating users. This should be a random string of your choice.
- `"DB_URL=mongodb://root:db-password@mongodb:27017/dbname?authSource=admin"` sets the URL of the database.
    - `root:db-password` corresponds to the root username and password we set up earlier.
    - `@mongodb:27017` says to go to port 27017 - the default MongoDB port, at hostname `mongodb`, which is what we set the database container hostname to.
    - `dbname` is the name of the database within Mongo that your data will be stored in, and can be set to anything you want following the rules set here: [https://docs.mongodb.com/manual/reference/limits/#restrictions-on-db-names](https://docs.mongodb.com/manual/reference/limits/#restrictions-on-db-names).
    - `?authSource=admin` says to use the `admin` database to find the user. This should not be changed.
- `DB_DEBUG` enables debug logs from the backend (omit completely to disable)
- `ADMIN_EMAIL` sets the email address of the default strangescout admin account
- `ADMIN_PASSWORD` sets the password for the default admin account
- `TBA_KEY` sets the API key for The Blue Alliance integration

---

Next we need to add a template file. StrangeScout uses a YAML file to define the game. Premade templates can be found under [Premade Game Templates](./premade-templates.md). To use your template, we need to mount it as a volume:

``` yaml hl_lines="20 21"
# docker-compose.yml

version: '3.3'

services:
    strangescout:
        image: theredalliance/strangescout:latest
        container_name: strangescout
        hostname: strangescout
        restart: always

        environment:
            - SECRET=jwt-secret
            - "DB_URL=mongodb://root:db-password@mongodb:27017/dbname?authSource=admin"
            - DB_DEBUG=true
            - ADMIN_EMAIL=email@yourdomain.tld
            - ADMIN_PASSWORD=supersecretpassword
            - TBA_KEY=xxx

        volumes:
            - ./template.yml:/template.yml
```

`./template.yml:/template.yml` tells docker to mount the file `./template.yml` at `/template.yml` inside the container. Change the first part of the mount to point to your template file.

---

Next we need to set some labels to tell Traefik how to route our container:

``` yaml hl_lines="23 24 25 26 27"
# docker-compose.yml

version: '3.3'

services:
    strangescout:
        image: theredalliance/strangescout:latest
        container_name: strangescout
        hostname: strangescout
        restart: always

        environment:
            - SECRET=jwt-secret
            - "DB_URL=mongodb://root:db-password@mongodb:27017/dbname?authSource=admin"
            - DB_DEBUG=true
            - ADMIN_EMAIL=email@yourdomain.tld
            - ADMIN_PASSWORD=supersecretpassword
            - TBA_KEY=xxx

        volumes:
            - ./template.yml:/template.yml

        labels:
            - traefik.http.routers.strangescout.rule=Host(`<yourdomain.tld>`)
            - traefik.http.routers.strangescout.entrypoints=websecure
            - traefik.http.routers.strangescout.tls.certresolver=leresolver
            - traefik.http.routers.strangescout.middlewares=compress
```

- ``traefik.http.routers.strangescout.rule=Host(`<yourdomain.tld>`)`` tells traefik what domain to route to StrangeScout. Change `<yourdomain.tld>` accordingly.
- `traefik.http.routers.strangescout.entrypoints=websecure` tells traefik to redirect traffic to HTTPS
- `traefik.http.routers.strangescout.tls.certresolver=leresolver` tells traefik to automatically get HTTPS certificates
- `traefik.http.routers.strangescout.middlewares=compress` tells traefik to compress traffic

---

The final step is to set the docker network:

``` yaml hl_lines="29 30 31 32 33 34"
# docker-compose.yml

version: '3.3'

services:
    strangescout:
        image: theredalliance/strangescout:latest
        container_name: strangescout
        hostname: strangescout
        restart: always

        environment:
            - SECRET=jwt-secret
            - "DB_URL=mongodb://root:db-password@mongodb:27017/dbname?authSource=admin"
            - DB_DEBUG=true
            - ADMIN_EMAIL=email@yourdomain.tld
            - ADMIN_PASSWORD=supersecretpassword
            - TBA_KEY=xxx

        volumes:
            - ./template.yml:/template.yml

        labels:
            - traefik.http.routers.strangescout.rule=Host(`<yourdomain.tld>`)
            - traefik.http.routers.strangescout.entrypoints=websecure
            - traefik.http.routers.strangescout.tls.certresolver=leresolver
            - traefik.http.routers.strangescout.middlewares=compress

        networks:
            - strangescout_main

networks:
    strangescout_main:
        external: true
```

## Starting StrangeScout

We can now start StrangeScout:

``` shell
docker-compose up -d
```

## Setting Up Webhooks

StrangeScout supports webhooks from The Blue Alliance API. So set them up, go to your api account on The Blue Alliance, and add a new webhook at url `https://<yourdomain.tld>/api/webhook`. Follow TBA's guide to set up the webhook. When you send the verification message, the code will be logged to the containers logs, along with the TBA_HMAC header. To finalize setup, add `TBA_HMAC` as an environment variable in the docker-compose file and set it to the logged TBA-HMAC header, and restart the container.