# openpost API test file reference

Files are named `*.api.yaml`, `*.api.yml` or `*.api.json`. `openpost test` with
no paths runs `./api-tests` (also `apitests/`, `tests/api/`, `.openpost/tests/`),
or every matching file below the current directory. Get the JSON Schema with
`openpost schema` (editors: add
`# yaml-language-server: $schema=https://unpkg.com/openpost/schema/api-test.schema.json`
as the first line).

## File structure

```yaml
name: Orders API                 # required
description: |                   # Markdown, used by `openpost docs`
  Order lifecycle.
version: 2.1.0
baseUrl: "{{baseUrl}}"           # prepended to relative request URLs
vars:                            # default variables
  baseUrl: http://localhost:3000
  apiKey: "{{$env.ORDERS_API_KEY}}"
environments:                    # chosen with --env <name>; override vars
  local:   { baseUrl: http://localhost:3000 }
  staging: { baseUrl: https://staging.example.com }
defaults:                        # applied to every request
  headers: { Accept: application/json }
  auth: { apiKey: { name: X-API-Key, value: "{{apiKey}}" } }
  timeout: 10000                 # ms (default 30000)
  retries: 0
  followRedirects: true
  expect:                        # merged into each request's expect
    headers:
      content-type: { contains: json }
tags: [orders]
data: data/customers.csv         # optional: run the whole file once per row
perf:                            # defaults for `openpost perf`
  concurrency: 10
  iterations: 200
  thresholds: { p95: "< 300", errorRate: "< 1%" }
requests:                        # required, run in order
  - …
```

## Request

| Key | Meaning |
|---|---|
| `name` | Required. Unique within the file (used by `--grep`, reports, docs). |
| `method` | `GET` (default), `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`. |
| `url` | Absolute, or relative to `baseUrl`. Supports `{{vars}}`. |
| `query` | Map of params; a list value repeats the param (`ids: [1, 2]`). |
| `headers` | Map; merged over `defaults.headers` (case-insensitive). |
| `json` | Object/array/scalar body, sent as `application/json`. |
| `body` | Raw string body (set `Content-Type` yourself). |
| `form` | `application/x-www-form-urlencoded` fields. |
| `graphql` | `{ query, variables, operationName }` sent as JSON. |
| `auth` | `{ bearer: "…" }`, `{ basic: { username, password } }`, `{ apiKey: { name, value, in: header\|query } }`; `null` disables `defaults.auth`. |
| `expect` | Assertions (below). Without `expect.status`, any status ≥ 400 fails. |
| `capture` | `varName: selector` — saved for later requests in this file. |
| `data` | Inline rows or `.csv`/`.json` path: run this request once per row; row keys become variables. |
| `timeout`, `retries`, `delay`, `followRedirects` | Per-request overrides (`retries` re-sends when the request or its assertions fail). |
| `skip` | `true` or a reason string. `only: true` runs only the marked requests. |
| `once` | Load tests: run once before the load (login/setup). Ignored by `test`. |
| `tags` | Filter with `--tag`. |
| `doc` / `description` | Markdown docs for `openpost docs`. |
| `perf.thresholds` | Per-endpoint load thresholds, e.g. `{ p95: "< 200" }`. |

Only one of `json`, `body`, `form`, `graphql` per request.

## Variables

- `{{name}}` — from (lowest → highest priority) `vars` → `environments.<env>` →
  `--env-file` → `--var name=value` → data row → captures.
- `{{$env.NAME}}` — process environment variable. Use for every secret.
- Built-ins: `{{$uuid}}` `{{$timestamp}}` (s) `{{$timestampMs}}` `{{$isoTimestamp}}`
  `{{$randomInt}}` `{{$randomInt(1,100)}}` `{{$randomString(12)}}` `{{$randomEmail}}`
  `{{$iteration}}` (data row / load iteration index).
- Dotted access into captured objects: `{{user.id}}`.
- A value that is exactly one placeholder keeps its type: `id: "{{userId}}"`
  stays a number if `userId` was captured as a number.
- Unresolved variables stop the request with a clear error (nothing is sent).

## Selectors (body assertions and captures)

| Selector | Selects |
|---|---|
| `$` | whole JSON body |
| `$.user.name`, `$["odd key"]` | object fields |
| `$.items[0]`, `$.items[-1]` | array index (negative from end) |
| `$.items[*].id` | all ids (array) — combine with `every`/`some`/`contains`/`length` |
| `$..id` | recursive: every `id` anywhere |
| `$.items.length` | array/string length (object: key count) |
| `header.location` | response header (captures and `body:` maps) |
| `status`, `time`, `body` | status code, total ms, raw body text |
| `regex:token=(\w+)` | first capture group in the raw body |

## Expectations

```yaml
expect:
  status: 201                  # or [200, 201], "2xx", "< 400", { in: [200, 204] }
  time: 500                    # total ms ≤ 500; or "< 500", { lt: 500 }
  size: "< 100000"             # body bytes
  json: true                   # body must be JSON (false: must not be)
  bodyContains: [created, id]  # raw substrings
  headers:
    content-type: { contains: application/json }
    cache-control: no-store
    x-request-id: { exists: true }
  body:                        # selector → expectation
    $.id: { type: integer, gt: 0 }
    $.status: active           # literal = equality
    $.email: "/^[^@]+@example\\.com$/"   # /regex/ string = matches
    $.total: ">= 1"            # comparison string
    $.items: { type: array, length: { between: [1, 50] } }
    $.items[*].price: { every: { gt: 0 } }
    $.items[*].sku: { contains: ABC-1 }
    $.deletedAt: { exists: false }
    $.meta: { contains: { page: 1 } }    # partial object match
  schema:                      # JSON Schema for the whole body
    type: object
    required: [id, items]
    properties:
      id: { type: integer }
      items: { type: array, items: { type: object, required: [sku] } }
```

An expectation is one of:

- a **literal** (`201`, `active`, `true`, `[1, 2]`, `{a: 1}`) → deep equality.
  Scalars compare by text, so `2` equals `"2"`; add `type:` to check types.
- a **comparison string**: `"< 500"`, `"<= 1"`, `"> 0"`, `">= 18"`, `"== x"`, `"!= 0"`.
- a **regex string**: `"/^abc/i"`.
- a **matcher object** (all keys must hold):

| Matcher | Passes when |
|---|---|
| `eq`, `ne` | equal / not equal |
| `gt`, `gte`, `lt`, `lte` | numeric comparison |
| `between: [a, b]` | `a ≤ x ≤ b` |
| `contains`, `notContains` | substring; array has element (or partial object); object has subset |
| `startsWith`, `endsWith` | string prefix/suffix |
| `matches`, `notMatches` | regex (`"pattern"` or `"/pattern/flags"`) |
| `in`, `notIn` | value is (not) one of a list |
| `exists: true/false` | selector finds (no) value |
| `type` | `string number integer boolean object array null` |
| `length` | number or matcher on length, e.g. `{ gte: 1 }` |
| `empty: true/false` | `""`, `[]`, `{}`, `null` |
| `schema` | JSON Schema for this value |
| `every`, `some` | matcher for every / at least one array item |

Supported JSON Schema keywords: `type` (incl. arrays of types, `nullable`),
`required`, `properties`, `additionalProperties`, `items`, `enum`, `const`,
`minimum`, `maximum`, `exclusiveMinimum/Maximum`, `multipleOf`, `minLength`,
`maxLength`, `pattern`, `format` (`email uuid date date-time uri ipv4`),
`minItems`, `maxItems`, `uniqueItems`, `contains`, `minProperties`, `allOf`,
`anyOf`, `oneOf`, `not`.

## Commands

| Command | Purpose |
|---|---|
| `openpost init [dir] [--base-url URL] [--ci github\|gitlab\|none]` | Scaffold tests, `.env.example`, CI workflow, `.gitignore`. |
| `openpost send <METHOD> <url> [-H] [-q] [-d] [-f] [--bearer] [--basic] [-e]… [-c]… [--save file --name n] [--json]` | One request with assertions; optionally saved to a test file. `-d @file.json` reads a file. |
| `openpost validate [paths]` | Check files; unknown keys/typos, bad regexes, missing baseUrl are errors (exit 2). No requests. |
| `openpost test [paths] [--env n] [--var k=v]… [--env-file f] [--grep p] [--tag t]… [--bail] [--retries n] [-r formats] [--report-dir d] [--json] [-k] [--show-secrets]` | Run tests. |
| `openpost perf [paths] [-c n] [-n n \| -d 30s] [--ramp-up 5s] [--rps n] [--warmup n] [-t expr]…` | Load test (plus `test` flags). `-t` replaces a file threshold on the same metric. |
| `openpost docs [paths] --format md\|html\|openapi\|openapi-yaml [--out f] [--run]` | Generate docs. |
| `openpost report a.json b.json -r html,junit` | Re-render or merge saved JSON reports (e.g. sharded CI). |
| `openpost schema` | JSON Schema for test files. |
| `openpost export tests <collection> [-o f]` / `openpost import tests <file>` | Convert API Studio collections ↔ test files. |
| `openpost import openapi <spec>` | Import an OpenAPI/Swagger spec as a collection. |
| `openpost run <collection> -r html,junit` | Run an API Studio collection with reports. |
| `openpost agent install [--platform …]` | Install these agent instructions for Claude Code, Cursor, Kiro, Copilot, Windsurf, Codex/AGENTS.md, Gemini, Cline. |

Global flags: `--json` (machine output), `--quiet`, `--verbose` (show passing
assertions and captures), `--no-color`, `--timeout <ms>`, `--dry-run` (list
requests without sending), `--output-file <path>` (also write JSON report).

`--grep`: substring (case-insensitive), glob (`"Get*"`), or `/regex/` (exact: `"/^List users$/"`).
`-r`: `json`, `junit`, `html`, `md`; `name:path` sets a file, e.g. `-r junit:out/junit.xml`.

## JSON output (`--json`)

```jsonc
{
  "ok": false,
  "kind": "test",
  "summary": { "requests": 8, "passed": 7, "failed": 1, "skipped": 0, "assertions": 33, "assertionsFailed": 1, "passRate": 87.5, "durationMs": 48 },
  "metrics": { "overall": { "latency": { "p50": 2, "p95": 14, "p99": 19 }, "throughput": 166.7, "errorRate": 0.125 }, "endpoints": [ … ] },
  "suites": [ { "name": "Tasks API", "file": "…", "requests": [ {
      "name": "List tasks", "method": "GET", "url": "http://…/tasks", "status": 200, "ok": false,
      "timings": { "ttfb": 2.1, "download": 0.3, "total": 2.4 },
      "assertions": [ { "target": "body $[*].title", "rule": "contains \"x\"", "passed": false, "actual": ["a", "b"] } ],
      "captured": { "taskId": "5" }, "response": { "headers": { … }, "body": "…" }
  } ] } ],
  "perf": { "concurrency": 20, "iterations": 1000, "thresholds": [ { "rule": "p95 < 250", "actual": 180, "passed": true } ] }
}
```

Reports mask sensitive headers and every `--env-file`/`$env` secret value (`--show-secrets` to disable).

## Recipes

**Login once, reuse the token**

```yaml
defaults:
  auth: { bearer: "{{token}}" }
requests:
  - name: Login
    once: true
    method: POST
    url: /auth/login
    auth: null
    json: { username: "{{$env.API_USER}}", password: "{{$env.API_PASSWORD}}" }
    expect: { status: 200, body: { $.token: { type: string } } }
    capture: { token: $.token }
```

**CRUD round trip** — create (capture id) → get (assert fields) → update →
get (assert change) → delete → get (`status: 404`).

**Negative cases** — no auth (`auth: null` → 401), invalid body (400/422),
unknown id (404), duplicate (409), wrong method (405).

**Pagination**

```yaml
- name: Page 2
  url: /items
  query: { page: 2, limit: 10 }
  expect:
    body:
      $.items.length: "<= 10"
      $.page: 2
      $.items[*].id: { every: { type: integer } }
```

**Data-driven validation**

```yaml
- name: Reject invalid emails
  method: POST
  url: /users
  json: { email: "{{email}}" }
  data:
    - { email: "" }
    - { email: "not-an-email" }
    - { email: "a@b" }
  expect: { status: 422 }
```

**Eventual consistency** — `retries: 5` with `delay: 500` on the request that polls.

**From OpenAPI** — `import openapi spec.yaml`, `export tests "<title>" -o
api-tests/contract.api.yaml`, then add `schema:` per response.

**Replacing a Python script** — each `requests.<method>()` call becomes a
request, each `assert` an `expect`, each reused value a `capture`.
