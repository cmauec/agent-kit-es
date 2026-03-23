# Slash Commands en el SDK

Aprende a usar slash commands para controlar sesiones de Claude Code a través del SDK

---

Los slash commands proporcionan una forma de controlar las sesiones de Claude Code con comandos especiales que comienzan con `/`. Estos comandos pueden enviarse a través del SDK para realizar acciones como limpiar el historial de conversación, compactar mensajes u obtener ayuda.

## Descubrir los Slash Commands Disponibles

El Claude Agent SDK proporciona información sobre los slash commands disponibles en el mensaje de inicialización del sistema. Accede a esta información cuando tu sesión comienza:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Hello Claude",
  options: { maxTurns: 1 }
})) {
  if (message.type === "system" && message.subtype === "init") {
    console.log("Available slash commands:", message.slash_commands);
    // Example output: ["/compact", "/clear", "/help"]
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query


async def main():
    async for message in query(prompt="Hello Claude", options={"max_turns": 1}):
        if message.type == "system" and message.subtype == "init":
            print("Available slash commands:", message.slash_commands)
            # Example output: ["/compact", "/clear", "/help"]


asyncio.run(main())
```

## Enviar Slash Commands

Envía slash commands incluyéndolos en tu cadena de prompt, igual que el texto normal:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Send a slash command
for await (const message of query({
  prompt: "/compact",
  options: { maxTurns: 1 }
})) {
  if (message.type === "result") {
    console.log("Command executed:", message.result);
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query


async def main():
    # Send a slash command
    async for message in query(prompt="/compact", options={"max_turns": 1}):
        if message.type == "result":
            print("Command executed:", message.result)


asyncio.run(main())
```

## Slash Commands Comunes

### `/compact` - Compactar el Historial de Conversación

El comando `/compact` reduce el tamaño del historial de conversación resumiendo los mensajes más antiguos y preservando el contexto importante:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "/compact",
  options: { maxTurns: 1 }
})) {
  if (message.type === "system" && message.subtype === "compact_boundary") {
    console.log("Compaction completed");
    console.log("Pre-compaction tokens:", message.compact_metadata.pre_tokens);
    console.log("Trigger:", message.compact_metadata.trigger);
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query


async def main():
    async for message in query(prompt="/compact", options={"max_turns": 1}):
        if message.type == "system" and message.subtype == "compact_boundary":
            print("Compaction completed")
            print("Pre-compaction tokens:", message.compact_metadata.pre_tokens)
            print("Trigger:", message.compact_metadata.trigger)


asyncio.run(main())
```

### `/clear` - Limpiar la Conversación

El comando `/clear` inicia una conversación nueva eliminando todo el historial previo:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Clear conversation and start fresh
for await (const message of query({
  prompt: "/clear",
  options: { maxTurns: 1 }
})) {
  if (message.type === "system" && message.subtype === "init") {
    console.log("Conversation cleared, new session started");
    console.log("Session ID:", message.session_id);
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query


async def main():
    # Clear conversation and start fresh
    async for message in query(prompt="/clear", options={"max_turns": 1}):
        if message.type == "system" and message.subtype == "init":
            print("Conversation cleared, new session started")
            print("Session ID:", message.session_id)


asyncio.run(main())
```

## Crear Slash Commands Personalizados

Además de usar los slash commands integrados, puedes crear tus propios comandos personalizados que estén disponibles a través del SDK. Los comandos personalizados se definen como archivos markdown en directorios específicos, de manera similar a cómo se configuran los subagentes.

> **Nota:** El directorio `.claude/commands/` es el formato heredado. El formato recomendado es `.claude/skills/<name>/SKILL.md`, que admite la misma invocación por slash command (`/name`) además de la invocación autónoma por parte de Claude. Consulta [Skills](./Agent%20Skills%20in%20the%20SDK.md) para ver el formato actual. La CLI sigue admitiendo ambos formatos, y los ejemplos a continuación siguen siendo válidos para `.claude/commands/`.

### Ubicaciones de Archivos

Los slash commands personalizados se almacenan en directorios designados según su alcance:

- **Comandos de proyecto**: `.claude/commands/` - Disponibles solo en el proyecto actual (heredado; se prefiere `.claude/skills/`)
- **Comandos personales**: `~/.claude/commands/` - Disponibles en todos tus proyectos (heredado; se prefiere `~/.claude/skills/`)

### Formato de Archivo

Cada comando personalizado es un archivo markdown donde:
- El nombre del archivo (sin la extensión `.md`) se convierte en el nombre del comando
- El contenido del archivo define lo que hace el comando
- El frontmatter YAML opcional proporciona configuración

#### Ejemplo Básico

Crea `.claude/commands/refactor.md`:

```markdown
Refactor the selected code to improve readability and maintainability.
Focus on clean code principles and best practices.
```

Esto crea el comando `/refactor` que puedes usar a través del SDK.

#### Con Frontmatter

Crea `.claude/commands/security-check.md`:

```markdown
---
allowed-tools: Read, Grep, Glob
description: Run security vulnerability scan
model: claude-opus-4-6
---

Analyze the codebase for security vulnerabilities including:
- SQL injection risks
- XSS vulnerabilities
- Exposed credentials
- Insecure configurations
```

### Usar Comandos Personalizados en el SDK

Una vez definidos en el sistema de archivos, los comandos personalizados están disponibles automáticamente a través del SDK:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Use a custom command
for await (const message of query({
  prompt: "/refactor src/auth/login.ts",
  options: { maxTurns: 3 }
})) {
  if (message.type === "assistant") {
    console.log("Refactoring suggestions:", message.message);
  }
}

// Custom commands appear in the slash_commands list
for await (const message of query({
  prompt: "Hello",
  options: { maxTurns: 1 }
})) {
  if (message.type === "system" && message.subtype === "init") {
    // Will include both built-in and custom commands
    console.log("Available commands:", message.slash_commands);
    // Example: ["/compact", "/clear", "/help", "/refactor", "/security-check"]
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query


async def main():
    # Use a custom command
    async for message in query(
        prompt="/refactor src/auth/login.py", options={"max_turns": 3}
    ):
        if message.type == "assistant":
            print("Refactoring suggestions:", message.message)

    # Custom commands appear in the slash_commands list
    async for message in query(prompt="Hello", options={"max_turns": 1}):
        if message.type == "system" and message.subtype == "init":
            # Will include both built-in and custom commands
            print("Available commands:", message.slash_commands)
            # Example: ["/compact", "/clear", "/help", "/refactor", "/security-check"]


asyncio.run(main())
```

### Funciones Avanzadas

#### Argumentos y Marcadores de Posición

Los comandos personalizados admiten argumentos dinámicos mediante marcadores de posición:

Crea `.claude/commands/fix-issue.md`:

```markdown
---
argument-hint: [issue-number] [priority]
description: Fix a GitHub issue
---

Fix issue #$1 with priority $2.
Check the issue description and implement the necessary changes.
```

Úsalo en el SDK:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Pass arguments to custom command
for await (const message of query({
  prompt: "/fix-issue 123 high",
  options: { maxTurns: 5 }
})) {
  // Command will process with $1="123" and $2="high"
  if (message.type === "result") {
    console.log("Issue fixed:", message.result);
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query


async def main():
    # Pass arguments to custom command
    async for message in query(prompt="/fix-issue 123 high", options={"max_turns": 5}):
        # Command will process with $1="123" and $2="high"
        if message.type == "result":
            print("Issue fixed:", message.result)


asyncio.run(main())
```

#### Ejecución de Comandos Bash

Los comandos personalizados pueden ejecutar comandos bash e incluir su salida:

Crea `.claude/commands/git-commit.md`:

```markdown
---
allowed-tools: Bash(git add:*), Bash(git status:*), Bash(git commit:*)
description: Create a git commit
---

## Context

- Current status: !`git status`
- Current diff: !`git diff HEAD`

## Task

Create a git commit with appropriate message based on the changes.
```

#### Referencias a Archivos

Incluye el contenido de archivos usando el prefijo `@`:

Crea `.claude/commands/review-config.md`:

```markdown
---
description: Review configuration files
---

Review the following configuration files for issues:
- Package config: @package.json
- TypeScript config: @tsconfig.json
- Environment config: @.env

Check for security issues, outdated dependencies, and misconfigurations.
```

### Organización con Espacios de Nombres

Organiza los comandos en subdirectorios para una mejor estructura:

```bash
.claude/commands/
├── frontend/
│   ├── component.md      # Creates /component (project:frontend)
│   └── style-check.md     # Creates /style-check (project:frontend)
├── backend/
│   ├── api-test.md        # Creates /api-test (project:backend)
│   └── db-migrate.md      # Creates /db-migrate (project:backend)
└── review.md              # Creates /review (project)
```

El subdirectorio aparece en la descripción del comando, pero no afecta al nombre del comando en sí.

### Ejemplos Prácticos

#### Comando de Revisión de Código

Crea `.claude/commands/code-review.md`:

```markdown
---
allowed-tools: Read, Grep, Glob, Bash(git diff:*)
description: Comprehensive code review
---

## Changed Files
!`git diff --name-only HEAD~1`

## Detailed Changes
!`git diff HEAD~1`

## Review Checklist

Review the above changes for:
1. Code quality and readability
2. Security vulnerabilities
3. Performance implications
4. Test coverage
5. Documentation completeness

Provide specific, actionable feedback organized by priority.
```

#### Comando de Ejecución de Tests

Crea `.claude/commands/test.md`:

```markdown
---
allowed-tools: Bash, Read, Edit
argument-hint: [test-pattern]
description: Run tests with optional pattern
---

Run tests matching pattern: $ARGUMENTS

1. Detect the test framework (Jest, pytest, etc.)
2. Run tests with the provided pattern
3. If tests fail, analyze and fix them
4. Re-run to verify fixes
```

Usa estos comandos a través del SDK:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Run code review
for await (const message of query({
  prompt: "/code-review",
  options: { maxTurns: 3 }
})) {
  // Process review feedback
}

// Run specific tests
for await (const message of query({
  prompt: "/test auth",
  options: { maxTurns: 5 }
})) {
  // Handle test results
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query


async def main():
    # Run code review
    async for message in query(prompt="/code-review", options={"max_turns": 3}):
        # Process review feedback
        pass

    # Run specific tests
    async for message in query(prompt="/test auth", options={"max_turns": 5}):
        # Handle test results
        pass


asyncio.run(main())
```

## Ver También

- [Slash Commands](https://code.claude.com/docs/en/slash-commands) - Documentación completa de slash commands
- [Subagents in the SDK](./Subagents%20in%20the%20SDK.md) - Configuración basada en el sistema de archivos para subagentes, similar a los slash commands
- [TypeScript SDK reference](/docs/en/agent-sdk/typescript) - Documentación completa de la API
- [SDK overview](../Agent%20SDK%20overview.md) - Conceptos generales del SDK
- [CLI reference](https://code.claude.com/docs/en/cli-reference) - Interfaz de línea de comandos
