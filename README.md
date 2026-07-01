# Add Visual Element

Skill global para **Claude Code**. Pipeline de ingesta de CSS/HTML a componente React reutilizable.

## Instalación

```bash
# Clonar en el directorio global de skills de Claude Code
cd ~/.claude/skills && git clone https://github.com/BenitoSeguraGonzalez/add-visual-element.git
```

O si preferís instalarlo dentro de un proyecto específico:

```bash
# Clonar en el skills directory del proyecto
cd tu-proyecto && mkdir -p .claude/skills && git clone https://github.com/BenitoSeguraGonzalez/add-visual-element.git .claude/skills/add-visual-element
```

## Verificación

Para confirmar que el skill se registró correctamente, ejecutá en Claude Code:

```
/list-skills
```

Debe aparecer `add-visual-element` en la lista. Una vez instalado, el skill se activa automáticamente cuando pedís crear elementos visuales o cuando compartís código CSS/HTML para indexar.

## Uso

### Pipeline completo

Compartí CSS/HTML y pedí agregarlo a la librería. El skill:

1. Detecta automáticamente tu ecosistema (barrel files, design.md, CLAUDE.md)
2. Extrae y analiza el código fuente
3. Pregunta nombre, categoría y variantes
4. Crea el componente React con estilos dinámicos
5. Genera un preview HTML compilado con esbuild + Tailwind
6. Indexa en el barrel y actualiza la documentación
7. Verifica tipos con `tsc --noEmit`

### Comandos

| Comando | Acción |
|---------|--------|
| `--check` | Comparar componentes locales contra el vault global. Reporta desactualizaciones. |
| `--preview <Componente>` | Genera preview visual de un componente existente sin modificar nada. |
| `--scope global` | Crear componente en el vault global (`~/.claude/nur-ui/`), disponible para todos los proyectos. |
| `--scope project` | Crear componente solo en el proyecto actual (default). |

### Refactor

Cuando querés reemplazar patrones duplicados por componentes de librería:

```
"auditá los componentes visuales del proyecto"
"refactorizá todos los KPI cards para usar KpiCard"
```

El skill ejecuta auditoría dry-run, planifica batches independientes, aplica cambios con checkpoint por batch, y permite rollback granular.

## Requisitos

El proyecto debe tener disponibles (se usan vía `npx`):

- `esbuild` — bundling del preview
- `tailwindcss` — compilación CSS on-demand

## Estructura

```
add-visual-element/
├── SKILL.md       ← Orquestador: pipeline 0→7, interceptación, comandos
├── preview.md     ← Preview HTML con esbuild + Tailwind, gallery/single, mockFetch
├── vault.md       ← Vault global↔local, metadata embebida, conflictos
├── catalog.md     ← Barrel index, design.md idempotente, manifest.json
└── enforce.md     ← Enforcement registry, auditoría dry-run, refactor quirúrgico
```

## Licencia

MIT
