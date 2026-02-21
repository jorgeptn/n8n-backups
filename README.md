# ⚙️ Respaldos de Automatizaciones n8n - Akienvio

Bienvenido al repositorio central de copias de seguridad de los flujos de trabajo de n8n para **Akienvio**.

## 📌 Propósito
Este repositorio almacena automáticamente el código en formato JSON de todos nuestros flujos de n8n. Esto nos permite tener un control de versiones, auditar cambios y restaurar cualquier proceso en caso de un fallo en el servidor.

## 🚀 ¿Cómo funciona el respaldo?
Contamos con un flujo maestro en n8n que se ejecuta diariamente. Este flujo:
1. Lee todos los flujos activos e inactivos mediante la API de n8n.
2. Compara el estado actual con el último respaldo guardado en GitHub.
3. Si hay un flujo nuevo, crea el archivo `.json`.
4. Si un flujo existente fue modificado, actualiza el archivo correspondiente.

## 📂 Estructura de Documentación
Para ver el detalle técnico y operativo de cada flujo individual, por favor consulta la carpeta `/docs` o lee los archivos Markdown correspondientes a cada proceso logístico.
