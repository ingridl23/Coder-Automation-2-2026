# Renovadas — Agente de Atención al Cliente (Proyecto Integrador)

Proyecto integrador del curso de Configuración e Interfaces Agénticas. Un solo flujo de n8n que se amplía módulo a módulo, desde el agente base (M1) hasta el Proyecto Final Integrador (M11).

## 🧵 Contexto

**Renovadas** es una tienda online de lentes Ray-Ban que opera bajo modalidad dropshipping, dependiendo de un excel compartido de proveedores para stock y precios. Este proyecto integrador busca resolver dos problemas reales del negocio:

1. **Back-office**: sincronizar el catálogo (stock/precios) que cambia constantemente por depender de proveedores externos — desarrollado en un integrador previo.
2. **Atención al cliente**: un agente conversacional que responda consultas de catálogo, gestione pedidos y derive a revisión humana los casos sensibles, usando siempre datos actualizados de Notion como fuente de verdad.

Desde M2, el agente tiene nombre y personalidad propia: **Rena**, la asistente virtual de Renovadas.

## 🗺️ Hoja de ruta del proyecto

| Módulo | Contenido | Estado |
|---|---|---|
| M1 | Agente base: Trigger + AI Agent + System Prompt + Tools + Log de observabilidad | ✅ |
| M2 | Multi-agente: Manager + Workers como sub-workflows | ✅ |
| M3 | Memoria: contexto de conversación por Session_ID | ✅ |
| M4 | Integraciones reales: Gmail Trigger + CRM (HubSpot) + Slack + HITL | ✅ |
| M5 | RAG / base documental (Vector store) | ⏳ |
| M6 | Voz: STT / TTS | ⏳ |
| ... | ... hasta el Proyecto Final Integrador | ⏳ |

**Regla de oro del curso:** no se rehace el flujo desde cero en cada checkpoint — cada módulo parte del `.json` del anterior y lo extiende.

## 📁 Estructura del repositorio

```
├── README.md
├── checkpoint1_ingrid_ledesma.json        ← M1: Agente base
├── checkpoint4_ingrid_ledesma.json        ← M4: Integraciones externas
├── entregable2/
│   ├── preentrega_modulo2_ledesma_ingrid.pdf   ← Entregable oficial de M2
│   ├── Manager_Renovadas.json
│   ├── Worker_Catalogo_Renovadas.json
│   ├── Worker_Tracking_Renovadas.json
│   ├── Worker_Pedidos_Renovadas.json
│   └── Worker_Escalamiento_Renovadas.json
```
A medida que avance el curso, se van a ir sumando carpetas `entregableN/` correspondientes a cada módulo, manteniendo el historial completo de la evolución del proyecto.

## 🤖 M1 — Agente Base y Motor de Razonamiento

### Arquitectura del flujo

- **Trigger**: Chat Trigger (`When chat message received`)
- **AI Agent**: modo Tools Agent, modelo Groq (Llama/Qwen con soporte de tool-calling), Max Iterations = 6
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

- **Manager** (`Manager_Renovadas.json`):
  - **Trigger**: Chat Trigger
  - **Router AI Agent** ("Rena"): clasifica cada mensaje entrante en `intent` (catalogo / tracking / pedidos / saludo), `confidence` y `risk` (LOW/HIGH), devolviendo un JSON estructurado.
  - **Code**: parsea la salida del Router a JSON, con manejo de errores.
  - **IF (risk)**: si `risk = HIGH`, deriva directo al Worker de Escalamiento, saltando el Switch.
  - **Switch (intent)**: para los casos de riesgo bajo, enruta según `intent` al Worker correspondiente.
  - **Execute Workflow** (uno por Worker): invoca al sub-workflow correspondiente con **"Wait For Sub-Workflow Completion" activado**, para esperar la respuesta antes de continuar.
  - **Edit Fields**: unifica la respuesta de cualquier Worker en un único formato para el chat.

- **Workers** (sub-workflows independientes, cada uno con su propio `Execute Workflow Trigger`):
  - `Worker_Catalogo_Renovadas.json` — consulta disponibilidad, precio y características en Notion.
  - `Worker_Tracking_Renovadas.json` — consulta el estado de un pedido y envía el código de seguimiento por Gmail.
  - `Worker_Pedidos_Renovadas.json` — registra un pedido nuevo en Notion.
  - `Worker_Escalamiento_Renovadas.json` — deriva el caso a revisión humana vía Slack (Human in the loop) y registra el resultado.

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

El entregable oficial de M2 es `entregable2/preentrega_modulo2_ledesma_ingrid.pdf`, con capturas del canvas del Manager, de al menos 2 Workers, de la configuración del Execute Workflow (payload + toggle de espera activado), de una ejecución exitosa end-to-end, y del log de observabilidad en Slack. Los `.json` de Manager y Workers se incluyen como respaldo adicional.

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

El archivo `checkpoint4_ingrid_ledesma.json` contiene el workflow completo exportado desde n8n, incluyendo los 4 nodos de rúbrica y todas las integraciones.

## ⚙️ Cómo importar el flujo

1. En n8n, ir a **Workflows → Import from File**
2. Seleccionar el `.json` correspondiente al checkpoint o Worker deseado
3. Configurar las credenciales propias de Notion, Slack, Gmail (OAuth2), HubSpot (Private App Token) y Airtable (API key)
4. Ajustar los IDs de canal de Slack y de base de datos de Notion/Airtable según el entorno propio
5. Para M2: importar primero los 4 Workers, luego el Manager, y verificar que cada nodo `Execute Workflow` del Manager apunte al Worker correspondiente ya importado
6. Para M4: verificar que el Gmail Trigger apunte a la cuenta correcta y que el canal de Slack exista
