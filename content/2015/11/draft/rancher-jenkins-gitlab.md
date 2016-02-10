+++
Categories = []
Description = ""
Tags = []
date = "2015-11-24T15:11:16+01:00"
title = "rancher jenkins gitlab"
draft = true

+++

# Create Machine RancherOS
docker-machine create -d virtualbox --virtualbox-boot2docker-url https://github.com/rancher/os/releases/download/v0.4.1/rancheros.iso --engine-insecure-registry=192.168.99.113:5000  node-02

# Run Agent
docker run -d --privileged -v /var/run/docker.sock:/var/run/docker.sock rancher/agent http://192.168.99.109:8080/v1/scripts/1BEF035774A57A80B8D7:1448373600000:k8MZKOxFjSKhbGo6rJYq8h8Wsw

# Download locally the agent
docker pull 192.168.99.120:5000/rancher/agent:v0.8.2 &&\
  docker tag 192.168.99.120:5000/rancher/agent:v0.8.2 rancher/agent:v0.8.2 &&\
  docker pull 192.168.99.120:5000/rancher/agent-instance:v0.5.0 &&\
  docker tag 192.168.99.120:5000/rancher/agent-instance:v0.5.0 rancher/agent-instance:v0.5.0
