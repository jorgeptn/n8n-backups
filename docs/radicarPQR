# 🤖 Automatización de PQRs con n8n y DeepSeek

Este repositorio contiene el flujo de n8n diseñado para automatizar la gestión, redacción y registro de PQRs (Peticiones, Quejas y Reclamos) para **Akienvio SAS**. 

El flujo recibe un archivo CSV a través de un chat, procesa cada registro individualmente utilizando un Agente de Inteligencia Artificial (DeepSeek) para redactar correos formales y legales, y finalmente crea borradores listos para ser enviados desde una cuenta corporativa de Gmail.

## 🚀 Características Principales

* **Procesamiento Masivo:** Capacidad para extraer y procesar múltiples registros desde un único archivo CSV adjunto en el chat.
* **Redacción Asistida por IA:** Utiliza **DeepSeek-Reasoner** integrado con LangChain para transformar datos crudos en textos formales, citando normativas cuando es necesario.
* **Plantilla HTML Corporativa:** Los correos generados se inyectan automáticamente en una plantilla HTML con el branding corporativo de la empresa.
* **Gestión de Errores y Logs:** Sistema robusto de observabilidad que estandariza los errores y registra el éxito o fracaso de cada ejecución para facilitar la auditoría.

## 📋 Estructura de Datos Requerida (CSV)

Para que el Agente IA funcione correctamente, el archivo CSV de entrada debe contener (como mínimo) las siguientes columnas:

* `paqueteria`: Nombre de la transportadora (ej. Coordinadora, Servientrega).
* `guia`: Número de seguimiento del paquete (identificador inmutable).
* `tipoPQR`: Motivo de la solicitud (ej. `guiaRetrasado`, `guiaMalEntregado`).
* `correo`: Dirección de correo electrónico del destinatario.
* `ultimoMovimiento`: Fecha del último estado registrado del paquete.
* `contextoPQR`: Detalles y observaciones adicionales para contextualizar el caso.

## ⚙️ Arquitectura del Flujo

El flujo de trabajo está dividido en cuatro fases lógicas:

### 1. Entrada y Validación
* **When chat message received:** Disparador que recibe el mensaje de texto o el archivo binario (CSV) a través del chat.
* **Validar y Normalizar Entrada (Code):** Extrae el contenido del CSV, lo divide en filas individuales y le añade metadatos de trazabilidad (`executionId`).
* **IF Entrada Invalida:** Enruta el proceso hacia el registro de errores si el documento está vacío o no tiene el formato correcto.

### 2. Generación con IA
* **AI Agent:** Un agente de LangChain que procesa la fila del CSV apoyado por un prompt de negocio especializado en PQRs. 
* **DeepSeek Chat Model:** El LLM encargado de interpretar el contexto, redactar el correo y devolver un objeto JSON estricto (`para`, `asunto`, `cuerpo`, `guia`).
* **Simple Memory:** Mantiene la coherencia de la sesión durante el procesamiento de la información.

### 3. Parseo y Preparación de Salida
* **Code in JavaScript:** Limpia las etiquetas Markdown que pueda generar la IA (como ````json ````) y convierte el texto plano en un objeto JSON válido.
* **IF Error Parseo JSON:** Valida que la estructura del JSON sea correcta antes de continuar.
* **Split Out:** Divide el arreglo de correos generados en *items* individuales para que n8n pueda procesar cada borrador por separado.

### 4. Salida y Observabilidad
* **Create a draft (Gmail):** Toma el JSON estructurado e inyecta el `asunto` y el `cuerpo` dentro de una plantilla HTML corporativa, guardándolo como borrador en la cuenta conectada.
* **Nodos de Log (Éxito / Error Estándar):** Finalizan el proceso registrando metadatos cruciales (`workflowVersion`, `finishedAt`, `status`) para mantener un historial limpio de las operaciones del sistema.

## 🛠️ Instalación y Configuración

### Prerrequisitos
1. Instancia de n8n operativa (versión que soporte los nodos `@n8n/n8n-nodes-langchain`).
2. Cuenta de Google Workspace / Gmail con acceso a la API habilitado.
3. API Key de DeepSeek.

### Pasos de importación
1. Clona este repositorio o descarga el archivo `.json` del flujo.
2. En tu instancia de n8n, ve a **Workflows** y haz clic en **Import from File** (o pega el código JSON directamente en el lienzo en blanco).
3. Configura las siguientes credenciales requeridas por los nodos:
   * **Gmail account:** Conecta mediante OAuth2 la cuenta de correo corporativa (ej. `soporte@akienvio.co`).
   * **DeepSeekPQRRadicador:** Ingresa tu API Key de DeepSeek en el nodo de modelo de lenguaje.
4. Activa el flujo de trabajo.
