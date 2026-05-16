---
name: competition-monitor
description: "查看竞赛实验进度、分数历史、当前状态。Use when user says \"查看进度\", \"check score\", \"实验记录\", \"competition status\", \"比赛进度\", or wants to see experiment progress and scores."
argument-hint: [detail-level]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Skill
---

# 查看竞赛进度

查看进度: **$ARGUMENTS**

## Workflow

### Step 1: 读取状态文件

```python
import json, os

# 读取分数历史
score_file = 'competition/SCORE_HISTORY.json'
if os.path.exists(score_file):
    with open(score_file) as f:
        history = json.load(f)
else:
    print("❌ 未找到分数历史，请先运行 /competition-iterate")
    exit()

# 读取迭代状态
state_file = 'competition/ITERATION_STATE.json'
state = json.load(open(state_file)) if os.path.exists(state_file) else None
```

### Step 2: 展示当前状态

输出格式:

```
📊 竞赛进度报告
═══════════════════════════════════════

🏆 当前最佳
   本地CV: [best_local_score] (Exp-[id]: [method])
   线上分数: [best_online_score] (Exp-[id])

📈 迭代状态
   已完成轮次: [N] / [MAX_ITERATIONS]
   耐心计数: [patience] / [PATIENCE]
   状态: [running / paused / completed]

📋 最近 5 次实验
   #[N] [method] → CV [score] ([+/-delta]) [gate_score]/10
   #[N-1] [method] → CV [score] ([+/-delta]) [gate_score]/10
   ...

📤 提交记录
   总提交次数: [N]
   今日已提交: [N] / [daily_limit]
   最佳线上: [score] (排名 #[rank] if available)

🔀 路径表现
   Path A (特征工程): [N]次实验, [M]次改进, 最佳提升 +[delta]
   Path B (模型升级): [N]次实验, [M]次改进, 最佳提升 +[delta]
   ...
```

### Step 3: 分数变化趋势

生成文本版分数曲线:

```
分数变化趋势 (metric: [name], direction: [min/max]):

Exp-001 ████████░░░░░░░░ 0.8145 (baseline)
Exp-002 █████████░░░░░░░ 0.8189 (+0.0044)
Exp-003 █████████░░░░░░░ 0.8201 (+0.0012)
Exp-004 ██████████░░░░░░ 0.8234 (+0.0033) ← local best
Exp-005 ██████████░░░░░░ 0.8250 (+0.0016) 🏆 new best

线上 vs 本地:
Exp-002 本地: 0.8189 | 线上: 0.8190 | 差距: +0.0001 ✓
Exp-005 本地: 0.8250 | 线上: 0.8230 | 差距: -0.0020 ⚠️
```

### Step 4: 检查运行中的实验

如果有实验正在运行:
```bash
# 本地
ps aux | grep "train.py" | grep -v grep

# 远程 (如果配置了)
ssh <server> "ps aux | grep train.py | grep -v grep"
```

如果有运行中的实验，报告:
- 进程 PID
- 运行时间
- GPU 使用情况 (nvidia-smi)
- 最新日志输出 (tail log file)

### Step 5: 建议下一步

根据当前状态给出建议:

- 如果迭代暂停: "建议继续 `/competition-iterate`"
- 如果耐心快耗尽: "当前路径效果不佳，建议切换方向"
- 如果有高分未提交: "Exp-[id] 评分 [X]/10，接近提交标准"
- 如果线上分数低于本地: "存在过拟合风险，建议加强正则化"
- 如果所有路径都试过: "建议搜索新方案 (WebSearch) 或调整策略"

### 可选: 详细模式

如果 $ARGUMENTS 包含 "detail" 或 "详细":
- 展示完整的 EXPERIMENT_LOG.md 内容
- 展示每个实验的详细配置差异
- 展示特征重要性排名 (如果有)
- 调用 `/competition-visualize all` 更新所有图表
