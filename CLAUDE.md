# CLAUDE.md — DeskcommCRM

> Instrucciones para futuras sesiones de Claude trabajando en este repo. Lectura obligatoria antes de cualquier tarea de código.

**Este archivo es la doctrina — la autoridad final sobre convenciones y anti-patterns.** Complementos, en el orden en que ayudan:

- [`AGENTS.md`](AGENTS.md) — el mismo contrato en forma portable (para Codex/Cursor/Copilot y similares). Se deriva de este archivo, no lo sustituye. **Al cambiar la doctrina aquí, verifica si `AGENTS.md` quedó desactualizado.**
- [`docs/index.md`](docs/index.md) — índice de los 149 docs, con regla de precedencia cuando dos docs no coinciden. Úsalo antes de ponerte a barrer `docs/`.
- [`docs/current-state.md`](docs/current-state.md) — qué está listo, incompleto y roto. **Léelo antes de estimar o prometer cualquier cosa.**
- [`docs/harness-audit.md`](docs/harness-audit.md) — dónde tiene huecos la verificación. Importante: `pnpm gov:verify` **no** cubre `test:db` ni `test:e2e` — verde ahí no es prueba para un cambio de schema o de UI.
- [`docs/threat-model.md`](docs/threat-model.md) — superficie de ataque real del self-host.

---

## Visión (1 párrafo)

DeskcommCRM es un sistema operativo de ventas open source con agentes de IA nativos — multinicho (e-commerce, clínicas, inmobiliarias, infoproductos, servicios), con WhatsApp como canal primario (vía WAHA). Agentes con RAG por tenant atienden, califican y mueven el embudo junto con humanos; el CRM entero se expone vía MCP. Monetización = self-host en VPS (alianza con HostGator), no suscripción. Arquitectura multi-tenant con RLS desde el día 1; LGPD nativa. Posicionamiento completo: `VISION.md`.

---

## Stack canónico

- **Frontend:** Next.js 16 App Router (Turbopack) + React 19 + TypeScript 6 estricto + Tailwind 4 (config en CSS — ver abajo) + shadcn/ui (style: `new-york`, neutral)
- **Backend:** Next.js Route Handlers (mismo repo); workers vía tabla `event_log` + cron
- **DB:** Supabase (Postgres). RLS en toda tabla tenant-aware. Extensions: `uuid-ossp`, `pgcrypto`, `vector`
- **Auth:** Supabase Auth vía `@supabase/ssr`. Cookie SameSite=Strict, HttpOnly, Secure
- **Realtime:** Supabase Realtime (postgres_changes + broadcast)
- **Storage:** Supabase Storage (bucket `whatsapp-media` privado, URLs firmadas)
- **WhatsApp:** WAHA Plus, engine NOWEB
- **Colas/eventos:** tabla `event_log` + workers (no usar Inngest/Trigger en el MVP)
- **Rate limit:** Upstash Redis sliding window
- **AI:** Vercel AI Gateway (Anthropic primario; OpenAI de respaldo para embeddings); strings tipo `"anthropic/claude-sonnet-4-6"`
- **Validación:** Zod en todo input externo (request body, webhook payload, env)
- **Observability:** Sentry con `beforeSend` sanitizado

---

## Convenciones críticas (NO NEGOCIABLES)

### Multi-tenancy
- `organization_id uuid not null references organizations(id) on delete cascade` en **toda** tabla tenant-aware
- Policy RLS `tenant_isolation_<tabla>_all` aplicada vía el helper `fn_user_org_ids()`
- El service role se salta la RLS — los handlers que usan el admin client **DEBEN** filtrar `organization_id` manualmente, resuelto de una fuente confiable (cookie/JWT/webhook secret/path token), **NUNCA del body**
- Toda query que cruza tablas tenant-aware filtra `organization_id` explícitamente
- El test de aislamiento (crea 2 tenants, verifica que no haya fuga) es obligatorio en el CI antes del merge

### Idempotencia y event sourcing ligero
- Mensajes de WhatsApp y eventos externos: `unique (organization_id, external_id)` + capturar `code === '23505'` en el INSERT
- Los POST de creación en la API aceptan el header `Idempotency-Key: <uuid>` (TTL 24h). El recibo vive en **Postgres** (`public.idempotency_keys`, único por organización + clave + endpoint), no en Upstash — ver `lib/api/idempotency.ts`. Qué rutas leen el header: `grep -rln 'Idempotency-Key' app/api/v1 --include='route.ts'`
- **Un trigger de Postgres NUNCA hace HTTP.** El trigger emite una fila en `event_log`; un worker (cron / listener de Realtime) la consume y dispara el side effect

### API REST `/api/v1/`
- Versionado por path. JSON snake_case. UUID v4. ISO-8601 UTC. Dinero en `_cents` + `currency` ISO-4217
- Wrapper de éxito: `{ data, meta?: { cursor, has_more, total } }`
- Wrapper de error: `{ error: { code, message, details? } }` — usar los helpers `ok()` / `fail()` de `lib/api/wrappers.ts`
- Paginación: cursor opaco base64+HMAC por default
- **La auth dual es la dirección del producto, y se cumple ruta por ruta.** Cookie de sesión para el
  frontend; `Authorization: Bearer dsk_...` (fila de `api_tokens`, resuelta en el servidor) para
  llamadas de servidor. El prefijo es **`dsk_`**, y quien lo exige es `lib/mcp/auth.ts` — `tok_` nunca
  existió en el código y estuvo escrito aquí, en `AGENTS.md` y en la Spec 09 hasta el 17/09/2026
  - El helper es `lib/api/auth-dual.ts`, y habilitar una ruta es **por ruta**: no hay interruptor general.
    Para saber cuáles ya aceptan bearer — el número cambia, el comando no:
    `git grep -ln "auth-dual" -- app/api/v1` (más `app/api/v1/contacts/route.ts`, que implementó
    el patrón inline y dio origen al helper)
  - **Llamar al helper en la ruta no basta:** el `proxy.ts` global corre antes de cualquier handler y solo
    reconoce cookie. Sin una entrada en `lib/auth/public-paths.ts` para ese camino, todo bearer
    recibe 401 del proxy antes de llegar al handler. "Público" ahí quiere decir "el proxy no decide",
    nunca "sin autenticación"
  - Decisión del dueño del producto el 17/09/2026: **convertir las rutas que cada integración necesite**,
    conforme aparezcan, en lugar de un namespace paralelo por cliente. Una ruta convertida le sirve a todo
    integrador. Contexto: PR #1008, que escribió 26 rutas paralelas porque no encontró por dónde entrar
- **API key NUNCA en query string** (se filtra en logs de Vercel/CF). Siempre en header
- El plaintext del bearer token se muestra **una vez** al crearlo; después solo el hash SHA256 en la DB
- Headers de rate limit: `X-RateLimit-*` + `Retry-After` en 429
- `X-Request-Id` en toda response (correlaciona con el audit log)

### Auth y RBAC
- Siempre `getUser()` (valida el JWT en el backend). NUNCA `getSession()` (confía en la cookie local)
- 4 roles dentro del tenant: `viewer` (1) < `agent` (2) < `manager` (3) < `admin` (4)
- El super-admin de plataforma es un rol transversal — `is_platform_admin` (decisión final en la Spec 01)
- MFA TOTP es **opcional y lo activa quien administra** — ya no se fuerza por rol. Quienes lo exigen son dos políticas independientes que SE SUMAN: `platform_admins.mfa_required` (para el super-admin) y `organizations.settings.security.mfa_required` (para el `admin` del tenant). El default de ambas es **no exigir**, y `bootstrap-owner.ts` graba `false` explícito. Regla pura en `lib/auth/politica-mfa.ts`
  - **Por qué cambió:** el gate era `isPlatformAdmin || role === "admin"`, sin opción, y `install.sh` crea al dueño como platform admin — así que TODA instalación self-host recibía un bloqueo de pantalla completa justo después del onboarding, un paso que el wizard nunca anunció. Decisión del dueño del producto; una seguridad que expulsa al usuario en la primera pantalla no protege a nadie
  - **⚠️ REGISTRAR y PROBAR son preguntas distintas.** La política decide el registro. En cambio `mfaEmDivida()` — el 403 `mfa_required` de las rutas — NO consulta la política: quien TIENE un factor lo prueba en la sesión, siempre. Atarlo a la política haría que a quien activa la verificación por voluntad propia se le ignore el factor
  - Activar/desactivar vive en **Configuración › Seguridad**; desactivar el propio factor exige sesión `aal2` (si no, una sesión robada desactiva la protección con un clic)
- El permiso por pipeline (`user_pipeline_access`) **NO** entra en el MVP
- Soporte temporal: todo handler que muta en `app/api/v1` declara `requireSupportWrite(`
  de `lib/impersonate/support.ts` **antes del efecto**. Es una guarda de efecto, no de rol — no sustituye
  `requireRole`/RBAC/MFA — y la exige el gate `tests/unit/suporte-cobertura-de-efeitos.test.ts`

### Audit log
- Toda mutación POST/PATCH/DELETE exitosa → 1 entrada en `api_audit_log` (fire-and-forget, p99 ≤500ms)
- **Una ronda de cron que no hizo nada NO es mutación y no audita** — y la que sí hizo algo, audita. `routing-worker` (1×/min) y el extinto `attendant-heartbeat` (1×/5min, eliminado en el #720) auditaban incondicionalmente: ~51.840 filas/mes en una instalación que no atiende a nadie, y en un VPS real el **95% del audit log** eran latidos de cron vacíos (`docs/testing/user-journey-map.md`, hallazgo 17). La guarda correcta es *auditar cuando hubo efecto*, nunca *dejar de auditar* — las dos direcciones las mide `tests/unit/cron-audita-so-quando-ha-efeito.test.ts`, que recorre el AST de **toda** ruta de `app/api/v1/cron/`
- El audit es append-only para los roles de PostgREST, y eso es del SCHEMA y no de la prosa: `anon`, `authenticated` y `service_role` no tienen GRANT de UPDATE, DELETE **ni TRUNCATE** en `api_audit_log` — **ni `service_role`** (migration 0258). El dueño (`postgres`) puede todo, como en cualquier tabla: la garantía es sobre los roles que asume PostgREST, nunca absoluta. Para comprobarlo en la fuente en vez de creerle a esta línea:

  ```bash
  psql "$SUPABASE_DB_URL" -c "select grantee, privilege_type from information_schema.role_table_grants
    where table_schema='public' and table_name='api_audit_log'
      and privilege_type in ('DELETE','UPDATE','TRUNCATE')
      and grantee in ('anon','authenticated','service_role','PUBLIC');"
  ```

  El resultado esperado es **vacío**. Sin el filtro de `grantee` aparecen las filas
  del dueño `postgres` — y no son un defecto.

  **Hasta la 0258 la primera frase de este ítem era falsa en el Supabase real, con el
  gate en verde.** Todo proyecto de Supabase nace con un default ACL de TABLAS en
  `public` que les concede todo a los tres roles — compruébalo con
  `select defaclacl from pg_default_acl where defaclobjtype = 'r' and defaclnamespace = 'public'::regnamespace;`
  —, y el `GRANT` enumerado que el dump emite para esta tabla solo AGREGA, no
  quita. Con la service key, que ignora la RLS, una fila elegida de la auditoría
  se podía borrar o reescribir por REST; `anon`/`authenticated` solo no lo hacían
  porque la RLS no tiene policy de UPDATE/DELETE.

  **La lección que sobrevive al arreglo es sobre la regla, no sobre el grant.** La
  sonda de `tests/invariants/retencao-poda-e-expurgo.test.ts` quedó en verde dos
  veces midiendo el universo equivocado: primero preguntando solo por DELETE/UPDATE con
  TRUNCATE concedido al lado; después preguntando por los tres en un Postgres donde el
  prelude de `scripts/test-db.sh` reproducía el default ACL de Supabase solo para
  FUNCIONES — una base donde el defecto no podía existir. Desde el issue #887 el
  prelude reproduce también el de TABLAS, y esa sonda pasó a medir Supabase.
  La prueba con control propio sigue siendo
  `tests/invariants/audit-log-sob-o-default-acl-do-supabase.test.ts`: concede el
  default ACL a la tabla, vuelve a aplicar el bloque de la 0258 extraído del baseline y solo entonces
  sondea. **Enumerar privilegios en el dump no protege ninguna tabla en el Supabase
  real**; lo que protege es un `revoke` explícito en el apéndice. Qué tablas enumera el dump
  en lugar de `GRANT ALL`:

  ```bash
  grep -nE '^GRANT [A-Z,]+ ON TABLE' supabase/baseline.sql | grep -v 'GRANT ALL'
  ```
- **Retención default de 5 años, configurable, y ahora EJECUTADA.** La purga es `public.fn_expurgar_auditoria_vencida` (`security definer`, **piso de 90 días dentro del cuerpo**, revocada de anon/authenticated), llamada en lotes por el cron `app/api/v1/cron/data-retention` (diario). El knob es `AUDIT_LOG_RETENTION_DAYS`. **No hay capa cold/S3** — el "hot 90 días, cold (S3) el resto" que este archivo afirmó durante meses nunca existió en código (auditoría del 2026-08-14: cero ocurrencias de archivado), y un self-host no tiene a dónde archivar: el Storage del cliente es la MISMA cuota de 1 GB, ya compartida con `whatsapp-media`. Para ver lo que está vigente: `grep -n "RETENCAO_AUDITORIA_DIAS" lib/retencao/politica.ts`
- Por qué una `security definer` de purga no es puerta de adulteración (el argumento completo está en el encabezado de la migration 0167): **no tiene selector de fila** — ningún parámetro de org, actor, acción o id, y el único predicado es `created_at < now() - N días`; el piso vive **en el cuerpo**, no en quien la llama; no es alcanzable por REST; es, desde la 0258, el **único** borrado de auditoría al alcance de la service key — y no elige fila, solo alcanza la punta más vieja que el piso; y **registra su propia erosión** (`retention.sweep_run`, con el conteo, en una fila demasiado nueva para que la siguiente llamada la alcance)
- Una falla al escribir el audit genera alerta en Sentry, no bloquea la mutación principal

### LGPD
- Anonimización preferida sobre delete. El nombre del contacto pasa a `Cliente Anonimizado #N`
- Cascade de redact: contact + conversations + messages (media eliminada del storage) + activities (preserva timestamps)
- Revertir una anonimización: 403 `lgpd_anonymization_irreversible`
- SLA: data_request entregado en D+7; redact ejecutado en D+15
- Action de audit obligatoria: `lgpd.data_request_received`, `lgpd.export_generated`, `lgpd.redact_executed`, `lgpd.consent_changed`

### WAHA
- Default fijo `devlikeapro/waha:latest-2026.7.2`, NOWEB. La prueba local creó dos sesiones CORE simultáneas hasta `SCAN_QR_CODE`; no prueba pairing, dos cuentas `WORKING` ni envío. No bloquear una segunda sesión por tier: revisar la respuesta estructurada y la postcondición de la operación.
- Engine NOWEB por default; WEBJS solo si se necesitan stickers animados / botones
- Auth: el env de WAHA recibe el **hash SHA512 hex** de la api key; el cliente envía el plaintext en `X-Api-Key`
- Webhooks: HMAC SHA512 con `crypto.timingSafeEqual`
- Anti-baneo: throttle 1 msg/1.2s + jitter ≤800ms. Campaña 1 msg/5s. Warm-up 7-14d. Spinning de copy. Ventana 7h-22h (domingo LIBERADO por default desde el 2026-08-20; la ventana es un knob por canal)
- Detección de STOP: la regla vive en `lib/opt-out/deteccao.ts` y es la MISMA en los dos lados —
  la ingesta (que graba `is_blocked=true`) y el runtime del agente. **Ya no es la palabra
  suelta:** solo bloquea la palabra AISLADA (el mensaje entero = la palabra) o un verbo de cese
  con OBJETO DE COMUNICACIÓN ("parar de me mandar", "sair da lista"). Mientras eran dos
  reglas, la ingesta bloqueaba al paciente que preguntó "tem como parar a dor?" — medido en
  clínica, 12 falsos positivos en un corpus de 32 frases de nicho.
  Cubre portugués y español, en los dos niveles (inequívoco y ambiguo) — hizo falta un PR
  además del #275 (que solo había cubierto el vocabulario inequívoco) para que el español ganara la
  capa ambigua y las construcciones con pronombre enclítico ("escribirme"). Para ver el vocabulario
  vigente sin confiar en esta línea:
  `sed -n '/PALAVRAS_DE_OPT_OUT/,/^]/p' lib/opt-out/deteccao.ts | grep -E '^ *"'`, y las frases de control en
  `tests/unit/opt-out-deteccao.test.ts`.
- Media: subir primero a Supabase Storage, pasar la URL a WAHA (no base64 inline)
- Multi-device: suscribirse a `message.any` (no solo a `message`); tratar `fromMe=true` sin duplicar
- Grupos: SKIP del binding con el CRM si `chatId.endsWith('@g.us')`. El remitente es `p.author`, no `p.from`
- Cron `recover-stuck-messages` (`app/api/v1/cron/recover-stuck-messages/route.ts`, agendado en el `scheduler` de `docker-compose.prod.yml`): marca `status='sending'` con >5min como `failed` **y abre un aviso en la Central** (`agent_inbox_items` kind `message_send_stuck`). No toca `queued`: ese estado tiene dueño (el agent-engine lo reprograma por `SEND_QUEUED_RETRY_MS`), y fallarlo perdería un mensaje que iba a salir. No reenvía — un envío doble es peor que un no-envío

### Marca propia (white-label)
- **Una imagen Docker sirve a todas las marcas.** Nada de `NEXT_PUBLIC_*` para marca, nada de `public/favicon.ico`, nada de imagen por revendedor — la imagen viene pre-buildeada y `update.sh` reescribe `APP_IMAGE` incondicionalmente
- **La base de datos está POR ENCIMA del `.env`.** `platform_branding` (instalación) y `organizations.settings.branding` (organización) son la fuente; `APP_NAME`/`APP_LOGO_URL`/`APP_ACCENT_HEX` son **semilla y piso de rollback** (`agent.sh` revierte la imagen, nunca la base de datos)
- **El resolvedor NUNCA lanza.** `lib/branding/instalacao.ts` y `lib/branding/saida.ts` degradan al default del producto y siguen: `branding()` corre en `app/layout.tsx`, y un throw ahí es un 500 en todas las pantallas
- **La salida sin DOM usa `marcaDaSaida()`** (`lib/branding/saida.ts`) — correo, remitente, ícono, `issuer` del MFA. Un hex y un frente legible, tema **claro** siempre. Nunca pases `MarcaResolvida` a una plantilla de correo
- **El PDF de LGPD NUNCA lleva marca.** Nombra al **controlador** (`organizations.legal_name`) y al DPO resuelto. Nombrar ahí al revendedor — que es encargado (operador) — invertiría los papeles en un documento que responde a un derecho legal. Vigilado en `tests/unit/mapas-de-arquitetura.test.ts`
- La fuga de marca en el código la vigila `tests/unit/branding.test.ts` (recorre `app|components|lib|workers|hooks`), con una allowlist que **solo se encoge**. Contexto de venta en [`docs/white-label.md`](docs/white-label.md); mapa en `docs/architecture/marca-propria.architecture.json`

### Doctrina DIRC (antes de agregar un campo)
- **D**uplicar — ¿vive aquí mismo?
- **I**ntegrar — ¿viene de otra tabla vía FK?
- **R**eferenciar — ¿solo un puntero?
- **C**alcular — ¿se puede computar on-demand?

### Modelado
- 5 tablas core del CRM: `crm_pipelines`, `crm_stages`, `crm_leads`, `crm_lead_activities` (timeline polimórfica), `crm_lead_links` (vínculos polimórficos)
- `position_in_stage numeric` (fractional indexing vía `midpoint()`) — **NUNCA `int`**
- `external_id` nullable (un mensaje outbound en `sending` todavía no tiene ID de WAHA)
- `type` es `text` + `check constraint`, **no enum** (un enum es difícil de extender)
  - **Excepción deliberada — columnas de vocabulario ABIERTO:** donde un clon puede tener filas con un valor
    legado (ej.: `crm_lead_activities.type`), el CHECK **no** entra: la constraint haría que el `update.sh`
    del clon se rompiera, y la doctrina de migrations lo prohíbe. En esos casos el vocabulario vive solo en
    TypeScript, el emisor usa una **constante compartida, nunca un string literal**, y la columna queda
    **fuera** del invariante `tests/invariants/vocabulario-banco-x-typescript.test.ts` — que cubre
    solo columnas que YA tienen CHECK. Ver el encabezado de ese archivo antes de "completar" el schema.
- `tags text[]` + índice GIN; se promueve a columna generada solo cuando se vuelve hot path
- `custom_fields jsonb` con schema declarativo en `pipeline.settings.fields`; Zod construido dinámicamente
- `vocabulary jsonb` en el pipeline permite renombrar lead/deal/won/lost (e-commerce: lead=Cliente, deal=Pedido, won=Pagado, lost=Cancelado)

---

## Anti-patterns prohibidos

1. Un string que debería ser FK (ej: `owner_email text` en lugar de `owner_user_id uuid`)
2. Duplicación sin source of truth declarado
3. Evento sin consumer (se emite y nadie lo escucha)
4. FK ausente que se vuelve inferencia por nombre
5. Campo sincronizado por cron cuando debería ser realtime/trigger
6. `jsonb` lock-in (la UI lee un path directo sin schema central)
7. Cascade fantasma (borrar un contact en cascade a messages pierde el historial)
8. Polimórfico sin estandarizar (`target_kind` cada lugar lo graba distinto)
9. **Trigger de Postgres que hace HTTP** (letal — espera la red dentro de la transacción)
10. Service role usado en un request handler sin filtrar `organization_id` manualmente
11. `getSession()` en el backend
12. API key en query string
13. Bearer en plaintext guardado en la DB (debe ser hash SHA256)
14. `console.log` olvidado en código mergeado (usa el logger estructurado o un breadcrumb de Sentry)

---

## Paths importantes

| Path | Contenido |
|---|---|
| `docs/prd/00-prd-master.md` | Visión general, alcance del MVP, KPIs |
| `docs/prd/01-prd-platform-base.md` | Auth, tenancy, RBAC, framework LGPD |
| `docs/prd/02-...06-` | Customer 360, WhatsApp, Pipeline, IA-RAG, Nuvemshop |
| `docs/specs/` | Specs técnicas detalladas (schema SQL, payloads exactos) |
| `docs/business-rules/` | Reglas de negocio fuera del código |
| `docs/research/reference-synthesis.md` | Arquitectura heredada del curso de WAHA |
| `tasks/todo.md` | Workflow de construcción actual |
| `app/globals.css` | **Tailwind 4 es CSS-first: no existe `tailwind.config.ts`.** Tokens en `:root` / `[data-theme]`, puente token → utilidad en el `@theme inline`, alcance del scanner en los `@source`. Vigilado por `tests/unit/tailwind-tokens.test.ts` |
| `lib/api/wrappers.ts` | `ok()`, `fail()`, tipos `ApiSuccess<T>` / `ApiError` |
| `lib/api/errors.ts` | Códigos de error canónicos |
| `lib/env.ts` | Validación Zod de las env vars (lanza en el startup si falta una crítica) |
| `lib/supabase/{browser,server,admin}.ts` | Clients canónicos |
| `app/api/v1/health/route.ts` | Health check (Supabase + Redis + WAHA) |
| `supabase/migrations/` | Schema versionado |
| `docs/runbooks/deploy.md` | **Deploy en producción — léelo ANTES de tocar el VPS** |

---

## Deploy en producción (NO NEGOCIABLE)

**En un VPS que ya tiene su propio proxy inverso (Hostinger, Coolify, Dokploy…), todo
`up -d` lleva LOS DOS archivos de compose:**

```bash
docker compose -f docker-compose.prod.yml -f docker-compose.traefik.yml --env-file .env up -d app
```

Omitir `-f docker-compose.traefik.yml` recrea el contenedor sin las labels de
enrutamiento; el Traefik del hosting deja de verlo y **el dominio entero
responde `404 page not found`** — con el contenedor `healthy`, porque el
healthcheck es un probe TCP interno y no sabe nada de enrutamiento.

Después de cualquier deploy, confirma que el dominio responde **307** (redirige
al login) y no 404. Verificaciones y el caso de build local en
`docs/runbooks/deploy.md`.

El camino normal **no construye nada en el VPS**: commit → push → PR → merge en
`main` → el CI publica en GHCR → el VPS hace pull. Una imagen construida en el VPS es una
excepción de emergencia y es deuda: existe solo en ese disco y cualquier `up -d` sin
`APP_PULL_POLICY=never` la sustituye en silencio.

Esa frase ya fue una verdad a medias: valía para el `app` y era falsa para el producto,
porque el servicio `worker` no tenía `image:` — se construía en el VPS de cada
cliente y ningún `update.sh` lo reconstruía nunca. Hoy los tres servicios
nuestros (`app`, `worker`, `scheduler`) son imágenes publicadas, y un test
rechaza que vuelva ese patrón. Ver la doctrina de abajo.

---

## Packaging y distribución — DOCTRINA (NO NEGOCIABLE)

Ley completa en [`docs/doctrine/packaging.md`](docs/doctrine/packaging.md);
decisiones estructurales y lo que se rechazó en
[`docs/adr/0001-packaging-e-distribuicao.md`](docs/adr/0001-packaging-e-distribuicao.md).
Lo no negociable, en cuatro líneas:

1. **Ningún servicio de `docker-compose.prod.yml` construye en la máquina del
   cliente.** Todo servicio declara `image:` de una imagen publicada; `build:`
   solo existe **al lado**, como escape. Un servicio solo-`build:` es invisible para
   `docker compose pull` e inmune a `up -d` sin `--build` — no solo es caro de
   instalar, es que **nunca se actualiza**.
2. **Publicar es un acto del CI.** Nunca de tu máquina: un build ARM local no corre
   en el VPS amd64 del cliente, y la falla solo aparece en su `up -d`. El job
   `imagens-ok` falla cuando cualquiera de las tres imágenes no se construye, y
   **es status check obligatorio desde el 2026-08-13** — la branch protection tiene
   `verify, build-and-size, invariants, e2e, imagens-ok`. (Este párrafo decía
   "todavía no es obligatorio" hasta el 2026-08-14; la activación era el paso final del
   merge de la doctrina y ocurrió.) Compruébalo en la fuente antes de confiar en esta línea.
3. **La instalación del cliente apunta a un número de versión, nunca a un tag móvil.**
   `latest` aquí significa **la punta de `main`**, no la última release — quien quiere la
   última release usa `stable`. `pull_policy` sigue a la mutabilidad del tag:
   inmutable → `missing`, móvil → `always`.
4. **Una dependencia upstream se referencia con tag fijo, nunca se republica.**
   Aplica a WAHA (licenciado — republicarlo es un pasivo jurídico), Redis, Caddy y
   `serverless-redis-http`.

Un bump de versión **no puede** exigir que el operador del VPS edite `.env`, el compose
o cualquier archivo a mano. Si lo exige, no entra: se vuelve un issue con plan de
migración y va a una major.
---

## Extensiones — DOCTRINA (NO NEGOCIABLE)

Ley completa en [`docs/doctrine/extensoes.md`](docs/doctrine/extensoes.md); el
contrato que existe hoy en
[`docs/specs/extensoes-declarativas-v1.md`](docs/specs/extensoes-declarativas-v1.md).
La pregunta que decide el destino de un cambio no es "¿esto le sirve a mucha gente?",
sino **"si ninguna organización activa esto, ¿la operación común sigue entera?"**.
Lo no negociable:

1. **El núcleo sigue siendo útil con cero extensiones.** Identidad, autorización,
   aislamiento, auditoría, contratos y cadena de envío son núcleo; jornada de nicho,
   apariencia e integración con datos y mantenimiento propios pueden ser extensión.
2. **Una extensión pide una capacidad con nombre; no importa código interno ni lee la base de datos.**
   Instalar no concede autoridad: toda escritura revalida actor, organización y rol
   actuales en la base de datos.
3. **La instancia decide el paquete; la organización decide el uso.** Instalar, actualizar,
   deshacer y quitar le corresponden al administrador de la instalación; activar y configurar, al
   administrador de la organización. La plataforma no reactiva una decisión de la organización.
4. **Toda operación es un recibo idempotente con salida por la pantalla, y todo cambio de
   puntero exige la revisión que vio la pantalla.** Quitar es lógico y preserva los datos.
5. **No anunciar lo que no existe** (SDK, código aislado, marketplace público), y
   no extraer del núcleo un recurso ya distribuido sin equivalencia y migración.
6. **Un módulo oficial con datos no pone tablas en el baseline para todos**
   ([ADR-0002](docs/adr/0002-tabelas-de-modulo-num-banco-so.md), aceptada el 17/09/2026). Una sola
   base de datos, schema `public`; las tablas nacen por una función provisionadora fija del módulo, cuando este se
   **instala en la instancia**. Nadie opera una segunda base de datos — es decisión del dueño, y sería imposible
   con llaves foráneas hacia el núcleo.

---

## Cómo correr en local

```bash
nvm use                    # node 22
npm install
cp .env.example .env.local  # completar
docker compose up -d        # WAHA local
npm run dev                 # http://localhost:3000
```

Ver `README.md` para los detalles del setup.

---

## Tests

```bash
pnpm typecheck   # tsc --noEmit -p tsconfig.typecheck.json (incluye tests/)
pnpm lint        # eslint next/core-web-vitals
pnpm test:unit   # Vitest (NO incluye tests/invariants/** — ver abajo)
pnpm test:db     # Postgres efímero + baseline install/update + 364 invariantes
pnpm test:e2e    # Playwright (requiere dev server)
```

**⚠️ `test:unit` NO es `tests/unit/`.** El script es `vitest run` **sin ruta**, y alcanza
el repositorio entero — incluidos los tests co-ubicados en `lib/`, `app/`, `components/` y `hooks/`.
Medido el 2026-08-28: `vitest run` alcanza **566 archivos**; `tests/unit/` tiene **388**.
Los 178 de fuera son 133 en `lib/`, 37 en `app/`, 3 en `components/`, 1 en `hooks/` y 4 en `tests/`.

Quien lee el nombre del script y corre `vitest run tests/unit` obtiene un **verde más chico y más fácil** sin
darse cuenta de que lo obtuvo — y fue lo que pasó en un PR: la suite se reportó en verde, y lo que
estaba verde era el recorte. El comando que vale es `pnpm test:unit`, sin ruta.

Dos trampas hermanas, las dos pagadas el mismo día:

- **Un gate escogido no es la suite.** `typecheck`, `lint`, `lint:channels` y los archivos de cerca
  pueden estar todos en verde mientras la suite tiene 17 fallas — ninguno de ellos toca el archivo que
  se rompió. Antes de abrir un PR, corre la suite, no los gates que recuerdas.
- **No recortes la salida.** `| tail -8` se queda con el pie y tira los NOMBRES de los archivos que
  fallaron, que es el único dato que permite conciliar después. Redirige y filtra:

  ```bash
  pnpm test:unit > /tmp/vt.log 2>&1; echo "exit=$?"
  grep -aE "Test Files|Tests " /tmp/vt.log | tail -2                   # ← la AUTORIDAD
  grep -aE "^ *FAIL " /tmp/vt.log | sed 's/ > .*//' | sort | uniq -c   # archivos + conteo
  ```

  **El pie es la autoridad; el `grep FAIL` es una conveniencia — y puede devolver
  vacío CON fallas.** Medido: en una ejecución sin TTY el reporter por default a veces
  imprime solo el resumen, y los nombres de los archivos en rojo nunca llegan a
  escribirse. Una ronda con `3 failed` produjo un log de 629 bytes donde `FAIL`
  no aparece en ninguna posición — y el vacío de esa sonda se lee exactamente como
  "ninguna falla".

  Por eso **compara las dos salidas antes de concluir** — y compara la línea
  correcta: `Test Files N failed` cuenta ARCHIVOS, `Tests N failed` cuenta CASOS, y el
  `uniq -c` del `grep` suma CASOS. El control es contra la segunda línea:

  ```bash
  r=$(grep -aE "^ *Tests " /tmp/vt.log | tail -1 | grep -oE "[0-9]+ failed" | head -1)
  g=$(grep -acE "^ *FAIL " /tmp/vt.log)
  echo "pie: ${r:-0 failed} | grep contó: $g"   # tienen que coincidir
  ```

  Si no coinciden, antes de diagnosticar *"sonda ciega"* y volver a correr la suite entera
  con `--reporter=verbose`, revisa si la divergencia se explica por una **falla de
  recolección o de hook** (que Vitest imprime en la sección dedicada `Failed Suites`,
  sumándose a las líneas `FAIL` sin entrar en el pie de casos `Tests ... failed`):

  ```bash
  grep -aoE "Failed Suites [0-9]+" /tmp/vt.log | grep -oE "[0-9]+"   # > 0 ⇒ un archivo/suite falló SIN ser por un caso
  grep -aqE "^ *Test Files" /tmp/vt.log && echo "log completo" || echo "log truncado — el cero no vale"
  ```

  El segundo comando es necesario: la sección solo aparece cuando existe una falla de
  suite, así que un log truncado o un comando que no corrió también devuelven cero.
  Si `Failed Suites` es > 0 (y el log está completo), la cuenta cierra: `Tests failed` + `Failed Suites` = `grep FAIL`.
  La causa (falla de sintaxis en la recolección o `Hook timed out` en `beforeAll`) está en el propio log.
  Reserva el diagnóstico de *"sonda ciega"* (cambiar a `--reporter=verbose`) para cuando
  las secciones tampoco expliquen la divergencia.

  **Y las dos pueden coincidir en cero con la suite reprobada.** Vitest sale con
  `exit=1` cuando hay un **error no manejado** durante la ejecución, aun con todos los
  tests pasando — y ese error aparece en una TERCERA línea del pie, que
  ninguna de las dos sondas de arriba lee:

  ```
   Test Files  866 passed (866)
        Tests  8904 passed | 1 expected fail (8905)
       Errors  1 error          ← solo el exit code vio esto
  ```

  Medido el 2026-09-15: `EnvironmentTeardownError: [vitest-worker]: Closing rpc
  while "onUserConsoleLog" was pending`, bajo carga. Pie `0 failed`, `grep FAIL`
  vacío, exit 1. **El exit code es la autoridad; el pie y el grep son su
  explicación.** Cuando el exit diverge de los dos, lee la línea `Errors` antes de concluir
  cualquier cosa:

  ```bash
  grep -aE "^ *Errors " /tmp/vt.log      # lo esperado es vacío
  ```

  (Comparar contra `Test Files` da una
  divergencia falsa: `2 failed` de archivos contra `7` de casos parece defecto
  de la sonda y es solo una regla cambiada.)

**Rojo local que NO es tuyo:** `lib/ai/dispatcher/rate-limit.test.ts` falla en 5 casos, con
15s de timeout cada uno, cuando el `.env.local` tiene `UPSTASH_REDIS_REST_URL`/`TOKEN` y el Redis al
que apuntan **no está arriba** (en este repo es el `serverless-redis-http` local, no la nube).
`tests/setup/vitest.setup.ts` carga el `.env.local` dentro del `process.env`, y el módulo
solo usa el contador en memoria cuando esas variables están **ausentes**. Probado en los dos sentidos.
En el CI no hay ningún `UPSTASH`, así que ahí el camino es el contador en memoria y el archivo pasa.

**Los invariantes no están en `test:unit`.** `vitest.config.ts` excluye `tests/invariants/**` a propósito: esa suite necesita un Postgres real y corre vía `vitest.db.config.ts`, orquestada por `scripts/test-db.sh`. Correr solo `pnpm test:unit` y concluir "todo está en verde" es un falso verde — el aislamiento RLS no se ejercitó.

Checks **obligatorios** en la branch protection de `main` (verificado en la configuración, no solo en el papel):

- **`verify`** (`ci.yml`) — typecheck + lint + test:unit.
- **`invariants`** (`ci.yml`) — **job de fachada**: no corre ninguna suite; falla cuando la matriz `invariants-majors` no cierra en `success`. Quien corre es la matriz, una pierna por cada major de Postgres que el producto dice soportar, y cada pierna hace dos pasadas: `pnpm test:db` (baseline en modo install con `ON_ERROR_STOP=1` y update, más los invariantes, incluido el aislamiento RLS entre 2 organizaciones) y `pnpm test:db:update` (actualización de una base CON datos). Para saber qué majors hay hoy, pregúntale al archivo en vez de a esta línea: `awk '/^  invariants-majors:/,/^  [a-z-]+:/' .github/workflows/ci.yml | grep -A6 'matrix:'`.
- **`build-and-size`** (`perf.yml`) — `pnpm build` en Node 22.
- **`e2e`** (`e2e.yml`) — levanta Supabase local, aplica el `baseline.sql` y corre **todas las specs de Playwright menos las que declara `FORA_DO_CI`** — **en un PR que alcanza algo que él mide**. Un PR solo de documentación, de tests de otra suite, de fragmentos o de un workflow ajeno se salta las partes (regla en `scripts/pr-alcanca-o-e2e.sh`; en la duda, corre), y ahí el `e2e` verde **no prueba ninguna pantalla**. El número se quitó de aquí a propósito: se pudrió **cinco** veces (la quinta el 2026-08-24, cuando entró `inbox-quem-manda.spec.ts`), y la condición que el PR #242 puso para dejar de recontar ya había vencido en la cuarta. Quien necesite el número corre el comando de abajo — un comando no envejece. Cuáles quedan fuera, y por qué, lo dice la propia variable — **no confíes en esta línea, léela**:

  ```bash
  git show origin/main:.github/workflows/e2e.yml | \
    python3 -c "import sys,re; y=sys.stdin.read(); print(sorted({s for _,c in re.findall(r'(FORA_DO_CI):\s*>-\n((?:[ ]{8,}.*\n)+)',y) for s in re.findall(r'[a-z0-9-]+\.spec\.ts',c)}))"
  ```

  **Esta frase ya envejeció TRES veces, y es el párrafo que denuncia las afirmaciones que envejecen.** Decía "la **única** fuera es `vps-fresh-onboarding`" cuando la variable ya listaba dos (2026-09-04, `inbox-tempo-real`); después siguió diciendo que el recorrido de instalación desde cero no tenía gate — y el 2026-09-19 el **#983** (@webtecnica) puso `vps-fresh-onboarding.spec.ts` a correr en el CI, con WAHA y Redis de verdad, así que la frase pasó a decir lo contrario del estado.

  Por eso sale y no vuelve: la pregunta "¿el recorrido de instalación desde cero tiene gate?" se responde con un **comando**, con el de arriba (lo que declara `FORA_DO_CI`) y con este, que dice a quién **invoca** el CI:

  ```bash
  git show origin/main:.github/workflows/e2e.yml | python3 -c "import sys,re; y=sys.stdin.read(); print('vps-fresh-onboarding no CI:', 'vps-fresh-onboarding.spec.ts' in {s for _,c in re.findall(r'(SPECS_PARTE_\d+):\s*>-\n((?:[ ]{8,}.*\n)+)',y) for s in re.findall(r'[a-z0-9-]+\.spec\.ts',c)})"
  ```

  Las dos salidas se cierran una contra la otra porque `tests/unit/e2e-cobertura-completa.test.ts` rechaza una spec que no esté ni en una `SPECS_PARTE_*` ni en `FORA_DO_CI`: la ausencia en la primera salida es presencia en la segunda, y ninguna spec cae en el hueco entre las dos.

  Lo que **no** envejece y es lo que importa: `vps-fresh-onboarding` es la **P0** de la doctrina de QA Visual porque la instalación desde cero es el producto que se vende. Tener gate no exime de la prueba por la pantalla (DoD 12) — el gate prueba que no hubo regresión, no que la experiencia quedó bien. Y la salvedad del inicio del ítem sigue en pie: en un PR que se salta las partes, el verde no prueba ninguna pantalla, incluida la de la instalación desde cero.

  **No confíes en un `grep` sobre el archivo entero.** `grep -oE '[a-z0-9-]+\.spec\.ts' .github/workflows/e2e.yml | sort -u | wc -l` cuenta a quien es CITADO, no a quien es INVOCADO: `FORA_DO_CI` es una variable YAML como las otras y entra en la cuenta. (Hasta el 2026-08-14 este párrafo culpaba a "menciones en comentarios", y eso es falso — medido, el conjunto de specs citadas fuera de una variable es **vacío**.) Lo que corre son las `SPECS_PARTE_*`:

  ```bash
  ls tests/e2e/*.spec.ts | wc -l                    # cuántas existen
  python3 - <<'PY'                                  # cuántas invoca el CI
  import re
  y = open(".github/workflows/e2e.yml", encoding="utf-8").read()
  print(len({s for _, c in re.findall(r'(SPECS_PARTE_\d+):\s*>-\n((?:[ ]{8,}.*\n)+)', y)
               for s in re.findall(r'[a-z0-9-]+\.spec\.ts', c)}))
  PY
  ```

  **Por qué ya no hay número aquí.** El arreglo que pedía este párrafo era poner la prosa bajo un gate — `tests/unit/e2e-cobertura-completa.test.ts` cobrando también el texto de aquí. Quitar el número es mejor y más barato: no hay nada que vigilar, y la diferencia entre disco y CI sigue vigilada donde importa, en el propio test, que rechaza toda spec nueva que no esté en `SPECS_PARTE_*` o en `FORA_DO_CI` **con motivo escrito**. Prosa que ningún gate lee es prosa que diverge; prosa que no afirma un número no tiene cómo divergir.
- **`imagens-ok`** (`publish-image.yml`) — falla cuando cualquiera de las tres imágenes Docker no se construye. **Es obligatorio desde el 2026-08-13**; este archivo decía lo contrario en otro párrafo (ver la doctrina de packaging de arriba, ya corregida).

Los **cinco** son **obligatorios** — medido el 2026-08-14 en la branch protection:

```console
$ gh api repos/melgarafael/DeskcommCRM/branches/main/protection --jq '.required_status_checks.contexts|join(", ")'
verify, build-and-size, invariants, e2e, imagens-ok
```

Dos correcciones que este bloque ya pagó: `e2e` entró a la lista después de que se escribiera el archivo, y
la versión anterior decía que "todavía no es obligatorio"; después entró `imagens-ok` y el archivo
siguió diciendo "cuatro". Una triage que lea cualquiera de esas versiones mide contra la regla equivocada —
que es el modo de falla nº 1 del procedimiento de triage. **Vuelve a comprobar en la fuente antes de confiar en
cualquier lista de aquí**, con el comando de arriba.

**Dónde corren los jobs.** La cuenta tiene el plan Pro: hasta **40** jobs simultáneos en las máquinas de GitHub
(medidos 39 el 18/09/2026, con 180 en la cola). Los jobs pesados del trabajo **nuestro** (push a `main`
y PR de una branch de este repositorio) pueden ir al **ejecutor propio** (`infra/executor-proprio/`)
cuando la variable de repositorio `EXECUTOR_PROPRIO` vale `ligado`; un PR de fork corre siempre en GitHub,
y la publicación de `main` también. Dos reglas que no se negocian:

- **La guarda contra forks vive en la máquina, no en el YAML.** En un PR de fork, GitHub corre el workflow del
  fork, que puede reescribir `runs-on:`. Quien lo rechaza es `infra/executor-proprio/so-o-que-e-nosso.sh`,
  grabado en la imagen como hook de entrada del runner. Cambiar la expresión de `runs-on` no es cambiar la
  seguridad — y aflojar la guarda sí lo es.
- **Una imagen que instala el parque nunca se construye en nuestra máquina.** `build-and-push` y
  `promover-stable` se quedan en `ubuntu-latest`; los jobs `*-sobe` solo van a la máquina en un PR.

Vigilado por `tests/unit/executor-proprio-so-roda-o-que-e-nosso.test.ts`. Botón de emergencia:
borrar la variable `EXECUTOR_PROPRIO` — los jobs nuevos vuelven al instante a GitHub. **La cola de merge
(merge queue) de GitHub no está disponible** en este repositorio (cuenta personal; medido el 18/09/2026:
la regla se rechaza con 422 y una regla común con el mismo formato se acepta) — la integración en lote de la
triage (`triagem/TRIAGEM.md` §3-quinquies) es lo que cumple ese papel.

Al tocar schema, RLS, RBAC, asignación, alcance, enrutamiento, follow-up, webhooks o automatizaciones: corre `pnpm test:db` **en local** antes de abrir el PR. Es el único camino que ejercita el `baseline.sql` que realmente aplica quien se auto-hospeda.

---

## QA Visual con Recursos Reales — DOCTRINA (producto self-host)

**DeskcommCRM se distribuye open source: la experiencia de quien lo instala en un VPS ES el producto.** Toda feature nueva (o fix de un comportamiento visible) DEBE probarse como **un usuario no técnico la usaría de verdad** — por el frontend, en un entorno que imita la instalación desde cero — antes de darla por "lista". No es opcional; es criterio de aceptación de toda sesión que toque UI o un flujo de usuario.

**Qué significa "recurso real" (y qué NO cuenta):**
- **Cuenta.** Prueba por la pantalla, manejando el navegador (Playwright), iniciando sesión con una cuenta de prueba real. `curl`/una llamada a la API **no** prueban UX — validan el backend, pero no lo que el usuario ve, pulsa y entiende. Usa curl solo como diagnóstico.
- **Base de datos fresca estilo VPS.** Postgres limpio aplicado desde `supabase/baseline.sql` (no desde las `migrations/` — la cadena fresh no levanta) + `scripts/bootstrap-owner.ts` (lo que hace `install.sh`). El entorno del test = lo que tiene el clon recién instalado: sin tus datos, sin tus envs opcionales.
- **Dependencias como en el VPS.** WAHA local, Redis local (`redis` + `serverless-redis-http`), drain del cron vía endpoint. Y **prueba con los envs opcionales AUSENTES** (ej.: sin `RESEND_API_KEY`) — es el estado real de un primer deploy, y es donde viven los peores bugs de primera impresión.
- **Efecto colateral externo probado con un receptor real.** Webhook outbound, envío — levanta un receptor HTTP de verdad y prueba lo que llegó (o que se bloqueó). Un mock no estresa el egress real (anti-SSRF, proyección del payload, https en prod).

**Prioridad: la primera impresión por encima de todo.** El onboarding y las primeras acciones (crear cuenta, conectar un canal, primer lead, primera invitación) son la primera impresión del usuario — un bug ahí es abandono. Prueba esos caminos primero y con el mayor rigor.

**Registro obligatorio (si no, el progreso es invisible):**
- Mapa de recorridos vivo en `docs/testing/user-journey-map.md` — casos por recorrido, prioridad (`[P0]` primera impresión) y hallazgos. Actualízalo cuando agregues cobertura o encuentres un bug.
- Specs en `tests/e2e/*.spec.ts` que manejan el **frontend** (no solo la API). Evidencia visual (screenshot/trace) en `evidence/<entrega>/`, que está versionada; nunca en una carpeta que el `.gitignore` ignora, porque si no la prueba no sale de tu máquina.
- Bug encontrado al ejecutar → **se corrige en la causa raíz**, con migration versionada si toca el schema (ver la doctrina de abajo), commit propio, y re-test en verde como prueba.

**Medidas de front-end con herramienta, nunca a ojo** (`getBoundingClientRect`/`getComputedStyle` en Playwright). Ver `feedback_protocolo_execucao_visivel` en la memoria.

**Receta de entorno fresco (no obvia):** base de datos = `baseline.sql` en un Supabase local **pg15** (`config.toml major_version = 15`). Ya fue pg17, por culpa de 9 `GRANT MAINTAIN` que `pg_dump` emitió por su cuenta; hoy quien cuida el piso es `tests/unit/baseline-no-piso-do-postgres.test.ts`; `next build` + `next start` (producción — `next dev` compila demasiado lento y Turbopack rompe `cookies()`); **worktree con `node_modules` real, nunca symlink** (Turbopack rechaza un symlink "out of filesystem root") y **fuera de `/tmp`** (se limpia a mitad de la sesión — haz commit de cada hito). Detalles en [[project_invite_e2e_and_bugs]].

---

## Higiene de branches — DOCTRINA (NO NEGOCIABLE)

**`main` es producción y es la fuente de la verdad. Toda branch empieza y se mantiene actualizada con `main`.** El trabajo iniciado en una branch atrasada genera conflictos y retrabajo — es la causa número uno de desastres en un entorno multi-sesión. Regla:

1. **ANTES de empezar CUALQUIER trabajo en una branch, actualízala con `main`:** `git fetch origin && git merge origin/main` (trae producción hacia adentro). Si la branch todavía no tiene commits propios, es fast-forward puro (`git merge --ff-only origin/main`). No se programa antes de eso.
2. **NUNCA `reset --hard`/force para "actualizar"** — borra trabajo. Solo dos caminos: **fast-forward** (branch sin commits propios) o **merge de `main` hacia adentro** (preserva los dos lados). `main` nunca se reescribe.
3. **NUNCA toques una branch/worktree con working tree sucio que no es tuyo.** Antes de actualizar cualquier branch, revisa `git status` y `git worktree list` — si está sucia y es de otra sesión, **déjala en paz** y avisa. El merge solo entra en un árbol limpio.
4. **Cuando una feature entra a `main`, todas las demás branches quedan atrasadas al instante.** Quien vaya a retomar cualquiera de ellas aplica primero la regla 1. Al terminar una feature, considera propagar `main` a las branches vivas limpias (FF a las que no tienen trabajo propio; merge en las divergentes limpias; saltar las sucias/conflictivas y reportarlo).
5. **Conflicto al actualizar = detente y resuélvelo con cabeza** (o escálalo), nunca elijas un lado en automático en una branch que no es tuya. Preservar el trabajo > branch "verde rápido".

---

## Migrations y base de datos — DOCTRINA (proyecto open source)

**Este proyecto es open source. Todo cambio de schema DEBE salir como una migration versionada** — quien clonó una versión antigua de la base de datos tiene que poder actualizar aplicando las migrations en orden. **Nunca** apliques un `ALTER`/`CREATE` suelto en la base sin el archivo correspondiente. Esto es criterio de aceptación de TODA sesión, no es opcional.

Proceso estándar (síguelo siempre):

1. **Archivo versionado** en `supabase/migrations/` con el patrón del repo: `<timestamp>_<NNNN>_<slug>.sql` (ej.: `20260706210000_0027_whatsapp_conversation_unification.sql`). `NNNN` es el siguiente número secuencial — y **no** es el del último archivo del listado:

   ```bash
   # LA POBLACIÓN de la pregunta: main del PRODUCTO (el remoto que apunta a
   # melgarafael/DeskcommCRM, con cualquier nombre) + TODO PR ABIERTO, incluso de
   # fork. El `ls` de abajo mide el DISCO, que responde una pregunta menor.
   pnpm checar:colisao-de-migration   # declara lo que midió y lo que no midió
   ```

   Si la lectura es manual, tres cosas son obligatorias: un `git fetch` antes (el árbol al día
   no es la main actual), `git ls-tree` de la main del PRODUCTO y no `ls` del disco, y el `NNNN` sacado
   con el **ancla del nombre canónico** aplicada al nombre **sin la carpeta** —
   `sed 's#.*/##' | sed -nE 's#^[0-9]{14}_([0-9]{4})_.*#\1#p'`. El
   `ls | grep -oE '_[0-9]{4}_'` que estaba aquí agarraba un `_NNNN_` del SLUG (con
   `…_0326_relatorio_2024_anual.sql` el techo se volvía 2024) y medía el árbol de
   trabajo, donde la 0336 ya podía estar reservada por un PR abierto (#1273).

   El `checar` mide el archivo que **ya** agregaste: sin migration nueva, responde
   `OK — nenhuma migration acrescentada` y no da número. Crea el archivo con un número provisional
   y córrelo; o, para asignar antes, usa la enumeración de lo que está en curso en `triagem/TRIAGEM.md`
   (modo de falla 37). El techo es la main **más** todo lo que está en curso, en NNNN **y** en timestamp.

   El nombre del archivo empieza por el **timestamp**, y el timestamp y el `NNNN` pueden no coincidir: el
   09/09/2026 el `ls | tail -1` devolvía el `_0230_` (timestamp del 07/09) mientras que el mayor `NNNN`
   era `_0231_` (timestamp del 05/09). Un contribuyente externo siguió la instrucción anterior al pie de la
   letra, eligió `0231`, y el `manifest-x-migrations` rechazó su PR por colisión — la
   instrucción era la equivocada, no él. Ordena por el número, nunca por el listado.

   **Y el número libre hoy puede estar tomado cuando entre tu PR.** La colisión solo aparece
   cuando se mergea el SEGUNDO PR de schema — medido el 19/09/2026: **11 PRs abiertos colisionaban
   con `main` con los cinco checks obligatorios en verde**. El `verify` **ya ejecuta** la guarda
   (`pnpm checar:colisao-de-migration`, el alias de `scripts/checar-colisao-de-migration.sh` —
   buscar el nombre del archivo en el `ci.yml` devuelve cero y miente), y aun así los 12 pasaron:
   cada uno midió la `main` del día en que corrió — el `verify` del #965 terminó el 16/09 y sigue en verde.
   Por eso hay dos capas más: el CI rechaza cuando **un número de este PR fue tomado** por una
   migration que entró a la base después de la vista previa (colisión, nunca atraso — un PR atrasado y sin colisión
   sigue en verde), y fuera de `pull_request` recorre el árbol entero — ningún `NNNN` ni timestamp puede aparecer dos veces en `main`. Antes de elegir el número cuando haya otros PRs de schema en curso, pídeselo
   a quien esté asignando en la ronda: **no hay reserva, quien mergea primero se queda con el número**.
   Para ver lo que está tomado ahora, incluido lo que todavía no se mergeó:

   ```bash
   pnpm checar:colisao-de-migration          # mide TU PR contra origin/main
   ```
2. **Idempotente siempre que se pueda**: `add column if not exists`, `create ... if not exists`, `create or replace function`. Una migration debe poder re-aplicarse sin romperse ni duplicar su efecto.
3. **Portable a `psql` puro** (los clones pueden no usar el MCP/CLI de Supabase): **sin** `create temporary table ... on commit drop` fuera de una transacción explícita; **sin** `BEGIN`/`COMMIT` explícito (el runner ya la envuelve en una transacción, como las demás migrations). Prefiere CTEs, subqueries de ventana y columnas-mapa (ej.: `is_merged_into`) a las temp tables.
4. **Data migrations genéricas**: si la migration corrige/deduplica datos, escríbela pensando en CUALQUIER base de un clon (no hardcodees IDs de tu tenant). Vuelve a apuntar las FKs consultando el catálogo (mapa de FKs de `information_schema`) para no perder historial.
5. **Regístrala en el MANIFEST**: agrega una línea en `supabase/migrations/MANIFEST.md` (tabla "Applied") describiendo versión, nombre y el QUÉ/POR QUÉ.
6. **Refléjala en `supabase/baseline.sql` (OBLIGATORIO — es lo que aplica el kit self-host).** El baseline es un dump `--schema-only` + un **apéndice idempotente** al final del archivo (bloques rotulados `-- ---- <cosa> (migration NNNN) ----`). El kit de HostGator aplica **solo el baseline.sql**, tanto en `install.sh` (base nueva, `ON_ERROR_STOP=1`) como en `update.sh` (re-aplica sobre una base existente, **sin** `ON_ERROR_STOP`). Así que todo cambio de schema posterior al snapshot DEBE agregarse al apéndice, **idempotente y auto-curativo**: `add column if not exists`, `create ... if not exists`, `create or replace function`, y — si el cambio agrega una constraint — **deduplicar/corregir los datos ANTES** de crear la constraint (si no, el `update.sh` de un clon con bugs se rompe). Sin esto, los clones no reciben el cambio (o se rompen al actualizar). Una migration agregada solo en `migrations/` pero no en el baseline **no llega a quienes se auto-hospedan**.
7. **Aplícala y pruébala**: aplícala vía `mcp__plugin_supabase_supabase__apply_migration` (o `supabase db push`), captura el estado ANTES/DESPUÉS y prueba invariantes (ej.: un conteo de filas que no puede cambiar). Si tocaste el contrato, regenera `lib/database.types.ts`. Para cambios de schema en el kit, valida el baseline en un Postgres desechable (`pgvector/pgvector:pg15` + extensiones) aplicando `install` (fresh, `ON_ERROR_STOP=1`) y `update` (re-aplicar, sin el flag) — los dos tienen que pasar.
8. **Backfill de datos rotos existentes**: una constraint nueva falla si los datos actuales la violan — la migration (y el apéndice del baseline) debe deduplicar/corregir ANTES de crear la constraint.
9. **Una función nueva en `public` nace EXPUESTA — revoca LOS DOS orígenes.** Toda `create function` en el schema `public` termina con:

   ```sql
   revoke execute on function public.fn_x(...) from public, anon;
   grant  execute on function public.fn_x(...) to <solo quien lo necesita>;
   ```

   Son dos orígenes distintos de `EXECUTE`, y tratar solo uno deja la función expuesta con el gate en verde: **(A)** el grant directo a `anon` del `ALTER DEFAULT PRIVILEGES ... GRANT ALL ON FUNCTIONS TO anon` del baseline, que vale para toda función creada después de él — es decir, para todo apéndice nuevo — y que `revoke from public` **no** quita; **(B)** el grant a `PUBLIC` que Postgres le da a cualquier función al crearla, que `revoke from anon` **no** quita. Sin los dos, PostgREST expone la función como un RPC alcanzable con la anon key, que va al navegador. Vigilado por `tests/invariants/hardening-definer-varredura.test.ts`, que recorre todas las `security definer` de `public` (issue #128 — la versión anterior revisaba una lista fija de 6, y 8 de 25 estaban expuestas).

10. **Leer el baseline con `grep` sobre el archivo entero mide la definición EQUIVOCADA.** El
    `baseline.sql` es dump + apéndice, así que la misma función aparece **varias
    veces** — y la que vale es la **última**, porque el archivo se aplica entero y
    en orden. Medido el 2026-09-20: `fn_meet_action` tenía **cuatro**
    definiciones; la primera (el cuerpo del dump) todavía traía `errcode='40001'` en las
    tres negativas permanentes, y la última — la que instala la base de datos — traía
    `PT409`. Una sonda de `grep`/`awk` anclada en la primera ocurrencia afirmó
    sobre el producto lo opuesto de lo que el producto hace. Lo mismo vale para
    `fn_lgpd_cascade_redact_contact`, que tiene ocho.

    **Las dos formas correctas**, y la primera decide:

    ```bash
    # (a) PREGÚNTALE A LA BASE, después de aplicar — es lo que tendrá el cliente
    pnpm test:db tests/invariants/<un caso que consulte>  # o, en un psql con el baseline ya aplicado:
    psql "$URL" -Atc "select pg_get_functiondef(p.oid) from pg_proc p
      join pg_namespace n on n.oid=p.pronamespace
      where n.nspname='public' and p.proname='fn_x'"
    ```

    ```bash
    # (b) ANCLA EN LA ÚLTIMA definición, cuando solo tengas el archivo a la mano
    python3 -c "
    s=open('supabase/baseline.sql').read()
    i=s.rfind('create or replace function public.fn_x')   # rfind, nunca find
    print(s[i:s.index('\$\$;', i)+3])"
    ```

    Contar ocurrencias en el archivo entero responde *"el archivo menciona"*, nunca
    *"la base de datos hace"*. Las dos preguntas divergen siempre que hay apéndice — y
    el apéndice es el mecanismo estándar de esta casa.

**Resumen del flujo de un cambio de schema:** archivo en `migrations/` (fuente de la verdad para el Supabase CLI) **+** apéndice idempotente en `baseline.sql` (para el kit self-host) **+** línea en el MANIFEST. Los dos artefactos de schema van juntos. Nunca edites migrations ya aplicadas — corrige con un "forward-fix" nuevo (y otro apéndice en el baseline).

---

## Skills relevantes a usar (Claude Code)

**Guías embutidas en este repositorio** (`.claude/skills/`, espejo generado de `.agents/skills/` — la
misma tabla vale para Codex, Cursor, OpenCode y Antigravity; ver `AGENTS.md`). Para tenerlas en
cualquier carpeta, `bash scripts/instalar-guias.sh`; si editas una guía en una branch, córrelo con `--fonte .`
en ese clon — en Claude Code la skill GLOBAL le gana a la del proyecto con el mismo nombre:

- `deskcomm-instalar` — instalar, actualizar o arreglar la instalación en un VPS
- `deskcomm-cliente-novo` — configurar el CRM para un cliente o nicho (agentes, enrutadores, follow-ups, conocimiento)
- `deskcomm-metricas` — desempeño, conversión, costo de IA, embudo, reporte
- `deskcomm-prompt` — afinar el prompt de un agente que no rinde
- `deskcomm-contribuir` — el espejo de la triage, antes del PR; se queda callado para el mantenedor
- `deskcomm-extensao` — crear una extensión en lugar de un PR al núcleo: regla de destino, contrato del paquete y envío
- `deskcomm-doutrina` — las tres reglas que más cuestan, antes de escribir código

Las guías tienen página pública en [deskcomm.com.br/guias](https://www.deskcomm.com.br/guias), escrita
a mano en `deskcomm-site/conteudo/guias.ts`: una guía creada, renombrada o con un comando nuevo pide el mismo
cambio allá — si no, la página enseña una guía que no existe. Esa y la del changelog salen del mismo PR del
`deskcomm-site`; mientras las dos no respondan 200, vale el `curl` que abre la sección "A vitrine" de
[`docs/doctrine/versionamento.md`](docs/doctrine/versionamento.md), no la frase de arriba.

- `superpowers:brainstorming` — antes de implementar una feature no trivial
- `superpowers:writing-plans` — para una tarea con más de 1 etapa de DB/API
- `superpowers:test-driven-development` — feature crítica (LGPD, RLS, anti-baneo)
- `superpowers:systematic-debugging` — bugs reportados
- `superpowers:verification-before-completion` — antes de declarar "listo"
- `tomik-db-doctrine` — referencia cruzada de la doctrina de schema
- `supabase:supabase` — cualquier tarea con Supabase
- `vercel:nextjs` — App Router, Server Components, edge runtime
- `vercel:ai-gateway` — config de fallback de provider
- `frontend-design` — UI distintiva (no caer en el shadcn-default genérico)

---

## Definition of Done

Antes de declarar una tarea como lista:

1. `npm run typecheck` pasa en cero
2. `npm run lint` en cero
3. Existen tests unit/e2e relevantes y pasan
4. RLS probada si la feature toca una tabla tenant-aware
5. Audit log emitido si hay una mutación relevante
6. Rate limit aplicado si la ruta es pública
7. Zod valida todo input externo
8. Sin `console.log` olvidado
9. Env vars nuevas agregadas en `.env.example` + `lib/env.ts`
10. Doc actualizada si cambió el contrato (PRD/spec)
11. **El cambio de schema salió como migration versionada + línea en el MANIFEST** (ver la Doctrina de Migrations) — los clones pueden actualizar
12. **Si tocó UI/flujo de usuario: probado por la pantalla como lo haría un usuario no técnico**, en un entorno fresco estilo VPS, con evidencia visual (ver la Doctrina de QA Visual con Recursos Reales) — curl no cuenta. Cuando el camino pasa por un agente de IA, el caso de aceptación mide el **par** (la pantalla a través del agente + la herramienta llamada directamente, con el mismo texto crudo) y solo cuenta como prueba cuando los dos coinciden — enmienda en [`docs/doctrine/prova-em-par.md`](docs/doctrine/prova-em-par.md) (#489)
13. **Living System Checklist respondido** (ley en `docs/doctrine/sistema-vivo.md`; razonamiento en el manual `docs/doctrine/sistema-vivo/`) — la feature no es una isla: tiene entrada + salida, emite actividad/log, aparece en pantalla, tiene puerta en la navegación, tiene mecanismo anti-muerte, **declara su lazo de retorno** (invariante 7 — qué cambia en el sistema cuando ella falla), y el mapa vivo (`docs/architecture/`) refleja la pieza nueva con ≥2 aristas. Una respuesta que no **nombra el artefacto concreto** (consumidor real, pantalla real, log real) no cuenta
14. **Una pantalla nueva tiene puerta** — declarada en **`lib/navigation/catalogo.ts`** (en el `NAV_CATALOG`, con su grupo), o en la allowlist de `tests/unit/navegacao-completude.test.ts` **con justificación escrita**. Tener pantalla y ser alcanzable son cosas distintas: el CI rechaza una pantalla que existe pero a la que solo se llega escribiendo la URL.

    ⚠️ Esta línea decía `lib/navigation/registry.ts`, y quien la siguiera abría un
    archivo **sin un solo lugar donde declarar**: el `registry.ts` solo deriva
    (`NAV_DESTINATIONS` sale de `NAV_CATALOG`) y reexporta. Es la CARA del
    módulo — es de él de donde importa el test, y por eso el error es fácil. Quien
    declara es el catálogo. Para comprobarlo sin creerle a esta línea:

    ```bash
    grep -c 'href:' lib/navigation/catalogo.ts lib/navigation/registry.ts
    ```
15. **Si tocó Dockerfile, compose o setup kit: el cambio llega a quien ya instaló** (ley en `docs/doctrine/packaging.md`) — ningún servicio de producción quedó solo-`build:`; una variable nueva tiene un default que no rompe un `.env` antiguo; la actualización no pide editar archivos a mano; y, si cambió lo que contiene la imagen, `update.sh` alcanza esa pieza. Corre `pnpm test:shell` — es el único gate que ejercita el kit
16. **Si el PR cambia un comportamiento, busca la afirmación de estado sobre ese comportamiento.** Solo
    sobre lo que cambiaste, y solo en los documentos de autoridad — no salgas a cazar por el repo. La
    documentación afirma cómo *está* el mundo, y una auditoría del 2026-08-14 encontró **227
    afirmaciones desactualizadas de 393 medidas**
    ([`docs/audits/2026-08-14-afirmacoes-de-estado.md`](docs/audits/2026-08-14-afirmacoes-de-estado.md)).
    Donde la afirmación pueda volverse un **comando**, cámbiala en vez de corregirla: un número corregido
    vuelve a envejecer; un `corre esto para saberlo` no envejece nunca

17. **Si el PR cambia un comportamiento visible para quien opera un VPS, trae su fragmento en
    `.changes/`** (ley en [`docs/doctrine/versionamento.md`](docs/doctrine/versionamento.md)).
    El fragmento declara **el efecto en el operador** — `nada_mudou` / `capacidade_nova` /
    `exige_acao` —, nunca el número: el número se calcula a partir del conjunto, y por eso
    dos sesiones paralelas ya no chocan. Compruébalo con `pnpm release:conferir`.
    El CI valida la FORMA de todo fragmento, pero **no** exige la presencia de uno — exigir
    presencia en un check obligatorio rechazaría el PR de Dependabot, el PR de un fork, y el propio PR
    de release, que consume los fragmentos y deja el directorio vacío. La presencia se exige
    aquí, y la exige quien revisa.
    **Toda versión publicada aparece en la página de changelog de la LP** (deskcomm.com.br/changelog,
    pt-BR/en/es). Nadie escribe en el sitio: la LP lee el `CHANGELOG.md` de `main`, y el último paso
    del corte (`release.yml`, job `cortar-tag`) falla cuando la versión no llegó. El texto del
    fragmento es, por lo tanto, una nota pública. Mientras las tres páginas no respondan 200 ese paso
    rechaza TODO corte — la vitrina viene de un PR del `deskcomm-site`, y el `curl` que dice en qué
    estado está abre la sección. Ley: sección "A vitrine" de `versionamento.md`.

18. **Si el PR cambia un comportamiento, declara el destino: núcleo, extensión, ambos o
    infraestructura** (ley en [`docs/doctrine/extensoes.md`](docs/doctrine/extensoes.md)), con la razón
    medida por la pregunta "si ninguna organización activa esto, ¿la operación común sigue entera?".
    "Ambos" trae el consumidor real del punto nuevo del núcleo y la prueba de los dos lados. Clasificar como
    extensión no autoriza quitar ni apagar lo que ya se distribuyó.

¿Un staff engineer lo aprobaría? Si no, itera.
