# Manual Técnico: Despliegue de Entornos DevSecOps con Dev Containers

**Asignatura:** Administración de Sistemas / DevSecOps
**Sistema Operativo Host:** Ubuntu 22.04 LTS (Nativo)
**Hardware:** Intel Core i3 | 16GB RAM (Optimización de recursos prioritaria)

---

## 1. Introducción

El objetivo de esta práctica es implementar entornos de desarrollo reproducibles, aislados y seguros utilizando **Dev Containers**. Esta tecnología soluciona el problema de "funciona en mi máquina" al desacoplar las herramientas de desarrollo del sistema operativo anfitrión.

### Estrategia de Implementación
Para garantizar la seguridad y el control total sobre el software, utilizaremos **imágenes propias (Custom Images)** definidas mediante `Dockerfile`. Esto nos permite:
* Controlar las versiones exactas de las herramientas.
* Reducir el tamaño de las imágenes (uso de versiones `slim`).
* Implementar medidas de seguridad (usuarios no-root cuando sea posible).

---

## 2. Preparación del Host (Pre-requisitos)

Dado que trabajamos en un equipo con recursos de CPU limitados, utilizaremos **Docker Engine nativo** en Linux para evitar la sobrecarga de máquinas virtuales o Docker Desktop.

### 2.1 Instalación y Reparación del Motor Docker
Se realiza una instalación limpia para evitar conflictos con virtualizadores previos (VirtualBox).

```bash
# 1. Eliminar módulos conflictivos y limpiar dependencias rotas
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

> **Verificación de Servicio:**
>
> ![Validación Docker Hola Mundo](img/pantallazo1.png)
> *Evidencia de que el servicio Docker está activo y el usuario tiene permisos correctos.*

### 2.2 Instalación de Visual Studio Code
Si el entorno no dispone de VS Code, lo instalaremos mediante paquetería `snap` (estándar en Ubuntu) para asegurar actualizaciones automáticas.

```bash
sudo snap install code --classic
```

### 2.3 Instalación de la Extensión "Dev Containers"
Para que la arquitectura cliente-servidor funcione, es obligatorio instalar la extensión que conecta el IDE con el motor Docker.

1.  Abrir VS Code.
2.  Ir a la barra lateral izquierda, icono de **Extensiones** (o `Ctrl+Shift+X`).
3.  Buscar: `Dev Containers`.
4.  Instalar la extensión oficial de Microsoft (`ms-vscode-remote.remote-containers`).

> **Extensión Instalada:**
>
> ![Extensión Dev Containers](img/pantallazo2.png)

---

## 3. Configuración del Espacio de Trabajo

Para mantener el orden y asegurar que VS Code detecte las configuraciones, crearemos una estructura de carpetas específica. Cada proyecto debe tener su propia carpeta `.devcontainer`.

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

> **Estructura de Directorios:**
>
> ![Estructura Tree](img/pantallazo3.png)

---

## 4. Implementación de los Entornos (Código)

A continuación, definimos la infraestructura como código (IaC) para cada entorno.

### 4.1 Entorno Scripting (Python) - Seguridad (DevSecOps)
**Objetivo:** Evitar ejecutar contenedores como `root`. Configuramos un usuario `vscode` con UID 1000.

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
**Objetivo:** Pre-instalar herramientas globales (`dotnet-ef`) en la imagen para reducir tiempos de espera.

**Justificación de Usuario (`root`):**
A diferencia de Python, en este contenedor mantenemos el usuario `root`. Esto se hace deliberadamente para facilitar la instalación de herramientas globales del SDK y la gestión de certificados SSL de desarrollo sin errores de permisos.

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
**Objetivo:** Usar volúmenes Docker para `node_modules` para evitar la lentitud de E/S en disco.

**Justificación de Usuario (`node`):**
Aquí **NO** utilizamos root. Usamos el usuario `node` que viene pre-configurado en la imagen oficial. Al igual que en Python, esto sigue las mejores prácticas de seguridad para aplicaciones web expuestas.

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
    * *Importante:* No abra la carpeta raíz, abra la subcarpeta del proyecto específico.

### Paso 2: Detección y Apertura en Contenedor
VS Code detectará automáticamente la carpeta `.devcontainer`.
1.  Aparecerá una notificación o se usará la paleta de comandos para reabrir en contenedor.
2.  Hacer clic en el botón azul **"Reopen in Container"**.

> **Proceso de Apertura:**
>
> ![Notificación o Menú Reopen](img/pantallazo4.0.png)
>
> ![Construyendo Contenedor](img/pantallazo4.1.png)

### Paso 3: Validación del Entorno
Una vez cargado el entorno (indicado por la etiqueta verde en la esquina inferior izquierda), abrir la terminal integrada (`Ctrl + ñ`) y verificar que las herramientas están correctamente instaladas:

* **Para Python:**
  Ejecutar: `whoami && python --version`
  *Resultado esperado:* Debe mostrar el usuario **vscode** y la versión **3.12.x**.
  ![Validación Python](img/pantallazo5.1.png)

* **Para .NET:**
  Ejecutar: `dotnet --list-sdks`
  *Resultado esperado:* Debe mostrar el SDK versión **9.0.x** instalado en `/usr/share/dotnet/sdk`.
  ![Validación .NET](img/pantallazo5.2.png)

* **Para Angular:**
  Ejecutar: `ng version`
  *Resultado esperado:* Debe mostrar el logotipo de Angular CLI y la versión **19.0.0**.
  ![Validación Angular](img/pantallazo5.3.png)

---

## 6. Conclusión

A través de esta práctica hemos logrado configurar un ciclo de vida de desarrollo moderno sobre un hardware limitado. La combinación de Docker nativo en Linux y las configuraciones optimizadas de Dev Containers (volúmenes y usuarios no-root) nos permite cumplir con los requisitos de DevSecOps sin comprometer el rendimiento del equipo local.
