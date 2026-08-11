# Tracing: PHP / Laravel

Applies to projects with a `composer.json`. Verified against four real runs on a Laravel 13 codebase.

## What counts as a connection in PHP/Laravel

In PHP connections are not visible from `import` statements the way they are in TypeScript. Look for:

| connection | how to find it |
|---|---|
| constructor injection | `__construct(private XService $x)` — the primary source of the graph |
| method call | `$this->x->method(` |
| facade | `Cache::`, `Log::`, `DB::`, `Http::` |
| queue | `dispatch(`, `SomeJob::dispatch(`, `->onQueue(` |
| event | `event(`, `->dispatch(`, listeners in `EventServiceProvider` |
| outbound HTTP | Saloon connectors and requests, `Http::get(`, Guzzle |
| database | Eloquent models, `DB::table(`, repositories |
| config | `config('x.y')` — often hides the real address of an external service |
| container | `app(`, `resolve(`, bindings in providers — **a hidden connection, check the providers** |

Dangerous spot: connections through the container and through class-name strings do not show up in an import graph. If a class looks unconnected, grep for its name across the project.

---

---

## Entry points to look for

- a console command (`app/Console/Commands/`, registered in a service provider or auto-discovered)
- an HTTP route or controller (`routes/*.php`)
- a queue job (`app/Jobs/`), an event listener (`EventServiceProvider`)
- an admin panel resource — MoonShine, Filament, Nova
- a package extension point — service provider, middleware, macro

---

## What the import graph misses

PHP hides more connections than most ecosystems, because the import list is not the dependency list:

- **the container** — `app()`, `resolve()`, contextual bindings in providers;
- **facades** — a static call that is really a container lookup;
- **string class names** — in config files, in `dispatch()`, in morph maps;
- **auto-discovery** — packages registering providers through `composer.json` `extra.laravel`;
- **model events and observers** — registered in a provider, fired implicitly on save.

If a class looks unconnected, grep its bare name across `config/`, `routes/` and providers before calling it dead.
