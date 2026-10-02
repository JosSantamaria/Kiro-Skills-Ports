# i-have-adhd — Kiro port

A skill that stops your coding agent from burying the answer. Action first.
Steps numbered. No "Hope this helps!".

Kiro port of [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) (MIT),
packaged as a Kiro Power (Agent Plugin format).

## Install

Powers panel → **Add Custom Power** → **Import power from GitHub** →
paste the repository URL and point it at this power's directory:

```
https://github.com/JosSantamaria/Kiro-Skills-Ports/tree/main/i-have-adhd
```

Or clone the repo and use **Import power from a folder** → select the
`i-have-adhd/` directory.

## Use

Turn it on:

> **adhd mode**  (or "i-have-adhd")

Turn it off:

> **stop adhd mode**  (or "normal mode")

While on, the 10 rules apply to every response.

## The 10 rules (full text in `skills/i-have-adhd/SKILL.md`)

1. Lead with the next action.
2. Number multi-step tasks.
3. End with one concrete next step.
4. Suppress tangents.
5. Restate state every turn.
6. Specific time estimates (minutes, not "a bit").
7. Make wins visible.
8. Matter-of-fact errors.
9. Cap visible lists to 5 items.
10. No preamble, no recap, no closers.

## Structure (Kiro Agent Plugin)

```
i-have-adhd/                 # power package root
├── plugin.json             # manifest (name, version, keywords...)
├── skills/
│   └── i-have-adhd/
│       └── SKILL.md         # the 10 rules (canonical behavior, from upstream)
├── LICENSE                  # MIT (dual copyright)
└── README.md
```

## Faithful to upstream

`skills/i-have-adhd/SKILL.md` is copied from the upstream `SKILL.md`, with only:
- the YAML front-matter trimmed to Kiro's skill fields (`name`, `description`), and
- rule 5 / exception 6 pointing at Kiro's task list and system prompt instead of
  a generic "harness".

## Credits & license

Original skill by [ayghri](https://github.com/ayghri/i-have-adhd), loosely based
on *The Adult ADHD Tool Kit* by J. Russell Ramsay and Anthony L. Rostain.
Kiro port by Joset Santamaria. MIT (dual copyright) — see `LICENSE`.
