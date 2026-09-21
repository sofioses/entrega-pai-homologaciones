# Entregable 3 — Cuadro Comparativo / Matriz de Costos

**Criterio de la rúbrica:** justificar qué modelo se usa por tarea y estimar el ahorro. Es un entregable en sí mismo, no solo "limitar tokens".

---

## 1. Principio de diseño

Cada tarea del pipeline tiene un perfil distinto (volumen de texto, complejidad de razonamiento, tolerancia a latencia). **No todas necesitan el modelo más caro.** La estrategia es asignar el modelo mínimo suficiente por tarea.

## 2. Matriz de decisión: modelo por tarea

| Tarea del flujo | Complejidad | Modelo elegido | Por qué | max_tokens |
|-----------------|-------------|----------------|---------|-----------|
| **Evaluación de riesgo CIA** (nodo Claude) | Media | **Claude Haiku 4.5** (`claude-haiku-4-5-20251001`) | Razonamiento estructurado sobre texto corto; devuelve JSON acotado. Haiku alcanza y es ~15× más barato que un modelo premium. | 700 |
| Validación de datos | Nula (lógica) | **Ninguno** (nodo IF) | Es una comparación de campos vacíos. Gastar un LLM acá sería desperdicio. | — |
| Parseo de la respuesta | Nula (código) | **Ninguno** (nodo Code) | `JSON.parse` en JS nativo. | — |
| Redacción de notificación Slack | Baja | **Ninguno** (template) | Texto fijo con variables. No requiere IA. | — |
| *(Escenario futuro)* Lectura densa de un pliego/contrato largo | Alta | **Claude Sonnet** | Documentos largos y densos justifican un modelo de lectura más potente. | 1500 |
| *(Escenario futuro)* Reproceso masivo nocturno de N solicitudes | Media, volumen alto | **Message Batches API (Haiku)** | Procesos masivos sin urgencia → API de lotes con ~50% de descuento. | 700 |

## 3. Comparación de modelos (referencia de precios públicos, USD por millón de tokens)

| Modelo | Input | Output | Uso recomendado |
|--------|-------|--------|-----------------|
| Claude Haiku 4.5 | ~$1.00 | ~$5.00 | Tareas simples/estructuradas, alto volumen. ← **usado acá** |
| Claude Sonnet | ~$3.00 | ~$15.00 | Lectura densa, razonamiento complejo. |
| Claude Opus | ~$15.00 | ~$75.00 | Solo casos que exigen máxima capacidad. |
| GPT-4o-mini | ~$0.15 | ~$0.60 | Alternativa económica para tareas triviales. |

> Los precios son aproximados y de referencia; el criterio de asignación es lo que se evalúa, no el número exacto.

## 4. Estimación de costo del flujo (por solicitud)

Con Haiku 4.5, `max_tokens=700`, y un consumo típico medido de ~210 tokens de input + ~180 de output:

| Concepto | Cálculo | Costo |
|----------|---------|-------|
| Input | 210 tok × $1.00/M | ~$0.00021 |
| Output | 180 tok × $5.00/M | ~$0.00090 |
| **Total por solicitud** | | **~$0.0011 USD** |

**Por cada 1.000 solicitudes ≈ US$1,11.**

## 5. Ahorro estimado vs. alternativas

| Estrategia | Costo x 1.000 solicitudes | Ahorro vs. Opus |
|------------|---------------------------|-----------------|
| Todo con Opus | ~$16,65 | — |
| Todo con Sonnet | ~$3,33 | 80% |
| **Haiku 4.5 (elegido)** | **~$1,11** | **93%** |
| Haiku 4.5 + Batches (reproceso masivo) | ~$0,56 | 97% |

**Palancas de ahorro aplicadas en el flujo:**
1. **Modelo por tarea** → Haiku en vez de un premium (93% de ahorro).
2. **`max_tokens` acotado a 700** → evita respuestas infladas.
3. **IF/Code en vez de IA** para validación y parseo → 0 costo en esos pasos.
4. **Trigger específico (webhook)** → una llamada IA por solicitud real, sin polling masivo.
5. **Batches** documentado para el escenario de reproceso masivo (−50% adicional).

## 6. Conclusión

El sistema usa **Claude Haiku 4.5 solo donde hace falta razonar**, resuelve el resto con lógica/código sin costo de IA, y deja documentada la ruta a Sonnet (lectura densa) y a Batches (volumen). Resultado: **~US$0,0011 por solicitud** y hasta **93% de ahorro** frente a usar un modelo premium para todo.
