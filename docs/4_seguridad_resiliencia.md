# Entregable 4 — Documentación de Seguridad y Resiliencia

**Criterio de la rúbrica:** minimización de datos + rutas de error (Error Handlers) + puntos de Human-in-the-loop, explicados.

---

## 1. Minimización de datos

El sistema aplica el principio de **recolectar y procesar solo lo mínimo necesario**:

- **Entrada acotada:** el webhook recibe 6 campos; solo 4 son obligatorios. No se piden datos personales del solicitante más allá del email corporativo.
- **A la IA solo lo necesario:** a Claude se le envía la descripción del sistema y los datos que maneja para clasificar riesgo — **no** credenciales, secretos, ni contenido sensible del activo.
- **Sin datos hardcodeados:** ninguna credencial, key, canal o ID va escrito en los nodos. Todo se resuelve por **credenciales de n8n** (Airtable PAT, Slack Bot Token, Anthropic key vía Header Auth) y por **variables del sistema** (`$json`, `$('Nodo')`, `$execution`, `$now`).
- **Secretos fuera del flujo:** las API keys viven en el gestor de credenciales de n8n, nunca en el JSON exportado ni en logs. En el video demo se ocultan.
- **Retención mínima:** Airtable guarda solo el resultado de la evaluación y su trazabilidad (estado, timestamps, thread ID), no copias del activo evaluado.

## 2. Rutas de error (Error Handlers / resiliencia)

El flujo es **capaz de guardar un registro de error** ante fallas, sin caerse. Dos rutas explícitas:

### Ruta A — Datos faltantes (Directiva tipo *Break*)
- **Disparador:** nodo `Validación - Datos Completos` (IF) rama FALSE.
- **Acción:** `Airtable - Log Error Datos` crea un registro con `Estado = Error - Datos incompletos` + `Error Log` describiendo qué faltó → `Slack - Alerta Datos Faltantes`.
- **Efecto:** el flujo **corta** antes de gastar una llamada a la IA. La solicitud incompleta queda auditada.

### Ruta B — Falla de la API de IA (Directiva tipo *Resume*)
- **Disparador:** nodo `Claude - Evaluación de Riesgo` con `onError: continueErrorOutput` (rama de error).
- **Acción:** `Airtable - Log Error API` guarda `Estado = Error - API` + el mensaje de error real → `Slack - Alerta Falla API`.
- **Efecto:** si Anthropic falla o hay timeout, el sistema **no se rompe**: registra el fallo, avisa, y la ejecución termina de forma controlada. Queda visible en el Dashboard como parte de la **tasa de errores**.

### Protección anti-bucle infinito
- Trigger por **webhook** → una ejecución por solicitud (no hay polling que se auto-dispare).
- El `Estado` avanza en **una sola dirección** (`Pendiente → Esperando Aprobación → Aprobado/Rechazado`); un registro nunca vuelve a un estado anterior ni se reprocesa.
- Los filtros comparan **tipos correctos** (string `notEmpty` en validación; string `equals` en el switch de decisión).

## 3. Puntos de Human-in-the-loop (HITL)

Para evitar el "efecto metralleta" (que el sistema ejecute acciones críticas sin control), el flujo **se detiene antes de cerrar una homologación**:

| Elemento | Cómo funciona |
|----------|---------------|
| **Punto de detención** | Nodo `Slack - Solicitar Aprobación (HITL)` en operación **"Send and Wait for Response"** (`sendAndWait` / `responseType: approval`). El flujo queda en pausa indefinida hasta recibir respuesta. |
| **Notificación** | El mismo nodo publica el resumen de la evaluación (sistema, riesgo, CIA, controles) con **dos botones interactivos nativos de Slack**: Aprobar / Rechazar. |
| **Espera de feedback** | El sistema **no marca nada como aprobado por sí solo**. Solo el clic humano en un botón reanuda la ejecución. Se usaron botones nativos (no links de texto) precisamente porque Slack escanea los links y podría auto-dispararlos: el botón exige una acción humana real. |
| **Ruteo posterior** | `Switch - Decisión Humana` lee el booleano `{{ $json.data.approved }}` (`true` → Aprobado por Humano, `false` → Rechazado), marca el registro + notifica el cierre en el **mismo hilo** (Thread ID). |

**Por qué es crítico este punto:** homologar un sistema es autorizar su uso en la entidad financiera (regulada por el ente de contralor bancario). La IA **asiste** en la evaluación de riesgo, pero la **decisión de homologar la toma siempre una persona**. El HITL garantiza trazabilidad y responsabilidad humana sobre la acción crítica.

## 4. Resumen de controles

| Control | Implementación |
|---------|----------------|
| Minimización de datos | Campos mínimos, sin secretos a la IA, credenciales fuera del flujo |
| Manejo de errores | 2 rutas de error con log en Airtable + alerta Slack |
| Anti-bucle | Webhook + máquina de estados unidireccional |
| Validación de tipos | IF `notEmpty` / Switch `equals` |
| HITL | Slack sendAndWait (botones nativos) antes de cerrar |
| Auditabilidad | Estado, timestamps, Thread ID y Error Log por cada solicitud |
