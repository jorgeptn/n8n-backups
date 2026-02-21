# Documentación de Automatización: Registro y Notificación Logística de Sucursales

**Propósito del Flujo:**
Automatizar el ciclo de vida de la creación de sucursales para Akienvio. El sistema opera en dos fases: primero, captura en tiempo real las solicitudes de creación desde Slack y las consolida en una base de datos central en Google Sheets; segundo, ejecuta un proceso por lotes diario para generar un reporte en Excel y enviarlo automáticamente a los aliados operativos (equipo de Inter Rapidísimo), actualizando el estado de los registros procesados para evitar duplicidades.

## Fase 1: Ingreso de Datos (Tiempo Real)
Esta sección del flujo se encarga de la ingesta continua de información a medida que ocurren las notificaciones operativas.

1. **Trigger (Slack): `address-notifications`**
   * **Acción:** Escucha de forma activa cualquier mensaje nuevo que ingrese al canal designado en Slack.
2. **Procesamiento (Code): `JavaScript: Transformar Data`**
   * **Acción:** Recibe el payload de Slack y, mediante expresiones regulares (Regex), extrae y limpia los campos clave del bloque de texto (Alias/Empresa, Dirección, Teléfono, Ciudad, Email de soporte). 
   * **Regla de Negocio:** Asigna por defecto el NIT de Akienvio y estructura la salida como un objeto JSON estandarizado.
3. **Base de Datos (Google Sheets): `Incluir Data en Fila de archivo`**
   * **Acción:** Inserta una nueva fila en el archivo maestro *AKIENVIO S.A.S SUCURSALES*.
   * **Regla de Negocio:** Marca automáticamente la columna `¿ENVIADO POR CORREO?` con el valor **"NO"**, dejándola en cola para el reporte diario.

---

## Fase 2: Reporte Diario y Cierre de Ciclo (Por Lotes)
Esta sección se encarga de la consolidación, notificación y actualización de estados, ejecutándose una vez al día.

1. **Trigger Programado (Schedule Trigger)**
   * **Acción:** Inicia la ejecución diaria estrictamente a las **17:00 horas**.
2. **Consulta (Google Sheets): `Get row(s) in sheet`**
   * **Acción:** Escanea el archivo maestro y extrae únicamente las filas donde la columna `¿ENVIADO POR CORREO?` es igual a **"NO"**.
3. **Enrutamiento Lógico (If): `Hay sucursales?`**
   * **Condición:** Evalúa si la cantidad de filas pendientes es mayor a cero (`{{ $input.all().length > 0 }}`).
   * **Ruta False:** Si no hay sucursales nuevas, el flujo se dirige al nodo `No Operation, do nothing` y finaliza exitosamente sin consumir recursos adicionales.
   * **Ruta True:** Si hay datos pendientes, el flujo continúa hacia la generación del reporte.
4. **Generación de Documento (Google Drive): `Copy file`**
   * **Acción:** Clona el archivo *plantillaSucursales* y lo renombra dinámicamente incluyendo la fecha del día.
5. **Restauración de Contexto (Code): `Restaurar Filas 1`**
   * **Acción:** Puente de memoria que recupera el listado completo de sucursales consultadas en el paso 2 para inyectarlas en el siguiente nodo.
6. **Poblado de Datos (Google Sheets): `Append row in sheet1`**
   * **Acción:** Inserta la información detallada de todas las sucursales pendientes dentro del nuevo archivo clonado.
7. **Conversión (Google Drive): `Download file`**
   * **Acción:** Descarga el documento recién poblado convirtiéndolo en un archivo binario en formato Excel (`.xlsx`).
8. **Distribución (Gmail): `Send a message`**
   * **Acción:** Envía un correo corporativo con formato HTML al equipo de Inter Rapidísimo y copias internas.
   * **Contenido:** Adjunta el archivo Excel y utiliza una plantilla con los colores corporativos y la firma del Director de Operaciones.
9. **Restauración de Contexto Final (Code): `Restaurar Filas 2`**
   * **Acción:** Recupera nuevamente el `row_number` de las filas procesadas originalmente para permitir su actualización individual.
10. **Cierre de Ciclo (Google Sheets): `Update row in sheet`**
    * **Acción:** Busca las filas procesadas en el archivo maestro y cambia el valor de la columna `¿ENVIADO POR CORREO?` a **"SI"**. Esto garantiza que no se vuelvan a reportar al día siguiente.
