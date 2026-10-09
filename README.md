# Kiro Skills Ports

Ports of community coding-agent skills, packaged as [Kiro](https://kiro.dev)
Powers (Agent Plugin format).

A single repo can hold **multiple Powers, each in its own directory** — Kiro
identifies each one by the `name` in its `plugin.json`. This repo is the hub for
all ports; add a new one by dropping a new directory with its own `plugin.json`
and `skills/`.

## Available ports

| Power (`name`) | Based on | What it does | License |
| --- | --- | --- | --- |
| [`i-have-adhd`](./i-have-adhd) | [ayghri/i-have-adhd](https://github.com/ayghri/i-have-adhd) | ADHD-friendly agent output: action-first, numbered steps, state restated, no filler. | MIT |
| [`waf-packaging`](./waf-packaging) | [OWASP CRS](https://github.com/coreruleset/coreruleset) + [modsecurity-crs-docker](https://github.com/coreruleset/modsecurity-crs-docker) | Package any web app behind a ModSecurity + OWASP CRS WAF container (reverse proxy), tuned for FastAPI. Phase-driven: insert in DetectionOnly → tune → block. | MIT |

## Install a port

Powers panel → **Add Custom Power** → **Import power from GitHub**, then point at
the port's directory, e.g.:

```
https://github.com/JosSantamaria/Kiro-Skills-Ports/tree/main/i-have-adhd
```

Each port directory is a self-contained power package:

```
<port-name>/
├── plugin.json             # Agent Plugin manifest (name, version, keywords)
├── skills/
│   └── <skill>/SKILL.md     # the skill behavior
├── LICENSE
└── README.md
```

## Why Agent Plugin format

Kiro's current Powers format is `plugin.json` (the open
[agent-plugins.org](https://agent-plugins.org) standard). Each power lives in
its own subdirectory with its own manifest; there is no index manifest — the
directory + `plugin.json name` is the identity. See
[kiro.dev/docs/powers](https://kiro.dev/docs/powers/create).

## Attribution

Every port keeps the upstream license and credits the original author. See each
port's `LICENSE` and `README.md`.
