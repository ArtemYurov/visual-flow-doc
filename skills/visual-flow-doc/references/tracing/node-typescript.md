# Tracing: Node / TypeScript

Applies to projects with a `package.json`. **Not yet verified against a real run** — unlike the PHP guide, treat this as a starting point and correct it from what the code actually shows.

---

## What counts as a connection

| connection | how to find it |
|---|---|
| import + call | `import { x } from './y'` followed by a call — the one case where the import graph is the dependency graph |
| default export of a factory | `export default function createThing()` — the connection is the call site, not the import |
| dependency injection | NestJS `@Injectable()` + constructor params, InversifyJS `@inject()`, Angular providers — resolved by **type or token**, not by import |
| queue / job | BullMQ `queue.add(`, `new Worker(`, Agenda, `pg-boss` — the handler is often in another file, matched by a **string job name** |
| event | `EventEmitter.on(`/`emit(`, Nest `@OnEvent()`, message brokers — matched by string |
| outbound HTTP | `fetch(`, `axios.`, `got(`, generated SDK clients |
| database | Prisma `prisma.model.`, Drizzle, TypeORM repositories, Knex, raw `pool.query(` |
| config | `process.env.X`, `zod`-parsed config objects — often hides the address of an external service |
| middleware | Express `app.use(`, Nest interceptors and guards, Next.js `middleware.ts` — runs on every request without appearing in any call chain |

---

## Entry points to look for

- `bin` and `scripts` in `package.json` — the actual commands people run
- an HTTP server: `app.listen(`, Nest `main.ts`, Fastify plugin registration
- **filesystem routing** — Next.js `app/**/page.tsx`, `route.ts`, `middleware.ts`; SvelteKit `+page.server.ts`; Remix `routes/`. Nothing imports these: the framework finds them by path
- serverless handlers — `exports.handler`, Vercel/Netlify function files
- queue workers and cron entries, often a separate process from the API
- `postinstall` and build hooks

---

## What the import graph misses

The import graph is closer to the truth than in PHP, but four things escape it:

- **filesystem routing** — a route file is an entry point with no importer at all;
- **barrel files** (`index.ts` re-exporting a directory) — the import points at the barrel, the implementation is elsewhere, and a symbol can travel through several of them;
- **decorators and metadata** — Nest and TypeORM resolve by token and by reflected type; the constructor parameter names a class that may never be imported at the call site;
- **dynamic loading** — `await import()`, plugin directories scanned at boot, `require` of a path built from config.

Two more that bite specifically when reading TypeScript:

- **`import type` disappears at runtime.** A type-only import is not a runtime connection. If the whole relationship is `import type`, the modules do not talk to each other at all — do not draw an arrow.
- **The published shape may differ from the source.** Check `exports` in `package.json`: what other packages consume is that map, not the directory layout.

---

## Traps when tracing

- **Async without await.** A promise created and not awaited is a fire-and-forget branch: the flow continues before the work finishes, and a failure there lands in an unhandled rejection rather than the caller's `catch`.
- **`try/catch` does not cover callbacks.** An error thrown inside a callback passed to something else escapes the surrounding `try`.
- **Middleware order is the flow.** In Express and Nest, the sequence of `use()`/`@UseGuards()` decides what runs before the handler; that order is part of the algorithm and rarely documented.
- **A worker is a separate process.** Drawing the enqueue and the handler as one continuous line hides the fact that they fail, retry and deploy independently.
- **Environment branches.** `if (process.env.NODE_ENV === 'production')` around a real behavioural difference is a branch, not configuration noise.
