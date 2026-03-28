---
name: full-review
description: Full consolidated code review that runs 4 specialized review skills in sequence (pr-review-toolkit, django-code-review, google code reviewer, coderabbit) and produces a single deduplicated report in Spanish grouped by severity. Use this skill whenever the user asks for a full review, complete review, "full-review", comprehensive code review, or wants to run all reviewers at once. Trigger even if the user just says "full review" or "/full-review".
---

## Tu tarea

Ejecutar 4 skills de code review en secuencia desde la sesión principal, consolidar sus resultados y producir un único reporte en español con los issues deduplicados y agrupados por severidad.

## Paso 1: Ejecutar los 4 reviewers en secuencia

Invoca cada skill usando la herramienta `Skill` y guarda el resultado completo antes de pasar al siguiente. Los skills deben ejecutarse desde la sesión principal — no delegues esto a subagentes.

1. `Skill("pr-review-toolkit:review-pr")` → guardar resultado como **resultado_A**
2. `Skill("django-code-review")` → guardar resultado como **resultado_B**
3. `Skill("google:code-reviewer")` → guardar resultado como **resultado_C**
4. `Skill("coderabbit:review")` → guardar resultado como **resultado_D**

Si algún skill falla, anótalo y continúa con el siguiente.

## Paso 2: Consolidar y deduplicar

Con los 4 resultados en mano, analiza cada issue encontrado:

1. **Identifica duplicados**: Si 2 o más reviewers reportan el mismo problema (mismo archivo, misma línea, o mismo tipo de bug aunque en palabras distintas), agrúpalos en un solo issue.
2. **Asigna severidad consolidada**: Usa la severidad más alta que cualquier reviewer haya asignado a ese issue.
3. **Cuenta confirmaciones**: Anota cuántos reviewers independientes lo detectaron (1/4, 2/4, etc.) — más confirmaciones = mayor confianza.

## Paso 3: Generar el reporte consolidado

Produce el reporte final en español con esta estructura exacta:

---

# Reporte de Code Review Consolidado

## Resumen Ejecutivo
[2-4 oraciones: número total de issues únicos encontrados, principales áreas de riesgo, y estado general del código]

**Reviewers ejecutados:** pr-review-toolkit · django-code-review · google:code-reviewer · coderabbit

---

## 🔴 Crítico
[Issues que deben resolverse antes de mergear — seguridad, pérdida de datos, crashes]

### [Título del issue]
- **Archivo/Línea:** `path/to/file.py:42`
- **Descripción:** [Qué está mal y por qué es un problema]
- **Confirmado por:** [N/4 reviewers: lista de cuáles]
- **Fix recomendado:** [Solución concreta]

---

## 🟠 Alto
[Bugs importantes, vulnerabilidades menores, problemas de rendimiento graves]

[Mismo formato]

---

## 🟡 Medio
[Code smells, violaciones de buenas prácticas, problemas de mantenibilidad]

[Mismo formato]

---

## 🔵 Bajo
[Mejoras de estilo, sugerencias, nitpicks]

[Mismo formato]

---

## ⚪ Issues de Baja Confianza
[Issues reportados por un solo reviewer — válidos pero no confirmados por los demás]

| Archivo | Issue | Reviewer | Severidad |
|---------|-------|----------|-----------|
| `file.py:10` | Descripción breve | coderabbit | Medio |

---

## Estadísticas
- **Issues únicos encontrados:** N
- **Issues duplicados consolidados:** N
- **Distribución:** Crítico: N · Alto: N · Medio: N · Bajo: N

---

## Notas sobre ejecución

- Si algún reviewer falla o devuelve un error, indícalo claramente en el reporte y continúa con los demás.
- Si el contexto no tiene cambios de código (no hay diff ni PR), los reviewers pueden no encontrar nada — indícalo en el resumen ejecutivo.
- Responde siempre en español, excepto nombres de archivos, código, y términos técnicos establecidos (IDOR, N+1, etc.).

---

## Paso 4: Validación con Opus y guardado del reporte

Una vez generado el reporte consolidado, lanza un subagente con modelo Opus para validar cada issue.

### 4a. Lanzar el agente validador

Usa la herramienta `Agent` con `model: "opus"` y pásale:
- El reporte consolidado completo (texto íntegro)
- El diff actual (`git diff`) para que pueda verificar cada issue contra el código real

**Prompt para el agente Opus:**

```
Eres un senior engineer validando un reporte de code review. Tu tarea es:

1. Para cada issue del reporte, leer el código real del repositorio y determinar si es:
   - ✅ VERDADERO: el problema existe en el código actual
   - ⚠️ FALSO POSITIVO: el problema no existe, fue mal interpretado, o ya está resuelto

2. Para los falsos positivos, agrega una nota explicando por qué es incorrecto.

3. Devuelve el reporte completo con estas modificaciones:
   - Issues verdaderos: sin cambios
   - Falsos positivos: agrega debajo del issue una línea "**⚠️ Falso Positivo:** [explicación de por qué no aplica]"
   - Al final del Resumen Ejecutivo, agrega una línea: "**Validación Opus:** X/Y issues confirmados, Z falsos positivos detectados."

No elimines ningún issue — solo anótalos. Devuelve el reporte completo modificado.

REPORTE A VALIDAR:
[REPORTE_CONSOLIDADO]

DIFF DEL CÓDIGO:
[GIT_DIFF]
```

Sustituye `[REPORTE_CONSOLIDADO]` y `[GIT_DIFF]` con los valores reales antes de lanzar el agente.

### 4b. Guardar el reporte validado

Una vez que el agente Opus devuelva el reporte validado, guárdalo en un archivo Markdown en el proyecto:

- **Ruta:** `.claude/reviews/review-YYYY-MM-DD-HH-MM.md`
- Usa la fecha y hora actuales en el nombre del archivo
- Crea el directorio `.claude/reviews/` si no existe

Confirma al usuario la ruta donde se guardó el reporte.
