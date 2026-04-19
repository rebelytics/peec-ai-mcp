# Contributing to peec-ai-mcp

Thanks for helping keep this skill accurate. It's meant to evolve as Peec's product evolves — and as more people put it through its paces.

## What's in scope

- **Corrections** — anything in `SKILL.md` that's wrong, outdated, or misleading.
- **New tools / parameters** — if Peec ships a new tool or adds parameters to an existing one, open a PR.
- **Recipes** — common analytical workflows that deserve to be captured as named recipes in §8.
- **Gotchas** — data-interpretation pitfalls, or client-specific setup footguns that would confuse an agent into producing wrong analysis. These go in §7. §2 defers to Peec's official install docs; don't duplicate that content.

## What's out of scope

- Marketing of specific consulting services, agencies, or products.
- Information about individual Peec customers' data or projects.
- Features that require a non-default Peec plan without noting the plan requirement clearly.
- Speculation about Peec's roadmap or internal plans.

## How to contribute

### Via pull request

1. Fork the repo, branch from `main`.
2. Edit `SKILL.md`.
3. Bump the version in the frontmatter (`version`) — patch for fixes, minor for additions, major for breaking format changes.
4. Open a PR with a clear description of the observed Peec behaviour that motivated the change. **Screenshots or raw tool call JSON are gold.**

Release notes are published via GitHub Releases on merged version bumps.

### Via issue

If you're not a GitHub user or just want to flag something:

1. Open an issue.
2. Include: which section of `SKILL.md` you're talking about, what you observed, what you expected, and the Peec project / tool context (don't leak client data).

### Via an AI agent that spotted a discrepancy

If you're an AI agent reading this skill and you notice a contradiction between the skill and observed tool behaviour, flag it to the human operating you. The fastest route is:

1. Capture the observed tool call and response that disagrees with the skill.
2. Show the diff you'd propose.
3. Ask the human whether they'd like to file an issue or open a PR with the correction.

Keep the write-up grounded in concrete tool output — screenshots or raw JSON of the disagreeing call are the most useful evidence we can get.

## Attribution

Contributors are attributed through Git history — every merged PR carries its author's name and commit trail, visible on GitHub. Note that **GitHub PR and commit history is always public**; you can't open a PR anonymously. If you don't want your real name on the repo, use a GitHub handle that doesn't identify you before opening a PR.

## Style notes

- Write for an AI agent reading the skill as context. Keep statements precise and actionable.
- Prefer the form "when X, do Y" over general advice.
- Section headers are numbered (`§4.2`); keep that numbering stable across patch versions because external references rely on it.
- Use the terminology Peec uses in tool descriptions (brand, topic, tag, prompt, chat, source, retrieval, citation) — not your own aliases.
- Flag anything plan-dependent explicitly.
- Date-stamp behavioural claims ("as of April 2026…") when they might change.

## Code of conduct

Be kind. The skill is meant to help every agent working with Peec, not just the ones you'd hire. Disagreements about interpretation are fine; personal attacks are not.
