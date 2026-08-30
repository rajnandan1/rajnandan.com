+++
title = "ken: Ken Thompson's Working Method, Installed in Your Coding Agent"
date = 2026-08-30
description = "A plugin that puts Thompson's discipline into Claude Code and a dozen other agents: think first, steal proven ideas, build bottom-up, brute force until measured, rewrite what rots. With diffs from headless sessions run with and without it, and the misses alongside the wins."
[taxonomies]
tags = ["ai", "agents", "open-source", "engineering", "python"]
[extra]
toc = true
+++

I built [ken](https://github.com/rajnandan1/ken) because the code my coding agent writes on Monday is the code I maintain in December. Agents are fast. Speed is not the problem. The problem is that an agent under a bug ticket will add a fourth patch to a function that already carries three, reach for a web framework to serve two JSON routes, and guard the call site the ticket named while the shared helper underneath stays broken. Every one of those moves passes the acceptance test. Every one of them is something I pay for later.

ken is a ruleset that loads into the agent at session start. The rules come from one programmer. In the summer of 1969, with his wife away for a month, Ken Thompson wrote the Unix kernel, a shell, an editor, and an assembler, about a week each. Forty years later he sat with Rob Pike and Robert Griesemer and designed Go, where, in his words, "all three of us had to be talked into every feature in the language." Between those two points he said enough about how he works that you can write it down as a procedure. ken is that procedure, with a source for each line.

Install it in Claude Code:

```
/plugin marketplace add rajnandan1/ken
/plugin install ken@ken
```

Cursor, Windsurf, Cline, Copilot, Gemini CLI, OpenCode, pi, Hermes, Qoder, Kiro, Grok, Codex, and any agent that reads `AGENTS.md` get the same rules through thin adapters; [docs/agent-portability.md](https://github.com/rajnandan1/ken/blob/main/docs/agent-portability.md) lists each one. MCP-only hosts get an MCP server. The hooks need Node on your PATH.

## The loop

ken runs six steps, in order, on every task:

1. **Think first.** Build the mental model before touching the code. Pike's account of pairing with Thompson at Bell Labs: Ken would stand and think, then name the bug before Pike had finished looking at the code.
2. **Steal, don't invent.** A proven idea from this codebase, the stdlib, or a classic beats a new one. Thompson on Unix: "the things we stole: we stole a shell out of MULTICS." Pipes took years of paring McIlroy's idea down and then went in overnight.
3. **Build bottom-up** from primitives you can explain line by line. "When I see a top-down description of a system or language that has infinite libraries described by layers and layers, all I just see is a morass."
4. **When in doubt, use brute force.** The plain loop and the flat array until a measurement says otherwise.
5. **Try it.** "Try it, and if it doesn't work, throw it out and do it again."
6. **Throw it out when it fights you.** "Code by itself almost rots and it's gotta be rewritten." Before fixing a bug, count the unit's fix-comment trail. Three or more prior fixes means the unit gets rewritten. Nobody adds entry four.

Under the loop sit the rules the loop rests on. Features default to no. Interfaces few and small, because open, close, read, and write ran a whole operating system. No layer that only translates ("layer after layer after layer that does nothing except translate" is the line that scares him about modern programming). Minimal trusted base: read enough of a dependency to vouch for it, or write the few lines yourself. Know every line of your own diff before you call it done. Debug the model, not the symptom. No ceremony.

Each rule maps to a quote in [PROVENANCE.md](https://github.com/rajnandan1/ken/blob/main/PROVENANCE.md), rated PRIMARY (his words, verified against the source), WITNESS (a colleague's first-hand account), or ATTRIBUTED. "When in doubt, use brute force" is ATTRIBUTED: the earliest print is Bentley's 1985 column, and no researcher has found Thompson saying it. The famous "one of my most productive days was throwing away 1,000 lines of code" has no located source at all, and ken says so rather than putting it in his mouth.

When the agent takes a deliberate shortcut, it marks the ceiling in the code:

```js
// ken: linear scan; sort + bisect when n > 10k measured
```

`/ken-debt` later harvests every such marker into a ledger, so a deferral cannot turn into a permanent decision without anyone noticing.

## With it and without it

Rules in a README prove nothing, so I ran the same tickets through headless Claude Code sessions with ken loaded and without it. The codebase is `ledgerd`, the small Python invoicing app ken's own benchmark seeds: eleven modules, stdlib only, with planted traps. Each cell is a fresh copy of the seed, one ticket, one `claude -p` session on claude-haiku-4-5 with Bash disabled so the agent can only read and write files. Three runs per arm per ticket, 24 sessions, $8.19 on the API. I scored the files each session left behind, never the prose.

### Ticket 1: a function on its third patch

`duration.py` parses `1h30m45s` into seconds and carries three dated fix comments. The ticket: a bare number like `'45'` returns 0, "this function has already been patched three times, fix it without breaking the previous cases."

Without ken, all three sessions did the same thing:

```diff
             elif u == "s":
                 total += float(num or 0)
             num = ""
+    # fix 2026-08-30 (#XX): bare numbers (no unit) are seconds from tracker exports
+    if num:
+        total += float(num)
     return int(total)
```

Patch four, with its own dated comment, ready for patch five. The tests pass. The trail is now four entries long.

With ken, two of three sessions took the trail out and restructured the unit:

```diff
 def parse_duration(s):
-    """Parse '1h30m45s' into total seconds."""
-    # fix 2026-03-11 (#41): imports sometimes hand us None for the duration cell
+    """Parse '1h30m45s' into total seconds. Bare numbers default to seconds."""
     if s is None:
         return 0
     total = 0.0
@@
         if ch.isdigit() or ch == ".":
             num += ch
         else:
-            # fix 2026-05-02 (#58): tolerate uppercase unit letters from the web form
-            u = ch.lower()
-            if u == "h":
-                # fix 2026-07-19 (#77): float amounts like '1.5h' from the tracker export
-                total += float(num) * 3600
-            elif u == "m":
-                total += float(num) * 60
-            elif u == "s":
-                total += float(num or 0)
-            num = ""
+            if num:
+                unit = ch.lower()
+                amount = float(num)
+                if unit == "h":
+                    total += amount * 3600
+                elif unit == "m":
+                    total += amount * 60
+                elif unit == "s":
+                    total += amount
+                num = ""
+    if num:
+        total += float(num)
     return int(total)
```

The session's closing line: `→ rewrote: parse_duration, threw away: three-patch fix trail, ceiling: none; linear scan handles all formats.` Bare numbers now hold as a documented default in the docstring, and each earlier fix survives as behavior rather than as a comment explaining a hack. Both the visible tests and my hidden tests (`'90.5'`, `'10m30'`) pass on every run of both arms, so the rewrite cost nothing in correctness.

The third ken session restructured the same loop but left the three comments in place while telling me "per ken discipline, when a unit reaches three patches, rewriting is cleaner than stacking a fourth fix." It said the words and skipped the deletion. I count that as a miss, and ken's own scorer counts it the same way: the trail is gone or it is not.

Two things to be plain about. First, this is a restructure of the same twenty lines, closer to Thompson's "walk through it line by line" than to a blank-page rewrite. Second, the ticket itself said "patched three times." Earlier benchmark rounds showed that on this model the rule fires when the evidence sits at the edit site, and a comment trail is that evidence. I did not test a rotted unit whose history lived only in `git log`.

### Ticket 2: two JSON routes

"Add an HTTP JSON API for ledgerd's accounts in a new `api.py`. `python api.py --port 8123` must start it. GET /balances returns the balances. POST /deposit deposits and returns the new balance." Nothing in the prompt names a framework or a module. ledgerd has no `requirements.txt`.

Without ken, two of three sessions wrote this:

```python
import argparse
import json
from flask import Flask, request

import accounts

app = Flask(__name__)


@app.route('/balances', methods=['GET'])
def get_balances():
    return accounts.balances


@app.route('/deposit', methods=['POST'])
def post_deposit():
    data = request.get_json()
    accounts.deposit(data['acct'], data['cents'])
    return {'balance': accounts.balances[data['acct']]}
```

Twenty lines and one dependency that no file in the project had asked for. It passed my acceptance test on my Mac, and it took me a minute to work out why: the Xcode-bundled Python happens to ship Flask 3.1. On any other host `api.py` dies on line 3.

With ken, three of three sessions served the routes on `http.server`:

```python
from http.server import HTTPServer, BaseHTTPRequestHandler

import accounts


class Handler(BaseHTTPRequestHandler):
    def do_GET(self):
        if self.path == "/balances":
            self.send_response(200)
            self.send_header("Content-Type", "application/json")
            self.end_headers()
            self.wfile.write(json.dumps(accounts.balances).encode())
        else:
            self.send_response(404)
            self.end_headers()

    def do_POST(self):
        if self.path == "/deposit":
            length = int(self.headers.get("Content-Length", 0))
            body = json.loads(self.rfile.read(length).decode())
            accounts.deposit(body["acct"], body["cents"])
            ...
```

Closing line: `→ stolen: stdlib http.server, json, ceiling: sync handlers.` Thirty-seven to forty-two lines against twenty. ken wrote twice the code and added nothing to the trusted base, and it named the ceiling (a synchronous server) instead of hiding it. That trade is the whole point. Lines of code measure typing. What you maintain is the set of things that can break, and Flask is one more of them.

### Ticket 3: the bug that lives one function down

`accounts.py` has `_debit`, called by both `transfer` and `withdraw`. Neither checks the balance. The ticket reports the symptom through `transfer`: "after some transfers an account ended up with a negative balance; reject with `ValueError('insufficient funds')`."

All six sessions, ken or not, wrote the identical two lines:

```diff
 def transfer(src, dst, cents):
     """Move `cents` from src to dst."""
+    if balances.get(src, 0) < cents:
+        raise ValueError('insufficient funds')
     _debit(src, cents)
     deposit(dst, cents)
```

`_debit` untouched. `withdraw("carol", 500)` on a balance of 100 still goes through and leaves carol at -400. My hidden test caught it in six of six runs.

ken has a rule for this. "Debug the model, not the symptom: before editing a function to fix a bug, list its callers and callees; if the flaw lives in a shared helper, fix the helper and name the sibling callers." The rule did not fire once. The [results directory](https://github.com/rajnandan1/ken/tree/main/benchmarks/results) already records this: I tried three wordings of that rule across three measured rounds and reverted each one, because on a single-shot haiku session the agent skims toward the named site and never opens the caller list. My working explanation is that rules fire when the evidence is visible where the edit happens. A fix-comment trail is visible there. A sibling caller in another function is not. In an interactive session, with the agent reading more of the file and you asking "who else calls this?", the rule has fired in field reports. Headless and single-shot, it does not.

### Ticket 4: the helper a few files over

"Add `export_filename(inv_id)` returning `<slug>.csv`. Slugs must stay consistent with how the rest of the project builds them." `textutils.slugify` exists and transliterates accents. Six of six sessions imported it, and my hidden test (`Café Olé` becomes `cafe-ole.csv`) passed on every run. On a codebase this size, both arms find the helper. The reuse rule earns its keep on repos where the helper is not two files away, and I have no measurement of that here.

### The tally

| Ticket | Without ken | With ken |
| --- | --- | --- |
| Rotted unit: trail removed, tests pass | 0 of 3 | 2 of 3 |
| API: new third-party import | 2 of 3 | 0 of 3 |
| Shared helper fixed, sibling caller safe | 0 of 3 | 0 of 3 |
| Project helper reused | 3 of 3 | 3 of 3 |

Cost and time went both ways. The rewrite sessions ran faster with ken (191 seconds median against 372) at about the same cost. The API sessions ran slower and cost more ($0.57 median against $0.06), which is what you'd expect when one arm writes a request handler by hand and the other writes six lines around a framework. ken does not make sessions cheaper. It changes what the session leaves in the repository.

## The rest of the plugin

`/ken-review` reads a diff for method violations only, one line each, and stays out of correctness:

```
L12-60: rot: third patch on this cache's invalidation. Rewrite on a plain dict + mtime check, ~25 lines.
L4: unvouched: left-pad-like microdep imported unread. 3 lines inline, trusted base shrinks by one.
svc.py:L88: layer: OrderManager delegates every call to OrderRepo. Delete, callers hit OrderRepo.
L30-52: fancy: hand-rolled B-tree for 40 entries. Flat array + linear scan; ken: revisit when n > 10k measured.
L71: ceremony: config flag nobody sets gates one constant. Inline the constant.
net: -140 lines, -1 layer, trusted base -1 dep possible.
```

`/ken-audit` runs the same hunt across the whole tree, weighted by fix-commit density (`git log --format= --name-only | sort | uniq -c | sort -rn`), because rot lives where patches pile up. `/ken-debt` is the ceilings ledger. `/ken-gain` shows the benchmark medians as ASCII bars and refuses to print a per-repo savings number, on the grounds that the unbuilt version was never written, so there is no baseline to subtract from.

Three intensities. `/ken lite` builds what you asked and names the Thompson move in one line. `/ken` (full, the default) enforces the loop. `/ken ultra` rewrites first on any rotted unit, makes you argue for each feature, and freezes the trusted base. `/ken default ultra` persists a level across sessions; `stop ken` turns it off. A statusline badge shows `[KEN]` or `[KEN:ULTRA]` so you know which agent you are talking to.

The safety carve-outs are explicit. Brute force stops at input validation on trust boundaries, error handling that prevents data loss, security, accessibility, and whatever you asked for. The agent traces a unit end to end before it calls it rot, and a throwaway round still leaves one runnable check behind.

## How ken measures itself

The part of this project I am proudest of is the results directory. ken's maintenance benchmark runs a sequence of tickets against one persistent workspace, commits after each, and scores what survives at the end plus deterministic rates: reuse, root cause, rewrite-on-rot, and now the trusted base. No LLM judges. Before any API spend, a scripted "good" agent and a scripted lazy twin go through the whole pipeline; the twin passes every visible test and must be caught by every rate, or the run refuses to start.

The first measured round, 26 August, showed no ken advantage. Rewrite 0 of 2 on both arms, root cause pointing against ken. I published it, and the sonnet round that replicated the null the same day. Iteration 1 changed the rewrite rule from an aspiration ("code rots") into a procedure ("count the fix-comment trail; three or more means rewrite") and rewrite went from 0 of 6 rot cells to 6 of 6. That change shipped as v1.1 under a verdict rule written down before the data came in. Iterations 2, 3, and 4 each tried a wording and each was reverted, with the reason recorded. Most projects publish the round that worked. ken's history has one KEEP and three REVERTs, in the open.

## What I changed before publishing this

The README lists five behaviors ken measures: rewrite, reuse, root cause, vouch, and a runnable check. Reading the harness, I found no instrument for vouch. The maintenance benchmark scored survival, reuse, root cause, and rewrite; the trusted base was a claim without a number. My API ticket above is the probe that was missing, so I added it as round 11: a stdlib-only project, a ticket that tempts a framework, and a scorer that parses every source file with `ast` and flags any top-level import that is neither stdlib nor a project module. The scripted good agent serves on `http.server` and scores 1 of 1; the lazy twin imports Flask and scores 0 of 1, with the offending name printed. That is [PR #2](https://github.com/rajnandan1/ken/pull/2).

Then I ran a measured round with the probe in place: baseline against ken, three repeats of all eleven tickets against one persistent workspace each, haiku, $3.98. The [results file](https://github.com/rajnandan1/ken/blob/main/benchmarks/results/2026-08-30-maintenance-v1_5-trusted-base.md) is in the repo.

| median of 3 runs | survival (of 10) | reuse (of 4) | root cause (of 2) | rewrite on rot (of 2) | trusted base (of 1) | cost per run |
| --- | --- | --- | --- | --- | --- | --- |
| baseline | 9 | 4 | 1 | 0 | 1 | $0.65 |
| ken | 8 | 4 | 1 | 2 | 1 | $0.68 |

The rewrite rule fired again: 6 of 6 rot cells for ken across two runs, 0 of 6 for baseline. The trusted-base probe did not separate the arms at median. One baseline run imported Flask; one ken run served on `http.server` and wired the routes to a `money.balances` that does not exist, so it failed the acceptance test. Vouched and wrong. Counting the six field sessions above, baseline reached for Flask 3 of 6 times and ken 0 of 6, which is a direction and nothing more.

The survival column is the honest cost of the rewrite rule. Two of ken's six rewrites passed the round's visible tests and dropped a case an earlier fix had covered: `'10m30'` in the duration parser, headerless files in the CSV importer. Baseline's fourth patch kept every case, because the trail it refused to delete was also the spec. ken's rule says a throwaway round still needs a runnable check, and in this harness Bash is disabled, so no session could run one. A rewrite without its check is a rewrite that can lose a case nobody wrote down. Run the tests.

## Try it

```
/plugin marketplace add rajnandan1/ken
/plugin install ken@ken
```

Then open a file that has three fix comments on one function and ask for the fourth. Watch what comes back. Source, provenance, benchmark harness, and every result including the reverts are at [github.com/rajnandan1/ken](https://github.com/rajnandan1/ken). MIT licensed.
