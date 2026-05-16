---
name: competition-visualize
description: "生成竞赛模型架构图、实验路径树、分数曲线等可视化。Use when user says \"可视化\", \"visualize\", \"画架构图\", \"model architecture\", or wants visual representations of the competition approach."
argument-hint: [what-to-visualize]
allowed-tools: Bash(*), Read, Write, Edit, Grep, Glob, Skill
---

# 竞赛可视化

生成可视化: **$ARGUMENTS**

## Constants

- **FORMAT = mermaid** — 输出格式: `mermaid` (默认), `ascii`, `python` (matplotlib)
- **DETAIL = full** — 细节级别: `brief`(概览), `full`(含维度和参数)

## Workflow

### 判断可视化类型

根据 $ARGUMENTS 或上下文判断需要生成什么:

1. **model** / **架构图** → 生成模型架构图
2. **tree** / **实验树** → 生成实验路径树
3. **score** / **分数** → 生成分数变化图
4. **feature** / **特征** → 生成特征工程流水线图
5. **all** / 无参数 → 生成所有

---

### Type 1: 模型架构图

读取 `src/models/` 下的模型代码，提取:
- 输入维度和类型
- 每一层的类型、参数、输出维度
- 激活函数
- 正则化 (dropout, batch norm)
- 损失函数
- 总参数量

生成 Mermaid 图到 `figures/model-architecture.mmd`:

```mermaid
graph TD
    subgraph 输入层
        A["输入: batch × features<br/>数值特征: N维<br/>类别特征: M维"]
    end

    subgraph 特征处理
        A --> B["数值: StandardScaler"]
        A --> C["类别: Embedding(dim=8)"]
        B --> D["Concat: N+M*8 维"]
        C --> D
    end

    subgraph 模型主体
        D --> E["Linear(input, 256)<br/>params: input×256"]
        E --> F["BatchNorm1d(256)"]
        F --> G["ReLU + Dropout(0.3)"]
        G --> H["Linear(256, 128)<br/>params: 256×128"]
        H --> I["ReLU + Dropout(0.2)"]
        I --> J["Linear(128, 1)<br/>params: 128×1"]
    end

    subgraph 输出
        J --> K["Output: batch × 1"]
        K --> L["Loss: MSELoss"]
    end

    style A fill:#e1f5fe
    style K fill:#e8f5e9
```

对于树模型 (LightGBM/XGBoost):
```mermaid
graph TD
    subgraph 数据流
        A["原始数据<br/>N samples × M features"] --> B["特征工程"]
        B --> C["数值特征: K维"]
        B --> D["类别特征: L维 (label encoded)"]
    end

    subgraph LightGBM
        C --> E["GBDT Ensemble"]
        D --> E
        E --> F["n_estimators=1000<br/>max_depth=7<br/>learning_rate=0.05<br/>num_leaves=63"]
    end

    subgraph 训练策略
        F --> G["5-Fold CV"]
        G --> H["Early Stopping<br/>patience=50"]
        H --> I["最终预测: 5个模型平均"]
    end
```

### Type 2: 实验路径树

读取 `competition/EXPERIMENT_PATH.md` 和 `competition/SCORE_HISTORY.json`。

生成 `figures/experiment-tree.mmd`:

```mermaid
graph TD
    BASE["🏁 Baseline<br/>CV: 0.8145"] --> A1
    BASE --> B1
    BASE --> C1

    subgraph "Path A: 特征工程"
        A1["A1: 统计特征<br/>CV: 0.8201 ✓"] --> A2["A2: 交叉特征<br/>CV: 0.8234 ✓"]
        A2 --> A3["A3: 时序特征<br/>⏳ 待实验"]
    end

    subgraph "Path B: 模型升级"
        B1["B1: XGBoost<br/>CV: 0.8189 ✓"] --> B2["B2: CatBoost<br/>⏳ 待实验"]
        B1 --> B3["B3: TabNet<br/>⏳ 待实验"]
    end

    subgraph "Path C: 集成"
        C1["C1: Stacking<br/>❌ 未开始"]
    end

    style A2 fill:#c8e6c9
    style BASE fill:#fff9c4
```

标记规则:
- ✓ 绿色: 已完成且有正向结果
- ❌ 红色: 已完成但效果不好
- ⏳ 黄色: 待实验
- 🏆 金色: 当前最佳

### Type 3: 分数变化图

读取 `competition/SCORE_HISTORY.json`。

生成 `figures/score-progression.mmd`:

```mermaid
xychart-beta
    title "分数变化趋势"
    x-axis ["Exp1", "Exp2", "Exp3", "Exp4", "Exp5"]
    y-axis "Local CV Score" 0.80 --> 0.90
    line [0.8145, 0.8189, 0.8201, 0.8234, 0.8250]
```

同时生成文本版本 (兼容不支持 Mermaid 的环境):
```
分数变化:
Exp-001 ████████░░ 0.8145 (baseline)
Exp-002 ████████░░ 0.8189 (+0.0044)
Exp-003 █████████░ 0.8201 (+0.0012)
Exp-004 █████████░ 0.8234 (+0.0033) ← best
Exp-005 █████████░ 0.8250 (+0.0016) ← new best 🏆
```

### Type 4: 特征工程流水线图

读取 `src/features/` 下的代码。

生成 `figures/feature-pipeline.mmd`:

```mermaid
graph LR
    subgraph 原始特征
        R1["数值(N)"]
        R2["类别(M)"]
        R3["时间(T)"]
        R4["文本(S)"]
    end

    subgraph 特征工程
        R1 --> F1["缺失值填充<br/>中位数/均值"]
        R2 --> F2["Label Encoding<br/>Target Encoding"]
        R3 --> F3["时间分解<br/>年/月/日/星期"]
        R4 --> F4["TF-IDF<br/>Word2Vec"]

        F1 --> G1["统计特征<br/>mean/std/skew"]
        F1 --> G2["交叉特征<br/>A×B, A/B"]
        F2 --> G3["频率编码"]
        F3 --> G4["滑动窗口统计"]
    end

    subgraph 最终特征
        G1 --> OUT["特征矩阵<br/>K维"]
        G2 --> OUT
        G3 --> OUT
        G4 --> OUT
        F4 --> OUT
    end
```

### 输出

所有可视化文件保存到 `figures/` 目录:
- `figures/model-architecture.mmd`
- `figures/experiment-tree.mmd`
- `figures/score-progression.mmd`
- `figures/feature-pipeline.mmd`

同时在 `competition/EXPERIMENT_PATH.md` 中内联引用这些图。

向用户展示生成的可视化内容 (直接在终端中渲染 Mermaid 代码块)。
