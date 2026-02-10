# Informe Técnico: Implementación de Entornos DevSecOps con Dev Containers

**Departamento:** Desarrollo / DevOps
**Proyecto:** Estandarización de Entornos de Desarrollo
**Autor:** [Tu Nombre]
**Fecha:** Febrero 2026

---

## 1. Resumen Ejecutivo

Este documento detalla la investigación e implementación técnica de entornos de desarrollo basados en contenedores (**Dev Containers**) para los stacks tecnológicos de la compañía (.NET, Angular y Python).

El objetivo es establecer una arquitectura de **Infraestructura como Código (IaC)** que garantice:
1.  **Reproducibilidad:** Eliminación de discrepancias entre entornos locales ("funciona en mi máquina").
2.  **Seguridad (DevSecOps):** Implementación de principios de mínimo privilegio en el ciclo de desarrollo.
3.  **Eficiencia:** Optimización del consumo de recursos mediante el uso de Docker nativo en Linux.

---

## 2. Investigación: Uso de Imágenes Personalizadas

### 2.1 Viabilidad Técnica
Se ha confirmado la necesidad de utilizar **Imágenes Propias (Custom Dockerfiles)** en lugar de las imágenes genéricas. Esta decisión estratégica permite:

* **Control de Versiones:** Fijar versiones exactas de runtimes (ej. Node 22, .NET 9) y herramientas críticas.
* **Seguridad:** Endurecimiento (hardening) de la imagen base y gestión de usuarios no privilegiados.
* **Rendimiento:** Uso de imágenes base `slim` (Debian Bookworm) para reducir la huella de memoria y disco.

### 2.2 Estrategia de Configuración
La configuración se centraliza en el archivo `devcontainer.json`, el cual orquesta el ciclo de vida del contenedor, mientras que la construcción del sistema operativo se delega al `Dockerfile`.

---

## 3. Arquitectura de la Solución

Se ha desplegado la solución sobre un host **Ubuntu 22.04 LTS** utilizando **Docker Engine Nativo**. Esta arquitectura evita la sobrecarga de virtualización, permitiendo un rendimiento óptimo en hardware estándar.

### 3.1 Prerrequisitos e Instalación
Se ha saneado el entorno host eliminando conflictos con módulos de virtualización heredados para garantizar la estabilidad del Docker Daemon.

```bash
# 1. Limpieza de conflictos previos
sudo apt-get remove virtualbox-dkms
sudo dpkg --configure -a

# 2. Instalación del Motor Docker
sudo apt-get update
sudo apt-get install docker.io docker-compose -y

# 3. Configuración de permisos (Rootless interaction)
sudo usermod -aG docker $USER
newgrp docker
```

> **Verificación del Motor Docker:**
>
> ![Estado del Servicio Docker](img/pantallazo1.png)
>
> *El motor Docker opera correctamente con permisos de usuario configurados.*

> **Integración con IDE:**
>
> ![Extensión Dev Containers](img/pantallazo2.png)
>
> *Se utiliza la extensión 'Dev Containers' para habilitar la arquitectura cliente-servidor.*

---

## 4. Estructura del Proyecto

Se ha definido una estructura de directorios modular, donde cada microservicio mantiene su propia configuración de infraestructura aislada.

```bash
# Generación de la estructura de carpetas
mkdir -p ~/practicas-devsecops/script-python/.devcontainer
mkdir -p ~/practicas-devsecops/backend-net/.devcontainer
mkdir -p ~/practicas-devsecops/frontend-angular/.devcontainer
```

> **Árbol de Directorios:**
>
> ![Estructura Tree](img/pantallazo3.png)

---

## 5. Definición de Entornos (Infraestructura como Código)

A continuación, se documenta la configuración técnica de los archivos generados para cada stack tecnológico.

### 5.1 Stack de Scripting (Python)
**Enfoque:** DevSecOps / Seguridad.
Se ha configurado un entorno donde el desarrollador opera con un usuario estándar (`vscode`, UID 1000) en lugar de `root`.

**Archivo:** `script-python/.devcontainer/Dockerfile`
```dockerfile
FROM python:3.12-slim-bookworm

ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# IMPLEMENTACIÓN DE SEGURIDAD: Usuario no-root
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

---

### 5.2 Stack Backend (.NET)
**Enfoque:** Productividad del Desarrollador.
A diferencia de Python, en este contenedor mantenemos el usuario `root`. Esto se hace deliberadamente para facilitar la instalación de herramientas globales del SDK y la gestión de certificados SSL de desarrollo sin errores de permisos. 

> [!WARNING]
> **En un entorno de producción, se cambiaría a un usuario sin privilegios por seguridad.**



**Archivo:** `backend-net/.devcontainer/Dockerfile`
```dockerfile
FROM [mcr.microsoft.com/dotnet/sdk:9.0-bookworm-slim](https://mcr.microsoft.com/dotnet/sdk:9.0-bookworm-slim)

# Pre-instalación de herramientas globales (Entity Framework)
RUN apt-get update && apt-get install -y git procps curl \
    && dotnet tool install --global dotnet-ef

ENV PATH="$PATH:/root/.dotnet/tools"

# Generación de certificados de confianza para desarrollo local
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

---

### 5.3 Stack Frontend (Angular)
**Enfoque:** Rendimiento de E/S.
Se utiliza un **Volumen Nombrado (Named Volume)** de Docker para aislar la carpeta `node_modules` del sistema de archivos del host, mejorando drásticamente la velocidad de compilación.

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

## 6. Proceso de Compilación (Ciclo de Vida)

Dando respuesta al requerimiento técnico sobre **cómo se compilan las imágenes** en esta implementación:

En este entorno de desarrollo, la compilación es **orquestada localmente por VS Code**:
1.  **Trigger:** Al abrir el proyecto, VS Code detecta la carpeta `.devcontainer`.
2.  **Build Context:** El IDE invoca al CLI de Docker utilizando el directorio actual como contexto.
3.  **Ejecución:** Se ejecuta `docker build`. Docker lee el `Dockerfile` y ejecuta las instrucciones capa por capa.

> **Evidencia del Proceso de Build:**
>
> ![Detección de Configuración](img/pantallazo4.0.png)
>
> *VS Code detecta la configuración y solicita reapertura.*
>
> ![Logs de Compilación](img/pantallazo4.1.png)
>
> *Traza de ejecución del comando docker build iniciada automáticamente por el IDE.*

---

## 7. Validación de los Entornos Desplegados

A continuación, se presentan las evidencias de funcionamiento y configuración correcta para cada uno de los entornos solicitados.

### 7.1 Validación Stack Python
Se verifica mediante el comando `whoami` que el usuario activo es **vscode** (no root) y que la versión del intérprete es la 3.12, cumpliendo los requisitos de seguridad.

> **Evidencia Python:**
>
> ![Validación Python](img/pantallazo5.1.png)

### 7.2 Validación Stack .NET
Se verifica mediante `dotnet --list-sdks` que el SDK 9.0 está correctamente instalado y accesible en el path del sistema.

> **Evidencia .NET:**
>
> ![Validación .NET](img/pantallazo5.2.png)

### 7.3 Validación Stack Angular
Se verifica mediante `ng version` que Angular CLI v19 está operativo sobre Node v22, confirmando la correcta construcción de la imagen.

> **Evidencia Angular:**
>
> ![Validación Angular](img/pantallazo5.3.png)

---

## 8. Conclusiones

La implementación realizada demuestra que es posible estandarizar entornos complejos (.NET, Angular, Python) utilizando contenedores. Se han cumplido los requisitos de la empresa:
1.  **Uso de imágenes personalizadas** para control granular de versiones.
2.  **Documentación técnica** de la infraestructura (Archivos Dockerfile y JSON).
3.  **Validación de seguridad y rendimiento** en cada stack.
