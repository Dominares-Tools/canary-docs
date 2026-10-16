# Changelog

## 1.2.1

Fixes from a review of 1.2.0. A 1.2.0 config behaves as before.

- `canary deployed`: a sha that is not 7 or more hex characters (`unknown`, `dev`, `main`, `v1.2.3`,
  `abc12`) now reads `unknown` (exit 2) instead of `stale` (exit 1). Real abbreviations and uppercase
  shas are still judged.
- Worker: a webhook event carries `hold_reason`, set while a deep check is holding a recovery open and
  `null` otherwise. The event's `reason` is unchanged.
- The User-Agent of the Python checks and of the Worker now carries the package version.
- The README now lists `deepcheck` and `deployed` among the files, and says a Twilio call that failed
  is not retried until the next reminder.

## 1.2.0

Escalation for the Worker, and the calendar for the Python checks. Everything new is off until you
turn it on; a 1.1 config behaves as before.

- Worker, reminders: `CANARY_REPEAT_EVERY_MIN` says an open incident again as a **STILL DOWN**
  message every N minutes (a new `repeat` action and webhook `event`). A reminder nobody took is
  offered again on the next run.
- Worker, relay first: with `CANARY_RELAY_URL` and `CANARY_RELAY_SECRET`, every alert is POSTed to your
  relay (`{kind, title, body}`) before anything else. Only HTTP 200 with `delivered: true` counts;
  anything else sends the email / webhook as before, and a body cut short to fit the limit goes by
  email too, whole.
- Worker, phone calls (Twilio): a **down** and each **reminder** also ring `CANARY_CALL_TO`, once per
  `CANARY_CALL_COALESCE_MIN` across every target, in parallel with the message. A failing Twilio never
  blocks the message; a failing message never silences the call, and a retried alert does not ring
  twice. No call for a setting to fix (a refused deep-check secret, a broken config). A target's
  `spoken` key says what it is called aloud. `CANARY_CALL_DRY_RUN=1` dials nothing.
- Worker, honest recovery: a chat target can name `hold_recovery_on: "<deep target>"`. It is not called
  recovered while that deep check fails or reports a fallback vendor (`failover.active`): one
  **DEGRADED** alert, then reminders, then **RECOVERED**. A deep check that cannot run never holds a
  recovery open; the recovered alert says it is unconfirmed.
- Worker: `CANARY_ALERT_LABEL` prefixes every alert for a drill. A deep target that gets no answer is
  an endpoint problem ("unreachable"), not a vendor failure.
- Probe keys `calendar`, `open_days` and `about`: a wrong weekday or date for tomorrow, the day after,
  or a written date (`wrong_day`), and an open-or-closed verdict that the weekdays you are open
  contradict (`wrong_hours`). The verdict is read only from what the reply says about the day asked
  about: "closed on Sunday, open again on Monday" is true on a Saturday asked about Monday. New module
  `canary.days`, a ready config `examples/hours-bot.json`, and the demo bot answers "are you open
  tomorrow?" from the clock.
- New `never_say` built-in `claims_not_to_know_hours`: "I don't have our closing time" from a bot that
  is told its hours.
- Problem kind `wrong_hours`; `canary check` lists the new checks.

## 1.1.0

Two additions that answer "is it really working?" from outside: the AI path, and the deploy.

- `canary.deep`: the target-side half of a deep check. A protected "prove your AI path works"
  endpoint with the shared-secret check (`secret_ok`, constant-time, header or Bearer, refuses when
  no secret is configured), a single-flight cache (`SingleFlightCache`, `AsyncSingleFlightCache`:
  one real vendor call per 60 seconds, failures cached too), failure classes (`classify_failure`:
  `billing`, `auth`, `model_missing`, `quota`, `timeout`, `empty`, `other`), `run_check` (a deadline
  around your check), `handle` (the whole endpoint, framework-free) and `wsgi_app`.
- `canary deepcheck URL --secret-env NAME`: call such an endpoint and judge it. Exit 0 the AI path
  works, 1 it is failing (the line says why), 2 the check could not run.
- Worker: a `deep` target kind that reads the same contract every run, with two strikes and one
  down / one recovered alert like any target; the alert names the cause in plain words.
- `canary deployed URL... --expect-sha SHA [--wait SECONDS] [--sha-path build.sha]`: ask each
  service's health JSON which commit it runs. Four outcomes (`deployed`, `stale`, `unknown`,
  `unreachable`); `unknown` is never folded into either `stale` or `deployed`. Exit 0 all
  deployed, 1 any proven stale, 2 none stale but one unreadable. A short sha matches a full one
  (7 characters or more, either way round). `--wait` keeps checking against the real clock.
- `canary demo` can also serve `GET /health` (`--build-sha`) and `GET /deep-check` (`--deep-secret`,
  `--deep-mode`), so both commands can be tried offline.
- README opens with a one-sentence description and a three-line example that runs.
- The test suite binds its local servers only in ports 8600 to 8609.

## 1.0.2

Documentation and packaging only. The code is unchanged.

- The README now opens with a one-line description and a three-line example you can paste,
  and the licence and access notes moved to the end.
- The package version (`pip show canary`, `canary.__version__`) now matches the release tag.
  Release 1.0.1 reported 1.0.0.

## 1.0.1

Licence and contact details only. The code is unchanged.

- Licence text updated to version 1.3.
- One public contact address in the licence and the README.

## 1.0.0 (2026-09-27)

First release.

- `canary run CONFIG`: ask a live chatbot every probe's question and grade the replies. Exit 0
  every answer was honest, 1 a probe failed, 2 the run could not check. A readable report names
  the probe that failed and why; `--json` prints the same as JSON.
- Configs in JSON, or TOML on Python 3.11+. `${NAME}` / `${NAME:-fallback}` environment variables;
  `{question}` / `{session_id}` in the URL, headers and body template. Unknown keys are errors.
- Replies read from Server-Sent Events (typed events, replace events, OpenAI-style deltas), a JSON
  path, or plain text. Several named endpoints, setup requests, per-run or per-probe sessions.
- Checks: failure markers, `must_contain` (all / any, strings or regexes), `never_say` patterns
  that understand denials, the built-in `chat_not_saved`, `cancel_is_immediate` and
  `placeholder_domain` checks, and today's weekday and date in a configured time zone (a
  condition such as "if today is Saturday" is not judged as a claim).
- The whole answer must arrive within `timeout`, streamed or not; redirects are not followed.
- `canary check CONFIG` validates a config without asking anything; `canary demo` serves a
  stand-in support bot (honest, lie, broken, down) to try it against.
- `worker/`: a Cloudflare Worker (Cron Trigger + SQLite-backed Durable Object) that probes health
  URLs, JSON health documents and a chat question every 5 minutes, and alerts once per incident
  by email (Resend) and/or a webhook signed with HMAC-SHA256. `canary.webhook.verify` checks the
  signature. A broken `CANARY_TARGETS` (including an unknown key) is its own `config` incident
  that closes when fixed and opens again if it breaks again.
- `examples/github-action.yml`: the full canary on a GitHub Actions schedule (run under
  `shell: bash`, so a failed canary fails the run), with a webhook alert on failure.
- `examples/support-bot.json` / `.toml`: a neutral support-bot config with exact price checks and
  no environment fallbacks.
