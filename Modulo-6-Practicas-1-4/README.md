# 🔐 Módulo 6 — Seguridad en Linux

> **Sistema Operativo:** Rocky Linux

> **Materia:** Sistemas Operativos III · ITLA

> ⚠️ Prácticas 6.3 y 6.4 realizadas en **Rocky Linux 8** (compatibilidad de paquetes)

> **Lista de reproducción:** https://www.youtube.com/playlist?list=PL1bMSHFyMPr78vdl2Gh_V7804hbPx44Vx
---

## 📋 Contenido del Módulo

| # | Práctica | Tema |
|---|----------|------|
| 6.1 | Cifrado GPG2 | Cifrado simétrico de archivos |
| 6.2 | IPTables y firewall-cmd | Filtrado de tráfico, comparación de herramientas |
| 6.3 | IDS Snort | Compilación desde código fuente, reglas de detección |
| 6.4 | 2FA con Google Authenticator | PAM + SSH, autenticación de doble factor |

---

## 6.1 — Cifrado de Archivos con GPG2

### Objetivo
Cifrar y descifrar un archivo de texto de forma simétrica utilizando GPG2.

### Comandos utilizados

**Instalar la herramienta:**
```bash
sudo dnf install gnupg2 -y
```

**Crear un archivo de prueba:**
```bash
echo "Julio Cesar Hernandez" > Prueba.txt
ls -l
cat Prueba.txt
```

**Cifrar el archivo de forma simétrica:**
```bash
gpg2 -c Prueba.txt
```
> El sistema solicita una contraseña dos veces. Esto genera `Prueba.txt.gpg`.

**Verificar que ambos archivos existen:**
```bash
ls
```

**Intentar leer el archivo cifrado (no será legible):**
```bash
cat Prueba.txt.gpg
```

**Descifrar el archivo con la contraseña original:**
```bash
gpg2 -d Prueba.txt.gpg
```
---

## 6.2 — IPTables vs. firewall-cmd

### Objetivo
Comparar dos mecanismos de filtrado de tráfico en Linux — `iptables` (reglas de bajo nivel) y `firewall-cmd` (gestor de alto nivel basado en zonas) — permitiendo y bloqueando tráfico HTTP, FTP y SSH.

### Preparación del entorno

**Detener firewalld para trabajar con iptables directamente:**
```bash
sudo systemctl stop firewalld
```

**Instalar los servicios a probar:**
```bash
sudo dnf install httpd vsftpd openssh-server -y
sudo systemctl start httpd vsftpd sshd
sudo systemctl enable httpd vsftpd sshd
```

**Verificar estado de los servicios:**
```bash
systemctl status httpd
systemctl status vsftpd
systemctl status sshd
```

### Parte A — Gestión con IPTables

**Permitir tráfico en los puertos 80, 21 y 22:**
```bash
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

**Eliminar las reglas de aceptación y bloquear el tráfico (REJECT):**
```bash
sudo iptables -D INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -D INPUT -p tcp --dport 22 -j ACCEPT

sudo iptables -A INPUT -p tcp --dport 80 -j REJECT
sudo iptables -A INPUT -p tcp --dport 21 -j REJECT
sudo iptables -A INPUT -p tcp --dport 22 -j REJECT
```

**Revertir el bloqueo y permitir el tráfico nuevamente:**
```bash
sudo iptables -D INPUT -p tcp --dport 80 -j REJECT
sudo iptables -D INPUT -p tcp --dport 21 -j REJECT
sudo iptables -D INPUT -p tcp --dport 22 -j REJECT

sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 21 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
```

### Parte B — Gestión con firewall-cmd

**Reactivar firewalld:**
```bash
sudo systemctl start firewalld
```

**Bloquear los servicios:**
```bash
sudo firewall-cmd --permanent --remove-service=http
sudo firewall-cmd --permanent --remove-service=ftp
sudo firewall-cmd --permanent --remove-service=ssh
sudo firewall-cmd --reload
```

**Permitir los servicios nuevamente:**
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=ftp
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --reload
```

> 💡 **Concepto clave:** `iptables` opera con reglas explícitas en cadenas (INPUT/OUTPUT/FORWARD) y requiere conocimiento manual del orden de evaluación. `firewall-cmd` abstrae esa complejidad mediante **zonas** y **servicios predefinidos**, siendo más mantenible en producción — por eso es el firewall por defecto en RHEL/Rocky desde hace varias versiones.

---

## 6.3 — Instalación y Configuración de IDS Snort

### Objetivo
Compilar **Snort 3** desde el código fuente en Rocky Linux y configurarlo como sistema de detección de intrusos (IDS), generando alertas en tiempo real sobre tráfico ICMP, HTTP, SSH y FTP.

> ⚠️ Práctica realizada en **Rocky Linux 8** por compatibilidad de dependencias de compilación.

### Habilitar repositorios necesarios

```bash
sudo dnf config-manager --set-enabled powertools   # RHEL 8
sudo dnf config-manager --set-enabled crb           # RHEL 9
```

### Configurar rutas de librerías locales

```bash
sudo nano /etc/ld.so.conf.d/local.conf
```
```
/usr/local/lib
/usr/local/lib64
```

### Instalación de dependencias de compilación

```bash
sudo dnf install -y gcc flex bison zlib zlib-devel libpcap libpcap-devel \
pcre pcre-devel tcpdump openssl openssl-devel \
hwloc hwloc-devel cmake cmake3 git wget make autoconf automake \
libtool libnet libnet-devel libyaml libyaml-devel file-devel \
doxygen rpm-build libmnl libmnl-devel nano which

sudo dnf groupinstall -y "Development Tools"
sudo dnf groupinstall -y "Development Libraries"
```

**Verificación adicional de paquetes (lista extendida):**
```bash
sudo dnf install flex bison gcc gcc-c++ make cmake autoconf libtool git nano \
unzip wget libpcap-devel pcre-devel libdnet-devel hwloc-devel openssl-devel \
zlib-devel luajit-devel pkgconfig pkgconf libunwind-devel libnfnetlink-devel \
libnetfilter_queue-devel libmnl-devel xz-devel gperftools libuuid-devel \
hyperscan hyperscan-devel -y
```

### Compilar e instalar libdnet

```bash
cd /tmp
wget https://github.com/ofalk/libdnet/archive/refs/tags/libdnet-1.16.tar.gz
tar -xvzf libdnet-1.16.tar.gz
cd libdnet-libdnet-1.16
./configure --prefix=/usr
make
sudo make install
```

### Compilar e instalar libdaq

```bash
git clone https://github.com/snort3/libdaq.git
cd libdaq/
./bootstrap
./configure && make && sudo make install
sudo ldconfig
cd ../
```

### Compilar e instalar Snort 3

```bash
git clone https://github.com/snort3/snort3.git
cd snort3

export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH
export PKG_CONFIG_PATH=/usr/local/lib64/pkgconfig:$PKG_CONFIG_PATH
export CFLAGS="-O3"
export CXXFLAGS="-O3 -fno-rtti"

./configure_cmake.sh --prefix=/usr/local/snort --enable-tcmalloc

cd build/
make -j$(nproc)
sudo make -j$(nproc) install
cd ../../
sudo ldconfig
```

**Crear enlace simbólico y verificar instalación:**
```bash
sudo ln -s /usr/local/snort/bin/snort /usr/bin/snort
snort -V
```

### Solución de problemas (si `snort -V` falla)

```bash
# Verificar binario
ls -la /usr/local/snort/bin/snort

# Verificar enlace simbólico
ls -la /usr/bin/snort

# Verificar PATH
echo $PATH

# Recrear enlaces simbólicos
sudo ln -sf /usr/local/snort/bin/snort /usr/local/bin/snort
sudo ln -sf /usr/local/snort/bin/snort /usr/bin/snort

# Agregar al PATH del sistema
echo 'export PATH=/usr/local/snort/bin:$PATH' | sudo tee -a /etc/profile.d/snort.sh
source /etc/profile.d/snort.sh

# Verificar
snort -V
```

### Configuración de Snort

**Definir la red local a monitorear:**
```bash
sudo nano /usr/local/snort/etc/snort/snort.lua
```
```lua
HOME_NET = '10.0.0.24/24'   -- IP y máscara del entorno
```

**Validar la configuración:**
```bash
snort -T -c /usr/local/snort/etc/snort/snort.lua
```

**Activar modo promiscuo en la interfaz de red:**
```bash
sudo ip link set dev ens160 promisc on
```

### Reglas de detección personalizadas

```bash
sudo nano /usr/local/snort/etc/snort/local.rules
```

| Regla | Detecta |
|-------|---------|
| `alert icmp any any -> any any (msg:"ICMP Traffic Detected"; sid:10000001; metadata:policy security-ips alert;)` | Tráfico ICMP (ping) |
| `alert tcp any any -> any 80 (msg:"HTTP Traffic Detected"; flow:to_server; sid:1000002; rev:1;)` | Tráfico HTTP |
| `alert tcp any any -> any 22 (msg:"SSH Traffic Detected"; flow:to_server; sid:1000003; rev:1;)` | Tráfico SSH |
| `alert tcp any any -> any 8080 (msg:"HTTP Alternative Port Traffic"; flow:to_server; sid:1000004; rev:1;)` | HTTP en puerto alternativo |
| `alert tcp any any -> any 21 (msg:"FTP Traffic Detected"; flow:to_server; sid:1000005; rev:1;)` | Tráfico FTP |

### Ejecución del IDS

```bash
sudo snort -c /usr/local/snort/etc/snort/snort.lua \
  -R /usr/local/snort/etc/snort/local.rules \
  -i ens160 -A alert_fast -s 65535 -k none
```

---

## 6.4 — Configurar 2FA con Google Authenticator para SSH

### Objetivo
Implementar autenticación de doble factor (2FA) para conexiones SSH combinando contraseña + código TOTP de Google Authenticator, usando el módulo PAM.

### Instalación de SSH y dependencias

```bash
sudo dnf install openssh-server
sudo systemctl start sshd
sudo systemctl enable sshd
sudo systemctl status sshd

sudo dnf install epel-release -y
sudo dnf install google-authenticator qrencode qrencode-libs
```

### Configurar Google Authenticator para el usuario

```bash
google-authenticator
```
> Genera un código QR y claves de respaldo. Se debe escanear el QR con la app Google Authenticator en el móvil y aceptar las configuraciones de seguridad recomendadas.

### Integración con PAM

```bash
sudo nano /etc/pam.d/sshd
```
**Agregar la línea:**
```
auth required pam_google_authenticator.so
```

### Configuración de SSH para exigir 2FA

```bash
sudo nano /etc/ssh/sshd_config
```
**Agregar/verificar:**
```
PasswordAuthentication yes
ChallengeResponseAuthentication yes
UsePAM yes
```

### Ajustes de contexto SELinux y permisos

```bash
sudo semanage fcontext -a -t auth_home_t "/home/jtibrey(/.*)?"
sudo restorecon -Rv /home/jtibrey

sudo chown jtibrey:jtibrey /home/jtibrey/.google_authenticator
sudo chmod 600 /home/jtibrey/.google_authenticator
sudo chmod 700 /home/jtibrey
sudo chown jtibrey:jtibrey /home/jtibrey
```

**Reiniciar el servicio SSH:**
```bash
sudo systemctl restart sshd
```

### Prueba de conexión con 2FA

```bash
ssh jtibrey@10.0.0.24
```
El sistema solicita primero la **contraseña** y luego el **código de Google Authenticator**.

> 💡 **Concepto clave:** PAM (Pluggable Authentication Modules) permite encadenar múltiples factores de autenticación sin modificar el código de las aplicaciones — `sshd` no "sabe" que está pidiendo 2FA, simplemente delega la decisión de autenticación a la pila de PAM configurada.

---

## 📌 Conclusiones del Módulo

Este módulo cubrió cuatro capas distintas de seguridad en Linux: **confidencialidad de datos en reposo** (GPG2), **control de acceso a nivel de red** (iptables/firewall-cmd), **detección de intrusos** (Snort, compilado manualmente para entender su arquitectura interna), y **autenticación robusta** (2FA vía PAM). En conjunto representan defensa en profundidad: ningún mecanismo por sí solo es suficiente, pero combinados reducen significativamente la superficie de ataque de un servidor Linux.

---

[⬅ Volver al índice general](../README.md)
