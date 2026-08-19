# Skills For Real Engineers

A collection of reusable AI assistant skill prompts for software engineering workflows. Each skill is a `SKILL.md` file that instructs an AI coding assistant on how to behave in a specific scenario.

```bash
npx skills@latest add gausszhou/skills
```

Forked from [mattpocock/skills](https://github.com/mattpocock/skills). Browse online at [skills.sh](https://skills.sh/gausszhou/skills).

## Skills

| Skill | Language | Description |
|---|---|---|
| [domain-modeling](skills/domain-modeling/SKILL.md) | Chinese | 构建并维护项目的领域模型——术语表、统一语言与架构决策记录（ADR）。 |
| [grilling](skills/grilling/SKILL.md) | Chinese | Relentlessly stress-test a plan or design by questioning the user one branch at a time. |
| [grill-me](skills/grill-me/SKILL.md) | Chinese | Delegation wrapper that routes to the `grilling` skill. |
| [grill-with-docs](skills/grill-with-docs/SKILL.md) | Chinese | Combination of `grilling` and `domain-modeling` — interviews while producing ADRs and glossaries. |
| [handoff](skills/handoff/SKILL.md) | Chinese | Condense the current conversation into a handoff document for another agent to pick up. |
| [use-appimage](skills/use-appimage/SKILL.md) | Chinese | Install, run, and integrate AppImage apps — menu entry, autostart, and troubleshooting. |


