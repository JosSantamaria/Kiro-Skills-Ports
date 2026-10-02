# i-have-adhd (Kiro port)

Shape agent output for a reader with ADHD: lead with the next action, number
multi-step work, restate state across turns, suppress tangents, give specific
time estimates, make wins visible, cut all filler.

Based on [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) (MIT) —
adapted for Kiro's steering interface.

## Activation

This port ships an **on-demand** steering file. Turn it on with the phrase:

> **i-have-adhd** (or "adhd mode on")

Kiro loads `steering/i-have-adhd.md` and applies the 10 rules to every response
until you say **"stop adhd mode"** or **"normal mode"**.

Keywords: adhd, adhd mode, output style, action-first, concise, formatting.

## What changes

Before:
> Great question! Let me think about this. Your auth flow has a few moving
> pieces: the middleware, the token verification, and the cookie handling.
> Looking at src/auth.ts... Hope this helps!

After:
> Run `npm install jsonwebtoken@latest`, then edit `src/auth.ts:42`.
> 1. Open `src/auth.ts`
> 2. Replace `verifyToken` (lines 42–58) with the snippet below
> 3. Run `npm test -- auth.spec.ts`
>
> Next: paste the first failing line if any test fails.

## The 10 rules (full text in steering/i-have-adhd.md)

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

## Install

**As a Power:** Kiro Powers panel → Add power from GitHub →
`JosSantamaria/Kiro-Skills-Ports` → select `i-have-adhd`.

**As a steering file:** copy `steering/i-have-adhd.md` into your workspace's
`.kiro/steering/` folder. It is set to manual inclusion, so it only applies when
you reference it or ask for "adhd mode".

## Structure

```
├── POWER.md
├── steering/
│   └── i-have-adhd.md   # the 10 rules (canonical behavior, from upstream)
├── LICENSE
└── README.md
```

## Credits

Loosely based on *The Adult ADHD Tool Kit* by J. Russell Ramsay and Anthony L.
Rostain. Original skill by [ayghri](https://github.com/ayghri/i-have-adhd).
Kiro adaptation by Joset Santamaria.

## License

MIT — dual copyright: original authors (upstream skill) + Joset Santamaria
(Kiro port). See `LICENSE`.
