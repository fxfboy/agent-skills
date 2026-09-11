# agent-skills

个人的 agent skill 集合，用 [`skills`](https://www.npmjs.com/package/skills) CLI 分发。

## 安装

```bash
npx skills add fxfboy/agent-skills
```

交互式选择要装的 skill 和目标 agent。非交互：

```bash
# 装到全局，指定 agent
npx skills add fxfboy/agent-skills --skill paseo-rescue -g -a claude-code -a codex -y

# 先看看里面有什么
npx skills add fxfboy/agent-skills --list
```

默认软链到各 agent 的 skills 目录（`~/.claude/skills`、`~/.codex/skills` 等），单一真源，`npx skills update` 可更新。软链不可用时加 `--copy`。

## Skills

| Skill | 做什么 |
| --- | --- |
| [paseo-rescue](skills/paseo-rescue/SKILL.md) | 接管一个中断的 [Paseo](https://paseo.sh) agent——限额、崩溃、报错停住的会话，把上下文捞过来自己继续干 |

### paseo-rescue

Claude 对话跑到一半撞上用量限额，想换 Codex 继续，但正因为限额，没法再用 handoff 把上下文交出去。

这个 skill 走反向：新开的 agent 自己去读被限额那个会话的 timeline。`get_agent_activity` 不向源 provider 发请求，所以源 agent 零额度消耗，`closed` 状态的也读得到。

需要 Paseo MCP 工具（`list_agents`、`get_agent_activity`、`get_agent_status`）。

用法——三种入口都行：

```
接管一下那个跨平台分析的 agent
接管 0e787eed-...
有个 agent 限额中断了，你接管一下继续干
```

最后一种不给目标时，它会列出候选让你选，不会自己猜。

## License

MIT
