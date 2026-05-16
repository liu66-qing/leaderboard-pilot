---
name: competition-iterate
description: "竞赛核心迭代循环：自动实现实验→训练→评估→门控决策→继续。Use when user says \"迭代实验\", \"iterate\", \"开始迭代\", \"自动跑实验\", \"competition iterate\", or wants autonomous experiment iteration."
argument-hint: [iteration-count-or-focus]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, Agent, Skill, mcp__codex__codex, mcp__codex__codex-reply
---

# 竞赛迭代循环

自动迭代实验: **$ARGUMENTS**

## Constants

- **MAX_ITERATIONS = 20** — 最大迭代次数
- **PATIENCE = 5** — 连续无改进次数达到此值则停止当前路径
- **SUBMIT_THRESHOLD = 9** — 提交门控阈值 (传递给 /competition-submit)
- **AUTO_PROCEED = true** — 自动继续下一轮，不等待用户确认
- **RE_PLAN_INTERVAL = 5** — 每N轮重新评估实验路径
- **CODE_REVIEW = false** — 是否用 GPT-5.4 审查实验代码
- **GPU_ENV = auto** — 运行环境: `local` / `remote` / `auto`(从CLAUDE.md读取)

> Override: `/competition-iterate 10 — patience: 3, code_review: true`

## Inputs

必需文件:
1. `competition/EXPERIMENT_PATH.md` — 实验路径计划
2. `competition/SCORE_HISTORY.json` — 分数历史
3. `CLAUDE.md` — 比赛配置

如果缺少，提示用户先运行 `/competition-plan`。

## Workflow

### Phase 0: 恢复状态

检查 `competition/ITERATION_STATE.json` 是否存在:

```python
import json, os

state_file = 'competition/ITERATION_STATE.json'
if os.path.exists(state_file):
    with open(state_file) as f:
        state = json.load(f)
    print(f"恢复迭代: 第 {state['current_iteration']} 轮")
    print(f"当前最佳: {state['best_local_cv']}")
else:
    state = {
        "current_iteration": 0,
        "current_experiment": None,
        "status": "starting",
        "best_local_cv": None,
        "best_online_score": None,
        "experiments_completed": 0,
        "experiments_remaining": [],
        "patience_counter": 0,
        "path_scores": {}
    }
```

### Phase 1: 选择下一个实验

从 `EXPERIMENT_PATH.md` 中选择下一个要执行的实验:

**选择策略:**
1. 按优先级: MUST-RUN > SHOULD-TRY > NICE-TO-HAVE
2. 按依赖: 先完成前置实验
3. 按路径表现: 优先选择历史表现好的路径
4. 跳过已完成的实验

```
选择逻辑:
1. 读取 EXPERIMENT_PATH.md 中所有实验节点
2. 过滤掉已完成的 (在 SCORE_HISTORY.json 中)
3. 过滤掉依赖未满足的
4. 按优先级排序
5. 如果当前路径连续 PATIENCE 次无改进，切换到下一条路径
6. 选择排名第一的实验
```

向用户报告:
```
🔄 迭代 #[N]: [实验名称]
路径: [Path X]
方法: [简要描述]
预期: [预期改进]
```

### Phase 2: 实现实验代码

根据实验计划，修改或创建代码:

**代码修改范围:**
- `src/models/` — 模型定义
- `src/features/` — 特征工程
- `src/train.py` — 训练逻辑
- `src/predict.py` — 推理逻辑
- `configs/exp_XXX.yaml` — 实验配置

**实现原则:**
- 保持代码整洁，每个实验有独立配置文件
- 不破坏之前能跑通的代码
- 新模型放在新文件中，通过配置切换
- 确保可复现 (固定 seed, 记录所有超参数)

**如果 CODE_REVIEW = true:**
调用 Codex MCP 让 GPT-5.4 审查代码:
```
请审查以下实验代码，检查:
1. 是否有逻辑错误
2. 是否有数据泄露风险
3. 训练/验证划分是否正确
4. 是否有明显的性能问题
```

### Phase 3: 运行训练

**检测运行环境:**
读取 CLAUDE.md 中的 `gpu` 配置:
- `local`: 直接运行
- `remote`: 通过 SSH 运行 (复用 /run-experiment 的逻辑)

**本地运行:**
```bash
python src/train.py --config configs/exp_<id>.yaml --seed 42 2>&1 | tee outputs/logs/exp_<id>.log
```

**远程运行:**
```bash
# 同步代码
rsync -avz --exclude='data/' --exclude='outputs/models/' ./ <server>:<remote_dir>/

# 远程执行
ssh <server> "cd <remote_dir> && python src/train.py --config configs/exp_<id>.yaml --seed 42"

# 拉回结果
rsync -avz <server>:<remote_dir>/outputs/ ./outputs/
```

**监控训练:**
- 检查是否有错误输出
- 如果训练失败: 分析错误，尝试修复，重新运行 (最多重试2次)
- 如果 OOM: 减小 batch_size 或模型大小

### Phase 4: 收集和评估结果

训练完成后，收集结果:

```python
import json
import numpy as np

# 读取交叉验证结果
results_file = f'outputs/results/exp_{exp_id}.json'
with open(results_file) as f:
    results = json.load(f)

cv_scores = results['fold_scores']  # [0.82, 0.81, 0.83, 0.82, 0.81]
cv_mean = np.mean(cv_scores)
cv_std = np.std(cv_scores)

print(f"CV Mean: {cv_mean:.4f} (±{cv_std:.4f})")
```

### Phase 5: 更新分数历史

```python
# 更新 SCORE_HISTORY.json
history = load_json('competition/SCORE_HISTORY.json')

new_entry = {
    "id": f"exp_{exp_id:03d}",
    "timestamp": datetime.now().isoformat(),
    "method": method_description,
    "local_cv": cv_mean,
    "local_cv_std": cv_std,
    "online_score": None,
    "submitted": False,
    "gate_score": None,
    "config": f"configs/exp_{exp_id:03d}.yaml",
    "path": current_path,
    "iteration": current_iteration
}

history['experiments'].append(new_entry)

# 更新最佳
if is_better(cv_mean, history['best_local']):
    history['best_local'] = {"id": new_entry['id'], "score": cv_mean}

save_json('competition/SCORE_HISTORY.json', history)
```

### Phase 6: 提交门控

调用 `/competition-submit` 的评分逻辑:

```
gate_score = evaluate_improvement(
    current_score=cv_mean,
    best_score=history['best_local']['score'] if history['best_local'] else None,
    cv_std=cv_std,
    method_novelty=novelty_level,
    metric_direction=config['metric_direction']
)
```

根据 gate_score 决策:
- **>= 9**: 调用 `/competition-submit` 执行提交
- **>= 8**: 通知用户 "接近提交标准"
- **< 8**: 继续迭代

### Phase 7: 更新实验记录

追加到 `competition/EXPERIMENT_LOG.md`:

```markdown
---

## Exp-[ID] — [date] [time]

- **迭代轮次**: #[N]
- **实验路径**: Path [X] → [node]
- **方法**: [详细描述做了什么改动]
- **配置**: `configs/exp_[id].yaml`
- **本地CV**: [score] (5-fold mean, std=[std])
- **对比最佳**: [+/-delta] vs [best_id] ([best_score])
- **改进评分**: [gate_score]/10
- **提交**: [是/否]
- **耗时**: [minutes]分钟
- **关键发现**: [这次实验学到了什么]
- **下一步**: [基于这次结果，下一步打算做什么]
- **复现命令**: `python src/train.py --config configs/exp_[id].yaml --seed 42`
```

### Phase 8: 决定是否继续

```python
# 更新耐心计数器
if not improved:
    state['patience_counter'] += 1
else:
    state['patience_counter'] = 0

# 停止条件
should_stop = (
    state['current_iteration'] >= MAX_ITERATIONS or
    state['patience_counter'] >= PATIENCE or
    len(state['experiments_remaining']) == 0
)

if should_stop:
    # 生成最终报告
    generate_final_report()
else:
    # 继续下一轮
    state['current_iteration'] += 1
    save_state()
    # 如果 AUTO_PROCEED: 直接进入下一轮
    # 否则: 等待用户确认
```

### Phase 9: 定期重新规划 (每 RE_PLAN_INTERVAL 轮)

每5轮重新评估实验路径:

1. 分析哪些路径产生了改进
2. 哪些路径连续失败
3. 是否有新的想法 (基于已有结果)
4. 更新 `EXPERIMENT_PATH.md` 中的优先级
5. 可能添加新的实验节点

```
📊 路径评估 (第 [N] 轮):
- Path A (特征工程): 3次实验, 2次改进, 平均提升 +0.003
- Path B (模型升级): 2次实验, 0次改进, 建议暂停
- Path C (集成): 未开始

决策: 继续 Path A, 暂停 Path B, 开始 Path C
```

### Phase 10: 保存迭代状态

每轮结束更新 `competition/ITERATION_STATE.json`:

```json
{
  "current_iteration": 7,
  "current_experiment": "exp_007",
  "status": "completed",
  "best_local_cv": 0.8234,
  "best_online_score": 0.8190,
  "experiments_completed": 7,
  "experiments_remaining": ["B2", "C1", "A3"],
  "patience_counter": 2,
  "path_scores": {
    "A": {"attempts": 3, "improvements": 2, "best_delta": 0.0089},
    "B": {"attempts": 2, "improvements": 0, "best_delta": -0.0012}
  },
  "timestamp": "2026-05-16T15:30:00"
}
```

---

## 最终报告 (迭代结束时)

当迭代循环结束，生成总结:

```
🏁 迭代完成

总轮次: [N]
最佳本地CV: [score] (Exp-[id])
最佳线上分数: [score] (Exp-[id])
提交次数: [N]
停止原因: [达到最大轮次 / 耐心耗尽 / 所有实验完成]

Top 3 实验:
1. Exp-[id]: [method] — CV [score]
2. Exp-[id]: [method] — CV [score]
3. Exp-[id]: [method] — CV [score]

关键发现:
- [finding 1]
- [finding 2]

建议下一步:
- [suggestion]
```

调用 `/competition-visualize score` 更新分数曲线。
