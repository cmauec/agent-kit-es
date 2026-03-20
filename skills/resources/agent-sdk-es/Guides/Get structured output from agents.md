# Obtener salidas estructuradas de agentes

Devuelve JSON validado desde flujos de trabajo de agentes usando JSON Schema, Zod o Pydantic. Obtén datos estructurados con seguridad de tipos después de un uso de herramientas en múltiples turnos.

---

Las salidas estructuradas te permiten definir la forma exacta de los datos que quieres recibir de un agente. El agente puede usar cualquier herramienta que necesite para completar la tarea, y aún así obtienes JSON validado que coincide con tu schema al final. Define un [JSON Schema](https://json-schema.org/understanding-json-schema/about) para la estructura que necesitas, y el SDK garantiza que la salida coincida con él.

Para seguridad de tipos completa, usa [Zod](#type-safe-schemas-with-zod-and-pydantic) (TypeScript) o [Pydantic](#type-safe-schemas-with-zod-and-pydantic) (Python) para definir tu schema y obtener objetos fuertemente tipados de vuelta.

## ¿Por qué salidas estructuradas?

Los agentes devuelven texto libre por defecto, lo cual funciona para el chat pero no cuando necesitas usar la salida de forma programática. Las salidas estructuradas te proporcionan datos tipados que puedes pasar directamente a la lógica de tu aplicación, base de datos o componentes de UI.

Considera una aplicación de recetas donde un agente busca en la web y trae recetas. Sin salidas estructuradas, obtienes texto libre que tendrías que parsear por tu cuenta. Con salidas estructuradas, defines la forma que quieres y obtienes datos tipados que puedes usar directamente en tu aplicación.

**Sin salidas estructuradas:**

```text
Here's a classic chocolate chip cookie recipe!

**Chocolate Chip Cookies**
Prep time: 15 minutes | Cook time: 10 minutes

Ingredients:
- 2 1/4 cups all-purpose flour
- 1 cup butter, softened
...
```

Para usar esto en tu aplicación, necesitarías parsear el título, convertir "15 minutes" a un número, separar los ingredientes de las instrucciones, y manejar formatos inconsistentes entre respuestas.

**Con salidas estructuradas:**

```json
{
  "name": "Chocolate Chip Cookies",
  "prep_time_minutes": 15,
  "cook_time_minutes": 10,
  "ingredients": [
    { "item": "all-purpose flour", "amount": 2.25, "unit": "cups" },
    { "item": "butter, softened", "amount": 1, "unit": "cup" }
  ],
  "steps": ["Preheat oven to 375°F", "Cream butter and sugar"]
}
```

Datos tipados que puedes usar directamente en tu UI.

## Inicio rápido

Para usar salidas estructuradas, define un [JSON Schema](https://json-schema.org/understanding-json-schema/about) que describa la forma de los datos que quieres, luego pásalo a `query()` mediante la opción `outputFormat` (TypeScript) o `output_format` (Python). Cuando el agente termina, el mensaje de resultado incluye un campo `structured_output` con datos validados que coinciden con tu schema.

El ejemplo a continuación le pide al agente que investigue Anthropic y devuelva el nombre de la empresa, el año de fundación y la sede como salida estructurada.

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Define the shape of data you want back
const schema = {
  type: "object",
  properties: {
    company_name: { type: "string" },
    founded_year: { type: "number" },
    headquarters: { type: "string" }
  },
  required: ["company_name"]
};

for await (const message of query({
  prompt: "Research Anthropic and provide key company information",
  options: {
    outputFormat: {
      type: "json_schema",
      schema: schema
    }
  }
})) {
  // The result message contains structured_output with validated data
  if (message.type === "result" && message.structured_output) {
    console.log(message.structured_output);
    // { company_name: "Anthropic", founded_year: 2021, headquarters: "San Francisco, CA" }
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

# Define the shape of data you want back
schema = {
    "type": "object",
    "properties": {
        "company_name": {"type": "string"},
        "founded_year": {"type": "number"},
        "headquarters": {"type": "string"},
    },
    "required": ["company_name"],
}


async def main():
    async for message in query(
        prompt="Research Anthropic and provide key company information",
        options=ClaudeAgentOptions(
            output_format={"type": "json_schema", "schema": schema}
        ),
    ):
        # The result message contains structured_output with validated data
        if isinstance(message, ResultMessage) and message.structured_output:
            print(message.structured_output)
            # {'company_name': 'Anthropic', 'founded_year': 2021, 'headquarters': 'San Francisco, CA'}


asyncio.run(main())
```

## Schemas con seguridad de tipos usando Zod y Pydantic

En lugar de escribir JSON Schema a mano, puedes usar [Zod](https://zod.dev/) (TypeScript) o [Pydantic](https://docs.pydantic.dev/latest/) (Python) para definir tu schema. Estas bibliotecas generan el JSON Schema por ti y te permiten parsear la respuesta en un objeto completamente tipado que puedes usar en todo tu código con autocompletado y verificación de tipos.

El ejemplo a continuación define un schema para un plan de implementación de funcionalidad con un resumen, lista de pasos (cada uno con nivel de complejidad) y riesgos potenciales. El agente planifica la funcionalidad y devuelve un objeto `FeaturePlan` tipado. Luego puedes acceder a propiedades como `plan.summary` e iterar sobre `plan.steps` con total seguridad de tipos.

**TypeScript**
```typescript
import { z } from "zod";
import { query } from "@anthropic-ai/claude-agent-sdk";

// Define schema with Zod
const FeaturePlan = z.object({
  feature_name: z.string(),
  summary: z.string(),
  steps: z.array(
    z.object({
      step_number: z.number(),
      description: z.string(),
      estimated_complexity: z.enum(["low", "medium", "high"])
    })
  ),
  risks: z.array(z.string())
});

type FeaturePlan = z.infer<typeof FeaturePlan>;

// Convert to JSON Schema
const schema = z.toJSONSchema(FeaturePlan);

// Use in query
for await (const message of query({
  prompt:
    "Plan how to add dark mode support to a React app. Break it into implementation steps.",
  options: {
    outputFormat: {
      type: "json_schema",
      schema: schema
    }
  }
})) {
  if (message.type === "result" && message.structured_output) {
    // Validate and get fully typed result
    const parsed = FeaturePlan.safeParse(message.structured_output);
    if (parsed.success) {
      const plan: FeaturePlan = parsed.data;
      console.log(`Feature: ${plan.feature_name}`);
      console.log(`Summary: ${plan.summary}`);
      plan.steps.forEach((step) => {
        console.log(`${step.step_number}. [${step.estimated_complexity}] ${step.description}`);
      });
    }
  }
}
```

**Python**
```python
import asyncio
from pydantic import BaseModel
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage


class Step(BaseModel):
    step_number: int
    description: str
    estimated_complexity: str  # 'low', 'medium', 'high'


class FeaturePlan(BaseModel):
    feature_name: str
    summary: str
    steps: list[Step]
    risks: list[str]


async def main():
    async for message in query(
        prompt="Plan how to add dark mode support to a React app. Break it into implementation steps.",
        options=ClaudeAgentOptions(
            output_format={
                "type": "json_schema",
                "schema": FeaturePlan.model_json_schema(),
            }
        ),
    ):
        if isinstance(message, ResultMessage) and message.structured_output:
            # Validate and get fully typed result
            plan = FeaturePlan.model_validate(message.structured_output)
            print(f"Feature: {plan.feature_name}")
            print(f"Summary: {plan.summary}")
            for step in plan.steps:
                print(
                    f"{step.step_number}. [{step.estimated_complexity}] {step.description}"
                )


asyncio.run(main())
```

**Beneficios:**
- Inferencia de tipos completa (TypeScript) y anotaciones de tipo (Python)
- Validación en tiempo de ejecución con `safeParse()` o `model_validate()`
- Mejores mensajes de error
- Schemas componibles y reutilizables

## Configuración del formato de salida

La opción `outputFormat` (TypeScript) o `output_format` (Python) acepta un objeto con:

- `type`: Establece en `"json_schema"` para salidas estructuradas
- `schema`: Un objeto [JSON Schema](https://json-schema.org/understanding-json-schema/about) que define la estructura de tu salida. Puedes generarlo desde un schema Zod con `z.toJSONSchema()` o desde un modelo Pydantic con `.model_json_schema()`

El SDK admite las características estándar de JSON Schema, incluyendo todos los tipos básicos (object, array, string, number, boolean, null), `enum`, `const`, `required`, objetos anidados y definiciones `$ref`. Para la lista completa de características admitidas y limitaciones, consulta [Limitaciones de JSON Schema](/docs/en/build-with-claude/structured-outputs#json-schema-limitations).

## Ejemplo: agente de seguimiento de TODOs

Este ejemplo demuestra cómo funcionan las salidas estructuradas con el uso de herramientas en múltiples pasos. El agente necesita encontrar comentarios TODO en el código base, luego buscar información de `git blame` para cada uno. Decide de forma autónoma qué herramientas usar (Grep para buscar, Bash para ejecutar comandos git) y combina los resultados en una única respuesta estructurada.

El schema incluye campos opcionales (`author` y `date`) ya que la información de `git blame` puede no estar disponible para todos los archivos. El agente completa lo que puede encontrar y omite el resto.

**TypeScript**
```typescript
import { query } from "@anthropic-ai/claude-agent-sdk";

// Define structure for TODO extraction
const todoSchema = {
  type: "object",
  properties: {
    todos: {
      type: "array",
      items: {
        type: "object",
        properties: {
          text: { type: "string" },
          file: { type: "string" },
          line: { type: "number" },
          author: { type: "string" },
          date: { type: "string" }
        },
        required: ["text", "file", "line"]
      }
    },
    total_count: { type: "number" }
  },
  required: ["todos", "total_count"]
};

// Agent uses Grep to find TODOs, Bash to get git blame info
for await (const message of query({
  prompt: "Find all TODO comments in this codebase and identify who added them",
  options: {
    outputFormat: {
      type: "json_schema",
      schema: todoSchema
    }
  }
})) {
  if (message.type === "result" && message.structured_output) {
    const data = message.structured_output;
    console.log(`Found ${data.total_count} TODOs`);
    data.todos.forEach((todo) => {
      console.log(`${todo.file}:${todo.line} - ${todo.text}`);
      if (todo.author) {
        console.log(`  Added by ${todo.author} on ${todo.date}`);
      }
    });
  }
}
```

**Python**
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

# Define structure for TODO extraction
todo_schema = {
    "type": "object",
    "properties": {
        "todos": {
            "type": "array",
            "items": {
                "type": "object",
                "properties": {
                    "text": {"type": "string"},
                    "file": {"type": "string"},
                    "line": {"type": "number"},
                    "author": {"type": "string"},
                    "date": {"type": "string"},
                },
                "required": ["text", "file", "line"],
            },
        },
        "total_count": {"type": "number"},
    },
    "required": ["todos", "total_count"],
}


async def main():
    # Agent uses Grep to find TODOs, Bash to get git blame info
    async for message in query(
        prompt="Find all TODO comments in this codebase and identify who added them",
        options=ClaudeAgentOptions(
            output_format={"type": "json_schema", "schema": todo_schema}
        ),
    ):
        if isinstance(message, ResultMessage) and message.structured_output:
            data = message.structured_output
            print(f"Found {data['total_count']} TODOs")
            for todo in data["todos"]:
                print(f"{todo['file']}:{todo['line']} - {todo['text']}")
                if "author" in todo:
                    print(f"  Added by {todo['author']} on {todo['date']}")


asyncio.run(main())
```

## Manejo de errores

La generación de salidas estructuradas puede fallar cuando el agente no puede producir JSON válido que coincida con tu schema. Esto ocurre típicamente cuando el schema es demasiado complejo para la tarea, la tarea en sí es ambigua, o el agente alcanza su límite de reintentos al intentar corregir errores de validación.

Cuando ocurre un error, el mensaje de resultado tiene un campo `subtype` que indica qué salió mal:

| Subtype | Significado |
|---------|---------|
| `success` | La salida fue generada y validada correctamente |
| `error_max_structured_output_retries` | El agente no pudo producir una salida válida después de múltiples intentos |

El ejemplo a continuación comprueba el campo `subtype` para determinar si la salida fue generada correctamente o si necesitas manejar un fallo:

**TypeScript**
```typescript
for await (const msg of query({
  prompt: "Extract contact info from the document",
  options: {
    outputFormat: {
      type: "json_schema",
      schema: contactSchema
    }
  }
})) {
  if (msg.type === "result") {
    if (msg.subtype === "success" && msg.structured_output) {
      // Use the validated output
      console.log(msg.structured_output);
    } else if (msg.subtype === "error_max_structured_output_retries") {
      // Handle the failure - retry with simpler prompt, fall back to unstructured, etc.
      console.error("Could not produce valid output");
    }
  }
}
```

**Python**
```python
async for message in query(
    prompt="Extract contact info from the document",
    options=ClaudeAgentOptions(
        output_format={"type": "json_schema", "schema": contact_schema}
    ),
):
    if isinstance(message, ResultMessage):
        if message.subtype == "success" and message.structured_output:
            # Use the validated output
            print(message.structured_output)
        elif message.subtype == "error_max_structured_output_retries":
            # Handle the failure
            print("Could not produce valid output")
```

**Consejos para evitar errores:**

- **Mantén los schemas enfocados.** Los schemas profundamente anidados con muchos campos requeridos son más difíciles de satisfacer. Empieza simple y añade complejidad según sea necesario.
- **Adapta el schema a la tarea.** Si la tarea puede no tener toda la información que tu schema requiere, haz que esos campos sean opcionales.
- **Usa prompts claros.** Los prompts ambiguos dificultan que el agente sepa qué salida producir.

## Recursos relacionados

- [Documentación de JSON Schema](https://json-schema.org/): aprende la sintaxis de JSON Schema para definir schemas complejos con objetos anidados, arrays, enums y restricciones de validación
- [Salidas estructuradas de la API](/docs/en/build-with-claude/structured-outputs): usa salidas estructuradas directamente con la API de Claude para solicitudes de un solo turno sin uso de herramientas
- [Herramientas personalizadas](/docs/en/agent-sdk/custom-tools): proporciona a tu agente herramientas personalizadas para llamar durante la ejecución antes de devolver la salida estructurada
