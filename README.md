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
| M3 | Memoria: contexto de conversación por Session_ID | ⏳ |
| M4 | Integraciones reales: CRM / Calendario / Workspace vía OAuth2 | ⏳ |
| M5 | RAG / base documental (Vector store) | ⏳ |
| M6 | Voz: STT / TTS | ⏳ |
| ... | ... hasta el Proyecto Final Integrador | ⏳ |

**Regla de oro del curso:** no se rehace el flujo desde cero en cada checkpoint — cada módulo parte del `.json` del anterior y lo extiende.

## 📁 Estructura del repositorio

```
├── README.md
├── checkpoint1_ingrid_ledesma.json        ← M1: Agente base
└── entregable2/
    ├── preentrega_modulo2_ledesma_ingrid.pdf   ← Entregable oficial de M2
    ├── Manager_Renovadas.json
    ├── Worker_Catalogo_Renovadas.json
    ├── Worker_Tracking_Renovadas.json
    ├── Worker_Pedidos_Renovadas.json
    └── Worker_Escalamiento_Renovadas.json
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

## ⚙️ Cómo importar el flujo

1. En n8n, ir a **Workflows → Import from File**
2. Seleccionar el `.json` correspondiente al checkpoint o Worker deseado
3. Configurar las credenciales propias de Notion, Slack y Gmail (OAuth2/API key)
4. Ajustar los IDs de canal de Slack y de base de datos de Notion según el entorno propio
5. Para M2: importar primero los 4 Workers, luego el Manager, y verificar que cada nodo `Execute Workflow` del Manager apunte al Worker correspondiente ya importado
