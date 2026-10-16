# canary — Know when your chatbot's answers go wrong

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/canary**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

canary asks your live chatbot the questions that matter, on a schedule. It checks each reply for the facts it must contain and the claims it must never make, and names the question that failed and why. It can also prove your AI path works end to end and check that a deploy landed.

## Get it

Buy canary at https://dominares.org/tools/canary: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the canary command, the deep-check endpoint kit, the Cloudflare Worker and a GitHub Actions workflow.

Requirements: The Python standard library. The included Cloudflare Worker and GitHub Actions workflow are optional; phone calls need your own Twilio account.

This page documents version 1.2.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

**canary asks your live chatbot real questions, proves your AI path still answers and your deploy really landed, and tells you the moment a reply goes missing, breaks, or says something untrue.**

```bash
export CANARY_CHAT_URL=http://127.0.0.1:8765/chat CANARY_CHAT_TOKEN=demo; canary demo & sleep 1; canary run examples/support-bot.json; kill $!
```

Run from the root of your copy of the repository. The last line starts a stand-in support bot, asks it the
five questions in `examples/support-bot.json`, prints `All 5 probes passed.` and exits 0, then stops the bot.

### What it does

A chatbot can be completely broken while every dashboard is green. A retired model id, a tool call
the code throws away, a knowledge base that quotes last year's price, prepaid AI credits that ran out:
each of these returns HTTP 200 with a friendly-sounding answer, or leaves `/health` green. Your tests
pass (none of them call the live model), `/health` says `ok` (it checks the database), and the first
person to notice is a customer.

canary does what that customer does. On a schedule, it asks your live bot the questions that matter
and grades every reply:

- **It answered.** Not an HTTP error, not a timeout, not an empty stream.
- **It is not your app's own failure line** ("Sorry, I hit a temporary glitch"), which usually
  arrives as HTTP 200.
- **It contains the facts it must**: the price, the plan name, the feature.
- **It says nothing you know is false**, and it understands denials: "chats are not stored" is not
  a claim that they are. Built-in checks catch a bot saying the chat is not saved, that cancelling
  takes effect at once, or citing `example.com` as a real site.
- **What it says about today is true**: "since today is Friday we're open until 6pm" on a Saturday
  fails. So do a wrong "tomorrow is Monday", and a bot that says it is closed on a day it is open.

It exits `0` when every answer is honest, `1` when a probe failed, `2` when it could not check, and
prints a report that names the probe that failed and why:

```
  FAIL  cancel         [default] How do I cancel?
        untrue: said that cancelling takes effect at once: 'Once you cancel, billing stops and your workspace is deactivated'
        why this probe exists: cancelling takes effect at the end of the paid period
        reply: Once you cancel, billing stops and your workspace is deactivated right away.
```

Beyond the questions, it has three more parts:

- **A deep-check endpoint for your service** (`canary.deep`) and a way to call it
  (`canary deepcheck`, the Worker's `deep` target): a protected route that makes one tiny real call
  to your AI vendor and says whether it worked, and why not. See
  [Prove the AI path works](#prove-the-ai-path-works).
- **`canary deployed`**: asks each service's health JSON which commit it is running and compares it
  with the commit you pushed, so a deploy that never landed is a red exit code, not a surprise. See
  [Prove the deploy landed](#prove-the-deploy-landed).
- A **Cloudflare Worker** (`worker/`) that watches your health URLs, deep checks and one chat question every
  5 minutes from outside your infrastructure and alerts once when something goes down and once when
  it recovers (email through Resend, or a signed webhook). It can also repeat the alert while an
  incident stays open, hand alerts to a relay of yours first with email as the fallback, ring a phone,
  and refuse to call a chat bot "recovered" while the AI path behind it still fails. See
  `worker/README.md`. And a **GitHub Actions workflow** (`examples/github-action.yml`) that runs the
  full canary on a schedule.

**Who it is for:** anyone running a customer-facing chatbot (support, sales, onboarding, a RAG app
over your docs) who wants to hear about a broken or lying bot before a customer does. It works
with any HTTP chat endpoint: your own backend, an OpenAI-compatible API, a streaming widget
endpoint.

The runtime is the Python standard library only (3.10 or newer). No SDK, no account, no agent.

### Install

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

Installing it adds the `canary` command. On Windows or a slim container with no time zone database, add
the `tz` extra if a probe checks the day in a zone other than UTC: `pip install '.[tz]'`.

### Try it, then break it

canary ships a stand-in bot, so you can see it work before pointing it at yours.

```bash
export CANARY_CHAT_URL=http://127.0.0.1:8765/chat CANARY_CHAT_TOKEN=demo   # the example reads both
canary demo &                                        # "Acme Notes" support bot on 127.0.0.1:8765
canary run examples/support-bot.json                 # 5 probes, all pass, exit 0
```

Now make the bot misbehave and run it again:

```bash
kill %1; canary demo --mode lie &                    # it lies about saving chats, cancelling and the day
canary run examples/support-bot.json                 # 3 probes FAIL, each named, exit 1
kill %1; canary demo --mode broken &                 # every answer is "a temporary glitch"
canary run examples/support-bot.json --json          # the same verdicts as JSON
kill %1
```

To point it at your bot: copy `examples/support-bot.json`, change the `endpoint` to your chat URL
and request shape, and write the questions your bot must get right. `canary check your.json`
validates it without asking anything. The example has no fallback for either variable on purpose:
unset, it is a config error (exit 2), never a quiet probe of the wrong URL.

### The config

JSON, or TOML on Python 3.11+ (`examples/support-bot.toml` is the same config). Every key is
checked; a typo is an error that names it rather than a check that is quietly switched off.

```json
{
  "name": "Acme Notes support bot",
  "timezone": "UTC",
  "endpoint": {
    "url": "${CANARY_CHAT_URL}",
    "method": "POST",
    "headers": {"Authorization": "Bearer ${CANARY_CHAT_TOKEN}"},
    "body": {"messages": [{"role": "user", "content": "{question}"}], "session_id": "{session_id}"},
    "response": {"mode": "sse", "type_field": "type", "text_types": ["text"], "path": "text"},
    "session_prefix": "canary-",
    "timeout": 60
  },
  "failure_markers": ["temporary glitch", "something went wrong"],
  "never_say": [{"builtin": "chat_not_saved"}, {"builtin": "placeholder_domain"}],
  "probes": [
    {"name": "starter-price", "question": "How much is the Starter plan per month?",
     "must_contain": {"regex": "\\$12(?!\\d|[.,]\\d)", "label": "$12"}},
    {"name": "cancel", "question": "How do I cancel?", "never_say": [{"builtin": "cancel_is_immediate"}]},
    {"name": "today", "question": "What day of the week is it today?", "today": true}
  ]
}
```

#### Top level

| Key | Meaning |
|---|---|
| `name` | Shown in the report. Default: the file name. |
| `endpoint` | The chat to ask (below). Or `endpoints`: several, by name, e.g. `{"sales": {...}, "support": {...}}`. Give exactly one of the two. |
| `probes` | The questions. At least one. |
| `failure_markers` | Your app's own "I failed" lines, matched case-insensitively in every reply. Any of them fails the probe, whatever the status code. |
| `never_say` | Claims no reply may make, checked on EVERY reply (see below). |
| `timezone` | The zone "today" is judged in (`UTC` by default; an IANA name like `America/New_York`). |

#### `endpoint`

| Key | Meaning |
|---|---|
| `url` | Required. `{question}` and `{session_id}` are filled in URL-encoded, for a GET API. |
| `method` | Default `POST`. |
| `headers` | Header: value. Values may use `{question}` / `{session_id}`. |
| `body` | A JSON template (object, list or string). `{question}` and `{session_id}` are filled into its strings BEFORE it is serialized, so a question with quotes or newlines stays valid JSON. An object or list is sent as JSON (`Content-Type: application/json` unless you set one). |
| `response` | How to read the reply: `"sse"`, `"json"`, `"text"`, or an object (below). Required. |
| `session_prefix` | Default `canary-`. Every session id canary sends starts with it, so your app can leave canary's chats out of its visitor numbers. |
| `session_scope` | `"run"` (default): one session for the whole run. `"probe"`: a new session (and setup) for every probe, so no answer can lean on an earlier one. |
| `setup` | Requests to make before asking, with the same session id: a session start, an intake form. Each takes `url`, `method`, `headers`, `body`, and `expect_status` (a status or a list; default any 2xx). A failed setup means the run could not check: exit 2. |
| `timeout` | Seconds for the whole answer, streamed or not. Default 60. |

**Environment variables:** any string in an endpoint may use `${NAME}`, or `${NAME:-fallback}`. An
unset or empty variable with no fallback is a config error that names it (exit 2): a blank token
would only turn into a misleading "the bot is down".

#### `response`

| `mode` | Reads | Keys |
|---|---|---|
| `sse` | A Server-Sent Events stream. Each event's `data:` lines are joined and read as JSON when they parse; the text at `path` in each event is appended. A data line that is not JSON is taken as text; `[DONE]` is ignored. | `path` (default `text`; an OpenAI-style stream: `choices.0.delta.content`), `type_field` + `text_types` (only events of those types add text), `replace_types` (an event that REPLACES everything so far, for servers that rewrite text they already streamed) |
| `json` | One JSON document. | `path` (required): `reply`, `choices.0.message.content`, or `$` for the whole document |
| `text` | The body as it is. | |

Paths are dotted, with list indexes (`choices.0.message.content`, `-1` for the last item).

#### `probes`

| Key | Meaning |
|---|---|
| `question` | Required. What to ask. |
| `name` | Shown in the report and used by `--probe`. Default: the question. Must be unique. |
| `endpoint` | Which endpoint, when there are several. |
| `must_contain` | A string, a `{"regex": "...", "label": "..."}` object (`regex` is a regular expression; `label` names it in the report), or a list of them. Strings match case-insensitively (apostrophes straightened) as substrings, so use a regex for a number: `"$12"` is also in "$120" and "$12.99", while `\$12(?!\d\|[.,]\d)` is not. |
| `match` | `"all"` (default): every item must be there. `"any"`: one is enough (`["$12", "twelve dollars"]`). |
| `case_sensitive` | For `must_contain`. Default false. |
| `never_say` | Claims this probe's reply must not make, on top of the global ones. |
| `today` | `true`: a weekday or date the reply gives for today must be right (see below). |
| `calendar` | `true`: a weekday or date the reply gives for tomorrow, the day after, or a written date ("Friday, October 3") must be right (see below). |
| `open_days` | The weekdays you are really open, e.g. `["mon", "tue", "wed", "thu", "fri"]`: a reply that says it is open or closed on the wrong one fails (see below). Also turns on `calendar`. |
| `about` | With `open_days`: the day the question is about, `"today"`, `"tomorrow"` (default) or `"day_after_tomorrow"`. |
| `why` | Why this probe exists. Printed when it fails, so whoever reads the alert knows what is at stake. |

A probe fails on the first of: no answer (`unreachable`), a non-2xx status (`http_status`), no text
(`empty`), a failure line (`failure_marker`). Otherwise every other problem is listed together:
each untrue claim (`untrue`), missing facts (`missing`), a wrong day (`wrong_day`), a wrong
open-or-closed verdict (`wrong_hours`).

#### `never_say`: claims, and denials of them

Each entry is a regular expression (a string), or an object:

```json
{"label": "that there is phone support", "pattern": "\\b(?:call us|phone support)\\b"}
```

| Key | Meaning |
|---|---|
| `pattern` | A Python regular expression, case-insensitive unless `case_sensitive` is true. |
| `label` | What the reply claimed, in words, for the report. Default: the pattern. |
| `unless_denied` | Default `true`: a match whose own clause denies it does not count. |
| `builtin` | Instead of `pattern`: one of the built-in checks below. |

**Denials.** A model never words things the same way twice, and an honest reply often names the
very thing it denies. So by default a match only counts when the clause leading up to it (from the
last comma, semicolon, colon, dash, bracket, or joining word such as "and", "but", "because", inside
one sentence) contains no denial ("not", "n't", "no", "never", "nothing", "without"):

| Pattern | Reply | Verdict |
|---|---|---|
| `stored` | "Chats are stored for 30 days." | a claim: fails |
| `stored` | "Chats are not stored." | denied: passes |
| `phone support` | "No problem! Our phone support line is open." | a claim: "No problem!" is its own sentence |
| `refund page` | "Why not check our refund page?" | a claim: "why not" is an invitation |

Set `"unless_denied": false` for a phrase that must never appear at all, denied or not.

**Built-in checks** (`{"builtin": "<name>"}`), for claims plain patterns get wrong:

| Name | Catches | Lets through |
|---|---|---|
| `chat_not_saved` | "this conversation isn't stored", "we don't record anything you type", "nothing you type here is saved", "we keep no record of this chat", "this is a private chat" | "we never store full card numbers", "audio isn't recorded", "it won't be kept private from our team", "this is not a private conversation" |
| `cancel_is_immediate` | "once you cancel, billing stops and your workspace is deactivated", "cancelling ends your subscription immediately", "cancellation is immediate", with no paid period mentioned | "billing stops at the end of the period you've paid for", "at your next renewal", "you keep access until then", "you can cancel immediately from Settings" |
| `placeholder_domain` | `example.com` / `.org` / `.net` named as a site (a template placeholder leaking into answers) | an email address at example.com (a visitor's address repeated back) |
| `claims_not_to_know_hours` | "I don't have our closing time", "I do not have the hours", "not sure of our opening hours" | "We close at 6pm today", "I don't have your order number" |

Use `chat_not_saved` when your chats are in fact saved, and `cancel_is_immediate` on a cancel
question when cancelling takes effect at the end of the billing period. Use `claims_not_to_know_hours`
on a probe about opening hours when the bot is told them.

#### `today`: the weekday and the date

With `"today": true`, a reply's claims about today are checked against the real date in `timezone`:

- weekday: "today is Friday", "today's a Friday", "it's Friday today", "happy Friday"
- date: "today is [Friday,] September 26[, 2026]", "today's date is 26 September 2026", "the date
  today is 2026-09-26"

A reply that names no weekday or date passes, and so does a condition: "if today is Saturday,
we're closed" quotes the hours without saying what today is. "Today" is taken at both ends of the
request, so a reply written across midnight is never called wrong for it.

#### `calendar` and `open_days`: tomorrow, the day after, and the hours

A bot that works out "the day after tomorrow is Sunday" on a Saturday, or says it is closed on a day
it is open, sends people to a closed door as surely as a wrong "today". Two more probe keys check it
against the real date in `timezone`:

- `"calendar": true`: a weekday or date claim about tomorrow or the day after must be right ("tomorrow
  is Monday", "the day after tomorrow (Sunday)", "tomorrow falls on the weekend"), and so must a
  written date ("Friday, October 3" when the 3rd is a Saturday; a date more than half a year away is
  not judged). Wrong claims fail as `wrong_day`.
- `"open_days": ["mon", "tue", "wed", "thu", "fri"]` with `"about": "tomorrow"`: the reply's open or
  closed verdict about that day is checked against the days you are open. It fails as `wrong_hours`
  ("said closed on Monday, a day you are open"), and `calendar` is checked too.

The verdict is read only from what the reply says about THAT day. On a Saturday asked about Monday,
"We are closed on Sunday, October 4, and will be open again on Monday, October 5" is true: the
"closed" is about Sunday. A clause that names the day and no other is judged on its own, so "No,
we'll be closed the day after tomorrow. We reopen Monday." still fails; a yes or a no answers the
question whatever other day the reply names; "closed until Tuesday" is closed on every day before
Tuesday. Only "we're open" or "we'll be open" says a closed day is open: "we're not usually open on
Saturdays", "Saturday isn't one of our open days" and "we reopen Monday" are all true on a Thursday.

As with `today`, a condition is not a claim ("if tomorrow is Monday, we open at 9"), and the date is
taken at both ends of the request. `examples/hours-bot.json` is a ready config for the demo bot:
`canary run examples/hours-bot.json`.

### Running it

```
canary run CONFIG [--json] [--probe NAME ...] [--timeout SECONDS]
canary check CONFIG
canary demo [--mode honest|lie|broken|down] [--host 127.0.0.1] [--port 8765]
            [--deep-secret SECRET [--deep-mode ok|billing|auth|other]] [--build-sha SHA]
canary deepcheck URL --secret-env NAME [--method GET|POST] [--timeout SECONDS] [--json]
canary deployed URL [URL ...] --expect-sha SHA [--wait SECONDS] [--sha-path build.sha] [--json]
canary --version
```

| Exit | `run` means |
|---|---|
| 0 | Every probe passed. |
| 1 | At least one probe failed: no answer, the app's failure line, a missing fact, an untrue claim, a wrong day. |
| 2 | Could not check: a bad config, a missing environment variable, a failed setup request, an unknown `--probe`. |

Treat 2 as an alarm too: a canary that cannot run tells you nothing, and silence is what it exists
to prevent.

`--json` prints the whole report, also on exit 2:

```json
{"config": "Acme Notes support bot", "ok": false, "exit_code": 1, "error": null,
 "started_at": "...", "finished_at": "...",
 "probes": [{"name": "cancel", "question": "How do I cancel?", "endpoint": "default",
             "session_id": "canary-3f9a1c0b2d4e", "http_status": 200, "reply": "...", "ms": 812,
             "why": "...", "ok": false,
             "problems": [{"kind": "untrue", "detail": "said that cancelling takes effect at once: '...'"}]}]}
```

Problem kinds: `unreachable`, `http_status`, `empty`, `failure_marker`, `untrue`, `missing`,
`wrong_day`, `wrong_hours`.

### Prove the AI path works

`/health` never asks the model anything. When your AI vendor's prepaid credits run out, every call
returns HTTP 402 and `/health` stays green for hours. The fix is a second endpoint that makes ONE
tiny real call to the vendor and reports what happened, plus a monitor that calls it.

**On your service** (`canary.deep`, standard library only, no web framework needed):

```python
import os

from canary import deep

def check():
    # one real, tiny call to your vendor; raise deep.VendorError(status, body) when it answers an error
    reply = my_vendor_client.generate("ping", max_tokens=1)
    return {"model": reply.model}

app = deep.wsgi_app(check, secret=os.environ["CANARY_SECRET"], path="/internal/deep-check")
```

`deep.wsgi_app` is a plain WSGI app (Flask, Django and `wsgiref` mount it). In FastAPI, aiohttp or
anything else, call the pieces yourself: `deep.handle(method, headers, check, secret=..., cache=...)`
returns `(status, document)`; `deep.secret_ok`, `deep.refuse`, `deep.classify_failure`,
`deep.SingleFlightCache` and `deep.AsyncSingleFlightCache` (for `async def` handlers) are each usable
alone.

**The contract** (what any monitor reads):

```
GET <your path>        X-Canary-Secret: <secret>    (or Authorization: Bearer <secret>)

200  {"ok": true,  "kind": null,      "model": "...", "http_status": 200, "latency_ms": 412,
      "checked_at": "2026-10-04T10:00:00+00:00", "cached": false}
200  {"ok": false, "kind": "billing", "model": "...", "http_status": 402, ...}
401  {"ok": false, "error": "unauthorized"}
```

- `ok` is the verdict. A failed check is still HTTP 200: the endpoint worked, and what it found is
  in the document.
- `kind` says why: `billing` (credits or billing), `auth` (a bad or revoked key), `model_missing`
  (a retired model id), `quota` (a rate limit), `timeout` (no answer within the deadline, 10 seconds
  by default), `empty` (it answered with nothing), `other`. `deep.classify_failure(status, body)` picks
  one from the vendor's HTTP status and body.
- Without the secret, a wrong one, or no secret configured on the service: 401 and nothing else, and
  your check is never run, so a stranger cannot cost you a vendor call.
- The answer is cached for 60 seconds, passed or failed, so a burst of callers (or one stuck in a
  loop) costs one vendor call. `cached` says when it was.
- Optional fields `provider` and `failover` (`{"active": true, "since": "..."}`) are passed through if
  your check returns them.

**Calling it:**

```bash
export DEEP_SECRET=...                       # never on the command line
canary deepcheck https://api.example.com/internal/deep-check --secret-env DEEP_SECRET
```

Exit `0`: the AI path works. Exit `1`: the endpoint says it is failing, and the line names why
(`FAIL  ...: the AI vendor refused: credits or billing (HTTP 402) on model-x`). Exit `2`: the check
could not run: the secret was refused, the route is not deployed, the service is down, or the answer
was not the contract. `--method POST` for a route that takes POST, `--json` for the verdict as JSON.
Try it offline: `canary demo --deep-secret s3cret --deep-mode billing` serves a `/deep-check` that
fails the way an out-of-credit vendor does.

**From the Worker**, a `deep` target (see `worker/README.md`) does the same every 5 minutes from
outside your infrastructure, with two strikes and one down / one recovered alert like any target.

### Prove the deploy landed

A push, a green build and a "deployed" notice prove nothing about what is serving traffic. Give
your services a health JSON that reports the commit they were built from (`{"build": {"sha": "..."}}`),
and ask them:

```bash
canary deployed web=https://app.example.com/health api=https://api.example.com/health \
    --expect-sha "$(git rev-parse HEAD)" --wait 300
```

`--wait SECONDS` keeps checking until every service reports the commit, or the time is up (real
elapsed time, slow requests included). Without it, it checks once. `--sha-path` says where the commit
is in the JSON (default `build.sha`, dotted: `meta.git.commit`). `--json` prints the result as JSON.

Each service ends in one of four states, and the difference is the point:

| State | Meaning |
|---|---|
| `deployed` | it reports the commit you expect: proven |
| `stale` | it reports a different commit: proven not there yet |
| `unknown` | it answers but reports no commit, or something that is not one (`unknown`, `dev`, `main`, fewer than 7 hex characters, a non-string): NOT proven wrong |
| `unreachable` | the request failed, or the answer was not JSON |

`unknown` is never folded into `stale` (that sends someone chasing a deploy that may well have
landed) or into `deployed` (a false success is what this exists to prevent).

| Exit | Meaning |
|---|---|
| 0 | every service runs the commit |
| 1 | at least one is `stale`: proven not there |
| 2 | none is stale, but one is `unknown` or `unreachable` (or the arguments were wrong: `--expect-sha` needs 7+ characters) |

Commits match when one is a prefix of the other, either way round, case-insensitively, and at
least 7 characters: the full 40-character sha a service reports matches the short one in
`git log --oneline`. `canary demo --build-sha abc1234def` serves a `/health` to try it on.

### Scheduling it

- **GitHub Actions**: `examples/github-action.yml` runs `canary run` twice an hour (as far as
  GitHub's scheduler allows; the file explains its throttling) and posts the report to a webhook
  when a run fails. A failed run also shows up the way any failed workflow does.
- **The Worker**: `worker/` is a Cloudflare Worker that probes your health URLs (and one chat
  question per target) every 5 minutes from outside your infrastructure, and alerts once per
  incident by email or signed webhook. Its README has the exact deploy steps.
- **Anything with cron**: `canary run canary.json || your-alert-command`.

### From Python

```python
from canary import load_config, run
from canary.report import render_text

report = run(load_config("canary.json"))
print(render_text(report))
assert report.ok, [(r.name, [p.detail for p in r.problems]) for r in report.results if not r.ok]
```

`run(config, transport=...)` takes any callable `(method, url, headers, body_bytes, timeout) ->
(status, text)`, so you can point the same probes at a test client. `canary.claims` (the claim
checks), `canary.truth.wrong_day`, `canary.webhook.verify` (for the Worker's signed alerts),
`canary.deep` (the service-side endpoint), `canary.deepcall.judge` and `canary.deployed.verify` are
usable on their own.

### Keeping canary out of your numbers

Every session id canary sends starts with `session_prefix` (`canary-` by default); setup requests
use the same id. Filter that prefix out of your analytics and "conversations waiting" lists, or
your monitor will show up as a busy visitor.

### Honest limits

- **Each probe is one real question to your bot**, billed by your model provider like any other.
  Five probes twice an hour is 240 questions a day.
- **Claim checks look for shapes, not meaning.** They are tuned to catch the wordings real bots use
  and to pass honest ones (the test suite pins both directions), but a sufficiently creative
  phrasing can slip past a pattern, and a denial that comes AFTER the claim ("chats are stored
  nowhere") is read as the claim. Write patterns for the false statements you actually fear, and
  add a probe when you see a new one.
- **Denials are English.** The negation words and clause breaks are English; patterns work in any
  language, but denial handling will not.
- **The hours check understands the shapes real bots use, not every sentence.** It is held to a table
  of seventy-six true and false replies in the test suite, in both directions, and to a month of
  generated ones, but a reply that is unusually worded can pass a wrong verdict or (rarely) fail a
  true one. It judges one day per probe, the weekdays you give it (not holidays), and English.
- **Numeric dates (9/10) are not judged**, because they mean different days in different countries.
- **One question per probe.** canary does not hold multi-turn conversations; a probe that needs
  context must carry it in its question (or in the body template).
- **It grades what comes back, not how.** A slow but correct answer passes if it has fully arrived
  within `timeout`. The deadline is checked between reads, so a stalled stream can run over it by
  at most one more `timeout` before it fails as `unreachable`.
- **A deep check costs one real vendor call per cache window** (60 seconds by default, one token
  with the usual one-token check): a monitor calling it every 5 minutes is about 288 a day.
- **`canary deployed` can only see what your service reports.** A service whose health JSON carries no
  commit is `unknown`, never `deployed`; it also reads the commit the process was started with, so
  it proves the new build is serving, not that every instance behind a load balancer is.
- **Redirects are not followed.** A 301 or 302 fails the probe as `http_status`, the same as in the
  Worker: point `url` at the address that answers.

### Files

```
canary/            the package (stdlib only)
  cli.py           canary run / check / demo / deepcheck / deployed
  config.py        loading and checking a config
  runner.py        asking and grading
  claims.py        never_say, denials and the built-in checks
  truth.py         today's weekday and date
  extract.py       sse / json / text replies
  report.py        the text and JSON reports
  webhook.py       verify the Worker's signed webhook alerts
  deep.py          the deep-check endpoint for your service (secret, cache, failure kinds, WSGI app)
  deepcall.py      call a deep-check endpoint and judge it (canary deepcheck)
  deployed.py      is each service running this commit? (canary deployed)
  days.py          tomorrow, the day after, written dates, and open-or-closed verdicts
  demo.py          the stand-in Acme Notes bot (also /health and /deep-check)
examples/          support-bot.json, support-bot.toml, hours-bot.json, github-action.yml
worker/            the Cloudflare Worker, its wrangler.jsonc template and README
tests/             the test suite (pytest; the Worker's tests run it under Node)
```

To run the tests, from inside the repository: `pip install -e '.[dev,tz]'`, then `pytest` (Node
20 or newer on the PATH for the Worker's tests).

---

Docs mirror of the private repository Dominares-Tools/canary; updated with each release.
