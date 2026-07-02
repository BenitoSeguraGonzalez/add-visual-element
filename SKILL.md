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

## Sin argumentos — README de uso

Cuando el skill se invoca sin argumentos (`/add-visual-element` solo), mostrar este bloque exacto como respuesta, completando las secciones dinámicas leyendo `manifest.json` y `vault.json`:

---

```
╔══════════════════════════════════════════════════════════════╗
║  add-visual-element  ·  Gestor de librería de componentes    ║
╚══════════════════════════════════════════════════════════════╝

  Ingesta CSS/HTML → componente React tipado, catalogado y
  documentado. Soporta vault global ↔ proyectos locales.
```

### Catálogo y visualización

| Comando | Qué hace |
|---------|----------|
| `/add-visual-element --view` | Abre el catálogo del proyecto (file://, sin servidor) |
| `/add-visual-element --view --scope global` | Abre el vault global |
| `/add-visual-element --rebuild` | Recompila todos los componentes y regenera el catálogo |
| `/add-visual-element --rebuild --scope global` | Rebuild del vault global |

### Agregar componentes

| Comando | Qué hace |
|---------|----------|
| `/add-visual-element <URL>` | Extrae el CSS/HTML de la URL y abre el pipeline |
| `/add-visual-element <código pegado>` | Usa el snippet directamente |
| `/add-visual-element <descripción>` | Crea el componente desde cero (ej: "creame un KPI card") |
| añadir `--scope global` | Guarda en el vault global en vez del proyecto local |

El pipeline hace: interceptar → detectar ecosistema → extraer código → clarificar (nombre / categoría / variantes) → crear `.tsx` → actualizar catálogo → indexar barrel → documentar.

### Gestión del vault global ↔ local

| Comando | Qué hace |
|---------|----------|
| `/add-visual-element --check` | Compara hashes locales vs vault — detecta componentes desactualizados |
| `/add-visual-element --preview <Nombre>` | Preview individual con todos los escenarios/variantes |

**Vault global** — `~/.claude/nur-ui/` — almacén de componentes reutilizables entre proyectos.
Un componente global nunca se usa directamente: siempre se copia al proyecto con metadata de origen (`source=global hash=... copied=...`).

### Catálogo optimizado (arquitectura actual)

```
previews/
  catalog.react.js   ← React runtime ~190KB — se compila UNA VEZ
  catalog.app.js     ← componentes ~29KB    — se recompila por cada cambio
catalog.html         ← abre con file://, sin servidor HTTP
```

Añadir un componente recompila solo `catalog.app.js` (no React). ~7.5x más rápido que un bundle monolítico.
`catalog.app.entry.tsx` usa marcadores `@nur-ui:start/end` para inserciones incrementales (no full-rewrite).
Skip por hash: si el archivo del componente no cambió, se omite el paso esbuild.

---

**Estado del proyecto actual** (leer `manifest.json` → `vault.json` y completar):

```
Proyecto local
  Librería : <LIB_NAME> en <LIB_DIR>
  Componentes : <N> componentes · <C> categorías
  Catálogo : <existe / no existe>
  Último añadido : <Nombre> (<fecha>)

Vault global  (~/.claude/nur-ui/)
  Componentes : <N> componentes
  Catálogo : <existe / no existe>
```

---

## Modos de invocación

El skill se activa por mención explícita o por interceptación:

| Gatillo | Acción |
|---------|--------|
| Usuario comparte CSS/HTML + pide agregarlo | Pipeline completo (paso 0→7) |
| Usuario pide crear elemento visual ("creame un KPI card") | Paso -1 (interceptar) → pipeline si no hay match |
| `--check` | Escanear locales vs globales (→ vault.md) |
| `--preview <Componente>` | Preview completo individual del componente (todos los escenarios, gallery mode) |
| `--view` | Abrir catálogo persistente existente directamente (→ preview.md) |
| `--rebuild` | Recompilar todos los componentes y regenerar el catálogo desde cero (→ preview.md) |
| `--rebuild --scope global` | Rebuild del catálogo global |
| `--rebuild --scope project` | Rebuild del catálogo local del proyecto (default si no se indica scope) |
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

### Paso 3.5: Actualizar catálogo persistente

→ Delegar a `preview.md`.
→ Compilar preview del componente nuevo (esbuild + Tailwind). Guardar en `previews/<Nombre>.html`.
→ Hacer backup atómico del catálogo actual (`catalog.html` → `catalog.html.bak`).
→ Regenerar `catalog.html` con todos los componentes. El nuevo lleva badge "✦ Último añadido".
→ Servir el catálogo actualizado al usuario para aprobación.
→ Si confirma: borrar `.bak`, continuar al Paso 4.
→ Si rechaza: restaurar `.bak`, el componente NO se indexa.
→ **No indexar hasta que el catálogo actualizado sea aprobado.**

### Paso 4: Indexar en barrel

→ Delegar a `catalog.md`.
→ Agregar export al `index.ts` en orden alfabético.
→ **OBLIGATORIO**: agregar entrada vacía `"usages": []` en `component-map.json` para el componente nuevo (→ `catalog.md § component-map.json`).

### Paso 5: Actualizar documentación

→ Delegar a `catalog.md`.
→ Actualizar `design.md` en bloques delimitados por marcadores (única fuente de verdad del catálogo).
→ Regenerar `manifest.json`.
→ Si es enforcement, actualizar `<!-- @nur-ui:start enforcement -->`.
→ No tocar `CLAUDE.md` ni ningún otro `.md` del proyecto — ver `catalog.md § Fuente de verdad del catálogo`.

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
- component-map.json: actualizado (<N> usages registrados)
```

**OBLIGATORIO en sustituciones masivas**: tras reemplazar instancias existentes por el componente, escanear cada archivo modificado, extraer línea exacta de cada `<NombreComponente`, y actualizar `usages` en `component-map.json` antes de cerrar el reporte. Sin este paso el reporte no está completo.

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
