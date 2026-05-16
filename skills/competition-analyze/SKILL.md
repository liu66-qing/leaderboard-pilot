---
name: competition-analyze
description: "解析 Kaggle/天池竞赛赛题，输出全方位赛题分析文档。Use when user says \"赛题解析\", \"analyze competition\", \"解析比赛\", \"分析赛题\", or wants to understand a competition before starting."
argument-hint: [competition-url-or-name]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, WebSearch, WebFetch, Agent, mcp__Claude_in_Chrome__navigate, mcp__Claude_in_Chrome__computer, mcp__Claude_in_Chrome__get_page_text, mcp__Claude_in_Chrome__read_page, mcp__Claude_in_Chrome__tabs_context_mcp, mcp__Claude_in_Chrome__tabs_create_mcp, mcp__Claude_in_Chrome__find
---

# 赛题解析

解析竞赛: **$ARGUMENTS**

## Constants

- **PLATFORM = auto** — 自动检测平台 (kaggle / tianchi)。可手动指定。
- **SEARCH_DEPTH = deep** — 搜索深度: `quick`(仅官方页面), `deep`(搜索论文+博客+历史方案)
- **LANGUAGE = zh** — 输出语言，默认中文

> Override: `/competition-analyze "house-prices" — platform: kaggle, search_depth: quick`

## Workflow

### Step 1: 检测平台和获取赛题信息

**判断平台:**
- URL 包含 `kaggle.com` → Kaggle
- URL 包含 `tianchi.aliyun.com` → 天池
- 无 URL → 用 WebSearch 搜索比赛名称，判断来源

**Kaggle 路径:**
```bash
# 检查 kaggle CLI 可用
kaggle --version

# 搜索比赛
kaggle competitions list --search "<competition_name>" -v

# 获取比赛详情
kaggle competitions list --search "<slug>" -v --csv
```

然后用 WebFetch 抓取比赛页面获取完整描述:
- `https://www.kaggle.com/competitions/<slug>/overview`
- `https://www.kaggle.com/competitions/<slug>/data`
- `https://www.kaggle.com/competitions/<slug>/discussion` (top discussions)

**天池路径 (分层降级策略):**

天池是 SPA 应用，WebFetch 无法获取动态内容。按以下优先级尝试:

**优先级 1: Chrome 浏览器自动化 (最佳)**
1. 先检查 Chrome 是否已连接: `mcp__Claude_in_Chrome__tabs_context_mcp`
2. 如果已连接:
   - `mcp__Claude_in_Chrome__tabs_create_mcp` 创建新标签
   - `mcp__Claude_in_Chrome__navigate` 导航到天池比赛页面
   - `mcp__Claude_in_Chrome__get_page_text` 提取页面文本
   - 如果页面显示需要登录: 提醒用户在浏览器中登录天池，等待后重试
3. 如果 Chrome 未连接: **立即跳到优先级 2，不要尝试 WebFetch**

**优先级 2: 多维度网络搜索 (Chrome不可用时)**

不要用 WebFetch 直接访问天池 URL（会失败）。改用多角度搜索:

```
搜索策略 (并行执行多个搜索):
1. WebSearch: "天池 <比赛名> 赛题 任务描述"
2. WebSearch: "天池 <比赛ID> 数据 评价指标"
3. WebSearch: "<比赛名> baseline 方案 CSDN OR 知乎 OR 博客园"
4. WebSearch: "tianchi <competition_name> solution github"
5. WebSearch: "<比赛名> 数据集 特征 提交格式"
```

从搜索结果中提取:
- CSDN/知乎/博客园上的赛题解读文章 (通常包含完整赛题描述)
- GitHub 上的 baseline 代码仓库 (README 通常有赛题说明)
- 比赛讨论帖 (DataWhale, Coggle 等社区)

对搜索到的博客/文章页面使用 WebFetch 提取内容（这些页面不需要登录）。

**优先级 3: 用户手动提供 (兜底)**

如果搜索也无法获取足够信息，**立即**请求用户协助，不要继续无效尝试:

```
⚠️ 无法自动获取天池赛题信息

请通过以下任一方式提供赛题信息:
1. 在浏览器中打开比赛页面，复制粘贴以下内容给我:
   - 任务描述
   - 评估指标
   - 数据说明
   - 提交格式要求
2. 或者: 连接 Chrome 浏览器 (确保 Claude in Chrome 扩展已安装并登录天池)
3. 或者: 提供比赛相关的博客/文章链接

我会基于你提供的信息继续分析。
```

**关键原则: 快速失败，不浪费时间在注定失败的尝试上。**

### Step 2: 下载并分析数据

**Kaggle:**
```bash
mkdir -p data/raw
kaggle competitions download <slug> -p data/raw/
# 解压
cd data/raw && unzip -o "*.zip" 2>/dev/null; cd -
```

**天池:**

当用户提供赛题信息时，**主动扫描其中的下载链接并自动下载**:

```
自动下载逻辑:
1. 扫描用户提供的文本，提取所有可能的数据下载 URL:
   - 直接文件链接 (*.zip, *.csv, *.json, *.tar.gz, *.rar, *.7z)
   - 天池数据下载链接 (通常包含 tianchi 或 aliyun 域名)
   - 百度网盘/Google Drive 链接 (提取后提醒用户手动下载)
   - 任何 HTTP/HTTPS 链接指向数据文件

2. 对每个可下载链接，自动执行:
   mkdir -p data/raw
   curl -L -o "data/raw/<filename>" "<url>"
   # 或如果文件名不明确:
   curl -L -J -O --output-dir data/raw/ "<url>"

3. 下载后自动解压:
   cd data/raw
   # zip 文件
   unzip -o "*.zip" 2>/dev/null
   # tar.gz 文件
   tar -xzf *.tar.gz 2>/dev/null
   # rar 文件 (如果 unrar 可用)
   unrar x *.rar 2>/dev/null
   cd -

4. 验证下载结果:
   ls -la data/raw/
   # 确认文件大小合理 (不是 HTML 错误页面)
   # 如果文件 < 1KB，可能是下载失败，检查内容

5. 如果下载链接需要认证 (返回 403/401):
   - 尝试通过 Chrome 浏览器下载 (如果已连接)
   - 否则告知用户: "链接 [url] 需要登录才能下载，请手动下载到 data/raw/"
```

**关键: 用户提供了下载链接就直接下，不要再问"数据是否已下载"。**
只有在链接确实无法自动下载时（需要登录、百度网盘等），才请求用户手动操作。

**数据分析 (通用):**
```python
import pandas as pd
import os

# 列出所有数据文件
for f in os.listdir('data/raw'):
    print(f, os.path.getsize(f'data/raw/{f}'))

# 读取训练集
train = pd.read_csv('data/raw/train.csv')
print(f"Shape: {train.shape}")
print(f"Columns: {list(train.columns)}")
print(f"Dtypes:\n{train.dtypes}")
print(f"Missing:\n{train.isnull().sum()}")
print(f"Target distribution:\n{train[TARGET_COL].describe()}")
```

### Step 3: 搜索相关资源

使用 WebSearch 搜索:
1. `"<competition_name>" top solution` — 历史优胜方案
2. `"<competition_name>" discussion kaggle` — 讨论区精华
3. `"<task_type>" machine learning approach 2024 2025` — 最新方法
4. `"<competition_name>" 方案 解题思路` — 中文社区方案
5. 相关学术论文 (如果是特定领域如 NLP/CV)

将搜索结果整理到 `competition/references/` 目录。

### Step 4: 输出赛题深度解析文档

**重要: 这不是赛题信息的复述！是深度分析报告。**

你必须像一个打过 100 场比赛的 Kaggle Grandmaster 一样思考：
- 这个任务的本质是什么？表面是 X，实际上是 Y
- 数据有什么坑？什么会导致线上线下不一致？
- 测试集 A/B 的分布可能有什么差异？
- 什么样的方案能稳定提分，什么方案看起来好但容易翻车？

**⚠️ 最重要: 提取并严格遵守赛题硬性约束！**

生成 `competition/COMPETITION_ANALYSIS.md`，结构如下:

```markdown
# 赛题深度解析: [比赛名称]

## 基本信息
[简要表格: 平台、链接、指标、截止日期、提交限制]

## ⚠️ 硬性约束 (红线，不可违反)

从赛题规则中提取所有硬性限制，后续所有方案必须在此范围内:

- **模型参数量限制**: [例如: ≤20B, ≤7B, 不限制]
- **推理时间限制**: [例如: 单样本≤5s, 总时间≤2h]
- **硬件限制**: [例如: 单卡A100, 4×V100]
- **禁止使用**: [例如: 不能用GPT-4 API, 不能用外部数据]
- **必须使用**: [例如: 必须基于指定baseline]
- **提交格式**: [严格的格式要求]
- **其他规则**: [任何违反即取消资格的规则]

> 🚨 **所有后续实验方案必须在这些约束内设计。**
> 如果某个优化方向会违反约束，即使效果再好也不能用。

## 任务本质分析

### 表面任务 vs 真正挑战
- **表面任务**: [官方描述的任务]
- **真正要解决的问题**: [深入分析这个任务的核心难点是什么]
- **为什么难**: [不是简单的 X，而是需要同时处理 Y 和 Z]

### 任务拆解
将复杂任务拆解为子问题:
1. 子问题 A: [描述] — 难度 [高/中/低]，为什么
2. 子问题 B: [描述] — 难度 [高/中/低]，为什么
3. 子问题之间的耦合关系: [如何相互影响]

### 评价指标深度分析
- **指标公式**: [数学表达]
- **指标特性**: 对什么敏感？对什么不敏感？
- **优化陷阱**: [例如 F1 需要调阈值，RMSE 对异常值敏感等]
- **线上线下一致性**: 这个指标在小样本上方差大吗？
- **提分策略**: 针对这个指标，什么操作能直接提分？

## 数据深度分析

### 数据集特点与雷区

⚠️ **雷区警告** (可能导致线上翻车的问题):
- [雷区1: 例如训练集和测试集分布不一致]
- [雷区2: 例如存在数据泄露的特征]
- [雷区3: 例如类别极度不平衡]
- [雷区4: 例如时间序列数据的未来信息泄露]

### 训练集分析
- 样本量是否足够？对于这个任务复杂度来说
- 标注质量: 是否有噪声标注？标注一致性如何？
- 分布特点: [长尾? 均匀? 聚类?]
- 关键统计: [具体数字]

### 测试集 A vs 测试集 B 分析 (如果有)
- **测试集A**: [发布时间、样本量、用途(排行榜)]
- **测试集B**: [发布时间、样本量、用途(最终评测)]
- **A→B 可能的分布偏移**:
  - [时间偏移: B 可能包含更新的数据]
  - [难度偏移: B 可能更难/更简单]
  - [分布偏移: B 可能有新的类别/模式]
- **防过拟合策略**:
  - 不要过度拟合测试集A的排行榜分数
  - 本地验证集的划分应该模拟 A→B 的偏移
  - [具体建议: 如何划分验证集才能更好预测B的表现]

### 过拟合风险评估
- **高风险因素**: [小数据量? 高维特征? 复杂模型?]
- **防过拟合措施**:
  1. [具体措施1: 例如 K-fold 交叉验证的正确姿势]
  2. [具体措施2: 例如正则化策略]
  3. [具体措施3: 例如数据增强方法]
- **验证集划分建议**: [如何划分才能最接近线上表现]

## 方案策略分析

### 推荐 Baseline (为什么选它)
- **方案**: [具体方案描述]
- **为什么从这个开始**:
  - 理由1: [例如: 这类任务的 SOTA 基本都基于此架构]
  - 理由2: [例如: 实现简单，能快速验证数据流水线]
  - 理由3: [例如: 社区验证过，baseline 分数可预期]
- **预期分数范围**: [基于类似比赛经验]
- **实现要点**: [关键的实现细节，不是泛泛而谈]

### 提分方向优先级 (为什么这么排)

| 优先级 | 方向 | 预期收益 | 为什么有效 | 风险 |
|--------|------|----------|------------|------|
| P0 | [方向] | [+X%] | [原因] | [风险] |
| P1 | [方向] | [+X%] | [原因] | [风险] |
| P2 | [方向] | [+X%] | [原因] | [风险] |

### 不推荐的方向 (为什么避开)
- ❌ [方向1]: [为什么在这个比赛中不适用]
- ❌ [方向2]: [为什么投入产出比低]

### 关键技术决策点
- **模型选择**: [为什么选 A 不选 B，基于数据特点]
- **特征工程 vs 模型能力**: [这个任务更依赖哪个？为什么？]
- **单模型 vs 集成**: [在这个数据量和任务下，什么时候该集成？]
- **预训练模型选择** (如果适用): [哪个预训练模型最匹配？为什么？]

## 竞赛策略建议

### 时间分配建议
- 第1天: [做什么]
- 第2-3天: [做什么]
- 第4-7天: [做什么]
- 最后冲刺: [做什么]

### 提交策略
- 每日提交次数有限，如何最大化信息量
- 什么时候提交探索性结果，什么时候提交稳定结果
- 如何利用 A 榜反馈但不过拟合 A 榜

### 团队协作建议 (如果适用)
- 可并行的工作方向
- 需要串行的依赖关系
```

**写作原则:**
1. 每个观点都要有"为什么" — 不是列清单，是给出推理
2. 具体到可执行 — "用 XGBoost" 不够，要说 "用 XGBoost 因为数据量 7000 且特征以类别为主，树模型天然处理类别特征"
3. 指出反直觉的点 — 什么看起来应该有效但实际上不行
4. 基于数据说话 — 读完数据后的分析，不是泛泛的建议

### Step 5: 初始化项目结构

创建项目目录结构:
```bash
mkdir -p competition/references data/{raw,processed,splits} src/{models,features} configs outputs/{models,predictions,submissions} figures
```

如果 `CLAUDE.md` 不存在，创建初始配置:
```markdown
## Competition Config
- platform: [kaggle/tianchi]
- competition_slug: [slug]
- competition_url: [url]
- metric: [metric_name]
- metric_direction: [minimize/maximize]
- submission_format: csv
- daily_submission_limit: [N]
- auto_submit: false
- local_cv_strategy: 5-fold
- target_column: [column_name]

## Environment
- gpu: local  # or remote
- ssh_alias: [server]  # if remote
- conda_env: [env_name]
- python: python3
```

### Step 6: 总结并展示

向用户展示:
1. 赛题解析文档已生成的位置
2. 数据概况摘要
3. 推荐的下一步操作 (通常是 `/competition-plan`)
