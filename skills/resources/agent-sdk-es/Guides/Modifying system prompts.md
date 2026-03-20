# Modificar system prompts

Aprende a personalizar el comportamiento de Claude modificando los system prompts mediante tres enfoques: output styles, systemPrompt con append y system prompts personalizados.

---

Los system prompts definen el comportamiento, las capacidades y el estilo de respuesta de Claude. El Claude Agent SDK ofrece tres formas de personalizar los system prompts: usando output styles (configuraciones persistentes basadas en archivos), añadiendo contenido al prompt de Claude Code, o usando un prompt completamente personalizado.

## Entendiendo los system prompts

Un system prompt es el conjunto de instrucciones inicial que define cómo se comporta Claude a lo largo de una conversación.

> **Nota:** **Comportamiento predeterminado:** El Agent SDK utiliza un **system prompt mínimo** por defecto. Solo contiene instrucciones esenciales para las herramientas, pero omite las directrices de codificación de Claude Code, el estilo de respuesta y el contexto del proyecto. Para incluir el system prompt completo de Claude Code, especifica `systemPrompt: { preset: "claude_code" }` en TypeScript o `system_prompt={"type": "preset", "preset": "claude_code"}` en Python.

El system prompt de Claude Code incluye:

- Instrucciones de uso de herramientas y herramientas disponibles
- Directrices de estilo y formato de código
- Configuración del tono y nivel de detalle de las respuestas
- Instrucciones de seguridad
- Contexto sobre el directorio de trabajo actual y el entorno

## Métodos de modificación

### Método 1: Archivos CLAUDE.md (instrucciones a nivel de proyecto)

Los archivos CLAUDE.md proporcionan contexto e instrucciones específicas del proyecto que el Agent SDK lee automáticamente cuando se ejecuta en un directorio. Sirven como "memoria" persistente para tu proyecto.

#### Cómo funciona CLAUDE.md con el SDK

**Ubicación y descubrimiento:**

- **Nivel de proyecto:** `CLAUDE.md` o `.claude/CLAUDE.md` en tu directorio de trabajo
- **Nivel de usuario:** `~/.claude/CLAUDE.md` para instrucciones globales aplicables a todos los proyectos

**IMPORTANTE:** El SDK solo lee los archivos CLAUDE.md cuando configuras explícitamente `settingSources` (TypeScript) o `setting_sources` (Python):

- Incluye `'project'` para cargar el CLAUDE.md del proyecto
- Incluye `'user'` para cargar el CLAUDE.md del usuario (`~/.claude/CLAUDE.md`)

El preset `claude_code` del system prompt NO carga automáticamente CLAUDE.md; también debes especificar las fuentes de configuración.

**Formato del contenido:**
Los archivos CLAUDE.md usan markdown simple y pueden contener:

- Directrices y estándares de codificación
- Contexto específico del proyecto
- Comandos o flujos de trabajo habituales
- Convenciones de API
- Requisitos de pruebas

#### Ejemplo de CLAUDE.md

```markdown
# Project Guidelines

## Code Style

- Use TypeScript strict mode
- Prefer functional components in React
- Always include JSDoc comments for public APIs

## Testing

- Run `npm test` before committing
- Maintain >80% code coverage
- Use jest for unit tests, playwright for E2E

## Commands

- Build: `npm run build`
- Dev server: `npm run dev`
- Type check: `npm run typecheck`
```

#### Usar CLAUDE.md con el SDK

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// IMPORTANT: You must specify settingSources to load CLAUDE.md
// The claude_code preset alone does NOT load CLAUDE.md files
const messages = [];

for await (const message of query({
  prompt: "Add a new React component for user profiles",
  options: {
    systemPrompt: {
      type: "preset",
      preset: "claude_code" // Use Claude Code's system prompt
    },
    settingSources: ["project"] // Required to load CLAUDE.md from project
  }
})) {
  messages.push(message);
}

// Now Claude has access to your project guidelines from CLAUDE.md
```

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions

# IMPORTANT: You must specify setting_sources to load CLAUDE.md
# The claude_code preset alone does NOT load CLAUDE.md files
messages = []

async for message in query(
    prompt="Add a new React component for user profiles",
    options=ClaudeAgentOptions(
        system_prompt={
            "type": "preset",
            "preset": "claude_code",  # Use Claude Code's system prompt
        },
        setting_sources=["project"],  # Required to load CLAUDE.md from project
    ),
):
    messages.append(message)

# Now Claude has access to your project guidelines from CLAUDE.md
```

#### Cuándo usar CLAUDE.md

**Ideal para:**

- **Contexto compartido con el equipo** - Directrices que todos deberían seguir
- **Convenciones del proyecto** - Estándares de codificación, estructura de archivos, patrones de nomenclatura
- **Comandos habituales** - Comandos de build, pruebas y despliegue específicos del proyecto
- **Memoria a largo plazo** - Contexto que debe persistir en todas las sesiones
- **Instrucciones versionadas** - Commiteadas en git para que el equipo esté sincronizado

**Características clave:**

- Persistente en todas las sesiones de un proyecto
- Compartida con el equipo vía git
- Descubrimiento automático (sin cambios en el código)
- Requiere cargar la configuración mediante `settingSources`

### Método 2: Output styles (configuraciones persistentes)

Los output styles son configuraciones guardadas que modifican el system prompt de Claude. Se almacenan como archivos markdown y pueden reutilizarse entre sesiones y proyectos.

#### Crear un output style

**TypeScript**
```typescript
import { writeFile, mkdir } from "fs/promises";
import { join } from "path";
import { homedir } from "os";

async function createOutputStyle(name: string, description: string, prompt: string) {
  // User-level: ~/.claude/output-styles
  // Project-level: .claude/output-styles
  const outputStylesDir = join(homedir(), ".claude", "output-styles");

  await mkdir(outputStylesDir, { recursive: true });

  const content = `---
name: ${name}
description: ${description}
---

${prompt}`;

  const filePath = join(outputStylesDir, `${name.toLowerCase().replace(/\s+/g, "-")}.md`);
  await writeFile(filePath, content, "utf-8");
}

// Example: Create a code review specialist
await createOutputStyle(
  "Code Reviewer",
  "Thorough code review assistant",
  `You are an expert code reviewer.

For every code submission:
1. Check for bugs and security issues
2. Evaluate performance
3. Suggest improvements
4. Rate code quality (1-10)`
);
```

**Python**
```python
from pathlib import Path


async def create_output_style(name: str, description: str, prompt: str):
    # User-level: ~/.claude/output-styles
    # Project-level: .claude/output-styles
    output_styles_dir = Path.home() / ".claude" / "output-styles"

    output_styles_dir.mkdir(parents=True, exist_ok=True)

    content = f"""---
name: {name}
description: {description}
---

{prompt}"""

    file_name = name.lower().replace(" ", "-") + ".md"
    file_path = output_styles_dir / file_name
    file_path.write_text(content, encoding="utf-8")


# Example: Create a code review specialist
await create_output_style(
    "Code Reviewer",
    "Thorough code review assistant",
    """You are an expert code reviewer.

For every code submission:
1. Check for bugs and security issues
2. Evaluate performance
3. Suggest improvements
4. Rate code quality (1-10)""",
)
```

#### Usar output styles

Una vez creados, activa los output styles mediante:

- **CLI**: `/output-style [style-name]`
- **Settings**: `.claude/settings.local.json`
- **Crear nuevo**: `/output-style:new [description]`

**Nota para usuarios del SDK:** Los output styles se cargan cuando incluyes `settingSources: ['user']` o `settingSources: ['project']` (TypeScript) / `setting_sources=["user"]` o `setting_sources=["project"]` (Python) en tus opciones.

### Método 3: Usar `systemPrompt` con append

Puedes usar el preset de Claude Code con una propiedad `append` para añadir tus instrucciones personalizadas preservando toda la funcionalidad integrada.

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const messages = [];

for await (const message of query({
  prompt: "Help me write a Python function to calculate fibonacci numbers",
  options: {
    systemPrompt: {
      type: "preset",
      preset: "claude_code",
      append: "Always include detailed docstrings and type hints in Python code."
    }
  }
})) {
  messages.push(message);
  if (message.type === "assistant") {
    console.log(message.message.content);
  }
}
```

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions

messages = []

async for message in query(
    prompt="Help me write a Python function to calculate fibonacci numbers",
    options=ClaudeAgentOptions(
        system_prompt={
            "type": "preset",
            "preset": "claude_code",
            "append": "Always include detailed docstrings and type hints in Python code.",
        }
    ),
):
    messages.append(message)
    if message.type == "assistant":
        print(message.message.content)
```

### Método 4: System prompts personalizados

Puedes proporcionar una cadena de texto personalizada como `systemPrompt` para reemplazar completamente el prompt predeterminado con tus propias instrucciones.

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

const customPrompt = `You are a Python coding specialist.
Follow these guidelines:
- Write clean, well-documented code
- Use type hints for all functions
- Include comprehensive docstrings
- Prefer functional programming patterns when appropriate
- Always explain your code choices`;

const messages = [];

for await (const message of query({
  prompt: "Create a data processing pipeline",
  options: {
    systemPrompt: customPrompt
  }
})) {
  messages.push(message);
  if (message.type === "assistant") {
    console.log(message.message.content);
  }
}
```

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions

custom_prompt = """You are a Python coding specialist.
Follow these guidelines:
- Write clean, well-documented code
- Use type hints for all functions
- Include comprehensive docstrings
- Prefer functional programming patterns when appropriate
- Always explain your code choices"""

messages = []

async for message in query(
    prompt="Create a data processing pipeline",
    options=ClaudeAgentOptions(system_prompt=custom_prompt),
):
    messages.append(message)
    if message.type == "assistant":
        print(message.message.content)
```

## Comparación de los cuatro enfoques

| Característica          | CLAUDE.md              | Output Styles          | `systemPrompt` con append  | `systemPrompt` personalizado |
| ----------------------- | ---------------------- | ---------------------- | -------------------------- | ---------------------------- |
| **Persistencia**        | Archivo por proyecto   | Guardado como archivos | Solo sesión                | Solo sesión                  |
| **Reutilización**       | Por proyecto           | Entre proyectos        | Duplicación en código      | Duplicación en código        |
| **Gestión**             | En el sistema de archivos | CLI + archivos      | En el código               | En el código                 |
| **Herramientas predeterminadas** | Preservadas | Preservadas            | Preservadas                | Perdidas (salvo que se incluyan) |
| **Seguridad integrada** | Mantenida              | Mantenida              | Mantenida                  | Debe añadirse                |
| **Contexto del entorno** | Automático            | Automático             | Automático                 | Debe proporcionarse          |
| **Nivel de personalización** | Solo adiciones   | Reemplaza el predeterminado | Solo adiciones        | Control total                |
| **Control de versiones** | Con el proyecto       | Sí                     | Con el código              | Con el código                |
| **Alcance**             | Específico del proyecto | Usuario o proyecto    | Sesión de código           | Sesión de código             |

**Nota:** "Con append" significa usar `systemPrompt: { type: "preset", preset: "claude_code", append: "..." }` en TypeScript o `system_prompt={"type": "preset", "preset": "claude_code", "append": "..."}` en Python.

## Casos de uso y buenas prácticas

### Cuándo usar CLAUDE.md

**Ideal para:**

- Estándares y convenciones de codificación específicos del proyecto
- Documentar la estructura y arquitectura del proyecto
- Listar comandos habituales (build, test, deploy)
- Contexto compartido con el equipo que debe estar bajo control de versiones
- Instrucciones que aplican a todo el uso del SDK en un proyecto

**Ejemplos:**

- "Todos los endpoints de API deben usar patrones async/await"
- "Ejecutar `npm run lint:fix` antes de commitear"
- "Las migraciones de base de datos están en el directorio `migrations/`"

**Importante:** Para cargar los archivos CLAUDE.md, debes configurar explícitamente `settingSources: ['project']` (TypeScript) o `setting_sources=["project"]` (Python). El preset `claude_code` del system prompt NO carga automáticamente CLAUDE.md sin esta configuración.

### Cuándo usar output styles

**Ideal para:**

- Cambios de comportamiento persistentes entre sesiones
- Configuraciones compartidas con el equipo
- Asistentes especializados (revisor de código, científico de datos, DevOps)
- Modificaciones complejas del prompt que necesitan versionado

**Ejemplos:**

- Crear un asistente dedicado a la optimización de SQL
- Construir un revisor de código enfocado en seguridad
- Desarrollar un asistente de enseñanza con una pedagogía específica

### Cuándo usar `systemPrompt` con append

**Ideal para:**

- Añadir estándares de codificación o preferencias específicas
- Personalizar el formato de la salida
- Añadir conocimiento específico del dominio
- Modificar el nivel de detalle de las respuestas
- Mejorar el comportamiento predeterminado de Claude Code sin perder las instrucciones de herramientas

### Cuándo usar `systemPrompt` personalizado

**Ideal para:**

- Control total sobre el comportamiento de Claude
- Tareas especializadas de una sola sesión
- Probar nuevas estrategias de prompt
- Situaciones en las que no se necesitan las herramientas predeterminadas
- Construir agentes especializados con comportamiento único

## Combinar enfoques

Puedes combinar estos métodos para obtener la máxima flexibilidad:

### Ejemplo: Output style con adiciones específicas de sesión

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Assuming "Code Reviewer" output style is active (via /output-style)
// Add session-specific focus areas
const messages = [];

for await (const message of query({
  prompt: "Review this authentication module",
  options: {
    systemPrompt: {
      type: "preset",
      preset: "claude_code",
      append: `
        For this review, prioritize:
        - OAuth 2.0 compliance
        - Token storage security
        - Session management
      `
    }
  }
})) {
  messages.push(message);
}
```

**Python**
```python
from claude_agent_sdk import query, ClaudeAgentOptions

# Assuming "Code Reviewer" output style is active (via /output-style)
# Add session-specific focus areas
messages = []

async for message in query(
    prompt="Review this authentication module",
    options=ClaudeAgentOptions(
        system_prompt={
            "type": "preset",
            "preset": "claude_code",
            "append": """
            For this review, prioritize:
            - OAuth 2.0 compliance
            - Token storage security
            - Session management
            """,
        }
    ),
):
    messages.append(message)
```

## Ver también

- [Output styles](https://code.claude.com/docs/en/output-styles) - Documentación completa de output styles
- [TypeScript SDK guide](/docs/en/agent-sdk/typescript) - Guía completa de uso del SDK
- [Configuration guide](https://code.claude.com/docs/en/settings) - Opciones generales de configuración
