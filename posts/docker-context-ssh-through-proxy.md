---
title: Docker Remote Context via SSH over Proxy
published: true
description: how to connect to remote docker host via ssh over proxy
tags: 'docker,devops,ssh,linux'
id: 3123608
date: '2025-12-23T22:26:32Z'
---

Securely manage a remote Docker daemon over SSH using Docker contexts, with SSH traffic routed through a SOCKS5 proxy.

If you want to connect to and manage a remote Docker daemon without exposing the Docker API over the network, you can use Docker contexts over SSH. In environments where direct access to the remote server is restricted or must pass through a controlled network path, the SSH connection can be routed through a SOCKS5 proxy.

By configuring SSH to use a proxy and defining a Docker context that targets the remote host via SSH, Docker commands executed locally are transparently forwarded to the remote Docker daemon. This allows you to use standard Docker workflows (`docker ps`, `docker compose`, `docker build`) as if the daemon were local, while still benefiting from SSH key authentication, proxy enforcement, and connection `keepalive` mechanisms.

### SSH Configuration

To connect to a remote server via SSH through a proxy, `ProxyCommand` is used together with `nc` (`netcat`). All SSH connections to the remote host are routed through the specified SOCKS5 (it could be HTTP) proxy.

The wildcard host configuration (`vps-*`) ensures that any matching host uses the proxy and `keepalive` settings automatically.

`~/.ssh/config`

```config
HOST vps-*
  ServerAliveInterval   10
  ProxyCommand nc --proxy-type socks5 --proxy 127.0.0.1:10808 %h %p

Host vps-docker
  HostName <HOST_IP>
  User <USER_NAME>
  Port <HOST_SSH_PORT>
  PreferredAuthentications publickey

```

- `ServerAliveInterval`: keeps the SSH connection alive by sending periodic packets, preventing timeouts during long-running operations.
- `ProxyCommand`: forces SSH traffic to pass through a local SOCKS5 proxy (e.g. a VPN tunnel).
- `PreferredAuthentications publickey`: Specifies that SSH should prefer (and effectively enforce) public key–based authentication instead of password-based authentication when connecting to the host.

### Docker Context

Docker can connect to the remote Docker daemon over SSH.</br>
The SSH connection itself is routed through the configured SOCKS5 proxy , ensuring all Docker traffic to the remote host passes through the proxy.

To define a Docker context:

```bash
docker context create vps-docker-host --docker host=ssh://<vps-docker>
```

To switch and use created context:

```bash
docker context use vps-docker-host
```

Once this context is created and activated, all Docker CLI commands (including docker ps, docker compose, and docker build) will operate against the remote Docker daemon instead of the local one.

## Resources

- [Docker Context][docker-ctx-docs]
- [SSH Config `~/.ssh/config`][ssh-config]

[docker-ctx-docs]:https://docs.docker.com/engine/manage-resources/contexts
[ssh-config]:https://man7.org/linux/man-pages/man5/ssh_config.5.html
