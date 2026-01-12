# CheckMK RAW

Sistema de monitorización IT profesional. Monitoriza servidores, aplicaciones, redes y servicios.

## Características

- 🖥️ **Monitorización completa**: Servidores, aplicaciones, red, cloud
- 📊 **Dashboards**: Visualización en tiempo real
- 🔔 **Alertas**: Notificaciones por email, SMS, Slack, etc.
- 📈 **Métricas**: Históricos de rendimiento y disponibilidad
- 🔌 **Agentes**: Linux, Windows, Docker, SNMP
- 🌐 **Multi-sitio**: Gestión centralizada de múltiples sites
- 📱 **Aplicación móvil**: Monitorización desde el móvil
- 🔧 **Extensible**: Plugins y checks personalizados

## Requisitos Previos

- Docker Engine instalado
- Docker Compose instalado
- **Para Traefik o NPM**: Red Docker `proxy` creada
- **Dominio configurado**: Para acceso HTTPS
- **Contraseña generada**: CMK_PASSWORD

⚠️ **IMPORTANTE**: CheckMK RAW es la versión gratuita. Para funcionalidades enterprise, considera CheckMK Enterprise.

## Generar Contraseña

**Antes de cualquier despliegue**, genera una contraseña segura:

```bash
# CMK_PASSWORD (usuario cmkadmin)
openssl rand -base64 32
```

Guarda el resultado, lo necesitarás en el archivo `.env`.

> ⚠️ **Importante**: Usa comillas simples en el archivo `.env` si la contraseña contiene caracteres especiales.
> Ejemplo: `CMK_PASSWORD='tu_password_generado'`

---

## Archivos de este Repositorio

Este repositorio contiene archivos de ejemplo:
- `docker-compose.yml` - Configuración base del contenedor
- `.env.example` - Plantilla de variables de entorno
- `docker-compose.override.traefik.yml.example` - Labels para Traefik
- `README.md` - Esta documentación

> 💡 **Tip**: Puedes copiar estos archivos manualmente o clonar el repositorio.

---

## Despliegue con Docker Compose

### 1. Crear Directorio y Archivos

```bash
# Crear directorio
mkdir checkmk
cd checkmk
```

### 2. Crear docker-compose.yml

Crea el archivo `docker-compose.yml`:

```yaml
services:
  checkmk:
    container_name: checkmk
    image: checkmk/check-mk-raw:latest
    restart: unless-stopped
    environment:
      CMK_SITE_ID: ${CMK_SITE_ID:-monitoring}
      CMK_PASSWORD: ${CMK_PASSWORD}
    volumes:
      - checkmk_data:/omd/sites
    networks:
      - proxy
    tmpfs:
      - /opt/omd/sites/${CMK_SITE_ID:-monitoring}/tmp:uid=1000,gid=1000

volumes:
  checkmk_data:

networks:
  proxy:
    external: true
```

### 3. Configurar Variables de Entorno

Crea el archivo `.env`:

```env
# Contraseña para cmkadmin (GENERAR NUEVA)
# Generar con: openssl rand -base64 32
CMK_PASSWORD='tu_password_generado'

# Site ID (opcional)
CMK_SITE_ID=monitoring

# Dominio
DOMAIN_HOST=checkmk.dominio.com
```

> ⚠️ **Importante**: Usa comillas simples si la contraseña contiene caracteres especiales.

### 4. (Opcional) Configurar Traefik

Si usas Traefik, crea `docker-compose.override.yml`:

```yaml
services:
  checkmk:
    labels:
      - traefik.enable=true
      - traefik.http.routers.checkmk-http.rule=Host(`${DOMAIN_HOST}`)
      - traefik.http.routers.checkmk-http.entrypoints=web
      - traefik.http.routers.checkmk-http.middlewares=redirect-to-https
      - traefik.http.routers.checkmk.rule=Host(`${DOMAIN_HOST}`)
      - traefik.http.routers.checkmk.entrypoints=websecure
      - traefik.http.routers.checkmk.tls.certresolver=letsencrypt
      - traefik.http.services.checkmk.loadbalancer.server.port=5000
      - traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https
```

### 5. Desplegar

```bash
# Crear red proxy si no existe
docker network create proxy

# Iniciar servicios
docker compose up -d

# Ver logs
docker compose logs -f checkmk
```

⏱️ La inicialización puede tardar **2-3 minutos** (creación del site, configuración OMD).

### 6. Verificar el Despliegue

```bash
# Ver logs en tiempo real
docker compose logs -f checkmk

# Verificar estado del site CheckMK
docker compose exec checkmk omd status

# Listar sites disponibles
docker compose exec checkmk omd sites
```

**Acceso**:
- Traefik: `https://<DOMAIN_HOST>/monitoring/` (ejemplo: `https://checkmk.example.com/monitoring/`)
- NPM: Configurar en NPM apuntando a `checkmk` puerto `5000` → `https://checkmk.example.com/monitoring/`

⚠️ **IMPORTANTE**: El path `/monitoring/` es obligatorio (nombre del site CheckMK).

**Credenciales iniciales**:
- Usuario: `cmkadmin`
- Contraseña: La configurada en `CMK_PASSWORD`

---

## Método Alternativo: Clonar desde Git

Si prefieres usar Git para mantener la configuración actualizada:

```bash
# Clonar repositorio
git clone https://git.ictiberia.com/groales/checkmk.git
cd checkmk

# Configurar variables de entorno
cp .env.example .env
nano .env  # Editar: CMK_PASSWORD, DOMAIN_HOST

# (Opcional) Para Traefik
cp docker-compose.override.traefik.yml.example docker-compose.override.yml

# Desplegar
docker compose up -d
```

---

## Comandos OMD Útiles

```bash
# Ver todos los servicios del site
docker compose exec checkmk omd status

# Reiniciar site
docker compose exec checkmk omd restart

# Backup manual
docker compose exec checkmk omd backup /tmp/backup.tar.gz

# Ver configuración del site
docker compose exec checkmk omd config
```

---

## Modos de Despliegue

### Traefik (Proxy Inverso con SSL automático)

**Requisitos**:
- Stack de Traefik desplegado
- Red `proxy` creada
- DNS apuntando al servidor

Si desplegaste con **Opción A (Git Repository)** y añadiste el override en el paso 5, ya está todo configurado.

Accede a `https://checkmk.tudominio.com`

### Nginx Proxy Manager (NPM)

**Requisitos**:
- NPM desplegado y accesible
- Red `proxy` creada
- DNS apuntando al servidor

**Pasos**:

1. Despliega el stack con el `docker-compose.yml` base (sin override)

2. En NPM, crea un nuevo **Proxy Host**:
   - **Domain Names**: `checkmk.tudominio.com`
   - **Scheme**: `http`
   - **Forward Hostname / IP**: `checkmk`
   - **Forward Port**: `5000`
   - **Cache Assets**: ✅ Activado
   - **Block Common Exploits**: ✅ Activado
   - **Websockets Support**: ✅ Activado

3. En la pestaña **SSL**:
   - **SSL Certificate**: Request a new SSL Certificate
   - **Force SSL**: ✅ Activado
   - **HTTP/2 Support**: ✅ Activado

4. Guarda y accede a `https://checkmk.tudominio.com`

---

## Configuración Inicial

### Primer Acceso

1. Accede a CheckMK: `https://checkmk.tudominio.com/monitoring/`
2. Login con:
   - **Usuario**: `cmkadmin`
   - **Contraseña**: La que configuraste en `CMK_PASSWORD`

### Configuración Básica

#### Cambiar Contraseña (Recomendado)

1. Click en **cmkadmin** (esquina superior derecha)
2. **Change password**
3. Ingresa contraseña actual y nueva
4. **Save**

#### Configurar Site

**Setup → General → Global settings**:

- **Site name**: Nombre descriptivo de tu site
- **Admin email**: Email del administrador
- **Timezone**: `Europe/Madrid`

#### Configurar Notificaciones

**Setup → Notifications**:

- **Email notifications**: Configura servidor SMTP
- **Slack/Teams**: Webhooks para notificaciones
- **SMS**: Proveedores de SMS

---

## Añadir Hosts

### Host Linux con Agente

1. **Setup → Hosts → Add host**
2. **Hostname**: `servidor01.example.com`
3. **IP address**: IP del servidor
4. **Monitoring agents**: `Check_MK Agent`
5. **Save & go to service configuration**

**Instalar agente en el host**:

```bash
# Descargar agente desde CheckMK
wget https://checkmk.tudominio.com/monitoring/check_mk/agents/check-mk-agent_2.x.x-1_all.deb

# Instalar
dpkg -i check-mk-agent_2.x.x-1_all.deb

# Permitir conexión desde CheckMK
# Edita /etc/xinetd.d/check-mk-agent o /etc/check_mk/check-mk-agent.cfg
```

6. **Discover services** en CheckMK
7. **Activate changes**

### Host Windows con Agente

1. Descarga el agente Windows desde CheckMK
2. Ejecuta `check_mk_agent.msi` en el servidor Windows
3. Configura IP del servidor CheckMK
4. En CheckMK, añade el host como arriba
5. **Discover services** y **Activate changes**

### Dispositivo de Red (SNMP)

1. **Setup → Hosts → Add host**
2. **Hostname**: `switch01.example.com`
3. **Monitoring agents**: `SNMP`
4. **SNMP community**: `public` (o tu community string)
5. **Save & discover services**
6. **Activate changes**

---

## Dashboards y Vistas

### Crear Dashboard Personalizado

1. **Customize → Dashboards**
2. **Add dashboard**
3. Añade widgets:
   - **Host statistics**: Resumen de hosts
   - **Service statistics**: Resumen de servicios
   - **Performance graphs**: Gráficas de rendimiento
   - **Top alerts**: Alertas recientes

### Vistas Personalizadas

**Customize → Views**:

- Crea vistas filtradas por grupos
- Exporta vistas como PDF/CSV
- Programa envíos automáticos

---

## Backup y Restauración

### Backup Manual

```bash
# Backup completo del site
docker exec checkmk omd backup /tmp/checkmk-backup-$(date +%Y%m%d).tar.gz

# Copiar backup fuera del contenedor
docker cp checkmk:/tmp/checkmk-backup-FECHA.tar.gz ./
```

### Backup Automático

Script `/root/backup-checkmk.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/backups/checkmk"
DATE=$(date +%Y%m%d-%H%M%S)
RETENTION_DAYS=7

mkdir -p $BACKUP_DIR

# Backup del site
docker exec checkmk omd backup /tmp/checkmk-backup-$DATE.tar.gz
docker cp checkmk:/tmp/checkmk-backup-$DATE.tar.gz $BACKUP_DIR/
docker exec checkmk rm /tmp/checkmk-backup-$DATE.tar.gz

# Limpiar backups antiguos
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "Backup completado: $DATE"
```

Cron:
```bash
chmod +x /root/backup-checkmk.sh
crontab -e

# Backup diario a las 3 AM
0 3 * * * /root/backup-checkmk.sh
```

### Restauración

```bash
# Copiar backup al contenedor
docker cp checkmk-backup-FECHA.tar.gz checkmk:/tmp/

# Restaurar
docker exec checkmk omd restore /tmp/checkmk-backup-FECHA.tar.gz

# Reiniciar
docker restart checkmk
```

---

## Actualización

```bash
# 1. Backup ANTES de actualizar
docker exec checkmk omd backup /tmp/checkmk-pre-update.tar.gz
docker cp checkmk:/tmp/checkmk-pre-update.tar.gz ./

# 2. Detener contenedor
docker stop checkmk

# 3. Actualizar imagen
docker pull checkmk/check-mk-raw:latest

# 4. Iniciar contenedor
docker start checkmk

# 5. Verificar versión
docker exec checkmk omd version

# 6. Acceder y verificar funcionamiento
```

---

## Solución de Problemas

### CheckMK no inicia

```bash
# Ver logs
docker logs checkmk --tail 100

# Verificar permisos del volumen
docker exec checkmk ls -la /omd/sites

# Reiniciar site
docker exec checkmk omd restart
```

### Agente no conecta

```bash
# Probar conexión desde CheckMK
docker exec checkmk check_mk_agent HOST

# Verificar firewall en host
# Puerto 6556 TCP debe estar abierto

# Ver logs del agente
# En Linux: /var/log/check_mk/
# En Windows: C:\ProgramData\checkmk\agent\log\
```

### Site corrupto

```bash
# Detener site
docker exec checkmk omd stop

# Verificar integridad
docker exec checkmk omd check

# Reparar (si es posible)
docker exec checkmk omd repair

# Restaurar desde backup (última opción)
docker exec checkmk omd restore /tmp/backup.tar.gz
```

---

## Variables de Entorno

### Requeridas

| Variable | Descripción | Ejemplo |
|----------|-------------|---------|
| `CMK_PASSWORD` | Contraseña usuario cmkadmin | `generada_con_openssl` |
| `DOMAIN_HOST` | Dominio completo | `checkmk.example.com` |

### Opcionales

| Variable | Descripción | Valor por defecto |
|----------|-------------|-------------------|
| `CMK_SITE_ID` | ID del site CheckMK | `monitoring` |

---

## Recursos

- [CheckMK Documentación Oficial](https://docs.checkmk.com/)
- [CheckMK Docker Hub](https://hub.docker.com/r/checkmk/check-mk-raw)
- [CheckMK Community](https://forum.checkmk.com/)
- [CheckMK GitHub](https://github.com/tribe29/checkmk)

---

## Licencia

CheckMK RAW Edition es software de código abierto bajo licencia GPLv2.
