# Claude Code Orchestration Template

[English](README.md) | **Español**

<!-- NOTA DE MANTENIMIENTO (diseño deliberado - no "corregir"): el stamp de fecha es provenance, no rot.
Registra cuándo se sincronizó esta traducción para que el lector pueda juzgar su frescura.
Se actualiza al re-sincronizar; no se elimina. Sin el stamp, el drift se vuelve indetectable. -->
> Traducción de [README.md](README.md) (canónico, en inglés), sincronizada a julio de 2026. Ante cualquier discrepancia, prevalece la versión en inglés.

Ruteo de modelos, compuertas de gasto y compuertas de revisión para [Claude Code](https://code.claude.com) — con degradación limpia cuando faltan las piezas opcionales.

**Qué es.** Una capa de orquestación drop-in para [Claude Code](https://code.claude.com): unos pocos subagents especializados, un skill reutilizable de Claude Code, y un único archivo de política corto (`CLAUDE.md`). Rutea cada tarea al modelo capaz más barato y mantiene un revisor independiente entre el autor y el merge. Opcionalmente se combina con el CLI de [OpenAI Codex](https://github.com/openai/codex-plugin-cc) para revisión de código con IA cross-family.

**Para quién es.** Desarrolladores que usan Claude Code y quieren orquestación de agentes con disciplina — ruteo de tareas deliberado, subagents aislados y flujos de revisión — en vez de prompting ad-hoc, con validación cross-model opcional.

**Qué resuelve.** Las sesiones multi-agente largas derivan y sobregastan. Este template mantiene limpio el contexto de tu sesión principal, rutea el trabajo mecánico a modelos baratos y el razonamiento difícil a los caros, y nunca deja que un modelo apruebe su propio diff.

**Qué incluye:** cuatro agentes de Claude Code (fast worker, deep reasoner, premium reasoner, diff reviewer), un skill delegation-protocol, compuertas de gasto y compuertas de revisión manuales — todo Markdown plano, sin scripts ni dependencias. Ver [Qué incluye](#qué-incluye) para el mapa completo.

**Qué NO es esto: un truco para ahorrar tokens.** Delegar normalmente *aumenta* los tokens totales: cada worker reconstruye su propio contexto, y solo vuelve un resumen. La guía de costos de Anthropic ubica a los agent teams en aproximadamente 7x los tokens de una sesión estándar cuando los teammates corren en plan mode; la delegación ordinaria con subagents queda por debajo de eso, pero el mecanismo — y la dirección — son los mismos. Lo que realmente comprás:

- **Aislamiento de contexto.** Los workers queman sus tokens en sus propias ventanas de contexto; solo un resumen regresa. Tu sesión principal se mantiene limpia, que es lo que preserva la calidad de decisión en sesiones largas.
- **Arbitraje de pools.** El trabajo cae donde es más barato: tareas mecánicas en modelos más baratos, y opcionalmente en la cuota de otro proveedor (OpenAI Codex) que quizás ya estés pagando.
- **Revisión decorrelacionada.** Dos modelos de la misma familia comparten puntos ciegos. Un revisor de otra familia encuentra bugs de corrección que la familia del autor tiende a pasar por alto.

Si un workflow te promete "70% de ahorro", te está vendiendo un tuit. Este vende disciplina de ruteo: modelo barato donde está el volumen, modelo caro donde está el leverage (planificación y revisión de diffs), otra familia donde importa la independencia.

## Qué incluye

Cada regla vive en la primitiva cuya semántica de carga le corresponde — esa ubicación es el diseño:

| Archivo | Se carga | Contenido |
|---|---|---|
| `CLAUDE.md` | en cada sesión (impuesto fijo — se mantiene corto) | la política: ruteo de modelos, risk paths, reglas de revisión |
| `.claude/agents/fast-worker.md` (Sonnet) | al invocarse | ejecuta tareas chicas y completamente especificadas; el diff más chico posible |
| `.claude/agents/deep-reasoner.md` (Opus) | al invocarse | una pregunta difícil y acotada entra, un análisis con calidad de decisión sale; solo lectura |
| `.claude/agents/premium-reasoner.md` (tier máximo) | al invocarse | el escalón de escalación; con compuerta de gasto, ver abajo |
| `.claude/agents/diff-reviewer.md` (Sonnet) | al invocarse | revisión adversarial de la branch antes del merge; solo lectura; fallback local |
| `.claude/skills/delegation-protocol/SKILL.md` | al delegar | template de spec de tareas, tabla de ruteo, fallbacks de cuota |

Los agentes son agnósticos del proyecto a propósito: todo lo específico (risk paths, comandos de validación, fronteras de datos) vive únicamente en `CLAUDE.md` — una sola fuente de verdad.

## Compatibilidad

Funciona donde funcione Claude Code: macOS, Linux y Windows (nativo o WSL). El template es Markdown plano — sin scripts, sin binarios, sin paths absolutos, sin supuestos de plataforma. Los comandos de ejemplo dentro de los agentes (`git diff`, `grep`) corren en el Bash tool de Claude Code, disponible en todas las plataformas (en Windows nativo corre vía Git Bash, que viene con Git for Windows). El único contenido específico de shell es la sección de Desinstalación, que trae bloques para bash y PowerShell.

## Instalación

1. Clic en **Use this template** (o copiá `CLAUDE.md` + `.claude/` a la raíz de tu repo).
2. Completá el bloque **Project commands** de `CLAUDE.md` (build / test / lint). Cada spec de delegación se ancla en ellos — sin comandos de validación, todo el protocolo es decorativo.
3. Reemplazá los placeholders de **Risk paths** por los tuyos reales (auth, pagos, migraciones, datos de usuarios...).
4. Reiniciá tu sesión de Claude Code — los archivos de subagents se cargan al inicio de sesión — y verificá con `/agents` que aparecen los cuatro workers.

## La compuerta de gasto

`premium-reasoner` es una frontera de facturación, así que tiene guards en capas en vez de buenas intenciones:

- **Token de autorización.** Se niega a correr salvo que el prompt de delegación lleve `PREMIUM-APPROVED`, que el orquestador tiene instruido agregar solo con tu autorización explícita. Falla cerrado: un rechazo cuesta una línea, no un análisis.
- **Peor caso acotado.** `maxTurns` limita cuánto puede gastar una invocación descontrolada; `effort` está fijado en el frontmatter para que el perfil de costo sea configuración explícita, no estado ambiente heredado.
- **Divulgación en runtime.** Cada reporte abre pidiéndote verificar de qué pool salió la invocación (`/usage`). No hay tarifas ni fechas hardcodeadas — los regímenes de facturación cambian más rápido que los templates; los artefactos deben codificar invariantes y divulgar regímenes en runtime.

¿Sin acceso a Fable? Editá una línea marcada en el agente (`model:`) o borralo — el ruteo degrada limpio. Si renombrás el agente o el token, corré `git grep -l PREMIUM-APPROVED` y actualizá cada archivo que devuelva en el mismo commit.

**Verificá que el pin realmente resolvió.** Después de instalar, chequeá `/agents`: los valores de modelo del frontmatter se validan contra el allowlist de modelos de tu organización, y un valor excluido o no disponible se saltea en silencio — el subagent corre entonces en el modelo *heredado*. La compuerta controla *cuándo* corre el premium; solo `/agents` confirma *sobre qué* corre.

## Opcional: plugin de Codex de OpenAI (revisión cross-family)

```
/plugin marketplace add openai/codex-plugin-cc
/plugin install codex@openai-codex
/reload-plugins
/codex:setup
```

Requisitos: Claude Code, Node.js 18.18+ (el plugin puede instalar el Codex CLI vía npm), y una cuenta de ChatGPT en cualquier plan (incluso Free) o una API key de OpenAI. El uso descuenta de **tus límites de Codex, no de los de Claude** — esa separación es el arbitraje.

<!-- La fecha/versión de abajo es provenance deliberado (última verificación), no rot. Actualizar al re-verificar; no eliminar. -->
**Acoplamiento de interfaz, declarado:** la tabla de ruteo de este template referencia `/codex:review`, `/codex:adversarial-review`, `/codex:rescue`, `/codex:status` y `/codex:result`, probados contra [openai/codex-plugin-cc](https://github.com/openai/codex-plugin-cc) a julio de 2026 (Codex CLI 0.142.x). Si el plugin renombra comandos, actualizá la tabla de ruteo del skill.

Sin el plugin nada se rompe: el protocolo rutea las revisiones al `diff-reviewer` local — por instrucción en la tabla de ruteo, no por intercepción — y su veredicto va a señalar honestamente que la revisión misma-familia es la garantía más débil.

**No actives el review gate automático del plugin como default.** Registra un hook de Stop que puede hacer loop entre Claude y Codex y drenar ambos pools de cuota. Revisá manualmente, a nivel de branch: `/codex:review --base main --background` (reemplazá `main` por tu branch default si difiere).

## Nota de seguridad

Codex corre en infraestructura de OpenAI: cada tarea o revisión delegada embarca allí el código tocado. Nunca delegues secretos, `.env*`, credenciales, datos de clientes ni fixtures con datos reales. Para repos confidenciales o de clientes, adaptá la sección **Data boundary** de `CLAUDE.md`: `diff-reviewer` local por defecto, Codex opt-in por tarea.

## Desinstalación

Todo esto son archivos declarativos dentro de tu repo — sin hooks, sin demonios, sin estado fuera del árbol. Superficie de instalación = superficie de desinstalación.

**Si fusionaste este template dentro de un `CLAUDE.md` preexistente, no borres ese archivo** — quitá solo las secciones de orquestación a mano (o restaurá tu versión previa con `git checkout <ref> -- CLAUDE.md`). Los comandos de abajo son seguros solo para una instalación standalone:

```bash
# bash / zsh / Git Bash
rm -f CLAUDE.md
rm -f .claude/agents/{fast-worker,deep-reasoner,premium-reasoner,diff-reviewer}.md
rm -rf .claude/skills/delegation-protocol
```

```powershell
# PowerShell (Windows) - acá no existe la brace expansion, por eso la lista explícita
Remove-Item CLAUDE.md -Force
Remove-Item .claude\agents\fast-worker.md, .claude\agents\deep-reasoner.md, .claude\agents\premium-reasoner.md, .claude\agents\diff-reviewer.md -Force
Remove-Item .claude\skills\delegation-protocol -Recurse -Force
```

Reiniciá la sesión; `/agents` no debería listar ninguno.

## Idiomas

El inglés es canónico — ante discrepancias gana [README.md](README.md). Esta traducción al español se mantiene junto al original; traducciones al portugués (o cualquier otro idioma) son bienvenidas vía PR.

Los archivos de instrucciones (`CLAUDE.md`, agentes, skill) están en inglés **por diseño** y no se traducen: Claude Code los lee en inglés e igual conversa con vos en tu idioma, así que un único set canónico de instrucciones no cuesta nada en la capa de interacción — y previene el peor drift posible: que dos traducciones de la misma política produzcan dos comportamientos distintos.

## Metadatos del repositorio (para mantenedores)

Configuración de GitHub recomendada para descubribilidad. Pegá estos valores a mano en el panel **About** y en **Settings** del repo — esta sección es solo documentación y no cambia ningún metadato remoto automáticamente. Los valores se dejan en inglés porque son los que se pegan literalmente en GitHub.

**About description:**

> Claude Code orchestration template for specialized agents, reusable skills, spend gates, and optional Codex review workflows.

**Topics:**

`claude-code` `claude-code-agents` `claude-code-skills` `subagents` `ai-agents` `agent-orchestration` `multi-agent` `codex` `codex-cli` `code-review` `developer-tools` `prompt-engineering` `llm-workflows` `automation`

**Social preview (opcional):** el ajuste de social preview de GitHub es una imagen, no un campo de texto. Texto sugerido para esa imagen (o como descripción Open Graph):

> Claude Code Orchestration Template - specialized agents, reusable skills, spend gates, and optional cross-family Codex review.

## Créditos

Diseñado para [Claude Code](https://code.claude.com), opcionalmente en pareja con el [plugin de Codex de OpenAI para Claude Code](https://github.com/openai/codex-plugin-cc).

Este repositorio no está afiliado a, respaldado por, ni mantenido por Anthropic ni OpenAI.

## Descargo

Template experimental. No garantiza por sí mismo ahorro de tokens, reducción de costos ni calidad de código. Un humano revisa cada diff antes del merge — las compuertas de revisión existen para informar a ese humano, nunca para reemplazarlo.

## Licencia

MIT — ver [LICENSE](LICENSE).
