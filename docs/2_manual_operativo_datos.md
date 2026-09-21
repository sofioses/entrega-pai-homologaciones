# Entregable 2 — Manual Operativo de Datos

Sistema: **Triage automático de homologaciones PAI** · Stack: n8n · Airtable · Claude · Slack

Este documento cubre (a) el esquema de tablas vinculadas de Airtable y (b) los esquemas JSON de transferencia entre cada integración, explicados.

---

## Parte A · Esquema de tablas vinculadas (Airtable)

La base `PAI Homologaciones` tiene **3 tablas relacionadas** para evitar datos aislados.

### Tabla 1 · `Solicitudes` (memoria principal del sistema)

| Campo | Tipo | Origen | Descripción |
|-------|------|--------|-------------|
| ID Solicitud | Autonumber | Airtable | Clave visible. |
| Fecha Solicitud | Created time | Airtable | Timestamp automático. |
| Nombre del Sistema | Single line text | Webhook | Sistema/activo a homologar. |
| Solicitante | Email | Webhook | Quien pide la homologación. |
| Área Solicitante | Single line text | Webhook | Área de negocio. |
| Tipo de Plataforma | Single select | Webhook | SaaS · Microservicio · PH · API · BOT. |
| Datos que Maneja | Long text | Webhook | Input principal para la IA. |
| Criticidad Declarada | Single select | Webhook | Alta · Media · Baja. |
| **Estado** | Single select | Sistema | Campo de estado del ciclo de vida (ver abajo). |
| Clasif. IA - Confidencialidad | Number (1–5) | Claude | Puntaje C. |
| Clasif. IA - Integridad | Number (1–5) | Claude | Puntaje I. |
| Clasif. IA - Disponibilidad | Number (1–5) | Claude | Puntaje D. |
| Nivel de Riesgo IA | Single select | Claude | Crítico · Alto · Medio · Bajo. |
| Controles Requeridos | Link → `Controles` | Claude/Analista | **Relación** al catálogo de controles. |
| Resumen IA | Long text | Claude | Justificación de la evaluación. |
| Analista Asignado | Link → `Analistas` | Sistema | **Relación** al analista PAI. |
| Slack Thread ID | Single line text | Slack | `ts` del mensaje → mapea el hilo del HITL. |
| Modelo IA / Tokens | Single line text | Sistema | Modelo usado (para KPIs de costo). |
| Fecha Aprobación | Date | Sistema | Fecha de la decisión humana. |
| Comentario Humano | Long text | Analista | Feedback de la aprobación/rechazo. |
| Error Log | Long text | Sistema | Solo en rutas de error. |

**Campo de estado (`Estado`) — máquina de estados:**

```
Pendiente ──► Esperando Aprobación ──► Aprobado por Humano
                                   └──► Rechazado
Pendiente ──► Error - Datos incompletos        (ruta de error 1)
Pendiente ──► Error - API                       (ruta de error 2)
```

El estado avanza en **una sola dirección** → un registro nunca se reprocesa (anti-bucle infinito).

### Tabla 2 · `Analistas` (relación)

| Campo | Tipo | Descripción |
|-------|------|-------------|
| Nombre | Single line text | Analista PAI. |
| Email | Email | Contacto. |
| Slack User ID | Single line text | Para mención/ruteo. |
| Solicitudes | Link → `Solicitudes` | Relación inversa (todas sus solicitudes). |

### Tabla 3 · `Controles de Seguridad` (catálogo, relación)

| Campo | Tipo | Descripción |
|-------|------|-------------|
| Control | Single line text | Ej. "MFA obligatorio", "Cifrado en reposo". |
| Categoría | Single select | C · I · D. |
| Descripción | Long text | Detalle del control. |
| Solicitudes | Link → `Solicitudes` | Relación inversa. |

**Diagrama de relaciones:**

```
   Analistas 1 ───< Solicitudes >─── N Controles de Seguridad
                        │
                    (memoria del
                     sistema)
```

> Las relaciones `Solicitudes → Analistas` y `Solicitudes → Controles` cumplen el requisito de "relaciones entre tablas para evitar datos aislados".

---

## Parte B · Esquemas JSON de transferencia (integraciones)

### B.1 · Entrada — Webhook (Trigger)

Lo que recibe el flujo cuando llega una solicitud. Se envía por `POST` al webhook.

```json
{
  "nombre_sistema": "Portal Proveedores Cloud",
  "solicitante": "jperez@comafi.com.ar",
  "area": "Compras",
  "tipo_plataforma": "SaaS",
  "datos_maneja": "Datos de proveedores, CUIT, datos de contacto y montos de facturación.",
  "criticidad": "Media"
}
```

| Campo | Obligatorio | Usado por |
|-------|-------------|-----------|
| `nombre_sistema` | ✅ (validado) | Airtable, Claude, Slack |
| `solicitante` | ✅ (validado) | Airtable, Slack |
| `tipo_plataforma` | ✅ (validado) | Airtable, Claude |
| `datos_maneja` | ✅ (validado) | Claude (input de riesgo) |
| `area` | ⬜ | Airtable |
| `criticidad` | ⬜ | Airtable |

> Si falta alguno de los 4 obligatorios, el nodo `Validación - Datos Completos` desvía a la **ruta de error de datos**.

### B.2 · Transferencia — n8n → Claude (API Messages)

Body que se arma dinámicamente (nodo `Preparar Prompt IA` + `Claude - Evaluación de Riesgo`). **Sin datos hardcodeados**: `system` y `content` salen de variables.

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 700,
  "system": "Sos un analista senior de seguridad... Responde EXCLUSIVAMENTE con un objeto JSON válido con esta forma exacta: {confidencialidad, integridad, disponibilidad, nivel_riesgo, controles, resumen}",
  "messages": [
    { "role": "user", "content": "Solicitud de homologación a evaluar:\n{...datos del webhook...}" }
  ]
}
```

Headers: `x-api-key` (credencial), `anthropic-version: 2023-06-01`, `content-type: application/json`.
`max_tokens` limitado a **700** para optimizar costo (ver entregable 3).

### B.3 · Respuesta — Claude → n8n

Claude devuelve el mensaje; el contenido útil está en `content[0].text` como JSON string.

```json
{
  "id": "msg_01...",
  "model": "claude-haiku-4-5-20251001",
  "content": [
    { "type": "text", "text": "{\"confidencialidad\":4,\"integridad\":3,\"disponibilidad\":3,\"nivel_riesgo\":\"Alto\",\"controles\":[\"MFA obligatorio\",\"Cifrado en reposo y tránsito\",\"Revisión de accesos trimestral\"],\"resumen\":\"El sistema maneja datos de proveedores y CUIT...\"}" }
  ],
  "usage": { "input_tokens": 210, "output_tokens": 180 }
}
```

El nodo `Parse - Evaluación IA` **mapea `content[0].text`** (variable de respuesta), limpia posibles ``` ```json ``` y hace `JSON.parse`. Si no es JSON válido → error controlado.

### B.4 · Transferencia — n8n → Airtable (Update evaluación)

```json
{
  "id": "recXXXXXXXX",
  "Estado": "Esperando Aprobación",
  "Clasif. IA - Confidencialidad": 4,
  "Clasif. IA - Integridad": 3,
  "Clasif. IA - Disponibilidad": 3,
  "Nivel de Riesgo IA": "Alto",
  "Resumen IA": "El sistema maneja datos de proveedores...\n\nControles requeridos: MFA obligatorio; Cifrado...",
  "Modelo IA / Tokens": "claude-haiku-4-5-20251001"
}
```

### B.5 · Transferencia — n8n → Slack (HITL) y respuesta

El HITL usa la operación nativa **"Send and Wait for Response"** del nodo Slack (`operation: sendAndWait`, `responseType: approval`). Slack publica un mensaje con la evaluación y **dos botones interactivos nativos** (Aprobar / Rechazar); la ejecución de n8n **queda en pausa** hasta que un humano hace clic. No se usan links de texto (que Slack escanea y auto-dispararía).

Mensaje que arma el nodo (los datos salen de variables, sin hardcodear):

```json
{
  "channel": "C0C3RJTTCJC",
  "message": ":mag: *Nueva evaluación de homologación*\n*Sistema:* {{ nombre }}\n*Nivel de Riesgo (IA):* Alto\n*CIA:* C4 / I3 / D3\n*Controles:* MFA obligatorio; Cifrado en reposo...",
  "approvalOptions": {
    "approvalType": "double",
    "buttonApprovalLabel": "Aprobar",
    "buttonDisapprovalLabel": "Rechazar"
  }
}
```

Cuando el analista hace clic, n8n reanuda y entrega la decisión como un booleano:

```json
{ "data": { "approved": true } }
```

El `Switch - Decisión Humana` lee `{{ $json.data.approved }}` y rutea a **Aprobado por Humano** (`true`) o **Rechazado** (`false`). El `ts` del mensaje se guarda en `Slack Thread ID` para trazabilidad del hilo.

### B.6 · Rutas de error (payloads que se registran)

**Datos incompletos:**
```json
{ "Estado": "Error - Datos incompletos", "Error Log": "Faltan campos obligatorios (nombre_sistema, solicitante, tipo_plataforma o datos_maneja)." }
```

**Falla de API IA:**
```json
{ "Estado": "Error - API", "Error Log": "{{ mensaje de error de la ejecución }}" }
```

---

## Resumen del recorrido del dato

```
Webhook (B.1) → Validación → Airtable crea (Pendiente)
  → Claude (B.2/B.3) → Parse → Airtable update (B.4, Esperando Aprobación)
  → Slack HITL sendAndWait (B.5) → pausa → decisión humana (clic en botón)
  → Airtable update (Aprobado/Rechazado) → Slack cierre en hilo
Errores → Airtable log (B.6) + Slack alerta
```
