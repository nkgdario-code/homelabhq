---
title: "Automatizar backups"
description: "Automatizar backups"
date: 2026-05-18
draft: false
tags: ["montar servidor NAS con Docker"]
---

## ¿Por qué automatizar backups en tu homelab?

Un homelab puede albergar desde bases de datos críticas hasta colecciones personales de fotos o proyectos en desarrollo. Sin un sistema de backups fiable, un fallo de disco, un error humano o un cambio de configuración no planificado pueden borrar semanas (o años) de trabajo. Docker simplifica la creación de soluciones de respaldo automatizadas gracias a su integración con herramientas nativas del sistema y soluciones de terceros. Este artículo explora cómo implementarlas sin complicaciones.

---

## Primeros pasos: ¿qué necesitas respaldar?

Antes de automatizar, identifica qué datos requieren protección:
- **Contenedores Docker**: Sus volúmenes (donde suelen residir datos persistentes como bases de datos —MariaDB, PostgreSQL—, archivos estáticos para Nextcloud, o configuraciones de servicios como Portainer).
- **Bases de datos fuera de Docker**: Como un Redis en localhost o un MariaDB instalado directamente en el host.
- **Configuraciones externas**: Archivos `/etc`, variables de entorno (`.env`) o scripts de configuración.
- **Metadatos de Docker**: Listados de contenedores, redes y volúmenes creados (`docker compose ls`, `docker volume ls`).

**Ejemplo práctico**:
```bash
# Lista recursos críticos en un homelab típico
docker ps --format '{{.Names}}' | grep -E 'postgres|nextcloud|portainer'
```
Reúne estos elementos en una lista para priorizar los backups.

---

## Herramientas para automatizar: los clásicos

### 1. `rsync` + `cron` (solución ligera)
Ideal para respaldar volúmenes Docker o directorios locales a un NAS o servidor secundario. Combínalo con `rsync` para copia incremental y `cron` para programar ejecuciones.

**Ejemplo con `docker cp` y `rsync`**:
```bash
#!/bin/bash
# backup_volumenes.sh

ORIGEN="/var/lib/docker/volumes/"
DESTINO="/mnt/backup/volumenes_docker/"
FECHA=$(date +%Y%m%d_%H%M)

# Copia incremental con rsync (conserva atributos y eliminaciones)
rsync -a --delete --stats --log-file=$DESTINO/rsync_$FECHA.log $ORIGEN $DESTINO$FECHA/

# Opción para datos críticos en contenedores en ejecución
docker exec postgres_db pg_dumpall -U admin > $DESTINO/pg_dumpall_$FECHA.sql
```
Programa su ejecución con `cron`:
```bash
# Ejecuta cada domingo a las 3 AM
0 3 * * 0 /home/user/scripts/backup_volumenes.sh >> /var/log/backups_docker.log
```

**Ventajas**: Sin dependencias adicionales. **Limitaciones**: No gestiona versiones ni cifrado.

---

### 2. `restic` (copias de seguridad modernas y versátiles)
[`restic`](https://restic.net/) es una herramienta de backup opensource con cifrado integrado, deduplicación y soporte para Amazon S3, Backblaze B2 o un servidor SFTP local. Su arquitectura es idónea para homelabs con conectividad limitada.

**Configuración básica con Backblaze B2**:
1. Instala `restic`:
   ```bash
   sudo apt install restic
   ```
2. Inicia un repositorio en Backblaze (ajusta los valores):
   ```bash
   export B2_ACCOUNT_ID="tu_id"
   export B2_ACCOUNT_KEY="tu_clave"
   restic -r b2:restic-homelab:/ init
   ```
3. Crea un script para respaldar volúmenes Docker y PostgreSQL:
   ```bash
   #!/bin/bash
   # backup_homelab.sh

   FECHA=$(date +%Y%m%d_%H%M)
   RESTIC_CACHE="/dev/shm/restic_cache"  # Mejor rendimiento en disco tmpfs

   # Backup de volúmenes activos (ej: nextcloud_data)
   restic -r b2:restic-homelab:/ volume nextcloud_data

   # Backup de bases de datos en ejecución
   docker exec postgres_db pg_dumpall -U admin | \
       restic -r b2:restic-homelab:/ pipe -r backup_sql/$FECHA.sql
   restic -r b2:restic-homelab:/ forget --keep-daily 30 --prune
   ```
4. Programa su ejecución:
   ```bash
   0 4 * * * /home/user/scripts/backup_homelab.sh > /var/log/restic_backup.log
   ```

**Ventajas**: Cifrado, deduplicación, multiprotocolo. **Limitaciones**: Inicialmente, requiere crear y testear el repositorio.

---

## Estrategias avanzadas: integrando Docker y herramientas externas

### Docker + `duplicati` (interfaz web para backups)
[Duplicati](https://www.duplicati.com/) ofrece una UI sencilla para gestionar copias de seguridad incrementales con múltiples backends (SFTP, Google Drive, Wasabi). Ideal si prefieres evitar scripts manuales.

**Ejemplo con volumen Docker**:
1. Ejecuta Duplicati en un contenedor:
   ```yaml
   # docker-compose.yml
   version: '3'
   services:
     duplicati:
       image: duplicati/duplicati:latest
       container_name: duplicati
       ports:
         - "8200:8200"
       volumes:
         - duplicati_data:/data
         - /var/lib/docker/volumes:/backup_volumes
         - /mnt/backup:/backup_externo
       restart: unless-stopped
   ```
2. Configura en la UI una tarea que copie `/backup_volumes/nextcloud_data` a un servidor SFTP remoto.

**Ventajas**: Interfaz gráfica intuitiva. **Limitaciones**: Menos flexible para automatizaciones complejas.

---
### Docker + `borg` (backup deduplicado y local)
[BorgBackup](https://www.borgbackup.org/) comprime y deduplica datos en repositorios locales o remotos (SFTP, AWS). Su rendimiento es excelente para datos estáticos como configuraciones o volúmenes de configuraciones (ej: traefik, portainer).

**Ejemplo con Borg**:
```bash
# Instalación
sudo apt install borg

# Crear repositorio en servidor remoto (ej: SSH)
borg init --encryption=repokey user@remotebackup:/mnt/borg_homelab

# Backup incremental
borg create --stats --progress \
   user@remotebackup:/mnt/borg_homelab::{hostname}-{now:%Y-%m-%d-%H%M} \
   /var/lib/docker/volumes/ \
   /home/user/configs/

# Programar con systemd (mejor que cron para logs)
sudo systemctl enable --now borg-backup.service
```

---
## Buenas prácticas para backups fiables

1. **Regla 3-2-1**: Mantén **3 copias** de los datos en **2 soportes distintos** (local + remoto) y **1 copia fuera de casa** (ej: Backblaze). Prioriza lo crítico.
2. **Prueba recuperaciones**: Semestralmente, restaura un subconjunto de datos en un entorno aislado para validar scripts.
3. **Notificaciones**: Usa herramientas como `apprise` ([ejemplo](https://github.com/caronc/apprise)) o notificaciones de Tuya/SmartThings para detectar fallos:
   ```bash
   # En backup_volumenes.sh
   if ! restic -r b2:restic-homelab:/ check; then
       apprise -t "Fallo en backup" -b "Revisa restic en $(hostname)"
   fi
   ```
4. **Seguridad**: Para datos sensibles, cifra los repositorios (Borg/B2) y utiliza contraseñas robustas. Evita guardar credenciales en `/etc/docker/daemon.json`.

---
## ¿Qué herramienta elegir?

| Requisito          | rsync+cron | Restic      | Borg        | Duplicati   |
|--------------------|------------|-------------|-------------|-------------|
| Facilidad          | ★★★★★     | ★★★★☆       | ★★★☆☆       | ★★★★★       |
| Versatilidad       | ★★★☆☆     | ★★★★★       | ★★★★★       | ★★★★☆       |
| Cifrado integrado  | No         | Sí          | Sí          | Sí          |
| Soporte remoto     | SFTP/NAS  | S3/B2/SFTP | SFTP/Local  | Multple     |
| Recuperación táctil| ★★★☆☆     | ★★★★☆       | ★★★★★       | ★★★★☆       |

---
**Conclusión**: Automatiza tus backups **hoy**, no cuando un disco falle. Empieza pequeño (ej: `rsync` de PostgreSQL a un USB) y escala con herramientas como Restic o Borg. La clave es la constancia: un backup solo es bueno si estás seguro de poder restaurarlo.
