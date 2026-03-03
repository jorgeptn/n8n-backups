# Documentación de Automatización: Registro y Notificación Logística de Sucursales

**Propósito del Flujo:**
Automatizar el ciclo de vida de la creación de sucursales para Akienvio. El sistema opera en dos fases: primero, captura en tiempo real las solicitudes de creación desde Slack y las consolida en una base de datos central en Google Sheets; segundo, ejecuta un proceso por lotes diario para generar un reporte en Excel y, tras **un proceso de aprobación manual**, enviarlo automáticamente a los aliados operativos (equipo de Inter Rapidísimo), actualizando el estado de los registros procesados para evitar duplicidades.

## Fase 1: Ingreso de Datos (Tiempo Real)
Esta sección del flujo se encarga de la ingesta continua de información a medida que ocurren las notificaciones operativas.

1. **Trigger (Slack): `address-notifications`**
   * **Acción:** Escucha de forma activa cualquier mensaje de tipo "message" que ingrese al canal designado en Slack.
2. **Procesamiento (Code): `JavaScript: Transformar Data`**
   * **Acción:** Recibe el payload de Slack y, mediante expresiones regulares (Regex), extrae y limpia los campos clave del bloque de texto (Alias/Empresa, Dirección, Teléfono, Ciudad, Email de soporte).
   * **Regla de Negocio:** Estructura la salida como un JSON estandarizado, asignando por defecto el NIT de Akienvio (`901702784-7`).
3. **Base de Datos (Google Sheets): `Incluir Data en Fila de archivo`**
   * **Acción:** Inserta la información recibida como una nueva fila en el archivo maestro *AKIENVIO S.A.S SUCURSALES*.
   * **Regla de Negocio:** Marca automáticamente la columna `¿ENVIADO POR CORREO?` con el valor "NO" y establece "ESPORADICA" en el horario de recogida.

## Fase 2: Procesamiento por Lotes, Aprobación y Distribución (Diario)
Esta fase agrupa la información pendiente diariamente, genera los reportes, exige validación manual y los distribuye.

1. **Disparador de Tiempo (Schedule Trigger): `Schedule Trigger1`**
   * **Acción:** Ejecuta automáticamente esta fase del flujo todos los días a las 17:00 (5:00 PM).
2. **Consulta de Pendientes (Google Sheets): `Get row(s) in sheet1`**
   * **Acción:** Busca y extrae todas las filas del archivo maestro donde la columna `¿ENVIADO POR CORREO?` sea estrictamente igual a "NO".
3. **Validación de Datos (If): `If1`**
   * **Acción:** Evalúa si la consulta anterior obtuvo datos (`$input.all().length > 0`). Si está vacía, el flujo termina silenciosamente (`No Operation, do nothing1`); si existen datos, continúa.
4. **Generación de Documento (Google Drive): `Copy file1`**
   * **Acción:** Clona el archivo *plantillaSucursales* y lo renombra dinámicamente utilizando la fecha del día.
5. **Restauración de Contexto (Code): `Code in JavaScript2`**
   * **Acción:** Nodo puente que recupera el listado completo de sucursales consultadas en el paso 2 para inyectarlas de vuelta en el flujo principal.
6. **Poblado de Datos (Google Sheets): `Append row in sheet`**
   * **Acción:** Inserta la información detallada de todas las sucursales pendientes dentro de la nueva hoja de cálculo clonada.
7. **Conversión a Excel (Google Drive): `Download file1`**
   * **Acción:** Descarga el documento recién poblado convirtiéndolo desde el formato de Google Sheets a un archivo binario en formato Excel (`.xlsx`).
8. **Solicitud de Autorización (Gmail): `Solicitar Aprobación`**
   * **Acción:** Envía un correo con formato HTML y el archivo Excel adjunto para solicitar la validación. Contiene enlaces interactivos para "Aprobar" o "Rechazar" el envío.
9. **Pausa del Flujo (Wait): `Wait for Approval`**
   * **Acción:** Detiene la ejecución en n8n hasta recibir la confirmación manual mediante un Webhook accionado desde el correo anterior.
10. **Verificación de Decisión (If): `Verificar Aprobación`**
    * **Acción:** Analiza el payload del Webhook. Si la decisión es estrictamente igual a `aprobar`, el flujo avanza.
11. **Recuperación de Adjunto (Code): `Recuperar Archivo`**
    * **Acción:** Extrae nuevamente el binario del archivo Excel descargado en el paso 7 para prepararlo para su distribución.
12. **Distribución a Aliados (Gmail): `Send a message1`**
    * **Acción:** Envía un correo corporativo al equipo de Inter Rapidísimo y con copias internas, entregando oficialmente el archivo autorizado.
13. **Restauración de Contexto Final (Code): `Code in JavaScript3`**
    * **Acción:** Recupera una vez más el arreglo original de las filas procesadas.
14. **Cierre de Ciclo (Google Sheets): `Update row in sheet1`**
    * **Acción:** Itera sobre el `row_number` de las filas procesadas en el archivo maestro original y actualiza el valor de la columna `¿ENVIADO POR CORREO?` a "SI".
