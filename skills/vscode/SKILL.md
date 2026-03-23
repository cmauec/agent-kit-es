---
name: vscode
description: Abre Visual Studio Code en el proyecto actual. Úsalo cuando el usuario diga "abre visual studio code", "abre vs code", "abre code", "abre el workspace", "abre el workspace de visual studio code", o cualquier variante que implique abrir VS Code en el directorio actual.
---

## Tu tarea

Abrir Visual Studio Code en el directorio de trabajo actual (`primary working directory`).

## Lógica de apertura

1. Busca si existe algún archivo con extensión `.code-workspace` en el directorio de trabajo actual.
2. **Si existe**, ábrelo con:
   ```bash
   open <archivo>.code-workspace
   ```
3. **Si no existe**, abre VS Code con el directorio actual:
   ```bash
   open -a "Visual Studio Code" .
   ```
   (Usa la ruta absoluta del directorio actual en lugar de `.`)

## Respuesta al usuario

Confirma brevemente qué se abrió:

- Si se abrió un workspace: `Workspace \`nombre.code-workspace\` abierto en Visual Studio Code.`
- Si se abrió el folder: `Folder \`/ruta/actual\` abierto en Visual Studio Code.`

Si hay algún error (VS Code no instalado, etc.), explícalo con claridad.
