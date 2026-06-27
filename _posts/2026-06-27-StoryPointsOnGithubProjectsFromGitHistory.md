---
layout: post
title: Story points and retro points on GitHub Projects, backfilled from git history
---

I track effort on [Requel](https://github.com/rreganjr/Requel) with a **Story Points** estimate up front, and a **Story Points (Retro)** value after the work is done so I can see how wrong the estimate was. Requel lives on GitHub, and I wanted the two-number scheme on my issues to help improve my original estimates, organized per release. This post is how I got there — the GitHub Projects model, the authentication dead-ends, a way to *derive* retro points from git history instead of guessing, and the helper scripts and the Claude skill that automate the bulk update.

I built all of this in a session with **Claude Cowork**: it wrote the scripts and read through the git history alongside me while I ran the `gh` commands and hit the walls. The dead-ends below are real ones we worked through together, and a few of them — the JSON-key casing especially — are exactly the kind of thing that's faster to find when something can grep the raw output for you. I'll call out where that mattered.

## Issues don't have custom fields — Projects do

The first thing to unlearn: a GitHub *issue* has no custom-field mechanism. There's nowhere on the issue itself to hang a number called "Story Points." Custom fields live on a **Project** (the table/board boards, formerly Projects v2), which is a separate object that *references* issues.

You add an issue to a Project and it becomes a row ("item"). The Project defines fields — Text, Number, Date, Single select, Iteration — and the values are stored on the *item*, not the issue. So Story Points is a Number field on the Project, and the same issue could in principle show different values in two different Projects.

That last detail matters for how you read and write the values: they're not on the issue page and not in the issues REST API. They live behind the GraphQL `ProjectV2` API, which `gh` wraps as `gh project ...`. Two number fields gave me what I wanted:

- **Story Points** — the up-front estimate
- **Story Points (Retro)** — filled in after the issue is closed

## The token rabbit hole

This is where I lost the most time, so I'll save you the same hour.

`gh project` needs permissions the default token usually doesn't have, and **the kind of token you use decides whether it can work at all.**

**Fine-grained tokens cannot write user-owned Projects.** This is the big one. I was using a fine-grained PAT, and reads worked — `gh project list` happily showed my project — but every write failed with:

```
GraphQL: Resource not accessible by personal access token (createProjectV2Field)
```

I went looking for a "Projects" permission to add and there isn't a usable one for a project owned by a *user* account. It's [a documented limitation](https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens): fine-grained tokens don't get write access to user-owned Projects v2. The reads succeeding is what makes this so confusing — it looks like a missing checkbox, not a hard wall.

**The fix is a classic token with the `project` scope.** On the classic-token page (`github.com/settings/tokens` → *Tokens (classic)*), `project` isn't something you search for — it's a checkbox near the bottom of the scope list, "Full control of projects." Tick it, keep `repo`, and you can write.

**You also need `read:org`, or you get `unknown owner type`.** With the classic token in place, `gh project list --owner rreganjr` worked, but the scripts failed with `unknown owner type`. `gh` resolves whether the `--owner` is a user or an org before doing anything, and that lookup needs `read:org`. It's not obvious from the error. So the working classic-token scope set is **`project` + `repo` + `read:org`**.

**And `GH_TOKEN` still wins over everything.** I [wrote previously]({% post_url 2026-05-12-GhCliMultiAccountWithDirenv %}) about using `direnv` to export a per-directory `GH_TOKEN`. That setup means `gh auth login` and `gh auth refresh` are no-ops for me — `gh` reads the env var and ignores the keychain. So "just run `gh auth refresh -s project`" doesn't apply: I had to regenerate the token's scopes and update the file `GH_TOKEN` points at. If you're on a fine-grained token via `GH_TOKEN`, editing its permissions on GitHub keeps the same token string, so the env var picks up the new scopes with no further change — but you can't grant a fine-grained token a capability it fundamentally lacks, which is exactly the wall above.

The quick test that you're unblocked:

```bash
gh project list --owner rreganjr     # lists projects, no permission error, no "unknown owner type"
```

## One project per release, matched by milestone

I settled on **one Project per release** rather than a single perpetual board. I already use **milestones** to mark releases (`v2.0`), so "which release does this issue belong to" is native metadata — I don't have to encode it by hand.

That lets a single input drive everything by naming convention:

| Input | Derives | Example |
|---|---|---|
| `RELEASE` | — | `2.0` |
| milestone | `v<RELEASE>` | `v2.0` |
| project title | `Requel <RELEASE>` | `Requel 2.0` |

So I pass `2.0` and the tooling knows the milestone to pull issues from and the project to write into. New release: make the `v2.1` milestone, tag its issues, run the same commands with `2.1`.

(I went back and forth on one-project-per-release versus a single board with a per-release view. The single board gives you cross-release velocity trends in one place, which is nice; per-release projects give cleaner isolation and archival. I picked isolation. Either way, *matching by milestone* beats matching by date or by hand.)

## Deriving retro points from git history

Here's the part I actually like. The estimate is a guess, but the **retro** value shouldn't be — by the time an issue is closed, the work already happened and git knows how much there was.

Two facts about how I work make this measurable. First, **I commit every day I touch an issue.** Second, this repo's commit convention puts the **full issue URL** on the first line of every commit:

```
https://github.com/rreganjr/Requel/issues/43

Modernize background analysis ...
```

So "distinct calendar days with a commit referencing issue N" is a faithful proxy for "days of work on N." My scale is deliberately rough: **about one working day is one point**, snapped to Fibonacci. Idle gaps fall out for free, because only days that *have* a commit get counted.

Counting the days is one line:

```bash
issue_commit_days() {   # usage: issue_commit_days 43
  git log --all -E --grep="issues/$1(\$|[^0-9])" --format='%ad' --date=short \
    | sort -u | grep -c .
}
```

The word-boundary regex matters: a naive `--grep='#38'` both misses everything (the convention is the URL, not `#38`) and false-matches `issues/380`. Match the `issues/N` URL form with a boundary.

Snapping to Fibonacci, with ties rounding up (so 4 → 5, matching my habit of rounding 12 → 13):

```bash
snap_fib() {            # usage: snap_fib 4 -> 5
  local d="$1"; local fib=(1 2 3 5 8 13 21 34 55 89); local best=0 bd=999999 diff
  for f in "${fib[@]}"; do
    diff=$(( f > d ? f - d : d - f ))
    if [ "$diff" -lt "$bd" ] || { [ "$diff" -eq "$bd" ] && [ "$f" -gt "$best" ]; }; then
      bd="$diff"; best="$f"
    fi
  done
  echo "$best"
}
```

Run against the real history, this produced numbers that matched my gut: `#40` (delete an orphaned jar) was 1 commit-day → 1; `#73` (personal access tokens) 3 → 3; `#69` (an MCP gateway) 4 → 5; `#43` (background analysis rework) 10 → 8.

And it correctly flagged an epic. **#38, the whole Echo2-to-Angular UI migration, was 39 distinct commit-days → 34.** That's not a story, it's a release's worth of work that happened to share one issue number (82 commits, ~68k lines, committed incrementally rather than squashed — which is exactly why the commit-day count captures it). A single point value on an issue like that is meaningless for velocity; the lesson the number teaches is "this should have been ten issues." I tag it as an epic and move on.

The estimate (initial Story Points) I leave at **0** unless I actually estimated up front. My design docs in `doc/` are designs, not estimates, and I'd rather have an honest zero than a number I reverse-engineered.

## The JSON-key gotcha that ate an afternoon

When I went to find which items already had a retro value (so I wouldn't clobber them), my filter matched nothing — even on items I could see had values in the UI. The cause is a small, very specific `gh` behavior:

**In `gh project item-list --format json`, custom-field values appear as top-level keys with only the *first letter* lowercased.** A field named "Story Points (Retro)" is not `story points (retro)` and not `Story Points (Retro)` — it's:

```
story Points (Retro)
```

Capital P, the rest verbatim. "Story Points" becomes `story Points`. An all-lowercase key silently matches nothing, no error. Once I knew, the filter was obvious:

```bash
gh project item-list 2 --owner rreganjr --format json \
  | jq -r '.items[] | select(.content.type=="Issue")
           | [.content.number, (.["story Points (Retro)"] // "—")] | @tsv'
```

If you're ever unsure what `gh` named a field's value, dump one item that has a value set and read the keys — don't assume the casing.

## Retro is for finished work — enforce it, don't just intend it

My rule is that an open issue should never carry a retro value; you point it once it's *done*. I wrote that rule down, and then promptly violated it: an early bulk run set retro values on whatever had commits, open issues included, because the script that sets a value didn't check state. A rule that isn't in the code isn't a rule.

So the value-setting script now checks `gh issue view <n> --json state` and refuses to write a retro on anything that isn't `CLOSED` (it'll still set an initial estimate — estimating before work is fine). And there's a cleanup pass for the ones that slipped through earlier, which clears *only* the retro field on open items and leaves estimates alone. A nice side effect: GitHub's built-in project workflow sets the **Status** column from issue state (`Todo` = open, `Done` = closed), so a quick scan of that column is enough to spot a stray retro on an open row.

## The scripts

The whole thing is a handful of small bash scripts over `gh`, sharing one config file so the release knob lives in exactly one place:

- **`retro-lib.sh`** — sourced by the rest; maps `RELEASE` → milestone + project title, and holds `commit_days`, `snap_fib`, and the project lookup.
- **`setup-project.sh <release>`** — create (or reuse) the release's Project and its two number fields, link it to the repo. Idempotent.
- **`set-points.sh <issue#> <initial> [retro]`** — point one issue; retro auto-derives from commit-days when omitted, and is refused on open issues.
- **`backfill-points.sh <release>`** — the bulk update: pull every closed issue in the milestone and set each one's retro. No hardcoded list — it's driven by the milestone, so it self-maintains.
- **`clear-open-retros.sh <release>`** — fix open issues that wrongly carry a retro (dry-run by default).
- **`audit-retros.sh <release>`** — read-only health check that flags open-with-retro, closed-without-retro, and "the recorded retro no longer matches the commit-day calc."

The day-to-day is two commands per release:

```bash
./scripts/setup-project.sh 2.0      # one time: project + fields
./scripts/backfill-points.sh 2.0    # retro every closed issue in milestone v2.0
```

## Packaging it as a skill (and not losing it to .gitignore)

Since I built this with Claude Cowork and expect to run it the same way next release, I had it write the whole procedure up as a project **skill** — a `SKILL.md` describing the release/milestone model, the commit-day method, the token requirements, and when to reach for each script. It lives in `.claude/skills/issue-retro/`, so a future session picks up the method instead of rediscovering it.

Which surfaced one last gotcha: **`.claude/` is in my `.gitignore`**, so the skill isn't version-controlled and would vanish on a fresh clone. The fix is to keep a tracked copy under `doc/`: both the raw `SKILL.md` (so changes are diffable) and a zipped `*.skill` package (so it's one click to reinstall). The working copy in `.claude/` stays the thing Claude actually loads; `doc/claude/skills/` is the backup and the distribution point.

## Why I wanted this

I like the estimate-versus-actual loop. The estimate is a planning tool and it's allowed to be wrong; the retro number is the feedback that makes the *next* estimate less wrong. Doing it by hand never happens, so it has to be cheap. Deriving the retro from commits I was already making — for free, in a way I trust more than my memory — is what made it cheap enough to actually do.

It also turned the migration epic into an honest data point: when a single issue eats 39 days, the tooling says 34 and I get the message. The number isn't the goal; noticing is.

## Future things I'd like to do

A few extensions that slot in cleanly but I haven't built yet:

**A weekly propose-only pass.** Run the retro calc on newly-closed issues on a schedule and hand me a table to approve, rather than me remembering to run the backfill.

**Reconciling the estimate.** Right now I leave the initial estimate at 0 for everything I didn't pre-estimate. Going forward I'd like to actually record an estimate when I open an issue, so the retro has something to be measured against and the Δ becomes the interesting column.

**Cross-release velocity.** Per-release projects make trend-over-time harder. A small report that reads the retro totals across all the `Requel <release>` projects would give me the velocity curve I gave up by not using a single board.
