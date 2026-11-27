---
permalink: /projects/contest-maker-150/
title: "Contest Maker 150: a Go API that builds timed practice contests from the NeetCode 150"
description: "A Go and Postgres service that picks unsolved problems by difficulty, runs a timed contest per user, and ships with tracing and metrics in Docker Compose."
type: project
order: 4
stack: "Go, Gin, GORM, PostgreSQL, OpenTelemetry, React, Docker"
tags: [go, gin, postgresql, gorm, opentelemetry, docker]
repo: https://github.com/Raghavendrapara/contest-maker-150
abstract: >
  Practising from a fixed list of 150 problems gets stale when you keep landing on ones you have already solved.
  I built a Go (Gin + GORM) API that generates timed contests from only the problems a user has not solved, ordered
  Easy to Medium to Hard, with JWT auth, Postgres persistence, OpenTelemetry tracing and Prometheus metrics, all run
  with Docker Compose. It shows layered Go service design, a small fan-out concurrency pattern and a container setup
  with an observability stack.
---

## The problem

I was working through the NeetCode 150 list and wanted something closer to a contest than a checklist: pick N problems I
have not solved yet, put them in a sensible difficulty order, start a clock, and record what I finished so the next
contest does not hand me the same problems again.

That is a small domain (users, problems, contests, solved records), which made it a good excuse to build a Go backend
the way I would build a Java service at work: clear layers, typed domain errors, auth, a real database, graceful
shutdown, and traces and metrics from day one. The React frontend is a thin client over the API.

## Architecture

<div class="diagram">
<svg viewBox="0 0 740 380" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Browser calls an nginx container serving the React build, which proxies /api to the Go API. The API reads and writes PostgreSQL, exports traces to Jaeger, and exposes /metrics which Prometheus scrapes; Grafana reads Prometheus.">
  <defs><marker id="contest-maker-150-a1" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dg-arrowhead"/></marker><marker id="contest-maker-150-a2" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" class="dg-arrowhead-accent"/></marker></defs>
  <rect x="165" y="24" width="565" height="340" rx="10" class="dg-group"/>
  <text x="182" y="46" class="dg-label">docker compose network</text>
  <rect x="20" y="160" width="120" height="56" rx="8" class="dg-box"/>
  <text x="80" y="184" text-anchor="middle" class="dg-text">Browser</text>
  <text x="80" y="202" text-anchor="middle" class="dg-sub">React SPA</text>
  <rect x="190" y="160" width="150" height="56" rx="8" class="dg-box"/>
  <text x="265" y="184" text-anchor="middle" class="dg-text">frontend</text>
  <text x="265" y="202" text-anchor="middle" class="dg-sub">nginx, proxies /api</text>
  <rect x="380" y="130" width="175" height="116" rx="8" class="dg-box-accent"/>
  <text x="467" y="160" text-anchor="middle" class="dg-text">Go API (Gin)</text>
  <text x="467" y="182" text-anchor="middle" class="dg-sub">middleware → handler</text>
  <text x="467" y="200" text-anchor="middle" class="dg-sub">→ service → repository</text>
  <text x="467" y="222" text-anchor="middle" class="dg-sub">JWT, zap, OTel spans</text>
  <rect x="595" y="160" width="120" height="56" rx="8" class="dg-box"/>
  <text x="655" y="184" text-anchor="middle" class="dg-text">PostgreSQL</text>
  <text x="655" y="202" text-anchor="middle" class="dg-sub">GORM AutoMigrate</text>
  <rect x="595" y="62" width="120" height="50" rx="8" class="dg-box"/>
  <text x="655" y="83" text-anchor="middle" class="dg-text">Jaeger</text>
  <text x="655" y="100" text-anchor="middle" class="dg-sub">OTLP over HTTP</text>
  <rect x="380" y="292" width="175" height="50" rx="8" class="dg-box"/>
  <text x="467" y="313" text-anchor="middle" class="dg-text">Prometheus</text>
  <text x="467" y="330" text-anchor="middle" class="dg-sub">scrapes /metrics</text>
  <rect x="190" y="292" width="150" height="50" rx="8" class="dg-box"/>
  <text x="265" y="313" text-anchor="middle" class="dg-text">Grafana</text>
  <text x="265" y="330" text-anchor="middle" class="dg-sub">datasources only</text>
  <line x1="140" y1="188" x2="188" y2="188" class="dg-line-accent" marker-end="url(#contest-maker-150-a2)"/>
  <line x1="340" y1="188" x2="378" y2="188" class="dg-line-accent" marker-end="url(#contest-maker-150-a2)"/>
  <line x1="555" y1="188" x2="593" y2="188" class="dg-line-accent" marker-end="url(#contest-maker-150-a2)"/>
  <line x1="555" y1="145" x2="593" y2="100" class="dg-line-dashed" marker-end="url(#contest-maker-150-a1)"/>
  <line x1="467" y1="292" x2="467" y2="248" class="dg-line-dashed" marker-end="url(#contest-maker-150-a1)"/>
  <line x1="340" y1="317" x2="378" y2="317" class="dg-line" marker-end="url(#contest-maker-150-a1)"/>
</svg>
<p class="diagram-caption">Figure 1. The six Compose services. The solid path is a user request; dashed lines are trace export (see the config note under "What I'd do next") and metric scraping.</p>
</div>

The backend follows a standard Go layout: `cmd/api/main.go` wires everything, and `internal/` holds `domain`,
`handler`, `service`, `repository`, `middleware`, `infrastructure` and `data`.

- **domain** holds the GORM models (`User`, `Problem`, `Contest`, `ContestProblem`, `Submission`), the repository
  interfaces, request/response DTOs and a set of sentinel errors such as `ErrActiveContestExists` and
  `ErrContestExpired`.
- **repository** implements those interfaces with GORM against Postgres.
- **service** holds the business rules: one active contest per user, problem selection, ownership checks, and
  password hashing and JWT issuing.
- **handler** binds and validates JSON with Gin's `binding` tags and maps domain errors to HTTP status codes (409 for
  an existing active contest, 403 for someone else's contest, 400 for an expired one).
- **middleware** runs on every request in this order: panic recovery, request logging with a request ID, CORS, an
  OpenTelemetry server span, and HTTP metrics. A JWT middleware guards the `/users` and `/contests` groups.

On startup the API loads config from environment variables, connects to Postgres with a sized connection pool, runs
`AutoMigrate`, and seeds the 150 problems from a JSON file compiled into the binary with `//go:embed`. The seeder skips
the insert if the table already has rows. The HTTP server starts in a goroutine, and `SIGINT`/`SIGTERM` trigger
`server.Shutdown` with a 30-second deadline, followed by a flush of the trace provider.

The React app (Vite, TypeScript, Zustand, React Query) has pages for signup, login, a dashboard, contest creation, the
active contest with a countdown timer, and history. Its Axios client queues requests during a token refresh.

## Key design decisions

**A contest is a row with a start time and a duration, not a running timer.** `Contest` stores `StartedAt` and
`DurationMinutes`; `IsExpired()` and the `time_remaining_seconds` in each response are computed from those. There is no
background job. An expired contest is flipped to `completed` lazily, the next time it is read or the user tries to
create a new one. The alternative was a scheduler or a goroutine per contest. That is more moving parts and does not
survive a restart. The cost is that the stored status can be stale until someone touches the contest.

**"Unsolved" is decided in SQL, not in Go.** Solved problems live in a `submissions` table, and the repository excludes
them with a `NOT IN` subquery. The alternative was loading the user's solved IDs and filtering in memory. With 150
problems either works, but the subquery keeps the rule in one place and still works if the table grows.

**Difficulty mix by contest size, with spill-over.** `calculateDistribution` picks the Easy/Medium/Hard split from the
requested count (for example 30/40/30 for 6-10 problems, 25/50/25 above that) and forces at least one of each once the
count is 3 or more. If a bucket runs short, the shortfall rolls over into the next, harder bucket. The final list is
sorted by difficulty weight so the contest ramps up. A plain random sample would have been simpler, but it would not
give the Easy-to-Hard ordering I wanted.

**Fan-out queries per difficulty.** Problem selection and the progress endpoint each run three queries, one per
difficulty, in goroutines and collect the results over a buffered channel. That is a deliberate exercise of the Go
fan-out/fan-in pattern. Against a local Postgres a single `GROUP BY` query would do the same job, and I did not measure
a difference either way.

**Small distroless image for the API.** The backend Dockerfile is a two-stage build: `golang:1.25-alpine` compiles a
static binary (`CGO_ENABLED=0`, `-ldflags="-w -s"`), and the runtime image is
`gcr.io/distroless/static-debian12:nonroot` with only the binary, CA certificates and zoneinfo. The trade-off is that
there is no shell or `wget` in the image, which matters for the Compose health check (see below).

## Inside the code

The selection step fans out one repository call per difficulty and fans the results back in. The buffered channel
sized to the number of workers means no goroutine blocks on send, and a separate goroutine closes the channel once the
`WaitGroup` drains:

```go
resultChan := make(chan difficultyResult, len(difficulties))
var wg sync.WaitGroup
fetchProblems := func(diff domain.Difficulty) {
	defer wg.Done()
	problems, err := s.problemRepo.
		FindUnsolvedByUserAndDifficulty(userID, diff)
	resultChan <- difficultyResult{
		difficulty: diff, problems: problems, err: err,
	}
}
for _, diff := range difficulties {
	wg.Add(1)
	go fetchProblems(diff)
}
go func() {
	wg.Wait()
	close(resultChan)
}()
// … collect into problemsByDifficulty, then apply the distribution
```

The shared `*rand.Rand` used for the partial Fisher-Yates shuffle is guarded by a `sync.Mutex`, because
`math/rand.Rand` is not safe for concurrent use and the service is a singleton shared by all request goroutines.

The "never repeat a solved problem" rule is one subquery in the repository:

```go
solvedSubquery := r.db.Model(&domain.Submission{}).
	Select("problem_id").
	Where("user_id = ?", userID)

result := r.db.Where("id NOT IN (?)", solvedSubquery).
	Where("difficulty = ?", difficulty).
	Order("RANDOM()").
	Find(&problems)
```

Marking a problem complete inside an active, unexpired contest writes a `Submission` row (once per user and problem),
and that row is what this subquery reads the next time a contest is generated.

## Running it

From the repository root:

```bash
cp .env.example .env          # set POSTGRES_PASSWORD and JWT_SECRET
docker-compose up -d
# frontend http://localhost:3000, API http://localhost:8080
# Jaeger :16686, Prometheus :9090, Grafana :3001
```

For local development the README runs the API with `go run cmd/api/main.go` (it expects a Postgres on localhost) and
the frontend with `npm install && npm run dev`.

## What I'd do next

- **Tests.** There are no Go or frontend tests yet. The domain logic (`calculateDistribution`, `IsExpired`, the
  shortfall carry-over) is pure and easy to table-test. The services already depend on repository interfaces, so
  in-memory fakes would cover the contest rules.
- **Fix the env-var mismatch in Compose.** `docker-compose.yml` sets `SERVER_ENVIRONMENT`, `TELEMETRY_OTEL_ENDPOINT`
  and `JWT_ACCESS_EXPIRY`, but `config.go` reads `ENVIRONMENT`, `OTEL_EXPORTER_OTLP_ENDPOINT` and
  `JWT_ACCESS_EXPIRY_MINUTES`. Inside Compose the API falls back to its defaults, so the trace exporter points at an
  `otel-collector` host that does not exist instead of Jaeger.
- **Health check in a distroless image.** The Compose health check calls `wget`, which the distroless runtime does not
  have. A `-healthcheck` flag on the binary, or a tiny probe binary, would fix it.
- **Propagate context into the database.** Every repository has a `WithContext` method, but services never call it, so
  GORM queries are not tied to request cancellation or to the trace. Adding the `otelgorm` plugin would give DB spans.
- **Make "one active contest" a database guarantee.** The check is read-then-insert in the service, so two concurrent
  requests can both pass it. A partial unique index on `contests(user_id) WHERE status = 'active'` would close that.
  Contest creation and adding its problems also run as two writes with a compensating delete instead of one
  transaction.
- **Finish the half-built pieces.** `GET /users/me/progress` returns per-difficulty counts, but `topic_progress` and
  `contest_stats` are never filled in. Several metric instruments (`contests.active`, `problems.solved`,
  `db.query.duration`) are created but never recorded. Un-marking a problem does not remove its submission. Grafana has
  datasources but no dashboards.
- **Versioned migrations.** `AutoMigrate` is fine for a single developer; golang-migrate or Atlas would make schema
  changes reviewable.

## Skills this shows

- **Go service structure:** `cmd/` plus `internal/` layering, interfaces in the domain package, constructor injection
  wired in `main.go`, and sentinel errors mapped to HTTP codes in handlers.
- **API design:** REST resources with sub-resource actions (`PATCH /contests/:id/problems/:problemId`,
  `POST /contests/:id/complete`), request validation via binding tags, and ownership checks on every contest route.
- **Concurrency:** fan-out/fan-in with `WaitGroup` and buffered channels, and a mutex around a non-thread-safe RNG.
- **Persistence:** a Postgres model with a composite-key join table, a `text[]` column for topics, subqueries for
  exclusion, connection-pool settings and an idempotent seed from an embedded file.
- **Auth:** bcrypt password hashing, HS256 access and refresh tokens with a `type` claim so a refresh token cannot be
  used as an access token, and a check on the signing method.
- **Operations:** graceful shutdown, `/health` with a DB ping, structured zap logs with request IDs, OpenTelemetry
  server spans with a 10% parent-based sampler, a Prometheus `/metrics` endpoint, and a multi-stage distroless build in
  a six-service Compose stack.
