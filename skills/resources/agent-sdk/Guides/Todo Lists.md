# Todo Lists

Rastrea y muestra tareas pendientes usando el Claude Agent SDK para una gestión organizada de tareas

---

El seguimiento de tareas (todos) proporciona una manera estructurada de gestionar tareas y mostrar el progreso a los usuarios. El Claude Agent SDK incluye funcionalidad de todos integrada que ayuda a organizar flujos de trabajo complejos y mantener a los usuarios informados sobre el avance de las tareas.

### Ciclo de vida de un Todo

Los todos siguen un ciclo de vida predecible:
1. **Creado** como `pending` cuando se identifican las tareas
2. **Activado** a `in_progress` cuando comienza el trabajo
3. **Completado** cuando la tarea finaliza exitosamente
4. **Eliminado** cuando todas las tareas de un grupo se completan

### Cuándo se usan los Todos

El SDK crea todos automáticamente para:
- **Tareas complejas de múltiples pasos** que requieren 3 o más acciones distintas
- **Listas de tareas proporcionadas por el usuario** cuando se mencionan múltiples elementos
- **Operaciones no triviales** que se benefician del seguimiento de progreso
- **Solicitudes explícitas** cuando los usuarios piden organización mediante todos

## Ejemplos

### Monitorear cambios en los Todos

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

for await (const message of query({
  prompt: "Optimize my React app performance and track progress with todos",
  options: { maxTurns: 15 }
})) {
  // Las actualizaciones de todos se reflejan en el flujo de mensajes
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if (block.type === "tool_use" && block.name === "TodoWrite") {
        const todos = block.input.todos;

        console.log("Todo Status Update:");
        todos.forEach((todo, index) => {
          const status =
            todo.status === "completed" ? "✅" : todo.status === "in_progress" ? "🔧" : "❌";
          console.log(`${index + 1}. ${status} ${todo.content}`);
        });
      }
    }
  }
}
```

**Python**
```python
from claude_agent_sdk import query, AssistantMessage, ToolUseBlock

async for message in query(
    prompt="Optimize my React app performance and track progress with todos",
    options={"max_turns": 15},
):
    # Las actualizaciones de todos se reflejan en el flujo de mensajes
    if isinstance(message, AssistantMessage):
        for block in message.content:
            if isinstance(block, ToolUseBlock) and block.name == "TodoWrite":
                todos = block.input["todos"]

                print("Todo Status Update:")
                for i, todo in enumerate(todos):
                    status = (
                        "✅"
                        if todo["status"] == "completed"
                        else "🔧"
                        if todo["status"] == "in_progress"
                        else "❌"
                    )
                    print(f"{i + 1}. {status} {todo['content']}")
```

### Visualización del progreso en tiempo real

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

class TodoTracker {
  private todos: any[] = [];

  displayProgress() {
    if (this.todos.length === 0) return;

    const completed = this.todos.filter((t) => t.status === "completed").length;
    const inProgress = this.todos.filter((t) => t.status === "in_progress").length;
    const total = this.todos.length;

    console.log(`\nProgress: ${completed}/${total} completed`);
    console.log(`Currently working on: ${inProgress} task(s)\n`);

    this.todos.forEach((todo, index) => {
      const icon =
        todo.status === "completed" ? "✅" : todo.status === "in_progress" ? "🔧" : "❌";
      const text = todo.status === "in_progress" ? todo.activeForm : todo.content;
      console.log(`${index + 1}. ${icon} ${text}`);
    });
  }

  async trackQuery(prompt: string) {
    for await (const message of query({
      prompt,
      options: { maxTurns: 20 }
    })) {
      if (message.type === "assistant") {
        for (const block of message.message.content) {
          if (block.type === "tool_use" && block.name === "TodoWrite") {
            this.todos = block.input.todos;
            this.displayProgress();
          }
        }
      }
    }
  }
}

// Uso
const tracker = new TodoTracker();
await tracker.trackQuery("Build a complete authentication system with todos");
```

**Python**
```python
from claude_agent_sdk import query, AssistantMessage, ToolUseBlock
from typing import List, Dict


class TodoTracker:
    def __init__(self):
        self.todos: List[Dict] = []

    def display_progress(self):
        if not self.todos:
            return

        completed = len([t for t in self.todos if t["status"] == "completed"])
        in_progress = len([t for t in self.todos if t["status"] == "in_progress"])
        total = len(self.todos)

        print(f"\nProgress: {completed}/{total} completed")
        print(f"Currently working on: {in_progress} task(s)\n")

        for i, todo in enumerate(self.todos):
            icon = (
                "✅"
                if todo["status"] == "completed"
                else "🔧"
                if todo["status"] == "in_progress"
                else "❌"
            )
            text = (
                todo["activeForm"]
                if todo["status"] == "in_progress"
                else todo["content"]
            )
            print(f"{i + 1}. {icon} {text}")

    async def track_query(self, prompt: str):
        async for message in query(prompt=prompt, options={"max_turns": 20}):
            if isinstance(message, AssistantMessage):
                for block in message.content:
                    if isinstance(block, ToolUseBlock) and block.name == "TodoWrite":
                        self.todos = block.input["todos"]
                        self.display_progress()


# Uso
tracker = TodoTracker()
await tracker.track_query("Build a complete authentication system with todos")
```

## Documentación relacionada

- [TypeScript SDK Reference](/docs/en/agent-sdk/typescript)
- [Python SDK Reference](/docs/en/agent-sdk/python)
- [Streaming vs Single Mode](./Streaming%20Input.md)
- [Custom Tools](./Custom%20Tools.md)
