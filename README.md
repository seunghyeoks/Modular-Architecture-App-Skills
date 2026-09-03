# Swift Modular Architecture Skills

Rules for an iOS modular architecture built from SPM modules and an XcodeGen app shell.
Written as a skill: documentation for people, and a reference coding agents consult.

Attach it to a project as a git submodule.

```bash
git submodule add git@github.com:seunghyeoks/Swift-Modular-Architecture-Skills.git \
  .agents/skills/modular-architecture
```

Several projects share the same rules this way, and when a rule changes it changes in one place.

## Contents

- [SKILL.md](SKILL.md) — layer definitions and the dependency matrix, module target layout,
  resource placement, host test branching, and how to add an app target

## Single source of truth

The dependency matrix is actually enforced by each project's `Scripts/lint-deps.py`, in the
`ALLOWED` dictionary at the top of that file — not by this document. If the two disagree,
the script wins.
