# i-have-adhd — Kiro port

A skill that stops your coding agent from burying the answer. Action first.
Steps numbered. No "Hope this helps!".

Kiro port of [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) (MIT).

## What it does

Shapes every agent response so an ADHD brain can act on it:

- Leads with the next action (command, path, snippet) — not preamble.
- Numbers multi-step work and ends with one concrete next step.
- Restates "step 3 of 5" every turn so you don't have to hold it in your head.
- Gives time estimates in concrete units (minutes, not "a bit").
- Makes finished work visible; states errors matter-of-factly.
- Cuts openers, recaps, and closers.

## Install

### Option A — as a Kiro Power
Powers panel → **Add power from GitHub** → `JosSantamaria/Kiro-Skills-Ports`
→ select **i-have-adhd**.

### Option B — as a steering file
Copy `steering/i-have-adhd.md` into your workspace's `.kiro/steering/` folder.
It uses `inclusion: manual`, so it stays dormant until you invoke it.

## Use

Turn it on:

> **i-have-adhd**  (or "adhd mode on")

Turn it off:

> **stop adhd mode**  (or "normal mode")

While on, the 10 rules apply to every response.

## Faithful to upstream

The 10 rules in `steering/i-have-adhd.md` are copied verbatim from the upstream
`SKILL.md`, with only two adaptations:
- The YAML front-matter uses Kiro's steering format (`inclusion: manual`).
- Rule 5 and exception 6 point at Kiro's task list / system prompt instead of a
  generic "harness".

## Credits & license

Original skill by [ayghri](https://github.com/ayghri/i-have-adhd), loosely based
on *The Adult ADHD Tool Kit* by J. Russell Ramsay and Anthony L. Rostain.
Kiro port by Joset Santamaria. MIT (dual copyright) — see `LICENSE`.
