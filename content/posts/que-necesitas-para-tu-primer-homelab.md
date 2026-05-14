---
title: "Qué necesitas para montar tu primer homelab (guía 2026)"
description: "Todo lo que necesitas para montar tu primer servidor en casa: hardware, software y primeros pasos. Guía para principiantes."
date: 2026-05-14
draft: false
tags: ["homelab", "principiantes", "hardware", "docker"]
---

# Qué necesitas para montar tu primer homelab

Montar un homelab es más fácil de lo que crees. Aquí tienes lo mínimo para empezar.

## Hardware mínimo

| Componente | Mínimo | Recomendado |
|---|---|---|
| CPU | 2 núcleos | 4+ núcleos |
| RAM | 4 GB | 8-16 GB |
| Disco | 120 GB SSD | 500 GB+ |
| PC cualquier viejo | ✅ | ✅ |

No necesitas un servidor caro. Un PC viejo, un Raspberry Pi 4/5 o un mini PC de segunda mano (50-100€) son suficientes para empezar.

## Software base

1. **Linux** (Ubuntu Server o Debian) — el sistema operativo
2. **Docker** — para instalar servicios en segundos
3. **SSH** — para conectarte desde cualquier ordenador

## Primeros servicios para instalar

``bash
# Con Docker, en 1 línea cada uno:
docker run -d -p 80:80 nginx                    # Web server
docker run -d -p 8080:8080 portainer/portainer   # Gestión visual
docker run -d -p 3001:3001 louislam/uptime-kuma  # Monitorización
``

**Tip**: empieza con Portainer para gestionar Docker desde el navegador. No necesitas terminal para lo básico.
