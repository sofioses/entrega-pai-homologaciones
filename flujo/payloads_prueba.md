# Payloads de prueba — Test de Estrés (5 corridas)

Enviar por `POST` a la URL del webhook del flujo, con `Content-Type: application/json`.
Incluye el **camino feliz** y el **camino infeliz** (datos incompletos) para verificar filtros y rutas de error.

---

### Corrida 1 — Riesgo Alto (camino feliz) → aprobar
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

### Corrida 2 — Riesgo Crítico (camino feliz) → aprobar
```json
{
  "nombre_sistema": "API Core Bancario - Consulta Saldos",
  "solicitante": "mlopez@comafi.com.ar",
  "area": "Canales Digitales",
  "tipo_plataforma": "API",
  "datos_maneja": "Saldos de cuentas, movimientos, datos de clientes y tarjetas.",
  "criticidad": "Alta"
}
```

### Corrida 3 — Riesgo Bajo (camino feliz) → rechazar (probar rama reject)
```json
{
  "nombre_sistema": "Chatbot FAQ Interno",
  "solicitante": "rgomez@comafi.com.ar",
  "area": "RRHH",
  "tipo_plataforma": "BOT",
  "datos_maneja": "Preguntas frecuentes de empleados sobre políticas internas. Sin datos sensibles.",
  "criticidad": "Baja"
}
```

### Corrida 4 — CAMINO INFELIZ: faltan datos → ruta de error de datos
```json
{
  "nombre_sistema": "Sistema sin descripción",
  "area": "Riesgos"
}
```
> Faltan `solicitante`, `tipo_plataforma` y `datos_maneja` → debe ir a `Error - Datos incompletos`.

### Corrida 5 — CAMINO INFELIZ: microservicio válido (camino feliz de control)
```json
{
  "nombre_sistema": "Microservicio Notificaciones Push",
  "solicitante": "adiaz@comafi.com.ar",
  "area": "Arquitectura",
  "tipo_plataforma": "Microservicio",
  "datos_maneja": "Tokens de dispositivos y mensajes de notificación a clientes.",
  "criticidad": "Media"
}
```

---

## Cómo probar la ruta de falla de API (opcional, 6ta corrida)
Para forzar `Error - API`: en la credencial del nodo `Claude - Evaluación de Riesgo`, poné temporalmente una API key inválida y enviá la Corrida 1. El flujo debe registrar `Error - API` en Airtable y alertar en Slack, sin romperse. Después restaurá la key válida.

## Checklist del test de estrés
- [ ] Corrida 1 → Aprobada (Estado: Aprobado por Humano)
- [ ] Corrida 2 → Aprobada
- [ ] Corrida 3 → Rechazada (Estado: Rechazado)
- [ ] Corrida 4 → Error - Datos incompletos (no llega a la IA)
- [ ] Corrida 5 → Aprobada
- [ ] (Opcional) Corrida 6 → Error - API
- [ ] Screenshot de cada resultado en Airtable + Slack para `/evidencias`
