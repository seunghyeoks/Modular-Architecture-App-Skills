# Swift Modular Architecture Skills

Layer rules and module composition for an iOS modular architecture.
Written as a skill: documentation for people, and a reference coding agents consult.

This describes the architecture only. How a given project builds it — which parts are Swift
Packages, which are Xcode targets — belongs to that project, not here.

Attach it to a project as a git submodule.

```bash
git submodule add git@github.com:seunghyeoks/Swift-Modular-Architecture-Skills.git \
  .agents/skills/modular-architecture
```

Several projects share the same rules this way, and when a rule changes it changes in one place.

## Contents

- [SKILL.md](SKILL.md) — layer definitions, the dependency matrix, how a module is composed,
  mock conventions, and error translation across layers

## Single source of truth

The dependency matrix is actually enforced by each project's `Scripts/lint-deps.py`, in the
`ALLOWED` dictionary at the top of that file — not by this document. If the two disagree,
the script wins.
