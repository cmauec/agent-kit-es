# Usar las funcionalidades de Claude Code en el SDK

Carga instrucciones de proyecto, skills, hooks y otras funcionalidades de Claude Code en tus agentes del SDK.

---

El Agent SDK está construido sobre la misma base que Claude Code, lo que significa que tus agentes del SDK tienen acceso a las mismas funcionalidades basadas en el sistema de archivos: instrucciones de proyecto (`CLAUDE.md` y reglas), skills, hooks y más.

Por defecto, el SDK no carga ninguna configuración del sistema de archivos. Tu agente se ejecuta en modo aislado solo con lo que le pasas de forma programática. Para cargar CLAUDE.md, skills o hooks del sistema de archivos, configura `settingSources` para indicarle al SDK dónde buscar.

Para una visión conceptual de lo que hace cada funcionalidad y cuándo usarla, consulta [Extend Claude Code](https://code.claude.com/docs/en/features-overview).

## Habilitar las funcionalidades de Claude Code con settingSources

La opción de fuentes de configuración ([`setting_sources`](/docs/en/agent-sdk/python#claude-agent-options) en Python, [`settingSources`](/docs/en/agent-sdk/typescript#setting-source) en TypeScript) controla qué configuraciones basadas en el sistema de archivos carga el SDK. Sin ella, tu agente no descubrirá skills, archivos `CLAUDE.md` ni hooks a nivel de proyecto.

Este ejemplo carga tanto la configuración a nivel de usuario como la de proyecto estableciendo `settingSources` en `["user", "project"]`:

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions, AssistantMessage, ResultMessage

async for message in query(
    prompt="Help me refactor the auth module",
    options=ClaudeAgentOptions(
        # "user" loads from ~/.claude/, "project" loads from ./.claude/ in cwd.
        # Together they give the agent access to CLAUDE.md, skills, hooks, and
        # permissions from both locations.
        setting_sources=["user", "project"],
        allowed_tools=["Read", "Edit", "Bash"],
    ),
):
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if hasattr(block, "text"):
                print(block.text)
    if isinstance(message, ResultMessage) and message.subtype == "success":
        print(f"\nResult: {message.result}")
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Help me refactor the auth module",
  options: {
    // "user" loads from ~/.claude/, "project" loads from ./.claude/ in cwd.
    // Together they give the agent access to CLAUDE.md, skills, hooks, and
    // permissions from both locations.
    settingSources: ["user", "project"],
    allowedTools: ["Read", "Edit", "Bash"]
  }
})) {
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if (block.type === "text") console.log(block.text);
    }
  }
  if (message.type === "result" && message.subtype === "success") {
    console.log(`\nResult: ${message.result}`);
  }
}
```

Cada fuente carga la configuración desde una ubicación específica, donde `<cwd>` es el directorio de trabajo que pasas mediante la opción `cwd` (o el directorio actual del proceso si no se especifica). Para la definición completa del tipo, consulta [`SettingSource`](/docs/en/agent-sdk/typescript#setting-source) (TypeScript) o [`SettingSource`](/docs/en/agent-sdk/python#setting-source) (Python).

| Fuente | Qué carga | Ubicación |
|:-------|:----------|:----------|
| `"project"` | CLAUDE.md de proyecto, `.claude/rules/*.md`, skills de proyecto, hooks de proyecto, `settings.json` de proyecto | `<cwd>/.claude/` y cada directorio padre hasta la raíz del sistema de archivos (deteniéndose cuando se encuentra un `.claude/` o no existen más padres) |
| `"user"` | CLAUDE.md de usuario, `~/.claude/rules/*.md`, skills de usuario, configuración de usuario | `~/.claude/` |
| `"local"` | CLAUDE.local.md (en gitignore), `.claude/settings.local.json` | `<cwd>/` |

Para replicar el comportamiento completo del CLI de Claude Code, usa `["user", "project", "local"]`.

> **Advertencia:** La opción `cwd` determina dónde busca el SDK la configuración del proyecto. Si ni `cwd` ni ninguno de sus directorios padre contiene una carpeta `.claude/`, las funcionalidades a nivel de proyecto no se cargarán. La memoria automática (el directorio `~/.claude/projects//memory/` que usa Claude Code para persistir notas entre sesiones interactivas) es una funcionalidad exclusiva del CLI y el SDK nunca la carga.

## Instrucciones de proyecto (CLAUDE.md y reglas)

Los archivos `CLAUDE.md` y `.claude/rules/*.md` le proporcionan a tu agente contexto persistente sobre tu proyecto: convenciones de código, comandos de compilación, decisiones de arquitectura e instrucciones. Cuando `settingSources` incluye `"project"` (como en el ejemplo anterior), el SDK carga estos archivos en el contexto al inicio de la sesión. El agente entonces sigue las convenciones de tu proyecto sin que tengas que repetirlas en cada prompt.

### Ubicaciones de carga de CLAUDE.md

| Nivel | Ubicación | Cuándo se carga |
|:------|:---------|:----------------|
| Proyecto (raíz) | `<cwd>/CLAUDE.md` o `<cwd>/.claude/CLAUDE.md` | `settingSources` incluye `"project"` |
| Reglas de proyecto | `<cwd>/.claude/rules/*.md` | `settingSources` incluye `"project"` |
| Proyecto (directorios padre) | Archivos `CLAUDE.md` en directorios por encima de `cwd` | `settingSources` incluye `"project"`, se cargan al inicio de la sesión |
| Proyecto (directorios hijo) | Archivos `CLAUDE.md` en subdirectorios de `cwd` | `settingSources` incluye `"project"`, se cargan bajo demanda cuando el agente lee un archivo en ese subárbol |
| Local (en gitignore) | `<cwd>/CLAUDE.local.md` | `settingSources` incluye `"local"` |
| Usuario | `~/.claude/CLAUDE.md` | `settingSources` incluye `"user"` |
| Reglas de usuario | `~/.claude/rules/*.md` | `settingSources` incluye `"user"` |

Todos los niveles son aditivos: si existen tanto archivos CLAUDE.md de proyecto como de usuario, el agente ve ambos. No hay una regla estricta de precedencia entre niveles; si las instrucciones entran en conflicto, el resultado depende de cómo las interprete Claude. Escribe reglas que no entren en conflicto o establece la precedencia explícitamente en el archivo más específico ("Estas instrucciones de proyecto anulan cualquier valor predeterminado a nivel de usuario que entre en conflicto").

> **Tip:** También puedes inyectar contexto directamente mediante `systemPrompt` sin usar archivos CLAUDE.md. Consulta [Modify system prompts](/docs/en/agent-sdk/modifying-system-prompts). Usa CLAUDE.md cuando quieras que el mismo contexto se comparta entre las sesiones interactivas de Claude Code y tus agentes del SDK.

Para saber cómo estructurar y organizar el contenido de CLAUDE.md, consulta [Manage Claude's memory](https://code.claude.com/docs/en/memory).

## Skills

Las skills son archivos markdown que le proporcionan a tu agente conocimiento especializado y flujos de trabajo invocables. A diferencia de `CLAUDE.md` (que se carga en cada sesión), las skills se cargan bajo demanda. El agente recibe las descripciones de las skills al inicio y carga el contenido completo cuando es relevante.

Para usar skills en el SDK, configura `settingSources` para que el agente descubra los archivos de skills desde el sistema de archivos. La herramienta `Skill` está habilitada por defecto cuando no especificas `allowedTools`. Si estás usando una lista de permisos en `allowedTools`, incluye `"Skill"` explícitamente.

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

# Skills in .claude/skills/ are discovered automatically
# when settingSources includes "project"
async for message in query(
    prompt="Review this PR using our code review checklist",
    options=ClaudeAgentOptions(
        setting_sources=["user", "project"],
        allowed_tools=["Skill", "Read", "Grep", "Glob"],
    ),
):
    if isinstance(message, ResultMessage) and message.subtype == "success":
        print(message.result)
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Skills in .claude/skills/ are discovered automatically
// when settingSources includes "project"
for await (const message of query({
  prompt: "Review this PR using our code review checklist",
  options: {
    settingSources: ["user", "project"],
    allowedTools: ["Skill", "Read", "Grep", "Glob"]
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

> **Nota:** Las skills deben crearse como artefactos del sistema de archivos (`.claude/skills/<name>/SKILL.md`). El SDK no tiene una API programática para registrar skills. Consulta [Agent Skills in the SDK](/docs/en/agent-sdk/skills) para más detalles.

Para más información sobre cómo crear y usar skills, consulta [Agent Skills in the SDK](/docs/en/agent-sdk/skills).

## Hooks

El SDK soporta dos formas de definir hooks, y se ejecutan en paralelo:

- **Hooks del sistema de archivos:** comandos de shell definidos en `settings.json`, cargados cuando `settingSources` incluye la fuente correspondiente. Son los mismos hooks que configurarías para [sesiones interactivas de Claude Code](https://code.claude.com/docs/en/hooks-guide).
- **Hooks programáticos:** funciones de callback pasadas directamente a `query()`. Se ejecutan en el proceso de tu aplicación y pueden devolver decisiones estructuradas. Consulta [Control execution with hooks](/docs/en/agent-sdk/hooks).

Ambos tipos se ejecutan durante el mismo ciclo de vida de hooks. Si ya tienes hooks en el `.claude/settings.json` de tu proyecto y configuras `settingSources: ["project"]`, esos hooks se ejecutarán automáticamente en el SDK sin configuración adicional.

Los callbacks de hooks reciben la entrada de la herramienta y devuelven un dict de decisión. Devolver `{}` (un dict vacío) significa permitir que la herramienta continúe. Devolver `{"decision": "block", "reason": "..."}` impide la ejecución y el motivo se envía a Claude como resultado de la herramienta. Consulta la [guía de hooks](/docs/en/agent-sdk/hooks) para conocer la firma completa del callback y los tipos de retorno.

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions, HookMatcher, ResultMessage


# PreToolUse hook callback. Positional args:
#   input_data: HookInput dict with tool_name, tool_input, hook_event_name
#   tool_use_id: str | None, the ID of the tool call being intercepted
#   context: HookContext, carries session metadata
async def audit_bash(input_data, tool_use_id, context):
    command = input_data.get("tool_input", {}).get("command", "")
    if "rm -rf" in command:
        return {"decision": "block", "reason": "Destructive command blocked"}
    return {}  # Empty dict: allow the tool to proceed


# Filesystem hooks from .claude/settings.json run automatically
# when settingSources loads them. You can also add programmatic hooks:
async for message in query(
    prompt="Refactor the auth module",
    options=ClaudeAgentOptions(
        setting_sources=["project"],  # Loads hooks from .claude/settings.json
        hooks={
            "PreToolUse": [
                HookMatcher(matcher="Bash", hooks=[audit_bash]),
            ]
        },
    ),
):
    if isinstance(message, ResultMessage) and message.subtype == "success":
        print(message.result)
```

**TypeScript**
```typescript
import { query, type HookInput, type HookJSONOutput } from "@anthropic-ai/claude-agent-sdk";

// PreToolUse hook callback. HookInput is a discriminated union on
// hook_event_name, so narrowing on it gives TypeScript the right
// tool_input shape for this event.
const auditBash = async (input: HookInput): Promise<HookJSONOutput> => {
  if (input.hook_event_name !== "PreToolUse") return {};
  const toolInput = input.tool_input as { command?: string };
  if (toolInput.command?.includes("rm -rf")) {
    return { decision: "block", reason: "Destructive command blocked" };
  }
  return {}; // Empty object: allow the tool to proceed
};

// Filesystem hooks from .claude/settings.json run automatically
// when settingSources loads them. You can also add programmatic hooks:
for await (const message of query({
  prompt: "Refactor the auth module",
  options: {
    settingSources: ["project"], // Loads hooks from .claude/settings.json
    hooks: {
      PreToolUse: [{ matcher: "Bash", hooks: [auditBash] }]
    }
  }
})) {
  if (message.type === "result" && message.subtype === "success") {
    console.log(message.result);
  }
}
```

### Cuándo usar cada tipo de hook

| Tipo de hook | Mejor para |
|:-------------|:-----------|
| **Del sistema de archivos** (`settings.json`) | Compartir hooks entre sesiones del CLI y del SDK. Soporta `"command"` (scripts de shell), `"http"` (POST a un endpoint), `"prompt"` (un LLM evalúa un prompt) y `"agent"` (lanza un agente verificador). Se disparan en el agente principal y en cualquier subagente que este lance. |
| **Programáticos** (callbacks en `query()`) | Lógica específica de la aplicación; devolver decisiones estructuradas; integración en el mismo proceso. Limitados a la sesión principal únicamente. |

> **Nota:** El SDK de TypeScript soporta eventos de hook adicionales más allá de Python, incluyendo `SessionStart`, `SessionEnd`, `TeammateIdle` y `TaskCompleted`. Consulta la [guía de hooks](/docs/en/agent-sdk/hooks) para la tabla completa de compatibilidad de eventos.

Para más detalles sobre hooks programáticos, consulta [Control execution with hooks](/docs/en/agent-sdk/hooks). Para la sintaxis de hooks del sistema de archivos, consulta [Hooks](https://code.claude.com/docs/en/hooks).

## Elegir la funcionalidad adecuada

El Agent SDK te da acceso a varias formas de extender el comportamiento de tu agente. Si no estás seguro de cuál usar, esta tabla relaciona los objetivos comunes con el enfoque correcto.

| Quieres... | Usa | Superficie del SDK |
|:-----------|:----|:-------------------|
| Establecer convenciones de proyecto que tu agente siempre siga | [CLAUDE.md](https://code.claude.com/docs/en/memory) | `settingSources: ["project"]` lo carga automáticamente |
| Darle al agente material de referencia que carga cuando es relevante | [Skills](/docs/en/agent-sdk/skills) | `settingSources` + `allowedTools: ["Skill"]` |
| Ejecutar un flujo de trabajo reutilizable (despliegue, revisión, release) | [Skills invocables por el usuario](/docs/en/agent-sdk/skills) | `settingSources` + `allowedTools: ["Skill"]` |
| Delegar una subtarea aislada a un contexto nuevo (investigación, revisión) | [Subagentes](/docs/en/agent-sdk/subagents) | parámetro `agents` + `allowedTools: ["Agent"]` |
| Coordinar múltiples instancias de Claude Code con listas de tareas compartidas y mensajería directa entre agentes | [Agent teams](https://code.claude.com/docs/en/agent-teams) | No se configura directamente mediante opciones del SDK. Los agent teams son una funcionalidad del CLI donde una sesión actúa como líder del equipo, coordinando el trabajo entre compañeros independientes |
| Ejecutar lógica determinista en llamadas a herramientas (auditoría, bloqueo, transformación) | [Hooks](/docs/en/agent-sdk/hooks) | parámetro `hooks` con callbacks, o scripts de shell cargados mediante `settingSources` |
| Darle a Claude acceso estructurado a herramientas de un servicio externo | [MCP](/docs/en/agent-sdk/mcp) | parámetro `mcpServers` |

> **Tip:** **Subagentes versus agent teams:** Los subagentes son efímeros y aislados: conversación nueva, una tarea, resumen devuelto al padre. Los agent teams coordinan múltiples instancias independientes de Claude Code que comparten una lista de tareas y se envían mensajes directamente entre sí. Los agent teams son una funcionalidad del CLI. Consulta [What subagents inherit](/docs/en/agent-sdk/subagents#what-subagents-inherit) y la [comparación con agent teams](https://code.claude.com/docs/en/agent-teams#compare-with-subagents) para más detalles.

Cada funcionalidad que habilitas se añade a la ventana de contexto de tu agente. Para conocer los costos por funcionalidad y cómo estas se combinan entre sí, consulta [Extend Claude Code](https://code.claude.com/docs/en/features-overview#understand-context-costs).

## Recursos relacionados

- [Extend Claude Code](https://code.claude.com/docs/en/features-overview): Visión conceptual de todas las funcionalidades de extensión, con tablas comparativas y análisis de costos de contexto
- [Skills in the SDK](/docs/en/agent-sdk/skills): Guía completa para usar skills de forma programática
- [Subagents](/docs/en/agent-sdk/subagents): Define e invoca subagentes para subtareas aisladas
- [Hooks](/docs/en/agent-sdk/hooks): Intercepta y controla el comportamiento del agente en puntos clave de ejecución
- [Permissions](/docs/en/agent-sdk/permissions): Controla el acceso a herramientas con modos, reglas y callbacks
- [System prompts](/docs/en/agent-sdk/modifying-system-prompts): Inyecta contexto sin usar archivos CLAUDE.md
