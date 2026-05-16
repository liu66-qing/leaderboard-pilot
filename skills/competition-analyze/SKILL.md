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

### Step 4: 输出赛题解析文档

生成 `competition/COMPETITION_ANALYSIS.md`，结构如下:

```markdown
# 赛题解析: [比赛名称]

## 基本信息
- **平台**: Kaggle / 天池
- **比赛链接**: [URL]
- **截止日期**: [date]
- **评价指标**: [metric] (越大/小越好)
- **提交限制**: 每日 [N] 次
- **奖金/奖品**: [info]

## 赛题概述
[用通俗易懂的语言描述比赛任务，让用户快速理解要做什么]

## 评价指标详解
[详细解释评价指标的计算方式、优化方向、常见陷阱]
[包含数学公式如果有的话]

## 数据分析
### 数据概览
- 训练集: [N] 行 × [M] 列
- 测试集: [N] 行 × [M] 列
- 文件列表: [...]

### 特征分析
[每个重要特征的类型、分布、缺失率]

### 目标变量分析
[分布、类别平衡、异常值]

### 数据质量问题
[缺失值、异常值、数据泄露风险]

## 相关比赛经验
[类似历史比赛的 top solution 总结]

## 常见方案总结
### 基线方案
[最简单能跑通的方案]

### 进阶方案
[中等复杂度的提分方案]

### 高阶方案
[top solution 级别的方案]

## 潜在难点与风险
- [难点1: 描述 + 应对策略]
- [难点2: ...]

## 推荐起步方案
[具体的第一步实现建议，包括模型选择、特征工程方向]
```

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
