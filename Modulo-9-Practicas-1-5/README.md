# 🤖 Módulo 9 — Infraestructura como Código (IaC) y Automatización

> **Sistema Operativo:** Rocky Linux (controlador) · Ubuntu/Rocky + Windows (nodos gestionados)
> **Materia:** Sistemas Operativos III · ITLA
> **Cloud:** Microsoft Azure

---

## 📋 Contenido del Módulo

| # | Práctica | Tema |
|---|----------|------|
| 9.1 | Webmin | Panel de administración web |
| 9.2 | Terraform en Azure | Aprovisionamiento de infraestructura como código |
| 9.3 | Ansible — Instalación y configuración | Gestión de nodos Linux y Windows |
| 9.4 | Ansible — Playbooks | Automatización declarativa multiplataforma |
| 9.5 | Ansible — Comandos Ad-Hoc | Ejecución de tareas puntuales |

---

## 9.1 — Instalación de Webmin

### Objetivo
Instalar Webmin como panel de administración web para Rocky Linux.

### Comandos utilizados

**Agregar el repositorio oficial de Webmin:**
```bash
sudo tee /etc/yum.repos.d/webmin.repo<<EOF
[Webmin]
name=Webmin Distribution Neutral
baseurl=https://download.webmin.com/download/yum
enabled=1
gpgcheck=1
gpgkey=https://download.webmin.com/jcameron-key.asc
EOF
```

**Instalar:**
```bash
sudo dnf install webmin -y
```

**Abrir el puerto en el firewall:**
```bash
sudo firewall-cmd --permanent --add-port=10000/tcp
sudo firewall-cmd --reload
```

**Acceder a la interfaz web:** `https://10.0.0.25:10000`

---

## 9.2 — Desplegar VM con Terraform en Azure

### Objetivo
Aprovisionar una máquina virtual Linux en **Microsoft Azure** de forma completamente declarativa utilizando Terraform, incluyendo red virtual, subnet, grupo de seguridad e IP pública.

### Instalación de Terraform

```bash
sudo dnf install -y yum-utils
sudo dnf config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo dnf -y install terraform
terraform -version
```

### Instalación de Azure CLI

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

echo -e "[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc" | sudo tee /etc/yum.repos.d/azure-cli.repo

sudo dnf install -y azure-cli
```

**Autenticarse en Azure:**
```bash
az login
```

### Generar llave SSH para la VM

```bash
ls ~/.ssh/id_rsa.pub
ssh-keygen -t rsa -b 4096
```

### Definición de infraestructura (main.tf)

```bash
mkdir ~/terraform-azure-vm
cd ~/terraform-azure-vm
nano main.tf
```

```hcl
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.110.0"
    }
  }
  required_version = ">= 1.1.0"
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
  }
}

# Grupo de recursos
resource "azurerm_resource_group" "rg" {
  name     = "TerraFormSOIIIRG"
  location = "canadacentral"
}

# Red virtual
resource "azurerm_virtual_network" "vnet" {
  name                = "TFSOIIIVnet"
  address_space       = ["10.0.0.0/16"]
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

# Subnet
resource "azurerm_subnet" "subnet" {
  name                 = "TFSOIIISubnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = ["10.0.1.0/24"]
  depends_on           = [azurerm_virtual_network.vnet]
}

# Network Security Group (permite SSH)
resource "azurerm_network_security_group" "nsg" {
  name                = "TFSOIIINSG"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                       = "SSH"
    priority                   = 1001
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "22"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }
}

# Asociar NSG a la Subnet
resource "azurerm_subnet_network_security_group_association" "subnet_assoc" {
  subnet_id                 = azurerm_subnet.subnet.id
  network_security_group_id = azurerm_network_security_group.nsg.id
}

# IP pública estática
resource "azurerm_public_ip" "pip" {
  name                = "TFSOIIIPublicIP"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

# Interfaz de red
resource "azurerm_network_interface" "nic" {
  name                = "TFSOIIINIC"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  ip_configuration {
    name                          = "internal"
    subnet_id                     = azurerm_subnet.subnet.id
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = azurerm_public_ip.pip.id
  }
}

# Máquina virtual Linux (Ubuntu 22.04)
resource "azurerm_linux_virtual_machine" "vm" {
  name                = "OS3VM"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  size                = "Standard_B2as_v2"
  admin_username      = "julio"

  network_interface_ids = [azurerm_network_interface.nic.id]

  os_disk {
    caching              = "ReadWrite"
    storage_account_type = "Standard_LRS"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "0001-com-ubuntu-server-jammy"
    sku       = "22_04-lts-gen2"
    version   = "latest"
  }

  admin_ssh_key {
    username   = "julio"
    public_key = file("~/.ssh/azure_vm_key.pub")
  }
}

# Output de la IP pública asignada
output "vm_public_ip" {
  description = "Dirección IP pública de la máquina virtual"
  value       = azurerm_public_ip.pip.ip_address
}
```

### Ciclo de vida de Terraform

```bash
terraform init        # Descarga el provider de Azure
sudo terraform fmt    # Formatea el código
terraform validate    # Valida la sintaxis
terraform plan        # Muestra el plan de ejecución (qué se va a crear)
terraform apply        # Aprovisiona la infraestructura
```

**Conectarse a la VM creada:**
```bash
ssh -i ~/.ssh/azure_vm_key julio@<IP_PUBLICA_VM>
```

> 💡 **Concepto clave:** Terraform usa un modelo **declarativo**: el archivo `.tf` describe el estado final deseado, no los pasos para llegar ahí. El grafo de dependencias (`depends_on`, referencias entre recursos) permite a Terraform determinar automáticamente el orden correcto de creación.

---

## 9.3 — Instalación y Configuración de Ansible

### Objetivo
Configurar un nodo controlador Ansible capaz de gestionar tanto clientes **Linux (SSH)** como **Windows (WinRM)**.

### Instalación del controlador

```bash
sudo dnf install -y ansible sshpass python3-pip
ansible --version
```

### Crear el usuario de servicio `ansible` (en el controlador y en los nodos Linux)

```bash
sudo adduser ansible
sudo passwd ansible
sudo usermod -aG wheel ansible
sudo usermod -aG sudo ansible
sudo visudo
```

**Agregar al archivo sudoers:**
```
ansible ALL=(ALL) NOPASSWD:ALL
```

### Preparar el nodo Windows (PowerShell como Administrador)

```powershell
net user ansible 1 /add
net localgroup Administrators ansible /add
```

### Configurar acceso SSH sin contraseña a nodos Linux

```bash
su - ansible
ssh-keygen -t rsa -b 4096 -C "ansible@controller"
ssh-copy-id ansible@172.185.29.172
```

### Habilitar WinRM en el nodo Windows

```powershell
winrm quickconfig
Enable-PSRemoting -Force
Set-Item -Path WSMan:\localhost\Service\Auth\Basic -Value $true
Set-Item -Path WSMan:\localhost\Service\AllowUnencrypted -Value $true
```

**Reglas de firewall para WinRM:**
```powershell
New-NetFirewallRule -Name "WinRM_TCP_IN"  -DisplayName "WinRM TCP Entrada"  -Protocol TCP -LocalPort 5985 -Direction Inbound  -Action Allow
New-NetFirewallRule -Name "WinRM_UDP_IN"  -DisplayName "WinRM UDP Entrada"  -Protocol UDP -LocalPort 5985 -Direction Inbound  -Action Allow
New-NetFirewallRule -Name "WinRM_TCP_OUT" -DisplayName "WinRM TCP Salida"   -Protocol TCP -LocalPort 5985 -Direction Outbound -Action Allow
New-NetFirewallRule -Name "WinRM_UDP_OUT" -DisplayName "WinRM UDP Salida"   -Protocol UDP -LocalPort 5985 -Direction Outbound -Action Allow
New-NetFirewallRule -DisplayName "Allow WinRM" -Name "AllowWinRM5985" -Protocol TCP -LocalPort 5985 -Action Allow
```

### Inventario de Ansible

```bash
nano /etc/ansible/hosts
```

```ini
[linux]
172.185.29.172 ansible_user=ansible ansible_connection=ssh

[win]
10.0.0.24 ansible_user=ansible ansible_password=1 ansible_connection=winrm ansible_winrm_transport=basic ansible_port=5985
```

### Pruebas de conectividad

```bash
ansible linux -m ping -i /etc/ansible/hosts
ansible win -m win_ping -i /etc/ansible/hosts
```

### 🛠️ Troubleshooting — Falla el ping a Windows

```bash
sudo /usr/bin/python3.12 -m ensurepip --upgrade
/usr/bin/python3.12 -m pip --version
sudo /usr/bin/python3.12 -m pip install --upgrade pip setuptools wheel
sudo /usr/bin/python3.12 -m pip install pywinrm "pywinrm[credssp]" requests-ntlm xmltodict
/usr/bin/python3.12 -c "import winrm; print('winrm OK')"
```

> 💡 **Causa común:** el módulo `win_ping` depende de la librería Python `pywinrm` en el controlador — si no está instalada para el intérprete Python correcto, la conexión a nodos Windows falla silenciosamente.

---

## 9.4 — Uso de Playbooks en Ansible

### Objetivo
Crear playbooks YAML para automatizar la instalación de software en nodos Windows y Linux de forma idempotente.

### Playbook 1 — Instalar Notepad++ en Windows

```bash
sudo nano install_notepadpp.yml
```

```yaml
- name: Instalar Notepad++ en Windows
  hosts: win
  tasks:
    - name: Descargar instalador de Notepad++
      win_get_url:
        url: https://github.com/notepad-plus-plus/notepad-plus-plus/releases/download/v8.6.8/npp.8.6.8.Installer.x64.exe
        dest: C:\Users\ansible\Downloads\npp_installer.exe

    - name: Instalar Notepad++
      win_package:
        path: C:\Users\ansible\Downloads\npp_installer.exe
        state: present
        arguments: /S
```

**Ejecutar:**
```bash
ansible-playbook install_notepadpp.yml
```

### Playbook 2 — Instalar NGINX en Linux

```bash
sudo nano install_nginx.yml
```

```yaml
- name: Instalar NGINX en Linux
  hosts: linux
  become: true
  tasks:
    - name: Instalar nginx
      dnf:
        name: nginx
        state: present

    - name: Iniciar y habilitar nginx
      systemd:
        name: nginx
        enabled: true
        state: started
```

**Ejecutar y verificar:**
```bash
ansible-playbook install_nginx.yml
sudo systemctl status nginx
```

### Extra — Playbook combinado: crear usuario + instalar herramienta

**Versión RedHat/Rocky:**
```yaml
- name: Crear usuario y agregarlo al grupo wheel + instalar bashtop
  hosts: linux
  become: true
  tasks:
    - name: Crear usuario adrian123
      user:
        name: adrian123
        state: present
        groups: wheel
        append: yes

    - name: Instalar bashtop
      dnf:
        name: bashtop
        state: present
```

**Versión Debian/Ubuntu (equivalente):**
```yaml
- hosts: linux
  become: yes
  tasks:
    - name: Agregar usuario
      user:
        name: adrian123
        password: "{{ 'adrian123' | password_hash('sha512') }}"
        groups: sudo
        append: yes
        shell: /bin/bash

    - name: Instalar Bashtop
      apt:
        name: bashtop
        update_cache: yes
        state: present
```

**Ejecutar:**
```bash
ansible-playbook create_user_and_install_bashtop.yml
```

### Extras — Crear usuario administrador en Windows + instalar software

```yaml
- name: Crear usuario en Windows con permisos de administrador e instalar Notepad++
  hosts: win
  tasks:
    - name: Crear usuario adrian123
      win_user:
        name: adrian123
        password: "F07030409"
        state: present

    - name: Agregar usuario adrian123 al grupo Administradores
      win_group_membership:
        name: Administrators
        members:
          - adrian123
        state: present

    - name: Descargar instalador de Notepad++
      win_get_url:
        url: https://github.com/notepad-plus-plus/notepad-plus-plus/releases/download/v8.6.8/npp.8.6.8.Installer.x64.exe
        dest: C:\Users\ansible\Downloads\npp_installer.exe

    - name: Instalar Notepad++
      win_package:
        path: C:\Users\ansible\Downloads\npp_installer.exe
        state: present
        arguments: /S
```

**Ejecutar:**
```bash
ansible-playbook create_user_and_install_notepadpp.yml
```

## 9.5 — Comandos Ad-Hoc con Ansible

### Objetivo
Ejecutar tareas puntuales sin necesidad de escribir un playbook completo, usando el modo *ad-hoc* de Ansible.

### Reiniciar un nodo Linux remoto

```bash
ansible linux -m reboot -b
```

### Copiar un archivo dentro de un nodo Windows remoto

```bash
ansible win -m win_copy \
  -a "src='C:/Users/ansible/Desktop/prueba.txt' dest='C:/Users/ansible/Documents/prueba.txt' remote_src=yes"
```

> 💡 **Concepto clave:** Los comandos ad-hoc son ideales para tareas de **una sola vez** (reiniciar, copiar un archivo, verificar conectividad). Para configuraciones repetibles y versionables, los **playbooks** son la herramienta correcta — esa es la diferencia fundamental entre automatización puntual y automatización como código (IaC).

---

## 📌 Conclusiones del Módulo

Este módulo integró tres herramientas que en conjunto forman un flujo DevOps/IaC completo: **Terraform** para aprovisionar la infraestructura subyacente en la nube (servidores, redes, seguridad), y **Ansible** para configurar el software dentro de esa infraestructura, tanto en Linux como en Windows. La progresión de comandos ad-hoc → playbooks simples → playbooks con detección de SO y manejo de errores (`block`/`rescue`) refleja cómo evoluciona la automatización real: de tareas manuales puntuales hacia código de infraestructura robusto, idempotente y multiplataforma.

---

[⬅ Volver al índice general](../README.md)
