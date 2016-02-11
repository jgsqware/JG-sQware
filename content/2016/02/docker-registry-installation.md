+++
date = "2016-02-10T11:00:00+01:00"
title = "Docker Registry - Docker Hub, at Home, for Free!"
hashtags = ["docker"]
+++



Docker containers is everywhere. You can found it on [Docker Hub](https://hub.docker.com), [Quay.io](https://quay.io),...
You can find easily a working container for your purpose (eg. database, code analysis, compilation). For development, it's perfect.

Now, you'll love to use it as far is possible in your deployment chain. And probably your company, as my client, doesn't want to have is specific container (with codebase, data,...) exposed and public in the cloud.
For sure, you can use private repositories to upload your images, but
  1. It's **in the cloud**
  2. It's **not free**

My client not use Cloud at all. He always look like for On-Premise.
Docker offers a complete solution for that: [Docker Datacenter](https://hub.docker.com/enterprise/trial)

So, we are one step further:
1. ~~It's in the cloud~~
2. It's not free

We will focused now on the second point:

### *How can I have a docker authenticated registry On-Premise, for free?*

Let's first take a look on [Docker Registry](https://docs.docker.com/registry/) page:

> You should use the Registry if you want to:

>  - tightly control where your images are being stored
>  - fully own your images distribution pipeline
>  - integrate image storage and distribution tightly into your in-house development workflow

So now, how can we deploy our registry, how can I connect to it and manage my images.

## Registry, Authentication: what's behind

Again, I will take the info from [Docker Registry Token Authentication](https://docs.docker.com/registry/spec/auth/token/)

![Docker Registry Token Authentication](https://docs.google.com/drawings/d/1EHZU9uBLmcH0kytDClBv6jv6WR4xZjE8RKEUw1mARJA/pub?w=480&h=360)

Basically, we will deploy a Token Based Authentication server *"Authorization service"* then configure the registry to connect to it for authentication.
I will use the [docker-compose v2](https://docs.docker.com/compose/) for running multiple containers.

The final version is on the github: [jgsqware/authenticated-registry](http://www.github.com/jgsqware/authenticated-registry)

## Token Based Authentication server and Registry configuration

### Token Based Authentication server

I will use [cesanta/docker_auth/](https://hub.docker.com/r/cesanta/docker_auth/), a go implementation of authentication server.

Supported authentication methods:

- Static list of users
- Google Sign-In (incl. Google for Work / GApps for domain) (documented here)
- LDAP bind
- MongoDB user collection

The following is based on Static list of User

First, I create the following directory structure

```bash
authenticated-registry/
├ docker-compose.yml
├── config
│   └── auth_config.yml # contain your static list of users
└── ssl
    ├── server.key # For this purpose ,
    └── server.pem # we will use self-signed certificate
```

The certificate is a self signed certificate:

```bash
$ cd authenticated-registry/ssl
$ sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout server.key -out server.pem
# I use auth as Common Name
# auth is the URI of my authentication server
```

Edit the `authenticated-registry/docker-compose.yml`:

```yaml
version: '2'

services:
  auth:
    image: cesanta/docker_auth
    ports:
      - "5001:5001"
    volumes:
      - ./config:/config:ro
      - ./ssl:/ssl
    command: /config/auth_config.yml
    container_name: "auth"
```

Edit the `authenticated-registry/config/auth_config.yml`:

```yaml
server:  # Server settings.
  # Address to listen on.
  addr: ":5001"
  # TLS certificate and key.
  certificate: "/ssl/server.pem"
  key: "/ssl/server.key"

token:  # Settings for the tokens.
  issuer: "auth_service"  # Must match issuer in the Registry config.
  expiration: 900


# Static user map.
users:
  # Password is specified as a BCrypt hash. Use htpasswd -B to generate.
  "admin":
    password: "$2y$05$C3oDhd2O3nvcacmvGxojN.MPPvcV7LApYQU3meFMU5GeC27kb.0sK"
  "jgsqware": # my user
    password: "$2y$05$oGKwJ8QJDLBOoTBmC/EQiefIMV1N9Yt9jpX3SqMoRqZRRql6q7yam"

acl:
  # Admin has full access to everything.
  - match: {account: "admin"}
    actions: ["*"]
  # Users have full right on their repository
  - match: {account: "/.+/", name: "${account}/*"}
    actions: ["*"]
  # Access is denied by default.
```
> For more information about the configuration options for this authentication server, refer to the [Github repo](https://github.com/cesanta/docker_auth).

To generate the password with htpasswd with BCrypt hash:

```
$ htpasswd -nbB jgsqware jgsqware
jgsqware:$2y$05$oGKwJ8QJDLBOoTBmC/EQiefIMV1N9Yt9jpX3SqMoRqZRRql6q7yam
```

### Registry

Docker offers his registry as a docker image. Nice guy!

I based this setup on *registry 2.1.1*.

> Update: Docker released the registry **2.3**. It change some Manifest info and how the image is stored.

The configuration of the registry is done by **environment variable**.

Edit the `authenticated-registry/docker-compose.yml`:

```yaml
version: '2'

services:
  auth:
    image: cesanta/docker_auth
    ports:
      - "5001:5001"
    volumes:
      - ./config:/config:ro
      - ./ssl:/ssl
    command: /config/auth_config.yml
    container_name: "auth"

  registry:
    image: registry:2.2.1
    ports:
      - 5000:5000
    volumes:
      - ./ssl:/ssl
    container_name: "registry"
    environment:
      - REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/var/lib/registry
      - REGISTRY_AUTH=token
      - REGISTRY_AUTH_TOKEN_REALM=https://auth:5001/auth # the authentication server URI
      - REGISTRY_AUTH_TOKEN_SERVICE="registry"
      - REGISTRY_AUTH_TOKEN_ISSUER="auth_service" # Should be the same as token.issuer from authenticated-registry/config/auth_config.yml
      - REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE=/ssl/server.pem
```

### Registry data persistence

I will use a [Docker Volume](http://docs.docker.com/engine/userguide/dockervolumes/) to persist registry data.

Edit the `authenticated-registry/docker-compose.yml`:

```yaml
version: '2'

services:
  auth:
    image: cesanta/docker_auth
    ports:
      - "5001:5001"
    volumes:
      - ./config:/config:ro
      - ./ssl:/ssl
    command: /config/auth_config.yml
    container_name: "auth"

  registry:
    image: registry:2.2.1
    ports:
      - 5000:5000
    volumes:
      - ./ssl:/ssl
      - registry-data:/var/lib/registry
    container_name: "registry"
    environment:
      - REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY=/var/lib/registry
      - REGISTRY_AUTH=token
      - REGISTRY_AUTH_TOKEN_REALM=https://auth:5001/auth # the authentication server URI
      - REGISTRY_AUTH_TOKEN_SERVICE="registry"
      - REGISTRY_AUTH_TOKEN_ISSUER="auth_service" # Should be the same as token.issuer from authenticated-registry/config/auth_config.yml
      - REGISTRY_AUTH_TOKEN_ROOTCERTBUNDLE=/ssl/server.pem

volumes:
  registry-data: # the real name will be <parent-folder-name>_registry-data (dash '-' int he folder name will be removed). eg. authenticatedregistry_registry-data
    driver: local
```

## Let's Run it!

To start the authenticated registry, simply run

```bash
# Run docker-compose up -d to start all container
# -d is for running container as daemon

$ docker-compose up -d
Creating network "authenticatedregistry_default" with the default driver
Creating registry
Creating auth

```

The running container can be seen by:

```bash
$ docker ps
CONTAINER ID        IMAGE                 COMMAND                  CREATED             STATUS              PORTS                    NAMES
6f706e4684fb        cesanta/docker_auth   "/auth_server /config"   47 minutes ago      Up 47 minutes       0.0.0.0:5001->5001/tcp   auth
073e1d94e452        registry:2.2.1        "/bin/registry /etc/d"   47 minutes ago      Up 47 minutes       0.0.0.0:5000->5000/tcp   registry
```

The created data volume cn be seen by:

```bash
$ docker volume ls
DRIVER              VOLUME NAME
local               authenticatedregistry_registry-data
```


Your own docker registry is now available:
- you can **login**,**pull**, **push** repository on localhost:5000

```bash
$ docker login localhost:5000
Username: jgsqware
Password:
Email: jgsqware@wemanity.com
Login Succeeded
```
```bash
$ docker tag jgsqware/ubuntu-git localhost:5000/jgsqware/ubuntu-git
$ docker push localhost:5000/jgsqware/ubuntu-git
The push refers to a repository [localhost:5000/jgsqware/ubuntu-git]
3ae0de97330a: Pushing [==========================>                        ]   163 MB/304.5 MB
5f70bf18a086: Pushed
dfd83ed44976: Pushed
217e6c75dcfc: Pushed
c0b5f9221fac: Pushing [==============================================>    ] 173.4 MB/187.7 MB

```

## Backup & Restore

### Backup registry data

As we have create a [Docker Volume](http://docs.docker.com/engine/userguide/dockervolumes/), we can backup the folder where the registry save its data to a tar file easily.

We will
  1. run a new container and mount the *registry-data* volume
  2. mount the **local folder** to `/backup` in the container.
  3. we will compress the registry data to a tarball.
  4. copy the tarball to the `/backup` folder to got it on our **local folder**

I use the jgsqware/registry-backup image. This image to the tarball and move it to backup

the Dockerfile is simple:

```
FROM alpine
MAINTAINER jgsqware <jgonzalez@wemanity.com>
WORKDIR /var/lib/registry

ENTRYPOINT ["tar","cvf", "/backup/registry-data.tar","."]
```

```bash
$ docker run -v authenticatedregistry_registry-data:/var/lib/registry -v $(pwd):/backup jgsqware/registry-backup
./
./docker/
./docker/registry/
./docker/registry/v2/
./docker/registry/v2/repositories/
./docker/registry/v2/repositories/jgsqware/
./docker/registry/v2/repositories/jgsqware/ubuntu-git/
...
```

You have now a file `registry-data.tar` in your local folder.

### Restore registry data

You can now restore the previously backuped registry data to a new registry data container.

  1. Run the docker-compose, you will have a registry data volume but it will be empty.
  2. Uncompress the `registry-data.tar` in the corresponding directory in the registry data.
    1. Run a new container and mount the *registry-data* volume
    2. Mount the **local folder** to `/backup` in the container.
    3. Uncompress the tarball to `/var/lib/registry`.

I use the jgsqware/registry-backup image. This image to the tarball and move it to backup

the Dockerfile is simple:

```
FROM alpine
MAINTAINER jgsqware <jgonzalez@wemanity.com>
WORKDIR /var/lib/registry

ENTRYPOINT ["tar","xvf", "/backup/registry-data.tar","."]
```

```bash
# The file need to be named 'registry-data.tar'
$ docker run -v authenticatedregistry_registry-data:/var/lib/registry -v $(pwd):/backup jgsqware/registry-restore
./
./docker/
./docker/registry/
./docker/registry/v2/
./docker/registry/v2/repositories/
./docker/registry/v2/repositories/jgsqware/
./docker/registry/v2/repositories/jgsqware/ubuntu-git/
./docker/registry/v2/repositories/jgsqware/ubuntu-git/_manifests/
./docker/registry/v2/repositories/jgsqware/ubuntu-git/_manifests/tags/
./docker/registry/v2/repositories/jgsqware/ubuntu-git/_manifests/tags/latest/
./docker/registry/v2/repositories/jgsqware/ubuntu-git/_manifests/tags/latest/index/
./docker/registry/v2/repositories/jgsqware/ubuntu-git/_manifests/tags/latest/index/sha256/
...
```

## Using the Docker Toolbox for OSX or Windows

Because Docker need to run on a real Linux Kernel (containerization tools exist only on Linux (for now)), you can install Docker on Windows and OSX with Docker Toolbox.

Docker Toolbox will install the client tools (docker-client, docker-compose, docker-machine) and virtualbox with boot2docker image.

boot2docker is a tiny linux os, on which docker is runnable.
Docker Toolbox will configure your docker client to connect to connect to the docker-machine (here is boot2docker).

### Docker Insecure Registry

To be able to use the registry in boot2docker, and because the registry is not secure by certificate (*it will be in another story*),
we need to tell docker to allow communication with this insecure registry.

It can be done by adding `--insecure-registry=<IP-OF-REGISTRY>:<PORT-OF-REGISTRY>` in the boot2docker profile.

So,

```bash
# SSH your docker-machine
# docker-machine ssh <DOCKER-MACHINE-NAME>
$ docker-machine ssh default

# Open in vi the boot2docker profile.
# profile is the configuration file used by docker when it start in boot2docker
$ sudo vi /var/lib/boot2docker/profile

# Add line '--insecure-registry <docker-machine-ip>:5000' in **EXTRA_ARGS**
EXTRA_ARGS='
--label provider=virtualbox
--insecure-registry 192.168.99.100:5000
'
# quit vi with :wq to save file

# Restart docker
$ sudo /etc/init.d/docker restart
# Quit docker machine by CTRL-D
```

### Open Registry port on boot2docker

You need to open the vm port5000 to put your registry on the network.

```bash
# use VBoxManage controlvm <VM-NAME> natpf1 <RULE-NAME>,tcp,<INTERFACE IP>,<VM-PORT-TO-OPEN>,,<DOCKER-PORT-TO-MAP>
# Here, we are creating a rule registry in vm registry, mapping the docker port 5000 to the vm port 5000 on every interface

$ VBoxManage controlvm registry natpf1 registry,tcp,0.0.0.0,5000,,5000
```

Now, you access over the network with:
- repository: on `<IP-OF-OSX/WINDOWS-HOST>:5000`


#### Errors... What?

- If you try to access your insecure registry without define it as insecure-registry, you will received the following error when you will try to *push/pull*

  ```bash
  $ docker pull 192.168.99.100:5000/jgsqware/ubuntu-git
       unable to ping registry endpoint https://192.168.99.101:5000/v0/
         v2 ping attempt failed with error: Get https://192.168.99.101:5000/v2/: tls: oversized record received with length 20527
           v1 ping attempt failed with error: Get https://192.168.99.101:5000/v1/_ping: tls: oversized record received with length 20527
  ```

- Sometimes, docker run out of his mind when restarting
  You can see it in log, with last line as
  ```bash
  # In your docker machine. Remember, SSH things.
  $ tail -f /var/log/docker.log
  level=fatal msg="Shutting down due to ServeAPI error: listen tcp 0.0.0.0:2376: bind: address already in use"
  ```

  If you see that, just kill docker and start it again
  ```bash
  $ sudo killall docker
  $ sudo /etc/init.d/docker start
  ```

  Now, it should be ok

  ```bash
  $ tail -f /var/log/docker.log
  level=info msg="Daemon has completed initialization"
  level=info msg="Docker daemon" commit=f4bf5c7 execdriver=native-0.2 graphdriver=aufs version=1.8.3
  ```

## What's next?

Now, you have your Authenticated Registry on-Premise. You can save your docker images.

The next step would be:
  1. Having a GUI for your registry
  2. Integrate the image build and deploy phase in CI
  3. Track vulnerabilities in your images


###### This post is inspired by this article: [Creating Private Docker Registry 2.0 with Token Authentication Service](https://the.binbashtheory.com/creating-private-docker-registry-2-0-with-token-authentication-service/)
