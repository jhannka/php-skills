---
name: engram-save-sync
description: Revisa memorias, guarda cambios en engram incluyendo prompts, sincroniza con la nube y resume.
---
# Engram Save and Sync Workflow

Esta skill automatiza el proceso de guardar el progreso en Engram, asegurando la captura de prompts y la sincronización con la nube.

## Triggers
- Cuando el usuario pida guardar y sincronizar ("guarda y sincroniza", "save and sync", "guardar cambios en engram y la nube").

## Instrucciones Paso a Paso

Debes seguir estrictamente estos pasos:

### 1. Revisar Memorias Existentes
- Ejecuta `mem_context` (y `mem_search` si es necesario) para verificar si el trabajo y los cambios recientes ya han sido guardados en la memoria para este proyecto.
- **Si ya existen memorias con los cambios:** Muestra un resumen de lo que ya se encuentra guardado al usuario y pregunta si aún así desea agregar algo más.
- **Si no existen memorias con los cambios:** Procede al Paso 2.

### 2. Guardar Memorias y Prompts
- Antes de guardar, ejecuta `mem_current_project` para confirmar el nombre del proyecto.
- Llama a `mem_save_prompt` para registrar explícitamente el prompt del usuario asociado a este trabajo.
- Llama a `mem_save` detallando el trabajo realizado (decisiones, correcciones, descubrimientos). Asegúrate de pasar explícitamente el nombre del proyecto y dejar `capture_prompt: true` (o no establecerlo a false) para vincular el prompt guardado.
- Llama a `mem_session_summary` si este es el final de una sesión de trabajo.

### 3. Sincronizar con Engram Cloud
- Llama a la herramienta o comando necesario para sincronizar el estado actual a la nube. Si existe la skill `engram-cloud-setup`, puedes invocarla o apoyarte en ella para correr el comando de sincronización. Si usas la terminal, asume los comandos estándar de sincronización de engram (ej. `agy engram sync`).

### 4. Resumir lo Guardado
- Provee un resumen al usuario explicando clara y concisamente las memorias exactas que fueron registradas en este proceso.

### 5. Rectificación Final
- Al final de tu mensaje al usuario, **DEBES** incluir una sección de rectificación explícita que valide que se cumplieron todos los puntos sin excepciones. Por ejemplo:
  > **Rectificación de puntos:**
  > 1. Revisión de memorias previas: [OK]
  > 2. Guardado de nuevas memorias y prompts: [OK]
  > 3. Sincronización con Engram Cloud: [OK]
  > 4. Resumen presentado: [OK]
