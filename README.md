# Estandarización de Entornos de Desarrollo (Dev Containers)

![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white)
![Visual Studio Code](https://img.shields.io/badge/Visual%20Studio%20Code-0078d7.svg?style=for-the-badge&logo=visual-studio-code&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

Este repositorio aloja la **documentación técnica y los estándares de configuración** para los entornos de desarrollo contenerizados de la compañía.

El objetivo es eliminar la fricción en la configuración de entornos locales ("en mi máquina funciona"), garantizar la seguridad mediante prácticas de **DevSecOps** y optimizar el uso de recursos en equipos con hardware limitado.

## 📂 Definición de Stacks Tecnológicos

A continuación se detallan los perfiles estandarizados. Haz clic en "Ver Código" para acceder a la configuración específica (`Dockerfile` y `devcontainer.json`) dentro del informe técnico.

| Stack | Tecnología Base | Enfoque Principal | Configuración |
| :--- | :--- | :--- | :--- |
| **Scripting** | Python 3.12 | **Seguridad** (Usuario no-root `vscode`) | [Ver Código](./documentacion.md#51-stack-de-scripting-python) |
| **Backend** | .NET 9.0 SDK | **Productividad** (Herramientas pre-instaladas) | [Ver Código](./documentacion.md#52-stack-backend-NET) |
| **Frontend** | Angular 19 / Node 22 | **Rendimiento** (Volúmenes para `node_modules`) | [Ver Código](./documentacion.md#53-stack-frontend-angular) |

## 📚 Documentación Completa

Este repositorio centraliza toda la investigación, las decisiones de arquitectura y las guías de implementación en un único documento maestro:

👉 **[LEER INFORME TÉCNICO DE IMPLEMENTACIÓN](./INFORME_TECNICO.md)**

En este documento encontrarás:
1.  Justificación del uso de **Imágenes Personalizadas**.
2.  Instrucciones para la **preparación del Host (Linux)**.
3.  Explicación del **ciclo de vida de compilación** en VS Code.
4.  **Evidencias de validación** de cada entorno.

## 🚀 Guía de Uso Rápido

Para implementar cualquiera de estos entornos en un nuevo proyecto:

1.  **Consultar:** Navega al stack deseado usando los enlaces de la tabla superior.
2.  **Copiar:** Copia el contenido de los bloques de código `Dockerfile` y `devcontainer.json` del informe.
3.  **Implementar:**
    * Crea una carpeta `.devcontainer` en la raíz de tu proyecto.
    * Pega los archivos copiados dentro.
4.  **Ejecutar:** Abre el proyecto en VS Code y selecciona **"Reopen in Container"**.

## 🛠️ Estructura del Repositorio

```text
.
├── img/                   # Capturas de pantalla y evidencias de validación
├── INFORME_TECNICO.md     # Manual técnico con todo el código y explicaciones
└── README.md              # Este archivo de portada
