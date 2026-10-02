# Kiro Skills Ports

Ports of community coding-agent skills, adapted for [Kiro](https://kiro.dev)'s
Power / steering interface.

Each skill lives in its own folder and is installable as a Kiro Power
(Powers panel → Add power from GitHub → `JosSantamaria/Kiro-Skills-Ports`,
then pick the subfolder) or by copying its steering file into
`.kiro/steering/` of your workspace.

## Available ports

| Port | Based on | What it does | License |
| --- | --- | --- | --- |
| [`i-have-adhd`](./i-have-adhd) | [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) | ADHD-friendly agent output: action-first, numbered steps, state restated, no filler. | MIT |

## How these ports work

The upstream skills ship manifests for many runtimes (Claude Code, Codex,
Cursor, OpenCode, Gemini...). Kiro doesn't use those manifests — it uses:

- **Powers** (`POWER.md` + `steering/`) added from a GitHub repo, and
- **Steering files** (`.kiro/steering/*.md`) that inject instructions into the
  agent's context, either always-on or on-demand.

So each port here provides a `POWER.md` (metadata + activation) and a steering
file that carries the actual behavior. The canonical rules are kept faithful to
upstream; only the packaging is Kiro-specific.

## Attribution

Every port keeps the upstream license and credits the original author. See each
port's `LICENSE` and `README.md`.
