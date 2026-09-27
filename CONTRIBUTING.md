# Cómo contribuir — DeskcommCRM

## Antes de empezar

0. Abre el repositorio en tu asistente de código (Claude Code, Codex, Cursor, OpenCode o
   Antigravity): la guía `deskcomm-contribuir` (`.agents/skills/deskcomm-contribuir/SKILL.md`) mide
   antes del PR lo que la triage mide después — branch atrasada, tripleta de migration, marca del fork en el
   diff, fragmento de release — y arma los hooks de git con `bash .agents/skills/deskcomm-contribuir/scripts/armar-hooks.sh`.
   Para tener las guías en cualquier carpeta: `bash scripts/instalar-guias.sh`. ¿Vas a **editar** una guía?
   Corre `bash scripts/instalar-guias.sh --fonte .` en tu clon — en Claude Code la skill global
   le gana a la del proyecto, y sin eso estarías probando la versión de `main`, no la tuya.
1. Lee [`CLAUDE.md`](CLAUDE.md) — convenciones no negociables.
2. Lee [`ARCHITECTURE.md`](ARCHITECTURE.md) — visión de 1 página.
3. Identifica el epic de origen en [`docs/stories/epics/MASTER.md`](docs/stories/epics/MASTER.md).

## Flujo

### Branches

```
feat/EPIC-XX-short-slug         # nueva feature
fix/EPIC-XX-short-slug          # bug fix
chore/short-slug                # chore (deps, configs)
docs/short-slug                 # solo docs
```

### Commits

Conventional commits + alcance `EPIC-XX`:

```
feat(EPIC-04): kanban drag-and-drop con fractional indexing
fix(EPIC-03): cron recover-stuck-messages marca como failed lo que lleva >5min en sending
docs(EPIC-12): mark complete + wave log
```

Se aceptan mensajes en español o en PT-BR. El asunto debe ir en imperativo y tener ≤72 caracteres.

### epic-executor

Los cambios grandes siguen [`docs/stories/epics/`](docs/stories/epics/). El `epic-executor` consume el frontmatter (`epic_id`, `priority`, `depends_on`, `status`) y ejecuta wave por wave con validación E2E continua.

Al terminar un epic:

1. Actualizar el frontmatter `status: pending → completed (partial: ...)` o `status: completed`.
2. Agregar el "Wave Completion Log" al final del archivo.
3. Actualizar la fila correspondiente en `docs/stories/epics/MASTER.md`.

### Proceso de PR

1. Branch a partir de `main`.
2. Implementar. Agregar tests (E2E para flujos, unit para lógica pura).
3. **Definition of Done.** La lista está dividida en dos por un motivo: hasta hace poco mezclaba
   lo que una máquina rechaza con lo que solo una persona percibe, y quien contribuía marcaba el checklist
   entero de buena fe para luego ser frenado por un gate del que nadie le había hablado.

   **Lo que el CI rechaza por sí solo** — córrelo antes de abrir el PR y no tendrás sorpresas:

   ```bash
   pnpm cercas    # ~30 s: las guardas estructurales (baseline, MANIFEST, docs, workflows, español del i18n, fragmentos de .changes/) — lo que más rechaza PRs
   pnpm typecheck && pnpm lint && pnpm lint:channels && pnpm test:unit && pnpm test:shell && pnpm build
   pnpm test:db   # necesita Docker; levanta un Postgres limpio y aplica el baseline
   ```

   **Lo que el CI NO ve** — queda en tus manos y en las de la revisión, y es donde viven los defectos caros:

   - RLS habilitada y policy `tenant_isolation_<tabla>_all` si creaste una tabla tenant-aware
     (el test de aislamiento cubre una lista fija de tablas; la tuya nueva no entra sola)
   - Audit log emitido si hay una mutación relevante
   - Rate limit aplicado si la ruta es pública
   - Zod validando todo input externo
   - Sin `console.log` olvidado (usa `lib/logger.ts`). **`pnpm lint` no lo rechaza** — la regla
     está como advertencia, así que pasa en verde; la revisión es humana
   - Env vars nuevas en `.env.example` **y** en `lib/env.ts`, con un default que no rompa una instalación nueva
   - El cambio de schema salió como **tripleta**: archivo en `supabase/migrations/`, apéndice idempotente
     en `supabase/baseline.sql` y línea en `MANIFEST.md`. El kit self-host aplica **solo el baseline** —
     una migration que no llega ahí no llega a quien instaló en un VPS. Ningún job de CI lo revisa
   - **Si tocaste `Dockerfile*`, `docker-compose*.yml` o `hostgator-setup-kit/`:** el cambio
     llega a quien **ya** instaló. Ley en [`docs/doctrine/packaging.md`](docs/doctrine/packaging.md).
     El CI rechaza un servicio solo-`build:`, una instalación en tag móvil y una imagen rota (`imagens-ok`);
     lo que queda en tus manos es el resto: una variable nueva con un default que no rompa un `.env` antiguo, y que la
     actualización no pida editar archivos a mano. **Ningún bump puede exigir que el operador
     del VPS edite algo a mano** — si lo exige, abre un issue con plan de migración en lugar de un PR
   - Docs actualizadas si cambió un contrato (PRD/spec)
   - `pnpm test:e2e` (el subconjunto relevante) — **opcional si contribuyes desde fuera**, ver abajo
4. Abrir el PR contra `main`. La descripción debe referenciar el epic y listar evidencias (logs/screenshots de los tests).
5. **¿Tocaste un documento de autoridad?** Corrige las afirmaciones de estado **de ese** documento —
   las que dicen qué está activo, qué falta, qué apunta a dónde. No salgas a cazar en los
   demás: la deuda decae sola si nadie la alimenta. Hallazgos medidos, con el comando de cada
   uno, en [`docs/audits/2026-08-14-afirmacoes-de-estado.md`](docs/audits/2026-08-14-afirmacoes-de-estado.md).

6. El CI debe pasar antes del merge. Obligatorios: `verify`, `invariants` (aislamiento RLS),
   `build-and-size`, `e2e` e `imagens-ok`.

   `imagens-ok` (en `.github/workflows/publish-image.yml`) construye las tres imágenes que
   instala quien se auto-hospeda, corre en cada PR y **bloquea** desde el 2026-08-13.

   Verde en `e2e` **no** significa "recorrido probado": él mismo imprime, en el resumen, qué specs no
   cubrió. Cuáles son, léelo del propio workflow en vez de esta línea — ya dijo que la que quedaba
   fuera era `vps-fresh-onboarding`, la instalación desde cero, y desde el PR #983 esa corre en el CI:

   ```bash
   git show origin/main:.github/workflows/e2e.yml | grep -A4 'FORA_DO_CI:'
   ```

   E incluso el recorrido que SÍ tiene gate sigue debiendo la prueba por la pantalla cuando lo tocas
   (DoD 12): el gate prueba que no hubo regresión, no que la experiencia quedó bien.

   > Esta lista decía "tres obligatorios" y llamaba al `e2e` no bloqueante. Estaba
   > desactualizada en los dos puntos, y quien la usara como regla mediría contra la regla equivocada.
   > Compruébalo en la fuente antes de confiar en cualquier lista escrita:
   > `gh api repos/melgarafael/DeskcommCRM/branches/main/protection --jq '.required_status_checks.contexts'`

### Tomar un issue — el protocolo

Existe porque ya fallamos en esto: el 2026-07-30 abrimos un issue, alguien de la comunidad
empezó a resolverlo, y un mantenedor entregó la misma corrección **21 segundos antes**
sin que ninguno de los dos pudiera ver al otro. Su trabajo se fue a la basura. Las reglas
de abajo son para que eso no se repita.

1. **Comenta "lo tomo" antes de programar.** Una línea basta. Un mantenedor te asigna el
   issue — a partir de ahí es tuyo y nadie más lo toca.
2. **Un issue con persona asignada no se duplica.** Si aun así quieres ayudar,
   comenta ofreciéndote; no abras un PR que compita.
3. **Un mantenedor no implementa un issue marcado `good first issue` o `help wanted`** sin
   antes asignárselo públicamente. Si ves uno de esos sin dueño, es tuyo para
   tomarlo — esa es la garantía que damos a cambio del paso 1.
4. **¿Sin respuesta 48h después del "lo tomo"?** Empieza de todas formas y dilo en el PR. La
   demora es nuestra, el costo no puede ser tuyo.

### Si contribuyes desde fuera (fork) — lee esto

Una cosa va a parecer error tuyo y no lo es:

- **Los workflows se quedan esperando aprobación** en tu primer PR. Es política de
  GitHub para quien nunca ha contribuido. Un mantenedor los libera; del segundo PR en adelante
  corren solos. Si tarda, comenta en el PR.

**Abre el PR desde una rama con nombre, nunca desde la `main` de tu fork.** Si la `main` del fork ya tiene
personalizaciones tuyas — y casi siempre las tiene, porque es de ahí de donde jala tu VPS —, el PR propone
esas personalizaciones al producto entero. Eso no genera conflictos y no enciende ningún gate: entran
en silencio a todas las instalaciones. Se midió (PR #465): siete archivos con la marca de un
cliente, seis de ellos haciendo merge sin un solo conflicto. El camino es `git checkout -b fix/lo-que-arreglas`
a partir de la `main` de **este** repositorio, con solo tu arreglo adentro.

**Con "Allow edits by maintainers" activado en tu PR, el proyecto puede empujar un arreglo directo a la
branch del PR** — un ajuste mecánico, o la `main` traída hacia adentro cuando hay conflicto. Siempre como
commit nuevo: nunca `--force`, nunca rebase, y tus commits se quedan como están. Avisamos en el PR antes
de empujar. Cuando eso pase, trae la branch antes de continuar (`git pull --no-rebase`) y solo
entonces vuelve a empujar; un `--force` de tu lado borraría lo que se empujó de este lado. Con la
opción desactivada, el arreglo va en una branch nuestra. En los dos caminos, el trabajo que es tuyo entra con
tu autoría.

**La marca de tu instalación no se cambia editando código.** No modifiques `DEFAULT_APP_NAME` en
`lib/branding.ts`, ni los títulos en `app/`. Manda la base de datos (`platform_branding`,
`organizations.settings.branding`), `APP_NAME` en el `.env` es la semilla que pregunta `install.sh`, y
el resto es la pantalla **Configuración › Marca**. Receta completa en [`docs/white-label.md`](docs/white-label.md).
Editar la constante cambia el default del PRODUCTO — y tu marca desaparece en el siguiente `git pull`, que es la
razón práctica por la que el camino soportado también es mejor para ti.

Y sobre el `pnpm test:e2e` del DoD: correr la suite completa exige Docker, una base sembrada y WAHA
local. **No frenamos PRs externos por eso** — manda lo que pudiste probar (unit + descripción de
lo que probaste a mano), que la prueba de pantalla queda a cargo del mantenedor. Exigir una prueba sin dar la
herramienta para producirla sería un peaje, no rigor.

### `tests/invariants/` está congelado — y eso vale para el COMPORTAMIENTO, no solo para el archivo

Los archivos de `tests/invariants/` guardan leyes del producto, y modificarlos pide una justificación
escrita. Dos cosas que no son obvias y ya le costaron tiempo a quien contribuye:

1. **La guarda es un hook local del mantenedor** (`core.hooksPath=loop/hooks`), no un check del CI.
   No vas a verla fallar en tu fork — lo que ves es que la integración se traba después.
2. **Un PR puede romper un invariante sin tocar su archivo.** Si tu arreglo cambia el
   comportamiento que afirma la ley, el rojo aparece ahí. Eso **no es un descuido tuyo** — es la
   señal de que existen dos reglas en competencia, la que está escrita y la que tú propones.

Cuando pase, **no borres ni aflojes la aserción**: di en el PR cuál es tu razón y deja la
elección explícita. Quien hace la triage escribe el cambio del invariante con la justificación exigida, o ajusta
el arreglo para preservar la ley anterior — y la decisión queda registrada en el PR, que es donde le sirve
a la siguiente persona.

### Texto de pantalla: toda frase nueva necesita el español

El producto habla portugués y español, y el CI rechaza **una frase nueva sin traducción**. La regla no
estaba escrita aquí hasta el 16/09/2026, y un PR de primera contribución fue rechazado por ella — la
falla era nuestra, no de quien contribuyó.

Si agregaste una frase que aparece en pantalla, pasa por `t("...")` **y** recibe una línea
en `lib/i18n/dicionario.ts`:

```ts
"Digite o identificador do modelo": { es: "Escribe el identificador del modelo" },
```

La clave es el texto en portugués (no un código). Solo el español necesita línea; lo demás se degrada
al portugués a propósito.

Para comprobarlo antes de abrir el PR, sin correr la suite entera:

```bash
pnpm test:unit tests/unit/i18n-espanhol-cobre-a-tela.test.ts
```

Rechaza en las dos direcciones: una clave usada en pantalla sin español, y prosa en portugués que no
pasó por `t()`. **Si no hablas español, mándalo de todas formas** y dilo en el PR — la traducción es
trabajo de diez segundos para quien hace la triage, y no es motivo para frenar un arreglo.

### Anti-patterns prohibidos

Lista completa en `CLAUDE.md`. Los más letales:

- Trigger de Postgres haciendo HTTP
- Service role usado en un handler sin filtrar `organization_id` manualmente
- `getSession()` en el backend (usa `getUser()`)
- API key en query string
- Bearer en plaintext en la DB
- `console.log` en código mergeado

## Setup local

Ver [`README.md`](README.md) §Desarrollo.

## Soporte

**[GitHub Discussions](https://github.com/melgarafael/DeskcommCRM/discussions)** — es el canal público,
funciona para cualquier persona y es donde la respuesta queda registrada para quien venga después. Para un bug,
[abre un issue](https://github.com/melgarafael/DeskcommCRM/issues/new/choose).

Si es algo que no cabe en público (seguridad, por ejemplo): `rafael@maudibrasil.com.br` — la misma
dirección del [`CODE_OF_CONDUCT.md`](CODE_OF_CONDUCT.md).

> Esta sección apuntaba a un Discord interno cuya invitación vive en un Notion privado — inalcanzable
> justamente para quien más la necesitaba, que es quien viene de fuera. Se queda aquí como recordatorio de que un
> canal de soporte se prueba desde fuera.
