# Rastrear costo y uso

Aprende a rastrear el uso de tokens, desduplicar llamadas a herramientas en paralelo y calcular costos con el Claude Agent SDK.

---

El Claude Agent SDK proporciona información detallada sobre el uso de tokens en cada interacción con Claude. Esta guía explica cómo rastrear correctamente los costos y comprender los reportes de uso, especialmente cuando se trabaja con llamadas a herramientas en paralelo y conversaciones de múltiples pasos.

Para la documentación completa de la API, consulta la [referencia del TypeScript SDK](/docs/en/agent-sdk/typescript) y la [referencia del Python SDK](/docs/en/agent-sdk/python).

## Entender el uso de tokens

Los SDKs de TypeScript y Python exponen datos de uso con distintos niveles de detalle:

- **TypeScript** proporciona desgloses de tokens por paso en cada mensaje del asistente (`message.message.id`, `message.message.usage`), el costo por modelo a través de `modelUsage`, y un total acumulado en el mensaje de resultado.
- **Python** proporciona el total acumulado en el mensaje de resultado (`total_cost_usd` y el diccionario `usage`). Los desgloses por paso no están disponibles en los mensajes individuales del asistente.

Ambos SDKs utilizan el mismo modelo de costos subyacente. La diferencia radica en cuánta granularidad expone cada SDK.

El rastreo de costos depende de entender cómo el SDK delimita los datos de uso:

- **Llamada a `query()`:** una invocación de la función `query()` del SDK. Una sola llamada puede involucrar múltiples pasos (Claude responde, usa herramientas, obtiene resultados, responde nuevamente). Cada llamada produce un mensaje [`result`](/docs/en/agent-sdk/typescript#sdk-result-message) al final.
- **Paso:** un ciclo único de solicitud/respuesta dentro de una llamada a `query()`. En TypeScript, cada paso produce mensajes del asistente con uso de tokens.
- **Sesión:** una serie de llamadas a `query()` vinculadas por un ID de sesión (usando la opción `resume`). Cada llamada a `query()` dentro de una sesión reporta su propio costo de forma independiente.

El siguiente diagrama muestra el flujo de mensajes de una sola llamada a `query()`, con el uso de tokens reportado en cada paso y el total autorizado al final:

![Diagrama que muestra una consulta que produce dos pasos de mensajes. El Paso 1 tiene cuatro mensajes del asistente que comparten el mismo ID y uso (contar una sola vez), el Paso 2 tiene un mensaje del asistente con un nuevo ID, y el mensaje de resultado final muestra total_cost_usd para la facturación.](/docs/images/agent-sdk/message-usage-flow.svg)

### 1. Cada paso produce mensajes del asistente

Cuando Claude responde, envía uno o más mensajes del asistente. En TypeScript, cada mensaje del asistente contiene un `BetaMessage` anidado (accesible mediante `message.message`) con un `id` y un objeto [`usage`](/docs/en/api/messages) con conteos de tokens (`input_tokens`, `output_tokens`). Cuando Claude usa múltiples herramientas en un turno, todos los mensajes de ese turno comparten el mismo `id`, por lo que se debe desduplicar por ID para evitar el doble conteo. En Python, el uso por paso no está disponible en los mensajes individuales.

### 2. El mensaje de resultado proporciona el total autorizado

Cuando la llamada a `query()` se completa, el SDK emite un mensaje de resultado con `total_cost_usd` y el `usage` acumulado. Esto está disponible tanto en TypeScript ([`SDKResultMessage`](/docs/en/agent-sdk/typescript#sdk-result-message)) como en Python ([`ResultMessage`](/docs/en/agent-sdk/python#result-message)). Si realizas múltiples llamadas a `query()` (por ejemplo, en una sesión de múltiples turnos), cada resultado solo refleja el costo de esa llamada individual. Si solo necesitas el costo total, puedes ignorar el uso por paso y leer este único valor.

## Obtener el costo total de una consulta

El mensaje de resultado ([TypeScript](/docs/en/agent-sdk/typescript#sdk-result-message), [Python](/docs/en/agent-sdk/python#result-message)) es el último mensaje en cada llamada a `query()`. Incluye `total_cost_usd`, el costo acumulado a lo largo de todos los pasos de esa llamada. Esto funciona tanto para resultados exitosos como para resultados con error. Si usas sesiones para realizar múltiples llamadas a `query()`, cada resultado solo refleja el costo de esa llamada individual.

Los siguientes ejemplos iteran sobre el flujo de mensajes de una llamada a `query()` e imprimen el costo total cuando llega el mensaje `result`:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({ prompt: "Summarize this project" })) {
  if (message.type === "result") {
    console.log(`Total cost: $${message.total_cost_usd}`);
  }
}
```

**Python**
```python
from claude_agent_sdk import query, ResultMessage
import asyncio


async def main():
    async for message in query(prompt="Summarize this project"):
        if isinstance(message, ResultMessage):
            print(f"Total cost: ${message.total_cost_usd or 0}")


asyncio.run(main())
```

## Rastrear uso detallado en TypeScript

El TypeScript SDK expone granularidad de uso adicional que no está disponible en Python. El `AssistantMessage` del Python SDK no expone el uso de tokens por paso ni los desgloses por modelo. Usa [`ResultMessage.usage`](/docs/en/agent-sdk/python#result-message) para los totales acumulados en su lugar.

### Rastrear uso por paso

Cada mensaje del asistente contiene un `BetaMessage` anidado (accesible mediante `message.message`) con un `id` y un objeto `usage` con conteos de tokens. Cuando Claude usa herramientas en paralelo, múltiples mensajes comparten el mismo `id` con datos de uso idénticos. Registra los IDs que ya hayas contado y omite los duplicados para evitar totales inflados.

> **Advertencia:** Las llamadas a herramientas en paralelo producen múltiples mensajes del asistente cuyo `BetaMessage` anidado comparte el mismo `id` y uso idéntico. Siempre desduplicar por ID para obtener conteos de tokens por paso precisos.

El siguiente ejemplo acumula tokens de entrada y salida a lo largo de todos los pasos, contando cada ID de mensaje único solo una vez:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const seenIds = new Set<string>();
let totalInputTokens = 0;
let totalOutputTokens = 0;

for await (const message of query({ prompt: "Summarize this project" })) {
  if (message.type === "assistant") {
    const msgId = message.message.id;

    // Parallel tool calls share the same ID, only count once
    if (!seenIds.has(msgId)) {
      seenIds.add(msgId);
      totalInputTokens += message.message.usage.input_tokens;
      totalOutputTokens += message.message.usage.output_tokens;
    }
  }
}

console.log(`Steps: ${seenIds.size}`);
console.log(`Input tokens: ${totalInputTokens}`);
console.log(`Output tokens: ${totalOutputTokens}`);
```

### Desglosar el uso por modelo

El mensaje de resultado incluye [`modelUsage`](/docs/en/agent-sdk/typescript#model-usage), un mapa de nombre de modelo a conteos de tokens y costo por modelo. Esto es útil cuando ejecutas múltiples modelos (por ejemplo, Haiku para subagentes y Opus para el agente principal) y quieres ver dónde se están consumiendo los tokens.

El siguiente ejemplo ejecuta una consulta e imprime el costo y el desglose de tokens para cada modelo utilizado:

```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({ prompt: "Summarize this project" })) {
  if (message.type !== "result") continue;

  for (const [modelName, usage] of Object.entries(message.modelUsage)) {
    console.log(`${modelName}: $${usage.costUSD.toFixed(4)}`);
    console.log(`  Input tokens: ${usage.inputTokens}`);
    console.log(`  Output tokens: ${usage.outputTokens}`);
    console.log(`  Cache read: ${usage.cacheReadInputTokens}`);
    console.log(`  Cache creation: ${usage.cacheCreationInputTokens}`);
  }
}
```

## Acumular costos a través de múltiples llamadas

Cada llamada a `query()` retorna su propio `total_cost_usd`. El SDK no proporciona un total a nivel de sesión, por lo que si tu aplicación realiza múltiples llamadas a `query()` (por ejemplo, en una sesión de múltiples turnos o entre diferentes usuarios), acumula los totales tú mismo.

Los siguientes ejemplos ejecutan dos llamadas a `query()` secuencialmente, agregan el `total_cost_usd` de cada llamada a un total acumulado, e imprimen tanto el costo por llamada como el costo combinado:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Track cumulative cost across multiple query() calls
let totalSpend = 0;

const prompts = [
  "Read the files in src/ and summarize the architecture",
  "List all exported functions in src/auth.ts"
];

for (const prompt of prompts) {
  for await (const message of query({ prompt })) {
    if (message.type === "result") {
      totalSpend += message.total_cost_usd ?? 0;
      console.log(`This call: $${message.total_cost_usd}`);
    }
  }
}

console.log(`Total spend: $${totalSpend.toFixed(4)}`);
```

**Python**
```python
from claude_agent_sdk import query, ResultMessage
import asyncio


async def main():
    # Track cumulative cost across multiple query() calls
    total_spend = 0.0

    prompts = [
        "Read the files in src/ and summarize the architecture",
        "List all exported functions in src/auth.ts",
    ]

    for prompt in prompts:
        async for message in query(prompt=prompt):
            if isinstance(message, ResultMessage):
                cost = message.total_cost_usd or 0
                total_spend += cost
                print(f"This call: ${cost}")

    print(f"Total spend: ${total_spend:.4f}")


asyncio.run(main())
```

## Manejar errores, caché y discrepancias en tokens

Para un rastreo de costos preciso, considera las conversaciones fallidas, los precios de los tokens en caché y las inconsistencias ocasionales en los reportes.

### Resolver discrepancias en tokens de salida

En casos excepcionales, podrías observar valores diferentes de `output_tokens` en mensajes con el mismo ID. Cuando esto ocurra:

1. **Usa el valor más alto:** el mensaje final de un grupo típicamente contiene el total preciso.
2. **Verifica contra el costo total:** el `total_cost_usd` en el mensaje de resultado es el valor autorizado.
3. **Reporta inconsistencias:** abre issues en el [repositorio de GitHub de Claude Code](https://github.com/anthropics/claude-code/issues).

### Rastrear costos en conversaciones fallidas

Tanto los mensajes de resultado exitosos como los de error incluyen `usage` y `total_cost_usd`. Si una conversación falla a mitad de camino, igual consumiste tokens hasta el punto del fallo. Siempre lee los datos de costo del mensaje de resultado independientemente de su `subtype`.

### Rastrear tokens en caché

El Agent SDK utiliza automáticamente el [almacenamiento en caché de prompts](/docs/en/build-with-claude/prompt-caching) para reducir costos en contenido repetido. No necesitas configurar el almacenamiento en caché tú mismo. El objeto de uso incluye dos campos adicionales para el rastreo de caché:

- `cache_creation_input_tokens`: tokens usados para crear nuevas entradas en caché (cobrados a una tarifa más alta que los tokens de entrada estándar).
- `cache_read_input_tokens`: tokens leídos de entradas de caché existentes (cobrados a una tarifa reducida).

Rastrénlos por separado de `input_tokens` para entender los ahorros del almacenamiento en caché. En TypeScript, estos campos están tipados en el objeto [`Usage`](/docs/en/agent-sdk/typescript#usage). En Python, aparecen como claves en el diccionario [`ResultMessage.usage`](/docs/en/agent-sdk/python#result-message) (por ejemplo, `message.usage.get("cache_read_input_tokens", 0)`).

## Documentación relacionada

- [Referencia del TypeScript SDK](/docs/en/agent-sdk/typescript) - Documentación completa de la API
- [Descripción general del SDK](/docs/en/agent-sdk/overview) - Comenzar con el SDK
- [Permisos del SDK](/docs/en/agent-sdk/permissions) - Gestionar permisos de herramientas
