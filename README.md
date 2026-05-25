# Agentic Customer Support para SaaS B2B

> Agente autónomo que resuelve consultas de soporte nivel 1 y escala a humano cuando no puede resolver

Agente conversacional que clasifica consultas, evalúa su propia confianza antes de responder y escala automáticamente cuando el confidence score es menor a 0.7 — enviando un resumen estructurado del caso al equipo humano vía Telegram en menos de 5 segundos.

**Demo:** próximamente  

---

## El problema que resuelve

Los chatbots de soporte tienen un problema universal: cuando no saben la respuesta, igual responden. En SaaS B2B, una respuesta incorrecta puede costar un cliente. Este agente sabe cuándo rendirse — y cuando lo hace, entrega al humano el contexto completo para actuar de inmediato.

---

## Arquitectura

```
Usuario escribe consulta
    ↓
AI Agent (GPT-4o-mini + memoria de contexto)
    ↓
Output estructurado: CONFIDENCE + RESOLVED + RESPONSE / ESCALATE_REASON + SUMMARY
    ↓
Nodo de decisión: extrae confidence score
    ↓
    ├── confidence ≥ 0.7 → Responde + guarda en Supabase
    └── confidence < 0.7 → Telegram con resumen estructurado
```

---

## Decisiones de diseño clave

**¿Por qué confidence score y no keywords de escalado?**
Una lista de keywords es frágil: un usuario puede describir una crisis sin usar ninguna palabra de la lista. El confidence score autoreportado por el modelo evalúa la complejidad semántica real de la consulta. Preferís escalar de más que responder mal en un contexto crítico.

**¿Por qué el umbral es 0.7 y no 0.5 o 0.9?**
0.7 es conservador: cubre la zona donde el modelo "cree que puede" pero con dudas razonables. Por debajo de 0.5 sería demasiado permisivo. Por encima de 0.9 el agente escalaría casi todo. El umbral correcto se calibra con feedback real en producción.

**¿Por qué Telegram en lugar de email para el escalado?**
La Telegram Bot API es la más simple del mercado: un token y podés mandar mensajes sin OAuth, sin Google Cloud Console, sin redirect URIs. El mensaje llega instantáneamente al celular del equipo. Para equipos pequeños de SaaS B2B, Telegram es más efectivo que email para alertas urgentes.

**¿Por qué memoria de contexto de 10 intercambios?**
Soporte nivel 1 rara vez necesita más de 10 turnos para resolver o escalar. Más contexto aumenta el costo por consulta sin agregar valor. Menos contexto hace que el agente "olvide" información crítica del inicio de la conversación.

---

## Stack

| Capa | Herramienta |
|---|---|
| Orquestación | n8n |
| LLM | OpenAI GPT-4o-mini |
| Memoria | Window Buffer Memory (10 intercambios) |
| Storage | Supabase |
| Escalado | Telegram Bot API |

---

## Schema de base de datos

```sql
create table support_conversations (
  id            bigserial primary key,
  session_id    text not null,
  user_message  text not null,
  agent_response text,
  intent        text,
  resolved      boolean default false,
  escalated     boolean default false,
  confidence    float,
  created_at    timestamp with time zone default now()
);

alter table support_conversations enable row level security;
```

---

## Variables de entorno necesarias

```env
OPENAI_API_KEY=
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
TELEGRAM_BOT_TOKEN=
TELEGRAM_CHAT_ID=
```

---

## Resultados del demo

- Consulta compleja (acceso a cuenta + demo urgente) → escalado correcto, confidence: 0.5
- Consulta simple (cambio de contraseña) → resuelta autónomamente, guardada en Supabase
- Tiempo de escalado detección → Telegram: < 5 segundos

---

## Lo que aprendí construyendo esto

Diseñar el criterio de escalado antes de construir el agente es la decisión de arquitectura más importante del sistema. Todo lo demás es plomería.
