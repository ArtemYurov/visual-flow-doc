# Tracing: Node / TypeScript

Applies to projects with a `package.json`. Verified once, against a Nuxt/Nitro subsystem. The sections on decorators/DI, queues, barrel files, dynamic `import()` and ORMs have **not** been exercised yet — treat those as a starting point.

---

## What counts as a connection

| connection | how to find it |
|---|---|
| import + call | `import { x } from './y'` followed by a call |
| **auto-import** | in convention-driven frameworks a symbol is global with **no import at all** — see below |
| default export of a factory | `export default function createThing()` — the connection is the call site |
| dependency injection | NestJS `@Injectable()` + constructor params, InversifyJS `@inject()`, Angular providers — resolved by **type or token**, not by import |
| queue / job | BullMQ `queue.add(`, `new Worker(`, Agenda, `pg-boss` — the handler is usually elsewhere, matched by a **string job name** |
| event | `EventEmitter.on(`/`emit(`, Nest `@OnEvent()`, brokers — matched by string |
| outbound HTTP | `fetch(`, `axios.`, `got(`, `$fetch(`, generated SDK clients |
| database | Prisma `prisma.model.`, Drizzle, TypeORM, Knex, raw `pool.query(` |
| **framework KV by mount name** | `useStorage("redis")`, Nitro/Nuxt cache handlers, Next.js cache — a **service locator keyed by a string** resolved against the framework config, not against any import |
| config | `process.env.X`, `zod`-parsed config objects — often hides the address of an external service |
| middleware | Express `app.use(`, Nest interceptors and guards, Nuxt/Nitro `server/middleware/` — runs on every request without appearing in any call chain |

---

## Entry points to look for

- `bin` and `scripts` in `package.json` — the commands people actually run
- an HTTP server: `app.listen(`, Nest `main.ts`, Fastify plugin registration
- **filesystem routing** — nothing imports these, the framework finds them by path:
  - Next.js `app/**/page.tsx`, `route.ts`, `middleware.ts`
  - SvelteKit `+page.server.ts`; Remix `routes/`
  - **Nuxt / Nitro** — `server/middleware/`, `server/plugins/`, `server/api/`, `server/routes/`, plus `modules` in `nuxt.config.ts`. Note that `app/middleware/` is the **client** layer with a different lifecycle: do not mix the two in one diagram
- serverless handlers — `exports.handler`, Vercel/Netlify function files
- queue workers and cron entries, often a separate process from the API
- `postinstall` and build hooks

---

## What the import graph misses

An import graph is closer to the dependency graph here than in PHP — but in convention-driven frameworks that closeness is an illusion.

**Auto-imports are the biggest gap.** In Nuxt/Nitro every export from `server/utils/` becomes a global, as does the whole of h3 (`defineEventHandler`, `getRequestURL`, `sendRedirect`, `getCookie`), plus `useStorage`, `defineNitroPlugin`, `$fetch`. A file can use six symbols and import none of them. Three consequences:

- **"nothing imports it" ≠ dead code.** Count *calls*, not imports, before declaring anything unused.
- **Read the generated map.** `.nuxt/types/nitro-imports.d.ts` (and its equivalents) lists exactly what became global — no guessing needed.
- **A file can be inconsistently explicit** — importing one helper while relying on auto-import for five others. The presence of imports says nothing about the absence of other connections.

Also invisible:

- **filesystem routing** — a route file is an entry point with no importer at all;
- **barrel files** (`index.ts` re-exporting a directory) — the import points at the barrel, and a symbol can travel through several;
- **decorators and metadata** — Nest and TypeORM resolve by token and reflected type;
- **dynamic loading** — `await import()`, plugin directories scanned at boot, `require` of a config-built path.

Two that bite specifically in TypeScript:

- **`import type` disappears at runtime.** If the whole relationship is `import type`, the modules do not talk — do not draw an arrow.
- **The published shape may differ from the source.** `exports` in `package.json` is what consumers see, not the directory layout.

---

## Read the framework, not just the app

**In this ecosystem the lifecycle contract is usually not written down in the application — and often not written down anywhere.** When the question is "does this run before that", "is this awaited", "what order do these execute in", the answer lives in `node_modules`, in the framework's own runtime source. Reading it is normal work, not a last resort.

Facts that only exist there: whether plugins are awaited, in what order files are scanned, whether a middleware chain short-circuits, what key pattern a storage driver scans, which URL property a request helper actually reads.

---

## Traps when tracing

- **Async without await.** A promise created and not awaited is a fire-and-forget branch: the flow continues before the work finishes. Watch for `array.forEach(async …)` — `forEach` never waits for its callbacks, so every `await` inside is decorative.
- **`try/catch` does not cover callbacks** — nor an async function invoked without `await`. A `try { plugin(app) } catch` around an async plugin catches nothing; the failure lands in `unhandledRejection`.
- **Order comes from filenames.** With explicit `use()` the order is visible. In convention-driven frameworks it comes from sorting **file names** — renaming a file silently changes the algorithm, and nothing in the code records the dependency.
- **A chain can short-circuit.** Once a handler marks the response as sent, the remaining middleware never runs — so an earlier redirect cancels a later one entirely.
- **A worker is a separate process.** Drawing enqueue and handler as one line hides that they fail, retry and deploy independently.
- **Environment branches.** `if (process.env.NODE_ENV === 'production')` around real behaviour is a branch, not noise.
- **`const X = process.env.X` at module level** freezes the value at import time. When a neighbouring module reads the same variable lazily, the two disagree — and both patterns often live in one repository.
- **Logger configuration is a property of the flow.** A transport with `level: "error"` silently discards every `info` and `warn`. Check the configured transport levels against the levels actually called: a subsystem can be logging its success messages into nowhere, leaving no positive confirmation that it ever worked.
