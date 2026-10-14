# Renovadas — Agente de Atención al Cliente (Proyecto Integrador)

Proyecto integrador del curso de Configuración e Interfaces Agénticas (Coderhouse — Automatización con IA). Un solo flujo de n8n que se amplía módulo a módulo, desde el agente base (M1) hasta el Proyecto Final Integrador.

**Alumna:** Ingrid Ledesma · **Cursada:** septiembre–octubre 2026

## 🧵 Contexto

**Renovadas** es una tienda online de lentes Ray-Ban originales que opera bajo modalidad dropshipping, dependiendo de un excel compartido de proveedores para stock y precios. Este proyecto integrador busca resolver dos problemas reales del negocio:

1. **Back-office**: sincronizar el catálogo (stock/precios) que cambia constantemente por depender de proveedores externos — desarrollado en un integrador previo.
2. **Atención al cliente**: un agente conversacional que responda consultas de catálogo, gestione pedidos y derive a revisión humana los casos sensibles, usando siempre datos actualizados de Notion como fuente de verdad.

Desde M2, el agente tiene nombre y personalidad propia: **Rena**, la asistente virtual de Renovadas.

## 🗺️ Hoja de ruta del proyecto

| Módulo | Contenido | Carpeta | Estado |
|---|---|---|---|
| M1 | Agente base: Trigger + AI Agent + System Prompt + Tools + Log de observabilidad | `entregable 1/` | ✅ |
| M2 | Multi-agente: Manager + Workers como sub-workflows | `entregable 2/` | ✅ |
| M3 | Memoria: contexto de conversación por Session_ID | `entregable 3/` | ✅ |
| M4 | Integraciones reales: Gmail Trigger + CRM (HubSpot) + Slack + HITL | `entregable 4/` | ✅ |
| M5 | RAG / base documental: LlamaParse + Cohere + Pinecone | `entregable 5/` | ✅ |
| M6 | Voz: Telegram + Whisper (STT) + ElevenLabs (TTS) | `entregable 6/` | ✅ |
| M7 | Diseño arquitectónico del sistema agéntico vertical (documento) | `entregable 7/` | ✅ |
| M8 | Supervisor + Juez (AI-as-a-Judge) + matriz A/B de costos | `entregable 8/` | ✅ |
| M9 | Gobernanza, TCO/ROI y trazabilidad (`traceId`) | `entregable 9/` | ✅ |
| M10 | — | `entregable 10/` | ⏳ |
| Final | Proyecto Final Integrador | `entrega final/` | ⏳ |

**Regla de oro del curso:** no se rehace el flujo desde cero en cada checkpoint — cada módulo parte del `.json` del anterior y lo extiende.

## 🧰 Stack

| Capa | Herramienta |
|---|---|
| Orquestación | n8n (self-hosted en VPS Hostinger KVM 2 + Easypanel) |
| LLM | Groq (`openai/gpt-oss-120b`) |
| Datos operativos | Notion (catálogo, pedidos) |
| Memoria y auditoría | Airtable (contexto por sesión, `Veredictos_Juez`) |
| Base vectorial / RAG | Pinecone (`renovadas-index`) + embeddings Cohere `embed-multilingual-v3.0` |
| Parseo documental | LlamaParse (LlamaCloud) |
| CRM | HubSpot |
| Canales | Gmail (texto), Telegram (voz), Slack (supervisión / HITL) |
| Voz | Whisper `whisper-large-v3` (STT) · ElevenLabs `eleven_multilingual_v2` (TTS) |

## 📁 Estructura del repositorio

```
├── README.md
├── docs/
│   └── entregable 1/capturas del flujo/          ← capturas de M1
├── entregable 1/
│   └── checkpoint1_Ingrid_ledesma.json           ← M1: Agente base
├── entregable 2/
│   ├── preentrega_modulo2_ledesma_ingrid.docx.pdf   ← Entregable oficial de M2
│   ├── Manager_Modulo2_Ledesma_Ingrid.json
│   ├── Worker_Catalogo_Modulo2_Ledesma_Ingrid.json
│   ├── Worker_Tracking_Modulo2_Ledesma_Ingrid.json
│   ├── Worker_Pedidos_Modulo2_Ledesma_Ingrid.json
│   ├── Worker_Escalamiento_Modulo2_Ledesma_Ingrid.json
│   └── imagenes del flujo/
├── entregable 3/
│   ├── PreEntrega_Modulo3_IngridLedesma.docx.pdf
│   └── entregable_3_memory_manager_worker.json   ← M3: memoria
├── entregable 4/
│   ├── checkpoint4_ingrid_ledesma.json           ← M4: integraciones externas
│   └── capturas de pantalla del flujo entregable 4/
├── entregable 5/
│   ├── PreEntrega_Modulo5_IngridLedesma.pdf
│   ├── Manual_Politicas_Renovadas_2026.docx      ← documento maestro indexado
│   └── checkpoint5_ingrid_ledesma.json           ← M5: RAG
├── entregable 6/
│   └── PreEntrega_Modulo6_IngridLedesma.docx     ← M6: capa de voz
├── entregable 7/
│   └── Entregable7_IngridLedesma.docx            ← M7: diseño arquitectónico
├── entregable 8/
│   ├── Checkpoint_Modulo8_Ledesma_Ingrid.pdf
│   └── checkpoint8_ingrid_ledesma.json           ← M8: Juez + M9: traceId
├── entregable 9/
│   └── PreEntrega_Modulo9_IngridLedesma.pdf      ← M9: gobernanza y TCO
├── entregable 10/                                 (pendiente)
└── entrega final/                                 (pendiente)
```

A medida que avance el curso se van sumando carpetas `entregable N/`, manteniendo el historial completo de la evolución del proyecto. Los módulos que no exportan un `.json` nuevo (M6, M7, M9) documentan cambios que viven en el workflow del checkpoint más cercano.

## 🤖 M1 — Agente Base y Motor de Razonamiento

### Arquitectura del flujo

- **Trigger**: Chat Trigger (`When chat message received`)
- **AI Agent**: modo Tools Agent, modelo Groq (con soporte de tool-calling), Max Iterations = 6
- **System Prompt modular**: Rol y Ámbito → Objetivo Operativo → Reglas de Uso de Herramientas → Protocolo de Escalamiento / Guardrails → Estilo
- **Tools conectadas al agente**:
  - `Consultar_Catalogo_Notion` — consulta disponibilidad, precio y características de los lentes
  - `Crear_Pedido_Notion` — registra un pedido nuevo cuando el cliente confirma los datos
  - `Consultar_Pedido_Notion` — verifica el estado de un pedido y su código de seguimiento
- **Flujo de salida condicional** (con nodos IF sobre la respuesta del agente):
  - `Send a message` (Slack) → log de observabilidad, se ejecuta siempre
  - `Enviar_Codigo_Seguimiento_Gmail` → se dispara solo si la respuesta incluye el envío de un código de seguimiento
  - `Human in the loop` (Slack Send & Wait) → se dispara solo si la respuesta indica que el caso requiere revisión humana (reclamos, cancelaciones, casos fuera de catálogo)

### Rol del agente

Agente de Atención al Cliente de Renovadas: responde consultas de catálogo consultando siempre Notion como fuente de verdad, gestiona el registro inicial de pedidos, y deriva a un humano los casos sensibles (reclamos, cancelaciones, pagos) después de reunir los datos mínimos del pedido.

### Limitaciones conocidas de esta versión

- Sin memoria de conversación (se implementa en M3): cada mensaje se procesa de forma aislada, por lo que el agente puede no recordar datos provistos en mensajes anteriores dentro de la misma sesión.
- Modelo Groq elegido por practicidad; requiere que el modelo seleccionado soporte tool-calling nativo (no todos los modelos disponibles en Groq lo soportan).

## 🧩 M2 — Arquitectura Multi-Agente (Manager + Workers)

### Motivación

El agente único de M1 mezclaba en un solo System Prompt la lógica de catálogo, pedidos, tracking y escalamiento, lo que dificulta el mantenimiento y la extensión del flujo. En M2 se separó esa lógica en agentes especializados, cada uno independiente y reutilizable, coordinados por un Manager que solo clasifica y enruta.

### Arquitectura del flujo

- **Manager** (`Manager_Modulo2_Ledesma_Ingrid.json`):
  - **Trigger**: Chat Trigger
  - **Router AI Agent** ("Rena"): clasifica cada mensaje entrante en `intent` (catalogo / tracking / pedidos / saludo), `confidence` y `risk` (LOW/HIGH), devolviendo un JSON estructurado.
  - **Code**: parsea la salida del Router a JSON, con manejo de errores.
  - **IF (risk)**: si `risk = HIGH`, deriva directo al Worker de Escalamiento, saltando el Switch.
  - **Switch (intent)**: para los casos de riesgo bajo, enruta según `intent` al Worker correspondiente.
  - **Execute Workflow** (uno por Worker): invoca al sub-workflow correspondiente con **"Wait For Sub-Workflow Completion" activado**, para esperar la respuesta antes de continuar.
  - **Edit Fields**: unifica la respuesta de cualquier Worker en un único formato para el chat.

- **Workers** (sub-workflows independientes, cada uno con su propio `Execute Workflow Trigger`):
  - `Worker_Catalogo_Modulo2_Ledesma_Ingrid.json` — consulta disponibilidad, precio y características en Notion.
  - `Worker_Tracking_Modulo2_Ledesma_Ingrid.json` — consulta el estado de un pedido y envía el código de seguimiento por Gmail.
  - `Worker_Pedidos_Modulo2_Ledesma_Ingrid.json` — registra un pedido nuevo en Notion.
  - `Worker_Escalamiento_Modulo2_Ledesma_Ingrid.json` — deriva el caso a revisión humana vía Slack (Human in the loop) y registra el resultado.

### Esquema de datos (contrato Manager ↔ Worker)

**Manager → Worker** (mínimo indispensable, evitando "data stuffing"):
```json
{
  "original_input": "texto del mensaje del cliente",
  "intent": "catalogo | tracking | pedidos | escalamiento",
  "confidence": 0.0,
  "risk": "LOW | HIGH"
}
```

**Worker → Manager** (formato uniforme en los 4 Workers, para poder unificarlo con un solo Edit Fields):
```json
{
  "status": "ok | error",
  "worker": "CATALOGO | TRACKING | PEDIDOS | ESCALAMIENTO",
  "respuesta": "texto de respuesta para el cliente",
  "requires_human": true
}
```

Cada Worker parsea la salida de su propio AI Agent con un nodo `Code` envuelto en try/catch: si el LLM no devuelve un JSON válido, se genera un objeto de contingencia (`status: "error", requires_human: true`) en lugar de romper el flujo, evitando el antipatrón de "falta de manejo de fallos en el Worker".

### Criterio de enrutamiento

1. Primero se evalúa el **riesgo** (`risk`): un caso de riesgo alto (reclamo, cancelación, pago, dato sensible) va directo al Worker de Escalamiento, sin pasar por la clasificación de intención.
2. Solo si el riesgo es bajo, se enruta por **intención** (`intent`) al Worker especializado correspondiente (catálogo, tracking o pedidos).

Esto prioriza la seguridad del cliente y evita que un caso sensible se resuelva automáticamente solo por coincidir con una intención de bajo riesgo.

### Antipatrones evitados

- **Mono-Bloque**: la lógica no está toda en un solo agente; cada Worker es un sub-workflow independiente y especializado.
- **Data Stuffing**: el payload Manager→Worker lleva solo los 4 campos necesarios, no todo el contexto de la conversación.
- **Ignorar Time-Outs**: el Execute Workflow del Manager tiene activado "Wait For Sub-Workflow Completion", asegurando que el Manager espere la respuesta real del Worker antes de continuar.
- **Falta de manejo de fallos en el Worker**: cada Worker valida su propia salida con try/catch antes de devolver la respuesta al Manager.

### Entregable

El entregable oficial de M2 es `entregable 2/preentrega_modulo2_ledesma_ingrid.docx.pdf`, con capturas del canvas del Manager, de al menos 2 Workers, de la configuración del Execute Workflow (payload + toggle de espera activado), de una ejecución exitosa end-to-end, y del log de observabilidad en Slack. Los `.json` de Manager y Workers y las capturas (`imagenes del flujo/`) se incluyen como respaldo adicional.

### Limitaciones conocidas de esta versión

- Sin memoria de conversación (se implementa en M3): el Router del Manager sigue clasificando cada mensaje de forma aislada.
- Modelo Groq elegido por practicidad; requiere que el modelo seleccionado soporte tool-calling nativo (no todos los modelos disponibles en Groq lo soportan).

## 🧠 M3 — Memoria y Contexto de Conversación

### Motivación

En M2 cada mensaje se procesaba de forma aislada: el Manager clasificaba sin saber qué había pasado antes. M3 agrega memoria persistente por sesión, permitiendo que el agente mantenga contexto a lo largo de la conversación.

### Arquitectura de memoria

- **Session ID**: derivado del remitente del email (`$json.From.replace(/[^a-zA-Z0-9]/g, '_')`), identifica de forma única cada conversación.
- **Lectura Airtable**: al inicio de cada ejecución, busca en Airtable si existe contexto previo para esa sesión usando `filterByFormula`.
- **IF (¿Existe contexto?)**: si encuentra un registro, lo carga como contexto recuperado; si no, inicializa uno vacío.
- **Simple Memory** (Buffer Window): ventana de 3 mensajes para mantener contexto conversacional inmediato dentro del AI Agent.
- **Circuito de persistencia** (después de la respuesta del Worker):
  - **¿Hay varios mensajes?**: si hay más de 5 mensajes acumulados, pasa por un LLM Chain que comprime el historial en un resumen estructurado (JSON con `asunto_principal`, `puntos_clave`, `accion_requerida`).
  - **Guardar resumen comprimido** / **Guardar conversación cruda**: upsert en Airtable con el session_id como clave, guardando el resumen, datos clave, estado del proceso y contador de mensajes.

### Campos de contexto

```json
{
  "ctx_resumen": "resumen acumulado de la conversación",
  "ctx_datos": "datos clave del cliente",
  "ctx_estado": "nuevo | En proceso | Resuelto | Escalado",
  "ctx_mensajes": 0
}
```

### Entregable

`entregable 3/entregable_3_memory_manager_worker.json` (flujo con memoria) + `PreEntrega_Modulo3_IngridLedesma.docx.pdf`.

## 🔌 M4 — Integraciones Externas (Gmail + HubSpot + Slack + HITL)

### Motivación

M3 seguía usando un Chat Trigger manual. M4 reemplaza el trigger por una integración real con Gmail y agrega tres integraciones externas que conectan el agente con herramientas de negocio reales: CRM (HubSpot), email (Gmail Draft como barrera HITL) y notificaciones (Slack).

### Nodos de rúbrica

El checkpoint 4 requiere 4 nodos específicos que cumplen requisitos de la rúbrica:

1. **IF anti auto-reply (`¿es auto-reply?`)**: filtra correos automáticos (auto-reply, out of office, undeliverable, no-reply@, y el propio email del agente) para evitar bucles infinitos. Usa 5 condiciones AND con `notContains` sobre `$json.Subject` y `$json.From`.

2. **HubSpot Lookup antes de Create (`HubSpot - Buscar contacto` → `¿Contacto existe?` → `HubSpot - Actualizar contacto`)**: busca el contacto en el CRM por email antes de crear/actualizar. Si existe, actualiza; si no, crea uno nuevo. Patrón Search → IF → Upsert.

3. **Gmail Create Draft (`Create a draft`)**: crea un borrador en Gmail con la respuesta del agente en lugar de enviar directamente. Es la barrera **Human-in-the-Loop (HITL)**: un humano debe revisar y aprobar el borrador antes de enviarlo al cliente.

4. **Set payload cleanup (`Set - Validar payload`)**: limpia y valida los datos antes de pasarlos a las integraciones externas. Extrae `contact_email`, `contact_name`, `email_subject`, `agent_response` e `is_valid`, garantizando que HubSpot y Slack reciban datos limpios y evitando errores 400 por campos vacíos u objetos binarios.

### Flujo completo (de punta a punta)

```
Gmail Trigger (cada minuto, INBOX)
  → ¿es auto-reply? (IF anti-loop)
    → [True] Limpiar payload (Set: email_from con regex, email_subject, email_body, session_id)
      → Lectura Airtable (buscar contexto por session_id)
        → ¿Existe contexto? → Contexto recuperado / Contexto nuevo
          → AI Agent (Groq, clasificación de intent)
            → Contrato (Code: parseo JSON + fallback)
              → ¿Es riesgo alto?
                → [True] Worker Escalamiento
                → [False] Switch (intent) → Worker correspondiente
                  → Respuesta de workers
                    → Rama 1: Circuito de memoria (Airtable)
                    → Rama 2: Set - Validar payload
                      → HubSpot - Buscar contacto
                        → ¿Contacto existe?
                          → HubSpot - Actualizar contacto (upsert)
                            → Create a draft (Gmail, HITL)
                              → Slack - Alerta equipo
    → [False] Rebote de no deseados (NoOp)
```

### Limpiar payload — Extracción de campos del Gmail Trigger

El Gmail Trigger v1.4 devuelve los campos con **mayúscula inicial** (`From`, `Subject`, `To`, `snippet`). El nodo `Limpiar payload` normaliza estos campos:

| Campo | Expresión | Nota |
|---|---|---|
| `email_from` | `={{ $json.From.match(/<(.+)>/)?.[1] \|\| $json.From }}` | Extrae el email limpio de `"Nombre <email>"` |
| `email_subject` | `={{ $json.Subject }}` | |
| `email_body` | `={{ $json.snippet }}` | Texto del email |
| `session_id` | `={{ $json.From.replace(/[^a-zA-Z0-9]/g, '_') }}` | Clave única para memoria |

### Integraciones configuradas

| Servicio | Nodo | Credencial | Función |
|---|---|---|---|
| Gmail | Gmail Trigger | OAuth2 | Captura emails entrantes (INBOX, cada minuto) |
| Gmail | Create a draft | OAuth2 | Crea borrador con la respuesta del agente (HITL) |
| HubSpot | Buscar / Actualizar contacto | Private App Token | CRM: busca y crea/actualiza contactos |
| Slack | Alerta equipo | API Token | Notifica al canal `#soporte-renovadas-human-in-the-loop` |
| Airtable | Lectura / Guardar | API Token | Memoria persistente por sesión |

### Entregable

El archivo `entregable 4/checkpoint4_ingrid_ledesma.json` contiene el workflow completo exportado desde n8n, incluyendo los 4 nodos de rúbrica y todas las integraciones. Las capturas están en `capturas de pantalla del flujo entregable 4/`.

## 📚 M5 — Conocimiento Organizacional (RAG)

### Motivación

Atención al cliente recibe de forma repetida las mismas consultas (autenticidad, envíos, devoluciones, medios de pago, talles), todas respondidas en el **Manual de Políticas y Procedimientos de Renovadas 2026**. El documento es demasiado largo para meterlo entero en el prompt, y las respuestas deben ser fieles al manual y no inventadas: caso de uso típico de RAG.

### Decisión de diseño (desvío justificado de la consigna)

La consigna describe un Vector Store Retrieve **Tool** acoplado al AI Agent en modo Tools Agent. En la implementación real:

- El parseo documental se hace con **LlamaParse** (LlamaCloud) y los vectores se guardan en **Pinecone**.
- La recuperación se resolvió con un nodo **Vector Store independiente en modo "Get Many"** (no como tool interna del agente), porque los modelos gratuitos de Groq no invocaban la herramienta de forma confiable dentro del ciclo ReAct. Separar clasificación (AI Agent) de recuperación + generación (rama RAG dedicada) resultó mucho más estable.

### Pipeline de ingesta

```
Subir PDF (Form Trigger)
  → Normalizar archivo (Code)
    → LlamaParse: subir PDF (HTTP)
      → Esperar 10 s → LlamaParse: ¿terminó? → ¿Estado del parseo? (Switch)
        → [error] Parseo falló (Stop and Error)
        → [ok] Chunking por títulos (Code)
          → Cargar chunk + metadata → Embeddings Cohere → Guardar en Pinecone
```

- **Curaduría previa**: se verificó jerarquía de títulos numerada (1 › 1.1 › 1.1.1) y tablas bien formadas para que LlamaParse las devuelva como celdas y no como texto plano (salida Markdown, tier `cost_effective`).
- **Chunking propio** (nodo Code): segmenta por secciones y antepone la ruta del título a cada fragmento (ej. `1. Información General › 1.1. Presentación del Producto`), lo que enriquece el embedding y permite citar la fuente. El text splitter está en modo "No volver a cortar" para no romper esos chunks.
- **Metadata por chunk**: `source`, sección, página, chunk.

### Calibración del Vector Store

| Parámetro | Valor |
|---|---|
| Operación | Get Many |
| Índice | `renovadas-index` (Pinecone, 1024 dim, ~35 vectores) |
| Embeddings | Cohere `embed-multilingual-v3.0` |
| Top-K | 35 |
| Minimum Score | 0 (sin umbral) |

**Por qué recall-first:** los scores quedaban comprimidos en una banda angosta (~0.29–0.37) porque Cohere v3 usa embeddings asimétricos (requiere `input_type: search_query` / `search_document`) y el nodo de Cohere en n8n no expone ese parámetro, así que ambos lados se embeddean igual y el ranking se degrada. Con una base de ~35 vectores, recuperar el total maximiza el recall con costo de tokens acotado (el LLM se ejecuta una sola vez gracias a la agregación). La precisión final la aporta el LLM de generación, que filtra el contexto y cita la sección. **En una base mayor y con el `input_type` corregido el balance óptimo sería el inverso: Top-K 4–6 + Minimum Score ~0.5.**

### Regla anti-alucinación

El prompt de generación obliga a responder solo con los fragmentos recuperados, citar la sección y usar la frase de escape exacta cuando el dato no está:

> "No sé: ese dato no figura en la documentación disponible. Te sugiero contactar a renovadasok@gmail.com."

### Validación

Batería de preguntas ciegas con vocabulario informal y sinónimos (autenticidad, reembolso por producto fallado, plazos de envío a Córdoba, garantía, cuotas, arrepentimiento, lentes recetados), incluyendo un caso que debe disparar la regla "No sé". Resultado: **7/7 correctas**. La limitación real detectada no está en el recall (total con Top-K=35) sino en la precisión de recuperación: se traen fragmentos de más que el LLM debe filtrar.

### Gobernanza de la base de conocimiento

- **Fuente única de verdad**: el Manual es el documento maestro; toda actualización se hace primero ahí y luego se re-ingesta. Nunca se edita un vector suelto.
- **Revisión**: auditoría trimestral de precios, plazos, medios de pago y políticas; revisión inmediata ante cualquier cambio.
- **Responsable**: un *content owner* aprueba cada cambio antes de la reingesta.
- **Reemplazo**: versión nueva del manual → modo "Reemplazar todo" en el formulario de ingesta, para evitar vectores duplicados u obsoletos.
- **Regresión**: después de cada reingesta se vuelve a correr la batería de preguntas ciegas antes de dar la carga por válida.
- **Datos sensibles**: solo políticas y procedimientos; no se indexan datos personales de clientes ni credenciales.

### Entregable

`entregable 5/checkpoint5_ingrid_ledesma.json` + `PreEntrega_Modulo5_IngridLedesma.pdf` + el documento maestro `Manual_Politicas_Renovadas_2026.docx`.

## 🎙️ M6 — Ecosistema Multimedia de Audio (Voice AI)

### Motivación y viabilidad

Muchos clientes prefieren mandar notas de voz (menor fricción que escribir), lo que reduce la fatiga cognitiva del usuario. Para cuidar el ROI se aplica una regla estricta de **200 caracteres por respuesta**, que acota los caracteres facturados por la API de TTS.

### Circuito de extremo a extremo

```
Telegram Trigger (nota de voz)
  → If ¿tiene audio? (voice.file_id is not empty)
    → [False] Caso - No recibe audio (pide una nota de voz)
    → [True] Recibe el audio (File/Get/Download)
      → Preparar el dato binario (Code: normaliza a .ogg)
        → Whisper STT (whisper-large-v3, idioma es, campo `data`)
          → IF - ¿Transcripción válida?
            → [False] Telegram - Contingencia (pedir reenvío)
            → [True] AI Agent (Tools Agent) + Pinecone como tool (RAG del M5)
              → Truncado defensivo - 200 chars (Code)
                → ElevenLabs TTS (eleven_multilingual_v2, stability 0.5, similarity 0.75)
                  → Telegram - Enviar Audio (Send Audio, binario `data`)
```

El agente de voz reutiliza la **misma base de conocimiento** (`renovadas-index`) del M5, conectada esta vez sí como herramienta de recuperación del Tools Agent.

### Controles

- **Doble barrera de costo**: la regla de 200 caracteres está en el System Message (*"Máximo 200 caracteres por respuesta. Usá frases cortas. No uses markdown. No incluyas URLs. No uses emojis. No inventes información."*) **y** en un nodo Code de truncado defensivo antes del TTS.
- **Contingencia no-code**: si la transcripción viene vacía (audio corrupto, ruido, silencio), el flujo desvía a un mensaje de Telegram que pide reenviar.
- **Compliance / destrucción de binarios**: los audios se procesan en memoria volátil. En Workflow Settings, `Save failed / successful / manual executions = Do not save`, por lo que los binarios de voz no quedan en el historial de ejecuciones.

### Entregable

`entregable 6/PreEntrega_Modulo6_IngridLedesma.docx` (documento con capturas). El flujo de voz quedó integrado y se exporta junto al checkpoint del M8.

## 🏛️ M7 — Diseño Arquitectónico del Sistema Agéntico Vertical

Documento de diseño (sin cambios de flujo). Vertical elegida: **Sales Ops & Customer Success**, con foco en el eslabón post-venta más costoso en tiempo: seguimiento de envíos y gestión de reclamos.

### Contenido

- **Baseline manual**: seguimiento de envíos ~5–8 min c/u, reclamos ~15–20 min c/u, actualización de precios/stock 2–3 h por tanda, espera del cliente de hasta 12–24 h (todo a cargo de una sola persona).
- **Framework de priorización**: impacto ALTO · viabilidad no-code ALTA · adopción ALTA.
- **Coreografía de la red de agentes**: Router ("Rena") + Workers (Catálogo, Tracking, Pedidos, Escalamiento) + rama RAG + capa de voz, con gobernanza HITL antes de que cualquier respuesta llegue al cliente. El diseño es **agnóstico al canal**: cambiar Gmail por WhatsApp o DMs de Instagram solo cambia el nodo de entrada/salida.
- **Mapa de permisos (mínimo privilegio)**: ningún agente escribe sobre Regla Pricing ni Proveedor; Pedidos solo crea en estado "Pendiente"; Tracking envía el código solo si existe y el mail está confirmado.
- **Semáforo de autonomía**:
  - 🟢 **Verde** (100% autónomo): clasificar intención, lecturas de catálogo/pedido, responder políticas citando el manual, responder por voz, reenviar un código ya cargado, guardar memoria.
  - 🟡 **Amarillo** (autónomo con registro): registrar pedido "Pendiente", crear/actualizar contacto en HubSpot. No cierra la operación.
  - 🔴 **Rojo** (freeze + humano obligatorio): reembolsos, cancelaciones, cambios de precio, reclamos y todo `risk = HIGH`. Además, toda respuesta al cliente se genera como borrador.
- **Decisión de diseño destacada**: el **Worker Escalamiento no tiene nodo de IA**. El camino más riesgoso del sistema es determinístico y siempre termina en una persona.
- **KPIs comerciales** y **próxima expansión** (Agente de Sincronización de Catálogo, que ataca las 2–3 h manuales por actualización).

### Entregable

`entregable 7/Entregable7_IngridLedesma.docx`.

## ⚖️ M8 — Supervisor + Juez (AI-as-a-Judge)

### Arquitectura

- **Un solo Juez, no cuatro copias**: implementado como sub-workflow `Juez_Renovadas` e invocado con Execute Workflow. Cambiar la rúbrica en un lugar actualiza a todos los Workers.
- **Punto de convergencia**: Workers y rama RAG desembocan en `Respuesta de workers`, que normaliza `respuesta_worker`, `contexto`, `pregunta` e `intento`. Ninguna ruta se escapa de la auditoría.
- **El Juez pone la nota, el código decide la ruta**: umbrales, máximo de reintentos y regla dura de dinero viven en un nodo Code auditable, no en el LLM.
- **Modelo**: `openai/gpt-oss-120b` en Groq, temperatura 0, con Structured Output Parser.

### Rúbrica (arranca en 5, mínimo 1)

| Criterio | Pregunta | Descuento |
|---|---|---|
| Exactitud factual | ¿Menciona datos que no están en el contexto recibido? | -2 |
| Coherencia con la intención | ¿Contesta lo que el cliente preguntó? | -1 |
| Tono y extensión | ¿Es empática, profesional y concisa (máx. 5 líneas)? | -1 |
| Accionabilidad | ¿Le da al cliente un próximo paso claro? | -1 |

Casos especiales: si el dato realmente no está y el Worker lo reconoce y deriva al mail de contacto, no se descuenta; si lo consultado no se vende y el Worker lo aclara, es correcto; si el contexto permitía aclararlo y el Worker contestó con un "No sé" genérico, se descuenta 1 por coherencia.

### Switch de tres rutas

| Ruta | Condición (decidida por código) | Acción |
|---|---|---|
| **ACEPTADO** | Nota 4 o 5 y sin regla dura | Entrega la respuesta, guarda memoria en Airtable y sigue el circuito de CRM (HubSpot → borrador Gmail → Slack) |
| **CORREGIR** | Nota 3 e `intento = 0` | `Mandar reintento por correccion` devuelve al Worker los `errores_juez` con `intento = 1`. **Máximo 1 reintento**; si vuelve a dar 3, pasa a ESCALAR |
| **ESCALAR** | Nota 1-2, veredicto RECHAZADO, o la respuesta menciona reembolso / devolución (regla dura) | Slack "Send and Wait for Response": el flujo se pausa y publica pregunta, nota, errores y respuesta propuesta con botones **Aprobar / Editar** |

### Reglas duras (nodo Code `Umbrales`)

```js
const siempre = /(reembols|devoluci|cr[eé]dito a favor)/i.test(texto);
const promo = (texto.match(/(descuento|bonificaci|gratis|sin cargo)/gi) || [])
  .some(p => !ctx.includes(p.toLowerCase()));   // solo cuenta si NO figura en el contexto
const prometeDinero = siempre || promo;
```

Reembolso y devolución **siempre** escalan a una persona. Los términos de promoción escalan solo cuando no figuran en el contexto, para no rechazar respuestas correctas que citan una política real.

### Esquema de salida del Juez

```json
{
  "errores": ["..."],
  "justificacion": "1-2 frases explicando el puntaje",
  "exactitud_factual": 3,
  "estado_veredicto": "ACEPTADO | CORREGIR | RECHAZADO"
}
```

`exactitud_factual` y `estado_veredicto` gobiernan el Switch (el estado final lo escribe el código); `errores` y `justificacion` son la crítica que se reenvía al Worker en CORREGIR y el detalle que se muestra en Slack y se guarda en auditoría.

### Auditoría

Cada veredicto se graba en Airtable (base `Veredictos_Juez`) con el nodo `Log veredicto Juez`, **en paralelo a la salida del Juez y antes del Switch**, para registrar también las rutas que no terminan en una respuesta automática. Campos: `Pregunta`, `Puntaje`, `estado`, `Errores`, `Justificacion`, `Respuesta`, `Intento`, `Regla_dura`.

### Matriz A/B (10 corridas)

Las mismas 5 preguntas en dos arquitecturas: **A** (Router → Worker → salida directa) y **B** (Router → Worker → Juez → Switch → salida).

| | A · sin Juez | B · con Juez | Diferencia |
|---|---|---|---|
| Precisión promedio (1-5) | 5,0 (criterio humano) | 4,6 (rúbrica) | -0,4 |
| Respuestas con nota < 5 | 0 de 5 | 2 de 5 | +2 |
| Escaladas a una persona | 0 de 5 | 1 de 5 | +1 |
| Tokens por ejecución (prom.) | 5.475 | 11.799 | ×2,15 |
| Costo por ejecución | US$ 0,0055 | US$ 0,0118 | +115 % |

**Lectura:** el Juez no mejoró las respuestas (la ruta CORREGIR no se activó) — las audita y bloquea las riesgosas. El caso testigo es P3 ("quiero devolver un producto que me llegó roto"): el Juez la calificó 5/5, pero la regla dura la pasó a ESCALAR porque mencionaba el proceso de reembolso. Pagar el doble compra una garantía concreta, no una mejora de calidad medible.

**Recomendación comercial:** activar el Juez de forma **selectiva** — obligatorio en intenciones con riesgo económico o reputacional (devoluciones, escalamiento, pedidos, precios), omitido en saludos y despedidas, y con la regla dura de dinero siempre encendida. Auditar la mitad del tráfico baja el costo proyectado de US$ 11,80 a ≈US$ 8,88 por 1.000 ejecuciones (25 % menos) sin sacar el control de donde está el riesgo.

**Notas de lectura de los datos:** 5 preguntas por arquitectura son ilustrativas, no estadísticas; la nota de A es criterio humano y la de B es la rúbrica del Juez. Entre tandas la nota cambió en 3 de 5 preguntas (±1 punto) aun con temperatura 0 — por eso se decide con umbrales y reglas duras, no con la nota como medida exacta.

### Entregable

`entregable 8/checkpoint8_ingrid_ledesma.json` + `Checkpoint_Modulo8_Ledesma_Ingrid.pdf`.

## 🛡️ M9 — Gobernanza, Monitoreo Financiero y Trazabilidad

Documento de plan (`entregable 9/PreEntrega_Modulo9_IngridLedesma.pdf`) más la implementación del `traceId` en el flujo. Tres dimensiones:

### Dimensión 1 — Economía del token

**KPIs de negocio**

| KPI | Valor actual | Meta | Medición |
|---|---|---|---|
| Tasa de resolución autónoma | 92 % | ≥ 80 % | Registros en `Veredictos_Juez` con estado ACEPTADO / total |
| Tiempo medio de respuesta | < 10 s | < 30 s | Timestamp de entrada vs. timestamp de respuesta |
| Errores financieros sin supervisión | 0 % | 0 % | Registros con `Regla_dura` no vacío y estado ≠ ESCALAR |

**TCO mensual a 1.000 ejecuciones: US$ 45,63**

| Componente | Tokens/ejec. | Costo mensual | % del total |
|---|---|---|---|
| Router | 967 | US$ 0,97 | 2,1 % |
| Workers (LLM Chain + RAG) | 4.986 | US$ 4,99 | 10,9 % |
| Juez | 5.847 | US$ 5,85 | 12,8 % |
| **Subtotal tokens** | **11.799** | **US$ 11,80** | **25,9 %** |
| VPS Hostinger KVM 2 + Easypanel | — | US$ 33,83 | 74,1 % |
| Airtable / Pinecone / Slack / HubSpot | — | US$ 0,00 (planes gratuitos) | 0 % |

**ROI ~903 %**: costo manual evitado US$ 457,82/mes (1.000 consultas repartidas entre catálogo, tracking, devoluciones y saludos, a ~US$ 3,87/hora de referencia) contra US$ 45,63/mes de TCO → ahorro neto US$ 412,19/mes (US$ 4.946,28/año). El agente se paga en poco más de 3 días de operación.

Notas de honestidad del cálculo: hoy el agente corre sobre el **plan gratuito de Groq** (gasto real en tokens US$ 0), pero el plan proyecta a precio de mercado porque el tier gratuito tiene límites de rate no sostenibles en producción; el valor unitario usado (US$ 0,001/1K tokens) es conservador, por lo que el ROI funciona como piso. **Optimización pendiente**: migrar el Juez a una API con Prompt Caching permitiría cachear la rúbrica (~800 tokens) y el contexto fijo del RAG; con Groq la optimización disponible es deduplicar fragmentos de Pinecone y mantener la rúbrica concisa.

**Control preventivo de API Keys**: llaves separadas por entorno (`GROQ_API_KEY_DEV` con tope de US$ 5/mes, `GROQ_API_KEY_PROD` con US$ 10/mes). El tope bajo en desarrollo neutraliza un bucle lógico accidental: Groq corta las llamadas y el flujo falla con un error controlado en vez de seguir consumiendo tokens.

### Dimensión 2 — Logs estructurados y trazabilidad

El `traceId` se **genera una sola vez** en un nodo Set posterior a `Payload unificado` (`String($now.toMillis()) + '-' + Math.random().toString(36).substring(2,10)`) y viaja sin modificarse por toda la cadena: contexto → Router → Workers → convergencia → sub-workflow del Juez, hasta grabarse en Airtable tanto en `Veredictos_Juez` como en la tabla de conversaciones.

Esquema de log máquina-procesable:

```json
{
  "traceId": "1791903709079-71e1hp67",
  "timestamp": "2026-10-13T12:03:00-03:00",
  "nodeName": "Juez_Renovadas",
  "toolOutput": {
    "errores": ["..."],
    "justificacion": "...",
    "exactitud_factual": 3,
    "estado_veredicto": "ESCALAR"
  },
  "systemPromptSnapshot": "Sos el Juez de calidad de Renovadas...",
  "confidenceScore": 3,
  "session_id": "021ec...",
  "regla_dura": "el borrador promete dinero al cliente"
}
```

`systemPromptSnapshot` (truncado) permite auditar **qué versión de la rúbrica** se usó en cada veredicto. Con el `traceId`, un auditor filtra en Airtable por un ID y ve en una sola búsqueda qué preguntó el cliente, qué respondió el Worker, qué nota le puso el Juez, qué errores detectó, si se aplicó una regla dura y si se escaló a Slack.

### Dimensión 3 — Riesgos, ética y compliance

| # | Riesgo | Control configurado | Dónde |
|---|---|---|---|
| 1 | Alucinación de datos | Criterio de exactitud factual (-2) en la rúbrica; nota 1-2 o RECHAZADO → ruta humana | `Juez_Renovadas` + `Umbrales` + Switch |
| 2 | Promesa financiera no autorizada | Regex de regla dura que fuerza ESCALAR independientemente de la nota | `Umbrales` (`prometeDinero`) — evidencia: P3 escalada con 5/5 |
| 3 | Prompt injection | Instrucciones de rol explícitas, temperatura 0, Structured Output Parser; el Juez recibe contexto y respuesta como **datos**, no como instrucciones | System Message del Juez |
| 4 | Sesgo algorítmico | Los 4 criterios se aplican a toda consulta; el Juez no tiene información del cliente; umbrales fijos en código | Rúbrica + `Umbrales` |
| 5 | Bucle lógico infinito | Máximo 1 reintento (`intento >= 1` → ESCALAR) + topes de gasto en Groq Console | `Umbrales` + Groq Console |

**Gobernanza de datos por proveedor**: Groq (no retiene datos para inferencia, DPA prohíbe entrenar con datos de clientes, **activar Zero Data Retention** en Console → Data Controls) · Pinecone (solo vectores y metadatos del usuario; revisar que no se indexe información sensible) · Airtable (datos propiedad del usuario; **no habilitar Airtable AI** en tablas con datos de clientes) · n8n (self-hosted en Easypanel; los datos del flujo no salen del servidor propio).

Para un despliegue real con datos de clientes, Legal debería revisar el DPA de Groq contra la normativa aplicable (ej. Ley 25.326 de Protección de Datos Personales en Argentina) y Sistemas verificar que ZDR está activo y que las llaves de producción están bajo la cuenta corporativa.

## ⚙️ Cómo importar el flujo

1. En n8n, ir a **Workflows → Import from File**
2. Seleccionar el `.json` correspondiente al checkpoint o Worker deseado
3. Configurar las credenciales propias:
   - **Notion** (catálogo y pedidos) y **Airtable** (memoria + `Veredictos_Juez`) — API token
   - **Gmail** — OAuth2 (trigger y creación de borradores)
   - **HubSpot** — Private App Token
   - **Slack** — API token (incluye los nodos *Send and Wait* del HITL)
   - **Groq** — API key (LLMs y Whisper)
   - **Pinecone** + **Cohere** — API keys (base vectorial y embeddings)
   - **LlamaCloud / LlamaParse** — API key (nodos HTTP de ingesta documental)
   - **Telegram** (bot token) y **ElevenLabs** — solo para la capa de voz
4. Ajustar los IDs de canal de Slack, las bases/tablas de Notion y Airtable y el nombre del índice de Pinecone según el entorno propio
5. **M2**: importar primero los 4 Workers, luego el Manager, y verificar que cada nodo `Execute Workflow` del Manager apunte al Worker correspondiente ya importado
6. **M4**: verificar que el Gmail Trigger apunte a la cuenta correcta y que el canal de Slack exista
7. **M5**: crear el índice de Pinecone con **1024 dimensiones** (coincide con `embed-multilingual-v3.0`) y correr el formulario de ingesta con el Manual antes de probar la rama RAG
8. **M6**: en Workflow Settings dejar `Save failed / successful / manual executions = Do not save` para que los binarios de voz no se persistan
9. **M8**: importar el sub-workflow `Juez_Renovadas` antes del flujo principal y apuntar el nodo `Call 'Juez_Renovadas'` a él; crear la base `Veredictos_Juez` en Airtable con los campos de auditoría (incluida la columna `traceId` del M9)
