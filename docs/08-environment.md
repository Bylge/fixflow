# 08 — Environment, CI and Deployment

Where the code runs, how it gets tested, how it ships, and what happens when a ship goes
wrong.

## Local — WSL2 + Docker Compose

Decided. The pilot host is Linux; the closer local sits to it, the fewer hours go to bugs
that only exist on one machine. `03-architecture.md` already refuses SQLite for the same
reason.

**The repository lives inside the WSL2 filesystem** — `~/code/fixflow`, not `/mnt/c/...`.
Cross-filesystem access is slow enough to be felt on every Composer install and every test
run, and file watching for Vite is unreliable across the boundary. This one has a cost
today: the docs currently sit on the Windows side and move at M1.

**Git starts now, not at M1.** The repository — `github.com/Bylge/fixflow` — is created on
the Windows side while the docs still live there, so planning changes are versioned and
revertible from the first commit. History travels with the folder, so the M1 move into WSL
costs nothing.

**Compose runs backing services only** — `postgres`, `redis`, `mailpit`. PHP, Artisan, the
queue worker and Vite run natively in WSL2. Containerising the app locally buys parity we
already get from the OS and costs a rebuild on every change; the deployable image, if we
end up building one, is a separate concern (`05-open-questions.md`).

**Wildcard subdomains.** `03-architecture.md` resolves tenants by host, so local needs
`*.fixflow.test`. `/etc/hosts` has no wildcards, and dnsmasq is more machinery than three
hostnames justify. Enumerate them:

```
127.0.0.1  fixflow.test  acme.fixflow.test  beta.fixflow.test  admin.fixflow.test
```

Two tenants is exactly what the isolation tests need, and a fourth entry costs one line.

**Required:** PHP 8.3+ with `pdo_pgsql`, `intl` (locale formatting), `fileinfo` (attachment
MIME validation), plus the Laravel baseline. Composer, Node LTS, Docker Desktop with the
WSL2 backend.

**Seeders build a multi-tenant world, not a single tenant.** Two tenants with overlapping
data, one user per default role in each, one user who is a member of both, and tickets
carrying internal notes. If local looks single-tenant, isolation bugs stay invisible until
production — which is the one place they must never appear.

## CI — GitHub Actions

Runs on every pull request and on push to main. Four jobs, parallel:

| Job | Does |
|---|---|
| `lint` | `pint --test` |
| `static` | Larastan, level max |
| `test` | Pest, against a PostgreSQL service container on the production major version |
| `i18n` | `en`/`pl` key parity and the literal-string lint (`07-conventions.md`) |

No version matrix. One PHP version — the one production runs. Testing combinations we will
never deploy is work that buys nothing.

**Main is branch-protected: all four jobs green, or no merge.** That is the entire review
process, so it does not get bypassed.

## CD

Deploy triggers on push to main, after CI passes. Never from a pull request.

1. Build / pull the release
2. `php artisan migrate --force`
3. Cache config, routes, views
4. **Restart the queue worker** — a running worker holds the old code in memory, and a
   worker silently running last week's Action is a genuinely nasty bug to chase
5. Hit the health route (`/up`)
6. Health check fails → the deploy fails loudly and visibly. A deploy that half-succeeds
   quietly is worse than one that stops

`php artisan down` only for migrations that actually require it. Most will not.

Secrets live in GitHub Actions secrets and in `.env` on the server. `.env` is never in git;
`.env.example` always is, and `07-conventions.md` makes updating it part of done.

## Rollback

The part that needs deciding before the pilot has data rather than after.

**Code rolls back.** Redeploy the previous commit. Always available, fast, boring.

**Schema does not.** Migrations are forward-only once the pilot is live (`CLAUDE.md`), so a
migration that turns out wrong is corrected by another migration, never by `migrate:rollback`
against real data.

Two rules follow, and they are the whole point of this section:

- **Destructive changes are split across two deploys.** Release N stops writing the column;
  release N+1 drops it. Never the same release. This is what makes "code rolls back"
  actually true — a rollback into a schema that no longer has the column is not a rollback
- **A verified backup is taken immediately before any migration that touches existing pilot
  data.** Not the nightly one. The one from four minutes ago

## Environments

Local (WSL2), CI (ephemeral), production (the pilot). No staging until there is something
to stage — `03-architecture.md`.

## Health and observability

`/up` serves the deploy check and, from M10, an uptime monitor. Error tracking is chosen
before real users arrive and is tracked as an open item, not assumed.
