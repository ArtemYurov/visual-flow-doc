# Tracing through code

The goal is to derive the diagram from what the code actually does, not from what it was meant to do.

---

## Order of work

1. **Find the entry point.** It can be anything:
   - a console command (`app/Console/Commands/`, `artisan`)
   - an HTTP route or controller
   - a queue job (`app/Jobs/`), an event listener
   - an admin panel resource (MoonShine, Filament, Nova)
   - a package extension point — service provider, middleware

2. **Read it in full.** Not in fragments: you need the order of steps, the flags, the early exits.

3. **Follow real connections downward** (see below), recording input and output at every step.

4. **Stop at external service boundaries.**

5. **Assemble the return path:** what goes back up, what gets written, what gets logged.

---

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

## What to write on a node

- **what it is** — the class or method in plain words, not a verbatim identifier
- **where** — `app/Services/Baxi/CatalogDiffer.php:88`
- **input** — the shape of the data: `ParsedProductPage`, `array{sku: string}`, `Collection<Boiler>`
- **output** — what it returns, or the side effect it produces
- when useful — 2–5 lines of the key code, no more

---

## Branching

Every `if`, `match`, early `return` or `throw` that belongs to the main storyline is a `node dec` with labeled branches.

Do not drag argument validation into the diagram. Show the branches that change the fate of the data: took a different source, skipped the write, broke out of the loop, went into the error report.

---

## The second pass in practice

The first pass gives you the trunk. The second pass is where a diagram stops being a summary of the docblocks and starts being a map of the code. Concretely, re-read what you already read, but with these questions.

**Go one level down into every node you drew as a single box.** A call you rendered as "applies the diff" may contain three early exits and a `match`. Those exits are branches of the flow, not implementation detail.

**Read every `catch` and every `Log::warning`.** The interesting question is not "is the error handled" but "what happens to the data". Three outcomes worth drawing separately: the item is dropped, the item is retried, the item is recorded as failed and the run continues.

**Look for failures that do not fail.** An operation that logs a warning, returns early and never reaches the failure list produces a successful exit code with incomplete data. That gap between "something was lost" and "exit code 0" belongs in the document — it is exactly what a reader needs to know and exactly what no docblock says.

**Ask what depends on what was skipped.** If a skipped parent means its children are silently skipped too, that is a cascade. Find it by asking, for each early exit: what did this exit fail to put into the cache / the id map / the queue, and who reads that later?

**Every cache is two paths.** Hit and miss. And a third question: what happens when the refresh fails — stale data, or a hard stop? The answer usually depends on a flag.

**Check the constructor against the body.** A dependency injected but never called in this flow is worth a sentence: it looks like part of the pipeline and is not. It is either a leftover or a deliberate placeholder — the code will tell you which.

**Read fields for arity, not for truth.** A metadata flag that can be `true`, `false` or absent carries three meanings, and the third one ("this stage never touched it") is usually the one that matters operationally.

**Run the flow twice in your head.** What does the second run write? If the answer is "nothing, because nothing changed", find the code that makes that true and say so — idempotency is a property readers rely on and rarely see documented.

---

## Common traps

- **Caching changes the flow.** If there is a cache, the diagram has two branches: hit and miss. A miss goes to the network, a hit does not.
- **Command flags change the flow.** `--force`, `--refresh`, `--dry-run` — either separate branches or an honest mention in the caption.
- **No transaction is a fact of the diagram.** When the sink is not your own database but someone else's HTTP API, partial writes are possible; that matters more than half the remaining detail.
- **Idempotency.** Note what happens on a second run: overwrite, skip, duplicates.
- **Stale comments and docblocks.** A docblock cannot be trusted when the code says otherwise.

---

## When the source is discussion, not code

A planned algorithm is drawn from the conversation and the plan, but the starting point is still the code: what exists today.

Then the diagram carries two layers — current and future — distinguished by classes (`now`/`fut`, `add`/`dead`), with a mandatory `legend`. That way the document stays useful both during the work and after it.

Never pass the planned off as the existing. If something is not there yet, it must be obvious at a glance.

---

## Check before handing over

- [ ] input and output are named explicitly
- [ ] every significant node has a path and a line
- [ ] no connection "by intuition" — all derived from code
- [ ] external services are terminal
- [ ] every decision branch has a label
- [ ] disagreements with plans and docblocks are either resolved or mentioned
- [ ] whatever could not be traced is said out loud

And from the second pass:

- [ ] every `catch` and early exit that changes the data's fate is a branch
- [ ] every cache appears as hit **and** miss, plus the refresh-failure path
- [ ] any failure that leaves a successful exit code is called out
- [ ] cascades are traced: what else disappears when this step is skipped
- [ ] injected-but-unused dependencies are named as such
- [ ] the second run is described: overwrite, skip, or no request at all
