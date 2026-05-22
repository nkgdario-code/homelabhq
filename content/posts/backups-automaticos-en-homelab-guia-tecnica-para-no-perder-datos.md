---
title: "Backups automáticos en homelab: guía técnica para no perder datos"
description: "Aprende a automatizar backups en tu homelab con rsync, BorgBackup y Duplicati. Estrategias de rotación, pruebas de recuperación y-storage cloud vs local para proteger tus datos."
date: 2026-05-22
draft: false
tags: ["backups automáticos", "homelab", "BorgBackup", "Docker"]
---

## Automatización de backups en homelab: estrategias esenciales para no perder datos

Mantener copias de seguridad fiables en un homelab es tan crítico como configurar correctamente los servicios. La pérdida de datos no solo implica reinstalar aplicaciones, sino también recuperar configuraciones complejas, bases de datos o archivos personales que pueden tardar horas o días en reconstruirse. La automatización es la clave para evitar olvidos humanos y garantizar recuperaciones rápidas ante fallos de hardware, errores de configuración o ataques de ransomware.

---

### **1. Componentes críticos para un sistema de backups robusto**

Un backup completo debe incluir:
- **Datos persistentes**: configuraciones de Docker (`/var/lib/docker/volumes/`), bases de datos (MariaDB, PostgreSQL), archivos de usuarios y configuraciones de aplicaciones (Nextcloud, Home Assistant).
- **Metadatos del sistema**: scripts de configuración, manuales de servicios y registros (logs) importantes.
- **Sistema operativo**: aunque no se suele respaldar, tener una imagen mínima del sistema operativo en un disco externo ayuda a recuperar rápidamente ante fallos catastróficos.

Para servidores pequeños (Raspberry Pi, mini-PCs), prioriza los datos persistentes sobre el SO. En servidores más potentes, usa clonaciones completas del disco (como `dd` o herramientas como Clonezilla) junto con backups de datos.

---

### **2. Herramientas clave: rsync, BorgBackup y Duplicati**

**rsync** es la herramienta más simple y eficiente para sincronizar archivos entre directorios locales o remotos (incluso por SSH). Ejemplo básico para respaldar la carpeta de Docker:

```bash
rsync -avz --delete /var/lib/docker/volumes/ /mnt/backups/docker_volumes/
```

- `-a`: modo archivo (preserva permisos, fechas, enlaces simbólicos).
- `-v`: modo verbose (para depuración).
- `-z`: comprime durante la transferencia.
- `--delete`: elimina archivos en destino que no existan en origen (útil para sincronizar).

**BorgBackup** es una solución más avanzada que usa deduplicación de datos, cifrado y compresión. Ideal para backups incrementales y remotos (opciona S3, SSH o repositorios locales). Instalación y ejemplo:

```bash
sudo apt install borgbackup
borg init --encryption=repokey /mnt/backups/homelab_borg
borg create --stats --progress /mnt/backups/homelab_borg::homelab-{now} /var/lib/docker/volumes/
```

- `now`: variable que genera una etiqueta con la fecha automáticamente.
- `--stats` y `--progress`: muestran información detallada durante la ejecución.

**Duplicati** es una opción con interfaz web, ideal para usuarios que prefieren evitar la terminal. Soporta múltiples proveedores de almacenamiento (Backblaze, Google Drive, Wasabi) y cifrado AES-256. Configuración básica mediante Docker:

```yaml
version: '3'
services:
  duplicati:
    image: duplicati/duplicati
    environment:
      - CLI_ARGS=--webservice-interface=any --server-data=/config
    volumes:
      - ./duplicati-config:/config
      - /var/lib/docker/volumes:/data_to_backup
      - /mnt/backups/duplicati:/backups
    ports:
      - 8200:8200
```

---
### **3. Estrategias de rotación: la regla 3-2-1**

Para evitar que un único backup sea vulnerable (ejemplo: un disco externo puede fallar), aplica la **regla 3-2-1**:
- **3 copias** de los datos (original + 2 copias de seguridad).
- **2 tipos distintos de almacenamiento** (ejemplo: NAS local + almacenamiento en la nube).
- **1 copia fuera de sitio** (cloud, otro hogar o un servidor remoto).

Ejemplo práctico con `borg` y rotación:
```bash
# Backup diario (7 días de retención)
borg create --stats /mnt/backups/homelab_borg::homelab-{now} /var/lib/docker/volumes/

# Borrar backups antiguos (más de 7 días)
borg prune --keep-daily 7 --keep-weekly 4 /mnt/backups/homelab_borg
```

---
### **4. Pruebas de recuperación: lo único que valida un backup**

De nada sirve tener backups si no se pueden restaurar. Establece un **plan de recuperación** que incluya:
1. **Pruebas trimestrales**: Restaura archivos críticos en un entorno de pruebas (un contenedor Docker temporal o una VM).
2. **Documentación**: Registra los pasos para restaurar cada servicio (ejemplo: "Para recuperar Home Assistant, clone el volumen de Docker y reinicia el servicio").
3. **Logs de verificación**: Usa herramientas como `borg check` para validar la integridad de los repositorios:

```bash
borg check /mnt/backups/homelab_borg --verify-data
```

Ejemplo de restauración con `borg`:
```bash
# Listar backups disponibles
borg list /mnt/backups/homelab_borg

# Restaurar un volumen específico
borg extract /mnt/backups/homelab_borg::homelab-2024-01-15 /var/lib/docker/volumes/postgres_data/
```

---
### **5. Cloud vs. Local: ventajas y costes**

| **Opción**       | **Ventajas**                          | **Desventajas**                     | **Coste aproximado** |
|-------------------|----------------------------------------|-------------------------------------|-----------------------|
| **Local (NAS)**  | Velocidad, coste reducido, control total | Riesgo físico (incendio, robo)     | 100-300€ (discos duros) |
| **Remoto (SSH)**  | Fuera de sitio, automático              | Requiere ancho de banda, latencia   | 5-15€/mes (VPS barato) |
| **Cloud (S3, B2)** | Escalable, seguro, accesible desde cualquier sitio | Coste mensual por almacenamiento | 5-20€/mes (100GB) |
| **Físico (exFAT)** | Restauración inmediata, sin dependencias | Poco práctico para +100GB, riesgo de fallo | 50-100€ (discos externos) |

**Recomendación**: Combina opciones. Ejemplo:
- **Copias locales diarias** (borg + NAS de discos duros).
- **Copias semanales a cloud** (Duplicati a Backblaze B2, ~6€/mes para 100GB).
- **Copias mensuales en un disco externo** (guardado en casa de un familiar).

---
### **6. Automatización con cron y Docker Compose**

Para evitar depender de un script manual, usa `cron` para programar backups y Docker Compose para servicios como Duplicati.

**Ejemplo con cron para borg** (ejecuta a las 3 AM diariamente):
```bash
0 3 * * * borg create --stats /mnt/backups/homelab_borg::homelab-{now} /var/lib/docker/volumes/ && borg prune --keep-daily 7 --keep-weekly 4 /mnt/backups/homelab_borg
```

**Ejemplo con Docker Compose (backup incremental con rsync)**:
```yaml
version: '3'
services:
  backup-rsync:
    image: atmoz/sftp
    volumes:
      - /var/lib/docker/volumes:/backup/source
    command: /backup/source /home/backupuser
    environment:
      - SFTP_USERS=backupuser:password
```

---
**Recuerda**: Un backup no es válido hasta que no se haya restaurado al menos una vez. Automatiza el proceso, pero dedica tiempo cada 6 meses a probar manualmente la recuperación de datos críticos.

---
