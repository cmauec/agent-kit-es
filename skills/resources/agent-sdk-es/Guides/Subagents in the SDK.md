# Subagentes en el SDK

Define e invoca subagentes para aislar el contexto, ejecutar tareas en paralelo y aplicar instrucciones especializadas en tus aplicaciones del Claude Agent SDK.

---

Los subagentes son instancias de agente separadas que tu agente principal puede crear para manejar subtareas específicas.
Usa subagentes para aislar el contexto en subtareas concretas, ejecutar múltiples análisis en paralelo y aplicar instrucciones especializadas sin sobrecargar el prompt del agente principal.

Esta guía explica cómo definir y usar subagentes en el SDK mediante el parámetro `agents`.

## Descripción general

Puedes crear subagentes de tres maneras:

- **Programáticamente**: usa el parámetro `agents` en las opciones de `query()` ([TypeScript](/docs/en/agent-sdk/typescript#agent-definition), [Python](/docs/en/agent-sdk/python#agent-definition))
- **Basado en el sistema de archivos**: define agentes como archivos markdown en directorios `.claude/agents/` (consulta [definir subagentes como archivos](https://code.claude.com/docs/en/sub-agents))
- **De propósito general integrado**: Claude puede invocar el subagente integrado `general-purpose` en cualquier momento a través de la herramienta Agent sin que tú definas nada

Esta guía se centra en el enfoque programático, que es el recomendado para aplicaciones con el SDK.

Cuando defines subagentes, Claude decide si invocarlos según el campo `description` de cada uno. Escribe descripciones claras que expliquen cuándo debe usarse el subagente, y Claude delegará automáticamente las tareas adecuadas. También puedes solicitar explícitamente un subagente por nombre en tu prompt (por ejemplo, "Use the code-reviewer agent to...").

## Ventajas de usar subagentes

### Aislamiento de contexto

Cada subagente se ejecuta en su propia conversación nueva. Las llamadas intermedias a herramientas y sus resultados permanecen dentro del subagente; solo su mensaje final regresa al agente principal. Consulta [Qué heredan los subagentes](#what-subagents-inherit) para saber exactamente qué hay en el contexto del subagente.

**Ejemplo:** un subagente `research-assistant` puede explorar docenas de archivos sin que ninguno de ese contenido se acumule en la conversación principal. El agente principal recibe un resumen conciso, no cada archivo que leyó el subagente.

### Paralelización
Múltiples subagentes pueden ejecutarse de forma concurrente, acelerando notablemente los flujos de trabajo complejos.

**Ejemplo:** durante una revisión de código, puedes ejecutar los subagentes `style-checker`, `security-scanner` y `test-coverage` simultáneamente, reduciendo el tiempo de revisión de minutos a segundos.

### Instrucciones y conocimiento especializados
Cada subagente puede tener prompts de sistema personalizados con experiencia, mejores prácticas y restricciones específicas.

**Ejemplo:** un subagente `database-migration` puede tener conocimiento detallado sobre las mejores prácticas de SQL, estrategias de rollback y verificaciones de integridad de datos que serían ruido innecesario en las instrucciones del agente principal.

### Restricciones de herramientas
Los subagentes pueden limitarse a herramientas específicas, reduciendo el riesgo de acciones no deseadas.

**Ejemplo:** un subagente `doc-reviewer` podría tener acceso únicamente a las herramientas Read y Grep, asegurando que pueda analizar pero nunca modificar accidentalmente tus archivos de documentación.

## Creación de subagentes

### Definición programática (recomendada)

Define subagentes directamente en tu código usando el parámetro `agents`. Este ejemplo crea dos subagentes: un revisor de código con acceso de solo lectura y un ejecutor de pruebas que puede ejecutar comandos. La herramienta `Agent` debe incluirse en `allowedTools` ya que Claude invoca los subagentes a través de ella.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition


async def main():
    async for message in query(
        prompt="Review the authentication module for security issues",
        options=ClaudeAgentOptions(
            # Agent tool is required for subagent invocation
            allowed_tools=["Read", "Grep", "Glob", "Agent"],
            agents={
                "code-reviewer": AgentDefinition(
                    # description tells Claude when to use this subagent
                    description="Expert code review specialist. Use for quality, security, and maintainability reviews.",
                    # prompt defines the subagent's behavior and expertise
                    prompt="""You are a code review specialist with expertise in security, performance, and best practices.

When reviewing code:
- Identify security vulnerabilities
- Check for performance issues
- Verify adherence to coding standards
- Suggest specific improvements

Be thorough but concise in your feedback.""",
                    # tools restricts what the subagent can do (read-only here)
                    tools=["Read", "Grep", "Glob"],
                    # model overrides the default model for this subagent
                    model="sonnet",
                ),
                "test-runner": AgentDefinition(
                    description="Runs and analyzes test suites. Use for test execution and coverage analysis.",
                    prompt="""You are a test execution specialist. Run tests and provide clear analysis of results.

Focus on:
- Running test commands
- Analyzing test output
- Identifying failing tests
- Suggesting fixes for failures""",
                    # Bash access lets this subagent run test commands
                    tools=["Bash", "Read", "Grep"],
                ),
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Review the authentication module for security issues",
  options: {
    // Agent tool is required for subagent invocation
    allowedTools: ["Read", "Grep", "Glob", "Agent"],
    agents: {
      "code-reviewer": {
        // description tells Claude when to use this subagent
        description:
          "Expert code review specialist. Use for quality, security, and maintainability reviews.",
        // prompt defines the subagent's behavior and expertise
        prompt: `You are a code review specialist with expertise in security, performance, and best practices.

When reviewing code:
- Identify security vulnerabilities
- Check for performance issues
- Verify adherence to coding standards
- Suggest specific improvements

Be thorough but concise in your feedback.`,
        // tools restricts what the subagent can do (read-only here)
        tools: ["Read", "Grep", "Glob"],
        // model overrides the default model for this subagent
        model: "sonnet"
      },
      "test-runner": {
        description:
          "Runs and analyzes test suites. Use for test execution and coverage analysis.",
        prompt: `You are a test execution specialist. Run tests and provide clear analysis of results.

Focus on:
- Running test commands
- Analyzing test output
- Identifying failing tests
- Suggesting fixes for failures`,
        // Bash access lets this subagent run test commands
        tools: ["Bash", "Read", "Grep"]
      }
    }
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

### Configuración de AgentDefinition

| Campo | Tipo | Requerido | Descripción |
|:------|:-----|:---------|:------------|
| `description` | `string` | Sí | Descripción en lenguaje natural de cuándo usar este agente |
| `prompt` | `string` | Sí | El prompt de sistema del agente que define su rol y comportamiento |
| `tools` | `string[]` | No | Array de nombres de herramientas permitidas. Si se omite, hereda todas las herramientas |
| `model` | `'sonnet' \| 'opus' \| 'haiku' \| 'inherit'` | No | Modelo a usar para este agente. Por defecto usa el modelo principal si se omite |

> **Nota:** Los subagentes no pueden crear sus propios subagentes. No incluyas `Agent` en el array `tools` de un subagente.

### Definición basada en el sistema de archivos (alternativa)

También puedes definir subagentes como archivos markdown en directorios `.claude/agents/`. Consulta la [documentación de subagentes de Claude Code](https://code.claude.com/docs/en/sub-agents) para más detalles sobre este enfoque. Los agentes definidos programáticamente tienen precedencia sobre los basados en archivos con el mismo nombre.

> **Nota:** Incluso sin definir subagentes personalizados, Claude puede crear el subagente integrado `general-purpose` cuando `Agent` está en tu `allowedTools`. Esto es útil para delegar tareas de investigación o exploración sin crear agentes especializados.

## Qué heredan los subagentes

La ventana de contexto de un subagente comienza vacía (sin conversación del agente principal), pero no está completamente en blanco. El único canal del agente principal al subagente es la cadena de texto del prompt de la herramienta Agent, por lo que debes incluir directamente en ese prompt cualquier ruta de archivo, mensaje de error o decisión que el subagente necesite.

| El subagente recibe | El subagente no recibe |
|:---|:---|
| Su propio prompt de sistema (`AgentDefinition.prompt`) y el prompt de la herramienta Agent | El historial de conversación o los resultados de herramientas del agente principal |
| El archivo CLAUDE.md del proyecto (cargado via `settingSources`) | Skills (a menos que estén listados en `AgentDefinition.skills`, solo TypeScript) |
| Definiciones de herramientas (heredadas del agente principal, o el subconjunto en `tools`) | El prompt de sistema del agente principal |

> **Nota:** El agente principal recibe el mensaje final del subagente tal cual como resultado de la herramienta Agent, pero puede resumirlo en su propia respuesta. Para preservar la salida del subagente de forma literal en la respuesta al usuario, incluye una instrucción al respecto en el prompt o en la opción `systemPrompt` que pasas a la llamada principal de `query()`.

## Invocación de subagentes

### Invocación automática

Claude decide automáticamente cuándo invocar subagentes según la tarea y la `description` de cada subagente. Por ejemplo, si defines un subagente `performance-optimizer` con la descripción "Performance optimization specialist for query tuning", Claude lo invocará cuando tu prompt mencione la optimización de consultas.

Escribe descripciones claras y específicas para que Claude pueda asignar las tareas al subagente correcto.

### Invocación explícita

Para garantizar que Claude use un subagente específico, menciónalo por nombre en tu prompt:

```text
"Use the code-reviewer agent to check the authentication module"
```

Esto evita la asignación automática e invoca directamente el subagente indicado.

### Configuración dinámica de agentes

Puedes crear definiciones de agentes dinámicamente según las condiciones en tiempo de ejecución. Este ejemplo crea un revisor de seguridad con distintos niveles de rigor, usando un modelo más potente para las revisiones más estrictas.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition


# Factory function that returns an AgentDefinition
# This pattern lets you customize agents based on runtime conditions
def create_security_agent(security_level: str) -> AgentDefinition:
    is_strict = security_level == "strict"
    return AgentDefinition(
        description="Security code reviewer",
        # Customize the prompt based on strictness level
        prompt=f"You are a {'strict' if is_strict else 'balanced'} security reviewer...",
        tools=["Read", "Grep", "Glob"],
        # Key insight: use a more capable model for high-stakes reviews
        model="opus" if is_strict else "sonnet",
    )


async def main():
    # The agent is created at query time, so each request can use different settings
    async for message in query(
        prompt="Review this PR for security issues",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Grep", "Glob", "Agent"],
            agents={
                # Call the factory with your desired configuration
                "security-reviewer": create_security_agent("strict")
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query, type AgentDefinition } from "@anthropic-ai/claude-agent-sdk";

// Factory function that returns an AgentDefinition
// This pattern lets you customize agents based on runtime conditions
function createSecurityAgent(securityLevel: "basic" | "strict"): AgentDefinition {
  const isStrict = securityLevel === "strict";
  return {
    description: "Security code reviewer",
    // Customize the prompt based on strictness level
    prompt: `You are a ${isStrict ? "strict" : "balanced"} security reviewer...`,
    tools: ["Read", "Grep", "Glob"],
    // Key insight: use a more capable model for high-stakes reviews
    model: isStrict ? "opus" : "sonnet"
  };
}

// The agent is created at query time, so each request can use different settings
for await (const message of query({
  prompt: "Review this PR for security issues",
  options: {
    allowedTools: ["Read", "Grep", "Glob", "Agent"],
    agents: {
      // Call the factory with your desired configuration
      "security-reviewer": createSecurityAgent("strict")
    }
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

## Detección de invocación de subagentes

Los subagentes se invocan a través de la herramienta Agent. Para detectar cuándo se invoca un subagente, comprueba los bloques `tool_use` donde `name` sea `"Agent"`. Los mensajes provenientes del contexto de un subagente incluyen un campo `parent_tool_use_id`.

> **Nota:** El nombre de la herramienta cambió de `"Task"` a `"Agent"` en Claude Code v2.1.63. Las versiones actuales del SDK emiten `"Agent"` en los bloques `tool_use`, pero aún usan `"Task"` en la lista de herramientas de `system:init` y en `result.permission_denials[].tool_name`. Verificar ambos valores en `block.name` garantiza compatibilidad entre versiones del SDK.

Este ejemplo itera sobre los mensajes en streaming, registrando cuándo se invoca un subagente y cuándo los mensajes posteriores provienen del contexto de ejecución de ese subagente.

> **Nota:** La estructura de los mensajes difiere entre SDKs. En Python, los bloques de contenido se acceden directamente mediante `message.content`. En TypeScript, `SDKAssistantMessage` envuelve el mensaje de la API de Claude, por lo que el contenido se accede mediante `message.message.content`.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition


async def main():
    async for message in query(
        prompt="Use the code-reviewer agent to review this codebase",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Glob", "Grep", "Agent"],
            agents={
                "code-reviewer": AgentDefinition(
                    description="Expert code reviewer.",
                    prompt="Analyze code quality and suggest improvements.",
                    tools=["Read", "Glob", "Grep"],
                )
            },
        ),
    ):
        # Check for subagent invocation. Match both names: older SDK
        # versions emitted "Task", current versions emit "Agent".
        if hasattr(message, "content") and message.content:
            for block in message.content:
                if getattr(block, "type", None) == "tool_use" and block.name in (
                    "Task",
                    "Agent",
                ):
                    print(f"Subagent invoked: {block.input.get('subagent_type')}")

        # Check if this message is from within a subagent's context
        if hasattr(message, "parent_tool_use_id") and message.parent_tool_use_id:
            print("  (running inside subagent)")

        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Use the code-reviewer agent to review this codebase",
  options: {
    allowedTools: ["Read", "Glob", "Grep", "Agent"],
    agents: {
      "code-reviewer": {
        description: "Expert code reviewer.",
        prompt: "Analyze code quality and suggest improvements.",
        tools: ["Read", "Glob", "Grep"]
      }
    }
  }
})) {
  const msg = message as any;

  // Check for subagent invocation. Match both names: older SDK versions
  // emitted "Task", current versions emit "Agent".
  for (const block of msg.message?.content ?? []) {
    if (block.type === "tool_use" && (block.name === "Task" || block.name === "Agent")) {
      console.log(`Subagent invoked: ${block.input.subagent_type}`);
    }
  }

  // Check if this message is from within a subagent's context
  if (msg.parent_tool_use_id) {
    console.log("  (running inside subagent)");
  }

  if ("result" in message) {
    console.log(message.result);
  }
}
```

## Reanudación de subagentes

Los subagentes pueden reanudarse para continuar donde se detuvieron. Los subagentes reanudados retienen todo su historial de conversación, incluyendo todas las llamadas a herramientas, resultados y razonamiento previos. El subagente retoma exactamente donde se detuvo en lugar de comenzar de nuevo.

Cuando un subagente completa su tarea, Claude recibe su ID de agente en el resultado de la herramienta Agent. Para reanudar un subagente programáticamente:

1. **Captura el ID de sesión**: extrae `session_id` de los mensajes durante la primera consulta
2. **Extrae el ID del agente**: analiza `agentId` del contenido del mensaje
3. **Reanuda la sesión**: pasa `resume: sessionId` en las opciones de la segunda consulta e incluye el ID del agente en tu prompt

> **Nota:** Debes reanudar la misma sesión para acceder a la transcripción del subagente. Cada llamada a `query()` inicia una nueva sesión por defecto, así que pasa `resume: sessionId` para continuar en la misma sesión.
>
> Si usas un agente personalizado (no uno integrado), también debes pasar la misma definición de agente en el parámetro `agents` en ambas consultas.

El ejemplo a continuación muestra este flujo: la primera consulta ejecuta un subagente y captura el ID de sesión y el ID del agente; luego la segunda consulta reanuda la sesión para hacer una pregunta de seguimiento que requiere contexto del primer análisis.

**TypeScript**
```typescript
import { query, type SDKMessage } from "@anthropic-ai/claude-agent-sdk";

// Helper to extract agentId from message content
// Stringify to avoid traversing different block types (TextBlock, ToolResultBlock, etc.)
function extractAgentId(message: SDKMessage): string | undefined {
  if (!("message" in message)) return undefined;
  // Stringify the content so we can search it without traversing nested blocks
  const content = JSON.stringify(message.message.content);
  const match = content.match(/agentId:\s*([a-f0-9-]+)/);
  return match?.[1];
}

let agentId: string | undefined;
let sessionId: string | undefined;

// First invocation - use the Explore agent to find API endpoints
for await (const message of query({
  prompt: "Use the Explore agent to find all API endpoints in this codebase",
  options: { allowedTools: ["Read", "Grep", "Glob", "Agent"] }
})) {
  // Capture session_id from ResultMessage (needed to resume this session)
  if ("session_id" in message) sessionId = message.session_id;
  // Search message content for the agentId (appears in Agent tool results)
  const extractedId = extractAgentId(message);
  if (extractedId) agentId = extractedId;
  // Print the final result
  if ("result" in message) console.log(message.result);
}

// Second invocation - resume and ask follow-up
if (agentId && sessionId) {
  for await (const message of query({
    prompt: `Resume agent ${agentId} and list the top 3 most complex endpoints`,
    options: { allowedTools: ["Read", "Grep", "Glob", "Agent"], resume: sessionId }
  })) {
    if ("result" in message) console.log(message.result);
  }
}
```

**Python**
```python
import asyncio
import json
import re
from claude_agent_sdk import query, ClaudeAgentOptions


def extract_agent_id(text: str) -> str | None:
    """Extract agentId from Agent tool result text."""
    match = re.search(r"agentId:\s*([a-f0-9-]+)", text)
    return match.group(1) if match else None


async def main():
    agent_id = None
    session_id = None

    # First invocation - use the Explore agent to find API endpoints
    async for message in query(
        prompt="Use the Explore agent to find all API endpoints in this codebase",
        options=ClaudeAgentOptions(allowed_tools=["Read", "Grep", "Glob", "Agent"]),
    ):
        # Capture session_id from ResultMessage (needed to resume this session)
        if hasattr(message, "session_id"):
            session_id = message.session_id
        # Search message content for the agentId (appears in Agent tool results)
        if hasattr(message, "content"):
            # Stringify the content so we can search it without traversing nested blocks
            content_str = json.dumps(message.content, default=str)
            extracted = extract_agent_id(content_str)
            if extracted:
                agent_id = extracted
        # Print the final result
        if hasattr(message, "result"):
            print(message.result)

    # Second invocation - resume and ask follow-up
    if agent_id and session_id:
        async for message in query(
            prompt=f"Resume agent {agent_id} and list the top 3 most complex endpoints",
            options=ClaudeAgentOptions(
                allowed_tools=["Read", "Grep", "Glob", "Agent"], resume=session_id
            ),
        ):
            if hasattr(message, "result"):
                print(message.result)


asyncio.run(main())
```

Las transcripciones de los subagentes persisten de forma independiente a la conversación principal:

- **Compactación de la conversación principal**: cuando la conversación principal se compacta, las transcripciones de los subagentes no se ven afectadas. Se almacenan en archivos separados.
- **Persistencia de sesión**: las transcripciones de los subagentes persisten dentro de su sesión. Puedes reanudar un subagente después de reiniciar Claude Code reanudando la misma sesión.
- **Limpieza automática**: las transcripciones se eliminan según la configuración `cleanupPeriodDays` (por defecto: 30 días).

## Restricciones de herramientas

Los subagentes pueden tener acceso restringido a herramientas mediante el campo `tools`:

- **Omitir el campo**: el agente hereda todas las herramientas disponibles (valor por defecto)
- **Especificar herramientas**: el agente solo puede usar las herramientas indicadas

Este ejemplo crea un agente de análisis de solo lectura que puede examinar código pero no modificar archivos ni ejecutar comandos.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, AgentDefinition


async def main():
    async for message in query(
        prompt="Analyze the architecture of this codebase",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Grep", "Glob", "Agent"],
            agents={
                "code-analyzer": AgentDefinition(
                    description="Static code analysis and architecture review",
                    prompt="""You are a code architecture analyst. Analyze code structure,
identify patterns, and suggest improvements without making changes.""",
                    # Read-only tools: no Edit, Write, or Bash access
                    tools=["Read", "Grep", "Glob"],
                )
            },
        ),
    ):
        if hasattr(message, "result"):
            print(message.result)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Analyze the architecture of this codebase",
  options: {
    allowedTools: ["Read", "Grep", "Glob", "Agent"],
    agents: {
      "code-analyzer": {
        description: "Static code analysis and architecture review",
        prompt: `You are a code architecture analyst. Analyze code structure,
identify patterns, and suggest improvements without making changes.`,
        // Read-only tools: no Edit, Write, or Bash access
        tools: ["Read", "Grep", "Glob"]
      }
    }
  }
})) {
  if ("result" in message) console.log(message.result);
}
```

### Combinaciones de herramientas comunes

| Caso de uso | Herramientas | Descripción |
|:---------|:------|:------------|
| Análisis de solo lectura | `Read`, `Grep`, `Glob` | Puede examinar código pero no modificarlo ni ejecutar comandos |
| Ejecución de pruebas | `Bash`, `Read`, `Grep` | Puede ejecutar comandos y analizar la salida |
| Modificación de código | `Read`, `Edit`, `Write`, `Grep`, `Glob` | Acceso completo de lectura/escritura sin ejecución de comandos |
| Acceso completo | Todas las herramientas | Hereda todas las herramientas del agente principal (omite el campo `tools`) |

## Solución de problemas

### Claude no delega en los subagentes

Si Claude completa las tareas directamente en lugar de delegar en tu subagente:

1. **Incluye la herramienta Agent**: los subagentes se invocan a través de la herramienta Agent, por lo que debe estar en `allowedTools`
2. **Usa un prompt explícito**: menciona el subagente por nombre en tu prompt (por ejemplo, "Use the code-reviewer agent to...")
3. **Escribe una descripción clara**: explica exactamente cuándo debe usarse el subagente para que Claude pueda asignar las tareas correctamente

### Los agentes basados en el sistema de archivos no se cargan

Los agentes definidos en `.claude/agents/` se cargan únicamente al inicio. Si creas un nuevo archivo de agente mientras Claude Code está en ejecución, reinicia la sesión para cargarlo.

### Windows: fallos con prompts muy largos

En Windows, los subagentes con prompts muy largos pueden fallar debido a los límites de longitud de la línea de comandos (8191 caracteres). Mantén los prompts concisos o usa agentes basados en el sistema de archivos para instrucciones complejas.

## Documentación relacionada

- [Subagentes de Claude Code](https://code.claude.com/docs/en/sub-agents): documentación completa sobre subagentes, incluidas las definiciones basadas en el sistema de archivos
- [Descripción general del SDK](/docs/en/agent-sdk/overview): primeros pasos con el Claude Agent SDK
