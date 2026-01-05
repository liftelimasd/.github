# Documentación de GitHub Workflows

Este documento describe los workflows de CI/CD utilizados en la organización `liftelimasd`.

## ⚠️ Deprecated (Obsoletos)

Los siguientes workflows han sido reemplazados por nuevas versiones más eficientes y centralizadas. No se recomienda su uso en nuevos proyectos.

### ❌ `pr-check.yml`
*   **Estado:** Deprecated.
*   **Función original:** Ejecutar tests y linter en cada Pull Request.
*   **Reemplazo:** Integrado en pipelines más modernos o no requerido en el flujo actual simplificado.

### ❌ `build-push.yml`
*   **Estado:** Deprecated.
*   **Función original:** Construcción y subida de imágenes Docker con gestión compleja de caché y notificaciones.
*   **Reemplazo:** Reemplazado por `kratos-deploy-pipeline.yml` (centralizado) y `deploy.yml` (local).

---

## ✅ Workflows Activos (KRATOS)

### 🛠️ 1. `kratos-deploy-pipeline.yml` (Centralizado)
Este es el "motor" del despliegue. Se almacena en un único lugar y es reutilizado por todos los proyectos.

*   **Ubicación:** `liftelimasd/.github/.github/workflows/kratos-deploy-pipeline.yml` (Repositorio de organización).
*   **Función:** Construir la imagen Docker, inyectar configuración y subirla al registro privado.
*   **Flujo de trabajo:**
    1.  **Checkout:** Descarga el código del proyecto.
    2.  **Versión:** Determina la versión (manual o automática desde git tag).
    3.  **Configuración:** Lee `service/configs/config.yaml` para obtener el nombre del servicio y los puertos.
    4.  **Secretos:** Si existe el secreto `ENV_FILE`, crea un archivo `.env` y lo incluye en la imagen.
    5.  **Build & Push:** Construye la imagen Docker optimizada (sin caché externa para velocidad) y la sube a `registry.liftel.es:5000`.
    6.  **Resumen:** Muestra una tabla limpia con el resultado del despliegue.

### 🚀 1.1. `deploy.yml` (Local en cada proyecto)
Este es el archivo que debe estar presente en cada repositorio de servicio (ej. `testproject`). Actúa como un "lanzador".

*   **Ubicación:** `.github/workflows/deploy.yml` (en el repositorio del proyecto).
*   **Función:** Iniciar el despliegue manual del servicio utilizando la lógica centralizada de la organización.
*   **Cómo usar:**
    1. Ir a la pestaña **Actions** en GitHub.
    2. Seleccionar **Auto Run Deploy Pipeline - Kratos Service**.
    3. Hacer clic en **Run workflow**.
    4. (Opcional) Escribir una versión específica (ej. `v2.0`). Si se deja vacío, usa el último tag de git.
*   **Características:**
    *   No contiene lógica compleja, solo llama a `kratos-deploy-pipeline.yml`.
    *   Pasa automáticamente los secretos (`REGISTRY_LOGIN`, `REGISTRY_PASS`, `ENV_FILE`).

*   **Contenido:**
```yml
name: Auto Run Deploy Pipeline - Kratos Service 

on:
  workflow_dispatch:
    inputs:
      version:
        description: 'Version (leave empty for latest git tag)'
        required: false
        default: ''

jobs:
  deploy:
    uses: liftelimasd/.github/.github/workflows/kratos-deploy-pipeline.yml@main
    with:
      version: ${{ inputs.version }}
      copy_env: true # Optional: defaults to true inside pipeline anyway
    secrets:
      REGISTRY_LOGIN: ${{ secrets.REGISTRY_LOGIN }}
      REGISTRY_PASS: ${{ secrets.REGISTRY_PASS }}
      ENV_FILE: ${{ secrets.ENV_FILE }}

```


### 🧹 2. `cleanup.yml` (Mantenimiento Global)
Script de mantenimiento para limpiar el historial de ejecuciones de GitHub Actions en **toda la organización**.

*   **Ubicación:** `liftelimasd/.github/.github/workflows/cleanup.yml` (Repositorio de organización).
*   **Trigger:** Manual (`workflow_dispatch`).
*   **Función:** Recorrer **todos** los repositorios de la organización y eliminar el historial antiguo de ejecuciones para ahorrar espacio y mantener el orden.
*   **Lógica:**
    *   Itera sobre cada repositorio de `liftelimasd`.
    *   Obtiene todas las ejecuciones de workflows.
    *   **Conserva:** Solo las 3 ejecuciones más recientes de cada repositorio (independientemente del tipo).
    *   **Elimina:** Todo el resto.
*   **Requisitos:** Necesita un token con permisos de administración de organización (`ORG_CLEANUP_TOKEN_RAUL`).

