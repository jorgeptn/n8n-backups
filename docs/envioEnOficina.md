# 📦 Flujo n8n: Notificación de Novedad en Oficina (Inter Rapidísimo)

## 📌 Resumen
Este subflujo de n8n es invocado por otro workflow principal. Su objetivo es recibir notificaciones de paquetería (ej. desde Slack), extraer datos clave mediante expresiones regulares (guía, teléfono, ciudad, novedad), verificar si el paquete está listo para recoger en una oficina, y generar un enlace dinámico de WhatsApp para notificar al cliente final.

## ⚙️ Arquitectura del Flujo

El flujo está dividido conceptualmente en tres fases principales:

### 1. Entrada y Validación
Recibe el *payload* desde el flujo padre. Extrae todo el texto (incluyendo bloques anidados) y utiliza Expresiones Regulares (Regex) para identificar:
* **Número de Guía**
* **Teléfono**
* **Ciudad**
* **Novedad**

> **Nota:** El flujo filtra específicamente las novedades que contienen los textos *"Tu pedido ha llegado a nuestra oficina"* y *"Está listo para ser reclamado"*. Si faltan datos o no es una novedad de oficina, el registro se ignora o se enruta al control de errores.

### 2. Enriquecimiento de Datos
A partir de la ciudad extraída, el flujo consulta una base de datos en Google Sheets (`Oficinas`, pestaña `oficinaList`) para buscar la dirección exacta de la oficina correspondiente al municipio.

### 3. Salida y Control de Errores
Se evalúa la información consolidada:
* **Teléfono:** Se limpia y normaliza (si tiene 10 dígitos, se asume formato de Colombia y se le agrega el prefijo `57`).
* **Construcción del Mensaje:** * *Ruta con oficina:* Mensaje que incluye la dirección física encontrada en el paso anterior.
  * *Ruta sin oficina:* Mensaje genérico con un enlace web para que el usuario busque la oficina más cercana.
* **Escritura Final:** Si todo es correcto, se registra el número de guía y el enlace de WhatsApp generado en una hoja de Google Sheets. Los errores de validación son capturados y estandarizados para el monitoreo aguas arriba.

---

## 🛠️ Detalle de los Nodos

| Nodo | Tipo | Descripción |
| :--- | :--- | :--- |
| **When Executed by Another Workflow** | `Trigger` | Punto de entrada passthrough desde el workflow principal. |
| **Validar y extraer datos** | `Code (JS)` | Extrae texto, aplica regex (`/guía\s*(\d+)/i`, `/TELÉFONO:\s*(\d+)/`, `/CIUDAD:\s*(.+)/`), valida la condición de la novedad y retorna un objeto estructurado. |
| **Datos válidos?** | `If` | Verifica que el parámetro `ok` retornado por el código anterior sea `true`. |
| **Buscar oficina por ciudad** | `Google Sheets` | Busca coincidencias en la columna *Municipio* usando la variable `CIUDAD`. |
| **Construir mensaje y enlace** | `Code (JS)` | Aplica *fallbacks* para evitar pérdida de datos, normaliza el número telefónico, construye la plantilla de texto y genera la URL de WhatsApp. |
| **Sin errores?** | `If` | Verifica que no existan errores de datos críticos tras la normalización. |
| **Guardar enlace en hoja** | `Google Sheets` | Hace un *append* a la hoja destino con las columnas `Guia` y `Mensaje`. |
| **Estandarizar error** | `Code (JS)` | Recolecta motivos de falla y genera un JSON uniforme de error para facilitar la depuración. |

---

## 🔐 Credenciales Requeridas
* **Google Sheets OAuth2 API:** El flujo utiliza una cuenta de Google, para leer el directorio de oficinas y escribir los enlaces de salida. Asegúrate de que las credenciales estén mapeadas correctamente en el entorno de despliegue.

---

## 📝 Ejemplo de Payload Esperado (Input)

El nodo inicial espera un JSON (frecuentemente proveniente de un evento de Slack) con una estructura que incluya `text` o `blocks`. Ejemplo simplificado:

```json
{
  "text": "Ocurrió una novedad en la guía 240047389670...",
  "blocks": [
    {
      "text": {
        "text": "CIUDAD: BOGOTA\nTELÉFONO: 3001234567\nNOVEDAD: Tu pedido ha llegado a nuestra oficina..."
      }
    }
  ]
}
