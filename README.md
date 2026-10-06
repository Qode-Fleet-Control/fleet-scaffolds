# fleet-scaffolds

Every Fleet Control starter in one place: the base harness (`fleet-template-v1`) and all
97 scaffold repos, as submodules of this one repo. Mount it once — Fleet Control's own repo
has it at `scaffolds/` — instead of a hundred separate submodules.

```sh
git clone --recurse-submodules git@github.com:Qode-Fleet-Control/fleet-scaffolds.git
# or, inside a clone: git submodule update --init --recursive
```

## Layout

```
fleet-template-v1/            the language-agnostic run contract every repo is cut from
<language>/<slug>/            one scaffold per framework, e.g. typescript/nextjs, php/laravel
typescript/fleet-scaffold/    the fleet's own full-stack TypeScript starter (fleet-scaffold-v1)
typescript/nextjs-admin/      the fleet's Next.js Admin dashboard (qode-nextjs-admin-template-v1)
github-top-languages-frameworks.json   the catalogue (the backend's copy is the one served)
```

**Pointers stay current by themselves.** `.github/workflows/sync.yml` moves every submodule
to its repo's `main` tip — hourly, on demand (Actions ▸ Sync submodule pointers ▸ Run), and
on a `repository_dispatch` of type `template-updated`. Change a template in its OWN repo
(PR → green Build check → merge); this repo follows. Do not commit inside a submodule here.

## About the scaffolds

Starter projects for the **top 10 languages on GitHub** (Octoverse 2025) × the
**top 10 frameworks** of each: 95 of the 100, plus two of the fleet's own TypeScript
starters: the full-stack `fleet-scaffold-v1` and the Next.js Admin dashboard
(`qode-nextjs-admin-template-v1`, added 2026-10-06; it was the Template Library's one
template until the library was retired). The five that cannot run in a Linux
container have none: Unity, .NET MAUI, WPF, Godot (C#) and Unreal Engine (C++).
The ranking is in `github-top-languages-frameworks.json`. The first 30
(TypeScript, Python, Go) were made 2026-09-21; the other 65 on 2026-10-05.

Each one also lives in its own repo, provisioned from `fleet-template-v1` and
wired for the fleet. **The repos are the source of truth** — they carry the
`fleet.conf` manifest, the `bin/` lifecycle scripts, CI and `.env.example` that
this directory does not. This tree holds only the application code.

## The repos

`Qode-Fleet-Control/qode-<framework>-template-v1`, public, one per framework (moved from
Qode-Platform on 2026-10-05; the old URLs redirect):

| TypeScript | Python | Go |
|---|---|---|
| react, nextjs, angular, vue, sveltekit | django, fastapi, flask, langchain, scrapy | gin, echo, fiber, chi, buffalo |
| astro, react-router, trpc, nuxt | pytorch, tensorflow-keras, pandas, scikit-learn, pytest | grpc-go, cobra, gorm, controller-runtime, testify |

| JavaScript | Java | C# | PHP |
|---|---|---|---|
| react-js, express, vue-js, nextjs-js, svelte | spring-boot, spring-framework, quarkus, micronaut, hibernate-orm | aspnetcore, efcore, blazor | laravel, symfony, wordpress, codeigniter, slim |
| electron, jquery, jest, fastify, threejs | jakarta-ee, junit5, apache-spark, netty, vertx | avalonia, xunit, dapper | yii, laminas, cakephp, phpunit, drupal |

| Shell | C++ | HCL |
|---|---|---|
| oh-my-zsh, bats-core, starship, oh-my-bash, prezto | qt, boost, opencv, grpc, googletest | terraform, opentofu, terragrunt, terraform-aws-modules, packer |
| homebrew, shellspec, bash-it, shunit2, fisher | eigen, drogon, poco, catch2 | nomad, consul, vault, terratest, tflint |

The JavaScript React / Vue / Next.js repos are `qode-<fw>-js-template-v1`, so their
names do not collide with the TypeScript ones.

**Two kinds.** An `http` template serves on `$PORT` (fleet.conf `DOCKER_START_CMD`
runs `docker compose up`). A `job` template (test suites, libraries, IaC, shell
setups) has no server: `DOCKER_START_CMD` is empty and `docker compose run --rm app`
runs the job.

**Build check.** Every repo runs `.github/workflows/build.yml` on each push and
pull request, with no secrets: an `http` template through `bin/run` to 200 at
`HEALTH_PATH`, `bin/restart`, and `bin/stop` leaving no container; a `job` template
via `docker compose run --rm app`. A scaffold reached `main` only once that was
green. CI (manual) pushes the image to Artifact Registry as for any workspace.

NestJS is **not** in this set: `qode-nestjs-template-v1` already existed with
real work in it, and was fixed in place instead (PR #15).

## Verified

All 13 HTTP templates serve at the ROOT of their own hostname — the fleet gives
every app `https://<hash>.<fleet app domain>/`, so there is no path prefix — and
answer **200** at `$HEALTH_PATH` on the fleet's injected `$PORT`.

The other 6 (scrapy, pytorch, tensorflow-keras, pandas, scikit-learn, pytest)
have no HTTP server, so `START_CMD` is empty by design: `bin/run` installs and
then stops at the start step. Verified on pytest — installs, `7 passed`.

Not verified by booting: the **nestjs** repo (its `AppModule` opens a TypeORM
connection and needs a real `DATABASE_URL`).

## Go

All ten are hand-written (Go has no equivalent of `create-next-app`) and, unlike
the JS side, were **built and tested locally before shipping** — Go 1.23.4 is
installed in the dev box, so `go build ./...` and `go test ./...` pass for every
module. gin/echo/fiber/chi/buffalo serve their routes at `/`; the other five
are not HTTP services and say so.

Two Go-specific notes: `go mod tidy` resolves these to **Go 1.25** (gin 1.12
alone requires it) and controller-runtime to **1.26**, so the images use
`golang:1.25-alpine` / `1.26-alpine`, not the pack's 1.23. GORM uses
`glebarez/sqlite` (pure Go) because the image builds `CGO_ENABLED=0` into
distroless, where cgo-based `mattn/go-sqlite3` cannot link.

## Things that will bite you

**Bind `0.0.0.0`.** nginx proxies to the container name, so an app listening on
127.0.0.1 is unreachable through its hostname.

**`env VAR=…`, not bare `VAR=…`,** in `START_CMD`: the start step is exec'd and
exec cannot take a leading assignment.

## Deviations from stock generator output

- **angular** — `serve` devDependency (Angular ships no production static
  server). CLI pinned to 20: 21+ needs Node ≥22.22.3.
- **sveltekit** — `adapter-auto` → `adapter-node`; adapter-auto fails the build
  outside a recognised host.
- **django** — `requirements.txt` added; `ALLOWED_HOSTS` reads
  `$DJANGO_ALLOWED_HOSTS` (the stock empty list 400s behind the ingress).
- **flask / scrapy / langchain** — `requirements.txt` added (no generator emits one).
- **langchain** — `httpx<0.28` pinned (langserve imports the removed
  `VerifyTypes`) and the CLI's `add_routes(app, NotImplemented)` replaced with a
  placeholder echo chain, which otherwise raises before the server listens.

## Origin

7 of the 20 have no official generator (FastAPI, Flask, PyTorch,
TensorFlow/Keras, pandas, scikit-learn, pytest) and are hand-written to the
layout each project's own docs teach. The other 13 are verbatim CLI output;
each repo's README names the exact command.

Generated 2026-09-21 on Node v22.12.0 / Python 3.12.3.
