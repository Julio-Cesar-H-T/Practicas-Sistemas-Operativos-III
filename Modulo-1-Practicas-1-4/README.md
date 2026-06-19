# 🖥️ Módulo 1 — Realización de Tareas Básicas de Administración del Sistema

> **Sistema Operativo:** Rocky Linux
> **Materia:** Sistemas Operativos III · ITLA
> **Lista de Reproducción:** https://www.youtube.com/playlist?list=PL1bMSHFyMPr6hB8Mdc0zd128RzLvOEC-v
---

## 📋 Contenido del Módulo

| # | Práctica | Tema |
|---|----------|------|
| 1.1 | Instalación del Sistema Operativo | Instalación gráfica de Rocky Linux |
| 1.2 | Configuración de Parámetros de Red | NetworkManager (nmcli / nmtui) |
| 1.3 | Gestión de Usuarios y Grupos | useradd, usermod, groupadd |
| 1.4 | Gestión de Permisos de Archivos | chmod, propietarios y grupos |

---

## 1.1 — Instalación del Sistema Operativo

Instalación de **Rocky Linux** realizada de forma 100% gráfica mediante el instalador Anaconda. No se requirieron comandos de terminal en esta etapa.

https://www.youtube.com/watch?v=SR7oZ9QimHM&1bMSHFyMPr6hB8Mdc0zd128RzLvOEC-v

---

## 1.2 — Configuración de Parámetros de Red

### Objetivo
Configurar la conectividad de red de la VM tanto por DHCP como por IP estática, usando NetworkManager.

### Comandos utilizados

**Comprobar la IP y estado actual de la red:**
```bash
ip a
nmcli device status
```

**Eliminar una conexión de red existente:**
```bash
sudo nmcli connection delete ens160
```

**Agregar conexión por DHCP vía CLI:**
```bash
sudo nmcli connection add type ethernet \
  con-name ens160 \
  ifname ens160 \
  ipv4.method auto \
  autoconnect yes
```

**Agregar IP estática vía CLI:**
```bash
sudo nmcli connection add type ethernet \
  con-name ens160 \
  ifname ens160 \
  ipv4.addresses 192.168.1.100/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "8.8.8.8 8.8.4.4" \
  ipv4.method manual \
  autoconnect yes
```

**Activar una conexión de red:**
```bash
sudo nmcli connection up ens160
```

**Interfaz gráfica de NetworkManager (modo texto):**
```bash
sudo nmtui
```

---

## 1.3 — Gestión de Usuarios y Grupos

### Objetivo
Crear, modificar y eliminar usuarios y grupos en Rocky Linux, incluyendo asignación de privilegios administrativos.

### Comandos utilizados

**Crear usuario:**
```bash
sudo useradd JulioCesar
```

**Asignar contraseña:**
```bash
sudo passwd JulioCesar
```

**Otorgar permisos de administrador (grupo `wheel`):**
```bash
sudo usermod -aG wheel JulioCesar
```

**Ver grupos a los que pertenece un usuario:**
```bash
groups JulioCesar
```

**Ver dónde se aloja el home del usuario:**
```bash
ls /home
```

**Crear un grupo:**
```bash
sudo groupadd Guest
```

**Crear otro usuario y asignarlo a un grupo:**
```bash
sudo useradd Hernandez
sudo usermod -aG Guest Hernandez
```

**Eliminar un usuario (incluyendo su home):**
```bash
sudo userdel -r Hernandez
```

**Eliminar un grupo:**
```bash
sudo groupdel Guest
```

---

## 1.4 — Gestión de Permisos de Archivos

### Objetivo
Practicar la creación de archivos/directorios y la modificación de permisos mediante `chmod`.

### Comandos utilizados

**Crear directorio y archivo de texto:**
```bash
mkdir materia
cd materia
touch estudiante.txt
```

**Editar el archivo con `vi`:**
```bash
vi estudiante.txt
# Modificar contenido y guardar con :wq
```

**Verificar el contenido guardado:**
```bash
cat estudiante.txt
```

**Ver archivos junto a sus permisos:**
```bash
ls -l
```

**Permisos: control total solo para el propietario:**
```bash
chmod 700 estudiante.txt
```

**Permisos: control total solo para el grupo:**
```bash
chmod 070 estudiante.txt
```

**Ver directorio actual:**
```bash
pwd
```

**Crear un segundo directorio y copiar el archivo:**
```bash
mkdir materia2
cp /home/Jtibrey/materia/estudiante.txt /home/Jtibrey/materia2
```

**Eliminar un directorio junto con su contenido:**
```bash
rm -r materia
```
---

## 📌 Conclusiones del Módulo

Este módulo cubrió las bases de la administración de Rocky Linux: instalación, configuración de red persistente con NetworkManager, gestión del ciclo de vida de usuarios/grupos, y el modelo de permisos Unix (propietario/grupo/otros) mediante `chmod` en notación octal.

---

[⬅ Volver al índice general](../README.md)
