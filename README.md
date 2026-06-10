**Point your coding agent at your repository and tell it: "run the governance audit in AUDIT.md."**

That's the whole thing. `AUDIT.md` is a self-contained instruction set any LLM coding agent (Claude Code, Cursor, Copilot, ChatGPT-with-tools) can execute against your current repo using only its existing `gh` auth and `git` — no installs, no API keys, no config. It runs read-only and prints a one-screen scorecard answering five governance questions: zero-review merge rate, maintainer PR review coverage, dropped security PRs, external-vs-internal merge latency, and bot-vs-human review deference — every number stamped with its date, window, and counting rule so you can re-run it yourself.

*Why:* most teams have never measured who actually reviews their code. This is the gift version of the audit we run for a living — point it at your own repo and find out before someone else does. ("self-audit" = the tool audits **whatever repo you point your agent at** — your repo, not ours.)

---

*Who Reviews the Reviewers? — a repo-governance self-audit, by Black Box Research Labs.*
*Sibling tool: find-your-kill-zone (https://github.com/Black-Box-Research-Labs/find-your-kill-zone). Point your agent at your repo and find your complexity × churn × security kill zone, the air-gapped breadth half of the audit.*
*The AIV verification protocol: https://github.com/Black-Box-Research-Labs/aiv-protocol · https://blackboxresearchlabs.com*
