# Renovadas — Agente de Atención al Cliente (Proyecto Integrador)

Proyecto integrador del curso de Configuración e Interfaces Agénticas. Un solo flujo de n8n que se amplía módulo a módulo, desde el agente base (M1) hasta el Proyecto Final Integrador (M11).

## 🧵 Contexto

**Renovadas** es una tienda online de lentes Ray-Ban que opera bajo modalidad dropshipping, dependiendo de un excel compartido de proveedores para stock y precios. Este proyecto integrador busca resolver dos problemas reales del negocio:

1. **Back-office**: sincronizar el catálogo (stock/precios) que cambia constantemente por depender de proveedores externos — desarrollado en un integrador previo.
2. **Atención al cliente**: un agente conversacional que responda consultas de catálogo, gestione pedidos y derive a revisión humana los casos sensibles, usando siempre datos actualizados de Notion como fuente de verdad.

## 🗺️ Hoja de ruta del proyecto

| Módulo | Contenido | Estado |
|---|---|---|
| M1 | Agente base: Trigger + AI Agent + System Prompt + Tools + Log de observabilidad | ✅ |
| M2 | Multi-agente: Manager + Workers como sub-workflows | ⏳ |
| M3 | Memoria: contexto de conversación por Session_ID | ⏳ |
| M4 | Integraciones reales: CRM / Calendario / Workspace vía OAuth2 | ⏳ |
| M5 | RAG / base documental (Vector store) | ⏳ |
| M6 | Voz: STT / TTS | ⏳ |
| ... | ... hasta el Proyecto Final Integrador | ⏳ |

**Regla de oro del curso:** no se rehace el flujo desde cero en cada checkpoint — cada módulo parte del `.json` del anterior y lo extiende.

## 📁 Estructura del repositorio

```
├── README.md
├── checkpoint1_ingrid_ledesma.json   ← M1: Agente base
```
A medida que avance el curso, se van a ir sumando los archivos `checkpointN_ingrid_ledesma.json` correspondientes a cada módulo, manteniendo el historial completo de la evolución del proyecto.

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

## ⚙️ Cómo importar el flujo

1. En n8n, ir a **Workflows → Import from File**
2. Seleccionar el `.json` correspondiente al checkpoint deseado
3. Configurar las credenciales propias de Notion, Slack y Gmail (OAuth2/API key)
4. Ajustar los IDs de canal de Slack y de base de datos de Notion según el entorno propio
