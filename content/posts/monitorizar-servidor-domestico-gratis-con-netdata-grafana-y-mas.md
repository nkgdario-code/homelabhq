---
title: "Monitorizar servidor doméstico gratis con Netdata, Grafana y más"
description: "Descubre herramientas gratuitas como Netdata, Grafana+Prometheus y Uptime Kuma para monitorizar tu homelab. Configuración paso a paso y casos prácticos para no perderte ni un fallo."
date: 2026-06-07
draft: false
tags: ["monitorización servidor", "docker arquitectura", "Netdata", "Prometheus"]
---

## Monitorización gratuita de servidores domésticos: herramientas imprescindibles para tu homelab

Montar un homelab implica gestionar infraestructura crítica, aunque sea a pequeña escala. Sin un sistema de monitorización adecuado, problemas como fallos de disco, sobrecarga de CPU o caídas de red pueden pasar desapercibidos hasta que es demasiado tarde. La buena noticia es que no necesitas soluciones comerciales carísimas para supervisar tu servidor doméstico. Existen herramientas gratuitas, con versiones comunitarias potentes y configuración sencilla que cubren las necesidades básicas de cualquier homelab.

### 1. Netdata: El termómetro en tiempo real de tu servidor

Netdata es una de las soluciones más populares para monitorizar servidores locales debido a su facilidad de implementación y su interfaz visual intuitiva. Se instala en menos de 60 segundos con un solo comando en Linux:

```bash
bash <(curl -Ss https://my-netdata.io/kickstart.sh)
```

Una vez instalado, proporciona métricas en tiempo real de CPU, RAM, disco, red y procesos sin necesidad de configuración adicional. Ejemplo práctico: Si tu servidor empieza a saturar la RAM, Netdata mostrará un gráfico con picos anormales que te permitirá identificar el proceso responsable antes de que el sistema se vuelva inestable. Además, incluye alertas configurables por email o Slack para evitar supervisar manualmente el dashboard.

*Limitación*: Aunque la versión comunitaria es suficiente para homelabs, si necesitas retención de datos a largo plazo (>24 horas), tendrás que usar la versión cloud (gratuita hasta 10 nodos).

### 2. Grafana + Prometheus: La trinidad de la observabilidad

Para homelabs más avanzados que requieran escalabilidad, Prometheus (recopilador de métricas) junto a Grafana (panel de visualización) forma un tandem casi imbatible. La configuración es sencilla si usas Docker:

**Estructura básica con Docker Compose:**
```yaml
version: '3'
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - 9090:9090
  grafana:
    image: grafana/grafana
    ports:
      - 3000:3000
    volumes:
      - grafana-storage:/var/lib/grafana
volumes:
  grafana-storage:
```

Ejemplo de monitorización avanzada: Imagina que quieres supervisar no solo el servidor físico, sino también un NAS con TrueNAS. Con Prometheus, puedes instalar [Prometheus Node Exporter](https://github.com/prometheus/node_exporter) en el NAS y visualizar todos los datos en un único panel de Grafana. Los dashboards de [Grafana Labs](https://grafana.com/grafana/dashboards/) incluyen plantillas para casi cualquier escenario.

*Ventaja clave*: La comunidad de Prometheus/Grafana es enorme. Encontrarás dashboards y exporters para *casi* cualquier dispositivo: routers, switches, cámaras IP, etc.

### 3. Uptime Kuma: Vigila que tu homelab esté siempre "up"

Uptime Kuma es la alternativa gratuita y open-source más popular a servicios como UptimeRobot. Monitoriza la disponibilidad de servicios web, APIs, puertos TCP y respuestas HTTP con avisos instantáneos. La instalación es aún más ligera que las anteriores:

```bash
docker run -d --name uptime-kuma -p 3001:3001 -v uptime-kuma:/app/data louislam/uptime-kuma:latest
```

Ejemplo práctico: Si ejecutas Nextcloud en tu homelab y quieres asegurarte de que el servicio no caiga cuando actualices la base de datos, configura un *ping* en Kuma que verifique cada 30 segundos que la URL `/status.php` responda correctamente. Si falla, recibirás una notificación en Discord, Telegram o email.

*Extra*: Incluye una función de *escrutnio de deficiencias* que simula fallos en cascada para probar tus alertas.

### 4. Portainer: Monitoriza los contenedores (y más)

Si tu homelab gira en torno a Docker, Portainer no solo te permite gestionar contenedores, sino que también incluye métricas básicas de recursos. La versión Community Edition (gratis) es suficiente para la mayoría:

```bash
docker run -d -p 9000:9000 -v /var/run/docker.sock:/var/run/docker.sock portainer/portainer-ce
```

Ejemplo avanzado: Usa la API de Portainer para extraer métricas de uso de CPU, memoria y red de cada contenedor y envíarlas a Prometheus integrando el exporter oficial [portainer-exporter](https://github.com/virely/portainer-exporter).

### 5. Alertas proactivas: Slack, Telegram y demás

Todas estas herramientas permiten notificaciones, pero ¿dónde recibirlas? Las opciones más prácticas para homelabs:

- **Slack**: Integra Prometheus o Uptime Kuma con [incoming webhooks](https://api.slack.com/messaging/webhooks).
- **Telegram**: Usa bots como [Prometheus alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/) o [Uptime Kuma Telegram Bot](https://github.com/s-schmid/uptime-kuma-telegram-bot).
- **Email**: Simple pero efectivo. Configura alertas por umbrales (ej: "RAM >80% durante 10 minutos").

---

### Conclusión: Elige según tu homelab

| Herramienta      | Caso de uso principal               | Complejidad | Requiere Docker |
|------------------|-------------------------------------|-------------|----------------|
| Netdata          | Visor general, alertas rápidas      | Baja        | No            |
| Prometheus/Grafana| Monitorización avanzada, escalable   | Media       | Sí            |
| Uptime Kuma      | Supervisión de servicios web/HTTP    | Baja        | Sí            |
| Portainer        | Estado de contenedores Docker        | Baja        | Sí            |

**Recomendación final**:
- Si buscas algo **solo* para supervisar el servidor físico, **Netdata** es la opción más sencilla.
- Si quieres **centralizar métricas de múltiples dispositivos** (incluyendo el NAS o routers), **Prometheus + Grafana** es la solución definitiva.
- Para **servicios críticos** (Nextcloud, Pi-hole, etc.), añade **Uptime Kuma** para alertas inmediatas.

---
