---
title: "Guía para automatizar backups en homelab: pasos claros y herramientas libres"
description: "Domina el arte de los backups automáticos en tu homelab: estrategias, herramientas como Duplicati o restic, casos prácticos y cómo evitar la pesadilla de perder datos. Tutorial paso a paso para Docker, NAS y máquinas virtuales."
date: 2026-05-30
draft: false
tags: ["backups", "automatización", "Docker", "homelab"]
---

---

## Automatizar backups en tu homelab: cómo dormir tranquilo sin miedo a perder datos

Perder información valiosa por un fallo de disco, un borrado accidental o un error humano es uno de los peores escenarios en un homelab. La solución no es cruzar los dedos y rezar por suerte, sino implementar un sistema de backups **automáticos, eficientes y verificables**, que funcionen incluso cuando estás dormido. Desde servidores Docker hasta máquinas locales o NAS, la automatización es la clave para eliminar el riesgo sin añadir complejidad. A continuación, te explicamos cómo montar un esquema robusto con herramientas libres, ejemplos prácticos y truquillos que ahorran dolores de cabeza.

---

### 1. Define tu estrategia: ¿qué, dónde y cuándo?

Antes de tocar un comando, necesarios tres pilares:

1. **Qué se respalda**: prioriza lo crítico. En un homelab suelen ser:
   - Volúmenes Docker (ej: bases de datos PostgreSQL, directorios de apps como Nextcloud).
   - Configuraciones manuales (ej: `/etc/nginx`, scripts personalizados).
   - Máquinas virtuales (si usas KVM o Proxmox).
   - Archivos personales (opcional, pero recomendado si el homelab también sirve como almacenamiento domestico).

2. **Dónde se guardan los backups**: la regla de oro es la **regla 3-2-1**:
   - **3** copias de cada dato.
   - En **2** medios distintos (ej: disco local + nube).
   - **1** copia fuera del lugar físico (ej: un proveedor como Backblaze B2 o AWS S3).

3. **Cuándo**: define ventanas realistas. Para un homelab personal, un horario nocturno suele ser suficiente, pero ajusta frecuencias según el volumen de cambios (ej: diario para bases de datos, semanal para archivos estáticos).

**Ejemplo práctico**:
```
Ruta origen local: /mnt/docker/vol_bases_datos (volumen Docker de WordPress)
Backup diario a: /mnt/backup/nas (disco local)
Copia semanal a: s3://mi-bucket-homelab (Backblaze B2)
```

---

### 2. Herramientas para automatizar: versatilidad sin complejidad

No reinventes la rueda. Estas son las opciones más equilibradas para un homelab:

#### a) Para Docker: *Volsync* + *duplicati*
- **Volsync** (de la suite [Veeam](https://www.veeam.com/)): sincroniza volúmenes Docker entre hosts o a un NAS. Ideal para backups en caliente (sin parar contenedores).
  ```bash
  docker run --rm -it -v /var/run/docker.sock:/var/run/docker.sock -v /backup:/backup \
    --name volsync volsync/volsync sync --src /ruta/volumen --dest /backup/volumen_backup
  ```
- **Duplicati** (software libre): cliente de backups cifrados con compresión y soporte multi-nube. Perfecto para mover copias a S3, Backblaze o un segundo disco.
  ```yaml
  # docker-compose.yml para Duplicati
  services:
    duplicati:
      image: duplicati/duplicati
      ports:
        - "8200:8200"
      volumes:
        - /backup:/backup
        - /mnt/docker/vol_bases_datos:/origen
  ```

#### b) Para máquinas virtuales: *Proxmox Backup Server (PBS)*
Si usas Proxmox, su herramienta integrada **PBS** es la más eficiente. Configura tareas automatizadas por VM:
1. Instala PBS en una VM o hardware dedicado.
2. Activa el add-on de backups en Proxmox.
3. programa copias incrementales diarias y retención de 30 días.

#### c) Para archivos sueltos: *rsync* + *cron*
Si solo necesitas copiar directorios locales, combínalo con `cron` para automatizar:
```bash
# Script de backup (/opt/backup_homelab.sh)
rsync -avz --delete /home/usuario/documentos /mnt/backup/nas/documents
rsync -avz /etc/nginx /mnt/backup/nas/configs
```
```bash
# Programar en cron (ejecutar: crontab -e)
0 3 * * * /opt/backup_homelab.sh >> /var/log/backup_homelab.log 2>&1
```

---
### 3. Verificación: ¿el backup sirve si lo necesitas?

Automatizar está bien, pero **un backup no verificado es basura**. Implementa esto:

1. **Pruebas de restauración**: ejecuta un cron semanal para restaurar un archivo aleatorio de cada backup.
   ```bash
   # Restaurar un archivo de WordPress
   tar -xzf /mnt/backup/nas/vol_bases_datos_backup.tar.gz -C /tmp/wp_test var/www/html/wp-config.php
   cmp /tmp/wp_test/wp-config.php /var/www/html/wp-config.php || echo "ERROR: Backup corrupto!"
   ```
2. **Alertas por fallos**: usa *Gotify*, *Apprise* o *Discord webhooks* para notificaciones.
   ```yaml
   # .env de Duplicati (webhook para Discord)
   DUPLICATI_WEBHOOK_URL=https://discord.com/api/webhooks/ID_TOKEN
   ```
3. **Checksums**: al copiar, genera y almacena hashes (SHA256) para detectar corrupción.
   ```bash
   sha256sum /mnt/backup/nas/vol_bases_datos_backup.tar.gz > /mnt/backup/nas/vol_bases_datos_backup.sha256
   ```

---
### 4. Ahorrando espacio: deduplicación y compresión

Los backups pueden llenar discos rápido. Usa estas técnicas:

- **Compresión**: en Duplicati o Duplicati, activa *zip* o *7z*.
- **Deduplicación**: *restic* (cliente CLI) la hace excepcionalmente bien.
  ```bash
  # Backup incrementales con restic (ejemplo para Nextcloud)
  restic -r s3:mi-bucket-homelab init  # Solo la primera vez
  restic -r s3:mi-bucket-homelab backup /mnt/docker/vol_nextcloud
  ```
- **Retention policies**: elimina backups antiguos automáticamente. En Duplicati, configura en la interfaz web:
  ```
  Retener: últimos 7 días, últimos 4 semanas, últimos 12 meses.
  ```

---
### 5. Caso práctico: backup integral de un homelab con Docker

Vamos a montar un sistema que:
- Copie los volúmenes de Docker (WordPress + PostgreSQL).
- Guarde configuraciones de Nginx.
- Suba una copia a Backblaze B2.
- Verifique integridad y notifique errores.

**1. Crea un volumen para backups locales**:
```bash
docker volume create backup_vol
```

**2. Configura Duplicati** (docker-compose.yml):
```yaml
version: '3'
services:
  duplicati:
    image: duplicati/duplicati
    restart: unless-stopped
    ports:
      - "8200:8200"
    volumes:
      - backup_vol:/backup
      - /var/run/docker.sock:/var/run/docker.sock
      - /etc/nginx:/configs/nginx

volumes:
  backup_vol:
```

**3. Defíne la tarea en Duplicati**:
1. Accede a `http://tuhost:8200`.
2. Añade un destino "Local folder path" → `/backup/`.
3. En el apartado de compresión: *zip*, nivel 5.
4. En *retention*: elimina backups de más de 30 días.
5. En *verification*: programa una comprobación semanal.

**4. Verifica el flujo**:
```bash
# Listar backups generados
restic -r s3:mi-bucket-homelab snapshots list

# Restaurar un archivo de WordPress (prueba)
restic -r s3:mi-bucket-homelab restore latest --target /tmp/restore_wp
```

---
### Cierre: preguntas frecuentes

- **¿En qué disco dejo los backups?** Nunca en el mismo que el origen. Usa: disco USB externo, NAS, o disco dedicado (ej: un WD Red).
- **¿Puedo cifrar los backups?** ¡Imprescindible! Duplicati y restic lo soportan de serie (clave o contraseña).
- **¿Qué pasa si falla el backup?** Usa la *retention policy* para mantener N backups (ej: 3 generaciones) y monitoriza logs con herramientas como *Logwatch* o *Prometheus + Grafana*.
- **¿Y si quiero backups en tiempo real?** Para servidores críticos, considera *rsync* en modo *--link-dest* o *BorgBackup* (más ligero que Duplicati).

---
---
