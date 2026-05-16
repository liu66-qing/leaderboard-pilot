---
name: competition-submit
description: "竞赛提交门控：评估实验改进程度(1-10分)，仅在显著提升(>=9/10)时提醒提交。Use when user says \"提交\", \"submit\", \"提交结果\", \"check if should submit\", or when competition-iterate triggers submission evaluation."
argument-hint: [experiment-id-or-latest]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Agent, mcp__Claude_in_Chrome__navigate, mcp__Claude_in_Chrome__computer, mcp__Claude_in_Chrome__get_page_text, mcp__Claude_in_Chrome__read_page, mcp__Claude_in_Chrome__tabs_context_mcp, mcp__Claude_in_Chrome__tabs_create_mcp, mcp__Claude_in_Chrome__find, mcp__Claude_in_Chrome__file_upload
---

# 提交门控与自动提交

评估并决定是否提交: **$ARGUMENTS**

## Constants

- **SUBMIT_THRESHOLD = 9** — 改进评分达到此值才提醒/自动提交 (1-10)
- **AUTO_SUBMIT = false** — 是否自动提交 (仅 Kaggle 支持全自动)
- **NOTIFY_THRESHOLD = 8** — 达到此分数时通知用户但不提交
- **DAILY_LIMIT_BUFFER = 1** — 保留的每日提交次数余量

## Workflow

### Step 1: 收集实验结果

读取最新实验结果:
```python
import json

# 读取分数历史
with open('competition/SCORE_HISTORY.json') as f:
    history = json.load(f)

# 获取当前实验结果
current_exp = history['experiments'][-1]
current_score = current_exp['local_cv']
current_std = current_exp['local_cv_std']
```

读取 CLAUDE.md 获取:
- `metric_direction` (minimize / maximize)
- `daily_submission_limit`
- `platform`

### Step 2: 计算改进评分 (1-10)

**评分公式:**

```
gate_score = w1 * improvement_score + w2 * magnitude_score + w3 * stability_score + w4 * novelty_score

权重:
- w1 = 0.40 (本地CV改进)
- w2 = 0.25 (改进幅度)
- w3 = 0.20 (稳定性)
- w4 = 0.15 (方法新颖度)
```

**各因子计算:**

**improvement_score (是否比当前最佳有改进):**
- 无改进或退步: 1-3
- 微小改进 (<0.5% relative): 4-5
- 中等改进 (0.5-2%): 6-7
- 较大改进 (2-5%): 8
- 显著改进 (>5%): 9-10

**magnitude_score (绝对改进大小):**
- 根据比赛 metric 的量级判断
- 例如 RMSE 从 0.12 降到 0.11 是显著的
- 例如 AUC 从 0.95 到 0.955 也是显著的
- 需要结合比赛 leaderboard 的分数密度判断

**stability_score (跨fold稳定性):**
- std/mean < 1%: 10
- std/mean 1-3%: 8
- std/mean 3-5%: 6
- std/mean > 5%: 4
- 单fold无交叉验证: 3

**novelty_score (方法新颖度):**
- 全新模型架构或方法: 9-10
- 新的特征工程方向: 7-8
- 超参数调优: 4-5
- 微小代码修改: 2-3

### Step 3: 做出决策

```
if gate_score >= SUBMIT_THRESHOLD (9):
    → 生成提交文件
    → 执行提交 (或提醒用户)
elif gate_score >= NOTIFY_THRESHOLD (8):
    → 通知用户: "接近提交标准，改进评分 8/10"
    → 不自动提交，但保存提交文件备用
else:
    → 仅记录到 EXPERIMENT_LOG.md
    → 继续下一轮迭代
```

### Step 4: 生成提交文件

```bash
python src/predict.py --config configs/<latest_config>.yaml --output outputs/submissions/submission_<exp_id>.csv
```

验证提交文件格式:
- 行数是否匹配测试集
- 列名是否正确
- 数值范围是否合理
- 无 NaN 值

### Step 5: 执行提交

**Kaggle (AUTO_SUBMIT=true):**
```bash
# 检查今日剩余提交次数
kaggle competitions submissions <slug> -v --csv | wc -l

# 提交
kaggle competitions submit <slug> \
  -f outputs/submissions/submission_<exp_id>.csv \
  -m "Exp-<id>: <method_description> | Local CV: <score>"

# 等待评分 (通常几秒到几分钟)
sleep 30

# 获取线上分数
kaggle competitions submissions <slug> -v --csv | head -2
```

**Kaggle (AUTO_SUBMIT=false) 或 天池:**
```
📊 建议提交!

改进评分: [X]/10
方法: [method_description]
本地CV: [old_best] → [new_score] (+[delta])
稳定性: std=[std_value]
提交文件: outputs/submissions/submission_<exp_id>.csv

[Kaggle] 请运行: kaggle competitions submit <slug> -f outputs/submissions/submission_<exp_id>.csv -m "<msg>"
[天池] 请在天池平台手动上传提交文件
```

**天池浏览器自动化提交 (如果用户授权):**
1. `mcp__Claude_in_Chrome__navigate` 到天池提交页面
2. `mcp__Claude_in_Chrome__find` 找到文件上传按钮
3. `mcp__Claude_in_Chrome__file_upload` 上传提交文件
4. `mcp__Claude_in_Chrome__find` 找到提交按钮并点击
5. 等待结果页面加载，提取线上分数

### Step 6: 记录提交结果

更新 `competition/SUBMISSION_LOG.md`:
```markdown
## 提交 #[N] — [date]

- **实验ID**: exp_[id]
- **方法**: [description]
- **本地CV**: [score] (std=[std])
- **线上分数**: [online_score]
- **本地vs线上差距**: [delta] ([overfitting/underfitting/consistent])
- **改进评分**: [gate_score]/10
- **排名变化**: [old_rank] → [new_rank] (如果可获取)
```

更新 `competition/SCORE_HISTORY.json`:
```python
history['experiments'][-1]['online_score'] = online_score
history['experiments'][-1]['submitted'] = True
if is_better(online_score, history['best_online']):
    history['best_online'] = {'id': exp_id, 'score': online_score}
```

### Step 7: 过拟合检测

比较本地CV和线上分数:
```
gap = abs(local_cv - online_score) / local_cv

if gap > 0.05:  # 5% 以上差距
    ⚠️ 警告: 本地CV与线上分数差距较大，可能存在过拟合
    建议:
    - 检查验证集划分是否合理
    - 增加正则化
    - 检查是否有数据泄露
```

### 输出

无论是否提交，都向用户报告:
- 改进评分及各因子得分
- 是否达到提交标准
- 如果提交了: 线上分数和排名
- 下一步建议
