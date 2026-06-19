# 📦 Módulo 2 — Sistemas Operativos en Ejecución

> **Sistema Operativo:** Rocky Linux
> **Materia:** Sistemas Operativos III · ITLA

---

## 📋 Contenido del Módulo

| # | Práctica | Tema |
|---|----------|------|
| 2.1 | Gestión de Paquetes y Directorios | dnf, repositorios, compilación desde Git |
| 2.2 | Automatización de Tareas | Crontab y AT |
| 2.3 | Administración de Discos y Particiones | fdisk, mkfs, mount |

---

## 2.1 — Gestión de Paquetes y Directorios

### Objetivo
Actualizar el sistema, gestionar repositorios con `dnf`, e instalar herramientas no disponibles oficialmente mediante compilación desde el código fuente (Git).

### Comandos utilizados

**Actualizar paquetes del sistema:**
```bash
sudo dnf update && upgrade
```

**Ver repositorios configurados:**
```bash
dnf repolist
```

**Buscar un paquete específico (Bashtop):**
```bash
dnf search bashtop
```

**Si no está disponible en los repositorios, clonar desde GitHub:**
```bash
sudo git clone https://github.com/aristocratos/bashtop.git
cd bashtop
./bashtop
```

**Eliminar archivos de configuración tras desinstalar:**
```bash
rm -rf /home/jtibrey/bashtop
```

**Limpiar caché y archivos residuales de dnf:**
```bash
sudo dnf clean all
```

---

## 2.2 — Automatización de Tareas con Crontab y AT

### Objetivo
Programar tareas recurrentes con `crontab` y tareas únicas con `at`.

### Comandos utilizados

**Editar el crontab del usuario actual:**
```bash
sudo crontab -e
```

**Tarea 1 — Actualizar el sistema todos los días a las 11:00 PM:**
```cron
0 23 * * * dnf update -y
```

**Tarea 2 — Reiniciar el sistema todos los domingos a las 3:00 AM:**
```cron
0 3 * * 0 /sbin/shutdown -f now
```

Guardar y salir del editor (vi):
```
Esc
:wq
```

**Habilitar el servicio AT (tareas de ejecución única):**
```bash
sudo systemctl start atd
sudo systemctl enable atd
```

**Ver contenido actual de archivos temporales:**
```bash
ls /tmp
```

**Programar un trabajo con AT (eliminar contenido de /tmp en 1 minuto):**
```bash
echo "rm -rf /tmp/*" | at now + 1 minute
```

**Verificar tras un minuto que la carpeta quedó vacía:**
```bash
ls /tmp
```

---

## 2.3 — Administración de Discos y Particiones

### Objetivo
Agregar un disco adicional a la VM, particionarlo, formatearlo y montarlo en distintas ubicaciones del sistema de archivos.

### Comandos utilizados

**Listar discos y particiones existentes:**
```bash
lsblk
```

**Crear una partición en el nuevo disco con `fdisk`:**
```bash
sudo fdisk /dev/nvme0n1
```

Dentro de la utilidad interactiva `fdisk`:
```
n        # nueva partición
p        # tipo primaria
[Enter]  # número de partición por defecto
[Enter]  # sector inicial por defecto
[Enter]  # sector final por defecto
w        # guardar cambios y salir
```

**Verificar que la partición fue creada:**
```bash
lsblk
```

**Formatear la partición en ext4:**
```bash
sudo mkfs.ext4 /dev/nvme0n1p1
```

**Crear un punto de montaje y montar el disco:**
```bash
cd Desktop
mkdir Julio
sudo mount /dev/nvme0n1p1 /home/jtibrey/Desktop/Julio
```

**Crear un archivo de prueba dentro del disco montado:**
```bash
cd Julio
sudo touch AdrianAlcantara.txt
```

**Desmontar el disco:**
```bash
cd ..
sudo umount /home/jtibrey/Desktop/Julio
```

**Montar el mismo disco en una ubicación distinta (`/mnt`):**
```bash
sudo mount /dev/nvme0n1p1 /mnt
```

**Verificar que el archivo persiste en la nueva ubicación:**
```bash
ls /mnt
```

---

## 📌 Conclusiones del Módulo

Este módulo demostró que los datos de una partición persisten independientemente del punto de montaje utilizado — el contenido no pertenece a la ruta, sino al sistema de archivos formateado en el disco. También se practicó la diferencia entre tareas recurrentes (`crontab`, basadas en patrones de tiempo) y tareas de ejecución única (`at`, basadas en un instante específico).

---

[⬅ Volver al índice general](../README.md)
