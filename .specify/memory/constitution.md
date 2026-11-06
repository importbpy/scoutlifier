<!--
Sync Impact Report
==================
Version change: (unversioned template) → 1.0.0
Bump rationale: Initial ratification; all placeholders replaced with project-specific content.

Modified principles (template placeholder → new title):
  - [PRINCIPLE_1_NAME] → I. Clean Architecture
  - [PRINCIPLE_2_NAME] → II. Human-Readable Code
  - [PRINCIPLE_3_NAME] → III. Fail Safe, Never Lose a Job (NON-NEGOTIABLE)
  - [PRINCIPLE_4_NAME] → IV. Configuration Through Environment
  - [PRINCIPLE_5_NAME] → V. Observable State Transitions
Added principles:
  - VI. Zero-Cost Operation
Added sections:
  - Pipeline Contract (replaces [SECTION_2_NAME])
  - Documentation & Development Workflow (replaces [SECTION_3_NAME])
Removed sections: none

Templates requiring updates:
  - .specify/templates/plan-template.md      ✅ no change required (reads constitution at runtime)
  - .specify/templates/spec-template.md      ✅ no change required
  - .specify/templates/tasks-template.md     ✅ no change required
Follow-up TODOs: none
-->

# Scoutlifier Constitution

Scoutlifier is an automated AI job scouting CLI agent. It fetches job listings, filters out
listings it has already seen, asks an LLM whether each new listing is relevant, notifies a
Discord channel about relevant listings, and records what it has processed.

## Core Principles

### I. Clean Architecture

- The codebase MUST be organized in layers with dependencies pointing inward only:
  **Domain** (job listing, evaluation result, pure rules) → **Application** (use cases such as
  "scout jobs") → **Adapters/Infrastructure** (scraper, database, LLM client, Discord client,
  configuration loader, logger) → **Entry point** (CLI).
- The domain and application layers MUST NOT import HTTP clients, database drivers, LLM SDKs,
  or environment-reading code. They depend only on interfaces (ports) they define.
- Each external system (job source, state store, LLM provider, notifier) MUST sit behind its
  own interface so it can be replaced or faked in tests without touching business logic.
- Wiring of concrete implementations MUST happen in a single composition root at the entry point.

**Rationale**: Free-tier LLM providers, job sites, and storage choices will change. Isolating
them keeps the core pipeline stable and testable.

### II. Human-Readable Code

- Names MUST be descriptive and spelled out. Abbreviations are forbidden unless widely
  understood (e.g., `id`, `url`, `http`, `json`, `api`, `llm`, `db`).
- Functions MUST do one thing and be short enough to understand without scrolling; prefer
  explicit control flow over clever constructs.
- Comments explain *why*, not *what*. Code that needs a comment to explain *what* MUST be
  rewritten for clarity first.
- Public types and use cases MUST carry type annotations / explicit types where the language
  supports them.

**Rationale**: This is a small, long-running personal agent; it must stay easy to revisit
months later.

### III. Fail Safe, Never Lose a Job (NON-NEGOTIABLE)

- A job MUST be marked as processed **only after** evaluation succeeded and, when relevant,
  the Discord notification was delivered successfully.
- If any step fails (scraper error, LLM timeout, rate limit, invalid or unparseable JSON,
  Discord error), the job MUST remain unprocessed so it is retried on the next run.
- A failure processing one job MUST NOT abort processing of other jobs; a failure of one
  target URL MUST NOT abort other target URLs.
- LLM responses MUST be validated against the evaluation schema (see Pipeline Contract)
  before use; any deviation is treated as a failure, never guessed or defaulted.
- All network calls MUST have explicit timeouts. Retries MUST be bounded and use backoff;
  rate-limit responses (e.g., HTTP 429) MUST be respected.
- The process MUST exit with a non-zero status when a run is aborted by a fatal error
  (e.g., missing required configuration), and with zero when it completed, even if
  individual jobs were deferred for retry.

**Rationale**: Silently dropping a job defeats the purpose of the agent; duplicates from a
retry are cheap, misses are not.

### IV. Configuration Through Environment

- All secrets and configuration (LLM API keys, LLM provider/model, Discord Webhook URL,
  target URLs, polling settings, database path) MUST be read from environment variables.
- Configuration MUST be loaded and validated once at startup; missing or invalid required
  values MUST fail fast with a clear message naming the variable.
- Secrets MUST NOT be hard-coded, committed, or written to logs. A `.env.example` file with
  every variable and a short description MUST be maintained; real `.env` files MUST be
  git-ignored.

**Rationale**: Enables safe deployment via cron, systemd, or Docker without code changes.

### V. Observable State Transitions

- Every job MUST log each state transition with its job id/url, e.g.:
  `DISCOVERED → SKIPPED_ALREADY_PROCESSED`, `DISCOVERED → EVALUATING → RELEVANT/NOT_RELEVANT`,
  `RELEVANT → NOTIFIED → PROCESSED`, or `→ FAILED (reason, will retry)`.
- Each run MUST log a start line and a summary line (counts of discovered, skipped, evaluated,
  relevant, notified, failed) so background activity is monitorable at a glance.
- Logs MUST go to standard output/standard error with timestamps and levels
  (`INFO` for transitions, `WARNING` for retriable failures, `ERROR` for fatal ones), so cron,
  systemd journal, and Docker capture them without extra tooling.

**Rationale**: The agent runs unattended; logs are the only window into its behaviour.

### VI. Zero-Cost Operation

- The default LLM provider MUST be one with a usable free tier; the provider MUST be
  swappable through configuration (Principle I and IV).
- Payloads sent to the LLM MUST be minimized: strip HTML, boilerplate, and irrelevant
  fields; truncate descriptions to a configured maximum length; keep the system prompt terse.
- Already-processed jobs MUST be filtered out **before** any LLM call, and before fetching
  per-job detail pages when the listing already provides the job id/url.
- State MUST be stored in a local, embedded database (e.g., SQLite); no paid or managed
  cloud services are required to run the agent.
- Polling MUST be conservative: one run per scheduled invocation, configurable interval,
  no tight loops, and conditional requests or caching where the source supports it.
- Dependencies MUST be kept minimal and lightweight.

**Rationale**: The agent is meant to run indefinitely for free on a personal machine or a
small server.

## Pipeline Contract

Every run executes the following stages in order, per job:

1. **Scraper** — fetch job listings from each configured target URL and normalize them into
   domain job listings with a stable id (source id or canonical url).
2. **State check** — if the job id or url already exists in the local database as processed,
   log `SKIPPED_ALREADY_PROCESSED` and stop for this job.
3. **Evaluator** — send the minimized job description to the LLM with a strict system prompt.
   The response MUST be exactly this JSON shape:
   `{ "relevant": bool, "reason": string, "salary_match": bool }`.
   Extra prose, missing fields, or wrong types are a failure (Principle III).
4. **Notifier** — if `relevant == true`, format a concise message and HTTP POST it to the
   Discord Webhook URL. Non-2xx responses are failures.
5. **Save state** — mark the job as processed (storing the evaluation result) only after
   stages 3 and 4 succeeded. Jobs evaluated as not relevant are also marked processed.

Changes to this contract (stage order, JSON schema, or the "save last" rule) require a
constitution amendment.

## Documentation & Development Workflow

- A `README.md` MUST be kept current and concise, covering: purpose, environment variable
  setup (referencing `.env.example`), install/build instructions, running locally, and simple
  deployment strategies — cron job, systemd service/timer, and Docker.
- Use cases and adapters MUST have automated tests; external systems (LLM, Discord, job sites)
  MUST be faked in tests, never called for real. Tests MUST cover the failure paths of
  Principle III (timeout, invalid JSON, webhook failure → job not marked processed).
- Every feature plan MUST include a Constitution Check against Principles I–VI; any
  violation MUST be justified in the plan's complexity tracking section.

## Governance

- This constitution supersedes other practices and conventions in this repository. Specs,
  plans, tasks, and code reviews MUST verify compliance with it.
- Amendments are made by editing this file (via `/speckit-constitution`), with a Sync Impact
  Report describing the change, and committed with a message stating the new version.
- Versioning follows semantic versioning: MAJOR for removing or redefining a principle or the
  Pipeline Contract; MINOR for adding a principle/section or materially expanding guidance;
  PATCH for clarifications and wording.
- Added complexity (new layers, services, paid dependencies) MUST be justified against
  Principles I and VI before adoption.

**Version**: 1.0.0 | **Ratified**: 2026-11-06 | **Last Amended**: 2026-11-06
