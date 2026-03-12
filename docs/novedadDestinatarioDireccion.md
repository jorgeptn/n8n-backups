# 📦 Flujo n8n: Novedad Destinatario Dirección

## 📌 Resumen
Este subflujo de n8n es invocado por un enrutador principal (`slackRouter`). Su objetivo es procesar novedades logísticas provenientes de Slack (canal **akienvio-notifications**) cuando se reporta un **"Problema con el destinatario"**. El flujo extrae los datos del mensaje, normaliza campos críticos como el número de teléfono, construye dinámicamente una plantilla de WhatsApp para solicitar la corrección de la dirección, y guarda la trazabilidad de éxitos y errores en Google Sheets.

## ⚙️ Arquitectura del Flujo

El flujo está dividido conceptualmente en cuatro fases principales:

### 1. Entrada y Consolidación
Recibe el *payload* enrutado desde `slackRouter` (filtrado por el canal `C0653G3LD8W` y el contenido de la novedad). Une todos los fragmentos del mensaje de Slack (`text`, `blocks`, `attachments`, `message`, `previous_message`) en una sola variable `textoCompleto` para facilitar su procesamiento.

### 2. Parseo y Validación de Negocio
Utiliza Expresiones Regulares (Regex) para extraer la información logística:
* **Guía, usuario, paquetería y servicio**
* **Novedad, ciudad, dirección, departamento/apartamento y código postal**
* **Teléfono**

> **Nota:** El flujo valida estrictamente que el tipo de novedad sea *"Problema con el destinatario"* (o variantes de dirección no encontrada/incompleta). Si los datos son inválidos o no corresponden, el registro se marca con `ok=false` y se enruta directamente a la estandarización de errores.

### 3. Normalización y Composición
Se evalúa y transforma la información extraída:
* **Teléfono:** Se limpia a formato numérico. Si tiene 10 dígitos, se le antepone el prefijo de Colombia (`57`). Se conservan el teléfono y la dirección originales para fines de auditoría.
* **Construcción del Mensaje:** Se genera una plantilla de WhatsApp (`mensajeWhatsapp`) que incluye un saludo, el aviso de que la transportadora no encuentra la dirección, y solicita datos exactos (dirección, complementos, barrio, referencias). También se genera el enlace directo (`https://wa.me/...`).
* **Metadatos:** Se añaden datos de trazabilidad (`canal`, `timestampSlack`, `workflow`, `tipoNovedad`, `guia`).

### 4. Persistencia y Cierre
Dependiendo de si ocurrieron errores en las fases anteriores:
* **Ruta de Éxito:** Escribe en la hoja de éxitos de Google Sheets (Guía, Teléfono, Paquetería, Ciudad, Novedad, Mensaje, Canal, Timestamp).
* **Ruta de Error:** Genera un JSON estandarizado (`stage`, `error`, `resumen del payload`) y lo escribe en la hoja de errores para facilitar el debugging. 

---

## 🛠️ Detalle de los Nodos

| Nodo | Tipo | Descripción |
| :--- | :--- | :--- |
| **When Executed by Another Workflow** | `Trigger` | Punto de entrada passthrough desde `slackRouter` (`workflowId: yoxhAe4eAWiZhPd9`). |
| **Consolidar texto Slack** | `Code (JS)` | Une todas las propiedades del payload de Slack en un único `textoCompleto`. |
| **Validar y extraer datos** | `Code (JS)` | Extrae datos logísticos con regex y valida que la novedad sea un problema de destinatario. |
| **Datos validos?** | `If` | Verifica que el parámetro `ok` retornado por el código anterior sea `true`. |
| **Normalizar datos** | `Code (JS)` | Limpia el teléfono a formato numérico, añade el prefijo `57` si aplica, y guarda copias originales. |
| **Construir mensaje y enlace** | `Code (JS)` | Crea la plantilla estructurada de WhatsApp para corrección de dirección y adjunta metadatos. |
| **Sin errores?** | `If` | Verifica que no existan errores críticos tras la normalización. |
| **Guardar exito en hoja** | `Google Sheets` | Hace un *append* a la hoja destino de éxitos con la información y el enlace de WhatsApp. |
| **Estandarizar error** | `Code (JS)` | Recolecta motivos de falla de cualquier etapa y genera un JSON uniforme de error controlado. |
| **Guardar error en hoja** | `Google Sheets` | Hace un *append* a la hoja de errores con el contexto del fallo para auditoría. |
| **Fin OK / Fin Error** | `NoOp` | Nodos visuales para marcar la finalización exitosa o fallida del flujo. |

---

## 🔐 Credenciales Requeridas
* **Google Sheets API:** Credencial requerida para lectura y escritura (*append*) en el documento de trazabilidad de éxitos y errores.
* **Slack Trigger:** Integración activa en el flujo padre (`slackRouter`) para escuchar los eventos del canal **akienvio-notifications**.

---

## 📝 Ejemplo de Payload Esperado (Input)

El nodo inicial espera el payload transmitido por el router con la estructura nativa de Slack. Ejemplo resumido:

```json
{
  "channel": "C0653G3LD8W",
  "ts": "1773270000.123456",
  "attachments": [
    {
      "text": "guía 75352527615\nUSUARIO: cliente@correo.com\nPAQUETERIA: Coordinadora\nSERVICIO: Contraentrega\nNOVEDAD: Problema con el destinatario...\n--DESTINO--\nCIUDAD: Bogotá\nDIRECCIÓN: Calle 123 # 45-67\nDEP/APTO: 402\nZIP_CODE: 110111\nTELÉFONO: 300 111 2233"
    }
  ]
}
