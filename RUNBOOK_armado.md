# Runbook de armado en vivo — PAI Homologaciones

Guía paso a paso para dejar el sistema funcionando. Seguí en orden; cada bloque termina con un ✅ checkpoint. Cuando te trabes en cualquier paso, avisame y lo destrabamos juntos.

Base Airtable ya creada: **PAI Homologaciones** → tabla **Solicitudes** con estos campos ya hechos: `Nombre del Sistema`, `Datos que Maneja`, `Solicitante`, `Estado` (6 estados), `Área Solicitante`, `Tipo de Plataforma`. Faltan los de abajo.

---

## PARTE 1 · Terminar la tabla `Solicitudes`

Para cada campo: botón **+** al final de las columnas → escribí el nombre → elegí el tipo → (si es selección, cargá las opciones) → **Crear**.

> Tip que funciona bien: al elegir el tipo, hacé clic en el selector de tipo y **escribí el nombre del tipo** en el buscador (ej. "Selección única"), después clic en el resultado. Es más confiable que buscarlo en la lista.

| Campo | Tipo | Opciones / nota |
|-------|------|-----------------|
| Criticidad Declarada | Selección única | Alta · Media · Baja |
| Clasif. IA - Confidencialidad | Número | Entero (0 decimales) |
| Clasif. IA - Integridad | Número | Entero |
| Clasif. IA - Disponibilidad | Número | Entero |
| Nivel de Riesgo IA | Selección única | Crítico · Alto · Medio · Bajo |
| Resumen IA | Texto largo | — |
| Modelo IA / Tokens | Texto de una sola línea | — |
| Slack Thread ID | Texto de una sola línea | — |
| Fecha Aprobación | Fecha | — |
| Comentario Humano | Texto largo | — |
| Error Log | Texto largo | — |

Después, **borrá los 2 campos `Attachments`** que venían por defecto: clic en el ▾ del encabezado del campo → *Eliminar campo*.

**✅ Checkpoint 1:** la tabla `Solicitudes` tiene todos los campos de arriba + los 6 ya hechos, y no quedan campos "Attachments".

---

## PARTE 2 · Crear las 2 tablas de relación

### Tabla `Analistas`
1. Arriba, al lado de la pestaña `Solicitudes`, clic en **+ Añadir o importar** → *Crear tabla en blanco* → nombre **Analistas**.
2. Campos: `Nombre` (primario, texto), `Email` (Email), `Slack User ID` (texto).
3. Cargá 1 fila de ejemplo con tu nombre/email para las pruebas.

### Tabla `Controles de Seguridad`
1. **+ Añadir o importar** → tabla en blanco → **Controles de Seguridad**.
2. Campos: `Control` (primario, texto), `Categoría` (Selección única: C · I · D), `Descripción` (texto largo).
3. Cargá 3–4 controles de ejemplo (ej. "MFA obligatorio" / C, "Cifrado en reposo" / C, "Backup diario" / D).

### Volver a `Solicitudes` y agregar las relaciones
Con las tablas creadas, en `Solicitudes` agregá 2 campos tipo **Vincular a otro registro**:
- `Analista Asignado` → vincular a **Analistas**.
- `Controles Requeridos` → vincular a **Controles de Seguridad**.

**✅ Checkpoint 2:** existen 3 tablas y `Solicitudes` tiene 2 campos de vínculo. (Esto cubre el requisito "relaciones entre tablas para evitar datos aislados").

---

## PARTE 3 · Token de Airtable (para n8n)

1. Airtable → foto de perfil (arriba der.) → **Developer hub** → **Personal access tokens** → *Create token*.
2. Scopes: `data.records:read`, `data.records:write`, `schema.bases:read`.
3. Access: agregá la base **PAI Homologaciones**.
4. Copiá el token (empieza con `pat...`). **Guardalo, lo pegás en n8n.**

**✅ Checkpoint 3:** tenés un PAT de Airtable con acceso a la base.

---

## PARTE 4 · Importar el flujo en n8n

1. n8n → **Workflows** → menú **⋮** → **Import from File** → elegí `flujo_pai_homologacion.json`.
2. Se abre el flujo con todos los nodos y las notas amarillas (sticky notes) de guía.

**✅ Checkpoint 4:** ves el flujo con el trigger a la izquierda y las 2 ramas de error abajo.

---

## PARTE 5 · Conectar credenciales (esto lo hacés vos)

> Estas 3 credenciales son tuyas y nunca las manejo yo.

### 5.1 Airtable
En cualquier nodo Airtable → *Credential* → **Create New** → pegá el **PAT** de la Parte 3.

### 5.2 Anthropic (Claude)
En el nodo `Claude - Evaluación de Riesgo` (HTTP Request) → *Credential* → **Create New → Header Auth**:
- **Name:** `x-api-key`
- **Value:** tu API key de Anthropic (empieza con `sk-ant-...`).

### 5.3 Slack
En cualquier nodo Slack → *Credential* → **Slack API** (o OAuth2) → pegá el **Bot Token** (`xoxb-...`).
Scopes mínimos del bot: `chat:write`. Invitá el bot a tu canal (ej. `/invite @tu-bot` en `#pai-homologaciones`).

**✅ Checkpoint 5:** las 3 credenciales quedan en verde al probarlas.

---

## PARTE 6 · Bindear base, tablas y canal

- En **cada nodo Airtable** (son 6): seleccioná Base = *PAI Homologaciones*, Table = *Solicitudes*. n8n carga las columnas; si alguna quedó con otro nombre, reasignala.
- En **cada nodo Slack** (son 4): elegí tu canal.
- Nodo `Claude`: confirmá `model: claude-haiku-4-5-20251001` y `max_tokens: 700`.

**✅ Checkpoint 6:** ningún nodo muestra ⚠️ de "campo/base sin seleccionar".

---

## PARTE 7 · Activar y obtener la URL del webhook

1. Abrí el nodo `Trigger - Recepción Solicitud` → copiá la **Production URL** (o Test URL para probar primero).
2. **Activá** el workflow (toggle arriba a la derecha).

**✅ Checkpoint 7:** tenés la URL del webhook.

---

## PARTE 8 · Test de estrés (5 corridas)

Usá los payloads de `payloads_prueba.md`. Enviá cada uno con Postman, Insomnia, o desde una terminal:

```bash
curl -X POST "URL_DEL_WEBHOOK" \
  -H "Content-Type: application/json" \
  -d '{"nombre_sistema":"Portal Proveedores Cloud","solicitante":"jperez@bancoejemplo.com.ar","area":"Compras","tipo_plataforma":"SaaS","datos_maneja":"Datos de proveedores, CUIT y montos.","criticidad":"Media"}'
```

Verificá que:
- Corridas 1, 2, 5 → registro creado, evaluación IA, mensaje en Slack, y al aprobar/rechazar cambia el `Estado`.
- Corrida 3 → probá el botón **Rechazar**.
- Corrida 4 (datos incompletos) → `Estado = Error - Datos incompletos`, sin llamar a la IA.
- (Opcional) key inválida → `Estado = Error - API`.

Sacá **screenshots** de Airtable (registros con estados) y Slack (mensajes con botones) para `/evidencias`.

**✅ Checkpoint 8:** las 5 corridas quedaron registradas con el estado correcto.

---

## PARTE 9 · Dashboard: Shared View pública

1. En `Solicitudes`, creá una vista Grid llamada **Dashboard KPIs**.
2. **Grupo** por `Estado` (y sub-grupo por `Nivel de Riesgo IA`).
3. Botón **Compartir y sincronizar** → *Create a shareable grid view link* → modo **solo lectura**.
4. Copiá el link (`https://airtable.com/app.../shr...`).

**✅ Checkpoint 9:** tenés el link público de solo lectura → va al README y al doc del Dashboard.

---

## PARTE 10 · Repo de GitHub

Subí la carpeta `entrega/` con esta estructura (ya te la dejo armada):

```
README.md
docs/  (PDF completo + diagrama + los 5 .md)
flujo/ (flujo_pai_homologacion.json + payloads_prueba.md)
evidencias/ (tus screenshots)
```

En el README, completá los 3 links: **DB (lectura)**, **Dashboard KPIs**, **Video**.

**✅ Checkpoint 10:** repo público con PDF, JSON y evidencias.

---

## PARTE 11 · Video demo (3 min) — guion

1. **0:00–0:30 — Contexto:** "Sistema de triage de homologaciones PAI: n8n + Airtable + Claude + Slack." Mostrá el diagrama del PDF.
2. **0:30–1:15 — Trigger + procesamiento:** enviá la Corrida 1 (curl/Postman). Mostrá el flujo ejecutándose nodo por nodo en n8n (validación → Airtable → Claude → Airtable).
3. **1:15–2:00 — HITL:** mostrá el mensaje en Slack con Aprobar/Rechazar; hacé clic en Aprobar; mostrá cómo el flujo reanuda y el `Estado` cambia en Airtable.
4. **2:00–2:40 — Camino infeliz:** enviá la Corrida 4 (datos incompletos) y mostrá el `Error - Datos incompletos` + alerta en Slack.
5. **2:40–3:00 — Dashboard:** mostrá la Shared View con los KPIs y la tasa de errores.

> ⚠️ Ocultá las API keys en pantalla (no abras las credenciales en el video).

**✅ Checkpoint 11:** video de ≤3 min mostrando trigger, procesamiento, HITL y resultado.

---

### Resumen de lo que va al repo (obligatorios de la rúbrica)
- [ ] Diagrama de arquitectura (PDF) ✔ ya generado
- [ ] Estructuras de datos + JSON (PDF) ✔ ya generado
- [ ] Matriz de costos ✔ ya generado
- [ ] Seguridad y resiliencia ✔ ya generado
- [ ] Dashboard de control → link público (Parte 9)
- [ ] JSON del flujo ✔ + evidencias (screenshots) + link DB lectura
