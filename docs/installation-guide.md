disqus:

## Before you install

The following installation guide will document setup with [Docker](https://www.docker.com/) because it's what we at *Team 1533 Triple Strange* and *The Red Alliance* use. It's also important to note that StrangeScout is designed to be run behind a reverse proxy, and as such does **not** have built-in support for HTTPS certificates. We do provide example configurations using the [traefik reverse proxy](https://containo.us/traefik/), but you are more than welcome to use your own reverse proxy, or even fork the code and roll in your own support for HTTPS.

### Dependencies

StrangeScout only has two true dependencies:

- [Docker](https://www.docker.com/)
- [Docker Compose](https://docs.docker.com/compose/)

*Docker* is the program that allows services to be run as individual *containers* on a host machine, and *Docker Compose* is a utility that simplifies choreography of multiple docker containers.

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

---

## Setting up the reverse proxy

Now that all dependencies have been installed we can begin setting up traefik for our reverse proxy.

### Creating a docker network

The first step to setting up traefik is to create a new docker network:

``` shell
docker network create strangescout_main
```

This will allow for containers on the same network to easily communicate with each other. In our case it allows for the StrangeScout server to communicate with the database in its own separate container.