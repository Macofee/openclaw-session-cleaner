# session-cleaner

一个用于清理 OpenClaw 无用会话的 Skill。自动清理过期的 session 文件和 `sessions.json` 中的无效条目，保持会话列表整洁。

## 功能

- 扫描并列出所有会话，分类显示状态
- 根据会话状态给出清理建议
- **先列后清** — 展示清理计划，由用户确认后再执行，避免误删
- 保留正在运行（`running`）的会话和各 Agent 的 main session
- 自动识别 `main` Agent 是否存在并做相应处理
- 支持跨平台（任何安装了 OpenClaw 的机器均可使用）

## 安装

```bash
openclaw skills install amiao-session-cleaner
```

## 使用方法

在 OpenClaw 中直接说：
- "清理会话"
- "删除旧 session"
- "整理会话列表"

## 工作流程

```
1. 列出所有会话（按 Agent 分组）
2. 给出清理建议（哪些保留/哪些删除）
3. 你确认清理范围
4. 执行清理 + 重启 Gateway
```

## 工作原理

### 会话存储结构

```
~/.openclaw/agents/<agent-id>/sessions/
├── sessions.json          # 会话索引
├── <uuid>.jsonl           # transcript 文件
├── <uuid>.trajectory.jsonl  # 任务轨迹文件（可清理）
└── <uuid>.checkpoint.jsonl  # 检查点文件（可清理）
```

### 保留策略

| 类型 | 是否保留 |
|------|---------|
| `running` 状态的会话 | ✅ 是 |
| 各 Agent 的 main session | ✅ 是 |
| `done` / `timeout` / `failed` 的子 Agent | ❌ 否 |
| `.trajectory.jsonl` / `.checkpoint.jsonl` | ❌ 否 |
| `main` Agent 的所有会话（若 main 不存在）| ❌ 否 |

## 注意事项

- **绝对不要在未经用户确认前删除任何会话**
- `running` 状态的会话绝对不能删
- 清理完成后需重启 Gateway，否则 Control UI 仍显示旧数据

## License

MIT