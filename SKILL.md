---
name: session-cleaner
description: 清理 OpenClaw 无用会话的工具。当需要清理旧会话、删除过期 session 文件、或整理会话列表时使用此技能。触发场景包括：用户说"清理会话"、"删除旧 session"、"清理子 Agent"、"整理会话列表"。
---

# Session Cleaner

清理 OpenClaw 无用会话，保持会话列表整洁。

## 前置判断

执行清理前，先确认：
- `main` Agent 是否仍然存在（如果不存在，它的 sessions.json 应清空）
- 哪些会话状态为 `running`（这些绝对不能删）

```bash
openclaw agents list
openclaw sessions --all-agents --limit all
```

## 清理步骤

### 步骤 1：清理孤立的 .jsonl 文件（可选）

删除已结束会话的 transcript、trajectory、checkpoint 文件，保留正在进行的：

```bash
OPENCLAW_HOME="${OPENCLAW_HOME:-$HOME/.openclaw}"
cd "$OPENCLAW_HOME/agents/$AGENT_ID/sessions"

# 删除 trajectory 和 checkpoint 文件（这些是历史任务痕迹，可以全删）
find . -name "*.trajectory.jsonl" -delete
find . -name "*.checkpoint*.jsonl" -delete

# 保留：当前运行的 session 和 main session（通过 sessions.json 决定）
ls *.jsonl | sort -t_ -k2  # 查看剩余文件
```

### 步骤 2：清理 sessions.json（核心）

直接编辑 sessions.json，只保留需要保留的条目：

```python
import json
import os
from pathlib import Path

OPENCLAW_HOME = os.path.expanduser(os.environ.get("OPENCLAW_HOME", "~/.openclaw"))

def clean_agent(agent_id):
    """清理指定 Agent 的 sessions.json"""
    sessions_file = Path(OPENCLAW_HOME) / "agents" / agent_id / "sessions" / "sessions.json"
    if not sessions_file.exists():
        print(f"[{agent_id}] sessions.json 不存在，跳过")
        return

    with open(sessions_file, 'r') as f:
        data = json.load(f)

    if not data:
        print(f"[{agent_id}] sessions.json 已是空的")
        return

    # 判断 agent 是否存在
    agents_json = Path(OPENCLAW_HOME) / "openclaw.json"
    with open(agents_json, 'r') as f:
        config = json.load(f)
    agent_ids = [a['id'] for a in config.get('agents', {}).get('list', [])]

    # main agent 不存在则清空
    if agent_id == 'main' and agent_id not in agent_ids:
        new_data = {}
        print(f"[{agent_id}] main Agent 不存在，清空 sessions.json")
    else:
        # 其他：保留 main session 和 running 会话
        new_data = {
            k: v for k, v in data.items()
            if ':main' in k or v.get('status') == 'running'
        }

    with open(sessions_file, 'w') as f:
        json.dump(new_data, f)

    removed = len(data) - len(new_data)
    print(f"[{agent_id}] 保留 {len(new_data)}，删除 {removed}")
    return removed

# 清理所有主要 Agent
for agent in ['dispatcher', 'file-assistant', 'knowledge', 'coder', 'main']:
    clean_agent(agent)
```

### 步骤 3：重启 Gateway

```bash
openclaw gateway restart
```

### 步骤 4：验证

```bash
openclaw sessions --all-agents --limit all
```

## 保留策略

| 类型 | 是否保留 |
|------|---------|
| `running` 状态的会话 | ✅ 是 |
| 各 Agent 的 main session（`:main`） | ✅ 是 |
| `done`/`timeout`/`failed` 的子 Agent | ❌ 否 |
| `main` Agent 的所有会话（若已不存在）| ❌ 否 |
| `.trajectory.jsonl` | ❌ 否 |
| `.checkpoint.jsonl` | ❌ 否 |

## 快速清理（一键脚本）

```bash
python3 - << 'PYEOF'
import json
import os
from pathlib import Path

home = os.path.expanduser(os.environ.get("OPENCLAW_HOME", os.path.expanduser("~/.openclaw")))
print(f"OpenClaw home: {home}")

# 读取当前 agent 列表
agents_file = Path(home) / "openclaw.json"
with open(agents_file) as f:
    config = json.load(f)
agent_ids = [a['id'] for a in config.get('agents', {}).get('list', []) or []]

target_agents = ['dispatcher', 'file-assistant', 'knowledge', 'coder', 'main']

for agent in target_agents:
    sf = Path(home) / "agents" / agent / "sessions" / "sessions.json"
    if not sf.exists():
        print(f"[{agent}] sessions.json 不存在")
        continue

    with open(sf) as f:
        data = json.load(f)

    if agent == 'main' and 'main' not in agent_ids:
        new_data = {}
        print(f"[{agent}] main 不存在，清空 ({len(data)} 条)")
    else:
        new_data = {k: v for k, v in data.items()
                    if ':main' in k or v.get('status') == 'running'}
        print(f"[{agent}] 保留 {len(new_data)}，删 {len(data)-len(new_data)}")

    with open(sf, 'w') as f:
        json.dump(new_data, f)

print("\n完成！重启 Gateway 使变更生效：openclaw gateway restart")
PYEOF
```

## 注意事项

- **绝对不要删除 running 状态的会话**
- 清理前建议备份：`cp sessions.json sessions.json.bak`
- 清理完成后必须重启 Gateway，否则 Control UI 仍显示旧数据
- 如果 `sessions.json` 为空但 Control UI 仍显示旧会话，重启 Gateway 即可