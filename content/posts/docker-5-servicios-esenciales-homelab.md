---
title: "Docker para tu homelab: los 5 servicios esenciales"
description: "Los 5 servicios que deberías instalar primero en tu servidor con Docker. Fáciles, útiles y gratuitos."
date: 2026-05-14
draft: false
tags: ["docker", "homelab", "servicios", "self-hosting"]
---

# Docker para tu homelab: 5 servicios esenciales

Una vez tienes Docker instalado, estos 5 servicios son los que más partido le sacarás.

## 1. Portainer — Gestión visual de Docker

``bash
docker run -d -p 9443:9443 --name portainer \
  -v /var/run/docker.sock:/var/run/docker.sock \
  portainer/portainer-ce
``
Gestiona contenedores, imágenes y volúmenes desde el navegador.

## 2. Uptime Kuma — Monitorización

``bash
docker run -d -p 3001:3001 --name uptime-kuma \
  -v uptime-kuma:/app/data \
  louislam/uptime-kuma
``
Monitoriza tus servicios y te avisa si algo cae.

## 3. Nginx Proxy Manager — Proxy inverso fácil

``bash
docker run -d -p 80:80 -p 443:443 -p 81:81 --name npm \
  -v npm-data:/data \
  jc21/nginx-proxy-manager
``
Accede a tus servicios con dominios bonitos (tuservicio.tudominio.com) y SSL automático.

## 4. Home Assistant — Domótica

``bash
docker run -d -p 8123:8123 --name home-assistant \
  -v ha-config:/config \
  ghcr.io/home-assistant/home-assistant:stable
``
Controla luces, sensores, temperatura y más desde un solo panel.

## 5. AdGuard Home — Bloqueador de anuncios en red

``bash
docker run -d -p 53:53/tcp -p 53:53/udp -p 80:80 --name adguard \
  -v adguard-work:/opt/adguardhome/work \
  adguard/adguardhome
``
Bloquea anuncios y rastreadores en TODOS los dispositivos de tu casa.
