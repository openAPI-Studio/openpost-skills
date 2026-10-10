# openpost API testing skill

Teaches AI coding agents to test, load-test and document HTTP APIs with the
[`openpost`](https://www.npmjs.com/package/openpost) CLI instead of writing
throwaway Python, JavaScript or curl scripts.

The agent writes `*.api.yaml` test files that live in your repository, runs
them with `npx openpost test`, reads failures from the output, and produces
HTML, JUnit, Markdown and JSON reports. The same files drive load tests
(`openpost perf`) with latency and error-rate thresholds, generate API docs
(Markdown, HTML or OpenAPI), and run in CI.

## Install

- **Claude Code:** `/plugin marketplace add openAPI-Studio/openpost-skills`, then `/plugin install openpost@openpost-skills`
- **Kiro:** Powers → Add Custom Power → Import power from GitHub → `https://github.com/openAPI-Studio/openpost-skills`
- **Any agent (Cursor, Copilot, Windsurf, Codex, Gemini, Cline):** `npx openpost agent install`

## Requirements

Node.js 18 or newer. The skill runs `npx openpost`, which downloads the CLI on
first use. Requests go only to the APIs you point it at; nothing is sent
anywhere else.

## Docs

https://docs.vectored.dev/apistudio/docs/ai-agent-skills.html

## Source

This repository is published from the `openpost` CLI's source. Edit the skill
there and copy the `skills/` folder here for each release.

## Privacy

The skill and the openpost CLI collect no telemetry, analytics or personal
data. Requests go only to the APIs you point them at, and reports are written
to your own disk. Secrets come from your environment or a git-ignored `.env`
file and are redacted from reports.

Privacy Policy: https://docs.vectored.dev/privacy.html

## Support

- Email: support@vectored.dev
- Support page: https://docs.vectored.dev/apistudio/support.html
- Issues: https://github.com/openAPI-Studio/openpost-skills/issues

## License

MIT. The skill files in this repository are MIT-licensed; the openpost CLI and API Studio app have their own license.
