# Add Visual Element

Skill global para **Claude Code** — Pipeline de ingesta de CSS/HTML a componente React reutilizable. Indexa, cataloga, previsualiza y enforcea consistencia visual en cualquier proyecto con librería de componentes.

## Instalación

```bash
# Clonar en el directorio global de skills de Claude Code
mkdir -p ~/.claude/skills && cd ~/.claude/skills && git clone https://github.com/BenitoSeguraGonzalez/add-visual-element.git
```

Instalación dentro de un proyecto específico:

```bash
cd tu-proyecto && mkdir -p .claude/skills && git clone https://github.com/BenitoSeguraGonzalez/add-visual-element.git .claude/skills/add-visual-element
```

### Verificación

Ejecuta en Claude Code:

```
/list-skills
```

Debe aparecer `add-visual-element` en la lista. El skill se activa automáticamente al compartir CSS/HTML o al pedir la creación de un elemento visual.

---

## Funcionalidades

### Pipeline de ingesta (8 pasos)

Cuando compartes código CSS/HTML y pides agregarlo como componente reutilizable, el skill ejecuta:

| Paso | Descripción |
|------|-------------|
| **-1** | **Interceptación** — Verifica el enforcement registry en `design.md`. Si el UI type ya tiene un componente asignado, lo sugiere en lugar de crear uno nuevo. |
| **0** | **Detección de ecosistema** — Encuentra barrel files, `design.md`, y `CLAUDE.md`. Determina `LIB_DIR`, `LIB_NAME`, y `LIB_IMPORT`. Si no hay librería, ofrece crear una. |
| **1** | **Extracción de código** — Desde URL (WebFetch) o código pegado. Identifica markup, CSS, variables, keyframes, pseudo-elementos. |
| **2** | **Clarificación** — 3 preguntas: nombre (PascalCase), categoría (`paneles`, `botones`, `inputs`, `feedback`, `navegacion`, `data-display`, `layout`), y variantes (traducidas a props: `variant`, `size`, `tone`). |
| **3** | **Creación del componente** — Genera archivo `.tsx` con: estilos dinámicos inline, interface exportada, sin tamaño fijo, atribución si es de terceros (UIverse, etc.). |
| **3.5** | **Preview** — Compila con esbuild + Tailwind, sirve vía HTTP temporal. Soporta modo gallery (múltiples escenarios) y single (controles interactivos). El usuario **debe** aprobar antes de indexar. |
| **4** | **Indexación** — Agrega `export { Nombre }` y `export type { NombreProps }` al barrel `index.ts` en orden alfabético. |
| **5** | **Documentación** — Actualiza `design.md` entre marcadores idempotentes (`<!-- @nur-ui:start catalog -->`). Regenera `manifest.json`. Actualiza enforcement registry si aplica. |
| **6** | **Verificación** — Ejecuta `npx tsc --noEmit`. Si falla, corrige antes de continuar. Si hay consumidores rotos, reporta y propone fixes. |
| **7** | **Reporte final** — Archivo creado, categoría, variantes, import path, preview URL, y catálogo actualizado. |

### Preview interactivo

Antes de indexar, cada componente se previsualiza en un servidor HTTP temporal:

- **esbuild** compila el `.tsx` con React como externo (CDN)
- **Tailwind CSS** se compila on-demand usando el `tailwind.config` del proyecto
- **Puerto**: base 48771, auto-incrementa hasta encontrar libre (máx 10 intentos)
- **Cleanup**: señal de cierre vía `fetch('/__nur-ui-close')` + TTL cleanup (>1h) en la próxima ejecución

#### Modos

| Modo | Descripción |
|------|-------------|
| **Gallery** | Múltiples escenarios en cuadrícula. Cada uno con su label, fondo, y props. Ideal para ver todas las variantes de una vez. |
| **Single** | Un escenario con panel de controles (selectores, inputs, toggles) para cambiar props en vivo. Ideal para iterar sobre un diseño. |

#### Archivo companion `.preview.tsx`

Opcional. Si existe junto al componente, define escenarios, datos mock, y triggers. Si no existe, el skill auto-genera uno con defaults razonables.

```tsx
// @nur-ui preview for DragonflyCard
export const previewMeta = {
  component: 'DragonflyCard',
  mode: 'gallery',       // 'gallery' | 'single'
  viewport: { width: 1200, height: 800 },
};

export const scenarios = [
  { label: 'Light', props: { variant: 'light', children: '...' } },
  { label: 'Dark', props: { variant: 'dark', children: '...' } },
  { label: 'Amber', props: { variant: 'amber', children: '...' } },
];
```

#### Triggers interactivos

| Trigger | Acción |
|---------|--------|
| `button` | Botón en la UI de preview que ejecuta una acción |
| `toggle` | Toggle boolean que cambia una prop |
| `hover` | Simula hover en un selector CSS |
| `click-outside` | Simula click fuera del componente |

#### Mock data y estados async

Soporte para componentes que cargan datos asíncronos:

```tsx
export const mockFetch = {
  delay: 800,           // ms antes de resolver
  response: { data: [...] },
  states: ['loading', 'error', 'empty', 'normal'],
};
```

El preview muestra cada estado (skeleton, error, vacío, normal) en tabs para validación visual completa.

### Sistema Vault (global ↔ local)

Un **vault global** en `~/.claude/nur-ui/` actúa como template repository. Ningún proyecto usa componentes globales directamente — siempre se copian al proyecto local.

| Scope | Dónde se crea | Dónde se usa |
|-------|--------------|-------------|
| `project` (default) | `<LIB_DIR>/` del proyecto | Solo este proyecto |
| `global` | `~/.claude/nur-ui/components/` | Se copia a cualquier proyecto bajo demanda |

#### Metadata embebida

Todo componente copiado del vault recibe un header que registra su origen:

```tsx
// @nur-ui: source=global hash=a1b2c3d4e5f6 copied=2026-07-01 category-global=paneles category-local=contenedores
```

Componentes creados localmente:

```tsx
// @nur-ui: source=project created=2026-07-01 category=paneles
```

#### Comandos del vault

```
"add-visual-element --check"          → Compara hashes locales vs vault. Reporta desactualizaciones.
"add-visual-element --update <Comp>"  → Muestra diff, pide confirmación, sincroniza desde vault.
"Promueve <Componente> a global"       → Copia un componente local al vault para reutilizarlo en otros proyectos.
"Renombra <A> a <B>"                  → Operación atómica multi-archivo con diff preview y rollback.
"Elimina <Componente>"                → Busca consumidores, reporta, no auto-corrige. Usuario decide.
```

#### Conflictos

Cuando `--check` detecta diferencias entre la versión local y el vault:

1. **Diff unificado** lado a lado
2. Opciones: usar vault, mantener local, mantener local y desvincular (`source=project`), o merge manual
3. Si el usuario desvincula, el componente deja de recibir actualizaciones del vault

### Enforcement Registry

Sistema de consistencia visual que impide la duplicación de patrones. Vive en `design.md` entre:

```markdown
<!-- @nur-ui:start enforcement -->
| UI Type | Componente | Regla |
|---------|-----------|-------|
| KPI Card | KpiCard | Usar KpiCard de NurUI. No crear tarjetas KPI manualmente. |
| Panel / Card Container | DragonflyCard | Usar DragonflyCard para cualquier contenedor tipo card. |
<!-- @nur-ui:end enforcement -->
```

#### Cómo funciona

1. **Interceptación obligatoria** — Antes de crear CUALQUIER elemento visual, se consulta el registry
2. **Match → usa el componente registrado** — No se permite crear variantes manuales
3. **No match → crea libremente** — Al finalizar, pregunta si quiere registrar el nuevo UI type
4. **Precedencia absoluta** — El registry gana sobre cualquier otra sección de `design.md`

#### Auditoría dry-run

```
"audita los componentes visuales del proyecto"
```

Escanea el proyecto en busca de patrones duplicados, agrupa por similitud estructural, y reporta qué patrones ya tienen componente en NurUI (candidatos a refactor) y cuáles no (candidatos a extraer).

### Refactor quirúrgico

```
"refactoriza todos los KPI cards para usar KpiCard"
```

| Fase | Descripción |
|------|-------------|
| **1. Auditoría** | Escanea consumidores, identifica variantes necesarias, reporta sin modificar. |
| **2. Planificación** | Agrupa en batches independientes por UI type. Cada batch es atómico. |
| **3. Ejecución** | Reemplaza implementación inline → componente. Verifica `tsc --noEmit` por batch. |
| **4. Checkpoint** | Después de cada batch: tipos limpios + preview de regresión. Usuario confirma. |
| **5. Rollback** | Si un batch falla: `git checkout` solo ese batch. Batches anteriores intactos. |

### Migración de proyectos existentes

Al correr `add-visual-element` por primera vez en un proyecto con componentes visuales pero sin librería:

1. **Backup** de `design.md` (si existe) → `design.md.bak-YYYYMMDD`
2. **Inserta marcadores** idempotentes al final sin tocar contenido existente
3. **Siembra el catálogo** desde componentes detectados en el proyecto
4. **No crea enforcement rules** — el usuario las define gradualmente
5. **Reporta**: "N componentes detectados. 0 enforcement rules."
6. **Validación**: si el usuario rechaza, se restaura el backup.

---

## Comandos

```
# Pipeline principal (gatillos automáticos)
"agrega esto a la librería"
"indexa este componente"
"onboard this element"

# Scope
"--scope global"       → vault global, disponible para todos los proyectos
"--scope project"      → solo proyecto actual (default)

# Vault
"--check"              → comparar locales vs vault
"--update <Componente>" → sincronizar desde vault

# Preview standalone
"--preview <Componente>" → generar preview sin modificar nada

# Refactor y auditoría
"audita los componentes visuales"
"refactoriza todos los <UI type> para usar <Componente>"

# Migración
"migra design.md a NurUI"
```

---

## Requisitos

- **Claude Code** (el skill se registra vía `~/.claude/skills/`)
- **esbuild** — disponible vía `npx esbuild`
- **tailwindcss** — disponible en el proyecto o vía `npx tailwindcss`

---

## Arquitectura

```
add-visual-element/
├── SKILL.md       ← Orquestador: pipeline 0→7, interceptación, comandos
├── preview.md     ← Preview HTML con esbuild + Tailwind, gallery/single, triggers, mockFetch
├── vault.md       ← Vault global↔local, metadata, conflictos, rename, delete, --check
├── catalog.md     ← Barrel index, design.md idempotente, manifest.json, template React
├── enforce.md     ← Enforcement registry, auditoría dry-run, refactor quirúrgico, migración
└── README.md      ← Este archivo
```

Cada módulo es autónomo. El orquestador (`SKILL.md`) delega a los módulos según la fase del pipeline.

---

## Licencia

MIT
