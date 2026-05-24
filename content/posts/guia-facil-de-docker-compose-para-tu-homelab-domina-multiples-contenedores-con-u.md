---
title: "** Guía fácil de Docker Compose para tu homelab: domina múltiples contenedores con un solo archivo"
description: "** Aprende a gestionar servicios Docker como un pro con Docker Compose: instalación, configuración YAML, ejemplos prácticos y consejos para no liarla en tu homelab."
date: 2026-05-24
draft: false
tags: ["** Docker", "Compose", "homelab", "servidores caseros"]
---

## Docker Compose: La forma más fácil de gestionar contenedores Docker en tu homelab

Si tienes un homelab con varios servicios funcionando o planeas desplegar múltiples aplicaciones, gestionar cada contenedor individualmente con comandos `docker run` se convierte en una tarea tediosa y propensa a errores. Aquí es donde **Docker Compose** brilla: simplifica el proceso al permitirte definir y ejecutar múltiples contenedores a través de un solo archivo YAML, automatizando redes, volúmenes y configuraciones en segundos.

### **¿Qué es Docker Compose y por qué usarlo?**

Docker Compose es una herramienta que extiende Docker mediante archivos de configuración (*docker-compose.yml*) para definir aplicaciones multicontenedor. En lugar de ejecutar manualmente `docker run` para cada servicio, escribes una receta declarativa que describe:
- **Dependencias entre contenedores** (ej: un servicio web necesita una base de datos).
- **Redes compartidas** (para que los servicios se comuniquen).
- **Volúmenes persistentes** (para guardar datos sin perderlos al reiniciar).
- **Variables de entorno** (claves API, credenciales, etc.).
- **Políticas de reinicio** (reiniciar automáticamente si falla un contenedor).

**Ventajas clave:**
- **Reproducibilidad**: El mismo archivo funciona en cualquier máquina con Docker Compose instalado.
- **Gestión más limpia**: Un solo comando (`docker compose up`) pone en marcha todos los servicios.
- **Escalabilidad**: Puedes añadir más servicios sin reescribir configuraciones complejas.

---

### **Instalación rápida (Linux/macOS/Windows)**

Docker Compose suele venir incluido con la instalación de Docker Desktop para Windows y macOS. Para Linux (recomendado para homelabs), instala la versión independiente:

```bash
# Descargar la última versión estable (ejemplo para Debian/Ubuntu)
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Verifica que funcione:
```bash
docker-compose --version
# Debe mostrar algo como: Docker Compose version v2.XX.X
```

---
### **Primeros pasos: Crear un archivo `docker-compose.yml`**

Crea un archivo llamado `docker-compose.yml` en una carpeta vacía. Este será el plano maestro de tu declaración de servicios. Ejemplo básico con **Traefik** (proxy inverso) + **Portainer** (gestión visual de Docker) + **Watchtower** (actualización automática de imágenes):

```yaml
version: '3.8'  # Versión de sintaxis de Compose (recomendado 3.8 o superior)

services:
  # ---- Traefik (Proxy inverso) ----
  traefik:
    image: traefik:v2.10
    container_name: traefik
    restart: unless-stopped
    ports:
      - "80:80"   # HTTP
      - "443:443" # HTTPS
      - "8080:8080" # Dashboard (solo para administración)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro  # Permiso para ver contenedores
      - ./traefik.yml:/etc/traefik/traefik.yml:ro
      - ./certs:/certs  # Certificados SSL
    command:
      - "--api.insecure=true"  # Habilitar dashboard (solo para desarrollo)
      - "--providers.docker=true"  # Configurar automáticamente servicios Docker
    networks:
      - lab

  # ---- Portainer (Gestión visual) ----
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9443:9443"
      - "9000:9000"  # UI web
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./portainer_data:/data

  # ---- Watchtower (Actualizaciones automáticas) ----
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    command: --cleanup  # Elimina imágenes antiguas
    environments:
      - WATCHTOWER_ENABLE=true

# Redes personalizadas (opcional pero recomendado)
networks:
  lab:
    driver: bridge
```

---
### **Desplegar tu stack en 3 comandos mágicos**

1. **Iniciar todos los servicios** (en segundo plano):
   ```bash
   docker-compose up -d
   ```
   `-d` desatacha el proceso para que siga ejecutándose. Verás el estado de cada contenedor con:
   ```bash
   docker-compose ps
   ```

2. **Escala un servicio** (ej: 3 instancias de un mismo contenedor en un balanceador):
   ```bash
   docker-compose up -d --scale web=3
   ```

3. **Detener y limpiar** (elimina contenedores pero **no datos persistentes**):
   ```bash
   docker-compose down
   ```
   Si quieres eliminar **todo** (incluyendo volúmenes):
   ```bash
   docker-compose down -v
   ```

---
### **Casos prácticos para tu homelab**

#### **1. Media server con Plex + Jellyfin + Bazarr**
*(El clásico de los homelabs: centralizar películas, series y subtítulos)*

```yaml
version: '3.8'
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    restart: unless-stopped
    ports:
      - "32400:32400"
    volumes:
      - ./plex_config:/config
      - /media/:/data/media  # Ruta donde tengas tus archivos
    environment:
      - TZ=Europe/Madrid
      - PUID=1000
      - PGID=1000
    networks:
      - media

  jellyfin:
    image: lscr.io/linuxserver/jellyfin:latest
    container_name: jellyfin
    restart: unless-stopped
    ports:
      - "8096:8096"
    volumes:
      - ./jellyfin_config:/config
      - /media/:/data/media
    environment:
      - TZ=Europe/Madrid
    depends_on:
      - plex  # Espera a que Plex esté listo

networks:
  media:
    driver: bridge
```

#### **2. Base de datos + Adminer (phpLiteAdmin) desde cero**
*(Para aprender sin miedo a romper algo)*

```yaml
version: '3.8'
services:
  mariadb:
    image: mariadb:10.6
    container_name: mariadb
    restart: unless-stopped
    volumes:
      - ./mariadb_data:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=supersecreto
      - MYSQL_DATABASE=homelab
      - MYSQL_USER=admin
      - MYSQL_PASSWORD=adminpass
    networks:
      - db

  adminer:
    image: adminer:latest
    container_name: adminer
    restart: unless-stopped
    ports:
      - "8080:8080"
    networks:
      - db
    depends_on:
      - mariadb

networks:
  db:
    driver: bridge
```

---
### **Trucos para no fallar (y dormir tranquilo)**

1. **Nombres de servicios coherentes**: Usa nombres descriptivos (ej: `nextcloud_db` en lugar de `db1`).
2. **Volúmenes externos**: Mapea rutas locales **antes** de iniciar el stack para evitar errores de permisos.
   ```yaml
   volumes:
     - /ruta/local:/ruta/contenedor
   ```
3. **Dependencias (`depends_on`)**: Útil para ordenar lanzamientos, pero **no garantiza que el servicio esté listo** (usa *healthchecks* si es crítico).
4. **Variables de entorno**: Nunca hardcodees credenciales. Usa ficheros `.env`:
   ```yaml
   environment:
     POSTGRES_PASSWORD: ${DB_PASSWORD}
   ```
   Y crea un `.env` en la misma carpeta:
   ```
   DB_PASSWORD=micontrasena123!
   ```
5. **Si algo falla**: Ejecuta `docker-compose logs [servicio]` para depurar. Los errores suelen estar en las últimas 20 líneas.

---
### **¿Docker Compose o Docker Swarm?**

Docker Compose es **ideal para homelabs** (aprendizaje, proyectos pequeños o medianos). Si tu infraestructura crece:
- **Docker Swarm**: Para clusters de producción (solo si necesitas balanceo de carga real).
- **Kubernetes**: Para entornos empresariales (pero con una curva de aprendizaje empinada).

Compose cubre el 95% de los casos en un homelab. Domínalo primero y luego explora alternativas si es necesario.

---
**
**
**
