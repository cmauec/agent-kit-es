# Agent Skills en el SDK

Extiende Claude con capacidades especializadas usando Agent Skills en el Claude Agent SDK

---

## Descripción general

Agent Skills extienden Claude con capacidades especializadas que Claude invoca de forma autónoma cuando son relevantes. Los Skills se empaquetan como archivos `SKILL.md` que contienen instrucciones, descripciones y recursos de apoyo opcionales.

Para información completa sobre Skills, incluyendo beneficios, arquitectura y pautas de autoría, consulta la [descripción general de Agent Skills](/docs/en/agents-and-tools/agent-skills/overview).

## Cómo funcionan los Skills con el SDK

Cuando se usa el Claude Agent SDK, los Skills:

1. **Se definen como artefactos del sistema de archivos**: Se crean como archivos `SKILL.md` en directorios específicos (`.claude/skills/`)
2. **Se cargan desde el sistema de archivos**: Los Skills se cargan desde ubicaciones configuradas en el sistema de archivos. Debes especificar `settingSources` (TypeScript) o `setting_sources` (Python) para cargar Skills desde el sistema de archivos
3. **Se descubren automáticamente**: Una vez que se cargan las configuraciones del sistema de archivos, los metadatos de los Skills se descubren al inicio desde los directorios de usuario y proyecto; el contenido completo se carga cuando se activan
4. **Son invocados por el modelo**: Claude elige de forma autónoma cuándo usarlos según el contexto
5. **Se habilitan mediante allowed_tools**: Agrega `"Skill"` a tu `allowed_tools` para habilitar los Skills

A diferencia de los subagentes (que pueden definirse de forma programática), los Skills deben crearse como artefactos del sistema de archivos. El SDK no proporciona una API programática para registrar Skills.

> **Nota:** **Comportamiento predeterminado**: De forma predeterminada, el SDK no carga ninguna configuración del sistema de archivos. Para usar Skills, debes configurar explícitamente `settingSources: ['user', 'project']` (TypeScript) o `setting_sources=["user", "project"]` (Python) en tus opciones.

## Usar Skills con el SDK

Para usar Skills con el SDK, necesitas:

1. Incluir `"Skill"` en tu configuración de `allowed_tools`
2. Configurar `settingSources`/`setting_sources` para cargar Skills desde el sistema de archivos

Una vez configurado, Claude descubre automáticamente los Skills desde los directorios especificados y los invoca cuando son relevantes para la solicitud del usuario.

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions


async def main():
    options = ClaudeAgentOptions(
        cwd="/path/to/project",  # Project with .claude/skills/
        setting_sources=["user", "project"],  # Load Skills from filesystem
        allowed_tools=["Skill", "Read", "Write", "Bash"],  # Enable Skill tool
    )

    async for message in query(
        prompt="Help me process this PDF document", options=options
    ):
        print(message)


asyncio.run(main())
```

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Help me process this PDF document",
  options: {
    cwd: "/path/to/project", // Project with .claude/skills/
    settingSources: ["user", "project"], // Load Skills from filesystem
    allowedTools: ["Skill", "Read", "Write", "Bash"] // Enable Skill tool
  }
})) {
  console.log(message);
}
```

## Ubicaciones de los Skills

Los Skills se cargan desde directorios del sistema de archivos según tu configuración de `settingSources`/`setting_sources`:

- **Project Skills** (`.claude/skills/`): Compartidos con tu equipo via git - se cargan cuando `setting_sources` incluye `"project"`
- **User Skills** (`~/.claude/skills/`): Skills personales para todos los proyectos - se cargan cuando `setting_sources` incluye `"user"`
- **Plugin Skills**: Incluidos con los plugins de Claude Code instalados

## Crear Skills

Los Skills se definen como directorios que contienen un archivo `SKILL.md` con frontmatter YAML y contenido en Markdown. El campo `description` determina cuándo Claude invoca tu Skill.

**Ejemplo de estructura de directorios**:
```bash
.claude/skills/processing-pdfs/
└── SKILL.md
```

Para una guía completa sobre la creación de Skills, incluyendo la estructura de SKILL.md, Skills de múltiples archivos y ejemplos, consulta:
- [Agent Skills in Claude Code](https://code.claude.com/docs/en/skills): Guía completa con ejemplos
- [Agent Skills Best Practices](/docs/en/agents-and-tools/agent-skills/best-practices): Pautas de autoría y convenciones de nomenclatura

## Restricciones de herramientas

> **Nota:** El campo de frontmatter `allowed-tools` en SKILL.md solo está soportado cuando se usa el CLI de Claude Code directamente. **No aplica cuando se usan Skills a través del SDK**.
>
> Cuando uses el SDK, controla el acceso a las herramientas mediante la opción principal `allowedTools` en tu configuración de consulta.

Para controlar el acceso a herramientas de los Skills en aplicaciones del SDK, usa `allowedTools` para pre-aprobar herramientas específicas. Sin un callback `canUseTool`, todo lo que no esté en la lista se deniega:

> **Nota:** Se asume que las sentencias de importación del primer ejemplo están presentes en los siguientes fragmentos de código.

**Python**
```python
options = ClaudeAgentOptions(
    setting_sources=["user", "project"],  # Load Skills from filesystem
    allowed_tools=["Skill", "Read", "Grep", "Glob"],
)

async for message in query(prompt="Analyze the codebase structure", options=options):
    print(message)
```

**TypeScript**
```typescript
for await (const message of query({
  prompt: "Analyze the codebase structure",
  options: {
    settingSources: ["user", "project"], // Load Skills from filesystem
    allowedTools: ["Skill", "Read", "Grep", "Glob"],
    permissionMode: "dontAsk" // Deny anything not in allowedTools
  }
})) {
  console.log(message);
}
```

## Descubrir los Skills disponibles

Para ver qué Skills están disponibles en tu aplicación del SDK, simplemente pregúntale a Claude:

**Python**
```python
options = ClaudeAgentOptions(
    setting_sources=["user", "project"],  # Load Skills from filesystem
    allowed_tools=["Skill"],
)

async for message in query(prompt="What Skills are available?", options=options):
    print(message)
```

**TypeScript**
```typescript
for await (const message of query({
  prompt: "What Skills are available?",
  options: {
    settingSources: ["user", "project"], // Load Skills from filesystem
    allowedTools: ["Skill"]
  }
})) {
  console.log(message);
}
```

Claude listará los Skills disponibles según tu directorio de trabajo actual y los plugins instalados.

## Probar Skills

Prueba los Skills haciendo preguntas que coincidan con sus descripciones:

**Python**
```python
options = ClaudeAgentOptions(
    cwd="/path/to/project",
    setting_sources=["user", "project"],  # Load Skills from filesystem
    allowed_tools=["Skill", "Read", "Bash"],
)

async for message in query(prompt="Extract text from invoice.pdf", options=options):
    print(message)
```

**TypeScript**
```typescript
for await (const message of query({
  prompt: "Extract text from invoice.pdf",
  options: {
    cwd: "/path/to/project",
    settingSources: ["user", "project"], // Load Skills from filesystem
    allowedTools: ["Skill", "Read", "Bash"]
  }
})) {
  console.log(message);
}
```

Claude invoca automáticamente el Skill relevante si la descripción coincide con tu solicitud.

## Solución de problemas

### Skills no encontrados

**Verifica la configuración de settingSources**: Los Skills solo se cargan cuando configuras explícitamente `settingSources`/`setting_sources`. Este es el problema más común:

**Python**
```python
# Wrong - Skills won't be loaded
options = ClaudeAgentOptions(allowed_tools=["Skill"])

# Correct - Skills will be loaded
options = ClaudeAgentOptions(
    setting_sources=["user", "project"],  # Required to load Skills
    allowed_tools=["Skill"],
)
```

**TypeScript**
```typescript
// Wrong - Skills won't be loaded
const options = {
  allowedTools: ["Skill"]
};

// Correct - Skills will be loaded
const options = {
  settingSources: ["user", "project"], // Required to load Skills
  allowedTools: ["Skill"]
};
```

Para más detalles sobre `settingSources`/`setting_sources`, consulta la referencia del SDK de TypeScript o Python.

**Verifica el directorio de trabajo**: El SDK carga los Skills en relación con la opción `cwd`. Asegúrate de que apunte a un directorio que contenga `.claude/skills/`:

**Python**
```python
# Ensure your cwd points to the directory containing .claude/skills/
options = ClaudeAgentOptions(
    cwd="/path/to/project",  # Must contain .claude/skills/
    setting_sources=["user", "project"],  # Required to load Skills
    allowed_tools=["Skill"],
)
```

**TypeScript**
```typescript
// Ensure your cwd points to the directory containing .claude/skills/
const options = {
  cwd: "/path/to/project", // Must contain .claude/skills/
  settingSources: ["user", "project"], // Required to load Skills
  allowedTools: ["Skill"]
};
```

Consulta la sección "Usar Skills con el SDK" anterior para ver el patrón completo.

**Verifica la ubicación en el sistema de archivos**:
```bash
# Check project Skills
ls .claude/skills/*/SKILL.md

# Check personal Skills
ls ~/.claude/skills/*/SKILL.md
```

### Skill no se utiliza

**Verifica que la herramienta Skill esté habilitada**: Confirma que `"Skill"` esté en tu `allowedTools`.

**Revisa la descripción**: Asegúrate de que sea específica e incluya palabras clave relevantes. Consulta [Agent Skills Best Practices](/docs/en/agents-and-tools/agent-skills/best-practices#writing-effective-descriptions) para obtener orientación sobre cómo escribir descripciones efectivas.

### Solución de problemas adicionales

Para la solución de problemas generales de Skills (sintaxis YAML, depuración, etc.), consulta la [sección de solución de problemas de Skills de Claude Code](https://code.claude.com/docs/en/skills#troubleshooting).

## Documentación relacionada

### Guías de Skills
- [Agent Skills in Claude Code](https://code.claude.com/docs/en/skills): Guía completa de Skills con creación, ejemplos y solución de problemas
- [Agent Skills Overview](/docs/en/agents-and-tools/agent-skills/overview): Descripción conceptual, beneficios y arquitectura
- [Agent Skills Best Practices](/docs/en/agents-and-tools/agent-skills/best-practices): Pautas de autoría para Skills efectivos
- [Agent Skills Cookbook](https://platform.claude.com/cookbook/skills-notebooks-01-skills-introduction): Skills de ejemplo y plantillas

### Recursos del SDK
- [Subagents in the SDK](./Subagents%20in%20the%20SDK.md): Agentes similares basados en el sistema de archivos con opciones programáticas
- [Slash Commands in the SDK](./Slash%20Commands%20in%20the%20SDK.md): Comandos invocados por el usuario
- [SDK Overview](../Agent%20SDK%20overview.md): Conceptos generales del SDK
- [TypeScript SDK Reference](/docs/en/agent-sdk/typescript): Documentación completa de la API
- [Python SDK Reference](/docs/en/agent-sdk/python): Documentación completa de la API
