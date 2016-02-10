+++
date = "2016-01-14T08:00:00+01:00"
title = "Docker Pro Tips"
+++

Docker Daemon
----------------------

## 1. Update Docker Daemon options

###  Debian

  Docker daemon is run via `systemd`

  Add `/etc/default/docker` in systemd configuration file:

  ```toml
  # In file /lib/systemd/system/docker.service
  [Unit]
  Description=Docker Application Container Engine
  Documentation=https://docs.docker.com
  After=network.target docker.socket
  Requires=docker.socket

  [Service]
  EnvironmentFile=-/etc/default/docker
  Type=notify
  ExecStart=/usr/bin/docker daemon -H fd://
  MountFlags=slave
  LimitNOFILE=1048576
  LimitNPROC=1048576
  LimitCORE=infinity

  [Install]
  WantedBy=multi-user.target
  ```
