# Entregable 5 — Dashboard de Control

**Criterio de la rúbrica:** un enlace público a una vista (Airtable Shared View) para monitorear KPIs y tasa de errores. Es **más que el link a la DB**: es un panel de control.

---

## 1. Qué mide el dashboard

El panel se arma sobre la tabla `Solicitudes` y muestra la salud operativa del sistema en vivo.

### KPIs

| KPI | Cómo se calcula | Para qué |
|-----|-----------------|----------|
| **Solicitudes totales** | Conteo de registros | Volumen procesado |
| **En espera de aprobación** | Registros con `Estado = Esperando Aprobación` | Cola pendiente de decisión humana (HITL) |
| **Aprobadas** | `Estado = Aprobado por Humano` | Homologaciones cerradas OK |
| **Rechazadas** | `Estado = Rechazado` | Decisiones negativas |
| **Distribución de riesgo** | Agrupar por `Nivel de Riesgo IA` | Cuántas Critico/Alto/Medio/Bajo |
| **Tasa de errores** | (Registros `Error - *`) / totales | Salud técnica del pipeline |
| **Tiempo a decisión** | `Fecha Aprobación` − `Fecha Solicitud` | Cuánto tarda el HITL |

## 2. Cómo se construye en Airtable

1. En la tabla `Solicitudes`, crear una vista tipo **Grid** llamada `Dashboard KPIs`.
2. **Agrupar** (Group) por el campo `Estado` → cada grupo muestra su conteo automáticamente.
3. Agregar un **sub-agrupamiento** por `Nivel de Riesgo IA` para ver la distribución.
4. (Opcional, plan con Interfaces) crear una **Interface** con elementos *Number* para cada KPI y un gráfico de torta por Estado.
5. **Compartir la vista** → `Share view` → activar **"Create a shareable grid view link"** en **modo solo lectura**. Ese es el enlace público del dashboard.

### Tasa de errores — vista dedicada
Crear un filtro `Estado is any of [Error - Datos incompletos, Error - API]`. El conteo de esa vista, sobre el total, es la **tasa de errores** del sistema.

## 3. Enlace público

> **https://airtable.com/app6FuI5UeGkgykO7/shrGLvz82yL1ROM9O**
>
> Vista `Dashboard KPIs` — grilla de solo lectura, agrupada por `Estado` y sub-agrupada por `Nivel de Riesgo IA`. Cada grupo muestra su conteo (los KPIs); la tasa de errores surge de los grupos `Error - *` sobre el total.

Este enlace va también en el README del repo como "Dashboard de Control" y como "link a la DB en modo lectura".

## 4. Checklist de entrega del dashboard

- [ ] Vista `Dashboard KPIs` agrupada por Estado.
- [ ] Vista/filtro de tasa de errores.
- [ ] Link compartido en **modo solo lectura**.
- [ ] Link pegado en este documento y en el README.
- [ ] Screenshot del dashboard en `/evidencias`.
