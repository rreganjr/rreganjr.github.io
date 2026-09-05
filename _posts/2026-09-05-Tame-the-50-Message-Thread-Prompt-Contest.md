---
layout: post
title: Tame the 50-message thread — testing a prompt like software, then losing to a rubric
---

Work ran an internal prompt-writing challenge: write a prompt that makes Claude summarize a long, messy
Slack or email thread into what actually matters — the decision, the open questions, and who owns what
next. One entry per person, no resubmits. Scored on five things: extraction focus, handling disagreement
and duplication, constraints (brevity, no invented facts), an output format an exec can skim, and edge
cases like "no decision was reached." Bonus for telling Claude what to *ignore*.

I could have written a good prompt in twenty minutes. Instead I spent two days building a test harness
for it, ran it well over a hundred times across five threads and three model tiers, found and fixed real
gaps, and then tied for first with two people who — as far as I know — did none of that. This post is
about what the testing found, why it didn't move the score, and what I'd do differently.

I paired on the whole thing with Claude, which wrote the prompt, the scorer, and most of the test threads,
and ran the blind evaluations as subagents so the model being tested never saw the answer keys.

## The first prompt

The brief was clear enough that the first draft came together quickly: a set of reading rules ("last word
wins — a decision made, reversed, and re-made counts only in its final form"), a definition of what counts
as a decision ("someone with standing states it and nobody overrides it"), an explicit ignore list
(greetings, emoji-only messages, the offsite, the dog photo, story points a human said to disregard), hard
constraints (never invent an owner or date; write `Unassigned` and `No date` literally; ≤250 words), a
fixed output shape (Status → Decisions → Open questions → Owners table → Watch out), a separate shape for
threads where nothing got decided, and a self-check list.

I had a small test kit to work with: a synthetic 50-message Slack thread with thirteen planted traps
(a decision made, reversed, and re-made; a date stated wrong, corrected, then requoted wrong by a
latecomer; a bot message with numbers that weren't the source of truth; a decision that lived only in an
edited message; a "let's take this offline" that never came back) and a shorter email thread where the
group never reached a decision. Each came with an answer key.

The first blind run caught the big traps and leaked the small ones: the offsite date showed up in "Watch
out," the JiraBot story points got mentioned even though someone in the thread had said to ignore them,
and the offline sync that never reported back was missing entirely. Two rounds of edits fixed those, and I
had something that looked done.

## Two reviews that found the same kind of gap

I ran the prompt past a second Claude session acting as a reviewer, with the answer key. Its diagnosis was
sharper than mine: the prompt defined what a decision was and what an open question was, but left two
middle states unclassified. A deferral ("park it, revisit after QA") had a plausible home under Decisions.
A follow-through that was owed but not delivered (two people took a topic offline and never came back)
had a plausible home under Watch out. The model was picking the wrong home because the prompt hadn't
assigned one. Five targeted edits closed it — a deferral is not a decision, an unreported sync always
produces both an open question and a table row, Watch out can never substitute for a section.

A second review pushed further: the answer key was essentially a rubric with a thread-specific
failure-mode list, and the prompt's closing checklist was generic. Adding "before writing, list the two or
three ways a skimmer could misread *this particular thread*" would generate that list on every run. And a
worked example — a short thread plus its ideal output — would anchor the format and the level of
conservatism better than more rules. I added both. The example was an original 13-message thread about a
pricing-page launch, so it shared nothing with the test kit.

That put the prompt at about 2,200 words. It scored well on the two kit threads. But by then I'd tuned it
against both of them, so passing them proved less than it looked.

## Building the scorer

The honest test needed threads the prompt had never seen and a way to grade outputs that didn't depend on
me eyeballing them. Claude wrote `score.py`: a JSON key per thread describes checks, and the script prints
a check-by-run matrix with pass rates. Two check types cover almost everything. A `regex` check matches
(or must not match) a pattern, optionally scoped to one section of the output. A `row` check finds a table
row whose Action matches a pattern and requires the Owner or Due to match:

```json
{"id": "ticket_on_dev",
 "desc": "analytics ticket landed on Dev (not Kevin or Priya)",
 "type": "row", "action": "ticket", "owner": "Dev", "owner_not": "Kevin|Priya"}
```

Every key also gets six generic checks for free: body word count under the cap, required sections present,
at most three Watch out bullets, no narration about what was ignored, no deferral language under
Decisions, and — the one that matters most — **no invented dates**: every "Mon DD" token in the output
has to appear in the thread.

Then two hold-out threads. Thread C was a 46-message email chain about a conference booth and a sponsored
symposium — a different domain, a different format, with email-native traps: a decision only in a P.S., a
forwarded approval from someone outside the thread, a vendor portal email with wrong numbers that a human
said to ignore, someone who volunteered for shipping and then un-volunteered, quoted text repeating a
stale date. Thread D was the opposite test: eight clean messages where everything was decided and
assigned, to see whether a prompt this strict would manufacture doubt where there was none.

And I wrote one myself — Thread E, a friends' vacation-planning thread with a date that gets locked, a
destination that doesn't, a participant who keeps pushing an option after five people said no, and someone
who repeatedly confuses July with August and puts the wrong calendar hold in. Claude read the thread and
wrote its key before it looked at mine.

## What the runs found

Five runs per thread on the first real version:

| thread | score |
|---|---|
| A (50-message stress test) | 97% |
| B (no decision reached) | 96% |
| C (hold-out email) | 95% |
| D (clean thread) | 100% |

Two things failed on every single run. Word count: every output was 253–330 body words against a 250
cap. The structure at 25 words per item simply lands at ~275 regardless of what the cap says, so I
replaced the cap with per-item limits (status ≤30 words, items ≤20, Watch out bullets ≤12). And the
vendor-portal glitch on Thread C — "Booth reservation updated. Size: 20x20" — showed up in Watch out on
5 of 5 runs even though the very next message said "ignore that, the portal glitched." The rule "ignore
anything a human said to disregard" lost when the noise was *about* a decided item. One sentence fixed it:
"report the corrected value; never mention the glitch, not even in Watch out." Zero leaks after.

Along the way about half the "failures" turned out to be scorer bugs. Hana *Park*'s surname matched the
"park it" deferral check. "No consensus" matched the "declared a winner" check. "(Decision 2)" was parsed
as the date Dec 2. A sentence boundary made "Sam: split. Decided Fri Sep 4" look like the split option had
been decided. Every one of these was in the scorer or a key, never in the prompt — which is a useful thing
to know about regex grading: it's cheap and repeatable and it lies about a tenth of the time, so you
re-score old runs every time a key changes.

## My thread found a real gap

Two things came out of Thread E that the synthetic threads hadn't exposed.

First, a rule collision. The organizer removed an option from the finalist list. Someone objected. The
organizer said "Noted" and did not reinstate it. Rule 2 said a decider's word stands; rule 7 said
unresolved disagreement goes under Open questions. Neither said what happens when someone objects *after*
the decision is made — and the model flipped between the two readings on consecutive runs. New rule: an
objection after the decider has spoken doesn't reopen the decision unless they reverse; note it in one
clause under the decision. Four fresh runs, 34/34 each.

Second, my key and Claude's disagreed, and the disagreement was about design, not facts. My key assigned
action items the thread never assigned — "Ron: confirm allergy constraints before booking," "Ben: plan for
late arrival." Sensible things an organizer would infer. Claude's key actively penalized them: the prompt's
hardest rule is *never invent an owner or action*, and inferring next steps is inventing. It was right,
and I'd have graded my own prompt down for following its own rules. That's the value of a second key —
not agreement, but finding where you want two incompatible things.

## Weaker models and the ablation question

The contest didn't say what model the judges would run, so I had Claude hand the same input files to
Sonnet and Haiku subagents. Sonnet was indistinguishable from the default model on C and D. Haiku lost
three to five points on the subtle items — the offline sync, a staffing row — and on my thread it computed
"Wed Mar 12" for a Wednesday that was Mar 11. A wrong date is worse than no date, so the weekday-to-date
rule got a hedge: convert only when certain.

The question I actually wanted answered was whether the 500-word worked example was earning its keep. On
the default model: no. C and D scored identically with and without it. On Haiku: yes, by two to three
points, and the failures without it were exactly the "invented facts" class — AV booking assigned to the
person who merely raised it, a computed due date the thread never gave, a Watch out section on the clean
thread. The example was what taught the weakest model what *not* to make up. Since the contest model was
unknown, keep the example.

## Then the form said 4,000 characters

The prompt was about 14,000 characters. The submission form rejected it with "Keep your prompt under 4000
characters" — a constraint that wasn't in the brief. Even the no-example version was 11,000.

This was a rewrite, not a trim. Claude built a 3,949-character version around the rules the testing had
shown mattered — every rule it kept was one a specific run had failed without — and dropped the example,
the pre-write pass explanation, and most of the prose. The first draft scored 95–99% but leaked in exactly
the places the example used to hold: items ran 40+ words despite the cap (Thread C hit 344 body words),
Unassigned actions got duplicated as open questions, and Watch out appeared on the clean thread to restate
a dependency already in the table. Three sentences replaced what the example had been doing implicitly:
"every Decision and Open item is ONE sentence, max 20 words"; "an Unassigned action is a table row, not
also an Open question"; "Watch out never restates a table dependency; omit for a clean thread."

Final 4k version against the full 14k prompt, default model, three runs each:

| | 4k submission | full prompt |
|---|---|---|
| A | 100% | 98% |
| B | 97% | 98% |
| C | 98% | 98% |
| D | 100% | 100% |
| E | 99% | 100% |

Haiku paid for the missing example — about five points — and the slips were format (dropping a section
header, losing the DECIDED label), not judgment. One more hardening line recovered half of it. There was
no room for more.

## The result

95 / 90 / 95 / 92 / 88 across the five criteria, average 92. Tied for first with two other entries.
The judge's one criticism: "the 'no thread provided' caveat weakens testing" — the bare `Thread:` line at
the end of the prompt, which I'd left as a placeholder. Claude had flagged it as optional. It should have
said delete it.

Ninety-two was also the top score before I entered. Three entries at 92 is a ceiling, not a coincidence.

## Claude's conclusion

I asked Claude what it made of the outcome. Its answer, lightly trimmed:

> We spent the effort making the prompt's *outputs* reliable — a hundred-plus blind runs, hold-out
> threads, weaker models, a scorer that catches invented dates. The judge never ran the prompt. It read the
> text and graded the design against five criteria. Those are different objectives, and the second one
> saturates: once a prompt visibly covers extraction, disagreement, invention, format, and edge cases, an
> LLM judge has little left to separate it from another prompt that also visibly covers them. The
> behavioral work is real and it matters if you actually use this on threads, but it was invisible to the
> scoring.
>
> Two things cost points that I can see. The lowest mark, 88 on edge cases, is almost certainly the
> compression — the 14k version had a full no-decision section and a deferral rule with examples; the 4k
> version has one dense paragraph. The judge rewards visible completeness, and that's where the cuts landed
> hardest. And the `Thread:` line is on me.
>
> If I'd been thinking about this correctly, the last step should have been to test against the judge,
> not just the keys — give a Claude instance the five criteria, have it score each candidate version, and
> see which one it preferred. We had the machinery for that and didn't point it at the right target. The
> contest scored the prompt as an artifact; we tested it as a tool.

I think that's right, and I'd add one thing. The testing didn't produce a higher score, but it produced
things a higher score wouldn't have: a rule about post-decision objections that I'd never have thought to
write, a measured answer to "does the example matter" instead of a belief, a specific reason not to let a
weak model convert weekdays to dates, and a folder with five threads, five keys, and a scorer that can
grade the next prompt in an afternoon. If the point of the exercise was a leaderboard position, I
over-invested. If the point was a prompt I'd trust on a real thread, I don't think there was a shorter
path.

## The prompt

This is the submitted version, 3,949 characters.

```
Summarize the thread below for an executive who wasn't in it and has 60 seconds. Report current state, not the story. Read it all first.

HOW TO READ IT
1. Last word wins. A decision made, reversed, and re-made counts only in its final form; a corrected date counts only as corrected, even if someone later repeats the old one. Never hedge between old and new.
2. A decision exists when someone with standing states it and nobody overrides it ("done unless objections by EOD" counts; "I like that" doesn't). A senior drive-by later walked back is closed. An objection after the decider has spoken doesn't reopen it unless they reverse; note it in one clause under the decision.
3. A deferral ("park it," "revisit after X") is not a decision -> Open questions; Unassigned if nobody owns the revisit.
4. Edited text is authoritative. Skip a deleted message; keep commitments made in replies to it.
5. Collapse duplicates: one question asked three times is one item.
6. "Someone should do X" is Unassigned, not the speaker's, unless it lands on someone later.
7. Disagreement resolved (concession, scope change) -> outcome only. Never resolved -> Open questions, one line per position, no side taken.
8. "Let's take this offline" that never comes back stays open even if the topic was later decided: an Open question AND a table row for whoever owes the report-back.
9. Bots aren't the source of truth. If a human says "ignore that," report the corrected value; never mention the glitch, not even in Watch out.
10. Anything pending an outside answer (legal, vendor, a sync) gets ⚠ plus what's pending.

IGNORE ENTIRELY - never mention or acknowledge it
Greetings, jokes, emoji-only messages, photos, lunch, offsite, pets; other projects unless they create an action here; ticket metadata unless a human said it matters; jabs and grievances; out-of-office unless it moves a date; tone, who was "right"; process talk unless it produced a scheduled forum; anything a human said to disregard.

HARD RULES
- Never invent an owner, date, number, or reason. Write "Unassigned" and "No date" literally; figures in a linked doc -> "in linked doc (not in thread)". Convert a weekday to a date only when timestamps make it certain.
- Hard cap 250 words outside the table; aim 150-200. Every Decision and Open item is ONE sentence, max 20 words; reasons and positions get a clause, not a sentence. No recommendations. Unsure if decided -> Open.
- An action item is a table row, never a Decision; an Unassigned action is a table row, not also an Open question.

OUTPUT - always these headers, this order. Write "None" under an empty one (omit only Watch out). Status starts with one of the three labels.

**Status:** DECIDED / PARTIALLY DECIDED / NO DECISION - one sentence: state, most urgent item.

**Decisions (final state)** - numbered. What was decided; only if it changed, what it replaced and why. ⚠ if contingent.

**Open questions** - numbered. The question; positions if disputed; who owes the answer; by when.

**Owners & next steps** - table `Owner | Action | Due`. One row per deliverable, final owner only; joint owners only if named jointly. Dated first.

**Watch out** - max 3 bullets of max 12 words: only what a skimmer would get WRONG (a moved date, an edited decision). Each points at an item above; never restates a table dependency; omit for a clean thread.

IF NO DECISION WAS REACHED
Replace Decisions with: **Options on the table** (each live option, one line, who backs it; no ranking, no "leaning"); **Why it's stuck** (the crux); **Decision forum** (when/where/who, or "None scheduled"); **Cost of not deciding** (only if stated). Positions go under Options only. A meeting to decide is a next step, not a decision.

BEFORE ANSWERING
Privately list 2-3 ways a skimmer could misread this thread; check your draft against them. Then: dates survived corrections? Open items resolved later? Deferrals or actions filed as decisions?

Thread:
```

If you use it, paste your thread after that last line — or delete the line and tell Claude to go get the
thread from Slack. Either way, drop the `Thread:` placeholder before you enter it in a contest.
