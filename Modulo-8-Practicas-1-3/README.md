# ☁️ Módulo 8 — Cloud Computing y Contenerización

> **Sistema Operativo:** Rocky Linux

> **Materia:** Sistemas Operativos III · ITLA

> **Lista de Reproducción:** https://www.youtube.com/playlist?list=PL1bMSHFyMPr6yIXO5FyGwC26rNOvzmioP
---

## 📋 Contenido del Módulo

| # | Práctica | Tema |
|---|----------|------|
| 8.1 | Instalación y Configuración de Docker | Docker Engine, Nginx y CUPS en contenedores |
| 8.2 | Instalación de Portainer | Gestión visual de Docker, agentes remotos |
| 8.3 | Despliegue de WordPress con Docker Compose | Multi-contenedor (WordPress + MySQL) |

---

## 8.1 — Instalación y Configuración de Docker

### Objetivo
Instalar Docker Engine en Rocky Linux desde el repositorio oficial, y desplegar contenedores funcionales de Nginx y CUPS con volúmenes persistentes.

> 📖 Referencia oficial: [docs.docker.com/engine/install/rhel](https://docs.docker.com/engine/install/rhel/)

### Desinstalar versiones previas o en conflicto

```bash
sudo dnf remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine \
                  podman \
                  runc
```

### Instalación desde el repositorio oficial

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

**Habilitar e iniciar el servicio:**
```bash
sudo systemctl enable --now docker
sudo systemctl start docker
sudo systemctl status docker
```

**Prueba de funcionamiento:**
```bash
sudo docker run hello-world
docker --version
```

### Desplegar Nginx con contenido personalizado

**Buscar y descargar la imagen:**
```bash
docker search nginx
docker pull nginx
sudo docker images
```

**Crear el contenido web a servir:**
```bash
sudo mkdir /home/website
cd /home/website/
nano index.html
```

```html
<html>
 <head>
  <title>DOCKER x NGINX</title>
 </head>
 <body>
  <H3>Julio Cesar Hernandez Tibrey<br>
  Matrícula: 2025-0702</H3>
 </body>
</html>
```

**Ejecutar el contenedor mapeando puerto y volumen:**
```bash
sudo docker run -d -p 8888:80 -v /home/website:/usr/share/nginx/html/ nginx
docker ps -a
```

**Verificación:** acceder desde el navegador a `127.0.0.1:8888`

> 💡 **Concepto clave:** El flag `-v /home/website:/usr/share/nginx/html/` monta un volumen **bind mount** — el contenido del host se refleja directamente dentro del contenedor sin necesidad de reconstruir la imagen.

### Extra — Desplegar CUPS en contenedor

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl enable --now docker
```

**Buscar y descargar la imagen:**
```bash
docker search cups
docker pull olbat/cupsd
```

**Crear directorio de configuración persistente:**
```bash
sudo mkdir -p /home/cups-config
```

**Ejecutar el contenedor:**
```bash
sudo docker run -d \
  --name cups-server \
  -p 631:631 \
  -v /home/cups-config:/etc/cups \
  --restart unless-stopped \
  olbat/cupsd
```

**Verificar y acceder:**
```bash
docker ps
```
Interfaz web disponible en: `http://127.0.0.1:631`

---

## 8.2 — Instalación de Portainer

### Objetivo
Desplegar **Portainer CE** como interfaz gráfica de gestión de Docker, y conectar un entorno Docker remoto mediante un agente.

> 📖 Referencia oficial: [docs.portainer.io](https://docs.portainer.io/start/install-ce/server/docker/linux)

### Instalación

**Crear el volumen de datos de Portainer:**
```bash
sudo docker volume create portainer_data
```

**Ejecutar el contenedor de Portainer:**
```bash
sudo docker run -d -p 9443:9443 --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:lts
```

**Verificar:**
```bash
docker ps -a
```

**Acceder a la interfaz web:** `https://127.0.0.1:9443`

### Conectar un entorno Docker remoto (agente)

1. Crear el usuario administrador y entrar a la interfaz
2. **Add Environments → Docker Standalone → Start Wizard**
3. Copiar el comando del agente generado por el Wizard y ejecutarlo en la CLI de la máquina remota
4. En Portainer, completar:
   - **Name:** nombre identificador del entorno
   - **Environment address:** `IP_DE_LA_MAQUINA:PUERTO_DEL_AGENTE` (ej. `10.0.0.25:9001`)
5. Clic en **Connect**, luego **Close**

**Gestión de contenedores desde la interfaz:**
```
[Environment creado] → Containers → Nginx → Stop
```

> 💡 **Concepto clave:** Portainer se comunica con el daemon de Docker mediante el socket `/var/run/docker.sock` montado como volumen — por eso el contenedor de Portainer puede controlar contenedores "hermanos" en el mismo host, un patrón conocido como **Docker-out-of-Docker (DooD)**.

---

## 8.3 — Despliegue de WordPress con Docker Compose

### Objetivo
Desplegar una aplicación multi-contenedor (WordPress + base de datos MySQL) utilizando **Docker Compose**, demostrando orquestación declarativa de servicios.

### Instalación de Docker Compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.20.2/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```

### Definición del stack

```bash
mkdir ~/wordpress-docker
cd ~/wordpress-docker
nano docker-compose.yml
```

```yaml
version: '3.8'
services:
  db:
    image: mysql:5.7
    container_name: wordpress_db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppassword
    volumes:
      - db_data:/var/lib/mysql
  wordpress:
    image: wordpress:latest
    container_name: wordpress_app
    depends_on:
      - db
    ports:
      - "8080:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppassword
      WORDPRESS_DB_NAME: wordpress
    volumes:
      - wp_data:/var/www/html
volumes:
  db_data:
  wp_data:
```

### Despliegue del stack

```bash
docker-compose up -d
```

**Verificación:** acceder desde el navegador a `http://localhost:8080` y completar el asistente de instalación de WordPress.

> 💡 **Concepto clave:** `depends_on` garantiza el **orden de arranque** (la base de datos inicia antes que WordPress), pero no espera a que el servicio esté realmente listo para aceptar conexiones — en producción esto se complementa con *healthchecks* o scripts de espera activa (`wait-for-it`). Los **volúmenes nombrados** (`db_data`, `wp_data`) garantizan que los datos persistan aunque los contenedores se destruyan y recreen.

---

## 📌 Conclusiones del Módulo

Este módulo recorrió la progresión natural de la contenerización: contenedores individuales con Docker (Nginx, CUPS), gestión visual centralizada con Portainer, y orquestación multi-contenedor con Docker Compose (WordPress + MySQL). Cada nivel resuelve un problema distinto — Docker resuelve *aislamiento y portabilidad*, Portainer resuelve *visibilidad y administración*, y Compose resuelve *definición declarativa de arquitecturas completas* como código versionable.

---

[⬅ Volver al índice general](../README.md)
