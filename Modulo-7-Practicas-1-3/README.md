# 🗄️ Módulo 7 — Controlador de Dominios y Compartir Archivos

> **Sistema Operativo:** Rocky Linux (servidor) · Ubuntu / Windows (clientes)

> **Materia:** Sistemas Operativos III · ITLA

> **Lista de Reproducción:** https://www.youtube.com/playlist?list=PL1bMSHFyMPr59lTYmFjlptwAy6hSr_9y-
---

## 📋 Contenido del Módulo

| # | Práctica | Tema |
|---|----------|------|
| 7.1 | Compartir archivos entre Linux con NFS | Servidor/cliente NFS, montaje persistente |
| 7.2 | File Server compatible con Windows (SAMBA) | Recurso compartido SMB, autenticación |
| 7.3 | Controlador de Dominio con cliente Windows | Samba AD DC, Kerberos, DNS, unión de dominio |

---

## 7.1 — Compartir Archivos entre Linux Utilizando NFS

### Objetivo
Configurar un servidor NFS en Rocky Linux y montar el recurso compartido desde un cliente Ubuntu, incluyendo montaje persistente vía `/etc/fstab`.

### Configuración del servidor (Rocky Linux)

**Instalar NFS:**
```bash
sudo dnf install nfs-utils -y
```

**Iniciar y habilitar el servicio:**
```bash
sudo systemctl start nfs-server
sudo systemctl enable nfs-server
sudo systemctl status nfs-server
```

**Crear el directorio compartido con archivos de prueba:**
```bash
sudo mkdir -p /var/nfs/os3
cd /var/nfs/os3
sudo touch Adrian{1..100}.txt
```

**Asignar permisos de acceso:**
```bash
sudo chown nobody:nobody /var/nfs/os3
sudo chmod 755 /var/nfs/os3
```

**Configurar el archivo de exportación:**
```bash
sudo nano /etc/exports
```
```
/var/nfs/os3 10.0.0.24/24(rw,sync,no_subtree_check)
```

**Aplicar la exportación:**
```bash
sudo exportfs -rav
```

**Permitir NFS en el firewall:**
```bash
sudo firewall-cmd --permanent --add-service=nfs
sudo firewall-cmd --reload
```

### Configuración del cliente (Ubuntu)

**Instalar el cliente NFS:**
```bash
sudo apt update
sudo apt install nfs-common -y
```

**Crear el punto de montaje:**
```bash
sudo mkdir -p /mnt/os3
```

**Montar manualmente el recurso:**
```bash
sudo mount -t nfs 10.0.0.24:/var/nfs/os3 /mnt/os3
df -h
```

**Hacer el montaje persistente entre reinicios:**
```bash
sudo nano /etc/fstab
```
```
10.0.0.24:/var/nfs/os3 /mnt/os3 nfs defaults 0 0
```

**Verificar el contenido montado:**
```bash
ls /mnt/os3
```

**Confirmar persistencia tras reinicio:**
```bash
sudo reboot
df -h
ls /mnt/os3
```

---

## 7.2 — File Server Compatible con Windows Utilizando SAMBA

### Objetivo
Crear un recurso compartido SMB en Rocky Linux accesible desde un cliente Windows como unidad de red, con autenticación de usuario.

### Configuración del servidor (Rocky Linux)

**Instalar Samba:**
```bash
sudo dnf install samba samba-client samba-common -y
```

**Habilitar e iniciar los servicios SMB/NMB:**
```bash
sudo systemctl enable --now smb nmb
```

**Crear el directorio compartido con contexto SELinux correcto:**
```bash
sudo mkdir -p /srv/samba/public
sudo chcon -t samba_share_t /srv/samba/public
cd /srv/samba/public
sudo touch Adrian{1..100}.txt
sudo chmod -R 777 /srv/samba/public
sudo chown -R nobody:nobody /srv/samba/public
```

**Permitir Samba en el firewall:**
```bash
sudo firewall-cmd --add-service=samba --permanent
sudo firewall-cmd --reload
```

**Configurar el recurso compartido:**
```bash
sudo nano /etc/samba/smb.conf
```
```ini
[public]
   path = /srv/samba/public
   browsable = yes
   writable = yes
   guest ok = yes
   read only = no
   force user = nobody
```

**Crear grupo y usuario para autenticación Samba:**
```bash
sudo groupadd sambausers
sudo useradd -M -s /sbin/nologin Julio
sudo smbpasswd -a Julio
sudo usermod -aG sambausers Julio
```

**Reiniciar los servicios:**
```bash
sudo systemctl restart smb nmb
```

### Conexión desde el cliente Windows

1. **File Explorer → This PC → Map network drive**
2. Ingresar la ruta: `\\<IP_DEL_SERVIDOR>\public`
3. Marcar **"Connect using different credentials"** e ingresar las credenciales creadas en el servidor (`Julio`)
4. Finalizar — el recurso queda montado como unidad de red

**Prueba de escritura bidireccional:**
Editar `Adrian99.txt` desde Windows, guardar, y verificar el cambio desde el servidor:
```bash
cd /srv/samba/public
cat Adrian99.txt
```

---

## 7.3 — Controlador de Dominio con Cliente Windows (Samba AD DC)

### Objetivo
Compilar Samba desde código fuente y desplegarlo como **controlador de dominio Active Directory**, integrando un cliente Windows al dominio mediante Kerberos y DNS.

> ⚠️ Práctica de mayor complejidad: requiere compilación manual de Samba con soporte AD DC, configuración de DNS interno y Kerberos.

### 1. Preparación del sistema

**Configurar el hostname** (no puede coincidir con el nombre del dominio):
```bash
hostnamectl set-hostname SO3.inet
echo "10.0.0.24 SO3.inet" >> /etc/hosts
```

**Verificar:**
```bash
hostname
hostname -f
```

**Actualizar el sistema e instalar dependencias base:**
```bash
dnf -y update
dnf -y install epel-release
sudo dnf config-manager --set-enabled powertools
dnf update
sudo dnf makecache
dnf -y install wget tar gcc make python3-devel
```

### 2. Descarga y compilación de Samba

```bash
mkdir -p /samba && cd /samba
wget https://download.samba.org/pub/samba/stable/samba-4.16.2.tar.gz
tar -zxvf samba-4.16.2.tar.gz && cd samba-4.16.2/
```

**Instalar dependencias de compilación:**
```bash
sudo dnf -y install \
  docbook-style-xsl python3-markdown bison dbus-devel flex gdb \
  gnutls-devel jansson-devel keyutils-libs-devel krb5-workstation \
  libacl-devel libaio-devel libarchive-devel libattr-devel libblkid-devel \
  libtasn1 libtasn1-tools libxml2-devel libxslt lmdb-devel openldap-devel \
  pam-devel perl perl-ExtUtils-MakeMaker perl-Parse-Yapp popt-devel \
  python3-cryptography python3-dns python3-gpg readline-devel rpcgen \
  systemd-devel zlib-devel perl-JSON gpgme-devel screen
```

**Compilar e instalar:**
```bash
./configure --prefix=/usr/local/samba --enable-fhs --with-ads --with-systemd
make -j$(nproc)
make install
```

**Agregar Samba al PATH:**
```bash
export PATH=/usr/local/samba/bin:/usr/local/samba/sbin:$PATH
```

```bash
nano ~/.bash_profile
```
```
PATH=$PATH:$HOME/bin:/usr/local/samba/bin:/usr/local/samba/sbin:$PATH
export PATH
```

### 3. Aprovisionamiento del dominio

> ⚠️ Verificar que `interfaces=` apunte al nombre real de la interfaz de red (`ip a` para confirmarlo).

```bash
samba-tool domain provision --use-rfc2307 --interactive \
  --option="interfaces= lo ens160" \
  --option="bind interfaces only=yes"
```

**Parámetros utilizados durante el aprovisionamiento:**

| Parámetro | Valor |
|-----------|-------|
| Dominio | `OS3.inet` |
| Nombre de dominio (NetBIOS) | `OS3` |
| Nombre de controlador | `dc` |
| Backend DNS | `SAMBA_INTERNAL` |
| Contraseña | *(mínimo 7 caracteres, con complejidad)* |

**Iniciar el servicio Samba:**
```bash
samba
```
> Si el sistema se reinicia, debe iniciarse con la ruta absoluta: `sudo /usr/local/samba/sbin/samba`

### 4. Verificación de DNS

```bash
ping google.com
ping OS3.inet
ping dc1.inet
```

### 5. Configuración de Kerberos

```bash
cp /usr/local/samba/var/lib/samba/private/krb5.conf /etc/krb5.conf
```

### 6. Reglas de firewall para AD DC

```bash
firewall-cmd --permanent --add-service={ldap,ldaps,kerberos,dns}
firewall-cmd --permanent --add-port={53,88,135,139,389,445,464,636,3268,3269}/tcp
firewall-cmd --permanent --add-port={53,88,123,138,389,464}/udp
firewall-cmd --reload
```

### 7. Configuración de DNS local del servidor

```bash
sudo nmcli con mod ens160 ipv4.dns 10.0.0.24
sudo nmcli con mod ens160 ipv4.ignore-auto-dns yes
sudo nmcli con up ens160
```

### 8. Registro DNS para el cliente Windows

```bash
samba-tool dns add 10.0.0.24 OS3.inet www A 10.0.0.25 -U Administrator
```

### 9. Crear usuario de dominio

```bash
sudo /usr/local/samba/bin/samba-tool user create lanegracubana "Julio20250702" \
  --given-name="La Negra" --surname="Cubana" --mail=lanegracubana@so3.inet
```

### 10. Unión del cliente Windows al dominio

**Configurar el DNS de la tarjeta de red en Windows:**
`Win + R → ncpa.cpl → [adaptador] → Propiedades → IPv4 → Propiedades`
Cambiar **DNS preferido** a la IP del servidor Linux; dejar el alternativo vacío.

**Limpiar caché DNS:**
```cmd
ipconfig /flushdns
```

**Verificar resolución del dominio:**
```cmd
ping OS3.inet
ping www.OS3.inet
```

**Unir la máquina al dominio:**
`Win + R → sysdm.cpl → Nombre del equipo → Cambiar → Miembro de un dominio: os3.inet`

Tras el reinicio solicitado, iniciar sesión con:
- **Usuario:** `OS3\Administrator` *(usar el formato `DOMINIO\usuario`, no solo `Administrator`)*
- **Contraseña:** la configurada durante el aprovisionamiento

---

## 🛠️ Troubleshooting

### Error: "Domain must not be equal to short host name"

```
ERROR(<class 'samba.provision.ProvisioningError'>): Provision failed -
ProvisioningError: guess_names: Domain 'OS3' must not be equal to short host name 'OS3'!
```

**Causa:** el hostname del servidor coincide con el nombre corto del dominio.
**Solución:** cambiar el hostname (`hostnamectl` + `/etc/hosts`), eliminar la configuración de aprovisionamiento fallida, y volver a ejecutar `domain provision`.

### Contraseña de administrador olvidada

```bash
samba-tool user setpassword [USUARIO]
```

### Inicio de sesión sigue fallando con credenciales correctas

```bash
# Abrir rango de puertos dinámicos RPC
sudo firewall-cmd --permanent --add-port=49152-65535/tcp
sudo firewall-cmd --reload

# Poner SELinux en modo permisivo temporalmente (diagnóstico)
sudo setenforce 0
sestatus
```

```powershell
# Desactivar el firewall de Windows temporalmente (diagnóstico)
Set-NetFirewallProfile -Profile Domain,Private,Public -Enabled False
```

> ⚠️ Estas medidas (SELinux permisivo, firewall desactivado) son **solo para diagnóstico en laboratorio** — nunca deben dejarse así en un entorno de producción.

---

## 📌 Conclusiones del Módulo

Este módulo cubrió tres niveles de compartición de recursos en red: **NFS** (nativo de Unix, ideal entre servidores Linux), **SAMBA standalone** (interoperabilidad simple con Windows vía SMB), y **Samba AD DC** (infraestructura de dominio completa con Kerberos y DNS integrado, replicando la funcionalidad de un Active Directory de Windows Server sobre Linux). La compilación manual de Samba con soporte AD DC permitió entender los componentes internos que normalmente quedan ocultos en una instalación por paquete binario.

---

[⬅ Volver al índice general](../README.md)
