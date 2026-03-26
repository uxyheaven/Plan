# AI 分析系统设计规范

## 核心需求回顾

1. 支持多场景数据分析和洞察
2. 场景可自定义分析思路和模型
3. 卡片入口展示数据
4. AI自主规划分析步骤、获取数据、选用分析框架
5. 用户可对数据/结果提问
6. LLM不参与计算，仅参与推理和规划

---

## 一、架构设计

### 1.1 整体架构图

```
┌─────────────────────────────────────────────────────────┐
│                    前端展示层                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │ 卡片 1   │  │ 卡片 2   │  │ 卡片 N   │               │
│  │(场景A)   │  │(场景B)   │  │ (场景X) │               │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘               │
└────────┼─────────────┼─────────────┼────────────────────┘
         │             │             │
         └─────────────┻─────────────┘
                       │
        ┌──────────────▼──────────────┐
        │    AI分析引擎（后端）         │
        ├──────────────────────────────┤
        │ • LLM 调用器                  │
        │ • 分析框架执行器              │
        │ • 数据获取器                  │
        │ • 结果聚合器                  │
        └──┬──────────────────────┬────┘
           │                      │
    ┌──────▼─────┐        ┌───────▼───────┐
    │ 场景配置库  │        │ 分析框架库    │
    │ (Scenario) │        │(Framework)    │
    │ • 分析思路  │        │ • SWOT        │
    │ • 数据源    │        │ • 5W2H        │
    │ • 指标定义  │        │ • 对标分析    │
    │ • Skill关联│        │ • 趋势预测    │
    └────────────┘        └───────────────┘
           │                      │
    ┌──────▼────────────────────▼──────┐
    │  数据服务集成层                    │
    ├─────────────────────────────────┤
    │ 数据源 A  │ 数据源 B │ 数据源 C  │
    │ (系统/API) (系统/API) (系统/API)  │
    └─────────────────────────────────┘
```

### 1.2 核心模块详解

#### A. 分析编排引擎（Analysis Orchestrator）

- **职责**：协调整个分析流程
- **输入**：场景配置 + 用户初始数据
- **流程**：
  1. 解析场景配置，确定分析思路
  2. 调用LLM生成分析计划
  3. 顺序执行计划中的各步骤
  4. 聚合所有结果
- **输出**：分析洞察 + 数据来源追溯

#### B. LLM推理层（LLM Interface）

- **职责**：与LLM交互，不涉及计算
- **调用时机**：
  1. 分析计划生成
  2. 数据获取指令生成
  3. 结果解释和洞察提取
  4. 用户提问回答

#### C. 场景配置（Scenario Config）

- **结构**：
  ```yaml
  scenario:
    name: '电商平台转化率分析'
    description: '分析转化漏斗各环节'
    analysisFramework: '5W2H + 对标分析'
    skills:
      - skill_id: 'conversion_funnel'
        priority: 1
        params:
          stages: [view, cart, checkout, paid]
      - skill_id: 'competitor_benchmark'
        priority: 2
    dataSources:
      - source: 'order_system'
        metrics: [pv, uv, conversion_rate]
      - source: 'user_system'
        metrics: [user_cohort, retention]
    initialDataDisplay:
      - metric: '转化率'
        value: '2.5%'
        trend: '↓ 5% MoM'
      - metric: '订单量'
        value: '12,450'
        trend: '↑ 12% MoM'
  ```

#### D. 分析框架库（Framework Library）

```
Framework配置示例：

框架1: SWOT分析
- 强项(Strengths): 优势标签识别
- 弱项(Weaknesses): 瓶颈识别
- 机会(Opportunities): 增长点识别
- 威胁(Threats): 风险识别

框架2: 5W2H分析
- Why: 原因分析
- What: 现象描述
- When: 时间模式
- Where: 地域/场景分析
- Who: 用户分析
- How: 改进方案
- How Much: 效果预期（不计算，由LLM评估）

框架3: 对标分析
- 自身指标
- 行业平均值
- 竞对指标
- 差距诊断
- 改进空间
```

#### E. Skill 框架

- **定义**：可复用的分析能力单元
- **职责**：获取特定数据或执行特定分析逻辑
- **特点**：
  - 原子性：独立功能单元
  - 可组合性：多个Skill可组合成完整分析
  - 可扩展性：易于添加新Skill

**Skill vs 场景 vs 框架的关系**（见后续详解）

---

## 二、用户使用流程

### 2.1 标准分析流程

```
用户 → 打开卡片 → 查看初始数据 → 触发分析 → 查看洞察 → 提问 → 获得答复
```

### 2.2 详细步骤

#### 步骤1：卡片展示（冷启动）

```
场景：用户进入系统，看到某个卡片
展示内容：
  - 卡片标题：场景名称
  - 关键指标：2-4个核心指标（预加载）
  - CTA按钮："深度分析" / "AI洞察"

示例：
┌─────────────────────────┐
│ 💡 转化率深度分析         │
├─────────────────────────┤
│ 当前转化率: 2.5% ↓5%    │
│ 订单量: 12,450 ↑12%     │
│ 客单价: ¥450 →           │
│ 复购率: 15% ↓2%          │
│                          │
│     [ AI深度分析 ]       │
└─────────────────────────┘
```

#### 步骤2：触发分析

用户点击"AI深度分析"
→ 后端加载场景配置
→ LLM生成分析计划（第1次LLM调用）
→ 开始执行分析步骤

#### 步骤3：分析执行

系统按照LLM生成的计划：

1. 并行获取各数据源数据（通过Skills）
2. 执行分析框架处理
3. LLM对结果进行解释和洞察提取（第2-3次LLM调用）

#### 步骤4：展示结果

展示完整的分析报告：

```
分析题目：转化率下降5% 的原因分析

【分析框架：5W2H + 对标分析】

┌─ Why & What
│  原因诊断：
│  • 设备端流量占比上升(61%)，转化率低
│  • 页面加载速度下降 +200ms
│  • 新客留存率下降 8%
│
├─ When & Where
│  时间模式：
│  • 周二-周三高峰时段转化率最低
│  • 北方地区下降幅度更大 (-8%)
│
├─ Who
│  用户分群：
│  • 新用户群体：转化率 1.2%（↓ 8%）
│  • 老用户群体：转化率 4.5%（↓ 2%）
│
├─ How（改进方案）
│  优化建议：
│  ✓ 优化移动端首屏加载（Quick Win）
│  ✓ 精准化新客转化漏斗（1-2周）
│  ✓ 地域化运营策略（2-4周）
│
└─ How Much（效果预期）
   预期提升：0.3-0.5% 转化率提升

【对标分析】
  ├─ 我们: 2.5% (行业 4.2%)
  ├─ 竞对A: 3.8%
  ├─ 竞对B: 3.2%
  └─ 差距诊断: 缺乏精准营销能力
```

#### 步骤5：用户提问

```
用户输入："为什么新用户转化率这么低？"
↓
后端LLM基于分析结果回答（第4次LLM调用）
↓
展示答复："根据分析，新用户转化率低主要因为..."
```

---

## 三、数据流

### 3.1 完整数据流图

```
┌──────────────┐
│ 用户打开卡片  │
└───────┬──────┘
        │
        ▼
┌────────────────────────┐
│ 预加载初始指标数据      │  # 来自缓存或快速查询
│ (2-4个核心指标)         │
└───────┬────────────────┘
        │
        ▼
┌────────────────────────┐
│ 用户触发"深度分析"      │
└───────┬────────────────┘
        │
        ▼
┌────────────────────────────────────┐
│ LLM生成分析计划 (第1次调用)         │  # 输入：场景 + 初始数据
│ • 需要获取哪些数据                  │  # 输出：分析步骤序列
│ • 采用哪些Skill                     │
│ • 适用哪些分析框架                  │
└───────┬────────────────────────────┘
        │
        ▼
┌────────────────────────────────────┐
│ 并行执行Skill获取数据               │
│ ├─ Skill A → 获取用户分群数据       │
│ ├─ Skill B → 获取时间序列数据       │
│ ├─ Skill C → 获取对标数据           │
│ └─ Skill D → 获取设备分布数据       │
│                                     │
│ 返回格式：                          │
│ {                                   │
│   skill_id: "user_segment",        │
│   data: [{segment: "新客", cvr: 1.2%}],
│   source: "user_system@2024-03-17" │
│ }                                   │
└───────┬────────────────────────────┘
        │
        ▼
┌────────────────────────────────────┐
│ 分析框架处理 (本地计算)             │
│ • 5W2H框架：分类整理数据             │
│ • SWOT框架：分类与评分               │
│ • 对标框架：计算指标差异            │
│                                     │
│ 返回中间结果 (无需再调LLM)         │
└───────┬────────────────────────────┘
        │
        ▼
┌────────────────────────────────────┐
│ LLM洞察提取 (第2次调用)             │  # 输入：框架处理结果
│ • 生成发现点                        │  # 输出：解释性文本+排序
│ • 排序优先级                        │
│ • 生成建议                          │
└───────┬────────────────────────────┘
        │
        ▼
┌────────────────────────────────────┐
│ LLM结果总结 (第3次调用)             │  # 输入：所有洞察
│ • 生成分析摘要                      │  # 输出：结构化报告
│ • 汇总建议                          │
│ • 添加数据来源标注                  │
└───────┬────────────────────────────┘
        │
        ▼
┌────────────────────────────────────┐
│ 展示完整分析报告                    │
│ + 数据来源追踪                      │
└────────────────────────────────────┘
        │
        ▼（用户提问）
┌────────────────────────────────────┐
│ 用户针对结果提问                    │
└───────┬────────────────────────────┘
        │
        ▼
┌────────────────────────────────────┐
│ LLM回答 (第4次调用)                 │  # 输入：提问 + 历史分析
│ • 基于分析上下文回答                │  # 输出：答复文本
│ • 保持逻辑一致性                    │
└───────┬────────────────────────────┘
        │
        ▼
┌────────────────────────────────────┐
│ 展示答复                            │
└────────────────────────────────────┘
```

### 3.2 数据存储周期

```
实时数据：
  • 初始指标（缓存5分钟）
  • 用户会话（内存）

分析过程数据：
  • 分析计划（会话内存）
  • Skill执行结果（会话内存）
  • LLM交互历史（会话内存）

可选持久化：
  • 分析报告（支持导出）
  • 分析历史（可选）
```

---

## 四、LLM 调用统计

### 4.1 调用次数

**总次数：4-5次调用**（每个分析流程）

### 4.2 各次调用详解

| 序号 | 调用名称     | 触发时机         | 输入内容              | 输出内容                   | 成本 | 重要性  |
| ---- | ------------ | ---------------- | --------------------- | -------------------------- | ---- | ------- |
| 1    | 分析计划生成 | 用户触发分析     | 场景配置 + 初始数据   | 分析步骤序列 + 需要的Skill | 中   | 🔴 关键 |
| 2    | 洞察提取     | 分析框架处理完成 | 框架处理结果 (JSON)   | 发现点 + 优先级排序        | 中   | 🔴 关键 |
| 3    | 结果总结     | 所有洞察生成后   | 所有发现点 + 建议     | 结构化报告文本             | 中   | 🟡 重要 |
| 4    | 用户问答     | 用户提问         | 提问 + 分析报告上下文 | 答复文本                   | 低   | 🟡 重要 |
| 5\*  | 进展通知     | 分析中（可选）   | 当前进度              | 进度描述                   | 低   | 🟢 辅助 |

\*可选：仅在长流程中使用

### 4.3 LLM调用示例

#### 调用1: 分析计划生成

**Prompt:**

```
场景: 电商平台转化率分析
初始数据:
  - 转化率: 2.5% (↓5% MoM)
  - 订单量: 12,450 (↑12% MoM)
  - 客单价: ¥450 (→)
  - 复购率: 15% (↓2%)

可用Skills:
  - user_cohort_analysis
  - funnel_analysis
  - time_series_analysis
  - benchmark_analysis
  - device_breakdown
  - geo_breakdown

可用分析框架:
  - 5W2H
  - SWOT
  - 对标分析

请做出分析计划。
```

**输出结构：**

```json
{
  "analysisTitle": "转化率下降原因诊断",
  "frameworks": ["5W2H", "对标分析"],
  "steps": [
    {
      "step": 1,
      "title": "用户分群分析",
      "skill": "user_cohort_analysis",
      "description": "分析新老客转化率差异"
    },
    {
      "step": 2,
      "title": "漏斗分析",
      "skill": "funnel_analysis",
      "description": "识别转化漏斗的关键卡点"
    },
    {
      "step": 3,
      "title": "时间序列分析",
      "skill": "time_series_analysis",
      "description": "找出转化率下降的时间节点"
    }
  ]
}
```

#### 调用2: 洞察提取

**输入：**

```
框架处理结果:
{
  "5W2H": {
    "Why": ["新客留存率下降", "页面加载速度变慢"],
    "What": ["转化率 2.5%", "对标差3.9%"],
    "When": ["周二-周三尤其严重", "高峰时段转化率最低"],
    "Where": ["北方地区下降幅度更大", "移动端下降8%"],
    "Who": ["新用户群体最严重", "老用户群体相对稳定"]
  },
  "benchmark": {
    "自身": 2.5,
    "行业平均": 4.2,
    "竞对A": 3.8,
    "竞对B": 3.2
  }
}

请提取关键洞察并排优先级。
```

**输出：**

```json
{
  "keyInsights": [
    {
      "rank": 1,
      "insight": "新用户转化率急剧下降(1.2%, ↓8%)",
      "evidence": ["用户分群分析", "时间序列分析"],
      "impact": "高"
    },
    {
      "rank": 2,
      "insight": "移动端性能问题严重",
      "evidence": ["设备分布分析", "时间序列分析"],
      "impact": "高"
    }
  ]
}
```

#### 调用3: 结果总结

**输入：**

```
所有洞察 + 建议 + 分析框架结果

请生成一份结构化的分析报告，包含：
1. 分析题目
2. 核心发现（3-5个）
3. 根本原因分析
4. 改进建议（按优先级）
5. 预期效果描述

注意：标注所有数据的来源系统和时间戳。
```

---

## 五、扩展性设计

### 5.1 新场景接入流程

```
第1步：编写场景配置
  └─ scenario.yaml (指定分析思路、框架、Skill组合)

第2步：开发/复用Skill
  └─ 如果需要新Skill，编写Skill逻辑

第3步：测试与上线
  └─ 验证配置 → 灰度发布 → 全量发布

新场景上线周期：1-3天
```

### 5.2 新分析框架接入

```
框架定义 → 框架处理模块开发 → 集成到调度引擎 → 在场景中引用

框架开发指南：
- 定义输入数据结构
- 定义输出结构
- 实现处理算法（仅涉及逻辑/分类，不涉及复杂计算）
- 编写单测
```

### 5.3 新Skill接入

```
Skill开发流程：

1. 定义Skill接口
   {
     skillId: "string",
     dataSource: "string",
     queryParams: {},
     outputSchema: {}
   }

2. 实现Skill执行器
   • 调用对应数据系统API
   • 返回标准格式结果
   • 记录数据来源元数据

3. 在场景中使用
   • scenario.yaml 中引用 skillId
   • LLM在分析计划中调度

4. 性能优化
   • 支持并行执行多个Skill
   • 支持结果缓存（TTL配置化）
```

### 5.4 系统架构的扩展点

| 扩展点       | 方向              | 周期   | 复杂度 |
| ------------ | ----------------- | ------ | ------ |
| 新场景       | 增加scenario.yaml | 1-3天  | 低     |
| 新Skill      | 开发Skill执行器   | 3-7天  | 中     |
| 新分析框架   | 开发框架处理模块  | 5-10天 | 中     |
| 新数据源     | 开发数据源连接器  | 3-7天  | 中     |
| 多LLM支持    | 适配不同LLM API   | 3-5天  | 低     |
| 实时推送     | 异步任务框架      | 2周    | 高     |
| 分析报告存储 | 持久化层          | 1周    | 中     |

---

## 六、Skill、场景、分析框架的关系

### 6.1 三层关系图

```
                    ┌─────────────────┐
                    │   分析框架库     │
                    │  (SWOT/5W2H/  │
                    │   对标分析)      │
                    └────────┬────────┘
                             ▲
                             │
                    ┌────────┴────────┐
                    │                 │
            ┌───────▼─────┐    ┌──────▼──────┐
            │  场景配置    │    │  Skill库    │
            │ (Scenario)   │    │  (可复用)   │
            └─────────────┘    └─────────────┘
                    ▲                 ▲
                    │                 │
        ┌───────────┘                 │
        │                             │
    ┌───▼──────────────────────────┐ │
    │   分析编排引擎               │ │
    │   1. 读场景配置              ├─┘
    │   2. 调度Skill获取数据       │
    │   3. 调用分析框架处理数据    │
    │   4. 调用LLM进行推理        │
    └────────────────────────────┘
```

### 6.2 具体关系说明

#### A. 场景 (Scenario) 的角色

**定义：** 特定业务问题的分析配置集合

**组成:**

```yaml
scenario:
  name: '转化率分析'
  frameworks: [5W2H, 对标分析] # 指定用哪些框架
  skills: # 指定用哪些Skill
    - user_cohort_analysis
    - funnel_analysis
    - benchmark_analysis
  dataSources: [order_system, user_system]
```

**特点：**

- 1个场景 = 1个业务问题单元
- 场景聚合多个Skill + 多个框架
- 场景是可配置的，无需开发

#### B. Skill 的角色

**定义：** 原子级的数据获取或分析能力

**生命周期：**

```
通用Skill库
  ├─ 用户分析类
  │  ├─ user_cohort_analysis (按新老客分群)
  │  ├─ user_retention_analysis (复购分析)
  │  └─ user_lifecycle_analysis (生命周期分析)
  ├─ 数据聚合类
  │  ├─ funnel_analysis (漏斗分析)
  │  ├─ time_series_analysis (趋势分析)
  │  └─ geo_breakdown (地域分布)
  ├─ 对标类
  │  ├─ benchmark_analysis (对标分析)
  │  └─ competitor_tracking (竞对追踪)
  └─ 其他
     ├─ device_breakdown (设备分布)
     └─ segment_analysis (客群分析)
```

**特点：**

- 高复用性：1个Skill可被多个场景使用
- 独立性：Skill可独立开发、测试、发布
- 可扩展性：易于添加新Skill

#### C. 分析框架 的角色

**定义：** 结构化的思维工具，用于组织和解释数据

**框架库：**

| 框架         | 用途            | 处理逻辑  | Skill配合                          |
| ------------ | --------------- | --------- | ---------------------------------- |
| 5W2H         | 多维度分析问题  | 数据分类  | 所有Skill都适配                    |
| SWOT         | 竞争力分析      | 评分+分类 | benchmark_analysis + user_analysis |
| 对标分析     | 与竞品/行业对比 | 数值计算  | benchmark_analysis                 |
| 漏斗诊断     | 流程优化        | 层级分析  | funnel_analysis                    |
| 趋势预测分析 | 时间序列预判    | 趋势识别  | time_series_analysis               |

**特点：**

- 框架是数据的"组织方式"
- 1个Skill的结果可被多个框架处理
- 框架处理不涉及复杂计算，只涉及逻辑和分类

### 6.3 三者的交互示例

**场景：转化率分析**

```
Scenario: 转化率分析
  │
  ├─► Framework 1: 5W2H分析
  │   └─ Why: 原因分析
  │       └─ Skill: user_cohort_analysis (谁的问题最严重)
  │       └─ Skill: time_series_analysis (何时开始下降)
  │   └─ What: 现象描述
  │       └─ Skill: funnel_analysis (漏斗哪个环节卡)
  │   └─ Where: 地域场景分析
  │       └─ Skill: geo_breakdown (哪个地区最严重)
  │   └─ Who: 用户分析
  │       └─ Skill: user_lifecycle_analysis (新老客分化)
  │   └─ How: 改进方案
  │       └─ LLM推理（基于以上Skill数据）
  │
  ├─► Framework 2: 对标分析
  │   ├─ 自身指标
  │   │  └─ Skill: funnel_analysis
  │   ├─ 竞对指标
  │   │  └─ Skill: benchmark_analysis
  │   └─ 差距诊断
  │      └─ LLM推理
  │
  └─► LLM总结
      └─ 基于两个框架的结果
      └─ 生成最终洞察
```

### 6.4 配置示例

```yaml
# scenarios/conversion_analysis.yaml
scenario:
  id: 'conv_analysis_v1'
  name: '电商转化率分析'

  # 指定分析框架
  analysisFrameworks:
    - framework: '5W2H'
      priority: 1
    - framework: 'benchmark'
      priority: 2

  # 指定Skill
  skills:
    - skillId: 'user_cohort_analysis'
      dataSource: 'user_system'
      params:
        groupBy: ['cohort', 'device_type']

    - skillId: 'funnel_analysis'
      dataSource: 'order_system'
      params:
        stages: ['view', 'cart', 'checkout', 'paid']

    - skillId: 'time_series_analysis'
      dataSource: 'order_system'
      params:
        metric: 'conversion_rate'
        granularity: 'daily'
        days: 30

    - skillId: 'geo_breakdown'
      dataSource: 'order_system'
      params:
        level: 'province'

    - skillId: 'benchmark_analysis'
      dataSource: 'benchmark_system'
      params:
        competitors: ['competitor_a', 'competitor_b']
        metrics: ['conversion_rate', 'aov']

  # 初始卡片展示
  cardDisplay:
    metrics:
      - name: '转化率'
        source: 'primary'
      - name: '订单量'
        source: 'primary'
      - name: '客单价'
        source: 'primary'
```

---

## 七、最核心的3个问题解决方案

### 问题1: 如何确保LLM生成的分析计划的正确性和完整性？

#### 背景

LLM作为分析编排的决策者，若计划不合理，会导致：

- 缺少关键数据维度
- 调度错误的Skill
- 分析框架不适配

#### 解决方案

**A. 分层验证机制**

```
① Prompt层面：硬约束清晰定义
   - 明确可用的Skill列表
   - 明确可用的框架列表
   - 给出"正确计划"的示例(Few-shot)
   - 限制输出JSON结构

② 编排层面：Schema验证
   - 解析LLM输出为JSON
   - 验证：
     • 所有SkillId存在？
     • 所有Framework存在？
     • 步骤是否有依赖冲突？
   - 验证失败 → 重试或human-in-the-loop

③ 执行层面：降级方案
   - 如果LLM计划执行失败，启用预设的备选计划
   - 备选计划基于场景配置的默认推荐
```

**B. 示例Prompt**

```
你是分析规划专家。根据以下信息生成分析计划。

【可用Skill】
- user_cohort_analysis: 用户分群分析，维度包括[新老客、设备、地域]
- funnel_analysis: 转化漏斗分析，可指定漏斗阶段
- time_series_analysis: 时间序列分析，可指定时间粒度
- benchmark_analysis: 对标分析，可指定竞对对象

【可用框架】
- 5W2H: 适合诊断问题原因，需要多维度数据支撑
- 对标分析: 适合对比分析，需要竞品和自身数据

【输出格式(JSON)】
必须遵循以下结构：
{
  "analysisTitle": "string",
  "reasoning": "string", // 为什么选择这个计划
  "frameworks": ["5W2H" | "benchmark"],
  "steps": [
    {
      "step": 1,
      "skillId": "必须从上述Skill列表选择",
      "description": "string"
    }
  ]
}

【当前业务背景】
场景：电商转化率下降分析
初始数据：
  - 转化率：2.5% (↓5% MoM)
  - 订单量：12,450 (↑12% MoM)

请生成分析计划。
```

**C. 验证代码示例**

```python
class AnalysisPlanValidator:
    def validate(self, plan: dict, scenario: dict) -> bool:
        # 1. Schema验证
        required_keys = ["analysisTitle", "frameworks", "steps"]
        if not all(k in plan for k in required_keys):
            raise ValidationError("Missing required keys")

        # 2. 检查框架是否有效
        valid_frameworks = scenario["validFrameworks"]
        if not all(f in valid_frameworks for f in plan["frameworks"]):
            raise ValidationError("Invalid framework")

        # 3. 检查Skill是否有效
        valid_skills = scenario["availableSkills"]
        for step in plan["steps"]:
            if step["skillId"] not in valid_skills:
                raise ValidationError(f"Skill {step['skillId']} not available")

        # 4. 检查是否有依赖冲突
        # ...

        return True
```

---

### 问题2: 如何追踪所有数据的来源，确保可信度？

#### 背景

用户需要知道：

- 这个数据来自哪个系统？
- 数据的时间戳是什么？
- 数据是否仍然有效？

#### 解决方案

**A. 数据溯源机制**

```
每条数据都携带元数据：

{
  "value": 2.5,
  "metric": "conversion_rate",
  "metadata": {
    "source": "order_system",           # 数据来源系统
    "sourceTimestamp": "2024-03-17 10:30:00",  # 数据时间
    "fetchTimestamp": "2024-03-17 11:00:00",   # 获取时间
    "skillId": "funnel_analysis",       # 获取数据的Skill
    "queryParams": {
      "startDate": "2024-03-16",
      "endDate": "2024-03-17"
    },
    "ttl": 300,                         # 数据有效期(秒)
    "confidence": 0.99                  # 可信度
  }
}
```

**B. 在报告中展示来源**

```
【分析结果】

转化率: 2.5% (↓5% MoM)
  数据来源: order_system
  数据时间: 2024-03-17 10:30
  获取时间: 2024-03-17 11:00
  [数据详情 ↗]

新客转化率: 1.2% (↓8% MoM)
  数据来源: user_system + order_system
  数据时间: 2024-03-17 10:30
  获取时间: 2024-03-17 11:00
  [数据详情 ↗]
```

**C. 前端展示设计**

```
┌─────────────────────────────┐
│ 分析结果                     │
├─────────────────────────────┤
│ 转化率: 2.5%                 │
│ ℹ️ 来源: order_system        │
│    更新时间: 11:00 (刚刚更新)│
│    [显示详细来源 ↗]          │
│                              │
├─ 数据来源详情 (展开)        │
│  • 数据系统: order_system    │
│  • 时间范围: 2024-03-16~17   │
│  • 数据行数: 12,450          │
│  • 有效期: 5分钟              │
│  • 最后更新: 2024-03-17 10:30 │
└─────────────────────────────┘
```

**D. 数据链路记录**

```python
class DataLineage:
    def track(self, metric_name: str) -> List[dict]:
        """追踪某个指标的完整数据链路"""
        return [
            {
                "stage": 1,
                "action": "Skill执行",
                "skillId": "funnel_analysis",
                "input": {"startDate": "2024-03-16"},
                "output": {"data": [...]},
                "timestamp": "2024-03-17 11:00:00"
            },
            {
                "stage": 2,
                "action": "框架处理",
                "framework": "5W2H",
                "input": {"funnel_data": [...]},
                "output": {"classified": [...]},
                "timestamp": "2024-03-17 11:00:01"
            },
            {
                "stage": 3,
                "action": "LLM推理",
                "prompt": "...",
                "output": "新客转化率低的原因是...",
                "timestamp": "2024-03-17 11:00:05"
            }
        ]
```

---

### 问题3: 如何优雅地处理分析失败和降级？

#### 背景

可能失败的场景：

- 某个数据源不可用
- Skill执行超时
- LLM调用失败
- 用户提问超出分析范围

#### 解决方案

**A. 分层降级策略**

```
容错级别设计：

┌─ 级别1：快速降级（最常见）
│  场景：某个Skill超时
│  处理：
│    • 使用缓存数据（如果有TTL未过期）
│    • 使用预设默认分析计划
│    • 标记该维度数据为"不可用"
│    • 继续执行其他Skill
│
├─ 级别2：部分降级
│  场景：部分框架无法应用
│  处理：
│    • 使用可用框架进行分析
│    • 向用户明确说明哪些框架因数据缺失而跳过
│    • 基于可用数据生成缩小范围的洞察
│
├─ 级别3：中断降级
│  场景：核心Skill失败
│  处理：
│    • 中断分析过程
│    • 展示错误信息
│    • 提示用户重试或使用快速分析
│
└─ 级别4：图形降级
   场景：LLM调用失败
   处理：
     • 仅展示原始数据+基础图表
     • 不生成LLM洞察
     • 提示"文本解读暂不可用"
```

**B. 实现示例**

```python
class ResilientAnalysisEngine:
    def execute_with_fallback(self, scenario: dict) -> Analysis:
        try:
            # 1. 生成分析计划
            plan = self.llm.generate_plan(scenario)

            # 2. 执行Skill(带超时)
            skill_results = self.execute_skills_parallel(plan, timeout=10)

            # 3. 处理部分失败
            available_skills = [s for s in skill_results if s["status"] == "success"]
            failed_skills = [s for s in skill_results if s["status"] == "failed"]

            if len(available_skills) < len(skill_results) * 0.7:  # 少于70%成功
                # 降级到Level 3: 中断
                return self.generate_fallback_error("数据获取失败，请稍后重试")

            # 4. 框架处理
            intermediate_results = self.apply_frameworks(available_skills)

            # 5. LLM推理(带重试)
            try:
                insights = self.llm.extract_insights(
                    intermediate_results,
                    max_retries=2
                )
            except LLMException:
                # 降级到Level 4: 图形降级
                insights = self.generate_basic_insights(intermediate_skills)

            # 6. 标注失败信息
            result["warnings"] = [
                f"无法获取{skill['skillId']}数据",
                f"LLM调用失败，展示基础分析"
            ]

            return result

        except Exception as e:
            # 最终降级
            return self.generate_minimal_response(scenario)

    def generate_minimal_response(self, scenario: dict) -> dict:
        """最小可用响应"""
        return {
            "status": "partial",
            "message": "分析异常，展示初始数据",
            "data": scenario["initialData"],
            "error": str(e)
        }
```

**C. 用户体验设计**

```
正常流程：
┌─────────────────────────────┐
│ 分析中... 30%               │
│ [█████░░░░░░░░░░░░]        │
│ • 正在加载用户分群数据...   │
│ • 等待其他分析...           │
└─────────────────────────────┘

部分失败（降级1）：
┌─────────────────────────────┐
│ ✓ 分析完成 (部分数据可用)   │
├─────────────────────────────┤
│ ⚠️ 注意：                     │
│ • 地域分布数据暂不可用      │
│ • 基于现有数据生成分析      │
│                              │
│ [分析结果] [重新分析]        │
└─────────────────────────────┘

中断（降级3）：
┌─────────────────────────────┐
│ ❌ 分析异常                  │
├─────────────────────────────┤
│ 原因: 订单系统暂时不可用     │
│ 建议: 2分钟后重试            │
│                              │
│ [快速查看初始数据]           │
│ [重新分析]                   │
└─────────────────────────────┘
```

**D. 查询结果缓存策略**

```
缓存工作流：

┌─ 收到Skill请求
  │
  ├─ 生成缓存Key
  │  key = hash(skillId + params)
  │
  ├─ 检查缓存
  │  If hit && !expired:
  │    └─ 返回缓存结果 (标记: "cached")
  │
  ├─ 如果缓存miss或过期
  │  └─ 执行实时查询(timeout=10s)
  │
  └─ 存储结果
     ├─ 缓存有效期基于数据类型
     │  业务数据: 5-30分钟
     │  基础数据: 1-2小时
     └─ 带TTL标记
```

---

## 八、系统实现建议

### 8.1 技术栈

```
后端：
  • Node.js + Express / Python + FastAPI
  • 分析编排: 自研或 Apache Airflow（轻量级）
  • 缓存: Redis
  • 消息队列: RabbitMQ (异步任务)

前端：
  • React + TypeScript
  • 图表库: ECharts / Recharts
  • 实时更新: WebSocket

数据：
  • 指标体系: 自建或ATable / 数据仓库
  • 数据源集成: 数据中台 或 自建API网关

LLM：
  • API: OpenAI / Anthropic / 本地模型
  • 成本控制: Prompt缓存 + 结果复用
```

### 8.2 性能目标

| 指标     | 目标        | 措施                  |
| -------- | ----------- | --------------------- |
| 首屏加载 | < 2s        | 预加载4个初始指标     |
| 分析完成 | 5-30s       | 并行执行Skill + 缓存  |
| LLM延迟  | 3-10s/次    | Prompt优化 + Few-shot |
| 总吞吐   | 100 req/min | 异步队列 + 限流       |

### 8.3 下一步工作

- [ ] 构建Skill库基础框架
- [ ] 实现场景配置系统
- [ ] 完成分析框架处理模块
- [ ] 集成LLM（先以Claude/GPT为目标）
- [ ] 搭建数据溯源系统
- [ ] 前端原型开发
- [ ] 灰度测试

---

## 总结

这个AI分析系统通过以下特色实现了需求：

✅ **多场景支持** - 通过可配置的Scenario框架
✅ **自主规划** - 通过LLM分析计划生成
✅ **高度可扩展** - Skill库 + 框架库 + 数据源解耦
✅ **提高准确性** - LLM仅负责推理规划，计算由系统完成
✅ **完整的数据溯源** - 每条数据都携带元数据和处理链路
✅ **优雅降级** - 分层容错保证用户体验

核心价值在于：**用更少的LLM调用，更清晰的数据流，完成更复杂的分析任务**。
