# Manual Técnico: Despliegue de Entornos DevSecOps con Dev Containers

**Asignatura:** Administración de Sistemas / DevSecOps
**Sistema Operativo Host:** Ubuntu 22.04 LTS (Nativo)
**Hardware:** Intel Core i3 | 16GB RAM (Optimización de recursos prioritaria)

---

## 1. Introducción

[cite_start]El objetivo de esta práctica es implementar entornos de desarrollo reproducibles, aislados y seguros utilizando **Dev Containers**[cite: 6]. [cite_start]Esta tecnología soluciona el problema de "funciona en mi máquina" al desacoplar las herramientas de desarrollo del sistema operativo anfitrión[cite: 6].

### Estrategia de Implementación
[cite_start]Para garantizar la seguridad y el control total sobre el software, utilizaremos **imágenes propias (Custom Images)** definidas mediante `Dockerfile`[cite: 24, 25]. Esto nos permite:
* [cite_start]Controlar las versiones exactas de las herramientas[cite: 35].
* Reducir el tamaño de las imágenes (uso de versiones `slim`).
* [cite_start]Implementar medidas de seguridad (usuarios no-root)[cite: 116].

---

## 2. Preparación del Host (Pre-requisitos)

Dado que trabajamos en un equipo con recursos de CPU limitados, utilizaremos **Docker Engine nativo** en Linux para evitar la sobrecarga de máquinas virtuales o Docker Desktop.

### 2.1 Instalación y Reparación del Motor Docker
Se realiza una instalación limpia para evitar conflictos con virtualizadores previos (VirtualBox).

```bash
# 1. Eliminar módulos conflictivos y limpiar
sudo apt-get remove virtualbox-dkms
sudo dpkg --configure -a

# 2. Instalar Docker Engine y Docker Compose
sudo apt-get update
sudo apt-get install docker.io docker-compose -y

# 3. Configuración de permisos de usuario (Vital para VS Code)
# Esto permite que VS Code hable con Docker sin pedir contraseña de root
sudo usermod -aG docker $USER
newgrp docker
```

> **[INSERTAR PANTALLAZO 1: Terminal con `docker run hello-world`]**
> *Evidencia de que el servicio Docker está activo y el usuario tiene permisos.*

### 2.2 Instalación de Visual Studio Code
Si el entorno no dispone de VS Code, lo instalaremos mediante paquetería `snap` (estándar en Ubuntu) para asegurar actualizaciones automáticas.

```bash
sudo snap install code --classic
```

### 2.3 Instalación de la Extensión "Dev Containers"
[cite_start]Para que la arquitectura cliente-servidor funcione[cite: 10], es obligatorio instalar la extensión que conecta el IDE con el motor Docker.

1.  Abrir VS Code.
2.  Ir a la barra lateral izquierda, icono de **Extensiones** (o `Ctrl+Shift+X`).
3.  Buscar: `Dev Containers`.
4.  Instalar la extensión oficial de Microsoft (`ms-vscode-remote.remote-containers`).

> **[INSERTAR PANTALLAZO 2: Captura de la pestaña de extensiones mostrando "Dev Containers" instalada]**

---

## 3. Configuración del Espacio de Trabajo

Para mantener el orden y asegurar que VS Code detecte las configuraciones, crearemos una estructura de carpetas específica. [cite_start]Cada proyecto debe tener su propia carpeta `.devcontainer`[cite: 21].

### 3.1 Creación de Directorios
Ejecuta los siguientes comandos en tu terminal para generar la estructura completa de una sola vez:

```bash
# Crear carpeta raíz de la práctica
mkdir -p ~/practicas-devsecops

# Crear subcarpetas para cada tecnología y sus configuraciones ocultas
mkdir -p ~/practicas-devsecops/script-python/.devcontainer
mkdir -p ~/practicas-devsecops/backend-net/.devcontainer
mkdir -p ~/practicas-devsecops/frontend-angular/.devcontainer

# Verificar la estructura
tree -a ~/practicas-devsecops
```

> **[INSERTAR PANTALLAZO 3: Salida del comando `tree` mostrando las carpetas creadas]**

---

## 4. Implementación de los Entornos (Código)

A continuación, definimos la infraestructura como código (IaC) para cada entorno.

### 4.1 Entorno Scripting (Python) - Seguridad (DevSecOps)
**Objetivo:** Evitar ejecutar contenedores como `root`. [cite_start]Configuramos un usuario `vscode` con UID 1000[cite: 127].

**Archivo:** `script-python/.devcontainer/Dockerfile`
```dockerfile
FROM python:3.12-slim-bookworm

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# SEGURIDAD: Creación de usuario no-root
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME \
    && apt-get update \
    && apt-get install -y sudo git \
    && echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

RUN pip install --no-cache-dir black pylint pytest
USER $USERNAME
```

**Archivo:** `script-python/.devcontainer/devcontainer.json`
```json
{
  "name": "Python Seguro",
  "build": { "dockerfile": "Dockerfile" },
  "remoteUser": "vscode",
  "customizations": {
    "vscode": {
      "settings": {
        "python.defaultInterpreterPath": "/usr/local/bin/python",
        "python.linting.enabled": true
      },
      "extensions": ["ms-python.python"]
    }
  }
}
```

### 4.2 Entorno Backend (.NET) - Productividad
[cite_start]**Objetivo:** Pre-instalar herramientas globales (`dotnet-ef`) en la imagen para reducir tiempos de espera[cite: 89].

**Archivo:** `backend-net/.devcontainer/Dockerfile`
```dockerfile
FROM [mcr.microsoft.com/dotnet/sdk:9.0-bookworm-slim](https://mcr.microsoft.com/dotnet/sdk:9.0-bookworm-slim)

# Instalación de herramientas globales en la capa de sistema
RUN apt-get update && apt-get install -y git procps curl \
    && dotnet tool install --global dotnet-ef

ENV PATH="$PATH:/root/.dotnet/tools"

# Certificados HTTPS para desarrollo local
RUN dotnet dev-certs https --clean && dotnet dev-certs https --trust
```

**Archivo:** `backend-net/.devcontainer/devcontainer.json`
```json
{
  "name": "Backend .NET Core",
  "build": { "dockerfile": "Dockerfile" },
  "forwardPorts": [5000, 5001],
  "postCreateCommand": "dotnet --version",
  "remoteUser": "root"
}
```

### 4.3 Entorno Frontend (Angular) - Rendimiento
[cite_start]**Objetivo:** Usar volúmenes Docker para `node_modules` para evitar la lentitud de E/S en disco[cite: 106].

**Archivo:** `frontend-angular/.devcontainer/Dockerfile`
```dockerfile
FROM node:22-bookworm-slim
# Versión fija de Angular CLI
RUN npm install -g @angular/cli@19.0.0
RUN apt-get update && apt-get install -y git
USER node
```

**Archivo:** `frontend-angular/.devcontainer/devcontainer.json`
```json
{
  "name": "Angular Frontend",
  "build": { "dockerfile": "Dockerfile" },
  "remoteUser": "node",
  "forwardPorts": [4200],
  "mounts": [
    "source=node_modules_cache,target=/workspace/node_modules,type=volume"
  ]
}
```

---

## 5. Procedimiento de Ejecución y Validación

Para iniciar cualquiera de los entornos creados, siga este procedimiento estándar en Visual Studio Code.

### Paso 1: Abrir la Carpeta del Proyecto
1.  En VS Code, ir a `Archivo > Abrir Carpeta...` (File > Open Folder).
2.  Seleccionar la carpeta específica del proyecto, por ejemplo: `~/practicas-devsecops/script-python`.
    * *Nota:* No abra la carpeta raíz, abra la subcarpeta del proyecto específico.

### Paso 2: Detección y Apertura en Contenedor
VS Code detectará automáticamente la carpeta `.devcontainer`.
1.  Aparecerá una notificación en la esquina inferior derecha: *"Folder contains a Dev Container configuration file"*.
2.  Hacer clic en el botón azul **"Reopen in Container"**.

**Método Alternativo (Si no sale la notificación):**
1.  Pulsar `F1` o `Ctrl+Shift+P` para abrir la Paleta de Comandos.
2.  Escribir: `Dev Containers: Reopen in Container`.
3.  Presionar Enter.

> **[INSERTAR PANTALLAZO 4: Ventana de VS Code mostrando el proceso "Starting Dev Container" o logs de construcción]**

### Paso 3: Validación del Entorno
Una vez cargado el entorno (indicado por la etiqueta verde en la esquina inferior izquierda), abrir la terminal integrada (`Ctrl + ñ`) y verificar:

* **Para Python:** Ejecutar `whoami`. Debe responder **vscode**.
* **Para .NET:** Ejecutar `dotnet --list-sdks`. Debe mostrar la versión 9.0.
* **Para Angular:** Ejecutar `ng version`. Debe mostrar Angular CLI 19.

> **[INSERTAR PANTALLAZO 5: Captura final con la terminal mostrando los comandos de validación exitosos]**

---

## 6. Conclusión

A través de esta práctica hemos logrado configurar un ciclo de vida de desarrollo moderno sobre un hardware limitado. La combinación de Docker nativo en Linux y las configuraciones optimizadas de Dev Containers (volúmenes y usuarios non-root) nos permite cumplir con los requisitos de DevSecOps sin comprometer el rendimiento del equipo local.

**Referencias:**
*Informe de Investigación sobre Dev Containers y Arquitectura de Sistemas.*
