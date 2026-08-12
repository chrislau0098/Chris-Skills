# claude-skills

自用的 Claude Code / Claude Agent Skills 集合，可直接装到本地使用。

## 目录

| Skill | 用途 |
|---|---|
| [figma-motion-storyboard](skills/figma-motion-storyboard/) | 在 Figma 画布上用原生 Motion 做时间轴动画。先生成「动画时间板」故事板框架，设计师把分镜填好后读回，写 `manualKeyframeTracks` 产出动画。含 Plugin API 常见错误、缓动与节奏判断、补偿轨道与速度衔接的推导 |

## 安装

Skill 放进 `~/.claude/skills/` 即可被 Claude Code 识别。推荐软链接，这样 `git pull` 之后自动生效。

```bash
git clone https://github.com/chrislau0098/claude-skills.git ~/Code/claude-skills
ln -s ~/Code/claude-skills/skills/figma-motion-storyboard ~/.claude/skills/figma-motion-storyboard
```

装完在 Claude Code 里输入 `/` 能看到 skill 名，或者直接描述任务让它自动触发。

只想装某一个又不想留仓库，直接把对应目录拷进 `~/.claude/skills/` 也可以。

## 结构约定

```
skills/<skill-name>/
├── SKILL.md          必需。frontmatter 的 name + description 决定何时被触发，正文控制在 100 行内
├── references/       按需加载的详细文档，SKILL.md 里用相对链接指向
├── templates/        产出物模板
└── scripts/          确定性操作的脚本（校验、格式化等）
```

`description` 是 agent 唯一看得到的选择依据，写清「做什么」和「什么情况下用」，最长 1024 字符。

## 依赖

`figma-motion-storyboard` 需要：

- Figma MCP（官方插件），提供 `use_figma` 工具和 `figma-use` / `figma-use-motion` 两个 skill
- Figma 账号开通 `metronome` 特性开关，否则 Motion API 不可用
- 可选：[`emil-design-eng`](https://github.com/emilkowalski/skills) 和 `motion` 两个 skill，用于缓动与节奏判断

## License

MIT
