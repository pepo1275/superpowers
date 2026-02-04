# Superpowers - Guia Interna de Referencia

> Documento generado el 4 de febrero de 2026.
> Recoge el analisis completo del framework, la infraestructura montada y las instrucciones operativas.

---

## 1. Que es Superpowers

Superpowers es un **plugin para coding agents** (Claude Code, Codex, OpenCode) que inyecta un framework de desarrollo estructurado. No es una aplicacion — es una coleccion de **skills composables** que guian al agente a traves de un workflow completo de ingenieria de software.

- **Repo original**: [obra/superpowers](https://github.com/obra/superpowers) (Jesse Vincent)
- **Nuestro fork**: [pepo1275/superpowers](https://github.com/pepo1275/superpowers)
- **Version actual**: 4.1.1
- **Licencia**: MIT
- **Lenguajes**: Shell (70.5%), JavaScript (19.0%), Python (5.2%), TypeScript (3.9%)

### Filosofia central

El framework se basa en cuatro principios:

1. **Test-Driven Development** como practica obligatoria, no opcional
2. **Sistematico sobre ad-hoc** — procesos definidos en vez de improvisacion
3. **Simplicidad** — reducir complejidad activamente
4. **Evidencia sobre afirmaciones** — verificar antes de declarar que algo funciona

---

## 2. Arquitectura del Plugin

```
superpowers/
├── .claude-plugin/
│   ├── plugin.json           # Metadatos: nombre, version, autor, licencia
│   └── marketplace.json      # Config para distribucion via marketplace
├── commands/                  # 3 slash commands (thin wrappers)
│   ├── brainstorm.md         # /brainstorm → invoca skill brainstorming
│   ├── write-plan.md         # /write-plan → invoca skill writing-plans
│   └── execute-plan.md       # /execute-plan → invoca skill executing-plans
├── skills/                    # 14 skills composables (el nucleo)
│   ├── brainstorming/
│   ├── dispatching-parallel-agents/
│   ├── executing-plans/
│   ├── finishing-a-development-branch/
│   ├── receiving-code-review/
│   ├── requesting-code-review/
│   ├── subagent-driven-development/
│   ├── systematic-debugging/
│   ├── test-driven-development/
│   ├── using-git-worktrees/
│   ├── using-superpowers/
│   ├── verification-before-completion/
│   ├── writing-plans/
│   └── writing-skills/
├── agents/
│   └── code-reviewer.md      # Subagente: Senior Code Reviewer
├── hooks/
│   ├── hooks.json            # Registro de hooks (SessionStart)
│   └── session-start.sh      # Inyecta using-superpowers al inicio de cada sesion
├── lib/
│   └── skills-core.js        # Motor: descubrimiento, parsing, shadowing de skills
├── tests/                     # Tests organizados por categoria
│   ├── claude-code/          # Test runner + helpers para Claude CLI
│   ├── explicit-skill-requests/  # Prompts de prueba para invocacion de skills
│   ├── opencode/             # Tests de integracion OpenCode
│   ├── skill-triggering/     # Tests de triggering automatico
│   └── subagent-driven-dev/  # Tests de flujo SDD con proyectos reales
├── docs/                      # Documentacion por plataforma
└── .github/
    ├── FUNDING.yml
    └── workflows/             # CI/CD (anadido por nosotros)
        ├── ci.yml
        └── sync-upstream.yml
```

### Patron de diseno: Commands como Thin Wrappers

Los 3 commands son identicos en estructura — cada `.md` tiene:
- `disable-model-invocation: true` (no consume tokens del modelo, solo redirige)
- Una sola instruccion: "Invoke the superpowers:SKILL_NAME skill and follow it exactly"

Esto separa la interfaz (lo que el usuario invoca) de la implementacion (lo que el skill contiene). Si quieres cambiar el comportamiento del brainstorming, editas el skill, no el command.

### Motor de Skills: lib/skills-core.js

Cinco funciones exportadas como ES modules:

| Funcion | Que hace |
|---------|----------|
| `extractFrontmatter(filePath)` | Parsea YAML frontmatter (name, description) de un SKILL.md |
| `findSkillsInDir(dir, sourceType, maxDepth)` | Busca recursivamente SKILL.md hasta 3 niveles de profundidad |
| `resolveSkillPath(skillName, superpowersDir, personalDir)` | Resuelve nombre a path con **shadowing**: personal > superpowers |
| `checkForUpdates(repoDir)` | Git fetch con timeout de 3s para no bloquear el bootstrap |
| `stripFrontmatter(content)` | Elimina frontmatter YAML, devuelve solo el contenido del skill |

**Shadowing**: Si creas `~/.claude/skills/brainstorming/SKILL.md`, sobreescribe el brainstorming de superpowers sin tocar el plugin. Para forzar el original: `superpowers:brainstorming`.

### Hook de SessionStart

Al iniciar cada sesion de Claude Code, el hook:
1. Lee el contenido de `skills/using-superpowers/SKILL.md`
2. Lo inyecta como contexto adicional envuelto en `<EXTREMELY_IMPORTANT>`
3. Verifica si hay skills legacy en `~/.config/superpowers/skills` y avisa de migrar
4. Matcher: `startup|resume|clear|compact` — se ejecuta siempre

### Agente: code-reviewer

Un subagente con system prompt de "Senior Code Reviewer" que evalua en 5 ejes:
1. **Plan Alignment** — cumple lo que se planeo
2. **Code Quality** — errores, tipos, defensivo
3. **Architecture** — SOLID, separacion, coupling
4. **Documentation** — comentarios, headers
5. **Issues** — categorizados como Critical / Important / Suggestion

---

## 3. El Workflow Completo (7 Fases)

```
/brainstorm ──→ worktree ──→ /write-plan ──→ /execute-plan ──→ TDD ──→ code review ──→ merge
   Fase 1         Fase 2        Fase 3          Fase 4        Fase 5     Fase 6       Fase 7
```

### Fase 1: Brainstorming
- Dialogo socratico: una pregunta a la vez, opciones multiple choice
- Presenta diseno en bloques de 200-300 palabras para validacion incremental
- No se escribe codigo hasta que el diseno esta aprobado

### Fase 2: Git Worktrees
- Crea workspace aislado en branch dedicado
- Prioridad de directorio: `.worktrees/` existente > CLAUDE.md > preguntar
- Verifica gitignore, instala dependencias (npm/cargo/pip/poetry/go), valida tests base

### Fase 3: Writing Plans
- Plan granular con tareas de 2-5 minutos
- Cada tarea incluye: archivos a tocar, codigo completo, comandos con output esperado
- Principios: true TDD, YAGNI, DRY

### Fase 4: Executing Plans (dos modos)

**Modo A: executing-plans** — Ejecucion en lotes de 3 tareas con checkpoint de revision humana.

**Modo B: subagent-driven-development** — Un subagente fresco por tarea + double-gate review:
- Gate 1: Verificacion de spec compliance (hace lo que se pidio?)
- Gate 2: Verificacion de code quality (esta bien hecho?)

### Fase 5: Test-Driven Development
Regla de hierro: "NO PRODUCTION CODE WITHOUT A FAILING TEST FIRST"
- Si escribes codigo antes de test → borrar y empezar de cero
- Ciclo RED (test falla) → GREEN (minimo codigo que pasa) → REFACTOR (limpiar)
- Unica excepcion: prototipos desechables aprobados por humano

### Fase 6: Code Review
- `requesting-code-review` envia SHA inicial/final al agente code-reviewer
- `receiving-code-review` framework para recibir feedback
- Pushback tecnico permitido: no es acuerdo performativo

### Fase 7: Branch Finalization
4 opciones: merge local, crear PR, mantener branch, descartar.
Cleanup automatico del worktree.

---

## 4. Las 14 Skills — Referencia Rapida

### Skills de Proceso
| Skill | Cuando se usa | Tamanio |
|-------|--------------|---------|
| brainstorming | Antes de cualquier trabajo creativo | 2,261 chars |
| writing-plans | Despues de brainstorm aprobado | 3,132 chars |
| executing-plans | Para ejecutar plan en lotes de 3 | 2,400 chars |
| subagent-driven-development | Alternativa a executing-plans con subagentes | 9,823 chars |

### Skills de Calidad
| Skill | Cuando se usa | Tamanio |
|-------|--------------|---------|
| test-driven-development | Siempre que se escribe codigo | 9,724 chars |
| systematic-debugging | Cuando hay bugs — 4 fases obligatorias | 9,718 chars |
| verification-before-completion | Antes de declarar algo "terminado" | 3,863 chars |

### Skills de Colaboracion
| Skill | Cuando se usa | Tamanio |
|-------|--------------|---------|
| requesting-code-review | Despues de completar implementacion | 2,540 chars |
| receiving-code-review | Al recibir feedback del reviewer | 5,990 chars |
| dispatching-parallel-agents | Problemas independientes, no secuenciales | 5,918 chars |

### Skills de Infraestructura
| Skill | Cuando se usa | Tamanio |
|-------|--------------|---------|
| using-git-worktrees | Al iniciar trabajo en feature | 5,380 chars |
| finishing-a-development-branch | Al terminar trabajo en branch | 3,969 chars |

### Meta-Skills
| Skill | Cuando se usa | Tamanio |
|-------|--------------|---------|
| using-superpowers | Auto-carga en SessionStart | 3,581 chars |
| writing-skills | Para crear nuevos skills | 22,227 chars |

---

## 5. Infraestructura CI/CD (lo que montamos)

### Remotes Git

```
origin   → github.com/pepo1275/superpowers   (nuestro fork, push aqui)
upstream → github.com/obra/superpowers        (repo original, solo lectura)
```

Sincronizacion manual:
```bash
git fetch upstream main
git log --oneline origin/main..upstream/main   # ver que hay nuevo
git merge upstream/main                         # integrar
git push origin main                            # subir
```

### Workflow CI: `.github/workflows/ci.yml`

Se ejecuta en **push a main** y en **pull requests**. Tres jobs en paralelo:

| Job | Que valida |
|-----|-----------|
| `validate-skills` | YAML frontmatter de las 14 skills + parsing con skills-core.js |
| `validate-plugin` | plugin.json (semver, campos), hooks.json (eventos validos, scripts existen), commands |
| `shellcheck` | Lint de todos los .sh en hooks/ y tests/claude-code/ |

### Workflow Sync: `.github/workflows/sync-upstream.yml`

- **Frecuencia**: Diario a las 08:00 UTC + trigger manual desde GitHub Actions
- **Logica**: Si hay commits nuevos en obra/superpowers, crea un PR automatico
- **Conflictos**: Si hay merge conflicts, falla y muestra comandos para resolver manualmente
- **Seguridad**: No hace merge directo a main, siempre via PR para revision

### Primer CI ejecutado

```
Commit: 9907e6e
Status: completed/success
Duracion: 10 segundos
```

---

## 6. Dos Modos de Uso: Plugin vs Fork Local

### Modo A: Plugin desde Marketplace (uso en produccion)

**Instalar:**
```bash
# Paso 1: Anadir el marketplace
claude /plugin marketplace add obra/superpowers-marketplace

# Paso 2: Instalar
claude /plugin install superpowers@superpowers-marketplace
```

**Caracteristicas:**
- Se actualiza automaticamente con el marketplace
- Disponible globalmente en todas las sesiones de Claude Code
- Los skills del plugin se cargan desde `~/.claude/plugins/cache/...`
- No refleja modificaciones locales

**Desinstalar:**
```bash
claude /plugin uninstall superpowers@superpowers-marketplace
```

### Modo B: Instalacion desde Fork Local (desarrollo)

**Instalar:**
```bash
claude /plugin install --path /Users/pepo/Dev/superpowers
```

**Caracteristicas:**
- Los cambios que hagas en `/Users/pepo/Dev/superpowers/skills/` se reflejan inmediatamente
- Ideal para desarrollar y probar nuevos skills
- Necesitas actualizar manualmente (`git pull`, `git merge upstream/main`)
- El CI de GitHub valida tus cambios al pushear

**Desinstalar:**
```bash
claude /plugin uninstall superpowers
```

### Modo Combinado (recomendado)

1. Instalar primero desde marketplace para validar que todo funciona
2. Probar con `/brainstorm`, `/write-plan`, `/execute-plan`
3. Cuando quieras modificar: desinstalar marketplace, instalar desde fork local
4. Desarrollar cambios en el fork
5. Push al fork → CI valida → si OK, usar en produccion

### Desactivar por Proyecto

Si tienes el plugin instalado pero no quieres que se active en un proyecto especifico, anade esto al `CLAUDE.md` del proyecto:

```markdown
Do not use superpowers skills in this project.
```

### Personalizar Skills sin Tocar el Plugin

Gracias al sistema de shadowing de `resolveSkillPath()`:
```bash
# Crear skill personal que sobreescribe brainstorming
mkdir -p ~/.claude/skills/brainstorming
# Editar tu version
vim ~/.claude/skills/brainstorming/SKILL.md
```

Tu version tiene prioridad. Para forzar el original: `superpowers:brainstorming`.

---

## 7. Testing

### Tests Existentes (del repo original)

Los tests usan `claude -p` (modo headless) y son de dos tipos:

**Rapidos (unit):**
```bash
cd tests/claude-code
./run-skill-tests.sh
```

**Integracion (10-30 min, usan API):**
```bash
cd tests/claude-code
./run-skill-tests.sh --integration
```

**Helpers disponibles** (`tests/claude-code/test-helpers.sh`):
- `run_claude "prompt"` — ejecuta Claude headless con timeout
- `assert_contains "output" "pattern"` — verifica patron en output
- `assert_not_contains` — verifica ausencia
- `assert_count` — cuenta ocurrencias
- `assert_order` — verifica orden de patrones
- `create_test_project` / `cleanup_test_project` — setup/teardown

### Validacion Estructural (CI, sin costo API)

Lo que el CI valida sin necesidad de llamar a la API de Claude:
- Cada skill tiene SKILL.md con frontmatter valido (name + description)
- skills-core.js puede parsear todos los skills sin errores
- plugin.json tiene campos requeridos y version semver
- hooks.json referencia eventos validos y scripts que existen
- Shell scripts pasan ShellCheck

---

## 8. Crear un Skill Nuevo

El skill `writing-skills` documenta el proceso completo. Resumen:

### Estructura

```
skills/mi-nuevo-skill/
└── SKILL.md
```

### Frontmatter obligatorio

```yaml
---
name: mi-nuevo-skill
description: Use when [condicion] - [que hace]
---
```

La description DEBE empezar con "Use when..." — esto es CSO (Claude Search Optimization) para que el agente encuentre el skill cuando lo necesite.

### Secciones del SKILL.md

1. Overview (que es)
2. When to Use (cuando aplicarlo)
3. Core Pattern (el patron central)
4. Quick Reference (referencia rapida)
5. Implementation (como hacerlo)
6. Common Mistakes (errores frecuentes)

### Regla de hierro

"NO SKILL WITHOUT A FAILING TEST FIRST" — Aplica TDD al propio skill:
- RED: Probar escenarios de presion sin el skill
- GREEN: Escribir skill minimo que resuelve los fallos
- REFACTOR: Cerrar loopholes y racionalizaciones

### Token budget

- Getting-started: < 150 palabras
- Skills frecuentes: < 200 palabras
- Otros: < 500 palabras

---

## 9. Operaciones Comunes

### Sincronizar con upstream

```bash
cd /Users/pepo/Dev/superpowers
git fetch upstream main
git log --oneline origin/main..upstream/main   # ver diferencias
git merge upstream/main
git push origin main
```

O esperar al PR automatico diario del workflow sync-upstream.

### Verificar skills localmente

```bash
# Validar frontmatter
for f in skills/*/SKILL.md; do echo "--- $f"; head -5 "$f"; done

# Parsear con skills-core.js
node --input-type=module -e "
import { findSkillsInDir } from './lib/skills-core.js';
const skills = findSkillsInDir('./skills', 'superpowers');
skills.forEach(s => console.log(s.name, ':', s.description.substring(0,60)));
"
```

### Probar un skill nuevo en el CI

```bash
git checkout -b feature/mi-nuevo-skill
# crear el skill...
git add skills/mi-nuevo-skill/SKILL.md
git commit -m "feat: add mi-nuevo-skill"
git push origin feature/mi-nuevo-skill
# crear PR → CI valida automaticamente
```

---

## 10. Estado Actual

| Elemento | Estado |
|----------|--------|
| Fork clonado en `/Users/pepo/Dev/superpowers` | OK |
| Remote `origin` → pepo1275/superpowers | OK |
| Remote `upstream` → obra/superpowers | OK |
| CI (validate-skills + validate-plugin + shellcheck) | OK, primer run exitoso |
| Sync upstream (PR automatico diario) | OK, workflow activo |
| Plugin instalado en Claude Code | Pendiente de instalar |
| Ultimo commit | `9907e6e` ci: add CI validation and upstream sync workflows |
