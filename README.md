# Entrega Final — Ecosistema de Automatización IA Autónomo para Negocios

**Curso:** IA Automation (Coderhouse) · **Alumna:** Sofía Osés
**Caso de uso:** Triage automático de solicitudes de homologación de seguridad IT (proceso PAI — banca financiera).

Sistema de automatización de punta a punta que recibe una solicitud de homologación, la evalúa con IA, la registra en una base de datos, exige aprobación humana antes de cerrar, y notifica el resultado — con rutas de error y sin intervención manual salvo el punto de control humano.

## Stack (4 categorías obligatorias)

| Categoría | Tecnología |
|-----------|-----------|
| Orquestador | **n8n** |
| Base de datos | **Airtable** (memoria y registro del sistema) |
| Procesamiento IA | **Claude Haiku 4.5** (`claude-haiku-4-5-20251001`, prompt estructurado, lógica de agente) |
| Canal de salida | **Slack** (notificaciones + Human-in-the-loop) |

## Los 5 entregables (uno por criterio de la rúbrica, 20% c/u)

| # | Entregable | Archivo |
|---|-----------|---------|
| 1 | **Diagrama de arquitectura** (PDF) | [`docs/diagrama_arquitectura.pdf`](docs/diagrama_arquitectura.pdf) |
| 2 | **Manual operativo de datos** (tablas + esquemas JSON) | [`docs/2_manual_operativo_datos.md`](docs/2_manual_operativo_datos.md) |
| 3 | **Matriz de costos** (modelo por tarea + ahorro) | [`docs/3_matriz_costos.md`](docs/3_matriz_costos.md) |
| 4 | **Seguridad y resiliencia** (minimización + errores + HITL) | [`docs/4_seguridad_resiliencia.md`](docs/4_seguridad_resiliencia.md) |
| 5 | **Dashboard de control** (Shared View con KPIs) | [`docs/5_dashboard_control.md`](docs/5_dashboard_control.md) |

## Archivos técnicos de respaldo

| Archivo | Descripción |
|---------|-------------|
| [`flujo/flujo_pai_homologacion.json`](flujo/flujo_pai_homologacion.json) | JSON del flujo n8n, listo para importar. |
| [`flujo/payloads_prueba.md`](flujo/payloads_prueba.md) | 5 payloads para el test de estrés (camino feliz + infeliz). |
| [`docs/diseno_arquitectura.md`](docs/diseno_arquitectura.md) | Diseño/blueprint del sistema. |
| `evidencias/` | Screenshots del flujo, Airtable y Slack. |

## Enlaces obligatorios

- 🔗 **Base de datos (modo lectura):** https://airtable.com/app6FuI5UeGkgykO7/shrGLvz82yL1ROM9O
- 📊 **Dashboard de control (KPIs + tasa de errores):** https://airtable.com/app6FuI5UeGkgykO7/shrGLvz82yL1ROM9O
- 🎥 **Video demo (3 min):** https://drive.google.com/drive/folders/1l6L_I9DzpAE3erGGd6Y7LxSwOiL5eBBm?usp=sharing

## Cómo reproducirlo

1. **Airtable:** crear la base `PAI Homologaciones` con las 3 tablas del [manual de datos](docs/2_manual_operativo_datos.md).
2. **n8n:** importar `flujo/flujo_pai_homologacion.json`.
3. **Credenciales:** configurar Airtable (PAT), Slack (Bot Token) y Claude (Header Auth `x-api-key`). Bindear base/tabla en los nodos Airtable y el canal en los nodos Slack.
4. **Activar** el flujo y copiar la URL del webhook.
5. **Probar** con los payloads de `flujo/payloads_prueba.md`.

## Verificaciones de seguridad (check previo)

- ✅ Filtro anti-bucle infinito (webhook + máquina de estados unidireccional).
- ✅ Comparación de tipos correctos en filtros (string `notEmpty` / `equals`).
- ✅ Prompt de IA dinámico con variables del sistema (sin hardcodear).
- ✅ Sin API keys ni datos sensibles en el JSON ni en el repo.
