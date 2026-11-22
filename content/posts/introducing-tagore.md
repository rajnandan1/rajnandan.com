+++
title = "tagore: A Claude Code Plugin for the Prose Other People Read"
date = 2026-11-22
description = "tagore is a Claude Code plugin, inspired by Oskar Austegard's declauding skill, with a skill and a Stop hook for docs and an output style for replies, so that the docs and replies Claude writes make sense on one read. With real runs of the skill, the newcomer read and the output style, and what each one missed."
[taxonomies]
tags = ["ai", "agents", "open-source", "writing", "python"]
[extra]
toc = true
+++

[tagore](https://github.com/rajnandan1/tagore) is a Claude Code plugin for the prose that Claude writes for other people: PR descriptions, commit messages, READMEs, Slack messages, and the replies Claude writes to you in the session. It has three parts. The `fair-copy` skill rewrites the sentence patterns that make text read as model-written. A Stop hook, which is a command that Claude Code runs each time Claude tries to end a turn, makes a subagent read the prose that changed in `.md` and `.html` files during the turn. A subagent is a separate Claude that Claude starts for one task, and this one never saw the earlier version of the text. An output style, which is a file of instructions that Claude Code sends with every request, makes Claude put the answer in the first sentence of each reply. The output style comes with a small reminder hook that repeats its main rules on each message you send.

Claude drafts a lot of what lands in my repositories now, and its drafts share habits. A header states a verdict where a label should be. A paragraph ends on a slogan that repeats the paragraph. A README written right after a bug fix explains how the bug was fixed, when the reader needs to know how the feature behaves. A status reply opens with what Claude did and reaches the result in the fourth sentence. I use tagore to catch these before the text leaves the session.

The `fair-copy` skill, its scripts and the Stop hook are inspired by [declauding](https://github.com/oaustegard/claude-skills/tree/366e2f2c61fbd4291774ae4d35f32bf66e3ee2e9/declauding), Oskar Austegard's skill for rewriting model-written prose. tagore adds the plugin packaging, the `straight-talk` output style with its reminder hook, rules ported from two other skills of that kind (blader/humanizer and Aboudjem/humanizer-skill), a skip list for files written for agents, two writing samples from Claude Fable 5.2 that back the skill's notes on that model, and the docs at [tagore.rajnandan.com](https://tagore.rajnandan.com/).

The name comes from Rabindranath Tagore, the Bengali poet who won the 1913 Nobel Prize in Literature. Bengali script hangs most letters from one line along the top, called the matra, and the tagore logo hangs its letters from one red line in the same way. The red is laterite, the red soil of Santiniketan, where Tagore built his school, and it is also the colour of an editor's pen.

Install it in Claude Code:

```
/plugin marketplace add rajnandan1/tagore
/plugin install tagore@tagore
```

The current version of the plugin is 0.4.0. tagore has two hooks: the Stop hook, and a reminder hook that comes with the output style. Both need `python3`, and the Stop hook uses `git` to find the files that changed in a git repository. The skill and both hooks are on after install, and the reminder hook stays silent until you turn the style on. The output style stays off until you run `/output-style tagore:straight-talk`. To turn all of tagore off, run `claude plugin disable tagore@tagore`.

## Replies and docs

tagore sorts what Claude writes into two kinds. A reply is text that Claude writes to you inside the session: an answer, a question, a status summary at the end of a turn. A doc is prose for a reader outside the session: a PR description, a commit message, a README, a Slack message. A draft that Claude shows you before it is posted is a doc too. The two kinds go wrong in different ways, so each gets its own part.

| Part | Text it works on | When it runs |
| --- | --- | --- |
| The `fair-copy` skill | Docs | When you ask for it, and by itself when the task is prose for a reader |
| The Stop hook | `.md` and `.html` files that changed during the turn | Each time Claude tries to end a turn |
| The `straight-talk` output style | Replies | With every request, while the style is on |
| The reminder hook | Replies | Each time you send a message, while the style is on |

## The fair-copy skill on an Opus 5 draft

Claude loads the skill by itself when the task is prose for a reader, such as a PR description or a README. You can also say "de-claude this" or "this reads like AI", or run `/tagore:fair-copy`. Claude first reads the whole draft, notes any contradictions in it, and writes down the one point it must make. Then four checks run, in this order:

1. `prose_lint.py`, a regex linter, flags known patterns.
2. `prose_review.py` sends the parts of the draft that a regex cannot judge to a model for review.
3. Claude reads every sentence, including the ones that nothing flagged, and rewrites the ones that perform a point instead of stating it.
4. `prose_diff.py` checks that the rewrite kept every fact of the draft.

The skill calls a sentence of the kind in check 3 staged. A staged sentence is built so that the reader feels a finding arrive, as with a reveal after a dash or a slogan that ends a paragraph, instead of stating the finding.

To show the checks, I ran them on a real model draft from the `evidence/` folder of the tagore repository. `opus-5-a.md` is a blog post called "Why we removed our cache" that Claude Opus 5 wrote with no style instruction. It comes from Oskar Austegard's model-register-drift experiment, which had several Claude models write the same post and counted the patterns in each. It opens like this:

> In March we put a read-through Redis cache in front of the hottest query path in `ledger-api`. In July we took it out. Median latency improved both times we touched it — that was the trap.

### The linter

`prose_lint.py` scans the draft with regular expressions and prints line numbers. On this draft it printed 12 candidates in 6 categories. Two of the categories:

```
[significance]  (2)
     L41  The real fix
          the real/actual X
     L53  the actual heuristic
          the real/actual X

[header]  (3)
      L1  Why we removed our cache
          coy header — could top three different sections; name the content
     L15  A cache doesn't make things faster. It makes some things faster and everything else slower
          thesis-shaped header — states a verdict instead of labelling
     L39  What we should have measured first
          coy header — could top three different sections; name the content
```

"The real fix" and "the actual heuristic" tell the reader that a thing matters instead of saying what it is. The L15 header is a two-sentence verdict. The other four categories flagged two lists of three, a participle at the end of a sentence, the em dash in the opening paragraph, and three density counts: 1.0 em dashes per 150 words, the phrase "in front of" used 3 times, and a count of words that are common in a fast-growing cluster of GitHub pull request descriptions.

The linter missed the last sentence of the section that the L15 header opens:

> That generalizes to a rule worth writing on a wall: **a read-through cache can only improve your tail if it also makes misses cheaper.** It doesn't. It makes them more expensive by exactly the cost of the cache round trips. Read-through caching is a p50 optimization sold as a latency optimization.

The last sentence repeats the paragraph as a slogan. A regex cannot tell a slogan from a fact. The skill says as much: its instructions in `SKILL.md` put the linter at roughly two thirds of what a careful pass finds (Austegard's estimate), and they say that a clean report is not evidence that a draft is clean, least of all on Opus 5 drafts, which stage their points in shapes that the linter cannot see.

### The structural review

`prose_review.py` covers part of the rest. It pulls out the slots that a regex cannot judge: every header, the opening sentence, the closing sentence of the post and each one-sentence paragraph. It sends only those slots to a model, with the entries about headers, closers and isolated lines from the skill's catalogue of patterns, `references/register.md`. With `GEMINI_API_KEY` set it calls Gemini, with `ANTHROPIC_API_KEY` set it calls Claude, and with neither it prints the prompt for Claude to hand to a subagent. I had no key set, so a Sonnet 5.5 subagent got the 9 slots of this draft (6 headers, the opening sentence, the closing sentence and one one-sentence paragraph) and returned 7 findings. This is the one on the closing sentence, where `n` is the slot number:

```json
{"n": 8, "tic": "Aphoristic closer", "why": "The closer compresses the piece into a quotable contrast ('doesn't buy speed; it buys...') and ends on a moral line ('your users only ever feel the slow ways').", "fix": "A network hop in front of Postgres adds a second system that can be slow."}
```

The other six findings were five section headers and the line "The numbers, at a steady 6,200 rps:", which stands alone as a paragraph before a table.

The review did not see the slogan about p50, the median latency, either. The script sends only the closing sentence of the whole post, and the slogan closes a section. That leaves it to check 3, in which Claude reads every sentence and asks whether the sentence says the thing or performs having had the thought. The skill has a profile file per model family that says where that family stages its points. The profile for Opus 5 says to read the last sentence of every paragraph, and it quotes this exact sentence as its example. Austegard wrote that profile from the samples of this same experiment, so this run is not an independent test. It shows how the checks fit together.

### Before and after

In check 3, Claude rewrote the cache section of the draft. This is the section before the pass:

```markdown
## A cache doesn't make things faster. It makes some things faster and everything else slower.

Our hit path was a Redis `GET` plus JSON decode: ~1.3 ms. Our miss path was `GET` (0.4 ms) → Postgres (5.2 ms) → JSON encode (0.9 ms) → `SETEX` (0.4 ms) ≈ 6.9 ms. So a miss cost about 33% more than the uncached query it replaced.

Now do the percentile arithmetic. With an 87% hit rate, every request above the 87th percentile is a miss. Your p99 is no longer the 99th percentile of *all* requests — it's roughly the 92nd percentile of the *miss* distribution, and the miss distribution is strictly worse than the uncached one.

That generalizes to a rule worth writing on a wall: **a read-through cache can only improve your tail if it also makes misses cheaper.** It doesn't. It makes them more expensive by exactly the cost of the cache round trips. Read-through caching is a p50 optimization sold as a latency optimization.
```

And this is the section after it:

```markdown
## Hit and miss latency

Our hit path was a Redis `GET` plus JSON decode: ~1.3 ms. Our miss path was `GET` (0.4 ms) → Postgres (5.2 ms) → JSON encode (0.9 ms) → `SETEX` (0.4 ms) ≈ 6.9 ms. So a miss cost about 33% more than the uncached query it replaced.

With an 87% hit rate, every request above the 87th percentile is a miss. The p99 of all requests is then roughly the 92nd percentile of the miss distribution, and the miss distribution is strictly worse than the uncached one.

A read-through cache can only improve your tail if it also makes misses cheaper. A read-through cache makes each miss more expensive by exactly the cost of the cache round trips.
```

The verdict header became a label. The pass cut "Now do the percentile arithmetic", the em dash, "a rule worth writing on a wall", "It doesn't." and the p50 slogan. The numbers and the claims stayed, and the next check tested that. One error also stayed. Both versions say that a miss costs more "by exactly the cost of the cache round trips", but by the draft's own numbers a miss adds 1.7 ms to the 5.2 ms query, and only 0.8 ms of that is the two Redis calls. The other 0.9 ms is the JSON encode. The skill tells Claude to read the whole draft before any check and note contradictions like this one, and in this run Claude did not.

### The fact check on the rewrite

A rewrite that removes staging can also remove a claim, and the result still reads well. `prose_diff.py` compares the draft with the rewrite and lists the numbers, names, quotes, links and hedges that the rewrite lost or invented. On the rewrite above, the script flagged one false positive, which I describe below. With that finding waived, it reported "every claim survives, nothing invented". Then Claude made a copy of that rewrite with two losses on purpose: "~1.3 ms" became "about a millisecond", and "strictly worse" became "worse". With the original section as `source.md` and that copy as `lossy.md`, it printed:

```
source.md -> lossy.md: 2 finding(s)

  [LOST ] number       '1.3'  source 1 -> rewrite 0
          a number in the source is not in the rewrite — every figure, date and version must survive
  [ADDED] proper       'Hit'  source 0 -> rewrite 1
          in the rewrite and nowhere in the source — the rewrite must not invent a fact, name, number, date or citation
```

It caught the lost number. It did not catch the lost "strictly". The script counts scope words such as "all", "every", "only" and "never", and "strictly" is not in its list, so that loss would reach the reader. The `Hit` finding is the false positive: the script took the first word of the new header, "Hit and miss latency", for a new name. The script tells you to treat each finding as a question, and `--waive Hit` clears it.

## The Stop hook and the newcomer read

The four checks miss one failure: text that leans on an earlier version that the reader never saw. The README habit from the start of this post is one case: a README written right after a bug fix explains the fix to a reader who needs to know how the feature behaves. Claude cannot catch this in its own text, because Claude saw the old version. The newcomer read catches it, because a subagent that never saw the old version reads the new text.

The Stop hook runs each time Claude tries to end a turn. It looks for `.md` and `.html` files that changed during the turn and counts only visible prose, so code blocks, inline code, tags and comments do not count. The hook blocks the end of the turn when two things are true. First, 20 or more characters of prose were added, removed or rewritten in total across those files. Second, since the last change to those files, Claude has not sent a subagent the newcomer-read prompt, which starts with `FRESH-EYES READ`. The hook then writes a packet, a Markdown file with the changed text, to `/tmp/fresh-eyes/` and gives Claude the prompt to send. For a Markdown file the packet holds each changed section whole, and for an HTML file it holds the visible text within 25 lines of each change. The subagent runs on Sonnet 5.5, reads only the packet, and lists each phrase that a newcomer would not understand. Claude fixes each phrase or says why it stays.

The hook blocks at most once per turn, and if the hook itself fails, it lets the turn end. It skips changelogs, generated pages, `CLAUDE.md`, everything in `.claude/` and other files written for agents. [The skip list](https://tagore.rajnandan.com/skipped-files/) has the rest. It does not see a doc that is never saved to a file, such as a PR description that Claude posts with `gh`.

I ran the newcomer read on this post while Claude and I wrote it, with the script and the prompt that the hook uses. The first read of the draft flagged that its description and opening paragraph used "Stop hook", "output style" and "subagent" without saying what they were. This was the opening before the read:

> [tagore](https://github.com/rajnandan1/tagore) is a Claude Code plugin for the prose that Claude writes for other people: PR descriptions, commit messages, READMEs, docs, and the replies Claude writes to you in the session. It rewrites the sentence patterns that make text read as model-written. It makes a fresh subagent read every doc that changed before Claude can end its turn. And it gives Claude an output style that puts the answer in the first sentence of each reply.

And this is the opening after it:

> [tagore](https://github.com/rajnandan1/tagore) is a Claude Code plugin for the prose that Claude writes for other people: PR descriptions, commit messages, READMEs, Slack messages, and the replies Claude writes to you in the session. It has three parts. The `fair-copy` skill rewrites the sentence patterns that make text read as model-written. A Stop hook, which is a command that Claude Code runs each time Claude tries to end a turn, makes a subagent read the prose that changed in `.md` and `.html` files during the turn. A subagent is a separate Claude that Claude starts for one task, and this one never saw the earlier version of the text. An output style, which is a file of instructions that Claude Code sends with every request, makes Claude put the answer in the first sentence of each reply. The output style comes with a small reminder hook that repeats its main rules on each message you send.

A later read flagged that the section on the output-style test counted four runs, although I had made six. That section now says that the table keeps four of the six runs.

## The straight-talk output style

`straight-talk` is the output style, and it covers replies. With it on, Claude puts the answer, the result or a warning in the first sentence. It keeps each number with its unit and its scope, says what it did not check, marks a guess as a guess, defines each new term at first use and keeps one name for one thing. It asks at most one question per sentence and ends with the question that blocks the work or with one next action. It uses no em dashes and no decorative emoji, and bold marks only the label at the start of a list item. A custom style drops Claude Code's built-in coding instructions unless its file sets `keep-coding-instructions: true`. This file sets it, so Claude codes the same way as before.

Nothing enforces a style, and in a long session Claude can drift away from it. So the style comes with a reminder hook, which adds this line to each message you send while the style is on:

```
straight-talk is on: answer first, keep numbers and conditions exact, define each new term, and finish with one next action.
```

### With it and without it

I asked Claude the same question in `claude -p` sessions on Sonnet 5.5. `claude -p` runs one prompt and prints the reply, without the interactive interface. Two sessions used the `default` output style, which means `straight-talk` was off, and two used `tagore:straight-talk`. Each session loaded tagore from a clone, with no tools and no other plugins. The question gave a nightly backup log in which `pg_dump` failed with "No space left on device", `df` showed the backup disk 97% full, and `/backups` held 14 entries under a policy that keeps 7 backups. It asked what went wrong and what to do. The table has four runs, which cost $0.098 in total. I also made two earlier runs without the style, and the last paragraph of this section says why the table leaves them out. In the table, a header line is a `##` heading or a line that holds only bold text, and a bold span is any run of bold text.

| Run | First line of the reply | Words | Header lines | Bold spans | Cost |
| --- | --- | --- | --- | --- | --- |
| default 1 | `**What probably went wrong**` | 391 | 3 | 5 | $0.016 |
| default 2 | `## What likely went wrong` | 504 | 3 | 12 | $0.018 |
| straight-talk 1 | The disk filled up during the dump because `/backups` holds 14 entries instead of the 7 your policy allows, so the cleanup step is not removing old backups. | 398 | 0 | 3 | $0.034 |
| straight-talk 2 | The backup failed because /backups filled up (97 GB of 100 GB used, 3.0 GB free) while `pg_dump` was writing, and the disk is full because the retention policy is not keeping it at 7 backups. | 349 | 0 | 2 | $0.030 |

These are the start and the last line of two of the replies, copied as Claude wrote them. Without the style (default 2):

```markdown
## What likely went wrong

The disk filled up during the dump. The 14 entries in `/backups` suggest retention isn't doing what you expect, so the disk was already near full before the run began.

Likely causes, roughly in order of probability:

[…]

If you paste the backup script and the `ls -lh /backups` output, I can pinpoint the retention bug and suggest a concrete patch.
```

With the style (straight-talk 2):

```markdown
The backup failed because /backups filled up (97 GB of 100 GB used, 3.0 GB free) while `pg_dump` was writing, and the disk is full because the retention policy is not keeping it at 7 backups. `ls /backups | wc -l` shows 14 entries, twice the policy.

The output does not show why there are 14. Two guesses, and what would confirm each:

[…]

Paste the `ls -lah /backups` output and the retention section of the script, and I can tell you which cause it is.
```

Both default replies open with a header and reach the answer in the line after it. Both straight-talk replies open with the answer. All four end by asking for the output of `ls -lah /backups` or `ls -lh /backups`, so on this question the default style already ends on a next action. One straight-talk reply defined `pg_dump` at first use ("Postgres's dump tool"), and in the step that deletes files it added that deleting cannot be undone, so the reader should check the name and size first.

In both straight-talk replies, the first sentence states a guess as a fact. Both say there that the retention policy, the cleanup that keeps 7 backups, is broken. Further down, both list a second guess: each backup might be two files, such as a dump and a checksum, and then 14 entries would be 7 backups and the cleanup would be working. The style tells Claude to put the answer first and also to mark a guess as a guess, and on this question the first rule won over the second. The default replies put "probably" and "likely" in their first line. Four runs on one question are too few to call this a pattern.

The straight-talk runs cost about twice as much as the default runs. I did not break the cost down by tokens, so I cannot say how much of the gap is the style file, which Claude Code sends with every request. Because Claude Code sends it with every request, tagore keeps that file at or under 5,000 bytes.

If you compare runs with and without the style in `claude -p`, set `outputStyle` in the project's `.claude/settings.local.json` for every run, including `"default"` for the runs without the style. The reminder hook looks for `outputStyle` in that file, then in the project's `.claude/settings.json`, then in `~/.claude/settings.json`, and it does not know about the `--setting-sources` flag of `claude -p`. My first two runs without the style had an empty project file and `--setting-sources project,local`. The hook found the style in my user settings and added its line to those sessions, although they had no style. The tagore docs list three cases in which the hook can print its line while the style is off: a style set by an organization policy, by another plugin or by the `--settings` flag. This is a fourth. The table uses the two runs after I set `"default"`.

## Network calls and files written

`prose_review.py` sends its slots to Google Gemini (`gemini-3.6-flash`) when `GEMINI_API_KEY` is set, or else to the Anthropic API (`claude-sonnet-5-5`) when `ANTHROPIC_API_KEY` is set. With neither key, it sends nothing and prints the prompt instead. `prose_rank.py` is an optional script that Claude can run before check 3, to rank sentences by how staged they look. It downloads the `sentence-transformers/all-MiniLM-L6-v2` model from Hugging Face and needs `torch` and `transformers`. The other scripts and both hooks run on your machine, and the newcomer read is a subagent of your own Claude Code session. The Stop hook writes its packets to `/tmp/fresh-eyes/` and one line per run to `~/.claude/fresh-eyes-log.jsonl`, with the time, the folder and the result. The reminder hook reads the settings files and writes nothing.

## Credit

tagore is inspired by Oskar Austegard's [declauding](https://github.com/oaustegard/claude-skills/tree/366e2f2c61fbd4291774ae4d35f32bf66e3ee2e9/declauding) skill. Entries 1 to 56 of the catalogue, the scripts, the newcomer read and the model-register-drift experiment behind most of the model profiles are Austegard's work, and [the credit page](https://tagore.rajnandan.com/credit/) has the details. tagore adds rules and examples from [blader/humanizer](https://github.com/blader/humanizer) v4.0.0 and [Aboudjem/humanizer-skill](https://github.com/Aboudjem/humanizer-skill), both under the MIT license. Entries 57 to 62 of the catalogue come from those two skills. Austegard took some of his entries from Simon Willison's llm-cliche-highlighter and from a paper on compressed syntax by Clark et al. (arXiv 2610.29162), and the skill names those sources.

## Try it

```
/plugin marketplace add rajnandan1/tagore
/plugin install tagore@tagore
/output-style tagore:straight-talk
```

Then ask Claude to update a README after a code change. When Claude tries to end the turn, the Stop hook hands Claude the newcomer-read prompt, Claude sends it to a subagent, and Claude fixes or answers each phrase that the subagent flags. The source is at [github.com/rajnandan1/tagore](https://github.com/rajnandan1/tagore) and the docs are at [tagore.rajnandan.com](https://tagore.rajnandan.com/). MIT licensed.
