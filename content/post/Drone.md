---
title: "Usar Drone CI/CD con Traefik en NixOS"
date: 2019-07-17T00:19:14+02:00
draft: false
tags: [Git,Github]
---

En este post vamos a ver como configurar nuestro propio entorno de CI/CD con Drone, integrado con Traefik como reverse proxy y en NixOS como sistema operativo usando repositorios de github.

![](https://avatars2.githubusercontent.com/u/2181346?s=200&v=4)

En primer lugar necesitamos crar una OAuth application en github y nuestro RPC secret podemos verlo en este [post](https://rubynor.com/blog/2020/06/setting-up-drone-ci-for-rails-apps/) o también en los [docs de drone](https://docs.drone.io/server/provider/github/).

Para crear nuestro RPC secret lo haremos de la siguiente forma:

> $ openssl rand -hex 16
bea26a2221fd8090ea38720fc445eca6

Una vez tenemos nuestro client ID y nuestro client secret procedemos a crear nuestro docker-compose.yml para levantar el container de drone.

```
version: '3.7'

services:
  drone-server:
    container_name: drone-server
    image: drone/drone:1
    restart: unless-stopped
    volumes:
      - ./drone:/data
    environment:
      - DRONE_GITHUB_CLIENT_ID=<Your_Client_ID>
      - DRONE_GITHUB_CLIENT_SECRET=<Your_Client_secret>
      - DRONE_RPC_SECRET=<Your_RPC_secret>
      - DRONE_SERVER_HOST=<Your_URL>
      - DRONE_SERVER_PROTO=https
      - DRONE_USER_CREATE=username:<Your_Github_username>,admin:true
    labels:
      - traefik.enable=true
      - traefik.http.routers.drone.entryPoints=web-secure
      - traefik.http.routers.drone.rule=Host(`Your_URL`)
      - traefik.http.routers.drone.tls.certresolver=default

```
A continuación levantamos nuestro container con:
> $ docker-compose up -d

Una vez levantado nuestro server vamos a levantar nuestro host runner executor para ello nos bajamos el binario de drone-exec desde la web de drone:

> $ curl -L https://github.com/drone-runners/drone-runner-exec/releases/latest/download/drone_runner_exec_linux_amd64.tar.gz | tar zx

Una vez bajado lo tenemos que mover a /root/bin/ para posteriormente en el configuration.nix añadir lo siguiente:

```
environment.etc.drone-runner-exec = {
  target = "drone-runner-exec/config";
  text = ''
  DRONE_RPC_PROTO=https
  DRONE_RPC_HOST=<Your_URL>
  DRONE_RPC_SECRET=<Your_RPC_secret>
  DRONE_UI_USERNAME=root
  DRONE_UI_PASSWORD=root
  '';
};

systemd.services.drone-runner-exec = {
  description = "Drone Exec Runner";
  startLimitIntervalSec = 5;
  serviceConfig = {
    ExecStart = "/root/bin/drone-runner-exec service run --config /etc/drone-runner-exec/config";
  };
  wantedBy = [ "multi-user.target" ];
  path = [ pkgs.git pkgs.docker pkgs.docker-compose ];
};
```
Una vez modificado nuestro configuration.nix:

> nixos-rebuild switch

Ahora accedemos a nuestra URL de Drone y nos autenticamos con nuestra cuenta de  Github.





