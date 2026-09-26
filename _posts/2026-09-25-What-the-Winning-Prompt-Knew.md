---
layout: post
title: What the winning prompt knew — reading a 100 next to my 97
---

The [last post](/2026/09/23/Kick-Off-a-Program-Calibrating-the-Judge.html) ended with my kickoff-email
prompt at 97 and an experiment showing that fixing the judge's three named deductions wouldn't have moved
the score. It couldn't say what would have. Now I have something better than a guess: Ben, a coworker,
scored a perfect 100 on the same challenge, and he's let me print his prompt. It's at the end of this
post.

I paired with Claude again. It read the two prompts side by side, reran the neutral judge panel from the
last post on both, and ran both prompts on two sets of program details so we could compare the emails
they produce. The short version: the two prompts are built on different ideas of what "context" means,
our stand-in judge sides with mine, the real judge sided with Ben's, and on actual inputs each prompt wins
the scenario that was built around its own rules.

## Two prompts for the same email

On the surface they're close. Both are just under 4,000 characters. Both cast Claude as the client's main
contact, fix the structure (what's next, timeline, owners, one ask), set a word limit, ban invented dates
and names, use bracketed placeholders, and end with a "Before you send" list for the sender.

The differences are in what each prompt spends its characters on.

**Mine spends them on a scenario and a template.** Roughly a quarter of it is a filled-in fictional
program ("Psoriasis in Focus", a made-up sponsor, ten milestones) with planted snags: a TBD time, an
unknown reviewer, two teams disagreeing on a date, `Ask: not chosen`. The rest is a compressed spec: an
exact output shape with `//` notes, a fallback chain for picking the ask, rules for conflicts, and a
silent self-check.

**Ben's spends them on knowing the job.** There's no scenario. Instead there's a list of the inputs he'll
supply, then a paragraph headed "How a program actually comes together": setup, the content package from
the client, the content build, the promotion plan, review and launch, reporting. One line in it carries a
dependency: "Nothing in step 3 can start without this." Then the structure, the tone and the hard rules,
all in full sentences.

Reading them next to each other, five things stand out.

**Context as knowledge, not data.** The rubric rewarded "a clear role and program/client context." I read
that as *give it a program*, and Claude agreed, which is why mine carries a fictional one. Ben read it as
*tell it what a senior person knows about how programs run*. His context is true of every program, not
just one invented one, so the prompt is reusable as written. It also sidesteps the problem I wrote about
last time: a judge reading my planted snags saw holes in the prompt, not proof that the rules worked.
Ben's prompt has nothing unresolved in it for a judge to find.

**Rules that carry their reasons.** Ben's ask rule is "Unless my inputs say otherwise, the ask is the
content package from step 2, requested as one deliverable with one date, because it is what gates
everything downstream." Mine is "Use the Ask detail if given; else the client's earliest-dated item;
none: kickoff-call availability." Both pick an ask. His says why, so a reader, human or model, can
extend it to a case it doesn't cover. Mine is a lookup table, and last time's judge read the compressed
chain as two options rather than a fallback. The same pattern runs through his whole prompt: "Readable on
a phone in about 60 seconds." "The client should never need a glossary." "Promotion is where kickoff emails
most often over-promise."

**Failure modes from the job, not from the rubric.** Ben has three rules I never thought of:

- no budget, fees, contract terms "or anything from the sales process"
- promotion only "as a step we will confirm together," with no channels, send dates, audience sizes or
  registration targets
- "Do not assign a task to a client-side person unless my inputs say they agreed to it."

None of those is in the rubric. They're where a real kickoff email goes wrong. My guardrails were the
general ones, like never invent a name, date or number. His are the specific ones.

**The "or ask" taken literally.** The rubric said to "leave clear placeholders *or ask*." Last time I lost
my Role point for a "never stall" rule that read as the opposite of permission to ask. Ben's has a
one-line threshold: "If I gave you no program name or client name at all, stop and ask for those two
before drafting." It covers both halves of the "or" and costs twenty-odd words.

**A declared interface instead of a blank.** My first contest entry lost points for an empty `Thread:`
line at the end. Ben's prompt also has no data in it, but it opens with "Inputs I will give you (use
exactly what I provide; do not invent anything I leave out)" and lists six of them. That reads as a design
decision rather than something missing.

What Ben's *doesn't* have is just as useful to know. It has no worked example, which two earlier judges
docked me for. It has no self-check. It has no rule for two inputs that disagree. It scored 100 anyway.
So "write the worked example," the lesson I underlined twice in the second post, was one judge's note,
not a law.

## The stand-in judge prefers mine

The last post showed the neutral panel couldn't order my own two entries. Ben's 100 gives it a third known
score to get right. Claude reran the same setup: the challenge text and scoring criteria, the six category
maximums, "return feedback, per-category scores with a comment, and a total," and nothing about the
leaderboard. Five fresh reads per entry: Haiku twice, Sonnet twice, Opus once.

| entry | real score | Haiku | Sonnet | Opus | mean |
|---|---|---|---|---|---|
| Ben's kickoff prompt | 100 | 100, 95 | 95, 96 | 94 | 96.0 |
| my kickoff prompt | 97 | 100, 100 | 98, 98 | 93 | 97.8 |
| my clinical-summary prompt (last post's run) | 99 | 97, 99 | 94, 94 | 90 | 94.8 |

The panel puts the real third place first. Sonnet, the steadiest reader, averages 98, 95.5 and 94, which
gets one of the three pairwise orderings right. My kickoff prompt's Sonnet and Opus reads matched the
last post's run exactly (98, 98 and 93), so the preference is stable, not one lucky read.

Then Claude asked Sonnet the question head-on: here are both entries, score each, say which is better. It
ran it twice with the order swapped.

| shown first | Ben's | mine | Ben's Context | my Context | preferred |
|---|---|---|---|---|---|
| Ben's | 79 | 95 | 9/17 | 17/17 | mine |
| mine | 78 | 96 | 8/17 | 17/17 | mine |

Side by side, the gap goes from about two points to sixteen and eighteen, and most of it is in one
category. From one of the reads: "No actual program or client is supplied — 'Inputs I will give you'
lists categories only... which costs it heavily against a criterion that explicitly rewards
program/client context."

That's the reading of the rubric Claude and I made when we chose the filled scenario, now coming back from
a judge as a verdict. The real judge disagreed with it. The last post's judge sheet gave my Context 17/17,
and Ben's 100 means his got full marks too. The real judge evidently took his process paragraph as
context. Our stand-in didn't.

There's a caveat I can't get around. My judge brief paraphrases the challenge text, and the real judge may
have had fuller wording about what "context" means. That's the point, though. A Claude judge built from my
reading of the rubric shares my reading of the rubric. It can find contradictions in a prompt, which is
what made it useful in the second contest. It can't tell me I've misread what's being asked, because it
misreads it the same way.

## On real inputs, each prompt wins its own scenario

The judge scores text, but the prompts are for writing emails. So Claude ran both prompts, two Sonnet runs
each, on two sets of details.

- **Scenario 1** was my psoriasis program, snags and all. For Ben's prompt, the details went in as his
  inputs.
- **Scenario 2** was new: an on-demand CME program with one agreed client deliverable, three other open
  requests, a procurement contact with nothing agreed, a kickoff "sometime next week," and a sales note
  promising 2,000 registrations, a 3-email series and a budget figure. Claude wrote it after reading Ben's
  prompt, so it tests his rules the way scenario 1 tests mine.

An Opus grader got the program notes and the four anonymized emails for each scenario, but not the
prompts. It played an experienced manager checking the drafts before one goes out. It listed every
invented or misstated fact, every client-relations risk, and every miss in the Before-you-send list, then
scored each draft from 1 to 10 on "would I let this go out after filling in the placeholders."

| scenario | my runs | Ben's runs | grader's ranking |
|---|---|---|---|
| 1, psoriasis (built for my rules) | 7, 6 | 5, 4 | mine, mine, Ben's, Ben's |
| 2, heart failure (built for his rules) | 4, 3 | 7, 5 | Ben's, Ben's, mine, mine |

That's a clean split, on two runs per cell and one grader read, so it's a small sample. The failures are
more useful than the scores, because every one of them traces to a line in a prompt.

**Mine, on scenario 2.** Both runs gave the client every open request anyway. The timeline had "Oct 23 -
Faculty headshots due (You)", the owners list had "You - content package, faculty headshots, logo
files", and the procurement contact got "Tess Okafor, Procurement - PO". The "One thing we need from you"
paragraph still held a single ask. I'd written "Other client items read as schedule, not requests," which
works when every client item is agreed and turns into four asks when they aren't. Both runs also invented
an owner for the kickoff call: "Jordan Kim, our PM, will get a kickoff call on the calendar for next week."
Nothing in the notes gave the kickoff call an owner. My silent self-check says "each milestone has date and
owner." When a milestone doesn't have one, the check makes one up. That's the third time in this series a
checklist or example of mine has produced content (the second post's "bullet 2 carries a real harm," the
last post's "here's how we'll get there"). One run also marked two dates "(hard stop)" that the notes never
called hard stops. The template's example of a hard-stop line was enough. Neither run's Before-you-send
list mentioned the sales numbers.

**Ben's, on scenario 2.** Both runs set the headshots, logos and PO aside for the sender, gave the
procurement contact no task, and kept the sales figures out of the email. One run added a warning: "Dana
may reference those figures back to you before you're ready to commit to them — worth aligning internally
before the promotion conversation." That's the kind of note a senior colleague would write in the margin.
It isn't perfect. One run dropped the kickoff call from the kickoff email entirely, because "Confirmed
dates only" beat the placeholder rule, and the same run put "Sept 22 — Contract signed" in the timeline.

**Ben's, on scenario 1.** Here the process paragraph works against him. All four of his runs, across both
scenarios, included some form of "we confirm the promotion plan together," and two listed faculty bios and
learning objectives among the build steps. Those steps come from his default sequence, which looks shaped
for an accredited on-demand program, with its pre- and post-tests and certificates. The psoriasis program
is a sponsored live broadcast. The grader, which had only the notes, flagged the steps as invented in
both scenarios. From inside Medlive, "we'll confirm promotion together"
may just be true of every program. Learning objectives on a sponsored broadcast probably aren't. Two
runs, one per scenario, also gave the client duties nobody had agreed to ("approvals and day-to-day
decisions", "program sign-off"), which breaks his own rule, and one leaned on brand language: "Glad to have Solvera's story in the mix this quarter."

**Mine, on scenario 1.** Both runs were accurate on the dates, gave every gap a placeholder, and turned the
replay-date conflict into a question for me. The grader's main complaint was that both assigned the second
speaker's approval to the client as "(You)" when the notes said the sponsor. That's a second ask, sitting
in the timeline.

So neither prompt is better on every input. Each one handles the failures its author thought of. My
scenario came from my rules, and the test kits in the earlier posts were built alongside the rules they
tested, so they could only find the failures I'd already imagined. Ben's three domain rules would never
have fired on my test kit, because it had no sales notes, no unagreed client tasks and no extra asks. They held on the
first input that had them.

## What I'm taking into the next one

**Write the prompt as a brief to a senior colleague.** Full sentences, and a reason on every rule that
needs one. The reason costs a few words and does two jobs: it tells the model how to extend the rule, and
it tells a judge the rule was thought through.

**Put knowledge in the context, not a scenario.** What does someone who does this job know that Claude
doesn't? For this task it was the order a program comes together in and which step gates the rest. Declare
the inputs and leave the data out. It makes the prompt reusable, and it removes the test fixtures a judge
would otherwise read as gaps.

**Name where this task goes wrong in real life.** The rubric lists what gets scored. It doesn't list where
the email actually burns a client relationship: sales numbers, over-promised promotion, tasks nobody agreed
to. That list comes from the job. If it isn't your job, ask someone whose job it is before writing a rule.

**Answer every clause of the rubric literally.** "Placeholders or ask" needs both halves. A one-line
condition under which to ask cost Ben twenty-odd words and cost me a point by its absence.

**Build at least one test input from someone else's worldview.** A corpus written from your own rules
tests whether the rules do what you meant, not whether you've got the right rules. The cheapest version is
a scenario written by someone else, or by Claude after reading a different prompt, as scenario 2 was here.

**Stop using the stand-in judge as a target.** Three known scores, and it ordered them wrong twice. Keep it
for what it was good at in the second contest: an adversarial read for contradictions and unsatisfiable
rules. Don't treat its category scores as the real judge's.

**Audit self-checks and examples for invention pressure.** Every "each X has a Y" check and every example
line gets read as a requirement. If a real input can lack the Y, the check needs "or a placeholder."

## Claude's conclusion

As before, I asked Claude what it made of it. Lightly trimmed:

> Last time I recommended the filled-in scenario because the rubric scores program/client context. The
> pairwise judge gave that same reasoning back, nearly word for word, as grounds for scoring Ben's prompt
> 78. That's the finding I'd keep. A Claude judge prompted with my reading of the rubric will agree with my
> reading of the rubric, so it can't catch the one error that matters most: misunderstanding what's being
> asked. Ben's prompt didn't beat mine on anything my judge was measuring. It beat mine on what "context"
> means, and my judge and I were wrong about that together.
>
> The emails say something similar about testing. I built the second post's corpus and last post's
> scenario from the prompt's own rules, so they checked whether the rules worked as intended. Ben's prompt
> contains three rules that don't come from the rubric or from testing. They come from knowing how
> kickoffs go wrong with a pharma client, and the first scenario that exercised them separated the two
> prompts by two and a half points on a ten-point scale. That knowledge was the input neither of us had.
>
> I don't want to overcorrect. Ben's prompt isn't better on every input. Its process paragraph put CME
> build steps into a sponsored broadcast, and "confirmed dates only" dropped the kickoff call from a
> kickoff email. And I can't see the real judge's sheet for his entry, so why it scored 100 is inference.
> What the evidence supports is narrower: where Ben spent his characters, on what the job knows, was worth
> more than where I spent mine, on proving the rules to a reader who only reads once.

I think that's right. Three contests in, the lesson I keep relearning is that the tests I build can only
find what I already suspect. Last time it was the judge I hadn't calibrated. This time it's the job I
don't do. Ben's prompt reads like it came from someone who knows how these programs run, and that turned
out to be most of the difference.

## Ben's prompt

The winning entry, printed with Ben's permission. It's just under 4,000 characters.

```
You are a senior client services lead at a medical education company that produces sponsored programs (live broadcasts, on-demand content, and digital campaigns) for pharmaceutical and biotech clients. A new program has just been approved and you are the client's main point of contact. Draft the kickoff email that opens the relationship: the note that makes a busy brand manager feel the program is already in motion and in competent hands.

Inputs I will give you (use exactly what I provide; do not invent anything I leave out):

Client name and primary contact(s)

Program name and format

Confirmed dates and milestones

Our team members and their roles

Client-side responsibilities already agreed

The single next action we need from the client, and its due date

How a program actually comes together, for your reference (use this as the default sequence when my inputs don't specify steps; it is an order, not a set of dates):

Program setup: we set up the program and its pages.

Content package from the client: agenda, faculty roster, and course materials (pre- and post-test questions, certificate details). Nothing in step 3 can start without this.

Content build: faculty bios, learning objectives, materials, and visuals go live in the program.

Promotion plan: how the program reaches its audience gets confirmed jointly before any traffic is turned on.

Review and launch: client sign-off, final quality check, go-live.

Reporting after launch.

Structure, in this order:

Subject line: specific and calm, naming the program. No exclamation points.

Opening (2 sentences max): warm thanks and one line showing you understand what this program is meant to do for them.

What happens next: 3 to 5 numbered steps, each one line, starting with what we do this week. Draw from the sequence above; when trimming, drop the later steps first.

Timeline and key milestones: a short dated list, earliest first. Confirmed dates only.

Who owns what: two short groups, "Our team" and "Your team," each with name, role, and one-line responsibility. Name yourself as the single point of contact.

One ask: a single, specific request with a due date, in its own short paragraph. Unless my inputs say otherwise, the ask is the content package from step 2, requested as one deliverable with one date, because it is what gates everything downstream.

Close: one warm sentence and your signature block.

Tone: confident, warm, plain. Write like a person who has run fifty of these, not a template. No "I hope this email finds you well," no jargon, no marketing adjectives.

Hard rules:

Under 300 words excluding subject line and signature. Readable on a phone in about 60 seconds.

Address the email to the primary contact; put other client contacts on cc in the header, not in the greeting.

No internal tool names, project codes, or process language. The client should never need a glossary.

Never commit to a date, deliverable, or name that is not in my inputs. Where something is missing, insert a bracketed placeholder in this exact style: [CLIENT CONTACT NAME], [KICKOFF CALL DATE], [CONTENT PACKAGE DUE]. Do not write around a gap or guess a plausible value.

Promotion is where kickoff emails most often over-promise. Mention it only as a step we will confirm together on a date; never name channels, send dates, audience sizes, or registration targets unless they are in my inputs.

Do not reference budget, fees, contract terms, or anything from the sales process.

Do not assign a task to a client-side person unless my inputs say they agreed to it.

One ask only. If my inputs contain several, keep the one that unblocks the earliest milestone and set the rest aside for me (see below), not for the client.

After the email, add a short section headed "Before you send," listing every placeholder you inserted and anything you set aside (extra asks, unconfirmed dates, ambiguous owners). If I gave you no program name or client name at all, stop and ask for those two before drafting.
```

Like my entries, it ends where your details go. Unlike mine, it tells you what they are.
