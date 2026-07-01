---
name: add-visual-element
description: Pipeline genérico de ingesta de CSS/HTML a componente React. El usuario da código → se crea componente en la librería visual del proyecto, se indexa, se actualiza documentación. Usar cuando el usuario comparte CSS/HTML y pide agregarlo como componente reutilizable. Frases: "agrega esto a la librería", "onboard this component", "indexa este elemento", "add visual element", "add to [lib-name]".
---

# Add Visual Element — Orquestador

Skill global de gestión de componentes visuales. Funciona en cualquier proyecto con librería de componentes (NurUI o similar).

## Arquitectura

```
add-visual-element/
├── SKILL.md       ← orquestador (este archivo): pipeline, interceptación, comandos
├── preview.md     ← generación de preview HTML con esbuild + Tailwind
├── vault.md       ← gestión global↔local, metadata, conflictos, rename, delete
├── catalog.md     ← barrel index, marcadores idempotentes, design.md, manifest.json
└── enforce.md     ← enforcement registry, refactor quirúrgico, auditoría
```

## Modos de invocación

El skill se activa por mención explícita o por interceptación:

| Gatillo | Acción |
|---------|--------|
| Usuario comparte CSS/HTML + pide agregarlo | Pipeline completo (paso 0→7) |
| Usuario pide crear elemento visual ("creame un KPI card") | Paso -1 (interceptar) → pipeline si no hay match |
| `--check` | Escanear locales vs globales (→ vault.md) |
| `--preview <Componente>` | Solo preview (→ preview.md) |
| `--scope global` | Crear en vault global, no en proyecto local |
| `--scope project` | Crear solo en proyecto local (default) |

## Pipeline principal

Ejecutar en orden. Cada paso delega al módulo correspondiente.

### Paso -1: Interceptación (OBLIGATORIO antes de crear cualquier UI)

Si el usuario pide crear/modificar un elemento visual ("creame un KPI card", "agregá un badge", "haceme un panel de..."):

1. Extraer el UI type del prompt (ej: "KPI card", "badge", "status indicator")
2. Leer `<!-- @nur-ui:start enforcement -->` en `docs/design/design.md`
3. Si hay match en el registry → reportar y proponer usar el componente registrado
4. Si no hay match → continuar con el pipeline, y al final preguntar si quiere registrar este UI type

**Regla**: cualquier skill o agente que vaya a crear UI debe consultar el enforcement registry primero. La interceptación no se limita a cuando se invoca `add-visual-element` — es transversal.

### Paso 0: Detectar ecosistema del proyecto

1. Buscar barrel files: `rg "export.*from" --glob "**/nur-ui/index.ts" --glob "**/ui-kit/index.ts"`
2. Buscar design.md: `ls docs/design/design.md`
3. Leer `CLAUDE.md` → buscar delegación a design.md
4. Si no hay librería → preguntar si crear una. Proponer estructura `src/<lib-name>/` con `index.ts`
5. Determinar variables de sesión:
   - `LIB_DIR`: directorio de la librería (ej: `frontend/src/nur-ui/`)
   - `LIB_NAME`: nombre (ej: `NurUI`)
   - `LIB_IMPORT`: path de import (ej: `@/nur-ui`)
   - `DESIGN_MD`: path a design.md (ej: `docs/design/design.md`)
   - `CLAUDEMD`: path a CLAUDE.md

### Paso 1: Extraer código fuente

- URL → WebFetch para obtener HTML+CSS completo
- Código pegado → usarlo directamente
- Identificar: markup, CSS, variables, @keyframes, pseudo-elementos

### Paso 2: Clarificar (3 preguntas juntas)

Hacer las 3 en un solo mensaje:
1. **Nombre** (PascalCase). Si no sabe, proponer basado en snippet original.
2. **Categoría** (ej: `paneles`, `botones`, `inputs`, `feedback`, `navegacion`, `data-display`, `layout`). Si es nueva, se crea.
3. **Variantes** — ¿el CSS tiene múltiples estilos? Traducir a props (`variant`, `size`, `tone`).

### Paso 3: Crear componente React

→ Delegar lógica de creación a `catalog.md` (template, estructura de archivo).
→ Reglas: sin tamaño fijo, estilos dinámicos inline, interface exportada, atribución si es de terceros.
→ Ver `catalog.md` para el template completo.

### Paso 3.5: Preview

→ Delegar a `preview.md`.
→ Generar `.preview.tsx` companion si el auto-detect no alcanza.
→ Preview HTML con esbuild + Tailwind on-demand.
→ El usuario aprueba/rechaza. Si rechaza, editar y re-preview.
→ **No indexar hasta que el preview sea aprobado.**

### Paso 4: Indexar en barrel

→ Delegar a `catalog.md`.
→ Agregar export al `index.ts` en orden alfabético.

### Paso 5: Actualizar documentación

→ Delegar a `catalog.md`.
→ Actualizar `design.md` en bloques delimitados por marcadores.
→ Regenerar `manifest.json`.
→ Si es enforcement, actualizar `<!-- @nur-ui:start enforcement -->`.

### Paso 6: Verificar

```bash
cd <frontend> && npx tsc --noEmit
```
Corregir errores antes de continuar. Si hay consumidores rotos, reportar y proponer fixes (→ `enforce.md`).

### Paso 7: Reporte final

```
Componente `<Nombre>` integrado en <LIB_NAME>.

- Archivo: <LIB_DIR>/<Nombre>.tsx
- Categoría: <categoría>
- Variantes: <lista>
- Import: import { <Nombre> } from '<LIB_IMPORT>'
- Preview: <URL temporal>
- Catálogo: actualizado en <DESIGN_MD>
```

## Scope: global vs project

Ver `vault.md` para el flujo completo. Resumen:

| Scope | Dónde se crea | Dónde se usa |
|-------|--------------|-------------|
| `project` (default) | `<LIB_DIR>/` | Solo este proyecto |
| `global` | `~/.claude/nur-ui/` (vault) | Se copia a cualquier proyecto bajo demanda |

- Un componente global NUNCA se usa directamente. Siempre se copia al proyecto local.
- La metadata embebida registra origen y fecha: `// @nur-ui: source=global hash=a1b2c3 copied=2026-07-01 category-global=paneles category-local=contenedores`
- `--check` compara hashes locales vs vault global y reporta desactualizaciones.

## Modo refactor (invocación explícita)

→ Delegar a `enforce.md`.

Cuando el usuario pide "refactorizá todos los KPIs para usar DragonflyCard":
1. Auditoría dry-run: escanear patrones visuales duplicados
2. Batches independientes por UI type
3. Checkpoint (tsc limpio) después de cada batch
4. Preview de regresión al final
5. Rollback granular si un batch falla

## Migración de design.md existente

→ Delegar a `catalog.md`.

Si el proyecto ya tiene `design.md` sin marcadores NurUI:
1. Backup `design.md.bak-YYYYMMDD`
2. Insertar marcadores al final (sin tocar contenido existente)
3. Sembrar catálogo inicial desde componentes detectados
4. Reportar: "N componentes detectados. 0 enforcement rules."
5. Usuario valida. Si rechaza, restaurar backup.
