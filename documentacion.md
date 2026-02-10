# Manual de Implementación de Entornos DevSecOps con Dev Containers

**Asignatura:** Administración de Sistemas / DevSecOps
**Entorno Objetivo:** Sistemas Linux con recursos limitados (Docker Nativo)

---

## 1. Introducción

El objetivo de esta práctica es implementar entornos de desarrollo reproducibles, aislados y seguros utilizando **Dev Containers**. Esta tecnología soluciona el problema de "funciona en mi máquina" al desacoplar las herramientas de desarrollo del sistema operativo anfitrión.

### Estrategia de Imágenes
Para garantizar la seguridad y el control total sobre el software, no utilizaremos las imágenes predeterminadas de VS Code. En su lugar, aplicaremos una estrategia de **imágenes propias (Custom Images)** definidas mediante `Dockerfile`. Esto nos permite:
* Controlar las versiones exactas de las herramientas.
* Reducir el tamaño de las imágenes (optimización de recursos).
* Implementar medidas de seguridad (usuarios no-root).

---

## 2. Preparación del Host (Linux)

Para maximizar el rendimiento en equipos con hardware limitado (poca RAM o CPU modesta), se opta por utilizar **Docker Engine nativo** directamente sobre Linux. Esto evita la sobrecarga de recursos que generan soluciones de escritorio o máquinas virtuales intermedias.

### 2.1 Instalación Limpia del Motor
Se recomienda una instalación mínima y limpia para evitar conflictos con otros gestores de virtualización.

```bash
# 1. Actualización de repositorios
sudo apt-get update

# 2. Instalación del motor Docker y Docker Compose
sudo apt-get install docker.io docker-compose -y

# 3. Configuración de permisos (Post-instalación)
# Esto permite ejecutar docker sin usar 'sudo' constantemente
sudo usermod -aG docker $USER
newgrp docker
```

> **[INSERTAR PANTALLAZO 1: Terminal mostrando la ejecución exitosa de `docker run hello-world`]**
> *Evidencia de que el motor Docker está operativo y los permisos de usuario son correctos.*

---

## 3. Implementación de los Entornos

Se han configurado tres entornos distintos. Cada uno demuestra una competencia técnica específica: Seguridad, Productividad y Rendimiento.

### 3.1 Entorno Scripting (Python)
**Enfoque: Seguridad (DevSecOps)**
En este entorno aplicamos el principio de mínimo privilegio. Por defecto, los contenedores corren como `root`, lo cual es un riesgo de seguridad. Aquí configuramos un usuario estándar.

* **Imagen Base:** `python:3.12-slim-bookworm` (Versión ligera).
* **Técnica:** Creación manual de un usuario no-root (`vscode`) con el mismo UID que el host (1000) para evitar conflictos de permisos en los archivos.

**Archivo:** `script-python/.devcontainer/Dockerfile`

```dockerfile
FROM python:3.12-slim-bookworm

# Variables para optimizar Python en contenedores
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
    # Permitir sudo sin contraseña (opcional para desarrollo)
    && echo "$USERNAME ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# Instalación de herramientas de calidad de código
RUN pip install --no-cache-dir black pylint pytest

# Cambiar contexto al usuario seguro
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

> **[INSERTAR PANTALLAZO 2: Terminal dentro de VS Code ejecutando `whoami`]**
> *Verificación de que el usuario activo es 'vscode' y no 'root'.*

---

### 3.2 Entorno Backend (.NET)
**Enfoque: Productividad**
El objetivo es reducir el tiempo de configuración diario ("Time-to-code"). En lugar de instalar herramientas manualmente cada vez que se inicia el contenedor, las "horneamos" dentro de la imagen.

* **Imagen Base:** `mcr.microsoft.com/dotnet/sdk:9.0-bookworm-slim`.
* **Técnica:** Pre-instalación de herramientas globales como `dotnet-ef` (Entity Framework) y generación de certificados HTTPS en el Dockerfile.

**Archivo:** `backend-net/.devcontainer/Dockerfile`

```dockerfile
FROM [mcr.microsoft.com/dotnet/sdk:9.0-bookworm-slim](https://mcr.microsoft.com/dotnet/sdk:9.0-bookworm-slim)

# Instalación de dependencias del sistema y herramienta EF Core
RUN apt-get update && apt-get install -y git procps curl \
    && dotnet tool install --global dotnet-ef

# Añadir herramientas al PATH es crítico para que la terminal las reconozca
ENV PATH="$PATH:/root/.dotnet/tools"

# Generación y confianza de certificados de desarrollo HTTPS
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

> **[INSERTAR PANTALLAZO 3: Terminal ejecutando `dotnet --list-sdks`]**
> *Confirmación de que el SDK está instalado y listo para compilar.*

---

### 3.3 Entorno Frontend (Angular)
**Enfoque: Rendimiento de E/S**
Uno de los mayores cuellos de botella en la virtualización es la sincronización de carpetas con miles de archivos pequeños, como `node_modules`.

* **Imagen Base:** `node:22-bookworm-slim`.
* **Técnica:** Uso de **Volúmenes Nombrados (Named Volumes)**. Configuramos Docker para que la carpeta `node_modules` se almacene en un volumen interno de alto rendimiento, en lugar de intentar sincronizarla con el disco duro del host.

**Archivo:** `frontend-angular/.devcontainer/Dockerfile`

```dockerfile
FROM node:22-bookworm-slim

# Instalación global de Angular CLI (fijando versión para estabilidad)
RUN npm install -g @angular/cli@19.0.0

RUN apt-get update && apt-get install -y git

# Uso del usuario predeterminado de Node por seguridad
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

> **[INSERTAR PANTALLAZO 4: Terminal ejecutando `ng version`]**
> *Verificación de Angular CLI correctamente instalado.*

---

## 4. Conclusiones

Mediante esta práctica se ha logrado establecer una arquitectura de desarrollo robusta y ligera.

1.  **Eficiencia:** El uso de Docker nativo en Linux permite ejecutar múltiples stacks tecnológicos en hardware modesto.
2.  **Seguridad:** La implementación de usuarios no privilegiados en los Dockerfiles mitiga riesgos de seguridad.
3.  **Reproducibilidad:** Toda la configuración está definida como código (IaC), permitiendo que cualquier miembro del equipo replique el entorno exacto con un solo comando.
