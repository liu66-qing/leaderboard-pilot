---
name: competition-pipeline
description: "Kaggle/天池竞赛全流程自动化：赛题解析→实验规划→自动迭代→智能提交。Use when user says \"打榜\", \"开始比赛\", \"competition pipeline\", \"全流程打榜\", \"kaggle\", \"天池比赛\", \"start competition\", or wants the full competition automation workflow."
argument-hint: [competition-url-or-name]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, Agent, Skill, mcp__codex__codex, mcp__codex__codex-reply, mcp__Claude_in_Chrome__navigate, mcp__Claude_in_Chrome__computer, mcp__Claude_in_Chrome__get_page_text, mcp__Claude_in_Chrome__read_page, mcp__Claude_in_Chrome__tabs_context_mcp, mcp__Claude_in_Chrome__tabs_create_mcp, mcp__Claude_in_Chrome__find, mcp__Claude_in_Chrome__file_upload
---

# 竞赛全流程自动化

竞赛: **$ARGUMENTS**

## Overview

全自动竞赛打榜流水线，从赛题解析到提交优化:

```
/competition-analyze → /competition-plan → /competition-iterate (loop) → /competition-submit
     (赛题解析)          (路径规划)           (自动迭代)                    (智能提交)
```

每个阶段的输出是下一阶段的输入。迭代循环是核心，会自动实现实验、训练、评估、决策。

## Constants

- **MAX_ITERATIONS = 20** — 最大迭代轮次
- **SUBMIT_THRESHOLD = 9** — 提交门控阈值 (1-10分，>=9才提醒提交)
- **AUTO_SUBMIT = false** — 是否自动提交 (Kaggle可设true，天池建议false)
- **AUTO_PROCEED = true** — 各阶段间自动继续，不等待用户确认
- **PLATFORM = auto** — 平台: `kaggle` / `tianchi` / `auto`(自动检测)
- **GPU_ENV = auto** — GPU环境: `local` / `remote` / `auto`
- **PATIENCE = 5** — 连续无改进停止阈值
- **CODE_REVIEW = false** — 是否用 GPT-5.4 审查代码

> Override: `/competition-pipeline "https://kaggle.com/c/house-prices" — auto_submit: true, max_iterations: 30`

## 核心行为原则

1. **有链接就下载** — 用户提供的文本中包含数据下载 URL 时，直接 curl 下载到 `data/raw/`，不要问"是否已下载"
2. **快速失败** — 天池 URL 不要用 WebFetch（SPA 必失败），Chrome 没连接就直接搜索或请求粘贴
3. **不中断流程** — 能自动做的绝不停下来问用户，只在真正需要人工介入时才暂停
4. **拿到数据就继续** — 数据就位后立即开始 EDA 和 Baseline，不要等待额外确认

## Pipeline

### Phase 0: 环境检测与初始化

**检测平台:**
- 从 $ARGUMENTS 中的 URL 判断平台
- 或从已有 CLAUDE.md 读取配置
- 或询问用户

**验证环境:**

Kaggle:
```bash
kaggle --version
# 如果失败: pip install kaggle
# 检查认证: ls ~/.kaggle/kaggle.json
```

天池:
- 先检查 Chrome 浏览器是否已连接 (`mcp__Claude_in_Chrome__tabs_context_mcp`)
- 如果已连接: 后续可通过浏览器自动获取赛题信息和提交
- 如果未连接: 告知用户将通过网络搜索+用户粘贴方式获取赛题信息
- **不要尝试 WebFetch 天池 URL** (SPA应用，必定失败)

**GPU 环境:**
```bash
# 本地 GPU
nvidia-smi --query-gpu=name,memory.total --format=csv,noheader 2>/dev/null

# 如果无本地 GPU，检查 CLAUDE.md 中的远程配置
```

**创建项目目录:**
```bash
mkdir -p competition/references data/{raw,processed,splits} src/{models,features} configs outputs/{models,predictions,submissions,logs,results} figures
```

**检查是否是恢复运行:**
如果 `competition/ITERATION_STATE.json` 存在且 status != "completed":
```
检测到未完成的迭代 (第 [N] 轮)
最佳本地CV: [score]
是否从断点继续? (AUTO_PROCEED=true 时自动继续)
```
→ 跳转到 Phase 3 继续迭代

---

### Phase 1: 赛题解析

调用: `/competition-analyze "$ARGUMENTS"`

**输出:**
- `competition/COMPETITION_ANALYSIS.md` — 完整赛题解析
- `CLAUDE.md` — 比赛配置 (如果不存在则创建)
- `data/raw/` — 下载的数据文件

**检查点:**
```
✅ Phase 1 完成: 赛题解析

比赛: [name]
平台: [platform]
指标: [metric] ([direction])
数据: [N] 训练样本, [M] 特征
任务: [task_type]

文档: competition/COMPETITION_ANALYSIS.md
```

如果 AUTO_PROCEED=false: 等待用户确认后继续。

---

### Phase 2: 实验路径规划

调用: `/competition-plan`

**输出:**
- `competition/EXPERIMENT_PATH.md` — 实验路径拆解
- `competition/SCORE_HISTORY.json` — 初始化分数追踪
- `competition/EXPERIMENT_LOG.md` — 初始化实验记录
- `figures/*.mmd` — 架构可视化

**检查点:**
```
✅ Phase 2 完成: 实验路径规划

实验路径: [N] 条
总实验节点: [M] 个
MUST-RUN: [K] 个
预计总耗时: [hours] 小时

路径概览:
├── Baseline: [description]
├── Path A: [description] ([N] experiments)
├── Path B: [description] ([N] experiments)
└── Path C: [description] ([N] experiments)

文档: competition/EXPERIMENT_PATH.md
```

---

### Phase 3: 自动迭代

调用: `/competition-iterate $MAX_ITERATIONS — patience: $PATIENCE`

这是最长的阶段，会自动循环执行:
1. 选择下一个实验
2. 实现代码
3. 训练模型
4. 评估结果
5. 门控决策 (是否提交)
6. 记录并继续

**迭代过程中的输出:**
- 每轮更新 `competition/EXPERIMENT_LOG.md`
- 每轮更新 `competition/SCORE_HISTORY.json`
- 每轮更新 `competition/ITERATION_STATE.json`
- 达到提交标准时调用 `/competition-submit`

**迭代结束条件:**
- 达到 MAX_ITERATIONS
- 连续 PATIENCE 轮无改进
- 所有计划实验完成
- 用户手动中断

---

### Phase 4: 最终报告

迭代结束后，生成最终总结:

```markdown
# 🏁 竞赛打榜报告

## 总览
- **比赛**: [name]
- **总迭代轮次**: [N]
- **总耗时**: [hours] 小时
- **最佳本地CV**: [score] (Exp-[id])
- **最佳线上分数**: [score] (Exp-[id])
- **总提交次数**: [N]

## Top 5 实验
| 排名 | 实验 | 方法 | 本地CV | 线上分数 |
|------|------|------|--------|----------|
| 1 | Exp-[id] | [method] | [score] | [score] |
| ... | ... | ... | ... | ... |

## 关键发现
- [finding 1]
- [finding 2]
- [finding 3]

## 有效方向
- [what worked]

## 无效方向
- [what didn't work]

## 建议后续
- [next steps if user wants to continue]
```

调用 `/competition-visualize all` 生成最终可视化。

---

## 错误处理

**训练失败:**
- 分析错误日志
- 常见修复: 减小 batch_size, 修复数据类型, 处理 NaN
- 最多重试 2 次，仍失败则跳过该实验

**数据下载失败:**
- Kaggle: 检查 API key, 检查比赛是否需要接受规则
- 天池: 提醒用户手动下载

**提交失败:**
- 检查提交文件格式
- 检查每日提交限额
- 检查比赛是否已结束

**GPU OOM:**
- 自动减小 batch_size (÷2)
- 如果仍 OOM: 减小模型大小或使用梯度累积

---

## 与其他 Skill 的集成

- `/research-lit` — 搜索相关论文 (在 Phase 1 中使用)
- `/mermaid-diagram` — 生成架构图 (在 Phase 2 中使用)
- `/competition-monitor` — 随时查看进度 (用户可独立调用)
- `/competition-visualize` — 更新可视化 (在 Phase 4 中使用)
- `/feishu-notify` — 发送通知 (提交成功时可选)
