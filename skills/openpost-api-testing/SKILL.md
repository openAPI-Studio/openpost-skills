---
name: openpost-api-testing
description: Test, load-test and document HTTP/REST/GraphQL APIs with the openpost CLI instead of writing ad-hoc Python/JS/curl scripts. Use whenever you need to call an API (localhost or remote), verify responses, check an endpoint you just built or changed, reproduce an API bug, measure latency/throughput, generate API docs, or add API tests to CI. Tests live in git-friendly *.api.yaml files.
---

# API testing with openpost

`openpost` is a headless API client. You describe requests and expectations in
YAML test files (`*.api.yaml`), run them with one command, and get a precise
pass/fail result, machine-readable JSON, and HTML/JUnit/Markdown reports. The
same files run in CI, double as API documentation, and drive load tests.

**Use it instead of** throwaway `python -c "import requests…"`, `node -e "fetch…"`,
or long `curl | jq` pipelines whenever you check an API. Those scripts get
deleted; a test file stays in the repo, so the next person (or agent) can rerun
it, and CI can enforce it.

## Golden rules

1. **Persist what you verify.** Any check worth running twice belongs in a
   `*.api.yaml` file under `api-tests/` (or the project's existing test dir).
   Use `openpost send` only for a first look, then save it with `--save`.
2. **Never put secrets in test files.** Use `{{$env.NAME}}` and pass values via
   environment variables or `--env-file api-tests/.env` (gitignored). Values from
   `--env-file` fill both `{{NAME}}` and `{{$env.NAME}}`. Reports and console
   output mask secret values (`••••••`); `--show-secrets` turns that off for
   local debugging only.
3. **Assert on behaviour, not noise.** Check status, key fields, types and
   schemas. Don't assert on timestamps, generated IDs or ordering unless that is
   the behaviour under test. Capture IDs and reuse them instead.
4. **Read failures from the output, don't guess.** Every failed assertion shows
   the selector, the rule, and the actual value. Fix the test or the code based
   on that, then rerun.
5. **Use `--json` when you need to parse results**, and exit codes for
   pass/fail: `0` passed · `1` assertions/thresholds failed · `2` invalid file,
   bad flags, or nothing matched.

## Setup

```bash
npx openpost --version            # no install needed; or: npm i -D openpost
npx openpost init                 # api-tests/example.api.yaml, .env.example, CI workflow, .gitignore entries
```

If the project already has `api-tests/` (or other `*.api.yaml` files), read a
couple of them first and follow their conventions (file split, naming, vars).

## Workflow

### 1. Explore an endpoint (one-off)

```bash
openpost send GET http://localhost:3000/health -e status=200
openpost send POST localhost:3000/users -d '{"name":"Ada"}' -e status=201 -e '$.id exists' -c id=$.id
openpost send GET localhost:3000/users/42 --bearer "$TOKEN" --json      # full response as JSON
```

`-e/--expect` one-liners: `status=200` `status=2xx` `time<500` `$.id exists`
`$.deleted !exists` `$.name="Ada"` `$.age>=18` `$.items.length>0`
`header.content-type~json` (contains) `$.slug~/^[a-z-]+$/` (regex) `$.tags type=array`.
That is the complete `-e` operator set; for anything else (schemas, `oneOf`,
`length`, data rows) write the YAML.

### 2. Save it as a test

```bash
openpost send GET localhost:3000/users -e status=200 -e '$.length>0' \
  --save api-tests/users.api.yaml --name "List users"
```

`--save` creates the file (with `baseUrl`) or appends/replaces the request by
name. Tokens and passwords in headers or query strings are moved to
`{{$env.NAME}}` automatically and the CLI prints which variables to export.
Then edit the YAML to add captures, schemas and more cases.

### 3. Write / extend the test file

```yaml
# api-tests/users.api.yaml
name: Users API
baseUrl: "{{baseUrl}}"
vars:
  baseUrl: http://localhost:3000
environments:                         # pick with --env staging
  staging: { baseUrl: https://staging.example.com }
defaults:
  headers: { Accept: application/json }
  auth: { bearer: "{{$env.API_TOKEN}}" }

requests:
  - name: Create user
    method: POST
    url: /users
    json: { name: Ada, email: "{{$randomEmail}}" }
    expect:
      status: 201
      body:
        $.id: { type: integer }
        $.name: Ada
    capture:
      userId: $.id                    # available as {{userId}} in later requests

  - name: Get user
    url: /users/{{userId}}
    expect:
      status: 200
      time: "< 300"
      schema:
        type: object
        required: [id, name, email]
        properties:
          id: { type: integer }
          email: { type: string, format: email }

  - name: Reject duplicate email
    method: POST
    url: /users
    json: { name: Ada, email: taken@example.com }
    expect:
      status: 409
      body:
        $.error: { contains: already }
```

Requests in a file run **in order**, and captures flow forward. Separate files
are independent. Full syntax: see [reference.md](reference.md).

### 4. Validate, then run

```bash
openpost validate api-tests/                       # schema check; unknown keys/typos are errors
openpost test api-tests/ --env local               # run everything
openpost test api-tests/users.api.yaml --grep "Get user"   # one request (+ the requests it depends on)
openpost test api-tests/ --grep "/^List users$/"   # exact name (plain --grep matches substrings)
openpost test --var baseUrl=http://localhost:8080 --env-file api-tests/.env
openpost test --json > /tmp/result.json            # parse summary.failed, suites[].requests[].assertions
```

`--grep`/`--tag` automatically include earlier requests whose captures the
selected ones need, so `--grep "Get user"` still runs "Create user" first.

### 5. Read the result and iterate

A failure looks like:

```
✗ GET    Get user  200 12ms
  ✗ body $.email matches schema (actual: …) — $.email: not a valid email
```

- Wrong expectation (the API is right) → fix the YAML.
- API bug → fix the code, rerun the same command. Don't weaken the assertion
  just to make it pass.
- `Unresolved variable(s): {{x}}` → define `x` in `vars`, `--var x=…`,
  `--env-file`, or capture it from an earlier request. For `{{$env.X}}`, export
  `X` or put it in the `--env-file`.
- `Not sent: {{x}} would come from "Login", which failed` → fix that earlier
  request first; the later failures are only a consequence.
- `connection refused` → the server isn't running; start it (in the
  background) and wait for its health endpoint before rerunning.

### 6. Report

```bash
openpost test api-tests/ -r html,junit,md,json --report-dir reports
```

- `reports/report.html` — self-contained dashboard: verdict, KPIs, latency
  histogram, status codes, slowest endpoints, every request with assertions,
  headers and bodies. Share or attach it.
- `reports/junit.xml` — for CI test tabs. `reports/report.md` — paste into a
  PR/issue. `reports/report.json` — full machine-readable result.
- In GitHub Actions the Markdown summary is added to the job summary
  automatically.

Summarise for the user: passed/failed counts, which requests failed and why
(selector, expected, actual), p95 latency, and the report path.

## Load / batch testing

```bash
openpost perf api-tests/users.api.yaml -c 20 -n 1000 -t "p95<250" -t "errorRate<1%"
openpost perf api-tests/ --grep "List users" -c 50 -d 30s --ramp-up 5s --rps 200 -r html
```

- Each iteration runs the selected requests in order (a user flow); `-c` virtual
  users run in parallel. `-n` total iterations or `-d` duration.
- Mark login/setup requests `once: true` so they run once before the load and
  share their captures (e.g. a token).
- Thresholds (`-t` or `perf.thresholds` in the file) make the run fail with exit 1:
  metrics `p50 p90 p95 p99 max avg ttfb_p95 errorRate errors rps count`.
  An `errorRate < 1%` threshold is applied by default. A `-t` on the command
  line replaces a file threshold on the same metric (e.g. `-t "p95<500"` wins
  over the file's `p95<250`).
- Load-test only local, staging or explicitly approved targets. **Never load-test
  production or third-party APIs unless the user explicitly asks.**
- Report latency percentiles (p50/p95/p99), throughput (req/s), error rate and
  which thresholds failed.

## Documentation from tests

```bash
openpost docs api-tests/ --format md   --out docs/API.md
openpost docs api-tests/ --format html --out docs/api.html --run   # --run embeds real example responses
openpost docs api-tests/ --format openapi --out openapi.json       # OpenAPI 3 from the tests
```

Add a `doc:` (Markdown) field to requests and a `description:` to the file to
make the generated docs read well. One file per resource (`users.api.yaml`,
`orders.api.yaml`) with a clear `name:` gives one docs section each; give
requests names that read as operations ("Create user", "Get user by id"). Docs generated this way only describe
behaviour that the tests verify.

## CI

`openpost init` writes `.github/workflows/api-tests.yml` (or `--ci gitlab`). The
core step is:

```bash
npx --yes openpost test api-tests --env local -r junit,html,json --report-dir reports
```

Pass secrets as CI secrets mapped to environment variables read through
`{{$env.NAME}}`. Upload `reports/` as an artifact.

## Working with the API Studio app

Collections made in the API Studio VS Code extension or desktop app can be
turned into test files and back:

```bash
openpost export tests "My Collection" -o api-tests/my-collection.api.yaml   # secrets → {{$env.*}}
openpost import tests api-tests/users.api.yaml                              # opens in the app
openpost run "My Collection" -r html,junit                                  # run a GUI collection with reports
```

## Checklist before you finish

- [ ] Tests are in `*.api.yaml` files in the repo, not in a temp script.
- [ ] `openpost validate` passes with no errors.
- [ ] `openpost test` passes (or the failures are real bugs you reported).
- [ ] No tokens, passwords or keys in the files; `.env` is gitignored.
- [ ] Happy path **and** at least one negative case (4xx) per endpoint you touched.
- [ ] You told the user the command to rerun the tests and where the report is.
