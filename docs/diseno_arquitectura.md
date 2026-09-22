# Diseño — Ecosistema de Automatización IA (Entrega Final)

**Caso de uso:** Triage automático de solicitudes de homologación de seguridad IT (proceso PAI, banca financiera).
**Stack:** n8n (orquestador) · Airtable (memoria/DB) · Claude (motor IA) · Slack (salida + HITL).

---

## 1. Qué resuelve el flujo (de punta a punta)

1. Llega una **solicitud de homologación** de un sistema/activo (proveedor o área interna).
2. El flujo se dispara, **valida** que los datos estén completos.
3. **Claude evalúa el riesgo**: clasifica Confidencialidad/Integridad/Disponibilidad (CIA), asigna un nivel de riesgo y lista los controles requeridos.
4. Guarda la evaluación en Airtable (**memoria del sistema**).
5. **Human-in-the-loop:** notifica al analista PAI por Slack y **espera su aprobación** antes de cerrar.
6. Según la decisión humana → marca el registro y notifica el cierre (o el rechazo) en el mismo hilo de Slack.
7. Si algo falla (datos faltantes o caída de la API), toma una **ruta de error** que registra el fallo y avisa, sin frenar el sistema.

---

## 2. Mapa de nodos del flujo n8n

| # | Nodo (nombre en n8n) | Tipo | Qué hace |
|---|----------------------|------|----------|
| 1 | `Trigger - Recepción Solicitud` | Webhook | Recibe la solicitud (JSON). Disparador específico → evita consumo innecesario. |
| 2 | `Validación - Datos Completos` | IF | Verifica campos obligatorios (nombre, tipo, solicitante, datos que maneja). |
| 3 | `Airtable - Crear Registro (Pendiente)` | Airtable Create | Persiste la solicitud como memoria. Estado = `Pendiente`. |
| 4 | `Claude - Evaluación de Riesgo` | Anthropic (Haiku 4.5) | Prompt estructurado + `max_tokens` limitado. Devuelve JSON: CIA, nivel de riesgo, controles, resumen. |
| 5 | `Airtable - Guardar Evaluación IA` | Airtable Update | Estado = `Esperando Aprobación`. Guarda CIA, riesgo, resumen, modelo y tokens. |
| 6 | `Slack - Solicitar Aprobación (HITL)` | Slack **Send and Wait** | **Punto de detención HITL.** Envía el resumen al analista con botones nativos **Aprobar / Rechazar** y deja la ejecución en pausa hasta el clic humano. Guarda el **Thread ID**. |
| 7 | `Switch - Decisión Humana` | Switch | Lee el booleano `data.approved` y rutea a Aprobado (`true`) / Rechazado (`false`). |
| 8a | `Airtable - Marcar Aprobado` → `Slack - Notificar Cierre` | Airtable + Slack | Estado = `Aprobado por Humano`. Notifica en el hilo (Thread ID). |
| 8b | `Airtable - Marcar Rechazado` → `Slack - Notificar Rechazo` | Airtable + Slack | Estado = `Rechazado`. Notifica en el hilo. |

### Rutas de error (resiliencia — obligatorio)

| Origen | Condición | Ruta |
|--------|-----------|------|
| Nodo 2 (Validación) rama FALSE | Faltan datos | `Airtable - Log Error Datos` (Estado = `Error - Datos incompletos`) → `Slack - Alerta Datos Faltantes` → fin. |
| Nodo 4 (Claude) `continueOnFail` / Error branch | Falla o timeout de la API de IA | `Airtable - Log Error API` (Estado = `Error - API`, guarda mensaje) → `Slack - Alerta Falla API` → fin. |

### Anti-bucle infinito
El trigger procesa una solicitud por ejecución (webhook). El estado avanza en una sola dirección (`Pendiente → Esperando Aprobación → Aprobado/Rechazado`), por lo que un registro nunca se reprocesa.

---

## 3. Esquema de Airtable — Base `PAI Homologaciones`

### Tabla 1 · `Solicitudes` (memoria principal)

| Campo | Tipo | Notas |
|-------|------|-------|
| ID Solicitud | Autonumber | Clave visible |
| Fecha Solicitud | Created time | Automático |
| Nombre del Sistema | Single line | Obligatorio |
| Solicitante | Email | Obligatorio |
| Área Solicitante | Single line | |
| Tipo de Plataforma | Single select | SaaS · Microservicio · PH · API · BOT |
| Datos que Maneja | Long text | Obligatorio (input para la IA) |
| Criticidad Declarada | Single select | Alta · Media · Baja |
| **Estado** | Single select | `Pendiente` · `Procesado por IA` · `Esperando Aprobación` · `Aprobado por Humano` · `Rechazado` · `Error - Datos incompletos` · `Error - API` |
| Clasif. IA - Confidencialidad | Number (1–5) | Lo completa Claude |
| Clasif. IA - Integridad | Number (1–5) | Lo completa Claude |
| Clasif. IA - Disponibilidad | Number (1–5) | Lo completa Claude |
| Nivel de Riesgo IA | Single line text | Critico · Alto · Medio · Bajo |
| Controles Requeridos | Link → `Controles` | Relación (evita datos aislados) |
| Resumen IA | Long text | Justificación de la evaluación |
| Analista Asignado | Link → `Analistas` | Relación |
| Slack Thread ID | Single line | Mapea el hilo del HITL |
| Modelo IA / Tokens | Single line | Para KPIs de costo |
| Fecha Aprobación | Date | |
| Comentario Humano | Long text | Feedback del analista |
| Error Log | Long text | Solo en rutas de error |

### Tabla 2 · `Analistas` (relación)
`Nombre` · `Email` · `Slack User ID` · `Solicitudes` (link inverso)

### Tabla 3 · `Controles de Seguridad` (catálogo, relación)
`Control` · `Categoría` (C/I/A) · `Descripción` · `Solicitudes` (link inverso)

> Las **relaciones** entre las 3 tablas cumplen el requisito de "evitar datos aislados".

### Vista para el Dashboard de control
Vista **`Dashboard KPIs`** (compartida en modo lectura pública), agrupada por `Estado`, con conteos por `Nivel de Riesgo IA` y filtro que aísla los `Error - *` para leer la **tasa de errores**.

---

## 4. Cómo cada pieza cubre los 5 criterios de la rúbrica (20% c/u)

| Criterio | Dónde se cubre |
|----------|----------------|
| Mapa de arquitectura | Sección 2 → se convierte en el diagrama PDF |
| Estructuras de datos documentadas | Sección 3 + esquemas JSON de transferencia (los genero en el paso siguiente) |
| Optimización de costos | Matriz de decisión de modelo por tarea (entregable aparte) |
| Seguridad y resiliencia | Rutas de error (sección 2) + HITL (nodos 6–7) + minimización de datos |
| Dashboard de control | Vista compartida `Dashboard KPIs` (sección 3) |

---

**Próximo paso:** con esto aprobado, genero el `.json` del flujo n8n listo para importar y los esquemas JSON de transferencia.
