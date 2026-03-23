---
name: commit-message
description: Genera un mensaje de commit de git (sin hacer commit)
---

## Contexto

Estado actual de git: !git status
Diff actual de git (cambios staged y unstaged): !git diff HEAD
Rama actual: !git branch --show-current
Commits recientes: !git log --oneline -10

## Argumentos

La skill acepta un argumento opcional: `copy`

- Si se pasa `copy` (e.g. `/commit-message copy`), después de mostrar el mensaje de commit DEBES ejecutar el siguiente comando bash para copiarlo al portapapeles:
  ```
  echo "<commit message>" | pbcopy
  ```
  Luego confirma al usuario que el mensaje fue copiado al portapapeles.
- Si NO se pasa `copy`, simplemente muestra el mensaje sin copiarlo.

## Tu tarea

Basándote en los cambios anteriores, genera un mensaje de commit siguiendo el estilo de los commits recientes.

**IMPORTANTE**:
Solo muestra el mensaje de commit. NO ejecutes git add ni git commit. El usuario copiará el mensaje y hará el commit manualmente.
SIEMPRE incluye la atribución de co-autoría al final del mensaje de commit:
  - Si sabes que otros modelos de Claude contribuyeron a los cambios de código, inclúyelos también (e.g., Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>)
  - Formato: Una línea Co-Authored-By: por cada modelo que contribuyó