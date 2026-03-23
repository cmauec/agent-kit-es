# Plugins in the SDK

Carga plugins personalizados para extender Claude Code con comandos, agentes, skills y hooks a través del Agent SDK

---

Los plugins te permiten extender Claude Code con funcionalidad personalizada que puede compartirse entre proyectos. A través del Agent SDK, puedes cargar plugins programáticamente desde directorios locales para agregar comandos slash personalizados, agentes, skills, hooks y servidores MCP a tus sesiones de agente.

## ¿Qué son los plugins?

Los plugins son paquetes de extensiones de Claude Code que pueden incluir:
- **Skills**: Capacidades invocadas por el modelo que Claude usa de forma autónoma (también pueden invocarse con `/skill-name`)
- **Agents**: Subagentes especializados para tareas específicas
- **Hooks**: Manejadores de eventos que responden al uso de herramientas y otros eventos
- **MCP servers**: Integraciones de herramientas externas a través del Model Context Protocol

> **Nota:** El directorio `commands/` es un formato heredado. Usa `skills/` para nuevos plugins. Claude Code sigue siendo compatible con ambos formatos por retrocompatibilidad.

Para información completa sobre la estructura de plugins y cómo crearlos, consulta [Plugins](https://code.claude.com/docs/en/plugins).

## Carga de plugins

Carga plugins proporcionando sus rutas en el sistema de archivos local en tu configuración de opciones. El SDK admite cargar múltiples plugins desde diferentes ubicaciones.

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Hello",
  options: {
    plugins: [
      { type: "local", path: "./my-plugin" },
      { type: "local", path: "/absolute/path/to/another-plugin" }
    ]
  }
})) {
  // Plugin commands, agents, and other features are now available
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query


async def main():
    async for message in query(
        prompt="Hello",
        options={
            "plugins": [
                {"type": "local", "path": "./my-plugin"},
                {"type": "local", "path": "/absolute/path/to/another-plugin"},
            ]
        },
    ):
        # Plugin commands, agents, and other features are now available
        pass


asyncio.run(main())
```

### Especificación de rutas

Las rutas de los plugins pueden ser:
- **Rutas relativas**: Se resuelven relativas a tu directorio de trabajo actual (por ejemplo, `"./plugins/my-plugin"`)
- **Rutas absolutas**: Rutas completas en el sistema de archivos (por ejemplo, `"/home/user/plugins/my-plugin"`)

> **Nota:** La ruta debe apuntar al directorio raíz del plugin (el directorio que contiene `.claude-plugin/plugin.json`).

## Verificación de la instalación del plugin

Cuando los plugins se cargan correctamente, aparecen en el mensaje de inicialización del sistema. Puedes verificar que tus plugins estén disponibles:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Hello",
  options: {
    plugins: [{ type: "local", path: "./my-plugin" }]
  }
})) {
  if (message.type === "system" && message.subtype === "init") {
    // Check loaded plugins
    console.log("Plugins:", message.plugins);
    // Example: [{ name: "my-plugin", path: "./my-plugin" }]

    // Check available commands from plugins
    console.log("Commands:", message.slash_commands);
    // Example: ["/help", "/compact", "my-plugin:custom-command"]
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query


async def main():
    async for message in query(
        prompt="Hello", options={"plugins": [{"type": "local", "path": "./my-plugin"}]}
    ):
        if message.type == "system" and message.subtype == "init":
            # Check loaded plugins
            print("Plugins:", message.data.get("plugins"))
            # Example: [{"name": "my-plugin", "path": "./my-plugin"}]

            # Check available commands from plugins
            print("Commands:", message.data.get("slash_commands"))
            # Example: ["/help", "/compact", "my-plugin:custom-command"]


asyncio.run(main())
```

## Uso de skills de plugins

Las skills de los plugins se agrupan automáticamente bajo un espacio de nombres con el nombre del plugin para evitar conflictos. Cuando se invocan como comandos slash, el formato es `plugin-name:skill-name`.

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Load a plugin with a custom /greet skill
for await (const message of query({
  prompt: "/my-plugin:greet", // Use plugin skill with namespace
  options: {
    plugins: [{ type: "local", path: "./my-plugin" }]
  }
})) {
  // Claude executes the custom greeting skill from the plugin
  if (message.type === "assistant") {
    console.log(message.content);
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query, AssistantMessage, TextBlock


async def main():
    # Load a plugin with a custom /greet skill
    async for message in query(
        prompt="/demo-plugin:greet",  # Use plugin skill with namespace
        options={"plugins": [{"type": "local", "path": "./plugins/demo-plugin"}]},
    ):
        # Claude executes the custom greeting skill from the plugin
        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    print(f"Claude: {block.text}")


asyncio.run(main())
```

> **Nota:** Si instalaste un plugin a través del CLI (por ejemplo, `/plugin install my-plugin@marketplace`), puedes seguir usándolo en el SDK proporcionando su ruta de instalación. Revisa `~/.claude/plugins/` para los plugins instalados con el CLI.

## Ejemplo completo

A continuación se muestra un ejemplo completo que demuestra la carga y el uso de plugins:

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";
import * as path from "path";

async function runWithPlugin() {
  const pluginPath = path.join(__dirname, "plugins", "my-plugin");

  console.log("Loading plugin from:", pluginPath);

  for await (const message of query({
    prompt: "What custom commands do you have available?",
    options: {
      plugins: [{ type: "local", path: pluginPath }],
      maxTurns: 3
    }
  })) {
    if (message.type === "system" && message.subtype === "init") {
      console.log("Loaded plugins:", message.plugins);
      console.log("Available commands:", message.slash_commands);
    }

    if (message.type === "assistant") {
      console.log("Assistant:", message.content);
    }
  }
}

runWithPlugin().catch(console.error);
```

**Python**
```python
#!/usr/bin/env python3
"""Example demonstrating how to use plugins with the Agent SDK."""

from pathlib import Path
import anyio
from claude_agent_sdk import (
    AssistantMessage,
    ClaudeAgentOptions,
    TextBlock,
    query,
)


async def run_with_plugin():
    """Example using a custom plugin."""
    plugin_path = Path(__file__).parent / "plugins" / "demo-plugin"

    print(f"Loading plugin from: {plugin_path}")

    options = ClaudeAgentOptions(
        plugins=[{"type": "local", "path": str(plugin_path)}],
        max_turns=3,
    )

    async for message in query(
        prompt="What custom commands do you have available?", options=options
    ):
        if message.type == "system" and message.subtype == "init":
            print(f"Loaded plugins: {message.data.get('plugins')}")
            print(f"Available commands: {message.data.get('slash_commands')}")

        if isinstance(message, AssistantMessage):
            for block in message.content:
                if isinstance(block, TextBlock):
                    print(f"Assistant: {block.text}")


if __name__ == "__main__":
    anyio.run(run_with_plugin)
```

## Referencia de estructura de plugins

Un directorio de plugin debe contener un archivo de manifiesto `.claude-plugin/plugin.json`. Opcionalmente puede incluir:

```text
my-plugin/
├── .claude-plugin/
│   └── plugin.json          # Required: plugin manifest
├── skills/                   # Agent Skills (invoked autonomously or via /skill-name)
│   └── my-skill/
│       └── SKILL.md
├── commands/                 # Legacy: use skills/ instead
│   └── custom-cmd.md
├── agents/                   # Custom agents
│   └── specialist.md
├── hooks/                    # Event handlers
│   └── hooks.json
└── .mcp.json                # MCP server definitions
```

Para información detallada sobre la creación de plugins, consulta:
- [Plugins](https://code.claude.com/docs/en/plugins) - Guía completa de desarrollo de plugins
- [Plugins reference](https://code.claude.com/docs/en/plugins-reference) - Especificaciones técnicas y esquemas

## Casos de uso comunes

### Desarrollo y pruebas

Carga plugins durante el desarrollo sin instalarlos globalmente:

```typescript
plugins: [{ type: "local", path: "./dev-plugins/my-plugin" }];
```

### Extensiones específicas de proyecto

Incluye plugins en el repositorio de tu proyecto para consistencia en todo el equipo:

```typescript
plugins: [{ type: "local", path: "./project-plugins/team-workflows" }];
```

### Múltiples fuentes de plugins

Combina plugins de diferentes ubicaciones:

```typescript
plugins: [
  { type: "local", path: "./local-plugin" },
  { type: "local", path: "~/.claude/custom-plugins/shared-plugin" }
];
```

## Solución de problemas

### El plugin no se carga

Si tu plugin no aparece en el mensaje de init:

1. **Verifica la ruta**: Asegúrate de que la ruta apunte al directorio raíz del plugin (que contiene `.claude-plugin/`)
2. **Valida plugin.json**: Asegúrate de que tu archivo de manifiesto tenga sintaxis JSON válida
3. **Verifica los permisos de archivo**: Asegúrate de que el directorio del plugin sea legible

### Las skills no aparecen

Si las skills del plugin no funcionan:

1. **Usa el espacio de nombres**: Las skills de plugins requieren el formato `plugin-name:skill-name` cuando se invocan como comandos slash
2. **Revisa el mensaje de init**: Verifica que la skill aparezca en `slash_commands` con el espacio de nombres correcto
3. **Valida los archivos de skill**: Asegúrate de que cada skill tenga un archivo `SKILL.md` en su propio subdirectorio dentro de `skills/` (por ejemplo, `skills/my-skill/SKILL.md`)

### Problemas de resolución de rutas

Si las rutas relativas no funcionan:

1. **Verifica el directorio de trabajo**: Las rutas relativas se resuelven desde tu directorio de trabajo actual
2. **Usa rutas absolutas**: Para mayor confiabilidad, considera usar rutas absolutas
3. **Normaliza las rutas**: Usa utilidades de rutas para construir las rutas correctamente

## Ver también

- [Plugins](https://code.claude.com/docs/en/plugins) - Guía completa de desarrollo de plugins
- [Plugins reference](https://code.claude.com/docs/en/plugins-reference) - Especificaciones técnicas
- [Slash Commands](./Slash%20Commands%20in%20the%20SDK.md) - Uso de comandos slash en el SDK
- [Subagents](./Subagents%20in%20the%20SDK.md) - Trabajo con agentes especializados
- [Skills](./Agent%20Skills%20in%20the%20SDK.md) - Uso de Agent Skills
