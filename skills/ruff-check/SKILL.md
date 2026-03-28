---
name: ruff-check
description: Ejecuta `ruff check --select ALL` sobre archivos o directorios Python para un análisis completo de calidad de código. Usa esta skill cuando el usuario escriba `/ruff-check`, quiera revisar código Python con ruff, pida un análisis de calidad, linting, o quiera ver qué reglas viola su código. También úsala si el usuario menciona "revisar con ruff", "ruff con todas las reglas", o "analizar mi código Python". También úsala cuando el usuario pida ver todas las reglas disponibles, quiera saber qué puede ignorar, o describa en lenguaje natural lo que quiere omitir del análisis.
---

## Modos de uso

La skill tiene tres modos. Determina cuál aplica según los argumentos:

### Modo 1: Ver todas las reglas disponibles
Si el usuario escribe `/ruff-check --rules` o pide "muéstrame las reglas" / "qué reglas hay":

Ejecuta `ruff rule --all --output-format json` y presenta una tabla organizada por categoría con código, nombre y descripción breve. Agrupa así:

| Prefijo | Categoría |
|---------|-----------|
| `E`/`W` | Estilo PEP 8 |
| `F`     | Errores lógicos (pyflakes) |
| `N`     | Nombres |
| `D`     | Docstrings |
| `ANN`   | Type annotations |
| `UP`    | Modernización Python |
| `B`     | Bugbear (errores comunes) |
| `C90`   | Complejidad ciclomática |
| `I`     | Imports (isort) |
| `T`     | Print / debugging |
| `PLR/PLC/PLE` | Pylint |
| `S`     | Seguridad (bandit) |
| `RUF`   | Reglas propias de ruff |

Muestra máximo 10 reglas por categoría para no saturar. Al final di: "Hay N reglas en total. Dime en lenguaje natural qué quieres ignorar y te sugiero los códigos."

### Modo 2: Traducir lenguaje natural a códigos de reglas
Si el usuario describe algo que quiere ignorar en lenguaje natural (sin usar códigos), por ejemplo:
- "no me importan los prints"
- "ignora los docstrings"
- "no quiero revisar nombres de variables"
- "ignora líneas largas y anotaciones de tipo"

Interpreta la intención y responde con una tabla de sugerencias:

| Lo que dijiste | Reglas sugeridas | Por qué |
|----------------|-----------------|---------|
| "prints" | `T201`, `T203` | `print()` y `pprint()` |
| "docstrings" | `D1xx`, `D2xx`, `D3xx`, `D4xx` | Todas las reglas de docstrings |
| "nombres" | `N801`–`N818` | Convenciones de nombres |
| "líneas largas" | `E501` | Longitud de línea |
| "type annotations" | `ANN001`–`ANN401` | Anotaciones de tipo |

Luego pregunta: "¿Quieres que ejecute el análisis ignorando estas reglas? ¿O ajustas alguna?"

Si confirma, ejecuta con `--ignore` usando los códigos sugeridos.

### Modo 3: Ejecutar el análisis (modo principal)
Si el usuario pasa un archivo/directorio o simplemente invoca la skill sin los modos anteriores.

El usuario puede invocar con distintas combinaciones:

- `/ruff-check` → revisa el directorio actual (`.`)
- `/ruff-check agents/parallelization.py` → revisa un archivo específico
- `/ruff-check src/` → revisa un directorio
- `/ruff-check --ignore T201` → revisa `.` ignorando la regla T201
- `/ruff-check agents/parallelization.py --ignore T201,D205` → archivo + reglas ignoradas

Extrae dos cosas de los argumentos:
1. **Target**: el archivo o directorio a revisar. Si no se especifica, usa `.`
2. **Ignore**: lista de reglas separadas por coma tras `--ignore`. Si no se especifica, no añadas el flag.

## Construir y ejecutar el comando (Modo 3)

Arma el comando así:

```
ruff check --select ALL [--ignore REGLAS] TARGET
```

Ejemplos:
- `ruff check --select ALL .`
- `ruff check --select ALL agents/parallelization.py`
- `ruff check --select ALL --ignore T201,D205 agents/parallelization.py`

Ejecútalo con la herramienta Bash desde el directorio de trabajo actual.

> Nota: ruff puede emitir warnings sobre reglas incompatibles (ej. D203 vs D211). Eso es normal — no es un error.

## Presentar los resultados

### Si no hay errores:
Muestra un mensaje de éxito:
> ✅ Sin errores — el código pasa todas las reglas de ruff.

### Si hay errores:

Muestra el output completo de ruff tal como aparece (con el contexto visual de líneas y flechas que ruff genera).

Luego añade un reporte detallado de issues con archivo y línea:

```
## Reporte de issues

| Archivo | Línea | Código | Descripción |
|---------|-------|--------|-------------|
| agents/parallelization.py | 18 | F401 | `AssistantMessage` imported but unused |
| agents/parallelization.py | 23 | E501 | Line too long (98 > 88) |
| ...     | ...   | ...    | ... |
```

Luego añade un resumen agrupado por categoría:

```
## Resumen por categoría

| Código | Cantidad | Descripción |
|--------|----------|-------------|
| F401   | 2        | Imports no usados |
| E501   | 8        | Líneas demasiado largas |
| ...    | ...      | ... |

Total: N errores
```

### Si hay errores fixables:

Al final, sugiere los comandos de corrección:

```
## Corrección automática disponible

# Fixes seguros:
ruff check --select ALL --fix [--ignore REGLAS] TARGET

# Fixes adicionales (revisar antes de aplicar):
ruff check --select ALL --unsafe-fixes [--ignore REGLAS] TARGET
```

Solo muestra la línea de `--unsafe-fixes` si ruff reportó fixes disponibles con esa opción.

## Referencia rápida de categorías comunes

Si el usuario no conoce los códigos, estos son los más frecuentes:

| Prefijo | Categoría |
|---------|-----------|
| `E`/`W` | Estilo PEP 8 (pycodestyle) |
| `F`     | Errores lógicos (pyflakes) |
| `N`     | Convenciones de nombres |
| `D`     | Docstrings (pydocstyle) |
| `ANN`   | Type annotations |
| `T201`  | Uso de `print` |
| `UP`    | Modernización de Python |
| `C90`   | Complejidad ciclomática |
| `PLR`   | Reglas de pylint |
| `I`     | Orden de imports (isort) |
