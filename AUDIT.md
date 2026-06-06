# Who Reviews the Reviewers? — a repo-governance self-audit
*by Black Box Research Labs*

> **You are an LLM coding agent (Claude Code, Cursor, Copilot, ChatGPT-with-tools, or similar).**
> The human pointed you at their repository and said "do this." This file is your complete instruction set. Execute it against **the current repository** and produce the scorecard at the end. You need nothing else — no API keys, no installs, no extra context.

---

## 0. What you are doing

You are running a **read-only governance audit** of the GitHub repository this agent is currently working in. You will answer 5 questions about how the project actually reviews its own code, using only:

- the **`gh` CLI** (already authenticated in this environment — verify with `gh auth status`)
- **`git`** (already present)
- **`jq`** (used **only** for the merge/aggregation steps that operate on already-projected fields; ships with most `gh` environments). **All extraction of `login`/`type`/`authorAssociation`/`state` is done with `gh`'s built-in `--jq` (gojq), never with a `gh api … | jq` pipe — see the ⚠️ box below.**
- optionally a **shallow clone** for two deep-mode metrics

**Hard rules — do not violate these:**

1. **READ-ONLY.** You will never write, push, comment, label, open, close, or modify anything in the user's repo or on GitHub. Every command below is a read. If you are about to run anything that mutates state, stop.
2. **SHOW YOUR WORK.** Every number you report must be printed *together with* the exact command that produced it *and* the raw counts. The audit must be re-runnable by hand from your output alone.
3. **STAMP EVERY NUMBER.** Never write "currently." Every figure is reported as: **`<value>, last 100 merged PRs, as of <YYYY-MM-DD>`** plus the counting rule used. Get today's date with `date -u +%Y-%m-%d` and reuse it on every line.
4. **DETERMINISTIC METHOD.** Use the exact classification rules and thresholds in §2 and the exact commands in §1/§3. Any agent running this file must get the same answer on the same repo at the same time. Do not improvise definitions; where a definition is a judgment call (Q2, Q4) state it explicitly and flag it.
5. **DON'T GUESS.** If a command returns nothing or errors, report the empty/error result honestly. Do not fabricate counts, reviewers, or PR numbers. If you cannot compute a metric, say "could not compute" and why.

> ### ⚠️ Read this before you run anything — three `gh`/`jq` gotchas this file already routes around
> These bit earlier runs; the commands below are written to avoid them. Do **not** "simplify" them back:
> 1. **`authorAssociation` is NOT a `gh pr list --json` field.** `gh pr list --json ...,authorAssociation` fails with `Unknown JSON field: "authorAssociation"`. It exists only at the REST top level (`gh api repos/<r>/pulls/<n>` → `.author_association`). That is why §1 enriches via `gh api`, not via `gh pr list`.
> 2. **`gh pr list --json reviews` lies about bots — bot detection on its output is UNSAFE.** It **strips the `[bot]` suffix** from review-author logins (`coderabbitai[bot]` → `coderabbitai`, `cubic-dev-ai[bot]` → `cubic-dev-ai`) **and omits `user.type`** (`author.is_bot` is always `null`). Run verbatim, every bot reviewer is mis-counted as a human and Q1/Q5 report the **opposite** of reality (0% bot share, 0% zero-review). **Review identities MUST come from `gh api repos/<r>/pulls/<n>/reviews`**, which returns the real `[bot]` suffix **and** `user.type == "Bot"`. Never derive reviewer identity from `gh pr list`.
> 3. **`gh api …/reviews | jq` is REJECTED by system `jq` ≥ 1.7.** Review **bodies** on real repos (`cubic-dev-ai`, `coderabbitai`, …) contain **literal control characters**. System `jq` ≥ 1.7 parses strictly and aborts the whole pipe with `jq: error (at <stdin>:N): control characters must be escaped` — so a `gh api repos/<r>/pulls/<n>/reviews | jq '…'` pipe **fails outright**, often silently swallowing the PR's reviews. The fix — used everywhere below — is to do **all** review/association extraction with **`gh api … --jq '…'`** (gh's bundled **gojq**, which tolerates raw control chars). **There is no `gh api …/reviews | jq` pipe anywhere in this file. Do not introduce one.** External `jq` only ever runs on the merge step, and only ever sees already-projected fields (`number`, `login`, `type`, `state`, `submitted_at`, `authorAssociation`) — **never a raw review `body`**.

---

## 1. Setup + build the working set (run once)

```bash
# Confirm auth + resolve the repo this agent is working in.
gh auth status
OWNER_REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner)
TODAY=$(date -u +%Y-%m-%d)

# Namespace ALL working files per repo + process so two concurrent runs (or two repos)
# never collide on shared /tmp paths. Everything below writes inside $WORKDIR.
WORKDIR="/tmp/wrr_$(echo "$OWNER_REPO" | tr / _)_$$"
mkdir -p "$WORKDIR"
echo "Auditing: $OWNER_REPO  as of $TODAY  (workdir: $WORKDIR)"
```

**Window.** The core audit uses the **last 100 merged PRs**. This keeps the run to a couple of minutes and respects rate limits. (Alternative window if the repo is low-volume or you want a time bound: add `--search "merged:>=$SINCE"` to the `gh pr list` below, where `SINCE` is 180 days ago — see the cross-platform `SINCE` helper in §4 — for a **trailing-180-day** window, and say so in your stamp. State which window you used.)

**Build the working set once** and reuse it for Q1, Q2, Q4, Q5. This is a **two-stage** pull:

- **Stage 1** — `gh pr list` gives clean, correctly-escaped PR metadata (numbers, dates, title, body, labels). It does **not** carry `authorAssociation` or trustworthy review-bot info (see the ⚠️ box above).
- **Stage 2** — for each PR, `gh api` adds the two fields `gh pr list` cannot: `author_association` and reviews with the real `[bot]` suffix + `user.type`. **Both Stage-2 reads use `gh api … --jq '…'` (gojq), so raw review bodies with control characters never reach external `jq`.**

```bash
# Stage 1: clean PR metadata (NO authorAssociation — that field is invalid here).
gh pr list --repo "$OWNER_REPO" --state merged --limit 100 \
  --json number,createdAt,mergedAt,title,labels,body \
  > "$WORKDIR/base.json"
echo "Stage 1: got $(jq 'length' "$WORKDIR/base.json") merged PRs (may be <100 on young repos)"

# Stage 2: enrich each PR with author_association + REAL review identities.
# Two cheap reads per PR; ~200 calls for a full 100-PR window, well within rate limits.
# Output is newline-delimited JSON so the merge step never re-parses a whole array.
#
# CRITICAL — extraction uses gh's BUILT-IN `--jq` (gojq), NOT a `gh api ... | jq` pipe:
#   * gojq tolerates the literal control characters that real review BODIES contain;
#     system jq >= 1.7 would abort with "control characters must be escaped".
#   * We project ONLY login + type per review (body is dropped at the gojq stage and
#     NEVER touches external jq downstream).
#   * Do NOT add --paginate to the per-PR /reviews call: it concatenates multiple JSON
#     arrays into one stream that the projection can't parse. Per-PR review counts are
#     small, so the plain single-page call below is correct and complete.
: > "$WORKDIR/enrich.jsonl"
TOTAL=$(jq 'length' "$WORKDIR/base.json"); i=0
for n in $(jq -r '.[].number' "$WORKDIR/base.json"); do
  i=$((i+1)); printf '\r  enriching %d/%d (PR #%s)   ' "$i" "$TOTAL" "$n" >&2
  # author_association via gh's --jq (gojq); emits one compact object:
  gh api "repos/$OWNER_REPO/pulls/$n" \
    --jq '{number, authorAssociation:.author_association}' 2>/dev/null \
    >> "$WORKDIR/enrich.jsonl"
  # reviews via gh's --jq (gojq) — projects login+type only, body never escapes gojq.
  # Plain single-page call (NO --paginate). $n is shell-substituted into the gojq filter.
  gh api "repos/$OWNER_REPO/pulls/$n/reviews" \
    --jq "{number:$n, apiReviews:[.[]|{login:.user.login, type:.user.type}]}" 2>/dev/null \
    >> "$WORKDIR/enrich.jsonl"
done; echo >&2

# Merge stage 1 + stage 2 into the single working set every question reads from.
# External jq here only ever sees already-projected fields (number/login/type/assoc) —
# never a raw review body — so it is safe under strict jq >= 1.7.
jq -s 'group_by(.number) | map(reduce .[] as $o ({}; . + $o)) | INDEX(.number|tostring)' \
  "$WORKDIR/enrich.jsonl" > "$WORKDIR/enr_map.json"
jq --slurpfile enr "$WORKDIR/enr_map.json" \
  '($enr[0]) as $e | [ .[] | . + ($e[(.number|tostring)] // {}) ]' \
  "$WORKDIR/base.json" > "$WORKDIR/merged.json"

echo "Working set: $(jq 'length' "$WORKDIR/merged.json") PRs in $WORKDIR/merged.json"

# SANITY CHECK — every PR number in the working set must belong to THIS repo.
# Catches a stale/foreign $WORKDIR file or a cross-repo collision. Abort if any leak.
BASE_NUMS=$(jq -S '[.[].number]|sort' "$WORKDIR/base.json")
MERGED_NUMS=$(jq -S '[.[].number]|sort' "$WORKDIR/merged.json")
if [ "$BASE_NUMS" != "$MERGED_NUMS" ]; then
  echo "FATAL: working-set PR numbers do not match Stage-1 set for $OWNER_REPO." >&2
  echo "       A foreign or stale file is in $WORKDIR — abort and re-run with a clean WORKDIR." >&2
  exit 1
fi
# You should see real [bot] suffixes and User/Bot types here:
jq -c '.[0:3][] | {n:.number, assoc:.authorAssociation, reviewers:[.apiReviews[]?|{login,type}]}' "$WORKDIR/merged.json"
```

> **What you now have in `$WORKDIR/merged.json`** — one object per merged PR with: `number, createdAt, mergedAt, title, body, labels, authorAssociation`, and `apiReviews[]` where each review carries `login` (with the real `[bot]` suffix) and `type` (`"User"` or `"Bot"`). Review **bodies are deliberately absent** — they were dropped at the gojq projection so no control-char-laden body ever reaches external `jq`. If `apiReviews` is empty for a PR, that PR genuinely had **zero** reviews. **All five questions read this one file** (`$WORKDIR/merged.json`) — do not re-fetch.
>
> ### Replaying against a FROZEN DUMP instead of live `gh` (schema remap is mandatory)
> The Q1/Q2/Q4/Q5 filters below read the **live** projection produced by §1: reviews under **`.apiReviews[]`**, with per-review keys **`.login`** and **`.type`**. If instead you are replaying these filters against a **pre-captured dump** of the working set, the dump may use a **different review schema** — commonly reviews under **`.reviews[]`**, author key **`.author`**, and **no `type` field at all**. Running the §3 filters **verbatim** against such a dump silently yields `total=N, zeroReview=N` (100% zero-human-review, 0% bot share) — the exact "opposite of reality" failure the ⚠️ box warns about — because `.apiReviews`, `r.type`, and `r.login` are all `null` on every dump record. **Before replaying on a dump you did not produce with §1, remap it to the live schema first** (one pass, no logic change):
> ```bash
> # Normalize a frozen dump to the live $WORKDIR/merged.json schema.
> # Maps .reviews[] -> .apiReviews[], .author -> .login, and (since the dump
> # drops type) reconstructs type from the [bot] suffix so the §2 isbot rule
> # still has its login signal. NO numeric logic changes — pure shape fixup.
> jq 'map(
>   if ((has("apiReviews")|not) and has("reviews")) then
>     .apiReviews = [ .reviews[]? | { login: (.author // .login),
>                                     type: (if (.type // null) != null then .type
>                                            elif ((.author // .login // "")|test("\\[bot\\]$";"i")) then "Bot"
>                                            else "User" end) } ]
>   else . end
> )' "$DUMP" > "$WORKDIR/merged.json"
> ```
> With this remap applied, every published number reproduces exactly. **The live §1 path needs no remap** — it already emits `.apiReviews`/`.login`/`.type`.

---

## 2. The deterministic classification rules (use these verbatim)

These are lifted as a spec from the Black Box audit engine. Apply them exactly.

**Bot account.** A reviewer is a **bot** if it satisfies *any*:
- the GitHub API marks it as a bot: `type == "Bot"` (this is the primary, authoritative signal — present because §1 pulls reviews from `gh api`), **OR**
- its login matches (case-insensitive) any of:
  - ends in `[bot]`  →  regex `/\[bot\]$/`  ← catches **every** GitHub App reviewer (`coderabbitai[bot]`, `cubic-dev-ai[bot]`, `ellipsis-dev[bot]`, `cursor[bot]`, `dependabot[bot]`, …) once the suffix is preserved
  - ends in `-bot`  →  regex `/-bot$/`
  - matches `/dependabot|renovate|github-actions|codecov|snyk-bot|greenkeeper|depfu|coderabbit|cubic|ellipsis|sourcery|qodo|codium/`

> **Why the extra names + why `type` is primary.** Earlier runs mis-classified the dominant reviewer as human two ways: (1) the data source stripped `[bot]`, and (2) modern AI reviewers like **coderabbitai**, **cubic-dev-ai**, **ellipsis-dev** don't contain the literal string `bot`. The §1 `gh api` source fixes (1) — `type:"Bot"` and the `[bot]` suffix now flow through, which alone re-classifies all of them correctly. The extra login names above are belt-and-suspenders for any reviewer that ever appears without the suffix. If you still see a high-volume reviewer classified as human whose own review bodies say "automated"/"AI"/"bot," report it explicitly as a likely mis-classification rather than trusting the rule blindly.
>
> **⚠️ When `type` is absent, classification rests ENTIRELY on the login regex — a latent fragility.** The live §1 path always supplies `type`, so the authoritative signal is present. But a **frozen dump** (see the remap box in §1) can drop `type` while preserving the `[bot]` suffix; on such input the `type=="Bot"` branch is dead and **every classification falls to the login regex alone**. That regex correctly catches `[bot]`-suffixed apps and the `-bot$` / hardcoded-vendor logins (e.g. `diffray-bot` via `-bot$`), but it has a **silent gap**: any bot whose login lacks `[bot]`/`-bot` **and** is not in the hardcoded vendor list will be counted as a **human** with no `type` fallback to catch it. If you are running without `type` (dump replay) and a high-volume reviewer's name looks tool-like but slips the regex, treat it as a suspected mis-classification and verify against the live `gh api .../reviews` `user.type` before trusting the human count.

**Human reviewer.** A reviewer that is **NOT a bot** by the rule above (engine equivalent: `type === "User"`). A PR is "human-reviewed" if it has **≥1** review from a human.

**Security PR.** A PR whose **title OR body OR any label** matches (case-insensitive), using **word boundaries** on the short tokens so `rce` does not match `source`/`force` and `cve` does not match arbitrary substrings:
```
/secur|vuln|xss|csrf|ssrf|exploit|advisory|inject|auth.*bypass|\bcve\b|\brce\b/i
```

**Thresholds (RED if exceeded):**
| Signal | RED when |
|---|---|
| Zero-review merge rate (Q1) | **> 20%** |
| Single-bot review share (Q5) | **> 50%** of all review events from one bot account |
| Security PR closed-unmerged (Q3) | **any** security PR closed without a merge (count + list) |
| Kill-zone hotspot (deep mode) | a file with **complexity ≥ 15 AND churn ≥ 15** |

Q2 and Q4 have no fixed numeric threshold — they require a stated definition and are reported as findings with the judgment call flagged (see below).

---

## 3. The 5 questions — exact computations

For each: run the command, capture the raw counts, apply the rule, stamp the result. The bot test below is shared; it reads `type` first (authoritative), then the login regex.

### Q1 — Zero-review merge rate
*What % of the last 100 merged PRs had zero **human** reviewers?*

> ### 🔴 DEFINITION — the headline is ZERO **HUMAN** REVIEWS; report the full decomposition
> The single most-bitten bug here is counting "no reviews of any kind" as the no-human rate, so the rule is pinned, not footnoted.
> - **`zeroHumanReview`** (the headline + the >20% gate) = merged PRs whose **human**-review count is `0`. A PR reviewed **only by bots** (e.g. only `coderabbitai[bot]`) has zero human reviews and **counts here.** Bot reviews do **not** rescue a PR from this bucket. `zeroHumanReview = total − humanReviewed`.
> - **Decomposition (report all three so this reconciles with any audit schema):** `zeroAnyReview` (no reviews at all) **+** `botOnlyReviewed` (reviews exist, none human) **=** `zeroHumanReview`. Some audit records publish the two sub-counts separately (e.g. `zeroReview`=21 **+** `botOnlyReviewed`=49 = 70); others publish only the union (`zeroReview`=71). Emitting all three reproduces both.
> - **`humanReviewed`** = merged PRs with **≥1** human review.
> - **Partition invariant (assert both):** `humanReviewed + botOnlyReviewed + zeroAnyReview == total`, **and** `zeroHumanReview == botOnlyReviewed + zeroAnyReview`. If either fails, the buckets are mis-derived.
> - **Do NOT report `zeroAnyReview` *alone* as the no-human rate** — that is the *zero-of-any-kind* under-count (it drops the bot-only PRs). The gate is always on `zeroHumanReview`.
> - **Percent check:** `noHumanReviewPct = 100 * zeroHumanReview / total`, and must back-solve to the printed `zeroHumanReview`.

```bash
jq -r '
  def isbot(r): (r.type == "Bot")
     or ((r.login // "") | test("\\[bot\\]$|-bot$|dependabot|renovate|github-actions|codecov|snyk-bot|greenkeeper|depfu|coderabbit|cubic|ellipsis|sourcery|qodo|codium";"i"));
  [ .[] | { n:.number,
            totalReviews: ([.apiReviews[]?] | length),
            # humanReviews counts ONLY non-bot reviewers. A bot-only-reviewed PR
            # has humanReviews==0 and is therefore (correctly) a zero-HUMAN-review PR.
            humanReviews: ([.apiReviews[]? | select(isbot(.)|not)] | length) } ] as $prs
  | ($prs|length) as $total
  | ([$prs[] | select(.humanReviews>0)] | length) as $humanReviewed
  # FULL DECOMPOSITION (so this reconciles with any audit schema):
  #   zeroAnyReview   = merged PRs with NO reviews at all (empty array)
  #   botOnlyReviewed = merged PRs WITH reviews but 0 human reviewers
  #   zeroHumanReview = humanReviews==0  ==  zeroAnyReview + botOnlyReviewed  (THE HEADLINE + the >20% gate)
  | ([$prs[] | select(.totalReviews==0)] | length) as $zeroAnyReview
  | ([$prs[] | select(.totalReviews>0 and .humanReviews==0)] | length) as $botOnlyReviewed
  | ([$prs[] | select(.humanReviews==0)] | length) as $zeroHumanReview
  | { total: $total,
      humanReviewed: $humanReviewed,
      zeroAnyReview: $zeroAnyReview,
      botOnlyReviewed: $botOnlyReviewed,
      zeroHumanReview: $zeroHumanReview,
      # invariants — BOTH must hold on a correct partition:
      partitionOK: (($humanReviewed + $zeroHumanReview) == $total
                    and ($zeroAnyReview + $botOnlyReviewed) == $zeroHumanReview),
      noHumanReviewPct: (if $total>0 then (100*$zeroHumanReview/$total) else 0 end),
      rate: (if $total>0 then (100*$zeroHumanReview/$total) else 0 end) }
' "$WORKDIR/merged.json"
```
Report: `zeroHumanReview/total = rate%`, with the decomposition `zeroAnyReview + botOnlyReviewed`. **RED if rate > 20%.** Also list up to 5 example zero-human-review PR numbers (`select(.humanReviews==0) | .n`) so the user can spot-check — and confirm `partitionOK == true` (i.e. `humanReviewed + botOnlyReviewed + zeroAnyReview == total`) before trusting the row. Bot-only-reviewed PRs (reviewed solely by e.g. `coderabbitai[bot]`) appear in this example list **by design** — they have zero human reviews.

### Q2 — Founder / maintainer PR review coverage
*Do PRs authored by the project's own owners/maintainers get reviewed, or do they self-merge unreviewed?*

> **JUDGMENT CALL — state this in your output.** "Founder/maintainer" has no perfect API field. Use this **primary definition** and flag it:
> **A PR is "maintainer-authored" if its `authorAssociation` is `OWNER` or `MEMBER`.**
> **⚠️ This definition is fragile on many real repos.** On a large number of OSS projects (notably `continuedev/continue`, `langchain-ai/langchain`, `crewAIInc/crewAI`), GitHub returns `author_association: CONTRIBUTOR` even for **co-founders / core maintainers** — e.g. the `continuedev/continue` CTO's own PRs come back as `CONTRIBUTOR`, not `OWNER`/`MEMBER`. So the `OWNER|MEMBER` bucket can be **empty** even though maintainers are merging daily. **If `maintainerPRs == 0`, do NOT report "100% coverage" or "no maintainer PRs" as a clean signal — report it as `INCONCLUSIVE` and fall back** to the top-committers cross-check below (deep mode) or to `gh api repos/$OWNER_REPO/collaborators` / the org-members list to identify maintainers, and say which you used.

```bash
jq -r '
  def isbot(r): (r.type == "Bot")
     or ((r.login // "") | test("\\[bot\\]$|-bot$|dependabot|renovate|github-actions|codecov|snyk-bot|greenkeeper|depfu|coderabbit|cubic|ellipsis|sourcery|qodo|codium";"i"));
  [ .[] | select(.authorAssociation=="OWNER" or .authorAssociation=="MEMBER")
        | { n:.number, assoc:.authorAssociation,
            humanReviews: ([.apiReviews[]? | select(isbot(.)|not)] | length) } ] as $m
  | { maintainerPRs: ($m|length),
      maintainerReviewed: ([$m[]|select(.humanReviews>0)]|length),
      maintainerZeroReview: ([$m[]|select(.humanReviews==0)]|length) }
  | . + { coveragePct: (if .maintainerPRs>0 then (100*.maintainerReviewed/.maintainerPRs) else null end) }
' "$WORKDIR/merged.json"
```
Report: `maintainerReviewed/maintainerPRs = coveragePct%` reviewed; `maintainerZeroReview` self-merged unreviewed. **State the definition used.** No fixed threshold — flag as a **finding** if a meaningful share of maintainer PRs merged with zero human review (the same governance gap as Q1, scoped to the people with merge rights). **If `maintainerPRs == 0`, report `INCONCLUSIVE — OWNER|MEMBER empty on this repo` and apply the fallback above.**

### Q3 — Security PRs closed without a fix landing
*Were any security-related PRs closed without ever being merged?*

This query is **independent of the merged-only working set** — it searches *closed-unmerged* PRs too. Note the **word-boundary** tokens (`\brce\b`, `\bcve\b`) so substrings like "sour**ce**", "for**ce**", "reinfor**ce**" do not produce false security matches.

```bash
# Security-matched, CLOSED, NOT merged, last 100 closed PRs.
gh pr list --repo "$OWNER_REPO" --state closed --limit 100 \
  --json number,title,labels,body,mergedAt,closedAt,url | jq -r '
  def secure(p): ( (p.title // "") + " " + (p.body // "") + " "
                   + ([p.labels[]?.name] | join(" ")) )
                 | test("secur|vuln|xss|csrf|ssrf|exploit|advisory|inject|auth.*bypass|\\bcve\\b|\\brce\\b";"i");
  [ .[] | select(.mergedAt==null) | select(secure(.))
        | { n:.number, title:.title, closedAt:.closedAt, url:.url } ] as $dropped
  | { closedUnmergedSecurityPRs: ($dropped|length), examples: ($dropped[0:5]) }
'
```
Report the **count of security-matched PRs closed without merging**, with the example PR numbers/URLs. **RED if count ≥ 1.**

> Note this Q3 pipe (`gh pr list … | jq`) is safe under strict `jq` — `gh pr list` JSON is properly escaped and carries **no raw review bodies** (the control-char hazard lives only on the `/reviews` API endpoint, which this query never touches).
>
> **HONEST LIMITS — state all three; this is keyword matching, NOT a vuln oracle.**
> 1. **Keyword matcher, not a vuln oracle.** Even with word boundaries, this matches on *keywords*; eyeball each hit's title before calling it a real security PR (e.g. a PR titled "improve **auth** flow UX" can match `auth.*bypass` only if "bypass" also appears — but a "**secur**ity headers" doc PR will match `secur`). In practice **most hits are false positives** — `inject` matches "dependency **inject**ion"/"**inject** a mock", `auth` matches routine auth-flow/login UX work, and `dependabot`-style automated PRs frequently trip the matcher. Report the matched titles so the user can dismiss obvious non-vulns. Do **not** assert "N dropped security PRs = governance failure" without reading the titles.
> 2. **Closed-unmerged ≠ fix never landed.** This finds security-keyword PRs that were *closed unmerged*. It does **NOT** prove "no fix EVER landed" — the fix may have re-landed under a different PR or been superseded by a maintainer commit. Verifying "the vulnerability is still live" requires CVE/advisory cross-reference this audit cannot fully automate. Report the **closed-unmerged-security-PR subset** as the signal, and say so. If GitHub Security Advisories are public for the repo, mention `gh api repos/$OWNER_REPO/security-advisories` as a manual next step (may require permissions).
> 3. **Raw count is window-size-sensitive.** The closed-unmerged-security count scales with how many closed PRs you scanned (here, last 100 closed). Report it **as a rate too** — `<k> closed-unmerged security hits / <N> closed PRs scanned` — so it is comparable across repos and windows, not just a bare integer.

### Q4 — External vs internal merge latency
*Do outside contributors wait longer to get merged than the core team?*

> **JUDGMENT CALL — state the bucketing.** Bucket each merged PR by `authorAssociation`:
> - **INTERNAL** = `OWNER` or `MEMBER`
> - **EXTERNAL** = `CONTRIBUTOR` or `FIRST_TIME_CONTRIBUTOR` or `FIRST_TIMER` or `NONE`
> Latency = `mergedAt − createdAt`, reported as the **median** per bucket (median, not mean, to resist outliers). Flag that `COLLABORATOR` PRs are excluded as ambiguous, and that association is GitHub's snapshot at audit time, not at PR-open time.
> **⚠️ Same fragility as Q2.** Because many maintainers come back as `CONTRIBUTOR` (see Q2), the **INTERNAL** bucket is frequently **empty** and **EXTERNAL** absorbs everyone — making a clean external-vs-internal comparison impossible on those repos. **If `internalCount == 0` (or either bucket has n < 5), report `INCONCLUSIVE — cannot separate cohorts via author_association on this repo`** rather than a misleading "external is slower" claim. As a fallback, you may bucket by a maintainer login list from `gh api repos/$OWNER_REPO/collaborators`; if you do, state it.

```bash
jq -r '
  def hrs(p): ((p.mergedAt|fromdateiso8601) - (p.createdAt|fromdateiso8601))/3600;
  def med(a): (a|sort) as $s | ($s|length) as $l
     | if $l==0 then null elif ($l%2==1) then $s[($l/2|floor)]
       else (($s[$l/2-1]+$s[$l/2])/2) end;
  [ .[] | select(.mergedAt!=null) ] as $m
  | ([ $m[] | select(.authorAssociation=="OWNER" or .authorAssociation=="MEMBER") | hrs(.) ]) as $int
  | ([ $m[] | select(.authorAssociation|IN("CONTRIBUTOR","FIRST_TIME_CONTRIBUTOR","FIRST_TIMER","NONE")) | hrs(.) ]) as $ext
  | { internalCount: ($int|length), internalMedianHours: (med($int)),
      externalCount: ($ext|length), externalMedianHours: (med($ext)) }
' "$WORKDIR/merged.json"
```
Report both medians (convert hours→days if large: `÷24`) and the n in each bucket. State the bucketing. No fixed threshold — flag as a **finding** if external median is materially higher than internal (the "two-speed" governance gap). **If either bucket has n < 5 (or internal is empty), say the sample is too small / cohorts inseparable and mark `INCONCLUSIVE`.**

### Q5 — Bot vs human review deference
*How much of code review is done by bots, and is one bot account dominating?*

```bash
jq -r '
  def isbot(r): (r.type == "Bot")
     or ((r.login // "") | test("\\[bot\\]$|-bot$|dependabot|renovate|github-actions|codecov|snyk-bot|greenkeeper|depfu|coderabbit|cubic|ellipsis|sourcery|qodo|codium";"i"));
  [ .[] | .apiReviews[]? ] as $rv
  | ($rv|length) as $allReviews
  | ([ $rv[] | select(isbot(.)) ] | length) as $botReviews
  | ([ $rv[] | select(isbot(.)) | .login ] | group_by(.)
        | map({login: .[0], count: length}) | sort_by(-.count)) as $byBot
  | { totalReviewEvents: $allReviews,
      botReviewEvents: $botReviews,
      botSharePct: (if $allReviews>0 then (100*$botReviews/$allReviews) else 0 end),
      topBot: ($byBot[0]),
      singleBotSharePct: (if $allReviews>0 and ($byBot|length)>0 then (100*$byBot[0].count/$allReviews) else 0 end) }
' "$WORKDIR/merged.json"
```
Report: bot-review share, and the **single-bot concentration** (the largest one bot's share of all review events). **RED if single-bot share > 50%.** Name the top bot. (If you got `topBot: null` AND `totalReviewEvents > 0`, your data source is wrong — you are almost certainly reading `gh pr list`'s stripped logins instead of `apiReviews`; re-run §1.)

---

## 4. OPTIONAL — deep mode (shallow clone; two git-walk metrics)

Run this **only if** a clone is feasible (disk + network OK). If not, **skip it and note "deep mode skipped — clone not feasible; core audit complete."** Core mode (§3) is fully valid on its own.

```bash
# Cross-platform "180 days ago" (works on macOS/BSD AND Linux/GNU date):
SINCE=$(date -u -v-180d +%Y-%m-%d 2>/dev/null || date -u -d '180 days ago' +%Y-%m-%d)
echo "deep-mode window start: $SINCE"

# Depth must cover the window. On high-velocity repos --depth 200 may be < 1 week,
# so the 180-day metrics below would see almost nothing. Use a depth that reaches $SINCE,
# falling back to a generous fixed depth, then a full clone, if the shallow clone is too short.
git clone --shallow-since="$SINCE" "https://github.com/$OWNER_REPO" "$WORKDIR/clone" 2>/dev/null \
  || git clone --depth 2000 "https://github.com/$OWNER_REPO" "$WORKDIR/clone" 2>/dev/null \
  || git clone "https://github.com/$OWNER_REPO" "$WORKDIR/clone" 2>/dev/null \
  && echo "cloned" || { echo "clone failed — skip deep mode, report core only"; }
# Confirm the clone actually reaches the window (else say so and treat deep numbers as a floor):
git -C "$WORKDIR/clone" --no-pager log -1 --format='oldest reachable commit: %ci' $(git -C "$WORKDIR/clone" rev-list --max-parents=0 HEAD 2>/dev/null | tail -1) 2>/dev/null
```

### Deep-A — SILO / bus-factor
*Is the codebase's knowledge concentrated in too few people?* (engine method: `git shortlog -sne`)

```bash
# NOTE: shortlog needs an explicit revision (HEAD). Without it, when stdout is not a TTY
# (i.e. always, when an agent captures output) it reads from EMPTY stdin and prints nothing.
git -C "$WORKDIR/clone" --no-pager shortlog -sne --since="$SINCE" HEAD | head -20
```
Compute: total commits in window, top-1 author share, **top-3 author share**. **Flag if top-3 share > 70%** (Knowledge-Silo threshold). Print the author table (names + commit counts) as your evidence. If the table is empty, your clone did not reach `$SINCE` (re-clone deeper) — do not report "no commits." Note that this is commit authorship, not per-file blame — a proxy for bus-factor, not precise per-file attribution.

### Deep-B — kill-zone (complexity × churn)
*Which files are both complex and changing constantly — the hotspots most likely to hide bugs?*
Kill-zone threshold (engine `analyze.ts`): **complexity ≥ 15 AND churn ≥ 15.**

Churn (changes per file in the window) without extra tools:
```bash
git -C "$WORKDIR/clone" --no-pager log --since="$SINCE" --name-only --pretty=format: HEAD \
  | grep -v '^$' | sort | uniq -c | sort -rn | awk '$1>=15' | head -20
```
This gives the **churn ≥ 15** candidate set. If it is empty, confirm the clone reached `$SINCE` (Deep-A check) before concluding "no hotspots." For complexity, if a linter/cc tool is already in the repo (e.g. `radon` for Python, `eslint` complexity rule) use it on those candidate files; otherwise report the high-churn list as **"kill-zone candidates (churn ≥ 15; complexity not measured — no analyzer present)"** and say complexity was not computed. Do not fabricate complexity numbers.

---

## 5. OUTPUT — the scorecard (this is your deliverable)

Print exactly this shape. Tight. One row per question. Fill `<>` with your real numbers, then the one-line verdict. **Every value carries its stamp.**

```
WHO REVIEWS THE REVIEWERS? — <OWNER/REPO>
Window: last 100 merged PRs (got N=<n>)  |  As of: <YYYY-MM-DD>  |  Mode: core[ + deep]

# | Question                          | Number (stamped)                          | Threshold        | Verdict | Reproduce
--+-----------------------------------+-------------------------------------------+------------------+---------+----------------------------------------
1 | Zero-review merge rate            | <z>/<n> = <r>%                             | >20% = RED       | <R/G>   | <Q1 jq command>
2 | Maintainer PR review coverage     | <a>/<b> = <c>% reviewed (def: OWNER|MEMBER)| judgment call    | <flag>  | <Q2 command>
3 | Security PRs closed w/o merge     | <k> closed-unmerged security PRs           | >=1 = RED        | <R/G>   | <Q3 command>
4 | External vs internal merge latency| ext med <x>d (n=<>) / int med <y>d (n=<>)  | judgment call    | <flag>  | <Q4 command>
5 | Bot/human review deference        | bot share <p>%; top bot <login> <s>%       | single-bot >50%  | <R/G>   | <Q5 command>
--+-----------------------------------+-------------------------------------------+------------------+---------+----------------------------------------
[deep] SILO bus-factor: top-3 = <t>% of <commits> commits (180d)        | >70% = flag | <flag> | git shortlog -sne --since="$SINCE" HEAD
[deep] Kill-zone: <m> files churn>=15 (complexity <measured/skipped>)    | cx>=15&churn>=15 | <flag> | git log --name-only ... HEAD | uniq -c

VERDICT: <one line — e.g. "2 RED (zero-review 31%, single-bot 64%); maintainer/latency INCONCLUSIVE (OWNER|MEMBER empty). As of <date>, last 100 merged PRs.">
```

**Rules for the scorecard:**
- Every number ends in its stamp pattern: `<value>, last 100 merged PRs, as of <date>` (compress in the table, but the verdict line must carry the stamp once).
- Beneath the table, paste the **raw counts** for each question (the jq output) so the audit is self-verifying.
- For Q2/Q4 the "Verdict" column says `FLAG`, `OK`, or `INCONCLUSIVE`, never RED/GREEN, because they're judgment calls — and you must have stated your definition. Use `INCONCLUSIVE` when the `OWNER|MEMBER` bucket is empty. **Never emit a silent `100%`/`clean` for an empty maintainer bucket — that is `INCONCLUSIVE`, not a pass.**
- **Report raw counts as rates, not bare integers.** Q3 (`<k>/<N> closed PRs scanned`), the deep-mode top-3 share (`% of <commits> in window`), and zero-review (`<z>/<n>`) are all **window-size-sensitive** — normalize them so they compare across repos and windows.
- **Before printing the Q1 row, assert the partition closes:** `humanReviewed + botOnlyReviewed + zeroAnyReview == total` (the `partitionOK` field). The `<z>` you print for zero-review is the **zero-HUMAN-review** count (`zeroAnyReview + botOnlyReviewed`). If `partitionOK` is `false`, do not ship the row — the buckets are mis-derived (see §6).
- If any data was missing (no clone, n<100, empty reviews, empty maintainer bucket), say so in the verdict line. Graceful degradation is honest, not a failure.

---

## 6. Honest limits (include a short version in your output)

- **Q3 is keyword matching, not a vuln oracle, and closed-unmerged ≠ fix-never-landed.** It reports security-*keyword* PRs *closed unmerged*. Word boundaries kill the worst substring false positives (`source`/`force`), but it is still keyword matching — and in practice **most hits are false positives** (`inject` → "dependency injection", `auth` → routine login/UX work, `dependabot` automated PRs). It does **not** prove a vulnerability is still live: a closed-unmerged hit may have re-landed under another PR or via a maintainer commit (needs CVE/advisory cross-reference no read-only script can fully automate). Eyeball every title; report the count **as a rate** (`<k>/<N> closed PRs scanned`).
- **Q2 ("founder") and Q4 (external/internal) are fragile and can be `INCONCLUSIVE`.** Both depend on `authorAssociation`, which GitHub frequently returns as `CONTRIBUTOR`/`COLLABORATOR` even for core maintainers — so the `OWNER|MEMBER` bucket can be empty. When it is, **report `INCONCLUSIVE`/`FLAG` — never a silent `100%` or `clean`** — and fall back to a collaborator/committer list. State your definition; don't overclaim.
- **Raw counts (Q3 hits, deep-mode top-3 share, zero-review) are window-size-sensitive — report normalized rates,** not bare integers, so figures compare across repos and windows.
- **The headline is `zeroHumanReview` (= `totalMerged − humanReviewed`); report the full decomposition.** A merged PR reviewed **only by bots** has zero human reviews and counts in `zeroHumanReview`. Emit all three — `zeroAnyReview + botOnlyReviewed = zeroHumanReview` — so the output reconciles with audit records that publish the sub-counts separately **or** only the union. Assert `humanReviewed + botOnlyReviewed + zeroAnyReview == totalMerged`. `noHumanReviewPct` must back-solve to `zeroHumanReview`; never report `zeroAnyReview` alone as the no-human rate.
- **Bot classification** depends on §1 pulling reviews from `gh api` (which preserves the `[bot]` suffix and `user.type`); deriving it from `gh pr list` output is UNSAFE — that source strips the `[bot]` suffix and omits `user.type`, silently mis-classifying every bot as a human. If your bot share is 0% on an active repo, you read the wrong source.
- All review/association extraction is done with `gh api … --jq` (gojq); a `gh api …/reviews | jq` pipe would be rejected by system `jq` ≥ 1.7 on real review bodies (control characters must be escaped). External `jq` only ever sees already-projected fields, never a raw body.
- Association and bot flags are read at audit time, which can differ from PR-open time.
- Small repos (n < ~30 merged PRs) make every rate noisy — say the sample is small.

---

*Who Reviews the Reviewers? — a repo-governance self-audit, by Black Box Research Labs.*
*The AIV verification protocol: https://github.com/Black-Box-Research-Labs/aiv-protocol · https://blackboxresearchlabs.com*
