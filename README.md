# Reusable Skills and Policy Hooks

PortfolioPilot is a stock portfolio teaching project with portfolio valuations, news, and an
assistant that explains authorized holdings and cited news. This learning activity adds reusable
instructions and policy checks to that application assistant. It does not add broker trading.

**Status:** milestone 24 is implemented. Mock briefings, database ownership checks, audit correlation,
and actual Claude Agent SDK startup isolation were verified. A live Claude-generated briefing was
not verified because the required credentials and model configuration were unavailable.

## 1. Purpose and What You Will Learn

The prompt asked the coding agent to build two **application-agent skills**:

- **Daily portfolio briefing:** organize valuation coverage, holdings, relevant news, risks, and
  research follow-ups into a consistent answer.
- **Earnings-news review:** separate reported earnings facts, portfolio exposure, interpretation,
  counterarguments, and uncertainties.

A **skill** is reusable Markdown instruction text. It describes how the assistant should approach
a task and organize its answer. A **tool** is executable application code, such as a function that
reads a portfolio from the database. Loading a skill does not authorize a trade or grant access to
someone else's holdings.

The prompt also asked for **hooks**: callbacks that the SDK runs before or after a tool operation.
They provide an additional policy check and a sanitized audit trail. **Authorization** means checking
whether the signed-in user may access a resource; tools must still perform that check themselves.

You will learn how to:

- Reuse task instructions without mixing them with the coding agent's project or personal settings.
- Configure the application's working directory and configuration sources explicitly.
- Deny forbidden operations before execution while retaining authorization inside tools.
- Record audit events that can be connected to a particular conversation and run.
- Test configuration isolation without making a paid model call.
- Distinguish deterministic mock results from observed live-model behavior.

## 2. Steps Actually Performed

### Step 1: Inspect the project and verify SDK support

The implementation began by reading [AGENTS.md](AGENTS.md), the
[project state](docs/project-state.md), and the [milestone plan](docs/project-plan.md), then inspecting
the existing SDK adapter, mock adapter, tools, specialist policy, and tests.

The agent read the official SDK documentation for
[skills](https://code.claude.com/docs/en/agent-sdk/skills),
[hooks](https://code.claude.com/docs/en/agent-sdk/hooks), and
[settings](https://code.claude.com/docs/en/settings), and checked the installed SDK type definitions.
The TypeScript reference webpage could not be retrieved; installed `sdk.d.ts` supplied the checked
API signatures. No dependency versions were upgraded: this activity uses SDK **0.3.276** and its
bundled CLI **2.1.276**.

Dependencies were restored from the existing local npm cache:

```powershell
npm ci --ignore-scripts --offline --cache .npm-cache
```

That command installed 348 packages and reported zero audit vulnerabilities at verification time.

### Step 2: Create the two application skill artifacts

The following files were created:

```text
packages/agent/runtime/skills/
  daily-portfolio-briefing/SKILL.md
  earnings-news-review/SKILL.md
```

Each file contains a name, a description that helps select the skill, and task instructions.
The instructions require cited evidence, freshness and synthetic-data labels, explicit uncertainty,
and a visible `Skill used:` marker. They contain no executable resources or permission grants.

### Step 3: Manage runtime loading and isolate configuration

`packages/agent/src/application-skills.ts` was added, and `packages/agent/src/index.ts` was updated
to prepare the live application's skills and configure the SDK. The key decisions were:

| Setting | Application behavior |
| --- | --- |
| `cwd` | Use the explicitly configured application runtime directory. |
| `CLAUDE_CONFIG_DIR` | Point SDK configuration and session storage at that same directory. |
| `settingSources: ['user']` | Load only the application's overridden configuration source. |
| Exact `skills` names | Enable the two reviewed application skills. |
| `plugins: []` and disabled cloud sync | Exclude personal plugins and synced skills/plugins. |
| Strict MCP configuration | Use explicitly supplied application tool servers. |

Here, the SDK source named `user` resolves to the application's **overridden configuration
directory**, not the developer's personal home directory. Project and local sources are excluded,
so ancestor project settings and instructions are not loaded. **MCP**, or Model Context Protocol,
is the protocol used to expose application tools to the SDK.

The runtime directory must be absolute and outside the source repository and personal agent
configuration. Unmanaged settings, extra skill resources, symbolic links, and changed instruction
files are rejected instead of silently overwritten. Live chat enables the `Skill` tool; shell,
filesystem, web, and mutation operations remain denied. Specialist agents cannot invoke these skills.
Shared article analysis remains configured with no skills or tools.

### Step 4: Add policy checks and sanitized audit events

`packages/agent/src/policy-hooks.ts` was created with:

- `PreToolUse`: check exact tool/skill names and MCP origin, then compose the existing specialist policy.
- `PostToolUse`: record success or a tool-reported error.
- `PostToolUseFailure`: record an execution failure.

The existing depth and usage restrictions in `packages/agent/src/delegation.ts` were preserved.
`packages/agent/src/streaming.ts` and `apps/api/lib/chat.ts` were updated so the server supplies the
authenticated actor and persisted conversation/run IDs. A **run** is one execution of an assistant
request. Audit events use those IDs, a UTC timestamp, tool name, outcome, and fixed metadata.
Arguments, responses, raw errors, credentials, SDK paths, and hidden reasoning are discarded.

### Step 5: Extend the mock adapter and acceptance tests

`packages/agent/src/mock-portfolio-agent.ts` now selects skills deterministically, reads the managed
instructions, follows their headings, and invokes the existing authorized read handlers. It labels
its output as a mock skill simulation and replays the policy/audit callbacks.

New tests were added in `packages/agent/test/skills-policy.test.ts` and
`packages/agent/test/skills.live.test.ts`. Existing adapter, streaming, registration, session, and
PostgreSQL tool tests were updated. The live test is optional and remains skipped by default.

The clean-directory test actually started the installed SDK subprocess with planted personal and
ancestor configuration. It discovered the application skills, confirmed hook registration, and
excluded the planted skills and MCP settings. It sent no prompt and made no model call.

### Step 6: Run checks and resolve local verification issues

The following repository commands were run successfully after resolving the issues below:

```powershell
npm run build:types
npm run typecheck
npm run lint
npm run build
npm run check:browser-boundary
$env:DATA_MODE='mock'
npm run test
```

Initial builds encountered Prisma's existing `EPERM` cache timestamp error. Pointing
`PRISMA_SCHEMA_ENGINE_BINARY` at the existing readable cached engine allowed generation to proceed.
A direct invocation of `scripts/run-workspaces.mjs` was rejected because it requires npm; subsequent
commands ran through npm. The Windows npm shim also required using the installed Node binary and
its adjacent `npm-cli.js`.

A new Next.js warning about tracing the whole project was resolved by making the skill asset reads
explicit and excluding external runtime paths from source tracing. The final API build was rerun.

Additional verification used real PostgreSQL/Redis integration tests and an HTTP smoke test against
the compiled Next.js API on loopback port 3014. The smoke test signed in as Alice, created a
conversation and briefing run, checked the persisted answer and correlated audit, then stopped the
server and removed its test conversation. Demo authentication was configured for development;
this was not a production authentication test.

### Step 7: Record the outcome

The implementation updated `apps/api/.env.example`, [project state](docs/project-state.md),
[version notes](docs/versions.md), [lesson 24](docs/lessons/24-application-skills-and-policy-hooks.md),
and [ADR 0017](docs/decisions/0017-managed-application-skills-and-audit.md).
An **ADR** is an architecture decision record explaining a design choice and its consequences.
This root README was created afterwards; there was no existing root README to preserve.

## 3. Results Achieved

### Observed verification results

| Check | Observed result during milestone 24 |
| --- | --- |
| Typecheck, lint, full build, browser dependency boundary | Passed. |
| Full mock test suite | 273 passed; 76 opt-in tests skipped. |
| Final agent suite | 124 passed; two live tests skipped. |
| Focused skill/policy suite | 7 passed. |
| PostgreSQL agent-tool acceptance | 9 passed. |
| PostgreSQL/Redis chat and event acceptance | 11 passed. |
| Actual SDK startup in a clean runtime | Managed skills found, hooks registered, planted configuration excluded. |
| Compiled API HTTP smoke | Persisted briefing, five headings, skill marker, and correlated audit passed. |

The separate integration counts are not added to the full-suite count. Their opt-in tests were
enabled in separate runs. These are recorded implementation results, not a claim that every
environment will produce the same results without setup.

### Resulting briefing behavior

The observed mock briefing uses these headings in order:

```markdown
[Mock answer] Skill used: daily-portfolio-briefing

## As of and coverage
## Portfolio snapshot
## Relevant news
## Risks and uncertainties
## Research follow-ups
```

The heading outline above omits the body for readability. The verified fixture body included
`mock skill simulation`, UTC timestamps, a synthetic-news citation, and values from the portfolio
tools, such as `ACME: ... market value 1001.00 USD`. Those values are teaching fixtures, not current
market data. The mock reports missing quotes and an explicit absence of relevant stored news when
those conditions occur.

Earnings-review mock output uses four headings: **Reported earnings facts**, **Portfolio exposure**,
**Interpretation and counterarguments**, and **Uncertainties and follow-up research**.

### Policy and audit behavior

The policy callback tests reject `Bash`, the forbidden `mcp__portfolio__recordTrade` sentinel,
unknown skills, and falsely attributed MCP tools. The trade sentinel is a test name; no broker tool
was implemented. These checks do not rely on a model obeying the prompt.

The real PostgreSQL acceptance test also showed that Bob could pass the generic read-tool policy
but still fail the tool's ownership check when requesting Alice's portfolio. Its audit recorded the
read as allowed before execution and failed afterwards.

An audit event has this shape; the IDs below are illustrative:

```json
{
  "event": "agent.tool.audit",
  "actorId": "demo-alice",
  "conversationId": "conversation-24",
  "runId": "run-24",
  "time": "2026-10-02T12:00:00.000Z",
  "toolName": "Skill",
  "outcome": "succeeded",
  "metadata": {
    "phase": "post",
    "subagent": false,
    "skill": "daily-portfolio-briefing",
    "redacted": true
  }
}
```

**Redacted** means sensitive payload details are omitted. The default audit destination is the API's
server log. Search for `agent.tool.audit` and a run ID to connect events to the corresponding request.

## 4. How to Run and Verify

Run commands from the repository root. Examples below use Windows PowerShell. Setup commands are
instructions for your environment; this documentation update did not rerun the application checks.

### Prerequisites

- Node.js **24.21.0** and npm **11.19.0**, as pinned by this project.
- Docker with a running engine for local PostgreSQL and Redis.
- Git for Windows/Git Bash when running the native SDK startup test on Windows.
- Free local ports 5173 for React, 3001 for the API, 5432 for PostgreSQL, and 6379 for Redis.
- The repository's migrations and synthetic demo seed for database-backed demonstrations.

No Claude or market-provider credentials are needed for mock mode. An npm workspace is one package
inside this repository; `-w @portfolio-pilot/agent` runs a command for that package only.

### Install and prepare the local demo

For a normal installation:

```powershell
node --version
npm --version
npm ci --ignore-scripts
```

If the included cache has all required packages, the offline install command in Step 1 is an
alternative. Next, configure this shell for the local demo:

```powershell
$env:NODE_ENV='development'
$env:DATA_MODE='mock'
$env:VITE_DATA_MODE='mock'
$env:AGENT_MODE='mock'
$env:AGENT_SPECIALISTS_ENABLED='false'
$env:RESEARCH_MCP_MODE='off'
$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_pilot'
$env:REDIS_URL='redis://127.0.0.1:6379'
$env:DEMO_AUTH_ENABLED='true'
$env:AUTH_BASE_URL='http://localhost:5173'
$env:AUTH_SECRET='local-course-demo-signing-secret-only-1234567890'

npm run infra:start
npm run build:types
npm run migrate:deploy -w @portfolio-pilot/db
$env:ALLOW_DEMO_SEED='true'
npm run seed:demo -w @portfolio-pilot/db
Remove-Item Env:ALLOW_DEMO_SEED
npm run dev
```

The example database password and signing secret are public, local teaching values. The seed checks
for development/test mode, explicit opt-in, and a loopback database address. **Migrations** create or
update the database schema; the **seed** inserts fictional example records. `npm run dev` starts
React and the API, not background worker roles. Existing example environment files are another
configuration reference; avoid replacing your own files.

### Try the briefing in the browser

1. Open `http://localhost:5173` and sign in using the Alice demo account.
2. Open **Assistant** and create a conversation scoped to **Growth**.
3. Ask: `Give me a daily portfolio briefing`.
4. Confirm the skill marker, five ordered headings, mock/freshness labels, and cited news or explicit
   lack of recent stored news. Old seed articles may fall outside the briefing's 24-hour window.
5. Ask: `Review earnings news` and look for the four earnings headings.
6. Reload and check that completed answers remain in the conversation.
7. Inspect the API terminal's `agent.tool.audit` events and their conversation/run IDs.

These are expected UI steps. This milestone verified the equivalent authenticated HTTP flow and
persistence, but did not run a browser visual test.

### Run the focused acceptance test

After installing dependencies and building shared packages:

```powershell
npm run test -w @portfolio-pilot/agent -- test/skills-policy.test.ts
```

This suite needs no database or Claude key. It checks both answer structures, unavailable evidence,
forbidden operations, audit redaction, specialist-policy composition, and actual SDK startup
isolation in a temporary clean directory.

### Run database-backed acceptance

Use a disposable, migrated loopback database whose name ends in `_verify`. The implementation used
an existing `portfolio_m22_verify` database on port 5546. For a new student environment, the following
commands prepare a separate database on the normal local PostgreSQL container. Run `createdb` once;
if that database already exists, skip its creation.

```powershell
docker exec portfolio-pilot-local-postgres-1 createdb -U portfolio_local portfolio_m24_verify
$demoDatabaseUrl=$env:DATABASE_URL
$env:DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5432/portfolio_m24_verify'
npm run migrate:deploy -w @portfolio-pilot/db
$env:AGENT_TOOLS_TEST_DATABASE_URL=$env:DATABASE_URL
npm run test -w @portfolio-pilot/api -- lib/agent-tools.integration.test.ts
$env:DATABASE_URL=$demoDatabaseUrl
Remove-Item Env:AGENT_TOOLS_TEST_DATABASE_URL
```

The acceptance suite seeds its demo prerequisites. It checks real owner-scoped reads and correlated
briefing audit records. The new-database setup above is a reproduction recipe, not an additional
verification run performed during this activity.

For the separately verified chat/SSE suite, use the already migrated dedicated `portfolio_m19_verify`
database required by that suite and an isolated local Redis database:

```powershell
$env:CHAT_TEST_DATABASE_URL='postgresql://portfolio_local:local_only_change_me@127.0.0.1:5546/portfolio_m19_verify'
$env:CHAT_TEST_REDIS_URL='redis://127.0.0.1:6379/11'
npm run test -w @portfolio-pilot/api -- lib/chat.integration.test.ts lib/agent-run-events.test.ts
Remove-Item Env:CHAT_TEST_DATABASE_URL
Remove-Item Env:CHAT_TEST_REDIS_URL
```

Port 5546 is the implementation machine's separate verification container, not the default Compose
port. Adjust the port for your test database. **SSE** means Server-Sent Events: the server streams
updates to the browser over a persistent HTTP connection.

### Run repository checks

```powershell
npm run typecheck
npm run lint
npm run build
npm run check:browser-boundary
$env:DATA_MODE='mock'
npm run test
```

The browser-boundary check verifies that browser packages do not import backend dependencies.
The repository's current `lint` scripts run TypeScript checks. Integration and live tests remain
skipped unless their required environment variables are supplied.

### Optional live Claude verification

First securely supply `ANTHROPIC_API_KEY`, a verified `AGENT_MODEL_ID`, and a clean absolute
`AGENT_WORKSPACE_DIR` outside this repository and personal agent configuration. Do not put a key
in browser configuration, skill text, or committed files. Then explicitly opt in:

```powershell
$env:RUN_LIVE_SKILLS_TEST='true'
npm run test -w @portfolio-pilot/agent -- test/skills.live.test.ts
Remove-Item Env:RUN_LIVE_SKILLS_TEST
```

This optional test makes a paid model call using synthetic owner-bound fixtures. It checks actual
Skill completion, briefing headings, and audit correlation. It has a $0.10 SDK budget cap, ten-turn
cap, and 60-second runtime timeout. It was **not executed** during milestone 24.

### Troubleshooting the issues encountered

- **Windows npm shim:** use your installed Node directory on PATH and invoke its adjacent
  `node_modules/npm/bin/npm-cli.js`. For example, after setting `$nodeInstall` to your installation:

  ```powershell
  $env:PATH="$nodeInstall;"+$env:PATH
  & "$nodeInstall/node.exe" "$nodeInstall/node_modules/npm/bin/npm-cli.js" run build:types
  ```

- **Prisma cache `EPERM` / `utime`:** set `PRISMA_SCHEMA_ENGINE_BINARY` to the absolute path of your
  existing readable cached schema engine, then retry `npm run build:types`. The successful
  implementation used that workaround rather than changing dependencies or filesystem permissions.
- **Docker access denied:** make sure the engine is running and your shell can access it. Default
  shell inspection was denied during implementation; an approved elevated read confirmed the
  existing containers. Database acceptance subsequently connected to their loopback ports.
- **Dirty runtime or modified skills:** choose a new clean application runtime directory. The loader
  intentionally refuses to overwrite unmanaged instructions.

## 5. Limitations and Unfinished Work

- Live Claude skill selection and answer adherence remain unverified. Mock replay and native SDK
  startup discovery do not establish live-model behavior.
- The full-suite run skipped 76 opt-in tests: 63 API tests, 11 worker tests, and two live Claude tests.
  Selected database suites were enabled and passed separately, as listed above.
- The TypeScript documentation endpoint was unavailable; installed SDK definitions were checked instead.
- Existing Vite directive/chunk-size warnings and three Next.js instrumentation Edge warnings remain.
  The new whole-project tracing warning introduced by the skill loader was resolved.
- No browser visual test, production authentication test, public deployment, or cloud provisioning
  was performed for this milestone.
- Audit events currently go to server logs; durable audit storage and retention are unfinished.
- A separate runtime directory isolates configuration; it is not an operating-system sandbox.
  SDK transcripts remain sensitive and stored on the current host.
- Durable agent workers, portable session storage, and production container packaging remain later
  milestones. The next milestone is **25: approvals and explicit cancellation**. Existing explicit
  cancellation predates this activity; the milestone 25 approval workflow has not been implemented.

For the detailed evidence, full file inventory, and next steps, read
[project state](docs/project-state.md), [lesson 24](docs/lessons/24-application-skills-and-policy-hooks.md),
and [ADR 0017](docs/decisions/0017-managed-application-skills-and-audit.md).
