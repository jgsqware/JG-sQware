# Install Gogs

1. Create volume
`docker volume create --name gogs-data`

1. Pull image
`docker pull gogs/gogs`

1. Run with volume
`docker run --name=gogs -p 10022:22 -p 10080:3000 -v gogs-data:/data -d gogs/gogs`

## Settings

Most of settings are obvious and easy to understand, but there are some settings can be confusing by running Gogs inside Docker:

### Database Settings

- **Database type**: SQLite3
- **Path**: data/gogs.db

### General Settings

- **Repository Root Path**: keep it as default value `/home/git/gogs-repositories` because `start.sh` already made a symbolic link for you.
- **Run User**: keep it as default value `git` because `start.sh` already setup a user with name `git`.
- **Domain**: fill in with Docker container IP(e.g. `192.168.99.100`). But if you want to access your Gogs instance from a different physical machine, please fill in with the hostname or IP address of the Docker host machine.
- **SSH Port**: Use the exposed port from Docker container. For example, your SSH server listens on `22` inside Docker, but you expose it by `10022:22`, then use `10022` for this value. **Builtin SSH server is not recommended inside Docker Container**
- **HTTP Port**: Use port you want Gogs to listen on inside Docker container. For example, your Gogs listens on `3000` inside Docker, and you expose it by `10080:3000`, but you still use `3000` for this value.
- **Application URL**: Use combination of **Domain** and **exposed HTTP Port** values(e.g. `http://192.168.99.100:10080/`).

# Vulnerable container

```Dockerfile
FROM busybox:latest
MAINTAINER Julien Garcia Gonzalez <jgonzalez@wemanity.com>

ENTRYPOINT echo "Hello from Vulnerable Container!"
```
# CoreOS Clair

## Installing GOPATH

```bash
# Cannot use ~ metashell character
mkdir /home/juliengarcia/.go

# In .zshrc
export GOPATH="~/.go"
export PATH=$PATH:$GOPATH/bin
```

## Install Clair

### Install Go Tool to analyze image locally

```bash
go get -u github.com/coreos/clair/contrib/analyze-local-images
```

### Copy base configuration file locally

```bash
# In your Clair folder
curl -O https://raw.githubusercontent.com/coreos/clair/master/config.example.yaml
```

### Run Clair container

```bash
docker run -it -v /tmp:/tmp -p 6060:6060 -p 6061:6061 -v <DIR_WITH_CONFIG>:/config:ro quay.io/coreos/clair:latest --config=/config/config.example.yaml
```
