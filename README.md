# 🏆 Leaderboard Pilot

> 睡觉的时候，让 AI 帮你打榜 🛌💤➡️📈

一套为 **Kaggle / 阿里天池** 竞赛设计的 Claude Code Skills，实现从赛题解析到自动迭代实验的全流程自动化。

```
你睡觉 😴  →  它跑实验 🔬  →  你醒来 ☀️  →  分数涨了 🚀
```

## ✨ 它能做什么

| 功能 | 说明 |
|------|------|
| 🔍 赛题解析 | 自动抓取比赛信息，输出中文分析文档 |
| 🗺️ 路径规划 | 从 baseline 到 SOTA 的实验路径树 |
| 🔄 自动迭代 | 实现代码→训练→评估→决策，循环往复 |
| 🚦 智能提交 | 只有显著提升 (9/10分) 才提醒你提交 |
| 📊 可视化 | Mermaid 模型架构图 + 实验树 + 分数曲线 |
| 📋 实验记录 | 每轮自动记录方法、结果、发现 |

## 🎯 核心设计理念

**不是每个小改动都值得提交。**

系统会对每次实验改进打分 (1-10)：

```
😢 1-3 分: 模型退步了，回滚
😐 4-5 分: 微小改进，继续迭代
🙂 6-7 分: 有进步但不够，记录继续
😊 8 分:   不错！通知你但不提交
🎉 9-10分: 显著提升！建议提交 🚀
```

只有真正有意义的改进才会打扰你。

## 🚀 快速开始

### 安装

把 `skills/` 目录下的所有文件夹复制到你的 Claude Code skills 目录：

```bash
# Windows
cp -r skills/* C:/Users/<你的用户名>/.claude/skills/

# Mac/Linux
cp -r skills/* ~/.claude/skills/
```

### 使用

```bash
# 🎬 全流程启动 (一句话搞定)
/competition-pipeline "https://www.kaggle.com/competitions/house-prices-advanced-regression-techniques"

# 🔍 只做赛题解析
/competition-analyze "天池 CCKS2026 文本属性抽取"

# 🗺️ 只做实验规划
/competition-plan

# 🔄 开始自动迭代 (可以去睡觉了 💤)
/competition-iterate 20

# 📊 醒来看看进度
/competition-monitor

# 🎨 生成可视化
/competition-visualize all
```

## 📁 项目结构

每个比赛会自动生成这样的目录结构：

```
your-competition/
├── 📄 CLAUDE.md                    # 比赛配置
├── 📂 competition/
│   ├── 📖 COMPETITION_ANALYSIS.md  # 赛题解析 (给你读的)
│   ├── 🗺️ EXPERIMENT_PATH.md       # 实验路径拆解
│   ├── 📋 EXPERIMENT_LOG.md        # 实验记录
│   ├── 📤 SUBMISSION_LOG.md        # 提交记录 + 分数
│   ├── 📊 SCORE_HISTORY.json       # 结构化分数历史
│   └── 🔄 ITERATION_STATE.json     # 迭代状态 (断点恢复)
├── 📂 data/raw/                    # 原始数据
├── 📂 src/                         # 模型代码
├── 📂 configs/                     # 实验配置
├── 📂 outputs/submissions/         # 提交文件
└── 📂 figures/                     # 架构图 + 分数曲线
```

## 🔧 Skill 一览

| Skill | 触发词 | 干啥的 |
|-------|--------|--------|
| `competition-pipeline` | "打榜", "开始比赛" | 🎬 主编排器，串联全流程 |
| `competition-analyze` | "赛题解析" | 🔍 解析赛题 + 下载数据 |
| `competition-plan` | "实验路径", "方案拆解" | 🗺️ 实验路径树 + 架构图 |
| `competition-iterate` | "迭代实验" | 🔄 核心循环 (最能干的) |
| `competition-submit` | "提交" | 🚦 9/10 门控 + 提交 |
| `competition-monitor` | "查看进度" | 📊 分数 + 状态报告 |
| `competition-visualize` | "可视化" | 🎨 Mermaid 图表 |

## 🌐 平台支持

### Kaggle ✅ 全自动

- 数据下载 → `kaggle competitions download`
- 提交 → `kaggle competitions submit`
- 查分 → `kaggle competitions submissions`
- 需要: `~/.kaggle/kaggle.json` (API Key)

### 天池 🔄 半自动

- 赛题获取 → Chrome 浏览器自动化 / 网络搜索 / 用户粘贴
- 数据下载 → 自动识别链接并 curl 下载
- 提交 → Chrome 自动化上传 / 提醒手动提交
- 需要: Claude in Chrome 扩展 (可选)

## ⚙️ 配置

在比赛项目的 `CLAUDE.md` 中配置：

```markdown
## Competition Config
- platform: kaggle
- competition_slug: house-prices-advanced-regression-techniques
- metric: RMSE
- metric_direction: minimize
- daily_submission_limit: 5
- auto_submit: false
- local_cv_strategy: 5-fold
- target_column: SalePrice

## Environment
- gpu: local
```

## 💡 设计灵感

受 [Auto-claude-code-research-in-sleep](https://github.com/wanshuiyin/Auto-claude-code-research-in-sleep) 启发，将自主研究循环的理念应用到竞赛打榜场景。

核心思路：**把竞赛打榜变成一个可以无人值守运行的自动化流水线。**

## 📝 License

MIT

---

*Built with ❤️ and Claude Code*

*去睡觉吧，明天醒来看分数 🌙*
