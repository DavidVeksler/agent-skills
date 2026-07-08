# agent-skills

[![skill-lint](https://github.com/DavidVeksler/agent-skills/actions/workflows/skill-lint.yml/badge.svg)](https://github.com/DavidVeksler/agent-skills/actions/workflows/skill-lint.yml)

A small collection of AI agent skills I actually use, published as plain Markdown.

A "skill" here is one `SKILL.md` file: a `name` + `description` header the router matches against, and a body of instructions the agent follows once it's invoked. That's the whole format. No SDK, no framework import, no vendor API call baked into the skill itself — just text an LLM reads before acting. Written for [Claude Code / Cowork skills](https://docs.claude.com/en/docs/claude-code/skills) because that's what I run day to day, but there is nothing Claude-specific in the instructions: drop a `SKILL.md` into any agent harness that can inject a file into context (a system prompt, a Cursor rule, a LangChain tool description, an OpenAI custom-GPT instruction block) and it works the same way.

## Skills

| Skill | What it does |
|---|---|
| [`cloudflare-stats`](skills/cloudflare-stats/SKILL.md) | Queries Cloudflare's GraphQL Analytics API for traffic on a zone you own — path-level stats for the last 7 days, zone-wide 30-day trends, top countries — without opening the dashboard. |
| [`ntfy`](skills/ntfy/SKILL.md) | Push a notification to your phone/desktop via [ntfy.sh](https://ntfy.sh) when a long-running or background task finishes, fails, or needs input — one `curl` call, no SDK. |

## Use

**Claude Code / Cowork:** copy a skill's folder into `.claude/skills/`.

**Anything else:** paste the body of `SKILL.md` (skip the frontmatter) into your system prompt, tool description, or rules file. The `description` field is written as a router-matching trigger list, so trim it to whatever your framework uses to decide when to invoke the skill.

Each skill documents its own required environment variables and has no dependency on the others.

## CI

Skills here are linted with [skill-lint](https://github.com/DavidVeksler/skill-lint), a deterministic checker for secrets, PII, missing human-in-the-loop gates on external actions, and status-language inflation. It runs on every push (see [`.github/workflows/skill-lint.yml`](.github/workflows/skill-lint.yml)). It's a report, not a verdict — a human still reviews every skill.

## Contributing

PRs adding a new skill are welcome. Keep it to one `SKILL.md` per skill, generic enough that it doesn't assume my machine, my accounts, or my employer. `skill-lint` will flag anything that looks like a secret, PII, or an unguarded external send.

## License

MIT.
