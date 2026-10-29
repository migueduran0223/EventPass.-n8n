# EventPass 

## Alcance

Este repositorio contiene la automatización backend del proyecto inicial EventPass: workflows n8n WF01–WF10 y persistencia independiente en Google Sheets. No incluye frontend, por decisión de alcance. Un cliente futuro puede comunicarse con los webhooks de n8n mediante HTTP; no debe conectarse directamente a Sheets.

La evaluación de check-in (WF11) pertenece a otro repositorio y no forma parte de este proyecto inicial.

## Arquitectura

```text
Cliente HTTP / Formulario n8n / Bot Telegram / Chat de n8n
                         ↓
                 Workflows WF01–WF10
                         ↓
       Reglas de negocio y persistencia en Google Sheets
                         ↓
              Gmail + Telegram + IA informativa
```

## Workflows

| Workflow | Trigger | Responsabilidad |
|---|---|---|
| WF01 — Usuarios CRUD | Webhook POST `/eventpass/usuarios` | Crear usuarios, consultar y actualizar el propio perfil, desactivar cuenta; validación de email y hashing de contraseña. |
| WF02 — Autenticación y sesiones | Webhook POST `/eventpass/auth` | Login, logout y validación de sesiones con expiración de 24 horas. |
| WF03a — Solicitar código Telegram | Webhook POST `/eventpass/telegram/codigo` | Emite un código temporal para una sesión válida. |
| WF03b — Procesar vinculación Telegram | Telegram Trigger | Recibe `/vincular CODIGO`, valida vigencia y uso, y guarda el chat_id. |
| WF04 — Administración de eventos | Form Trigger n8n | Crear, leer, actualizar, cerrar o cancelar eventos; uso administrativo desde n8n. |
| WF05 — Catálogo público | Webhook GET `/eventpass/catalogo` | Lista, detalle y filtro por categoría; calcula disponibilidad desde capacidad menos CONFIRMADAS. |
| WF06 — Inscripciones CRUD | Webhook POST `/eventpass/inscripciones` | Crear, listar, actualizar datos permitidos o cancelar; valida sesión, Telegram, cupos y duplicados. |
| WF07 — Lista de espera | Schedule cada 5 minutos | Promueve inscripciones en orden FIFO cuando se libera un cupo. Zona horaria: America/Bogota. |
| WF08 — Recordatorios | Schedule cada 30 minutos | Detecta recordatorios de 24 h y 1 h y evita duplicados con clave usuario-evento-tipo. Zona horaria: America/Bogota. |
| WF09 — Notificaciones | Execute Sub-workflow Trigger | Centraliza Gmail y Telegram, registra el resultado por canal y evita repetir los canales ya enviados para la misma clave idempotente. |
| WF10 — EventPass Assistant | Chat Trigger alojado por n8n | Asistente informativo con memoria; consulta solo eventos publicados y guarda conversaciones y mensajes. No ejecuta operaciones del negocio. |

WF03 se divide en dos workflows porque usa triggers distintos: webhook y Telegram Trigger.

## Endpoints

Los workflows están guardados inactivos. Una vez publicados, las rutas de producción usan este formato:

`https://miguelrosario0221.app.n8n.cloud/webhook/<ruta>`

- `POST /eventpass/usuarios`
- `POST /eventpass/auth`
- `POST /eventpass/telegram/codigo`
- `GET /eventpass/catalogo`
- `POST /eventpass/inscripciones`

WF04 publica un formulario de n8n al activarse. WF03b recibe mensajes directamente del bot Telegram. WF10 se usa mediante el chat alojado por n8n.

## Google Sheets

Cada dominio tiene un archivo independiente. Las credenciales OAuth están almacenadas en n8n y no están escritas en este repositorio.

| Archivo | Hojas |
|---|---|
| [EP01_Usuarios](https://docs.google.com/spreadsheets/d/1--0E5mKFWKdowk0MgFPNhUevMPGJGMZ7WzsEzCkqa3Q/edit) | Usuarios, Auditoria_Usuarios |
| [EP02_Sesiones](https://docs.google.com/spreadsheets/d/11kqf6drIGzfc-SoLxOWoML9rIASGF7i-T5-bFeXkZa4/edit) | Sesiones |
| [EP03_Telegram](https://docs.google.com/spreadsheets/d/1tmNkyXAXnoSXLMaDc-yKMKDGuj3obmMPONUlQrhvLVs/edit) | Codigos_Vinculacion, Vinculaciones |
| [EP04_Eventos](https://docs.google.com/spreadsheets/d/1poduWlID1AnfihGcgojYTl6gF61-QpJAfqS8tWJywVY/edit) | Eventos, Auditoria_Eventos |
| [EP05_Catalogo_Log](https://docs.google.com/spreadsheets/d/1HYzJF8RPMn9qUkWRcxNu6O_mBf9dy4qXXKjv88iyFRc/edit) | Consultas_Catalogo |
| [EP06_Inscripciones](https://docs.google.com/spreadsheets/d/12cYmDkPbowVQC3dCOlnWL5LXeFmo0wXQU6EWFtYcD7I/edit) | Inscripciones, Auditoria_Inscripciones |
| [EP07_Reasignaciones](https://docs.google.com/spreadsheets/d/1kX3n1LpoDUF3VD5DeM3uIG9LJSr70x8yh0ItWvuRltA/edit) | Reasignaciones |
| [EP08_Recordatorios](https://docs.google.com/spreadsheets/d/1rYnYdNmomX0P1DEzDh1R1aw0eMi5OzmXUQMiiHEPA4U/edit) | Recordatorios |
| [EP09_Notificaciones](https://docs.google.com/spreadsheets/d/1w5ABRhvi3-p_vdjyfI-yZTZTJHGxgTvkJHZTcGyP-Jk/edit) | Notificaciones |
| [EP10_Soporte](https://docs.google.com/spreadsheets/d/1BOWniK6PrLZS_xmdUS0l3lqky4F5U1r_WVXqrUAQzIw/edit) | Conversaciones, Mensajes |

EP04 contiene seis eventos de ejemplo, en cinco categorías, con estados PUBLICADO, CERRADO y CANCELADO. EP06 contiene dos registros sintéticos `DEMO` para mostrar disponibilidad agotada; no representan personas reales.

## Reglas y estados

- Usuarios: `ACTIVO` / `INACTIVO`; el borrado es lógico.
- Contraseñas: PBKDF2-SHA256, salt aleatorio individual de 16 bytes y 120.000 iteraciones.
- Sesiones: token aleatorio con expiración de 24 horas; no se acepta una sesión cerrada, expirada o asociada a un usuario inactivo.
- Telegram: códigos `PENDIENTE`, `USADO`, `EXPIRADO` o `CANCELADO`; vinculaciones `ACTIVA` o `INACTIVA`.
- Eventos: `BORRADOR`, `PUBLICADO`, `CERRADO` o `CANCELADO`.
- Inscripciones: `CONFIRMADA`, `LISTA_ESPERA` o `CANCELADA`.
- Notificaciones: cada canal registra `PENDIENTE`, `ENVIADO`, `ERROR` o `NO_APLICA`.
- Conversaciones: `ABIERTA` o `CERRADA`; los mensajes se identifican como `USER`, `ASSISTANT` o `SYSTEM`.
- Idempotencia: evita inscripciones activas duplicadas por usuario-evento, recordatorios repetidos por usuario-evento-tipo y reenvíos de notificaciones ya entregadas.
- Disponibilidad: se calcula a partir de la capacidad y las inscripciones `CONFIRMADA`; no se mantiene un contador manual.

## Asistente informativo

WF10 utiliza el chat alojado de n8n (no una aplicación frontend). Su modelo es GPT-5 mini mediante créditos Gateway de n8n. Tiene memoria por sesión y registra cada turno en EP10_Soporte. Puede responder sobre eventos publicados y el funcionamiento de EventPass. No tiene herramientas para crear usuarios, iniciar sesiones, inscribir o cancelar asistentes, ni cambiar eventos, cupos o estados.

## Archivos de workflows

Los JSON se encuentran en `n8n/`:

- `WF01_usuarios_crud.json`
- `WF02_auth_sesiones.json`
- `WF03_vinculacion_telegram_codigo.json`
- `WF03_vinculacion_telegram_bot.json`
- `WF04_eventos_crud.json`
- `WF05_catalogo_publico.json`
- `WF06_inscripciones_crud.json`
- `WF07_reasignacion_lista_espera.json`
- `WF08_recordatorios.json`
- `WF09_notificaciones.json`
- `WF10_eventpass_assistant.json`

## Publicación

Los workflows se encuentran como borradores inactivos para evitar exponer webhooks o iniciar notificaciones automáticas sin revisión. Antes de usar el sistema, revisa credenciales, permisos y configuración en n8n. Publica primero WF09 y luego los workflows que lo llaman; activa WF03b para recibir comandos Telegram y publica WF07/WF08 cuando quieras habilitar schedules.

Este repositorio no incluye frontend, código desplegable en Vercel ni variables locales, por lo que no requiere `.env.example`.
