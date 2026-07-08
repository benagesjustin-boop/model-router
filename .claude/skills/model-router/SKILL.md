---
name: model-router
description: Selector de patrón de orquestación + ejecución con el modelo correcto por rol. v2 unifica 4 patrones — P0 inline (default, sin agentes), P1 brief+executors mecánicos, P2 war-game+executors que razonan (patrón Sprint), P3 research fan-out paralelo — y elige por tarea con gates de entrada. Usar cuando el usuario pida separar pensar caro de ejecutar barato, orquestar subagentes, ejecutar un war-game, o ahorrar tokens en una tarea multi-paso. Triggers "/model-router", "investigá a fondo y ejecutá barato", "research caro build barato", "qué patrón uso para esto", "ejecutá el war-game", "orquestá esto con subagentes". NO usar para tareas triviales de 1 paso — eso es P0 sin ceremonia: hacerlo directo.
allowed-tools:
  - Agent
  - Read
  - Write
  - Edit
  - Bash
  - Grep
  - Glob
  - AskUserQuestion
---

# Model Router v2 — Selector de patrón de orquestación

Cuatro patrones, un selector, un contrato de executor unificado. Base de evidencia: deep-dive propio (7 agents en paralelo + abogado del diablo, 2026-07) sobre docs de Anthropic y uso real en producción.

**Principio rector (Anthropic):** un agente usa ~4x los tokens de un chat; multi-agent ~15x. La orquestación es un gate de costo-beneficio, no un default. Descomponer por FRONTERAS DE CONTEXTO, no por fases del problema. Si dudás → P0.

## Paso 0 — Selector (obligatorio, output visible de 1 línea)

Evaluar en orden y declarar: `Patrón: PX — <motivo en una frase>`.

| Gate (en orden) | Si aplica → |
|---|---|
| ¿Tarea de 1 paso, <3 archivos, o el contexto ya está cargado en esta sesión? | **P0 INLINE** |
| ¿Lógica secuencial con dependencias entre pasos (código con lógica de negocio, refactor encadenado, plan→implement→test)? | **P2 WAR-GAME** (si hay war-game escrito) o **P0** (si es chico) |
| ¿N piezas independientes con spec 100% resuelta (variantes, archivos templados, outputs disjuntos)? | **P1 BRIEF+EXECUTORS** |
| ¿Research breadth-first: sub-preguntas independientes, cada una investigable sola, retorno = resumen corto? | **P3 FAN-OUT** |
| Ninguno claro | **P0** y decirlo |

**Anti-patrones que el selector debe atrapar:**
- Piezas "independientes" que comparten estado/contexto → NO es P1, es P2 o P0 (el handoff pierde contexto: "telephone game").
- Research que se resuelve en <2 búsquedas → P0, sin agentes.
- Lanzar un agente para tarea que el orquestador hace en 2 tool calls → el cold-start (re-leer CLAUDE.md/contexto) cuesta más que lo que ahorra.
- Sesión ya corre el modelo caro y la tarea es pensar/escribir un doc → inline SIEMPRE (no pagar el mismo modelo dos veces).

**Permiso:** invocar el skill autoriza el research/planning. Los Agent calls de ejecución (P1/P2/P3) requieren confirmación explícita del usuario ANTES del primer lanzamiento (regla de sesión: agentes se lanzan con permiso). Una confirmación cubre todo el batch declarado, no hace falta re-pedir por agente.

## Reglas de tokens — transversales a todo patrón

1. **Effort ≤ high, siempre.** xhigh/max dan el mismo output y queman el doble. Subagents mecánicos → effort low.
2. **Modelo explícito en cada Agent call** (`model: "sonnet"` o `"haiku"`) — nunca heredar el caro por omisión.
3. **Prompt de spawn mínimo pero completo**: todo lo que va en el prompt entra al contexto del agente desde el inicio. Contrato completo sí, contexto decorativo no.
4. **No poll de agentes en background** — el harness notifica al completar.
5. **Cache-awareness**: no editar CLAUDE.md/tools a mitad de sesión larga (invalida cache jerárquico). Contenido especializado vive en skills (on-demand), no en CLAUDE.md (residente).
6. **Retornos cortos**: pedir al agente output acotado — el aislamiento de contexto no protege contra un reporte verboso (3k tokens devueltos = 3k tokens pagados por el caller).

## Contrato de executor — unificado (P1, P2 y P3)

Todo prompt de executor incluye, sin excepción:

1. **Rol y autoridad**: "Vos SOS el executor. Escribí el código/contenido, no pidas permiso, no preguntes — ejecutá." (Sin esto: stalls pidiendo permiso, confirmado en uso real.)
2. **Objetivo explícito** + qué NO hacer (límites de scope).
3. **Formato de output y paths exactos** (disjuntos si hay paralelo).
4. **Criterios de aceptación no-negociables** medibles (tests pasan, archivo existe, N palabras, schema).
5. **Action triggers**: condiciones que MANDAN actuar autónomo + la única condición de escalar (bloqueado de verdad, no "duda").
6. **Contexto pegado literal** (brief/move completo) — nunca "ver el brief anterior": el agente arranca frío, no ve la sesión.

## Verificación — en artefactos, nunca en el transcript

Los agentes generan "completion language" sin importar el estado real (confirmado externamente y 2x en uso propio). Antes de reportar cualquier pieza como completa:

1. `ls` de cada path esperado.
2. Leer y confirmar no-vacío / formato correcto.
3. Si hay código: correr los tests correspondientes.
4. Falta algo → diagnosticar con fallo CLASIFICADO (qué check falló + últimas líneas de output), no blind-retry.

**Escalation ladder:** falla haiku → 1 retry en sonnet con el fallo clasificado en el prompt. Falla sonnet → parar y reportar. Nunca escalar al modelo caro para ejecutar — mata el ahorro.

---

## P0 — INLINE (default)

Sin agentes. El orquestador hace el trabajo directo con sus tools. El ahorro viene de effort ≤high y de no pagar cold-starts, no de delegar. Cubre: tareas de 1-2 pasos, edición de pocos archivos, lógica que necesita el contexto ya cargado, docs/notas/planes escritos por la sesión actual.

## P1 — BRIEF + EXECUTORS MECÁNICOS (v1 clásico)

**Cuándo:** N piezas independientes, spec totalmente resuelta, cero juicio nuevo por pieza (variantes de contenido, archivos templados, conversiones).

**Gate de entrada — completitud del spec:** si el brief deja ambigüedad, el executor barato la AMPLIFICA (evidencia: schemas rotos, improvisación en edge cases, rumiar sin actuar). El brief debe tener: decisiones tomadas, instrucciones exactas, 1 ejemplo concreto de una unidad de output, assumptions, executor recomendado (mecánico puro → haiku; necesita cohesión → sonnet).

**Flujo:**
1. Brief: si la sesión ya es el modelo caro → INLINE (nunca Agent call de research que re-derive contexto). Si no → 1 Agent call al modelo caro con presupuesto fijo de búsquedas. Guardar brief a scratchpad.
2. Checklist de calidad del brief: ¿criterios medibles? ¿paths exactos? ¿ejemplo concreto? Máximo 1 re-round de corrección.
3. Gate de confirmación con el usuario (brief + executor recomendado). Sin excepción.
4. Ejecución: 1 pieza → foreground; ≥2 piezas → paralelo en background, mismo mensaje, brief pegado literal en cada prompt, paths disjuntos.
5. Verificación en disco (arriba).

## P2 — WAR-GAME + EXECUTORS QUE RAZONAN (patrón Sprint 7/8/9)

**Cuándo:** código con lógica de negocio, multi-move, dependencias secuenciales. Lo que v1 prohibía — ahora tiene patrón propio, validado 3x en producción (JARVIS Sprints 7, 8, 9).

**Estructura:**
1. **Pensar caro UNA vez:** Fable/Opus escribe el war-game move-by-move (no un plan lineal): por move → contrato/firma, reglas duras, fallo más probable + señal + contramovimiento, test, verificación del orquestador. El war-game se guarda como archivo markdown versionado junto al proyecto.
2. **Ejecutar por move:** un executor Sonnet por move (contrato de executor arriba + el move pegado literal). Moves dependientes = SECUENCIAL, nunca paralelo. Moves genuinamente independientes pueden ir en paralelo solo si tocan archivos disjuntos.
3. **Gate entre moves (ground truth per step):** el orquestador verifica en disco + corre la suite de tests ANTES de lanzar el siguiente move. Sin verde, no hay move siguiente. (Por qué: error compuesto — 85% de acierto por paso = ~20% a 10 pasos; el gate corta la acumulación.)
4. **Anti-self-grading cuando aplica:** en features con tests nuevos, el que escribe los tests ≠ el que escribe la implementación (dos executors distintos).
5. **Cierre:** suite completa verde + commit + validación humana como último move (DoD nunca es "tests verdes" solo).

## P3 — RESEARCH FAN-OUT (deep-dive)

**Cuándo:** el único caso donde multi-agent paralelo GANA (Anthropic): research breadth-first con sub-preguntas independientes que exceden lo que una sesión cubre secuencialmente rápido.

**Reglas:**
- Base de conocimiento propia ANTES de decomponer: lo que ya tenés documentado no se investiga afuera.
- 5-8 sub-preguntas independientes + 1 abogado del diablo, TODOS en un solo mensaje (paralelo real), workers Sonnet.
- Delegación con contrato (Anthropic): objetivo explícito + formato de output (JSON con claims/fuentes/confianza) + guía de tools + límites. Delegación vaga = trabajo duplicado.
- Retorno = hallazgos condensados; el orquestador sintetiza y persiste (raws + brief a tu base de conocimiento).
- No relanzar un agente para repreguntar algo chico — SendMessage al agente existente conserva su contexto.

---

## Resumen del flujo

```
Paso 0: selector → "Patrón: PX — motivo" (1 línea visible)
P0 → hacerlo directo (effort ≤high)
P1 → brief (inline si sesión cara) → checklist → confirm gate → executors paralelos → verificar disco
P2 → war-game escrito → confirm → executor Sonnet por move → gate disco+tests entre moves → cierre humano
P3 → conocimiento propio primero → decomponer → fan-out paralelo Sonnet + devil → sintetizar → persistir
Transversal: contrato de executor completo · verificación en artefactos · retry clasificado (haiku→sonnet→stop) · effort ≤high
```
