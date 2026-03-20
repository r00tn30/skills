# Skills

Standalone skill collection for AI-assisted software development.

## Overview

This repository contains reusable skills that guide an agent toward consistent, high-quality outputs for specific domains and workflows.

Each skill is self-contained and includes:
- a main `SKILL.md` with behavioral rules and instructions
- reference material in `references/`
- quick guides in `assets/` (when applicable)

## Repository Structure

```text
skills/
├── prompt-writer/
│   ├── SKILL.md
│   ├── assets/
│   └── references/
└── python-clean-code/
    ├── SKILL.md
    ├── README.md
    ├── SUMMARY.md
    ├── assets/
    └── references/
```

## Available Skills

### `prompt-writer`
Practical patterns and references for writing clear, reliable prompts.

### `python-clean-code`
Python-focused clean code guidance with smell detection, refactoring techniques, and applied examples.

## How to Work in This Repo

1. Make updates directly in the relevant skill folder.
2. Keep `SKILL.md` concise and actionable.
3. Put long-form explanations in `references/`.
4. Keep examples and cheat sheets in `assets/`.
5. Validate wording for clarity and consistency before committing.

## Contributing

- Prefer small, focused changes.
- Keep language implementation-oriented and tool-agnostic.
- Update related docs when behavior changes.

## Git Model

This project uses a single Git repository at the root (`skills/`).
Individual skill folders are regular directories, not separate repositories.
