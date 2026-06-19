# 🔄 Módulo 5 — Alta Disponibilidad y Clustering

> **Sistema Operativo:** Rocky Linux (2 nodos)

> **Materia:** Sistemas Operativos III · ITLA

> **Lista de Reproducción:** <a href="https://www.youtube.com/playlist?list=PL1bMSHFyMPr7EdUOwUpbB-8D4kleaqhA_" target="_blank">
---

## 📋 Contenido del Módulo

| # | Práctica | Tema |
|---|----------|------|
| 5.1 | Sincronización de Carpetas con Rsync | Backups automatizados entre nodos vía SSH + Cron |
| 5.2 | Clúster de Alta Disponibilidad HTTP | Keepalived (VRRP) + Pacemaker/Corosync |

---

## 5.1 — Sincronización de Carpetas con Rsync

### Objetivo
Configurar sincronización automatizada de archivos entre dos servidores Rocky Linux mediante `rsync` sobre SSH, con autenticación por llaves y ejecución programada por `cron`.

### Entorno

| Servidor | Rol | IP |
|----------|-----|-----|
| Principal | Origen de los archivos | *(local)* |
| Secundario (clon) | Destino de la sincronización | `192.168.1.100` |

### Instalación

**En ambos servidores:**
```bash
sudo dnf install rsync
```

### Configuración de acceso SSH sin contraseña

**Generar par de llaves y copiar la pública al servidor secundario:**
```bash
ssh-keygen
ssh-copy-id jtibrey@192.168.1.100
```

### Preparación de los directorios de prueba

**En el servidor principal — crear carpeta con 100 archivos de prueba:**
```bash
mkdir ~/archivos
cd ~/archivos
touch Caleta{1..100}
ls
```

**En el servidor secundario — crear la carpeta destino:**
```bash
mkdir ~/sync
```

### Primera sincronización manual

**Desde el servidor principal:**
```bash
rsync -avz ~/archivo/ jtibrey@192.168.1.100:~/sync/
```

**Verificar en el servidor secundario:**
```bash
cd ~/sync
ls
```

### Automatización con script + cron

**Crear el script de sincronización:**
```bash
nano script.sh
```

**Contenido:**
```bash
#!/bin/bash
rsync -avz ~/archivos jtibrey@192.168.1.100:~/sync
```

**Dar permisos de ejecución:**
```bash
chmod +x ~/script.sh
```

**Programar la ejecución cada minuto vía crontab:**
```bash
crontab -e
```
```cron
* * * * * /home/jtibrey/script.sh
```

### Prueba de funcionamiento

**Crear archivos adicionales en el servidor principal:**
```bash
cd ~/archivos
touch SDnorte{1..10}
ls
```

**Tras un minuto, verificar en el servidor secundario:**
```bash
cd ~/sync
ls
```

Si los archivos `SDnorte1` a `SDnorte10` aparecen en `~/sync`, la sincronización automatizada funciona correctamente.

---

## 5.2 — Clúster de Alta Disponibilidad HTTP

### Objetivo
Construir un clúster activo-pasivo de dos nodos Apache, donde una **IP virtual flotante** migra automáticamente al nodo disponible en caso de falla, garantizando continuidad del servicio web.

### Entorno

| Servidor | Rol Keepalived | IP | Prioridad |
|----------|-----------------|-----|-----------|
| Servidor 1 | MASTER | `192.168.100.104` | 99 |
| Servidor 2 | BACKUP | `192.168.100.105` | 50 |
| — | IP Virtual (VRRP) | `192.168.100.201` | — |
| — | IP Virtual (Pacemaker) | `192.168.100.200` | — |

### Parte A — Clúster con Pacemaker + Corosync

**Habilitar repositorios de alta disponibilidad (en ambos servidores):**
```bash
sudo dnf install -y epel-release
sudo dnf config-manager --set-enabled ha
sudo dnf config-manager --enable ha
```

**Instalar Pacemaker y pcs:**
```bash
sudo dnf install -y pacemaker pcs
```

**Habilitar y arrancar el servicio de gestión del clúster:**
```bash
sudo systemctl enable pcsd
sudo systemctl start pcsd
```

**Configurar la contraseña del usuario `hacluster` (en ambos servidores):**
```bash
sudo passwd hacluster
```

**Permitir tráfico de alta disponibilidad en el firewall:**
```bash
sudo firewall-cmd --permanent --add-service=high-availability
sudo firewall-cmd --reload
```

### Autenticación y creación del clúster (ejecutar solo en un nodo)

**Autenticar los nodos entre sí:**
```bash
sudo pcs host auth 192.168.100.104 192.168.100.105 -u hacluster -p 12345678
```

**Crear el clúster:**
```bash
sudo pcs cluster setup cluster-jtibrey 192.168.100.104 192.168.100.105
```

**Habilitar e iniciar el clúster en todos los nodos:**
```bash
sudo pcs cluster enable --all
sudo pcs cluster start --all
```

**Verificar el estado del clúster:**
```bash
sudo pcs status
```

**Deshabilitar STONITH (opcional, para entorno de laboratorio):**
```bash
sudo pcs property set stonith-enabled=false
```

> ⚠️ STONITH ("Shoot The Other Node In The Head") es un mecanismo de protección que aísla nodos con fallas para evitar split-brain. Se deshabilita aquí solo por simplicidad de laboratorio — **en producción debe permanecer activo**.

### Configuración de la IP virtual gestionada por Pacemaker

```bash
sudo pcs resource create IP_Flotante ocf:heartbeat:IPaddr2 \
  ip=192.168.100.200 cidr_netmask=24 op monitor interval=30s
```

### Prueba de failover

**Desde la máquina host, hacer ping continuo a la IP virtual:**
```bash
ping 192.168.100.200
```

Mientras el ping está activo, **reiniciar los servidores de forma intercalada**. Si el ping no se interrumpe, Pacemaker migró correctamente la IP virtual al nodo disponible — confirmando que el failover funciona.

---

### Parte B — Configuración de Keepalived (VRRP)

**Instalar en ambos servidores:**
```bash
sudo dnf install keepalived
```

**Reemplazar el archivo de configuración:**
```bash
sudo rm /etc/keepalived/keepalived.conf
sudo nano /etc/keepalived/keepalived.conf
```

**Configuración del nodo MASTER (Servidor 1):**
```conf
vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 51
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 12345
    }
    virtual_ipaddress {
        192.168.100.201
    }
}
```

**Configuración del nodo BACKUP (Servidor 2):**
```conf
vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 50
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 12345
    }
    virtual_ipaddress {
        192.168.100.201
    }
}
```

**Iniciar y habilitar Keepalived (en ambos nodos):**
```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
sudo systemctl status keepalived
sudo systemctl restart httpd
```

> 💡 **Concepto clave (VRRP):** El nodo con mayor `priority` toma el rol MASTER y posee la IP virtual. Si falla, el nodo BACKUP detecta la ausencia de anuncios VRRP y asume automáticamente la IP virtual.

---

### Parte C — Instalación de Apache en ambos nodos

```bash
sudo dnf install httpd
```

**Crear página de identificación por servidor:**
```bash
# Servidor 1
echo "<h1>Servidor 1 - IP: 192.168.100.104</h1>" | sudo tee /var/www/html/index.html

# Servidor 2
echo "<h1>Servidor 2 - IP: 192.168.100.105</h1>" | sudo tee /var/www/html/index.html
```

**Asignar propietario correcto:**
```bash
sudo chown jtibrey:jtibrey /var/www/html/index.html
```

**Habilitar HTTP en el firewall:**
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

**Habilitar y arrancar Apache:**
```bash
sudo systemctl enable httpd
sudo systemctl start httpd
sudo systemctl status httpd
```


## 📌 Conclusiones del Módulo

Este módulo demostró dos enfoques complementarios de alta disponibilidad: **Rsync + Cron** para redundancia de datos (backup asíncrono), y **Keepalived/Pacemaker** para redundancia de servicio (failover de IP en tiempo real). La diferencia clave es que Rsync sincroniza *contenido*, mientras que Keepalived y Pacemaker garantizan que el *servicio* permanezca accesible incluso ante la caída total de un nodo — son capas distintas de resiliencia que normalmente se combinan en producción.

---

[⬅ Volver al índice general](../README.md)
