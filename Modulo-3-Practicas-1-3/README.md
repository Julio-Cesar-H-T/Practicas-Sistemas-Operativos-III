# ⚙️ Módulo 3 — Tareas Avanzadas y Administración de Servicios de Red

> **Sistema Operativo:** Rocky Linux

> **Materia:** Sistemas Operativos III · ITLA

> **Lista de Reproducción:** https://www.youtube.com/playlist?list=PL1bMSHFyMPr4p5LDMDzXb1sElo7lkbdA7
---

## 📋 Contenido del Módulo

| # | Práctica | Tema |
|---|----------|------|
| 3.1 | Edición de GRUB2 y Recuperación de Contraseña Root | Modo de recuperación, SELinux |
| 3.2 | Shell Scripting | Bash scripting, automatización |
| 3.3 | Servicio SSH | Acceso remoto, autenticación por llaves |

---

## 3.1 — Edición de GRUB2 y Recuperación de Contraseña Root

### Objetivo
Modificar el tiempo de espera del menú GRUB2 y realizar una recuperación de contraseña root mediante el modo de recuperación del sistema.

### Procedimiento

**Editar el archivo de configuración de GRUB2:**
```bash
sudo nano /etc/default/grub
```

Se localiza la línea `GRUB_TIMEOUT=5` y se modifica el valor (ej. a `20` segundos). Guardar con `Ctrl + O` y salir con `Ctrl + X`.

**Aplicar la nueva configuración:**
```bash
sudo grub2-mkconfig -o /boot/grub2/grub.cfg
```

### Recuperación de contraseña root

1. Reiniciar la máquina y presionar **`e`** apenas aparezca el menú GRUB.
2. Ubicar la línea que comienza con `linux /vmlinuz-...`.
3. Cambiar `ro` por `rw`, e ir al final de la línea para agregar:
   ```
   init=/bin/bash
   ```
4. Presionar `Ctrl + X` para arrancar con la configuración modificada.

**Una vez dentro del sistema en modo bash root, cambiar la contraseña:**
```bash
passwd
```

**(Opcional pero recomendado) Forzar el reetiquetado de contexto SELinux en el próximo arranque:**
```bash
touch /.autorelabel
```

**Salir del modo de recuperación y continuar el arranque normal:**
```bash
exec /sbin/init
```

> ⚠️ **Nota de seguridad:** Este procedimiento demuestra por qué la seguridad física del servidor es tan crítica como la seguridad lógica — cualquier persona con acceso a la consola de arranque puede resetear la contraseña root sin credenciales previas. La mitigación estándar es proteger el propio GRUB con contraseña.

---

## 3.2 — Shell Scripting en Rocky Linux

### Objetivo
Crear dos scripts Bash funcionales: uno para respaldos automatizados y otro interactivo para capturar configuración de red.

### Script 1 — Backup de directorio home con fecha y hora

**Crear el script:**
```bash
nano backup_home.sh
```

**Contenido:**
```bash
#!/bin/bash
FECHA=$(date +"%d-%m-%Y:%H:%M")
tar -cvzf /home/$(whoami)/backup_home_$FECHA.tar.gz /home/$(whoami)
```

**Dar permisos de ejecución y correr:**
```bash
chmod +x backup_home.sh
./backup_home.sh
```

**Verificar el resultado:**
```bash
cd /home/usuario
ls -l
```

### Script 2 — Captura de configuración de red de forma interactiva

**Crear el script:**
```bash
sudo nano Iftest.sh
```

**Contenido:**
```bash
#!/bin/bash
read -p "Ingrese el nombre del archivo: " nombre
ifconfig > /home/$(whoami)/Desktop/"$nombre".txt
echo "El archivo se ha guardado en el escritorio con el nombre $nombre.txt"
```

**Dar permisos de ejecución y correr:**
```bash
sudo chmod +x Iftest.sh
./Iftest.sh
```

El script solicita un nombre (ej. `Iftest`) y guarda la salida de `ifconfig` en `Iftest.txt` dentro del escritorio.

**Verificar el archivo generado:**
```bash
cd Desktop
ls -l
cat Iftest.txt
```

---

## 3.3 — Servicio SSH entre Rocky Linux y Windows

### Objetivo
Configurar acceso remoto SSH entre una VM Rocky Linux y una máquina física Windows, incluyendo autenticación sin contraseña mediante llaves SSH.

### Instalación y activación del servicio (en Rocky Linux)

```bash
sudo dnf install openssh-server
sudo systemctl start sshd
sudo systemctl enable sshd
```

**Verificar el estado del servicio:**
```bash
sudo systemctl status sshd
```

### Conexión remota desde Windows hacia Rocky Linux

```bash
ssh jtibrey@10.0.0.45
```

Tras ingresar la contraseña del usuario Linux, se puede operar la terminal remota:
```bash
ls
ls -l
```

**Cerrar la sesión:**
```bash
exit
```

### Autenticación sin contraseña mediante llaves SSH

**Generar el par de llaves (desde Windows):**
```bash
ssh-keygen
```

**Copiar la llave pública generada:**
```bash
notepad C:/users/usuario/direccion_de_la_llave
```

**En Rocky Linux, preparar el directorio de llaves autorizadas:**
```bash
cd .ssh        # verificar si existe
mkdir .ssh     # si no existe
cd .ssh
sudo nano authorized_keys
```

Se pega la llave pública copiada desde Windows, se guarda y se cierra el editor.

**Volver a conectar desde Windows:**
```bash
ssh jtibrey@10.0.0.45
```

Si la llave fue correctamente configurada, la conexión se establece **sin solicitar contraseña**.

---

## 📌 Conclusiones del Módulo

Este módulo cubrió tres pilares de la administración avanzada de Linux: recuperación de acceso ante pérdida de credenciales (con sus implicaciones de seguridad física), automatización mediante shell scripting, y acceso remoto seguro mediante SSH con autenticación basada en llaves asimétricas — un mecanismo significativamente más seguro que la autenticación por contraseña.

---

[⬅ Volver al índice general](../README.md)
