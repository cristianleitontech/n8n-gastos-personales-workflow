# n8n Gastos Personales Workflow

Workflow de n8n para registrar gastos personales de forma automática desde 3 fuentes: facturas en Google Drive, alertas bancarias por Gmail y registro manual por Telegram.

Video completo en YouTube: [Cristian Leiton Tech](https://www.youtube.com/@CristianLeitonTech)

## Qué hace

- **Google Drive**: Monitorea una carpeta. Cuando subes una factura (PDF o imagen), la procesa con OCR (Google Cloud Vision) y extrae los datos con GPT-4o.
- **Gmail**: Detecta correos transaccionales de tu banco, clasifica el tipo (compra, retiro, transferencia, QR) y extrae los datos automáticamente.
- **Telegram**: Le escribes al bot algo como "almuerzo en restaurante 45k" y GPT-4o interpreta el monto, lugar y categoría.

Los 3 flujos convergen en **Google Sheets** donde se registra todo.

## Stack

| Servicio | Uso |
|----------|-----|
| [n8n](https://n8n.io/) (self-hosted) | Orquestador del workflow |
| Google Cloud Vision API | OCR para facturas (PDF e imágenes) |
| Azure OpenAI GPT-4o | Extracción de datos estructurados con IA |
| Google Sheets | Base de datos de gastos |
| Google Drive | Ingesta de facturas |
| Gmail | Ingesta de alertas bancarias |
| Telegram Bot API | Registro manual y notificaciones |
| [Exchange Rate API](https://open.er-api.com) | Conversión USD a COP en tiempo real |

## Requisitos previos

1. **n8n** instalado (self-hosted o cloud)
2. **Google Cloud** con Vision API habilitada y Service Account configurado
3. **Azure OpenAI** con acceso a GPT-4o (o adaptar a OpenAI directo)
4. **Google Sheets** con una hoja que tenga las columnas: `fecha`, `hora`, `establecimiento`, `categoria`, `total`, `metodo_pago`, `fecha_procesado`
5. **Google Drive** con dos carpetas: `Unprocessed` y `Processed`
6. **Bot de Telegram** creado con [@BotFather](https://t.me/BotFather)
7. **Gmail** con OAuth2 configurado en n8n

## Cómo importar

1. Descarga `GastosPersonalesCristianLeitonTech.json`
2. En n8n, ve a **Workflows** > **Import from File**
3. Selecciona el archivo JSON
4. Configura las credenciales (ver sección siguiente)

## Configuración

El archivo JSON tiene placeholders que debes reemplazar con tus propios valores. Puedes hacerlo directamente en n8n después de importar.

### Credenciales que debes configurar en n8n

| Credencial | Nodos que la usan |
|------------|-------------------|
| Google Service Account | Vigilar Carpeta Drive, Descargar Archivo, Guardar en Google Sheets, Mover a Carpeta Procesados, Renombrar Archivo Procesado |
| Azure OpenAI API | GPT-4o Facturas, GPT-4o Emails, GPT-4o Telegram, GPT-4o QR |
| Header Auth (API Key de Vision) | OCR Google Vision - PDF, OCR Google Vision - Imagen |
| Gmail OAuth2 | Recibir Email Bancario |
| Telegram Bot API | Todos los nodos de Telegram (alertas, notificaciones, trigger) |

### Valores que debes cambiar

Busca estos placeholders en los nodos y reemplázalos:

| Placeholder | Dónde | Qué poner |
|-------------|-------|-----------|
| `TU_FOLDER_ID_UNPROCESSED` | Vigilar Carpeta Drive | ID de tu carpeta de Google Drive para facturas nuevas |
| `TU_FOLDER_ID_PROCESSED` | Mover a Carpeta Procesados | ID de tu carpeta de Google Drive para facturas procesadas |
| `TU_SPREADSHEET_ID` | Guardar en Google Sheets | ID de tu Google Sheets |
| `TU_TELEGRAM_CHAT_ID` | Nodos de Telegram | Tu chat ID de Telegram (obtenerlo con `getUpdates`) |
| `alertas@tu_banco.com` | Recibir Email Bancario | Email de notificaciones de tu banco |
| `tu_email_personal@gmail.com` | Recibir Email Bancario | Tu email personal (si recibes recibos ahí) |

### Cómo obtener tu Telegram Chat ID

1. Crea un bot con [@BotFather](https://t.me/BotFather) usando `/newbot`
2. Envíale un mensaje al bot
3. Visita `https://api.telegram.org/bot<TU_TOKEN>/getUpdates`
4. Busca `"chat":{"id": XXXXXXX}` — ese es tu chat ID

### Cómo obtener IDs de Google Drive

Abre la carpeta en Google Drive. La URL tiene el formato:
```
https://drive.google.com/drive/folders/ESTE_ES_TU_FOLDER_ID
```

### Cómo obtener el ID de Google Sheets

Abre tu hoja de cálculo. La URL tiene el formato:
```
https://docs.google.com/spreadsheets/d/ESTE_ES_TU_SPREADSHEET_ID/edit
```

## Estructura del Workflow

```
Google Drive Trigger ──► Descargar ──► Base64 ──► Switch (PDF/Imagen)
                                                    ├── OCR PDF ──► Agregar texto
                                                    └── OCR Imagen ──► Extraer texto
                                                            └──► Mapear ──► IA (GPT-4o) ──► Reglas ──► Tasa USD ──► Convertir ──┐
                                                                                                                                │
Gmail Trigger ──► IA (GPT-4o) ──► Switch (tipo transaccion)                                                                    │
                                    ├── Compra/Retiro ──► Mapear ──► Tasa USD ──► Convertir ──────────────────────────────────────┤
                                    ├── Transferencia ──► Notificar por Telegram                                                │
                                    └── QR ──► Pedir contexto ──► IA ──► Mapear ──► Tasa USD ──► Convertir ──────────────────────┤
                                                                                                                                │
Telegram Trigger ──► IA (GPT-4o) ──► Mapear ──► Tasa USD ──► Convertir ─────────────────────────────────────────────────────────┤
                                                                                                                                │
                                                                                    Google Sheets ◄─────────────────────────────┘
                                                                                        │
                                                                                        ├── Si viene de Drive ──► Mover + Renombrar
                                                                                        └── Si no ──► Construir mensaje ──► Alerta Telegram

Error Trigger ──► Notificar error por Telegram
```

## Personalización

- **Categorías**: Edita los schemas de los nodos `Extraer Datos * (IA)` para agregar tus propias reglas de categorización
- **Reglas de negocio**: El nodo `Preparar Datos Factura` tiene reglas para forzar categorías por nombre de establecimiento
- **Mensajes de Telegram**: El nodo `Construir Mensaje Notificacion` tiene emojis y comentarios personalizables por categoría
- **Umbral de alerta**: En el nodo `Construir Mensaje Notificacion`, cambia el valor `200000` por tu umbral preferido
- **Zona horaria**: Busca `America/Bogota` y cámbiala por tu zona horaria

## ¿Te sirvió?

Si este workflow te ahorró tiempo o te dio ideas para tus propias automatizaciones, suscríbete al canal **[Cristian Leiton Tech](https://www.youtube.com/@CristianLeitonTech)** donde publico más proyectos como este: automatizaciones reales, en producción, con n8n, IA y herramientas cloud.

Dale estrella al repo si te fue útil y déjame un comentario en el video si tienes dudas o quieres que cubra algún tema en particular.

## Licencia

MIT — usa, modifica y comparte libremente.

---

Hecho por [Cristian Leiton Tech](https://www.youtube.com/@CristianLeitonTech)
