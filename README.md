# Skills For Real Engineers

A collection of reusable AI assistant skill prompts for software engineering workflows. Each skill is a `SKILL.md` file that instructs an AI coding assistant on how to behave in a specific scenario.

```bash
npx skills@latest add gausszhou/skills
```

Forked from [mattpocock/skills](https://github.com/mattpocock/skills). Browse online at [skills.sh](https://skills.sh/gausszhou/skills).

## Skills

| Skill | Language | Description |
|---|---|---|
| [domain-modeling](skills/domain-modeling/SKILL.md) | Chinese | 构建并打磨项目的领域模型——术语表、统一语言与架构决策记录（ADR）。 |
| [grilling](skills/grilling/SKILL.md) | Chinese | 对用户进行无情盘问，逐一压力测试方案或设计的每个分支决策。 |
| [grill-me](skills/grill-me/SKILL.md) | Chinese | `grilling` 技能的委托封装，直接触发一次盘问会话。 |
| [grill-with-docs](skills/grill-with-docs/SKILL.md) | Chinese | 结合 `grilling` 与 `domain-modeling`——边访谈边产出 ADR 与术语表。 |
| [handoff](skills/handoff/SKILL.md) | Chinese | 将当前对话精简为一份交接文档，供其他代理接手继续工作。 |
| [takeover](skills/takeover/SKILL.md) | Chinese | 读取交接文档并接手上一代理的工作，保持交接链不断。 |
| [use-appimage](skills/use-appimage/SKILL.md) | Chinese | AppImage 的安装、运行与集成——应用菜单入口、开机自启与故障排查。 |


