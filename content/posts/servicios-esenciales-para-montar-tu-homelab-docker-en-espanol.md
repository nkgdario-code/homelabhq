---
title: "Servicios esenciales para montar tu homelab Docker en español"
description: "Descubre los 5 servicios imprescindibles para tu homelab doméstico: desde NAS hasta VPN. Guía técnica con configuraciones paso a paso en Docker"
date: 2026-06-08
draft: false
tags: ["homelab", "Docker", "NAS", "Traefik"]
---

## **Los 5 servicios esenciales para montar tu homelab hoy mismo**

Si tienes un homelab en casa, sabes que mantener los servicios básicos funcionando es clave para experimentar sin dolores de cabeza. No se trata solo de instalar cosas al azar, sino de construir una base sólida con herramientas que cubran necesidades reales: almacenamiento, red, automatización y seguridad. Estos servicios esenciales forman el esqueleto de cualquier laboratorio doméstico funcional, ya sea para aprender, desarrollar proyectos o simplemente disfrutar de tecnología en tu propia red.

---

### **1. NAS doméstico: El corazón de tu almacenamiento**
Un sistema de archivos accesible desde cualquier dispositivo no es un lujo, es una necesidad. **Nextcloud** o **TrueNAS (CORE/SCALE)** son las dos caras de una misma moneda: por un lado, Nextcloud ofrece sincronización de archivos, calendario, contactos y hasta aplicaciones de oficina (como Collabora Online); por otro, TrueNAS brinda almacenamiento en ZFS (con snapshots y compresión), ideal para datos críticos o backups.

#### *Ejemplo práctico:*
```yaml
# Docker Compose para Nextcloud + base de datos MariaDB
version: '3.8'
services:
  nextcloud:
    image: nextcloud:latest
    ports:
      - "8080:80"
    volumes:
      - ./data:/var/www/html/data
      - ./config:/var/www/html/config
    depends_on:
      - mariadb
  mariadb:
    image: mariadb:10.6
    environment:
      MYSQL_ROOT_PASSWORD: tu_contraseña_secreta
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: otra_contraseña_segura
```
*Consejo:* Usa un disco dedicado o RAID-Z en TrueNAS para evitar perder datos. Si optas por Nextcloud, configura un almacenamiento externo (como un disco duro USB asignado por UUID) para liberar espacio en el sistema.

---

### **2. Gestor dinámico de dominios: Reverse proxy y TLS**
Navegar por tu homelab con `servicio.local` en lugar de `localhost:5678` no es solo comodidad, sino seguridad. **Traefik** o **NPM (Nginx Proxy Manager)** actúan como puertas de enlace inversas: redirigen tráfico HTTPS de forma automática, gestionan certificados SSL con Let’s Encrypt y simplifican el acceso a múltiples servicios sin abrir puertos directamente al router.

#### *Ejemplo práctico:*
```yaml
# Traefik + middleware para autenticación básica
services:
  traefik:
    image: traefik:v2.10
    command:
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.email=tu@email.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./letsencrypt:/letsencrypt
      - /var/run/docker.sock:/var/run/docker.sock:ro
```
*Detalle importante:* Traefik usa etiquetas Docker para configuración dinámica. Ejemplo en otro servicio:
```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.routers.nextcloud.rule=Host(`cloud.tudominio.com`)"
  - "traefik.http.routers.nextcloud.tls.certresolver=myresolver"
```

---

### **3. Base de datos unificada: PostgreSQL**
No subestimes el poder de una base de datos centralizada. **PostgreSQL** es la opción más equilibrada para homelabs: soporta geodatos, JSONB avanzado y tiene réplicas fáciles de configurar. Es la base de servicios como **Gitea**, **Wikijs** o **ownCloud**, y su rendimiento es notable incluso en hardware modesto.

#### *Ejemplo práctico:*
```bash
# Backup automático con cron + docker
0 2 * * * docker exec postgres_container pg_dump -U postgres -Fc postgres > /backups/postgres_$(date +%F).dump
docker exec postgres_container find /backups -type f -mtime +7 -delete
```
*Opciones:* Si prefieres algo más ligero, **MariaDB** es un buen sustituto para casos de uso tradicionales (como WordPress o Nextcloud).

---
### **4. Monitorización y logs: El ojo que todo lo ve**
Un homelab sin métricas es como navegar en la oscuridad. **Grafana** + **Prometheus** (o **Telegraf**) permiten visualizar el estado de tu hardware, servicios y tráfico en tiempo real. Incluye alarmas para CPU, RAM, disco y caída de servicios, evitando sorpresas.

#### *Ejemplo práctico:*
```yaml
# Grafana + Prometheus en Docker Compose
services:
  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - ./grafana-storage:/var/lib/grafana
```
*Configuración rápida:* Añade un datasource en Grafana apuntando a `http://prometheus:9090` y usa el dashboard **ID 1860** (node-exporter-full) para empezar.

---
### **5. VPN para acceso remoto seguro: WireGuard**
Acceder a tu homelab desde fuera de casa requiere cifrado. **WireGuard** es la opción más eficiente: velocidad de red irreal (cercana a 10 Gbps en hardware moderno) y mínima configuración. Combínalo con **Cloudflare Tunnel** para evitar abrir puertos en el router (si tienes IP dinámica).

#### *Ejemplo práctico:*
```bash
# Generar claves para el servidor (haz esto una sola vez)
umask 077
wg genkey | tee privatekey | wg pubkey > publickey

# Configurar Docker con WireGuard
docker run -d \
  --name=wg-easy \
  -e WG_HOST=tu.dominio.com \
  -e PASSWORD=contraseña_segura \
  -p 51820:51820/udp \
  -v ./wg-easy-config:/config \
  -v /lib/modules:/lib/modules \
  --cap-add=NET_ADMIN \
  --sysctl net.ipv4.ip_forward=1 \
  --sysctl net.ipv4.conf.all.src_valid_mark=1 \
  weejewel/wg-easy:latest
```
*Consejo:* Usa clientes como **WireGuard App** (iOS/Android) o **qBittorrent + WireGuard** para tráfico P2P seguro.

---

## **Resumen rápido: ¿Por dónde empezar?**
1. **Almacenamiento** → Nextcloud o TrueNAS.
2. **Red** → Traefik + certificado SSL automático.
3. **Datos** → PostgreSQL (o MariaDB).
4. **Monitorización** → Grafana + Prometheus.
5. **Acceso remoto** → WireGuard (o Cloudflare Tunnel).

*Extra:* Si tu homelab es nuevo, prioriza **TrueNAS** para datos y **WireGuard** para moverte a producción rápido. Para proyectos más específicos, añade servicios como **Gitea** (Git privado), **Home Assistant** (IoT) o **Jellyfin** (multimedia) sobre esta base.

---
