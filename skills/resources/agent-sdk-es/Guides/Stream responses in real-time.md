# Stream responses in real-time

Obtén respuestas en tiempo real del Agent SDK a medida que el texto y las llamadas a herramientas se generan de forma incremental

---

Por defecto, el Agent SDK entrega objetos `AssistantMessage` completos una vez que Claude termina de generar cada respuesta. Para recibir actualizaciones incrementales a medida que se generan el texto y las llamadas a herramientas, activa el streaming de mensajes parciales configurando `include_partial_messages` (Python) o `includePartialMessages` (TypeScript) en `true` dentro de tus opciones.

> **Consejo:** Esta página cubre el streaming de salida (recibir tokens en tiempo real). Para los modos de entrada (cómo envías mensajes), consulta [Send messages to agents](/docs/en/agent-sdk/streaming-vs-single-mode). También puedes [hacer streaming de respuestas usando el Agent SDK a través de la CLI](https://code.claude.com/docs/en/headless).

## Activar el streaming de salida

Para activar el streaming, configura `include_partial_messages` (Python) o `includePartialMessages` (TypeScript) en `true` dentro de tus opciones. Esto hace que el SDK entregue mensajes `StreamEvent` que contienen eventos crudos de la API a medida que llegan, además de los habituales `AssistantMessage` y `ResultMessage`.

Tu código entonces necesita:
1. Revisar el tipo de cada mensaje para distinguir `StreamEvent` de los demás tipos de mensaje
2. Para `StreamEvent`, extraer el campo `event` y verificar su `type`
3. Buscar eventos `content_block_delta` donde `delta.type` sea `text_delta`, que contienen los fragmentos de texto reales

El siguiente ejemplo activa el streaming e imprime los fragmentos de texto a medida que llegan. Observa las verificaciones de tipo anidadas: primero para `StreamEvent`, luego para `content_block_delta`, y finalmente para `text_delta`:

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions
from claude_agent_sdk.types import StreamEvent
import asyncio


async def stream_response():
    options = ClaudeAgentOptions(
        include_partial_messages=True,
        allowed_tools=["Bash", "Read"],
    )

    async for message in query(prompt="List the files in my project", options=options):
        if isinstance(message, StreamEvent):
            event = message.event
            if event.get("type") == "content_block_delta":
                delta = event.get("delta", {})
                if delta.get("type") == "text_delta":
                    print(delta.get("text", ""), end="", flush=True)


asyncio.run(stream_response())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "List the files in my project",
  options: {
    includePartialMessages: true,
    allowedTools: ["Bash", "Read"]
  }
})) {
  if (message.type === "stream_event") {
    const event = message.event;
    if (event.type === "content_block_delta") {
      if (event.delta.type === "text_delta") {
        process.stdout.write(event.delta.text);
      }
    }
  }
}
```

## Referencia de StreamEvent

Cuando los mensajes parciales están activados, recibes eventos crudos de streaming de la API de Claude envueltos en un objeto. El tipo tiene nombres diferentes en cada SDK:

- **Python**: `StreamEvent` (importar desde `claude_agent_sdk.types`)
- **TypeScript**: `SDKPartialAssistantMessage` con `type: 'stream_event'`

Ambos contienen eventos crudos de la API de Claude, no texto acumulado. Tú mismo debes extraer y acumular los deltas de texto. A continuación se muestra la estructura de cada tipo:

**Python**
```python
@dataclass
class StreamEvent:
    uuid: str  # Unique identifier for this event
    session_id: str  # Session identifier
    event: dict[str, Any]  # The raw Claude API stream event
    parent_tool_use_id: str | None  # Parent tool ID if from a subagent
```

**TypeScript**
```typescript
type SDKPartialAssistantMessage = {
  type: "stream_event";
  event: RawMessageStreamEvent; // From Anthropic SDK
  parent_tool_use_id: string | null;
  uuid: UUID;
  session_id: string;
};
```

El campo `event` contiene el evento de streaming crudo de la [API de Claude](/docs/en/build-with-claude/streaming#event-types). Los tipos de evento más comunes son:

| Tipo de evento | Descripción |
|:---------------|:------------|
| `message_start` | Inicio de un nuevo mensaje |
| `content_block_start` | Inicio de un nuevo bloque de contenido (texto o uso de herramienta) |
| `content_block_delta` | Actualización incremental del contenido |
| `content_block_stop` | Fin de un bloque de contenido |
| `message_delta` | Actualizaciones a nivel de mensaje (motivo de detención, uso) |
| `message_stop` | Fin del mensaje |

## Flujo de mensajes

Con los mensajes parciales activados, recibes los mensajes en este orden:

```text
StreamEvent (message_start)
StreamEvent (content_block_start) - text block
StreamEvent (content_block_delta) - text chunks...
StreamEvent (content_block_stop)
StreamEvent (content_block_start) - tool_use block
StreamEvent (content_block_delta) - tool input chunks...
StreamEvent (content_block_stop)
StreamEvent (message_delta)
StreamEvent (message_stop)
AssistantMessage - complete message with all content
... tool executes ...
... more streaming events for next turn ...
ResultMessage - final result
```

Sin los mensajes parciales activados (`include_partial_messages` en Python, `includePartialMessages` en TypeScript), recibes todos los tipos de mensaje excepto `StreamEvent`. Los tipos más comunes incluyen `SystemMessage` (inicialización de sesión), `AssistantMessage` (respuestas completas), `ResultMessage` (resultado final) y `CompactBoundaryMessage` (indica cuándo se compactó el historial de conversación).

## Streaming de respuestas de texto

Para mostrar el texto a medida que se genera, busca eventos `content_block_delta` donde `delta.type` sea `text_delta`. Estos contienen los fragmentos de texto incrementales. El siguiente ejemplo imprime cada fragmento a medida que llega:

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions
from claude_agent_sdk.types import StreamEvent
import asyncio


async def stream_text():
    options = ClaudeAgentOptions(include_partial_messages=True)

    async for message in query(prompt="Explain how databases work", options=options):
        if isinstance(message, StreamEvent):
            event = message.event
            if event.get("type") == "content_block_delta":
                delta = event.get("delta", {})
                if delta.get("type") == "text_delta":
                    # Print each text chunk as it arrives
                    print(delta.get("text", ""), end="", flush=True)

    print()  # Final newline


asyncio.run(stream_text())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Explain how databases work",
  options: { includePartialMessages: true }
})) {
  if (message.type === "stream_event") {
    const event = message.event;
    if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
      process.stdout.write(event.delta.text);
    }
  }
}

console.log(); // Final newline
```

## Streaming de llamadas a herramientas

Las llamadas a herramientas también se transmiten de forma incremental. Puedes detectar cuándo comienzan las herramientas, recibir su entrada a medida que se genera y ver cuándo finalizan. El siguiente ejemplo rastrea la herramienta que se está ejecutando actualmente y acumula la entrada JSON a medida que llega por streaming. Utiliza tres tipos de eventos:

- `content_block_start`: la herramienta comienza
- `content_block_delta` con `input_json_delta`: llegan fragmentos de entrada
- `content_block_stop`: la llamada a la herramienta se completa

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions
from claude_agent_sdk.types import StreamEvent
import asyncio


async def stream_tool_calls():
    options = ClaudeAgentOptions(
        include_partial_messages=True,
        allowed_tools=["Read", "Bash"],
    )

    # Track the current tool and accumulate its input JSON
    current_tool = None
    tool_input = ""

    async for message in query(prompt="Read the README.md file", options=options):
        if isinstance(message, StreamEvent):
            event = message.event
            event_type = event.get("type")

            if event_type == "content_block_start":
                # New tool call is starting
                content_block = event.get("content_block", {})
                if content_block.get("type") == "tool_use":
                    current_tool = content_block.get("name")
                    tool_input = ""
                    print(f"Starting tool: {current_tool}")

            elif event_type == "content_block_delta":
                delta = event.get("delta", {})
                if delta.get("type") == "input_json_delta":
                    # Accumulate JSON input as it streams in
                    chunk = delta.get("partial_json", "")
                    tool_input += chunk
                    print(f"  Input chunk: {chunk}")

            elif event_type == "content_block_stop":
                # Tool call complete - show final input
                if current_tool:
                    print(f"Tool {current_tool} called with: {tool_input}")
                    current_tool = None


asyncio.run(stream_tool_calls())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Track the current tool and accumulate its input JSON
let currentTool: string | null = null;
let toolInput = "";

for await (const message of query({
  prompt: "Read the README.md file",
  options: {
    includePartialMessages: true,
    allowedTools: ["Read", "Bash"]
  }
})) {
  if (message.type === "stream_event") {
    const event = message.event;

    if (event.type === "content_block_start") {
      // New tool call is starting
      if (event.content_block.type === "tool_use") {
        currentTool = event.content_block.name;
        toolInput = "";
        console.log(`Starting tool: ${currentTool}`);
      }
    } else if (event.type === "content_block_delta") {
      if (event.delta.type === "input_json_delta") {
        // Accumulate JSON input as it streams in
        const chunk = event.delta.partial_json;
        toolInput += chunk;
        console.log(`  Input chunk: ${chunk}`);
      }
    } else if (event.type === "content_block_stop") {
      // Tool call complete - show final input
      if (currentTool) {
        console.log(`Tool ${currentTool} called with: ${toolInput}`);
        currentTool = null;
      }
    }
  }
}
```

## Construir una UI con streaming

Este ejemplo combina el streaming de texto y herramientas en una interfaz de usuario coherente. Rastrea si el agente está ejecutando una herramienta en ese momento (usando una bandera `in_tool`) para mostrar indicadores de estado como `[Using Read...]` mientras las herramientas se ejecutan. El texto fluye normalmente cuando no hay una herramienta en curso, y la finalización de una herramienta activa un mensaje de "done". Este patrón es útil para interfaces de chat que necesitan mostrar el progreso durante tareas de agente de varios pasos.

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage
from claude_agent_sdk.types import StreamEvent
import asyncio
import sys


async def streaming_ui():
    options = ClaudeAgentOptions(
        include_partial_messages=True,
        allowed_tools=["Read", "Bash", "Grep"],
    )

    # Track whether we're currently in a tool call
    in_tool = False

    async for message in query(
        prompt="Find all TODO comments in the codebase", options=options
    ):
        if isinstance(message, StreamEvent):
            event = message.event
            event_type = event.get("type")

            if event_type == "content_block_start":
                content_block = event.get("content_block", {})
                if content_block.get("type") == "tool_use":
                    # Tool call is starting - show status indicator
                    tool_name = content_block.get("name")
                    print(f"\n[Using {tool_name}...]", end="", flush=True)
                    in_tool = True

            elif event_type == "content_block_delta":
                delta = event.get("delta", {})
                # Only stream text when not executing a tool
                if delta.get("type") == "text_delta" and not in_tool:
                    sys.stdout.write(delta.get("text", ""))
                    sys.stdout.flush()

            elif event_type == "content_block_stop":
                if in_tool:
                    # Tool call finished
                    print(" done", flush=True)
                    in_tool = False

        elif isinstance(message, ResultMessage):
            # Agent finished all work
            print(f"\n\n--- Complete ---")


asyncio.run(streaming_ui())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Track whether we're currently in a tool call
let inTool = false;

for await (const message of query({
  prompt: "Find all TODO comments in the codebase",
  options: {
    includePartialMessages: true,
    allowedTools: ["Read", "Bash", "Grep"]
  }
})) {
  if (message.type === "stream_event") {
    const event = message.event;

    if (event.type === "content_block_start") {
      if (event.content_block.type === "tool_use") {
        // Tool call is starting - show status indicator
        process.stdout.write(`\n[Using ${event.content_block.name}...]`);
        inTool = true;
      }
    } else if (event.type === "content_block_delta") {
      // Only stream text when not executing a tool
      if (event.delta.type === "text_delta" && !inTool) {
        process.stdout.write(event.delta.text);
      }
    } else if (event.type === "content_block_stop") {
      if (inTool) {
        // Tool call finished
        console.log(" done");
        inTool = false;
      }
    }
  } else if (message.type === "result") {
    // Agent finished all work
    console.log("\n\n--- Complete ---");
  }
}
```

## Limitaciones conocidas

Algunas funcionalidades del SDK son incompatibles con el streaming:

- **Extended thinking**: cuando configuras explícitamente `max_thinking_tokens` (Python) o `maxThinkingTokens` (TypeScript), los mensajes `StreamEvent` no se emiten. Solo recibirás mensajes completos después de cada turno. Ten en cuenta que el thinking está desactivado por defecto en el SDK, por lo que el streaming funciona a menos que lo actives.
- **Structured output**: el resultado JSON aparece únicamente en el `ResultMessage.structured_output` final, no como deltas de streaming. Consulta [structured outputs](/docs/en/agent-sdk/structured-outputs) para más detalles.

## Próximos pasos

Ahora que puedes hacer streaming de texto y llamadas a herramientas en tiempo real, explora estos temas relacionados:

- [Interactive vs one-shot queries](/docs/en/agent-sdk/streaming-vs-single-mode): elige entre modos de entrada según tu caso de uso
- [Structured outputs](/docs/en/agent-sdk/structured-outputs): obtén respuestas JSON tipadas del agente
- [Permissions](/docs/en/agent-sdk/permissions): controla qué herramientas puede usar el agente
