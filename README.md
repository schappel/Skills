# Skills

My personal agent skills.

Each skill lives in `skills/<name>/` with a `SKILL.md` and any supporting reference files. They are mine to change — I maintain them here rather than editing a shared copy.

## Layout

```
skills/
  pr-description/     Write a pull request description
  zap-style/          Zap house C#/TypeScript style, scored against a 7-axis rubric
```

## Use

Claude Code loads skills from `~/.claude/skills`, so each one is symlinked from there into this repo:

```powershell
New-Item -ItemType SymbolicLink `
  -Path   "$HOME\.claude\skills\<name>" `
  -Target "D:\Personal\Skills\skills\<name>"
```

## Provenance

`zap-style` and `pr-description` started as copies from [ZapSkills](https://github.com/ZapTechnology/ZapSkills). Those are shared starting points, not a library to maintain in place — each developer forks what they need and takes it from there. Changes made here do not flow back.
