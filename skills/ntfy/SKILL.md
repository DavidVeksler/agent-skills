---
name: ntfy
description: |
  Send push notifications to the user's phone/desktop via ntfy.sh whenever the
  agent needs to surface something asynchronously: a long-running task finished,
  a build/test pass or fail, a deploy completed, an unrecoverable error, a
  decision is needed before the agent can proceed, or progress milestones on
  work the user isn't actively watching. Triggers on requests like "ping me
  when this is done", "notify me on failure", "send a push when X completes",
  or proactively when the agent is about to block on user input after a long
  silence. Uses HTTP POST to ntfy.sh — no auth, no SDK, just curl.
  Use this skill when the user asks to be pinged or notified asynchronously
  about work they are not actively watching. Do NOT use this for routine inline progress
  the user is reading in real time, for per-tool-call logging, or for any
  message containing secrets, credentials, or PII (ntfy topics are public by
  default).
---

# ntfy — push notifications for the user

## When to use this skill

Invoke this skill when **the user will not be watching the terminal** and
something material happens. Concretely:

- A long-running task (>60s) completes — successfully or not
- The agent is about to block waiting for input after a quiet stretch
- A test suite, build, deploy, migration, or CI job changes state
- An unrecoverable error needs human attention
- A scheduled or background job hits a milestone the user asked to be told about

Do NOT use for:

- Routine inline progress the user is reading in real time
- Every tool call (this is push-to-phone, not a log)
- Anything containing secrets, credentials, or PII — **ntfy topics are public**
  by default and anyone who guesses the topic name can read them

## Configuration

Read the topic from the `NTFY_TOPIC` environment variable. If it is unset, ask
the user once for a topic name and then store it for the session; do not pick
one yourself.

Optional environment variables, in priority order:

- `NTFY_SERVER` — base URL (default: `https://ntfy.sh`). Use this for
  self-hosted instances. Strip any trailing slash.
- `NTFY_TOKEN` — bearer token for authenticated topics. Send as
  `Authorization: Bearer $NTFY_TOKEN`.
- `NTFY_DEFAULT_PRIORITY` — `1`–`5`, default `3`.

If `NTFY_TOPIC` is missing AND the user hasn't asked to be notified, do not
send anything — just continue the task.

## How to send

One curl call. Always include `--silent --show-error --max-time 10` so a flaky
network never blocks the agent.

```bash
curl --silent --show-error --max-time 10 \
  -H "Title: ${TITLE}" \
  -H "Priority: ${PRIORITY:-3}" \
  -H "Tags: ${TAGS}" \
  ${CLICK_URL:+-H "Click: ${CLICK_URL}"} \
  ${NTFY_TOKEN:+-H "Authorization: Bearer ${NTFY_TOKEN}"} \
  -d "${MESSAGE}" \
  "${NTFY_SERVER:-https://ntfy.sh}/${NTFY_TOPIC}"
```

**Never fail the parent task because ntfy failed.** Wrap the call so a non-zero
exit is logged but swallowed:

```bash
notify() { curl ... || echo "ntfy delivery failed (ignored): $?" >&2; }
```

## Message conventions

Keep the title under 40 chars, the body under 200. Lead with state, not story.

| State | Priority | Tags (emoji shortcuts) |
|---|---|---|
| Success / done | 3 | `white_check_mark` |
| Needs input / blocked | 4 | `question` |
| Warning / partial failure | 4 | `warning` |
| Hard failure / error | 5 | `rotating_light` |
| FYI / progress milestone | 2 | `information_source` |

Examples (each is one `-d` body, separate from `Title:`):

- Title `✅ deploy prod` / body `web-api v2.41.0 live in 4m12s. 0 errors.`
- Title `❓ needs input` / body `Migration 0042 will drop column users.legacy_id. Confirm?`
- Title `🚨 build failed` / body `main branch: 3 test failures in PaymentService. See logs.`

If the project name is available (git repo, working directory basename),
prefix the title with it: `[your-project] ✅ tests passed`.

## Click-through

When the notification refers to a URL the user can open (a PR, a CI run, a log
in the cloud console), set the `Click:` header so tapping the notification
opens it. Skip it for purely local events.

## Self-check before sending

Before each call, verify:

1. The body contains no secrets, tokens, API keys, or customer PII.
2. The topic is set and not a placeholder like `your-topic-here`.
3. The message is actionable on a phone screen — a glance tells the user
   whether they need to come back to the terminal.

## Examples

**Long-running test suite**

```
Title: [your-project] ✅ tests passed
Priority: 3
Tags: white_check_mark
Body: 1,247 tests, 8m33s. Coverage 87.2% (+0.4%).
```

**Agent is about to block**

```
Title: [your-project] ❓ needs input
Priority: 4
Tags: question
Click: https://github.com/your-org/your-project/pull/812
Body: PR #812 ready for review. Should I merge or wait?
```

**Hard failure**

```
Title: [your-project] 🚨 deploy failed
Priority: 5
Tags: rotating_light
Body: Rollback complete. Cause: migration 0042 timeout on prod replica.
```

## Subscribing (one-time setup, for reference only)

The user subscribes by installing the ntfy app (iOS/Android/web) and adding
the same topic name. Topics are created on first publish; no registration.
For private use, the user should pick a long random topic name (treat it like
a password) or run a self-hosted ntfy with auth and set `NTFY_TOKEN`.
