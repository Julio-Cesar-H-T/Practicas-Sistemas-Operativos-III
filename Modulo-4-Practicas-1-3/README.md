# 🌐 Módulo 4 — Administración de Servicios de Red (Parte 2)

> **Sistema Operativo:** Rocky Linux
> **Materia:** Sistemas Operativos III · ITLA
> **Lista de Reproducción:** https://www.youtube.com/playlist?list=PL1bMSHFyMPr6jFAfxDsYgejmvWuqomW-G
---

## 📋 Contenido del Módulo

| # | Práctica | Tema |
|---|----------|------|
| 4.1 | Servidor HTTP Apache2 | Virtual Hosts, múltiples sitios y puertos |
| 4.2 | Servidor de Correos | Postfix con relay SMTP (Gmail) |
| 4.3 | Servidor de Impresión | CUPS, impresora PDF compartida con Windows |

---

## 4.1 — Instalar un Servidor HTTP Apache2

### Objetivo
Desplegar dos sitios web estáticos independientes en el mismo servidor Apache, diferenciados por nombre de dominio (Virtual Host) y por puerto.

### Instalación y arranque del servicio

```bash
sudo dnf install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
sudo systemctl status httpd
```

### Sitio 1 — "Hola Mundo" (puerto 80)

**Crear el directorio del sitio:**
```bash
cd /var/www/html
sudo mkdir hola_mundo
cd hola_mundo
sudo nano index.html
```

**Contenido del HTML:**
```html
<!DOCTYPE html>
<html>
<head><title>Hola Mundo</title></head>
<body><h1>Hola Mundo</h1></body>
</html>
```

**Configurar el Virtual Host en Apache:**
```bash
sudo nano /etc/httpd/conf/httpd.conf
```

Verificar que exista `Listen 80` y agregar al final del archivo:
```apache
<VirtualHost *:80>
     ServerName hola.localhost
     DocumentRoot "/var/www/html/hola_mundo"
     ServerAdmin root@localhost
</VirtualHost>
```

**Reiniciar Apache:**
```bash
sudo systemctl restart httpd
```

**(Opcional) Resolver el nombre localmente:**
```bash
sudo nano /etc/hosts
# Agregar: 127.0.0.1    hola.localhost
```

**Verificación:** acceder desde el navegador a `http://hola.localhost`

### Sitio 2 — Página de estudiante (puerto 8080)

**Crear el directorio y el HTML:**
```bash
cd /var/www/html
sudo mkdir estudiante
cd estudiante
sudo nano index.html
```

**Contenido del HTML:**
```html
<!DOCTYPE html>
<html>
<head><title>Estudiante</title></head>
<body>
<h2>Julio Cesar Hernandez Tibrey</h2>
<p>Matrícula: 2025-0702</p>
<p>Materia: Sistemas Operativos III<br>
Docente: Adrian Alcántara</p>
</body>
</html>
```

**Configurar Apache para escuchar en el puerto 8080 y agregar el segundo Virtual Host:**
```bash
sudo nano /etc/httpd/conf/httpd.conf
```

```apache
Listen 8080

<VirtualHost *:8080>
     ServerName estudiante.localhost
     DocumentRoot "/var/www/html/estudiante"
     ServerAdmin root@localhost
</VirtualHost>
```

**Reiniciar Apache:**
```bash
sudo systemctl restart httpd
```

**(Opcional) Resolver el nombre localmente:**
```bash
sudo nano /etc/hosts
# Agregar: 127.0.0.1    estudiante.localhost
```

**Verificación:** acceder desde el navegador a `http://estudiante.localhost:8080`


> 💡 **Concepto clave:** Este laboratorio demuestra dos formas distintas de hospedar múltiples sitios en un solo servidor Apache — **Virtual Hosting por nombre** (mismo puerto, distinto `ServerName`) y **separación por puerto** (`Listen` adicional + Virtual Host dedicado).

---

## 4.2 — Instalar un Servidor de Correos

### Objetivo
Configurar **Postfix** como agente de transferencia de correo (MTA), utilizando Gmail como relay SMTP autenticado para el envío de correos salientes.

### Instalación y arranque del servicio

```bash
sudo dnf install postfix -y
sudo systemctl enable postfix
sudo systemctl start postfix
sudo systemctl status postfix
```

### Apertura de puertos en el firewall

```bash
sudo firewall-cmd --permanent --add-port=25/tcp    # SMTP
sudo firewall-cmd --permanent --add-port=465/tcp   # SMTPS
sudo firewall-cmd --permanent --add-port=587/tcp   # Submission (STARTTLS)
sudo firewall-cmd --reload
```

### Configuración del relay SMTP (Gmail)

**Editar la configuración principal:**
```bash
sudo nano /etc/postfix/main.cf
```

**Parámetros clave a verificar/agregar:**
```ini
myorigin = /etc/mailname
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost
mynetworks = 127.0.0.0/8, 10.0.0.0/24
relayhost = [smtp.gmail.com]:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
```

**Crear el archivo de credenciales SMTP:**
```bash
sudo nano /etc/postfix/sasl_passwd
```

```
[smtp.gmail.com]:587 tu_correo@gmail.com:contraseña_de_aplicación
```

> 🔑 La contraseña de aplicación se genera en [myaccount.google.com/apppasswords](https://myaccount.google.com/apppasswords) — requiere verificación en dos pasos activa en la cuenta de Gmail.

**Restringir permisos del archivo de credenciales:**
```bash
sudo chmod 600 /etc/postfix/sasl_passwd
```

**Generar el hash que utiliza Postfix:**
```bash
sudo postmap /etc/postfix/sasl_passwd
```

**Reiniciar el servicio:**
```bash
sudo systemctl restart postfix
```

### Envío de un correo de prueba

**Crear el contenido del mensaje:**
```bash
nano mensaje.txt
```

```
Julio Cesar Hernández Tibrey
Matrícula: 2025-0702
```

**Enviar el correo:**
```bash
mail -s "AsuntoDelCorreo" ejemplo@gmail.com < mensaje.txt
```

> ⚠️ **Nota de seguridad:** Las credenciales SMTP en `sasl_passwd` quedan en texto plano antes del hash — por eso es crítico restringir sus permisos a `600` (solo root) inmediatamente después de crearlas.

---

## 4.3 — Instalar un Servidor de Impresión

### Objetivo
Configurar **CUPS** (Common Unix Printing System) en Rocky Linux para compartir una impresora virtual PDF accesible desde un cliente Windows en red.

### Instalación y arranque del servicio

```bash
sudo dnf install cups -y
sudo systemctl enable --now cups
sudo systemctl status cups
```

**Instalar el backend de impresora virtual PDF:**
```bash
sudo dnf install cups-pdf -y
```

### Configuración de red y firewall

```bash
sudo firewall-cmd --add-port=631/tcp --permanent
sudo firewall-cmd --reload
```

### Configuración de acceso remoto

```bash
sudo nano /etc/cups/cupsd.conf
```

- Cambiar `Listen localhost:631` por `Port 631` (escucha en todas las interfaces).
- Agregar `Allow from 10.0.0.37` (IP del cliente Windows) en las secciones:
  `<Location />`, `<Location /admin>`, `<Location /admin/conf>`, `<Location /printers>`.

**Reiniciar el servicio:**
```bash
sudo systemctl restart cups
```

**Habilitar administración remota y compartir impresoras:**
```bash
cupsctl --remote-admin --remote-any --share-printers
```

### Configuración desde la interfaz web de CUPS

1. Desde el cliente Windows (`10.0.0.37`), acceder a `http://10.0.0.43:631`
2. Ir a **Administration → Add Printer**
3. Seleccionar la impresora PDF, asignar un nombre, marcar **"Share this printer"**
4. Elegir marca **Generic** y modelo **"Generic CUPS-PDF Printer"**
5. Finalizar con **Add Printer**

### Configuración desde Windows

1. **Settings → Devices → Printers & scanners → Add printer**
2. **"The printer I want isn't listed" → Select a shared printer by name**
3. Pegar la URL generada por CUPS (ej. `http://10.0.0.43/printers/Julio`), **quitando la "s" de https si la tuviera**

### Prueba de impresión

1. Abrir Word, escribir un texto, `Ctrl + P`
2. Seleccionar la impresora creada y hacer clic en **Print**

### Verificación

- En la interfaz web de CUPS: `http://10.0.0.43/printers/Julio` muestra los trabajos completados
- En el servidor, el PDF generado se almacena en:
  ```
  /var/spool/cups-pdf/ANONYMOUS/
  ```

---

## 📌 Conclusiones del Módulo

Este módulo cubrió tres servicios de red fundamentales en cualquier infraestructura empresarial: publicación web con múltiples sitios en un mismo servidor (Apache), transporte de correo saliente autenticado mediante relay externo (Postfix), e impresión compartida en red multiplataforma (CUPS). En los tres casos se trabajó la apertura controlada de puertos en el firewall — un recordatorio constante de que cada servicio expuesto amplía la superficie de ataque del sistema.

---

[⬅ Volver al índice general](../README.md)
