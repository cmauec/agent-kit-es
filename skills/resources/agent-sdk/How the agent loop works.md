# Cómo funciona el bucle del agente

Comprende el ciclo de vida de los mensajes, la ejecución de herramientas, la ventana de contexto y la arquitectura que impulsan tus agentes del SDK.

---

El Agent SDK te permite integrar el bucle de agente autónomo de Claude Code en tus propias aplicaciones. El SDK es un paquete independiente que te otorga control programático sobre herramientas, permisos, límites de costo y salida. No necesitas tener instalado el CLI de Claude Code para usarlo.

Cuando inicias un agente, el SDK ejecuta el mismo [bucle de ejecución que impulsa Claude Code](https://code.claude.com/docs/en/how-claude-code-works#the-agentic-loop): Claude evalúa tu prompt, llama a herramientas para tomar acción, recibe los resultados y repite el proceso hasta que la tarea esté completa. Esta página explica qué sucede dentro de ese bucle para que puedas construir, depurar y optimizar tus agentes de manera efectiva.

## El bucle en resumen

Cada sesión de agente sigue el mismo ciclo:

![Bucle del agente: el prompt entra, Claude evalúa, se bifurca hacia llamadas a herramientas o respuesta final](/docs/images/agent-loop-diagram.svg)

1. **Recibir el prompt.** Claude recibe tu prompt, junto con el system prompt, las definiciones de herramientas y el historial de conversación. El SDK emite un [`SystemMessage`](#tipos-de-mensajes) con subtipo `"init"` que contiene los metadatos de la sesión.
2. **Evaluar y responder.** Claude evalúa el estado actual y determina cómo proceder. Puede responder con texto, solicitar una o más llamadas a herramientas, o ambas cosas. El SDK emite un [`AssistantMessage`](#tipos-de-mensajes) que contiene el texto y cualquier solicitud de llamada a herramientas.
3. **Ejecutar herramientas.** El SDK ejecuta cada herramienta solicitada y recopila los resultados. Cada conjunto de resultados de herramientas se devuelve a Claude para la siguiente decisión. Puedes usar [hooks](./Guides/Intercept%20and%20control%20agent%20behavior%20with%20hooks.md) para interceptar, modificar o bloquear llamadas a herramientas antes de que se ejecuten.
4. **Repetir.** Los pasos 2 y 3 se repiten en ciclo. Cada ciclo completo es un turno. Claude continúa llamando herramientas y procesando resultados hasta que produce una respuesta sin llamadas a herramientas.
5. **Devolver el resultado.** El SDK emite un [`AssistantMessage`](#tipos-de-mensajes) final con la respuesta de texto (sin llamadas a herramientas), seguido de un [`ResultMessage`](#tipos-de-mensajes) con el texto final, el uso de tokens, el costo y el ID de sesión.

Una pregunta sencilla ("¿qué archivos hay aquí?") puede tomar uno o dos turnos llamando a `Glob` y respondiendo con los resultados. Una tarea compleja ("refactoriza el módulo de autenticación y actualiza las pruebas") puede encadenar docenas de llamadas a herramientas a lo largo de muchos turnos, leyendo archivos, editando código y ejecutando pruebas, con Claude ajustando su enfoque en función de cada resultado.

## Turnos y mensajes

Un turno es un viaje de ida y vuelta dentro del bucle: Claude produce una salida que incluye llamadas a herramientas, el SDK ejecuta esas herramientas, y los resultados se devuelven a Claude automáticamente. Esto sucede sin devolver el control a tu código. Los turnos continúan hasta que Claude produce una salida sin llamadas a herramientas, momento en el que el bucle termina y se entrega el resultado final.

Considera cómo podría verse una sesión completa para el prompt "Fix the failing tests in auth.ts".

Primero, el SDK envía tu prompt a Claude y emite un [`SystemMessage`](#tipos-de-mensajes) con los metadatos de la sesión. Luego comienza el bucle:

1. **Turno 1:** Claude llama a `Bash` para ejecutar `npm test`. El SDK emite un [`AssistantMessage`](#tipos-de-mensajes) con la llamada a la herramienta, ejecuta el comando y luego emite un [`UserMessage`](#tipos-de-mensajes) con la salida (tres fallos).
2. **Turno 2:** Claude llama a `Read` en `auth.ts` y `auth.test.ts`. El SDK devuelve el contenido de los archivos y emite un `AssistantMessage`.
3. **Turno 3:** Claude llama a `Edit` para corregir `auth.ts`, luego llama a `Bash` para volver a ejecutar `npm test`. Las tres pruebas pasan. El SDK emite un `AssistantMessage`.
4. **Turno final:** Claude produce una respuesta solo de texto sin llamadas a herramientas: "Fixed the auth bug, all three tests pass now." El SDK emite un `AssistantMessage` final con este texto, seguido de un [`ResultMessage`](#tipos-de-mensajes) con el mismo texto más el costo y el uso.

Eso fueron cuatro turnos: tres con llamadas a herramientas y uno de respuesta solo de texto final.

Puedes limitar el bucle con `max_turns` / `maxTurns`, que cuenta solo los turnos con uso de herramientas. Por ejemplo, `max_turns=2` en el bucle anterior habría detenido la ejecución antes del paso de edición. También puedes usar `max_budget_usd` / `maxBudgetUsd` para limitar los turnos en función de un umbral de gasto.

Sin límites, el bucle se ejecuta hasta que Claude termine por sí solo, lo cual está bien para tareas bien definidas, pero puede prolongarse con prompts abiertos ("mejora este codebase"). Establecer un presupuesto es una buena opción predeterminada para agentes en producción. Consulta [Turnos y presupuesto](#turnos-y-presupuesto) a continuación para la referencia de opciones.

## Tipos de mensajes

Mientras se ejecuta el bucle, el SDK emite un flujo de mensajes. Cada mensaje tiene un tipo que indica en qué etapa del bucle se originó. Los cinco tipos principales son:

- **`SystemMessage`:** eventos del ciclo de vida de la sesión. El campo `subtype` los distingue: `"init"` es el primer mensaje (metadatos de la sesión), y `"compact_boundary"` se dispara después de la [compactación](#compactación-automática).
- **`AssistantMessage`:** se emite después de cada respuesta de Claude, incluyendo la final solo de texto. Contiene bloques de contenido de texto y bloques de llamadas a herramientas de ese turno.
- **`UserMessage`:** se emite después de cada ejecución de herramienta con el contenido del resultado que se devuelve a Claude. También se emite para cualquier entrada de usuario que transmitas en medio del bucle.
- **`StreamEvent`:** solo se emite cuando los mensajes parciales están habilitados. Contiene eventos de streaming sin procesar de la API (deltas de texto, fragmentos de entrada de herramientas). Consulta [Stream responses](./Guides/Stream%20responses%20in%20real-time.md).
- **`ResultMessage`:** el último mensaje, siempre. Contiene el resultado de texto final, el uso de tokens, el costo y el ID de sesión. Verifica el campo `subtype` para determinar si la tarea se completó con éxito o alcanzó un límite. Consulta [Manejar el resultado](#manejar-el-resultado).

Estos cinco tipos cubren el ciclo de vida completo del bucle del agente en ambos SDKs. El SDK de TypeScript también emite eventos de observabilidad adicionales (eventos de hooks, progreso de herramientas, límites de velocidad, notificaciones de tareas) que proporcionan detalles adicionales pero no son necesarios para ejecutar el bucle. Consulta la [referencia de tipos de mensajes de Python](/docs/en/agent-sdk/python#message-types) y la [referencia de tipos de mensajes de TypeScript](/docs/en/agent-sdk/typescript#message-types) para las listas completas.

### Manejar mensajes

Qué mensajes manejas depende de lo que estés construyendo:

- **Solo resultados finales:** maneja `ResultMessage` para obtener la salida, el costo y si la tarea se completó con éxito o alcanzó un límite.
- **Actualizaciones de progreso:** maneja `AssistantMessage` para ver qué está haciendo Claude en cada turno, incluyendo qué herramientas llamó.
- **Streaming en vivo:** habilita los mensajes parciales (`include_partial_messages` en Python, `includePartialMessages` en TypeScript) para recibir mensajes `StreamEvent` en tiempo real. Consulta [Stream responses in real-time](./Guides/Stream%20responses%20in%20real-time.md).

La forma de verificar los tipos de mensajes depende del SDK:

- **Python:** verifica los tipos de mensajes con `isinstance()` contra las clases importadas desde `claude_agent_sdk` (por ejemplo, `isinstance(message, ResultMessage)`).
- **TypeScript:** verifica el campo de cadena `type` (por ejemplo, `message.type === "result"`). `AssistantMessage` y `UserMessage` envuelven el mensaje sin procesar de la API en un campo `.message`, por lo que los bloques de contenido están en `message.message.content`, no en `message.content`.

**Python**

```python Python
from claude_agent_sdk import query, AssistantMessage, ResultMessage

async for message in query(prompt="Summarize this project"):
    if isinstance(message, AssistantMessage):
        print(f"Turn completed: {len(message.content)} content blocks")
    if isinstance(message, ResultMessage):
        if message.subtype == "success":
            print(message.result)
        else:
            print(f"Stopped: {message.subtype}")
```

**TypeScript**

```typescript TypeScript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({ prompt: "Summarize this project" })) {
  if (message.type === "assistant") {
    console.log(`Turn completed: ${message.message.content.length} content blocks`);
  }
  if (message.type === "result") {
    if (message.subtype === "success") {
      console.log(message.result);
    } else {
      console.log(`Stopped: ${message.subtype}`);
    }
  }
}
```

## Ejecución de herramientas

Las herramientas le dan a tu agente la capacidad de tomar acción. Sin herramientas, Claude solo puede responder con texto. Con herramientas, Claude puede leer archivos, ejecutar comandos, buscar código e interactuar con servicios externos.

### Herramientas integradas

El SDK incluye las mismas herramientas que impulsan Claude Code:

| Categoría | Herramientas | Qué hacen |
|:---------|:------|:-------------|
| **Operaciones de archivos** | `Read`, `Edit`, `Write` | Leer, modificar y crear archivos |
| **Búsqueda** | `Glob`, `Grep` | Encontrar archivos por patrón, buscar contenido con regex |
| **Ejecución** | `Bash` | Ejecutar comandos de shell, scripts, operaciones de git |
| **Web** | `WebSearch`, `WebFetch` | Buscar en la web, obtener y analizar páginas |
| **Descubrimiento** | `ToolSearch` | Encontrar y cargar herramientas dinámicamente bajo demanda en lugar de precargarlas todas |
| **Orquestación** | `Agent`, `Skill`, `AskUserQuestion`, `TodoWrite` | Crear subagentes, invocar skills, preguntar al usuario, rastrear tareas |

Más allá de las herramientas integradas, puedes:

- **Conectar servicios externos** con [servidores MCP](./Guides/Connect%20to%20external%20tools%20with%20MCP.md) (bases de datos, navegadores, APIs)
- **Definir herramientas personalizadas** con [manejadores de herramientas personalizadas](./Guides/Custom%20Tools.md)
- **Cargar skills de proyecto** mediante [fuentes de configuración](./Core%20concepts/Use%20Claude%20Code%20features%20in%20the%20SDK.md) para flujos de trabajo reutilizables

### Permisos de herramientas

Claude determina qué herramientas llamar según la tarea, pero tú controlas si esas llamadas pueden ejecutarse. Puedes aprobar automáticamente herramientas específicas, bloquear otras completamente, o requerir aprobación para todo. Tres opciones funcionan en conjunto para determinar qué se ejecuta:

- **`allowed_tools` / `allowedTools`** aprueba automáticamente las herramientas listadas. Un agente de solo lectura con `["Read", "Glob", "Grep"]` en su lista de herramientas permitidas ejecuta esas herramientas sin solicitar confirmación. Las herramientas no listadas siguen disponibles pero requieren permiso.
- **`disallowed_tools` / `disallowedTools`** bloquea las herramientas listadas, independientemente de otras configuraciones. Consulta [Permissions](./Guides/Configure%20permissions.md) para ver el orden en que se verifican las reglas antes de que se ejecute una herramienta.
- **`permission_mode` / `permissionMode`** controla qué sucede con las herramientas que no están cubiertas por reglas de permitir o denegar. Consulta [Modo de permiso](#modo-de-permiso) para ver los modos disponibles.

También puedes delimitar herramientas individuales con reglas como `"Bash(npm:*)"` para permitir solo comandos específicos. Consulta [Permissions](./Guides/Configure%20permissions.md) para la sintaxis completa de reglas.

Cuando se deniega una herramienta, Claude recibe un mensaje de rechazo como resultado de la herramienta y normalmente intenta un enfoque diferente o informa que no pudo continuar.

### Ejecución paralela de herramientas

Cuando Claude solicita múltiples llamadas a herramientas en un solo turno, ambos SDKs pueden ejecutarlas de forma concurrente o secuencial según la herramienta. Las herramientas de solo lectura (como `Read`, `Glob`, `Grep` y las herramientas MCP marcadas como de solo lectura) pueden ejecutarse de forma concurrente. Las herramientas que modifican el estado (como `Edit`, `Write` y `Bash`) se ejecutan secuencialmente para evitar conflictos.

Las herramientas personalizadas se ejecutan secuencialmente de forma predeterminada. Para habilitar la ejecución paralela en una herramienta personalizada, márcala como de solo lectura en sus anotaciones: `readOnly` en [TypeScript](/docs/en/agent-sdk/typescript#tool) o `readOnlyHint` en [Python](/docs/en/agent-sdk/python#tool).

## Controlar cómo se ejecuta el bucle

Puedes limitar cuántos turnos toma el bucle, cuánto cuesta, qué tan profundo razona Claude y si las herramientas requieren aprobación antes de ejecutarse. Todos estos son campos en `ClaudeAgentOptions` (Python) / `Options` (TypeScript).

### Turnos y presupuesto

| Opción | Qué controla | Valor predeterminado |
|:-------|:----------------|:--------|
| Turnos máximos (`max_turns` / `maxTurns`) | Máximo de viajes de ida y vuelta con uso de herramientas | Sin límite |
| Presupuesto máximo (`max_budget_usd` / `maxBudgetUsd`) | Costo máximo antes de detenerse | Sin límite |

Cuando se alcanza cualquiera de los límites, el SDK devuelve un `ResultMessage` con un subtipo de error correspondiente (`error_max_turns` o `error_max_budget_usd`). Consulta [Manejar el resultado](#manejar-el-resultado) para saber cómo verificar estos subtipos, y `ClaudeAgentOptions` / `Options` para la sintaxis.

### Nivel de esfuerzo

La opción `effort` controla cuánto razonamiento aplica Claude. Los niveles de esfuerzo más bajos usan menos tokens por turno y reducen el costo. No todos los modelos admiten el parámetro de esfuerzo. Consulta [Effort](/docs/en/build-with-claude/effort) para ver qué modelos lo admiten.

| Nivel | Comportamiento | Ideal para |
|:------|:---------|:---------|
| `"low"` | Razonamiento mínimo, respuestas rápidas | Búsquedas de archivos, listado de directorios |
| `"medium"` | Razonamiento equilibrado | Ediciones rutinarias, tareas estándar |
| `"high"` | Análisis exhaustivo | Refactorizaciones, depuración |
| `"max"` | Profundidad de razonamiento máxima | Problemas de múltiples pasos que requieren análisis profundo |

Si no configuras `effort`, el SDK de Python deja el parámetro sin establecer y delega al comportamiento predeterminado del modelo. El SDK de TypeScript usa `"high"` por defecto.

> **Nota:** `effort` intercambia latencia y costo de tokens por profundidad de razonamiento dentro de cada respuesta. [Extended thinking](/docs/en/build-with-claude/extended-thinking) es una característica separada que produce bloques de cadena de pensamiento visibles en la salida. Son independientes: puedes establecer `effort: "low"` con extended thinking habilitado, o `effort: "max"` sin él.

Usa un esfuerzo más bajo para agentes que realizan tareas simples y bien definidas (como listar archivos o ejecutar un único grep) para reducir el costo y la latencia. `effort` se configura en las opciones de nivel superior de `query()`, no por subagente.

### Modo de permiso

La opción de modo de permiso (`permission_mode` en Python, `permissionMode` en TypeScript) controla si el agente solicita aprobación antes de usar herramientas:

| Modo | Comportamiento |
|:-----|:---------|
| `"default"` | Las herramientas no cubiertas por reglas de permiso activan tu callback de aprobación; sin callback significa denegar |
| `"acceptEdits"` | Aprueba automáticamente las ediciones de archivos; otras herramientas siguen las reglas predeterminadas |
| `"plan"` | Sin ejecución de herramientas; Claude produce un plan para revisión |
| `"dontAsk"` (solo TypeScript) | Nunca solicita confirmación. Las herramientas pre-aprobadas por [reglas de permisos](https://code.claude.com/docs/en/settings#permission-settings) se ejecutan, todo lo demás se deniega |
| `"bypassPermissions"` | Ejecuta todas las herramientas permitidas sin preguntar. No se puede usar cuando se ejecuta como root en Unix. Úsalo solo en entornos aislados donde las acciones del agente no puedan afectar sistemas que te importen |

Para aplicaciones interactivas, usa `"default"` con un callback de aprobación de herramientas para mostrar las solicitudes de aprobación. Para agentes autónomos en una máquina de desarrollo, `"acceptEdits"` aprueba automáticamente las ediciones de archivos mientras sigue requiriendo reglas de permisos para `Bash`. Reserva `"bypassPermissions"` para CI, contenedores u otros entornos aislados. Consulta [Permissions](./Guides/Configure%20permissions.md) para más detalles.

### Modelo

Si no configuras `model`, el SDK usa el modelo predeterminado de Claude Code, que depende de tu método de autenticación y suscripción. Configúralo explícitamente (por ejemplo, `model="claude-sonnet-4-6"`) para fijar un modelo específico o usar un modelo más pequeño para agentes más rápidos y económicos. Consulta [models](/docs/en/about-claude/models) para ver los IDs disponibles.

## La ventana de contexto

La ventana de contexto es la cantidad total de información disponible para Claude durante una sesión. No se reinicia entre turnos dentro de una sesión. Todo se acumula: el system prompt, las definiciones de herramientas, el historial de conversación, las entradas de herramientas y las salidas de herramientas. El contenido que permanece igual entre turnos (system prompt, definiciones de herramientas, CLAUDE.md) se almacena automáticamente en [prompt cache](/docs/en/build-with-claude/prompt-caching), lo que reduce el costo y la latencia para los prefijos repetidos.

### Qué consume contexto

Así es como cada componente afecta el contexto en el SDK:

| Fuente | Cuándo se carga | Impacto |
|:-------|:-------------|:-------|
| **System prompt** | Cada solicitud | Costo fijo pequeño, siempre presente |
| **Archivos CLAUDE.md** | Inicio de sesión, cuando [`settingSources`](./Core%20concepts/Use%20Claude%20Code%20features%20in%20the%20SDK.md) está habilitado | Contenido completo en cada solicitud (pero con prompt cache, solo la primera solicitud paga el costo completo) |
| **Definiciones de herramientas** | Cada solicitud | Cada herramienta agrega su esquema; usa [MCP tool search](./Guides/Connect%20to%20external%20tools%20with%20MCP.md#mcp-tool-search) para cargar herramientas bajo demanda en lugar de todas a la vez |
| **Historial de conversación** | Se acumula a lo largo de los turnos | Crece con cada turno: prompts, respuestas, entradas de herramientas, salidas de herramientas |
| **Descripciones de skills** | Inicio de sesión (con fuentes de configuración habilitadas) | Resúmenes cortos; el contenido completo se carga solo cuando se invocan |

Las salidas grandes de herramientas consumen contexto significativo. Leer un archivo grande o ejecutar un comando con salida verbosa puede usar miles de tokens en un solo turno. El contexto se acumula entre turnos, por lo que las sesiones más largas con muchas llamadas a herramientas acumulan significativamente más contexto que las cortas.

### Compactación automática

Cuando la ventana de contexto se acerca a su límite, el SDK compacta automáticamente la conversación: resume el historial más antiguo para liberar espacio, manteniendo intactos tus intercambios más recientes y las decisiones clave. El SDK emite un `SystemMessage` con subtipo `"compact_boundary"` en el stream cuando esto ocurre.

La compactación reemplaza los mensajes más antiguos con un resumen, por lo que las instrucciones específicas de etapas tempranas de la conversación pueden no preservarse. Las reglas persistentes deben estar en CLAUDE.md (cargado mediante [`settingSources`](./Core%20concepts/Use%20Claude%20Code%20features%20in%20the%20SDK.md)) en lugar de en el prompt inicial, porque el contenido de CLAUDE.md se reinyecta en cada solicitud.

Puedes personalizar el comportamiento de compactación de varias formas:

- **Instrucciones de resumen en CLAUDE.md:** El compactador lee tu CLAUDE.md como cualquier otro contexto, por lo que puedes incluir una sección que le indique qué preservar al resumir. El encabezado de la sección es libre (no es una cadena mágica); el compactador coincide por intención.
- **Hook `PreCompact`:** Ejecuta lógica personalizada antes de que ocurra la compactación, por ejemplo para archivar la transcripción completa. El hook recibe un campo `trigger` (`manual` o `auto`). Consulta [hooks](./Guides/Intercept%20and%20control%20agent%20behavior%20with%20hooks.md).
- **Compactación manual:** Envía `/compact` como cadena de prompt para activar la compactación bajo demanda. (Los slash commands enviados de esta manera son entradas del SDK, no atajos exclusivos del CLI. Consulta [slash commands en el SDK](./Guides/Slash%20Commands%20in%20the%20SDK.md).)

Agrega una sección al CLAUDE.md de tu proyecto indicando al compactador qué preservar. El nombre del encabezado no es especial; usa cualquier etiqueta clara.

```markdown CLAUDE.md
# Summary instructions

When summarizing this conversation, always preserve:
- The current task objective and acceptance criteria
- File paths that have been read or modified
- Test results and error messages
- Decisions made and the reasoning behind them
```

### Mantener el contexto eficiente

Algunas estrategias para agentes de larga duración:

- **Usa subagentes para subtareas.** Cada subagente comienza con una conversación nueva (sin historial de mensajes previo, aunque sí carga su propio system prompt y contexto a nivel de proyecto como CLAUDE.md). No ve los turnos del agente padre, y solo su respuesta final regresa al padre como resultado de herramienta. El contexto del agente principal crece con ese resumen, no con la transcripción completa de la subtarea. Consulta [What subagents inherit](./Guides/Subagents%20in%20the%20SDK.md#what-subagents-inherit) para más detalles.
- **Sé selectivo con las herramientas.** Cada definición de herramienta ocupa espacio de contexto. Usa el campo `tools` en `AgentDefinition` para limitar los subagentes al conjunto mínimo que necesitan, y usa [MCP tool search](./Guides/Connect%20to%20external%20tools%20with%20MCP.md#mcp-tool-search) para cargar herramientas bajo demanda en lugar de precargarlas todas.
- **Vigila los costos de los servidores MCP.** Cada servidor MCP agrega todos sus esquemas de herramientas a cada solicitud. Unos pocos servidores con muchas herramientas pueden consumir contexto significativo antes de que el agente haga cualquier trabajo. La herramienta `ToolSearch` puede ayudar cargando herramientas bajo demanda en lugar de precargarlas todas. Consulta [MCP tool search](./Guides/Connect%20to%20external%20tools%20with%20MCP.md#mcp-tool-search) para la configuración.
- **Usa un esfuerzo menor para tareas rutinarias.** Configura [effort](#nivel-de-esfuerzo) en `"low"` para agentes que solo necesitan leer archivos o listar directorios. Esto reduce el uso de tokens y el costo.

Para un desglose detallado de los costos de contexto por característica, consulta [Understand context costs](https://code.claude.com/docs/en/features-overview#understand-context-costs).

## Sesiones y continuidad

Cada interacción con el SDK crea o continúa una sesión. Captura el ID de sesión desde `ResultMessage.session_id` (disponible en ambos SDKs) para reanudarla más adelante. El SDK de TypeScript también lo expone como un campo directo en el `SystemMessage` de inicio; en Python está anidado en `SystemMessage.data`.

Cuando reanudas, se restaura el contexto completo de los turnos anteriores: los archivos que se leyeron, el análisis realizado y las acciones tomadas. También puedes bifurcar una sesión para explorar un enfoque diferente sin modificar la original.

Consulta [Session management](./Core%20concepts/Work%20with%20sessions.md) para la guía completa sobre los patrones de reanudación, continuación y bifurcación.

> **Nota:** En Python, `ClaudeSDKClient` maneja los IDs de sesión automáticamente entre múltiples llamadas. Consulta la [referencia del SDK de Python](/docs/en/agent-sdk/python#choosing-between-query-and-claude-sdk-client) para más detalles.

## Manejar el resultado

Cuando el bucle termina, el `ResultMessage` te indica qué ocurrió y te proporciona la salida. El campo `subtype` (disponible en ambos SDKs) es la forma principal de verificar el estado de terminación.

| Subtipo del resultado | Qué ocurrió | ¿Campo `result` disponible? |
|:------------|:-------------|:-------------------------:|
| `success` | Claude completó la tarea normalmente | Sí |
| `error_max_turns` | Se alcanzó el límite de `maxTurns` antes de terminar | No |
| `error_max_budget_usd` | Se alcanzó el límite de `maxBudgetUsd` antes de terminar | No |
| `error_during_execution` | Un error interrumpió el bucle (por ejemplo, un fallo de API o una solicitud cancelada) | No |
| `error_max_structured_output_retries` | La validación de salida estructurada falló después del límite de reintentos configurado | No |

El campo `result` (la salida de texto final) solo está presente en la variante `success`, por lo que siempre debes verificar el subtipo antes de leerlo. Todos los subtipos de resultado incluyen `total_cost_usd`, `usage`, `num_turns` y `session_id` para que puedas rastrear el costo y reanudar incluso después de errores. En Python, `total_cost_usd` y `usage` están tipados como opcionales y pueden ser `None` en algunas rutas de error, así que protégelos antes de formatearlos. Consulta [Tracking costs and usage](./Guides/Track%20cost%20and%20usage.md) para más detalles sobre la interpretación de los campos `usage`.

El resultado también incluye un campo `stop_reason` (`string | null` en TypeScript, `str | None` en Python) que indica por qué el modelo dejó de generar en su turno final. Los valores comunes son `end_turn` (el modelo terminó normalmente), `max_tokens` (se alcanzó el límite de tokens de salida) y `refusal` (el modelo rechazó la solicitud). En los subtipos de resultado de error, `stop_reason` lleva el valor de la última respuesta del asistente antes de que terminara el bucle. Para detectar rechazos, verifica `stop_reason === "refusal"` (TypeScript) o `stop_reason == "refusal"` (Python). Consulta [`SDKResultMessage`](/docs/en/agent-sdk/typescript#sdk-result-message) (TypeScript) o [`ResultMessage`](/docs/en/agent-sdk/python#result-message) (Python) para el tipo completo.

## Hooks

Los [Hooks](./Guides/Intercept%20and%20control%20agent%20behavior%20with%20hooks.md) son callbacks que se disparan en puntos específicos del bucle: antes de que se ejecute una herramienta, después de que devuelva un resultado, cuando el agente termina, etc. Algunos hooks de uso común son:

| Hook | Cuándo se dispara | Usos comunes |
|:-----|:-------------|:------------|
| `PreToolUse` | Antes de que se ejecute una herramienta | Validar entradas, bloquear comandos peligrosos |
| `PostToolUse` | Después de que una herramienta devuelve un resultado | Auditar salidas, activar efectos secundarios |
| `UserPromptSubmit` | Cuando se envía un prompt | Inyectar contexto adicional en los prompts |
| `Stop` | Cuando el agente termina | Validar el resultado, guardar el estado de la sesión |
| `SubagentStart` / `SubagentStop` | Cuando un subagente se crea o completa | Rastrear y agregar resultados de tareas paralelas |
| `PreCompact` | Antes de la compactación de contexto | Archivar la transcripción completa antes de resumir |

Los hooks se ejecutan en el proceso de tu aplicación, no dentro de la ventana de contexto del agente, por lo que no consumen contexto. Los hooks también pueden interrumpir el bucle: un hook `PreToolUse` que rechaza una llamada a herramienta impide su ejecución, y Claude recibe el mensaje de rechazo en su lugar.

Ambos SDKs admiten todos los eventos anteriores. El SDK de TypeScript incluye eventos adicionales que Python aún no admite. Consulta [Control execution with hooks](./Guides/Intercept%20and%20control%20agent%20behavior%20with%20hooks.md) para la lista completa de eventos, disponibilidad por SDK y la API completa de callbacks.

## Juntando todo

Este ejemplo combina los conceptos clave de esta página en un solo agente que corrige pruebas fallidas. Configura el agente con herramientas permitidas (aprobadas automáticamente para que el agente se ejecute de forma autónoma), configuración de proyecto y límites de seguridad en turnos y nivel de razonamiento. Mientras se ejecuta el bucle, captura el ID de sesión para una posible reanudación, maneja el resultado final e imprime el costo total.

**Python**

```python Python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage


async def run_agent():
    session_id = None

    async for message in query(
        prompt="Find and fix the bug causing test failures in the auth module",
        options=ClaudeAgentOptions(
            allowed_tools=[
                "Read",
                "Edit",
                "Bash",
                "Glob",
                "Grep",
            ],  # Listing tools here auto-approves them (no prompting)
            setting_sources=[
                "project"
            ],  # Load CLAUDE.md, skills, hooks from current directory
            max_turns=30,  # Prevent runaway sessions
            effort="high",  # Thorough reasoning for complex debugging
        ),
    ):
        # Handle the final result
        if isinstance(message, ResultMessage):
            session_id = message.session_id  # Save for potential resumption

            if message.subtype == "success":
                print(f"Done: {message.result}")
            elif message.subtype == "error_max_turns":
                # Agent ran out of turns. Resume with a higher limit.
                print(f"Hit turn limit. Resume session {session_id} to continue.")
            elif message.subtype == "error_max_budget_usd":
                print("Hit budget limit.")
            else:
                print(f"Stopped: {message.subtype}")
            if message.total_cost_usd is not None:
                print(f"Cost: ${message.total_cost_usd:.4f}")


asyncio.run(run_agent())
```

**TypeScript**

```typescript TypeScript
import { query } from "@anthropic-ai/claude-agent-sdk";

let sessionId: string | undefined;

for await (const message of query({
  prompt: "Find and fix the bug causing test failures in the auth module",
  options: {
    allowedTools: ["Read", "Edit", "Bash", "Glob", "Grep"], // Listing tools here auto-approves them (no prompting)
    settingSources: ["project"], // Load CLAUDE.md, skills, hooks from current directory
    maxTurns: 30, // Prevent runaway sessions
    effort: "high" // Thorough reasoning for complex debugging
  }
})) {
  // Save the session ID to resume later if needed
  if (message.type === "system" && message.subtype === "init") {
    sessionId = message.session_id;
  }

  // Handle the final result
  if (message.type === "result") {
    if (message.subtype === "success") {
      console.log(`Done: ${message.result}`);
    } else if (message.subtype === "error_max_turns") {
      // Agent ran out of turns. Resume with a higher limit.
      console.log(`Hit turn limit. Resume session ${sessionId} to continue.`);
    } else if (message.subtype === "error_max_budget_usd") {
      console.log("Hit budget limit.");
    } else {
      console.log(`Stopped: ${message.subtype}`);
    }
    console.log(`Cost: $${message.total_cost_usd.toFixed(4)}`);
  }
}
```

## Próximos pasos

Ahora que entiendes el bucle, aquí te indicamos a dónde ir según lo que estés construyendo:

- **¿Aún no has ejecutado un agente?** Comienza con el [quickstart](./Quickstart.md) para instalar el SDK y ver un ejemplo completo en funcionamiento de principio a fin.
- **¿Listo para integrarlo en tu proyecto?** [Carga CLAUDE.md, skills y filesystem hooks](./Core%20concepts/Use%20Claude%20Code%20features%20in%20the%20SDK.md) para que el agente siga las convenciones de tu proyecto automáticamente.
- **¿Construyendo una interfaz interactiva?** Habilita el [streaming](./Guides/Stream%20responses%20in%20real-time.md) para mostrar texto en vivo y llamadas a herramientas mientras se ejecuta el bucle.
- **¿Necesitas un control más estricto sobre lo que puede hacer el agente?** Restringe el acceso a herramientas con [permissions](./Guides/Configure%20permissions.md), y usa [hooks](./Guides/Intercept%20and%20control%20agent%20behavior%20with%20hooks.md) para auditar, bloquear o transformar llamadas a herramientas antes de que se ejecuten.
- **¿Ejecutando tareas largas o costosas?** Delega trabajo aislado a [subagentes](./Guides/Subagents%20in%20the%20SDK.md) para mantener tu contexto principal liviano.

Para una visión conceptual más amplia del bucle agéntico (no específica del SDK), consulta [How Claude Code works](https://code.claude.com/docs/en/how-claude-code-works).
