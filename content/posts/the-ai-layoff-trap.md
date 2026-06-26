+++
title = "The AI Layoff Trap"
description = "A March 2026 paper shows why competing firms automate past the point that helps anyone, hurting workers and owners alike, and why most fixes miss. I turned the model into a simulator you can run."
date = 2026-06-26
updated = 2026-06-26

[taxonomies]
tags = ["ai", "economics", "game-theory", "analysis"]

[extra]
toc = true
comment = false
+++

Picture a town with 1,000 companies and 100,000 workers. Each company employs 100 people, pays them a wage, and sells to them. The money leaves through the front door as salary and comes back through the till. That loop is the local economy.

Now hand every company the same AI. One firm runs the numbers. Replace a worker earning $50k a year, run the software for $20k, keep $30k. Easy call. So it makes the call. So does the company next door, and the one after that.

For a while each firm is richer. Then the people they let go stop spending, the till runs thin, and profit climbs, peaks, and slides back under where it began. Every firm followed the smart local incentive, and the town ended up poorer than before any of it started.

That is the argument in a paper I read earlier this year, "The AI Layoff Trap" by Brett Hemenway Falk and Gerry Tsoukalas (arXiv:2603.20617, March 2026). The math is tight and the takeaway stuck with me. I wanted to poke at it, nudge a number and watch the curves move, so I built a simulator for it: [ailayoffs.rajnandan.com](https://ailayoffs.rajnandan.com/). This post is the idea behind the toy.

## The trap is one fraction

A single fraction drives the whole thing. A firm that automates banks the entire saving. But the demand drop from its laid-off workers barely touches it. The rest spills onto rivals.

In the town above, a company that fires people feels about one-thousandth of the demand damage it caused. The other 999 parts are somebody else's problem. That asymmetry makes automating the dominant move: it pays off whatever the rivals do, even as everyone choosing it sinks the whole place. A prisoner's dilemma with a payroll.

The paper measures the demand each laid-off worker takes out the door with one expression: ℓ = λ(1−η)w. Read it slowly. A worker spends half of income locally (λ = 0.5), recovers about 30% of that income after losing the job (η = 0.3), and earned $50k (w). That works out to $17,500 of local spending gone per person. Run a layoff wave through it and the missing demand is not rounding noise.

## Both the firm and the worker lose

The comfortable version of this story is that workers lose and owners win. The paper rejects it. Past a point, owners lose too.

There is an automation rate that is best for the town as a whole, the cooperative optimum. Firms blow past it. Each one's private best response sits further out, the Nash equilibrium, and by the time they arrive both profit and worker income have already turned down. That gap is pure waste, not a transfer. Economists call it deadweight loss, and both workers and owners eat it.

Put workers on one axis and owners on the other. The point both groups land on sits to the lower-left of where they could be standing together. Richer is available to everyone. The market structure will not let them walk there on their own.

Two forces we expect to soften this make it worse. More competition widens the trap, because each firm's share of the damage shrinks as the number of firms grows. A monopolist automates about right, since it swallows the full demand loss itself. Cheaper and sharper AI widens the trap too. The paper names the chase a Red Queen race: everyone sprints to out-automate everyone else, the advantages cancel at equilibrium, and the distortion is all that's left.

## Why the obvious fixes don't work

I expected the paper to arrive at something I had heard before. Instead it walks through the usual remedies and shows each one sliding off the problem, for the same reason every time.

| Fix | Why it misses |
| --- | --- |
| Wage cuts | Shrink the saving and the demand loss together. They change when the trap bites, not whether. |
| More entrants | Split the demand loss across more firms, so each feels even less of it. The trap gets wider. |
| Capital or profit tax | Scales profit up or down. It drops out of the decision about the next worker. |
| UBI | Raises the floor everyone stands on. The gap between a firm's choices, the thing it acts on, stays put. |
| Upskilling | Helps only if displaced workers fully recover their income. Short of that, the gap stays open. |
| Worker equity | Narrows the gap. It closes only past an ownership share that workers can't reach while they spend less than they earn. |
| Handshake deals | Automating beats cooperating for any one firm, so no voluntary pact holds. Talk is cheap. |

The thread tying these together: anything that moves payoff levels leaves the per-worker decision alone. The trap lives on the margin, in the call about the next worker, not in the totals at the bottom of the spreadsheet. Move the totals and the firm still automates the same way.

## The one lever that works

One instrument bites. A Pigouvian automation tax, set at τ* = ℓ(1−1/N), charges a firm for the demand loss it shoves onto everyone else. In the town above that lands near the full $17,500 per displaced worker. Now the firm feels the cost it used to externalize, and it automates only when the real saving beats the real social cost. The private call lines up with the town's call.

The revenue is where it turns clever. Route it into retraining or wage insurance and you raise η, the slice of income a displaced worker gets back. Raise η and ℓ shrinks, which shrinks the tax you need next year. The cure eats into the disease over time.

## So I built a simulator

Reading the proofs, I kept converting numbers in my head. A simulator does that for you. [ailayoffs.rajnandan.com](https://ailayoffs.rajnandan.com/) has two halves.

Story mode runs the town. Pick how many workers each firm replaces per round and how many rounds pass, hit run, and watch 100,000 dots flip from employed to displaced while the profit curve rises and then dives under the start line.

The full model exposes every knob: number of firms, wages, AI cost, integration friction, how much workers spend locally, how much income they recover. The Nash equilibrium, the cooperative optimum, and the policy instruments all redraw live. There are preset scenarios as well, a clean prisoner's dilemma, a monopoly, a fragmented market, an upskilling utopia where η climbs above 1 and the trap reverses, near-free AI, and the Pigouvian tax switched on. Load one, then start dragging sliders and trying to break it. That is where the model stops being a claim you read and turns into something you argue with.

## What I take from it

A caveat first. This is a stylized model. Symmetric firms, one sector, no decades-long reabsorption of workers into industries nobody has invented yet. The real economy has escape valves this leaves out, so read it as a mechanism and not a forecast.

The mechanism is the part worth keeping. The return on replacing one person can be real for a firm and a trap for everyone at once, and no single company can opt out without handing rivals an edge. That is why the paper lands on policy rather than a plea for firms to behave. Asking a company to automate less for the common good is asking it to lose.

I build software that automates work, so this one sits close. The honest reading is that local ROI and aggregate harm can both be true at the same moment, and the fix sits above any one firm's pay grade. Worth holding in your head the next time the math on replacing a person looks too clean. Go run the simulator and watch how fast the curve turns.
