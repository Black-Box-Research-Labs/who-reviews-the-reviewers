**Point your coding agent at your repository and tell it: "run the governance audit in AUDIT.md."**

That's the whole thing. `AUDIT.md` is a self-contained instruction set any LLM coding agent (Claude Code, Cursor, Copilot, ChatGPT-with-tools) can execute against your current repo using only its existing `gh` auth and `git` — no installs, no API keys, no config. It runs read-only and prints a one-screen scorecard answering five governance questions: zero-review merge rate, maintainer PR review coverage, dropped security PRs, external-vs-internal merge latency, and bot-vs-human review deference — every number stamped with its date, window, and counting rule so you can re-run it yourself.

*Why:* most teams have never measured who actually reviews their code. This is the gift version of the audit we run for a living — point it at your own repo and find out before someone else does. ("self-audit" = the tool audits **whatever repo you point your agent at** — your repo, not ours.)

---

**The talk this came from.** *Who Reviews the Reviewers? A Live Forensic Audit of an AI Agent Framework*. BSides San Antonio, 13 June 2026, 41 min: https://www.youtube.com/watch?v=pe_jo6j9GsU

The talk opens on a CTO who opened a pull request adding six autonomous agents, allowed to rewrite their own source code, and merged it himself eleven seconds later with zero review. It then runs the forensic engine live on stage against a popular AI agent framework. The five questions in `AUDIT.md` are the talk's closing ask: the things a defender should be able to answer about their own repository on Monday morning.

---

*Who Reviews the Reviewers? — a repo-governance self-audit, by Black Box Research Labs.*
*Sibling tool: find-your-kill-zone (https://github.com/Black-Box-Research-Labs/find-your-kill-zone). Point your agent at your repo and find your complexity × churn × security kill zone, the air-gapped breadth half of the audit.*
*The AIV verification protocol: https://github.com/Black-Box-Research-Labs/aiv-protocol · https://blackboxresearchlabs.com*
