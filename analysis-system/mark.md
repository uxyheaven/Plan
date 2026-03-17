# 核心设计要点标记

## 🎯 最核心的设计原则

### 1. **分离关注点**

- ✅ LLM 仅实现：推理、规划、解释
- ✅ 系统实现：计算、数据获取、框架处理
- ❌ 不让LLM参与：数据计算、数值比较、数据排序

### 2. **可组合性**

- Skill是原子单元，可独立复用
- 框架是处理方式，框架处理是本地计算，无需LLM
- 场景是多个Skill + 多个框架的组合

### 3. **透明性**

- 每条数据都有来源标注
- 完整的处理链路可追踪
- 用户可知道"为什么这么说"

---

## 📊 必读的核心流程

### 用户角度:

```
点击卡片 → 查看初始数据 → 点击"深度分析"
→ AI生成分析计划(LLM Call #1)
→ 并行获取数据(通过Skills)
→ 本地处理数据(通过框架)
→ LLM提取洞察(LLM Call #2-3)
→ 展示报告
→ 提问 → LLM回答(LLM Call #4)
```

### 系统角度:

```
场景配置 + 初始数据
    ↓
LLM生成分析计划(JSON: 要执行的步骤序列)
    ↓
Skill执行器(并行运行)→ 获取数据 + 记录来源
    ↓
框架处理器(顺序处理) → 数据分类[不涉及LLM]
    ↓
LLM洞察提取 → 转为可读文本
    ↓
展示报告
```

---

## 🔑 关键术语

### Scenario (场景)

- 是什么: 特定业务问题的分析配置包
- 包含: 分析思路 + 指定的Skills + 指定的框架 + 数据源
- 特点: 无代码配置，可复用，易扩展
- 示例: "电商转化率分析", "用户留存分析"

### Skill (技能)

- 是什么: 原子级数据获取/分析能力
- 职责: 调用某个数据系统的API，返回标准格式结果
- 特点: 独立开发、可复用、可并行执行
- 示例: user_cohort_analysis, funnel_analysis, benchmark_analysis
- 返回格式必须包含: data + source + timestamp + confidence

### Framework (分析框架)

- 是什么: 结构化的思维工具，帮助组织数据
- 不涉及: LLM调用（本地处理）
- 功能: 数据分类、评分、排序、对标calc
- 示例: 5W2H(分类), SWOT(评分), 对标(数值compute)

### LLM Role (LLM角色)

- Call #1: 分析计划生成 → 规划能力
- Call #2: 洞察提取 → 推理能力
- Call #3: 报告生成 → 表达能力
- Call #4: 用户问答 → 综合能力
- 不参与: 数据计算、结果排序

---

## ⚙️ 关键设计决策

### 决策1: 为什么数据来自多个系统还能确保准确性？

→ 通过元数据标注(来源系统、时间戳、可信度) + 数据链路记录
→ 每条数据都是"溯源可查"的，而不是"记录保存"的

### 决策2: 为什么LLM不参与计算？

→ LLM在数值计算上容易出错(幻觉问题)
→ 将LLM限制在"推理、规划、解释"的强项
→ 系统确保计算准确，LLM提供洞察

### 决策3: 为什么用4次LLM调用而不是1次？

→ 1次调用风险:
• Prompt太长，容易出错
• 无法中间验证
• 无法并行优化
→ 4次专用调用好处:
• 每次职责清晰，准确度高
• 支持中间检查和修正
• 结果可缓存和复用

---

## 🛑 常见陷阱及解决方案

### ⚠️ 陷阱1: 让LLM直接处理原始数据

❌ 错误: 把成千上万条日志直接feed给LLM
✅ 正确: 先经过框架处理(聚合、分类)，再给LLM

### ⚠️ 陷阱2: 不标注数据来源

❌ 错误: 报告中只有数字，不知道来自哪个系统
✅ 正确: 每条数据都标上 source + timestamp + confidence

### ⚠️ 陷阱3: 让LLM完全自主规划

❌ 错误: LLM选择不存在的Skill，或生成无效计划
✅ 正确: 使用JSON schema约束 + 白名单验证 + 降级机制

### ⚠️ 陷阱4: 忽视分析框架

❌ 错误: 只依赖LLM的自由表达
✅ 正确: 基于框架的结构化处理，LLM仅负责解释

---

## 📈 扩展性检查表

### 添加新场景 (1-3天)

- [ ] 编写 scenario.yaml (指定框架 + Skills)
- [ ] 验证场景配置
- [ ] 灰度测试

### 添加新Skill (3-7天)

- [ ] 定义Skill接口
- [ ] 实现数据获取逻辑
- [ ] 编写单元测试
- [ ] 在scenario中引用

### 添加新框架 (5-10天)

- [ ] 定义框架输入/输出结构
- [ ] 实现处理算法(纯逻辑/计算)
- [ ] 集成到调度引擎
- [ ] 在Prompt中更新框架列表

### 支持新数据源 (3-7天)

- [ ] 开发数据源连接器
- [ ] 定义Skill接口
- [ ] 集成到系统中

---

## 💾 数据流四个关键点

### 点1: 初始数据 (冷启动)

- 来自: 快速查询或缓存
- 用途: 展示在卡片上
- TTL: 5分钟

### 点2: 分析过程数据 (会话期间)

- Skills执行结果 (并行)
- 框架处理结果 (本地计算)
- LLM调用结果 (逐次积累)
- 存储: 会话内存

### 点3: 数据溯源信息 (全程记录)

- 每条数据 + 来源元数据
- 完整的处理链路
- 可用于审计和重现

### 点4: 可选持久化 (用户决定)

- 分析报告的导出
- 分析历史(可选记录)
- 不强制保存，只提供选项

---

## 🎨 前端展示关键

### 卡片设计

```
┌─ 标题(场景名)
├─ 2-4个核心指标
│  ├─ 数值
│  ├─ 环比/同比
│  └─ 小图表或趋势
├─ 数据来源说明 (点击展开)
└─ [AI深度分析 Button]
```

### 分析报告设计

```
┌─ 分析题目
├─ 执行摘要
├─ 核心发现(使用icons)
├─ 根本原因分析
├─ 改进建议(优先级排序)
├─ 预期效果
├─ 数据来源详情(可展开)
└─ [AI问答窗口]
```

### 数据源标注设计

```
指标: 转化率 2.5%
来源: 📍 order_system
时间: 🕐 2024-03-17 10:30
有效期: ⏰ 5分钟内
[显示详细来源]
```

---

## 🚀 性能目标

| 指标     | 目标       | 实现方式       |
| -------- | ---------- | -------------- |
| 卡片首屏 | <2s        | 预加载初始指标 |
| 分析启动 | <5s        | LLM规划        |
| 数据获取 | <10s       | 并行Skill执行  |
| 报告就绪 | <30s       | 异步流水线     |
| LLM延迟  | 3-10s/Call | Prompt优化     |

---

## 📝 实现时间表 (建议)

### 第1周: 基础架构

- [ ] 场景配置系统
- [ ] Skill框架设计
- [ ] 分析框架处理模块

### 第2周: 核心流程

- [ ] LLM集成(4个调用)
- [ ] 数据溯源系统
- [ ] 前端原型(卡片+报告)

### 第3周: 验证和优化

- [ ] 集成测试
- [ ] 性能优化
- [ ] 错误处理(降级方案)

### 第4周+: 迭代

- [ ] 新场景接入
- [ ] 新Skill开发
- [ ] 产品迭代

---

## ❓ FAQ

**Q: 如果LLM的分析计划有问题怎么办?**
A: 有多层验证:

1. Schema验证(Skill存在不存在？)
2. 逻辑验证(依赖关系对不对?)
3. 执行验证(能否成功执行?)
4. 降级方案(读取预设的备选计划)

**Q: 数据源不可用会怎样?**
A: 分层降级:

1. 尝试使用缓存数据
2. 跳过该Skill，继续其他分析
3. 基于可用数据生成缩小范围的结果
4. 最后降级到仅展示初始数据

**Q: Skill和框架哪个更重要?**
A: 互相依存:

- Skill提供"数据"
- 框架提供"结构"
- 一个没有数据空结构，一个没有结构数据混乱
- 最重要的是它们的组合方式(由LLM规划)

**Q: 为什么要记录数据来源而不直接记录数据?**
A: 三个原因:

1. 节省存储空间
2. 确保数据总是最新(下次查询重新获取)
3. 避免沦为孤立的历史数据

**Q: 这套系统能处理多大量级的数据?**
A: 系统设计支持:

- 初始数据: <100条指标
- Skill返回结果: <10K行
- 分析步骤: <8步
- 并行Skill: <5个
- 通常完整分析 <30秒
  [分析规划] ← LLM Call #1
  ├─ 输入：初始数据、场景配置、可用模型列表
  ├─ 输出：分析计划、需求数据指标列表、选定模型
  └─ 记录：LLM输入输出、timestamp
  ↓
  [数据获取] ← 执行获取的数据指标
  ├─ 调用各数据源API
  ├─ 记录每条数据的来源系统、接口、获取时间
  ├─ 数据清洗和验证
  └─ 返回：数据字典 {指标名: {值, 来源, 获取时间}}
  ↓
  [模型分析] ← 调用选定的分析模型
  ├─ 输入：数据字典 + 模型参数
  ├─ 纯计算，不涉及LLM
  └─ 输出：分析结果 {结果类型, 数值/矩阵, 可视化建议}
  ↓
  [洞察生成] ← LLM Call #2
  ├─ 输入：分析结果 + 数据来源 + 场景context
  ├─ 输出：可读的洞察，包含数据来源说明
  └─ 生成初始分析报告
  ↓
  [结果展示] → 显示卡片数据 + 分析洞察 + 提问窗口

```

### 2.2 问答交互流程

```

用户提问 → [问答路由]
↓
[理解需求判断]
├─ 需要现有数据回答？ → [检索缓存数据] → [LLM Call #3: 生成回答]
│
├─ 需要新数据回答？ → [执行新的数据获取] → [可能重新分析] → [LLM Call #3: 生成回答]
│
└─ 需要重新分析？ → [分析规划LLM] → [数据获取] → [模型分析] → [LLM Call #2/3]
↓
[生成回答] ← 考虑会话历史 + 已有分析结果
↓
显示回答 + 数据来源 + 后续操作建议

````

## 3. LLM 调用详情

### 3.1 三次核心 LLM 调用

#### LLM Call #1: 分析规划 (Analysis Planning)

**触发条件**：初始化分析或用户重新提问需要新分析策略

**输入**：

```json
{
  "scenario": "用户销售分析",
  "initial_data": {
    "user_id": "12345",
    "regions": ["华东", "华北"],
    "time_period": "2024年Q1"
  },
  "available_models": [
    "trend_analysis",
    "correlation_analysis",
    "segmentation",
    "anomaly_detection"
  ],
  "scenario_config": {
    "analysis_thoughts": "先看销售趋势，再分析地区差异，最后找行为特征",
    "key_metrics": ["销售额", "订单数", "客户数", "转化率"]
  },
  "data_sources": {
    "sales_api": "xxx",
    "user_api": "xxx"
  }
}
````

**输出**：

```json
{
  "analysis_plan": [
    "获取销售趋势数据",
    "对比地区销售表现",
    "分析客户行为特征",
    "识别异常数据"
  ],
  "required_metrics": [
    "daily_sales",
    "regional_sales",
    "customer_count",
    "conversion_rate",
    "repeat_rate",
    "avg_order_value"
  ],
  "selected_models": [
    {
      "type": "trend_analysis",
      "focus": "sales",
      "config": { "window": 7 }
    },
    {
      "type": "correlation_analysis",
      "variables": ["area", "sales", "customer_count"]
    }
  ],
  "reasoning": "选择这些模型的原因..."
}
```

#### LLM Call #2: 洞察生成 (Insight Generation)

**触发条件**：分析模型执行完成

**输入**：

```json
{
  "analysis_results": {
    "trend": "销售呈上升趋势，周增长5%",
    "correlation": { "area_sales": 0.92, "customer_sales": 0.88 },
    "anomalies": [{ "date": "2024-01-15", "value": 50000, "deviation": "3倍" }]
  },
  "data_sources": {
    "sales_data": {
      "source_system": "销售系统v2.1",
      "api_endpoint": "sales_api/metrics",
      "fetch_time": "2024-03-17T10:30:00Z",
      "record_count": 90
    },
    "user_data": {
      "source_system": "用户中心系统",
      "api_endpoint": "user_api/stats",
      "fetch_time": "2024-03-17T10:31:00Z"
    }
  },
  "scenario_context": "用户销售分析 - 地区对比"
}
```

**输出**：

```markdown
## 数据洞察

### 关键发现

1. **销售增长趋势**：近30天销售额呈稳健上升趋势，周均增长5%，表明市场需求稳定
2. **地区差异明显**：华东地区销售贡献65%，华北仅35%，建议加强华北地区推广
3. **异常波动**：1月15日销售额为5万，较平均值高3倍，需要进一步调查原因

### 数据可信度说明

- 数据来源：销售系统v2.1、用户中心系统
- 数据获取时间：2024-03-17 10:30-10:31 UTC
- 样本量：90条记录
- 数据有效性：已验证无缺失值

### 后续建议

- 深入分析1月15日的销售峰值原因
- 制定针对华北地区的增长策略
- 关注客户留存率变化
```

#### LLM Call #3: 问答回复 (Q&A Response)

**触发条件**：用户提问

**输入示例**：

```json
{
  "user_question": "为什么华北地区销售不好？",
  "conversation_history": [
    {"role": "assistant", "content": "初始分析洞察..."},
    {"role": "user", "content": "还有什么其他发现吗？"}
  ],
  "available_data": {
    "current_analysis": {结果...},
    "cached_insights": {...}
  },
  "scenario": "用户销售分析"
}
```

**输出示例**：

```
根据分析数据，华北地区销售不佳的可能原因：

1. **市场渗透率低** - 华北客户数为华东的30%
2. **转化率差异** - 华北转化率为8.5%，华东为12.3%
3. **客户粘性弱** - 华北重复购买率为22%，华东为35%

建议先深入调研华北市场的客户画像，了解转化率低的具体原因。需要吗？
```

### 3.2 LLM 调用流程图

```
分析初始化
    ↓
[LLM Call #1: 分析规划]
    ├─ 规划: 需要获取哪些数据 + 用哪些模型
    └─ 输出: 分析计划
    ↓
[数据获取] (纯API调用，不涉及LLM)
    ├─ 调用多个数据源
    └─ 返回: 数据集
    ↓
[模型分析] (纯计算，不涉及LLM)
    ├─ 执行选定的分析模型
    └─ 返回: 分析结果
    ↓
[LLM Call #2: 洞察生成]
    ├─ 输入: 分析结果 + 数据来源
    └─ 输出: 可读的洞察报告
    ↓
用户提问 ┐
          ├→ [判断是否需要新数据/重新分析]
用户操作 ┘    ├─ 需要 → 重复分析流程
               └─ 不需要 → [LLM Call #3: 问答]

[LLM Call #3: 问答]
    ├─ 输入: 用户提问 + 历史对话 + 已有分析
    └─ 输出: 回答
```

## 4. 扩展性

### 4.1 新场景接入

**1. 场景配置模板**

```yaml
# scenarios/user_analysis.yaml
name: '用户销售分析'
description: '分析用户销售表现和地区差异'

analysis_thoughts: |
  1. 第一步：获取销售趋势，理解整体表现
  2. 第二步：按地区分析，找出差异
  3. 第三步：分析客户特征，挖掘增长机会

available_models:
  - trend_analysis
  - correlation_analysis
  - segmentation
  - anomaly_detection

default_model: trend_analysis

prompt_template: |
  你是一个数据分析专家。
  场景：{{scenario_name}}
  初始数据：{{initial_data}}
  可用模型：{{model_list}}
  请规划分析步骤...

data_sources:
  - name: sales_api
    endpoint: 'https://api.xxx/sales'
    required_fields: [date, region, sales]

  - name: user_api
    endpoint: 'https://api.xxx/users'
    required_fields: [user_id, created_at]
```

**2. 接入步骤**

- 编写场景配置 YAML
- 定义数据来源和字段映射
- 选择适用的分析模型
- 在 UI 上添加卡片入口
- 测试端到端流程

**3. 验证清单**

- [ ] 数据源可正确调用
- [ ] LLM 规划合理
- [ ] 模型执行成功
- [ ] 洞察生成有效

### 4.2 新分析模型接入

**1. 模型接口标准**

```python
# 所有模型需要实现此接口
class AnalysisModel(ABC):
    @abstractmethod
    def prepare(self, config: Dict) -> None:
        """初始化模型参数"""
        pass

    @abstractmethod
    def analyze(self, data: DataFrame) -> AnalysisResult:
        """
        执行分析
        :param data: 输入数据
        :return: 分析结果（不涉及LLM）
        """
        pass

    @abstractmethod
    def get_metadata(self) -> ModelMetadata:
        """返回模型元数据"""
        pass

class AnalysisResult:
    result_type: str  # "trend" / "correlation" / "segment" etc.
    values: Dict[str, Any]  # 具体结果数值
    visualization_suggestion: str  # 建议的图表类型
    data_source_trace: Dict  # 数据来源追踪
```

**2. 自定义模型示例**

```python
# models/custom_regression_model.py
class RegressionModel(AnalysisModel):
    def analyze(self, data: DataFrame) -> AnalysisResult:
        # 纯数学计算，无LLM参与
        from sklearn.linear_model import LinearRegression
        X = data[['feature1', 'feature2']]
        y = data['target']
        model = LinearRegression()
        model.fit(X, y)

        return AnalysisResult(
            result_type="regression",
            values={
                "coefficients": model.coef_.tolist(),
                "r_squared": model.score(X, y)
            },
            visualization_suggestion="scatter_with_line"
        )
```

**3. 模型注册**

```python
# 在模型库中注册
MODEL_REGISTRY = {
    "trend_analysis": TrendAnalysisModel,
    "correlation_analysis": CorrelationAnalysisModel,
    "custom_regression": RegressionModel,
}
```

### 4.3 新数据源接入

**1. 数据源配置**

```yaml
# data_sources/config/new_source.yaml
name: '新数据源系统'
type: 'api' # api / database / cache

connection:
  endpoint: 'https://api.newservice.com'
  auth_type: 'bearer_token'
  timeout: 30

endpoints:
  /metrics:
    fields:
      - name: metric_id
        type: string
        required: true
      - name: value
        type: float
      - name: dimension
        type: string

  /trace:
    fields:
      - name: trace_id
        type: string

# 数据来源标识
source_identifier: 'new_system_v1.0'
```

**2. 适配器实现**

```python
# data_source_adapters/new_source_adapter.py
class NewSourceAdapter(DataSourceAdapter):
    def fetch_metric(self, metric_name: str, filters: Dict) -> DataFrame:
        """获取指标"""
        response = self.client.get(
            f"/metrics/{metric_name}",
            params=filters
        )
        df = DataFrame(response.json())
        return df

    def get_source_metadata(self) -> Dict:
        """返回来源元数据"""
        return {
            "system": "新数据源系统",
            "version": "1.0",
            "fetch_time": datetime.now(),
            "api_endpoint": self.endpoint
        }
```

### 4.4 系统扩展矩阵

```
┌─────────────┬──────────────┬──────────────┬──────────────┐
│             │ 场景         │ 分析模型      │ 数据源       │
├─────────────┼──────────────┼──────────────┼──────────────┤
│ 扩展方式    │ 配置文件YAML │ Python类+注册 │ YAML配置+适配│
│ 无需改核心代│ ✓            │ ~            │ ✓            │
│ 复杂度      │ 低           │ 中           │ 低-中        │
│ 测试难度    │ 低           │ 中           │ 中           │
│ 隔离性      │ 好           │ 好           │ 好           │
└─────────────┴──────────────┴──────────────┴──────────────┘
```

## 5. Skill、场景、分析模型的关系

### 5.1 概念关系图

```
┌─────────────────────────────────────────────────────────┐
│                      Skill (技能)                        │
│  定义：分析某类问题的能力和知识，包含通用的分析思路     │
│  例：用户行为分析技能、销售性能分析技能                  │
├─────────────────────────────────────────────────────────┤
│                         ↓                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │  场景 (Scenario / Scene)                         │  │
│  │  定义：某个具体的分析场景，继承自Skill           │  │
│  │  关系：N个场景可继承自1个Skill                   │  │
│  │  例：用户销售分析场景、用户留存分析场景           │  │
│  └──────────────────────────────────────────────────┘  │
│                         ↓                               │
│  ┌──────────────────────────────────────────────────┐  │
│  │  分析模型组合 (Model Set)                        │  │
│  │  定义：该场景可用的分析模型列表                   │  │
│  │  关系：1个场景配置多个模型                       │  │
│  │  例：趋势分析、相关性分析、异常检测                │  │
│  └──────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
```

### 5.2 详细关系说明

#### Skill (技能层)

```yaml
# skills/user_behavior_analysis.yaml
name: '用户行为分析'
description: '分析用户的行为模式、转化路径、留存情况'

# 通用分析思路
universal_analysis_pattern: |
  步骤1：理解当前数据的基本特征
  步骤2：识别关键行为指标
  步骤3：发现潜在的行为规律
  步骤4：定位问题和机会

# 推荐使用的模型类型
suggested_models:
  - trend_analysis # 适用于时间序列分析
  - funnel_analysis # 适用于转化路径
  - cohort_analysis # 适用于用户分群
  - churn_prediction # 适用于留存预测

# 关键数据指标
key_indicators:
  - user_count
  - active_user_ratio
  - session_count
  - event_count
  - conversion_rate
  - churn_rate

# LLM提示词模板
llm_prompt_template: |
  您是一个{skill_name}专家。
  现在需要分析用户行为数据...
```

#### Scene (场景层)

```yaml
# scenarios/user_scene/sales_analysis.yaml
name: '用户销售分析'
inherits_from: 'user_behavior_analysis' # 继承Skill

description: '分析用户的销售行为和购买决策'

# 场景特定的分析思路（覆盖Skill的通用思路）
analysis_thoughts: |
  1. 了解销售总体趋势和波形特征
  2. 按地区、品类、客户段进行分析
  3. 识别销售异常和峰值原因
  4. 挖掘增长机会

# 场景特定的必用模型
required_models:
  - trend_analysis # 必须使用
  - correlation_analysis # 必须使用

# 场景特定的可选模型
optional_models:
  - segmentation
  - anomaly_detection

# 场景特定的数据源配置
data_sources:
  sales_system:
    endpoint: 'sales_api/metrics'
    fields: [sales, orders, revenue]

  user_system:
    endpoint: 'user_api/profiles'
    fields: [user_id, region, segment]

# 场景特定的LLM提示词（注入场景上下文）
scene_context: |
  这是用户销售分析场景。
  重点关注销售额、订单数、地区差异。
  需要识别销售异常和增长机会。
```

另一个场景：

```yaml
# scenarios/user_scene/retention_analysis.yaml
name: '用户留存分析'
inherits_from: 'user_behavior_analysis' # 继承同一个Skill

description: '分析用户的留存率和流失原因'

analysis_thoughts: |
  1. 计算各时间段的留存率
  2. 分析流失用户的共同特征
  3. 识别高风险流失群体
  4. 制定留存策略

required_models:
  - cohort_analysis
  - churn_prediction

optional_models:
  - correlation_analysis
  - causality_analysis

data_sources:
  user_system:
    endpoint: 'user_api/activity'
    fields: [user_id, created_at, last_active_date]
```

#### AnalysisModel (分析模型层)

```python
# models/trend_analysis.py
class TrendAnalysisModel(AnalysisModel):
    """
    趋势分析模型
    可被多个Skill和Scene使用
    """

    def analyze(self, data: DataFrame, config: Dict) -> AnalysisResult:
        # 纯计算逻辑，与Skill/Scene无关
        from statsmodels.tsa.seasonal import seasonal_decompose
        result = seasonal_decompose(data['value'], period=7)
        return AnalysisResult(...)

# models/cohort_analysis.py
class CohortAnalysisModel(AnalysisModel):
    """
    队列分析模型
    主要用于留存分析Scene
    """
    pass

# 模型注册（供LLM call#1调用）
SKILL_MODEL_MAPPING = {
    "user_behavior_analysis": [
        "trend_analysis",
        "funnel_analysis",
        "cohort_analysis",
        "churn_prediction"
    ],
    "sales_analysis": [
        "trend_analysis",
        "correlation_analysis"
    ]
}
```

### 5.3 关系应用流程

```
用户点击卡片（场景：用户销售分析）
    ↓
[场景识别] → 确定为 "user_sales_analysis" 场景
    ↓
[加载配置]
    ├─ 加载场景配置
    ├─ 继承所属Skill (user_behavior_analysis)
    ├─ 获取可用模型集合 [trend_analysis, correlation_analysis, ...]
    └─ 获取分析思路和LLM提示词
    ↓
[LLM Call #1: 分析规划]
    ├─ 上下文：场景配置 + Skill知识 + 可用模型
    ├─ 输出：选择使用 {trend_analysis, correlation_analysis}
    └─ 理由：这个Skill的最佳实践推荐
    ↓
[执行分析]
    ├─ 调用 TrendAnalysisModel + CorrelationAnalysisModel
    └─ 纯计算，这些模型既可用于本场景，也可用于其他Skill
    ↓
[LLM Call #2: 洞察生成]
    ├─ 上下文：分析结果 + 场景特定的分析思路
    └─ 输出：针对销售分析场景的可读洞察
```

### 5.4 扩展示意

```
Skill 维度扩展：
├─ user_behavior_analysis (已有)
├─ sales_performance_analysis (新增Skill)
│  ├─ revenue_analysis (新Scene)
│  ├─ product_analysis (新Scene)
│  └─ customer_analysis (新Scene)
└─ market_trend_analysis (新Skill)

Model 维度扩展：
├─ trend_analysis (已有)
├─ correlation_analysis (已有)
├─ time_series_forecast (新Model)
├─ deep_learning_model (新Model)
└─ causal_inference (新Model)

DataSource 维度扩展：
├─ sales_api (已有)
├─ user_api (已有)
├─ product_api (新)
├─ marketing_api (新)
└─ external_market_data (新)
```

## 6. 核心问题的解决思路

### 问题1: 如何确保拥有分析所需的所有数据？(Autonomy)

#### 问题分析

- LLM虽然规划分析，但如何知道需要哪些数据？
- 数据可能分散在多个系统，如何完整获取？
- 如何避免遗漏关键数据或重复获取？

#### 解决思路

**分层递进策略**：

```
第1层：显式声明
├─ 场景配置中预先定义key_metrics和data_sources
├─ 这个关键指标集是场景分析的必需最小集
└─ 优点：快速、可靠、可预测

第2层：LLM智能规划
├─ 基于初始数据和显式指标，LLM规划补充数据
├─ LLM输入：场景的key_metrics + 已有数据 + 可用数据源列表
├─ LLM判断：还需要哪些数据来完整分析
└─ 示例：销售分析有了销售额，LLM会建议补充客户数、转化率等

第3层：动态验证
├─ 执行LLM规划的数据获取
├─ 尝试从各数据源获取，记录成功/失败
├─ 如果关键数据获取失败，立即告知用户
├─ 基于现有数据进行分析（降级策略）

第4层：交互补充
├─ 用户在问答阶段提问时，可触发新数据获取
├─ 问答引擎判断是否需要新数据
├─ 按需获取，增量分析
```

**实现代码架构**：

```python
class DataPlanningEngine:
    """数据规划引擎"""

    def plan_data_requirements(
        self,
        scenario: Scenario,
        initial_data: Dict,
        discovered_metrics: List[str]
    ) -> DataRequirements:
        """
        规划所需数据

        Args:
            scenario: 场景配置
            initial_data: 用户提供的初始数据
            discovered_metrics: 已发现的可用指标列表

        Returns:
            需要获取的数据列表 + 优先级
        """
        # 第1层：基准指标（必需）
        base_metrics = set(scenario.key_metrics)

        # 第2层：LLM智能规划
        llm_suggestions = self._llm_plan_additional_metrics(
            scenario=scenario,
            initial_data=initial_data,
            base_metrics=base_metrics,
            available_sources=self.data_sources
        )

        # 合并 + 去重 + 排序
        all_metrics = self._merge_and_prioritize(
            base_metrics,
            llm_suggestions
        )

        # 第3层：验证可获取性
        verified_requirements = self._verify_availability(
            all_metrics,
            self.data_sources
        )

        return verified_requirements

    def _llm_plan_additional_metrics(
        self,
        scenario: Scenario,
        initial_data: Dict,
        base_metrics: Set,
        available_sources: Dict
    ) -> List[str]:
        """
        使用LLM规划补充数据
        *** 注意：这是额外的LLM调用（mini调用），不在3次主调用中 ***
        """
        prompt = f"""
        场景：{scenario.name}
        当前数据：{initial_data}
        基准指标：{base_metrics}
        分析思路：{scenario.analysis_thoughts}

        根据分析思路，还需要哪些补充指标？
        可用的数据源：{available_sources.keys()}

        返回建议的补充指标列表（JSON格式）
        [{{"metric": "xxx", "source": "xxx", "priority": "high/medium/low"}}, ...]
        """

        response = llm.call(prompt)
        suggestions = json.loads(response)
        return [s['metric'] for s in suggestions]

    def _verify_availability(
        self,
        metrics: List[str],
        data_sources: Dict
    ) -> DataRequirements:
        """验证数据可获取性"""
        results = {
            "available": [],
            "unavailable": [],
            "partial": []
        }

        for metric in metrics:
            try:
                source_info = self._find_metric_source(metric)
                if source_info:
                    results["available"].append({
                        "metric": metric,
                        "source": source_info,
                        "status": "ready"
                    })
                else:
                    results["unavailable"].append(metric)
            except Exception as e:
                results["partial"].append({
                    "metric": metric,
                    "error": str(e)
                })

        return DataRequirements(**results)

class DataAcquisitionExecutor:
    """数据获取执行器"""

    def execute(
        self,
        requirements: DataRequirements,
        scenario: Scenario
    ) -> AcquisitionResult:
        """
        执行数据获取

        Returns:
            {
              "success_data": {...},
              "failed_metrics": [...],
              "source_trace": {
                "metric_name": {
                  "source_system": "sales_api v2.1",
                  "endpoint": "xxx",
                  "fetch_time": "2024-03-17T10:30Z",
                  "record_count": 100
                }
              }
            }
        """
        result = {
            "success_data": {},
            "failed_metrics": [],
            "source_trace": {}
        }

        for item in requirements.available:
            try:
                data = self._fetch_from_source(
                    item['source'],
                    item['metric']
                )
                result["success_data"][item['metric']] = data
                result["source_trace"][item['metric']] = {
                    "source_system": item['source'].system,
                    "endpoint": item['source'].endpoint,
                    "fetch_time": datetime.now().isoformat(),
                    "record_count": len(data)
                }
            except Exception as e:
                result["failed_metrics"].append(item['metric'])
                logger.error(f"Failed to fetch {item['metric']}: {e}")

        # 如果关键指标获取失败，告知用户
        critical_metrics = set(scenario.key_metrics)
        failed_set = set(result['failed_metrics'])
        if critical_metrics & failed_set:
            result["warnings"] = [
                f"Critical metric failed: {m}"
                for m in critical_metrics & failed_set
            ]

        return result
```

**关键特点**：

- ✓ 分层结构，优先级清晰
- ✓ 显式 + 智能结合，balance可靠性和灵活性
- ✓ 完整的来源追踪
- ✓ 降级处理，不会因为部分数据缺失而完全失败

---

### 问题2: 如何在多模型选择中做出最优决策？(Model Selection)

#### 问题分析

- 一个场景可能有多个模型可用，LLM如何选择？
- 不同模型适合不同的数据特征，如何匹配？
- 如何避免模型误选导致分析结果不准确？

#### 解决思路

**多维度模型选择策略**：

```
┌──────────────────────────────────────────────────────────┐
│            模型选择决策框架                              │
├──────────────────────────────────────────────────────────┤
│                                                          │
│  维度1：问题类型适配                                    │
│  ├─ 时间序列数据 → trend_analysis                       │
│  ├─ 变量关系 → correlation_analysis                     │
│  ├─ 用户群体 → segmentation                            │
│  └─ 异常检测 → anomaly_detection                        │
│                                                          │
│  维度2：数据特征匹配                                    │
│  ├─ 样本量大(>1000) → 可用复杂模型                      │
│  ├─ 样本量小(<100) → 用简单稳定模型                    │
│  ├─ 缺失值多(>30%) → 选择鲁棒模型                      │
│  └─ 异常值多 → 用异常处理能力强的                      │
│                                                          │
│  维度3：模型性能                                        │
│  ├─ 历史准确率                                         │
│  ├─ 场景相似度                                         │
│  └─ 用户反馈评分                                       │
│                                                          │
│  维度4：场景约束                                        │
│  ├─ 必需模型 (required)                                │
│  ├─ 推荐模型 (recommended)                              │
│  └─ 可选模型 (optional)                                 │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

**实现架构**：

```python
class ModelSelectionEngine:
    """模型选择引擎"""

    def select_models(
        self,
        scenario: Scenario,
        data: DataFrame,
        available_metrics: List[str]
    ) -> ModelSelectionResult:
        """
        多维度模型选择

        Returns:
            {
              "selected_models": [
                {
                  "model": "trend_analysis",
                  "reason": "...",
                  "confidence": 0.95,
                  "data_compatibility": 0.92
                }
              ],
              "alternatives": [...],
              "reasoning_chain": "..."
            }
        """

        # 维度1: 基于问题类型
        problem_type = self._infer_problem_type(
            scenario,
            available_metrics
        )
        candidates_by_type = self._filter_by_problem_type(
            scenario.available_models,
            problem_type
        )

        # 维度2: 基于数据特征
        data_characteristics = self._analyze_data_characteristics(data)
        candidates_by_data = self._filter_by_data_fit(
            candidates_by_type,
            data_characteristics
        )

        # 维度3: 基于模型历史表现
        candidates_by_performance = self._rank_by_performance(
            candidates_by_data,
            scenario_id=scenario.id,
            data_char=data_characteristics
        )

        # 维度4: 应用场景约束
        final_selection = self._apply_scenario_constraints(
            candidates_by_performance,
            scenario
        )

        return ModelSelectionResult(
            selected=final_selection,
            reasoning=self._generate_reasoning(...)
        )

    def _analyze_data_characteristics(self, data: DataFrame) -> Dict:
        """分析数据特征"""
        return {
            "sample_size": len(data),
            "feature_count": len(data.columns),
            "missing_ratio": data.isnull().sum().sum() / data.size,
            "outlier_ratio": self._detect_outliers(data).sum() / len(data),
            "distribution": self._infer_distribution(data),
            "temporal": self._detect_temporal_pattern(data),
            "dimensionality": "high" if len(data.columns) > 10 else "low"
        }

    def _filter_by_data_fit(
        self,
        models: List[AnalysisModel],
        characteristics: Dict
    ) -> List[AnalysisModel]:
        """根据数据特征筛选模型"""

        # 定义每个模型的数据需求
        model_requirements = {
            "trend_analysis": {
                "min_sample": 7,  # 至少7个时间点
                "temporal_required": True,
                "max_missing_ratio": 0.5,
                "robust_to_outliers": False  # 需要数据清洁
            },
            "correlation_analysis": {
                "min_sample": 30,  # 至少30个样本
                "temporal_required": False,
                "max_missing_ratio": 0.3,
                "robust_to_outliers": True
            },
            "segmentation": {
                "min_sample": 20,
                "temporal_required": False,
                "max_missing_ratio": 0.2,
                "robust_to_outliers": True
            },
            "anomaly_detection": {
                "min_sample": 50,
                "temporal_required": True,
                "max_missing_ratio": 0.3,
                "robust_to_outliers": True
            }
        }

        suitable_models = []
        for model in models:
            req = model_requirements.get(model.name)
            if not req:
                continue

            # 检查是否满足要求
            if self._meets_requirements(characteristics, req):
                suitable_models.append(model)

        return suitable_models

    def _rank_by_performance(
        self,
        models: List[AnalysisModel],
        scenario_id: str,
        data_char: Dict
    ) -> List[Tuple[AnalysisModel, float]]:
        """根据历史表现排序"""

        ranking = []
        for model in models:
            # 获取该模型在类似场景中的历史表现
            historical_accuracy = self._get_historical_accuracy(
                model.name,
                scenario_id,
                data_char
            )

            ranking.append((model, historical_accuracy))

        # 按准确率排序
        return sorted(ranking, key=lambda x: x[1], reverse=True)

    def _apply_scenario_constraints(
        self,
        ranked_models: List[Tuple[AnalysisModel, float]],
        scenario: Scenario
    ) -> List[AnalysisModel]:
        """应用场景约束"""

        # 必需模型一定要选
        selected = list(scenario.required_models)

        # 推荐模型优先选
        recommended_names = {m.name for m in scenario.recommended_models}
        for model, score in ranked_models:
            if model.name in recommended_names:
                if model not in selected:
                    selected.append(model)

        # 补充可选模型（高分的）
        optional_names = {m.name for m in scenario.optional_models}
        for model, score in ranked_models:
            if score > 0.7 and model.name in optional_names:
                if model not in selected and len(selected) < 3:
                    selected.append(model)

        return selected
```

**LLM 输入示例**（在分析规划LLM Call中）：

```json
{
  "scenario": "user_sales_analysis",
  "initial_data": {...},
  "data_characteristics": {
    "sample_size": 500,
    "feature_count": 8,
    "missing_ratio": 0.05,
    "temporal": true,
    "distribution": "right_skewed"
  },
  "available_models": [
    {
      "name": "trend_analysis",
      "suitability_score": 0.95,
      "description": "适合时间序列数据"
    },
    {
      "name": "correlation_analysis",
      "suitability_score": 0.87,
      "description": "适合变量关系分析"
    },
    {
      "name": "anomaly_detection",
      "suitability_score": 0.75,
      "description": "适合异常检测"
    }
  ],
  "required_models": ["trend_analysis"],
  "recommended_models": ["correlation_analysis"],
  "scenario_analysis_thoughts": "先分析趋势，再分析关系..."
}
```

**关键特点**：

- ✓ 多维度决策，避免单一视角
- ✓ 数据驱动，结合历史表现
- ✓ 约束保证，不会违反场景要求
- ✓ 可解释性强，能说明为什么选这个模型

---

### 问题3: 如何确保数据来源准确性和可追溯性？(Data Lineage & Accuracy)

#### 问题分析

- 数据来自多个系统，各系统数据版本可能不同
- 用户对分析结果提问时，需要知道是基于什么数据
- 如何验证数据的准确性和完整性？
- 出现问题时，如何回溯数据来源和处理流程？

#### 解决思路

**三层数据血缘追踪体系**：

```
┌──────────────────────────────────────────────────────────────┐
│          数据血缘追踪系统（Data Lineage Tracking）            │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│ 第1层：源数据血缘                                            │
│ ├─ 数据来源系统和版本                                        │
│ ├─ API端点和获取时间                                         │
│ ├─ 数据行数和字段数                                          │
│ ├─ 数据质量指标（缺失率、异常值跳出调用链                    │
│ └─ 获取状态和错误（如有）                                     │
│                                                              │
│ 第2层：变换血缘                                              │
│ ├─ 数据清洗步骤                                              │
│ │  ├─ 缺失值处理方式                                        │
│ │  ├─ 异常值处理方式                                        │
│ │  ├─ 数据去重或聚合                                        │
│ │  └─ 处理前后样本量对比                                     │
│ ├─ 特征工程                                                 │
│ │  ├─ 衍生字段定义                                          │
│ │  ├─ 变换逻辑（公式）                                      │
│ │  └─ 变换验证结果                                          │
│ └─ 聚合逻辑                                                  │
│    ├─ 分组维度                                              │
│    ├─ 聚合函数                                              │
│    └─ 聚合结果校验                                          │
│                                                              │
│ 第3层：模型血缘                                              │
│ ├─ 使用的模型名称和版本                                      │
│ ├─ 模型参数配置                                              │
│ ├─ 模型中间结果                                              │
│ ├─ 模型输出和置信度                                          │
│ └─ 模型验证结果（如適用）                                     │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

**实现架构**：

```python
class DataLineageTracker:
    """数据血缘追踪器"""

    def __init__(self):
        self.lineage_cache = {}

    def track_data_fetch(
        self,
        metric_name: str,
        data_source: DataSource,
        data: DataFrame,
        fetch_time: datetime
    ) -> DataLineage:
        """追踪源数据获取"""

        lineage_id = f"{metric_name}_{fetch_time.timestamp()}"

        lineage_info = {
            "id": lineage_id,
            "type": "data_fetch",
            "timestamp": fetch_time.isoformat(),
            "source": {
                "system": data_source.system_name,
                "system_version": data_source.system_version,
                "endpoint": data_source.endpoint,
                "api_version": data_source.api_version
            },
            "data_quality": {
                "total_rows": len(data),
                "total_columns": len(data.columns),
                "missing_ratio": data.isnull().sum().sum() / (len(data) * len(data.columns)),
                "null_by_column": data.isnull().sum().to_dict(),
                "outlier_detection": self._detect_outliers(data),
                "data_types": data.dtypes.to_dict()
            },
            "status": "success",
            "hash": self._compute_data_hash(data)  # 用于完整性验证
        }

        # 存储
        self.lineage_cache[lineage_id] = lineage_info

        return DataLineage(**lineage_info)

    def track_data_transformation(
        self,
        input_lineage_id: str,
        transformation_type: str,  # "cleaning" / "engineering" / "aggregation"
        transformation_logic: Dict,
        output_data: DataFrame,
        execute_time: float
    ) -> DataLineage:
        """追踪数据变换"""

        transformation_id = f"transform_{datetime.now().timestamp()}"

        # 获取输入数据信息
        input_info = self.lineage_cache.get(input_lineage_id, {})
        input_shape = (input_info.get("data_quality", {}).get("total_rows"), 0)

        lineage_info = {
            "id": transformation_id,
            "type": "transformation",
            "parent_id": input_lineage_id,
            "transformation_type": transformation_type,
            "timestamp": datetime.now().isoformat(),
            "logic": transformation_logic,
            # 记录变换前后的数据特征对比
            "impact": {
                "input_rows": input_shape[0],
                "output_rows": len(output_data),
                "row_change_ratio": (len(output_data) - input_shape[0]) / input_shape[0] if input_shape[0] > 0 else 0,
                "column_change": len(output_data.columns),
                "missing_ratio_before": input_info.get("data_quality", {}).get("missing_ratio"),
                "missing_ratio_after": output_data.isnull().sum().sum() / (len(output_data) * len(output_data.columns))
            },
            "execute_time_ms": execute_time,
            "details": {
                "cleaning": self._track_cleaning(transformation_logic) if transformation_type == "cleaning" else None,
                "engineering": self._track_engineering(transformation_logic) if transformation_type == "engineering" else None,
                "aggregation": self._track_aggregation(transformation_logic) if transformation_type == "aggregation" else None
            },
            "hash": self._compute_data_hash(output_data)
        }

        self.lineage_cache[transformation_id] = lineage_info

        return DataLineage(**lineage_info)

    def track_model_execution(
        self,
        model_name: str,
        model_version: str,
        input_lineage_id: str,
        model_config: Dict,
        model_output: Dict,
        execution_time: float
    ) -> ModelLineage:
        """追踪模型执行"""

        model_exec_id = f"model_{model_name}_{datetime.now().timestamp()}"

        lineage_info = {
            "id": model_exec_id,
            "type": "model_execution",
            "parent_id": input_lineage_id,
            "timestamp": datetime.now().isoformat(),
            "model": {
                "name": model_name,
                "version": model_version,
                "config": model_config
            },
            "output": model_output,
            "execution": {
                "duration_ms": execution_time,
                "status": "success"
            },
            "validation": {
                "output_validation": self._validate_model_output(model_output),
                "result_sanity": self._check_result_sanity(model_output)
            }
        }

        self.lineage_cache[model_exec_id] = lineage_info

        return ModelLineage(**lineage_info)

    def generate_lineage_report(
        self,
        final_insight_id: str
    ) -> LineageReport:
        """
        生成完整的数据血缘报告

        用于：
        1. 呈现给用户，说明分析基础
        2. 审计和复现
        3. 问题排查
        """

        # 回溯整个计算链
        chain = self._trace_lineage_chain(final_insight_id)

        report = {
            "final_insight_id": final_insight_id,
            "generated_at": datetime.now().isoformat(),
            "lineage_chain": chain,
            "data_sources": self._extract_data_sources(chain),
            "transformations": self._extract_transformations(chain),
            "models_used": self._extract_models(chain),
            "data_quality_summary": self._summarize_quality(chain),
            "assumptions": self._extract_assumptions(chain),
            "limitations": self._extract_limitations(chain)
        }

        return LineageReport(**report)

    def _trace_lineage_chain(self, lineage_id: str) -> List[Dict]:
        """递归回溯数据血缘链"""
        chain = []
        current_id = lineage_id

        while current_id:
            if current_id in self.lineage_cache:
                info = self.lineage_cache[current_id]
                chain.append(info)
                current_id = info.get("parent_id")
            else:
                break

        return list(reversed(chain))

    def _compute_data_hash(self, data: DataFrame) -> str:
        """计算数据哈希，用于完整性验证"""
        import hashlib
        data_str = data.to_string()
        return hashlib.sha256(data_str.encode()).hexdigest()

class DataAccuracyValidator:
    """数据准确性验证器"""

    def validate_data_consistency(
        self,
        metric_name: str,
        lineage: DataLineage
    ) -> ValidationResult:
        """验证数据一致性"""

        checks = {
            "schema_validation": self._check_schema(lineage),
            "range_validation": self._check_value_range(lineage),
            "duplicate_detection": self._check_duplicates(lineage),
            "temporal_consistency": self._check_temporal_order(lineage),
            "business_rule_validation": self._check_business_rules(metric_name, lineage)
        }

        result = {
            "passed_checks": sum(1 for v in checks.values() if v['passed']),
            "total_checks": len(checks),
            "accuracy_score": sum(1 for v in checks.values() if v['passed']) / len(checks),
            "details": checks
        }

        return ValidationResult(**result)

    def compare_data_versions(
        self,
        metric_name: str,
        lineage1: DataLineage,
        lineage2: DataLineage
    ) -> ComparisonResult:
        """对比不同版本的数据，检测数据变化"""

        # 获取两个版本的数据
        data1 = self._reconstruct_data(lineage1)
        data2 = self._reconstruct_data(lineage2)

        comparison = {
            "data1_rows": len(data1),
            "data2_rows": len(data2),
            "row_difference": len(data2) - len(data1),
            "value_changes": self._detect_value_changes(data1, data2),
            "new_records": self._find_new_records(data1, data2),
            "deleted_records": self._find_deleted_records(data1, data2),
            "data_drift_detected": self._detect_data_drift(data1, data2)
        }

        return ComparisonResult(**comparison)
```

**用户问答时的信息展示**：

```python
class UserFacingLineage:
    """用户端的血缘展示"""

    def generate_data_source_explanation(
        self,
        lineage_report: LineageReport
    ) -> str:
        """
        生成用户理解的数据来源说明
        """

        explanation = f"""
        ## 分析流程和数据来源说明

        ### 第一步：数据获取
        本次分析使用了以下数据源：

        {self._format_data_sources(lineage_report.data_sources)}

        数据获取时间：{lineage_report.data_sources[0]['fetch_time']}
        样本量：{lineage_report.data_sources[0]['row_count']} 行

        ### 第二步：数据处理
        为了确保分析准确，对数据进行了以下处理：

        {self._format_transformations(lineage_report.transformations)}

        ### 第三步：模型分析
        使用了以下分析模型：

        {self._format_models(lineage_report.models_used)}

        ### 数据质量声明
        - 数据完整性：{lineage_report.data_quality_summary['completeness']}%
        - 异常值分比：{lineage_report.data_quality_summary['outlier_ratio']}%
        - 最后更新：{lineage_report.data_sources[-1]['last_update_time']}

        ### 重要假设
        {self._format_assumptions(lineage_report.assumptions)}

        ### 分析限制
        {self._format_limitations(lineage_report.limitations)}
        """

        return explanation

    def _format_data_sources(self, sources: List[Dict]) -> str:
        """格式化数据源信息"""
        result = []
        for src in sources:
            result.append(f"""
            - **{src['system_name']}** (版本 {src['system_version']})
              - 数据指标：{src['metric_name']}
              - 样本量：{src['row_count']} 行，{src['column_count']} 列
              - 缺失率：{src['missing_ratio']}%
              - 获取方式：{src['endpoint']}
              - 获取时间：{src['fetch_time']}
            """)
        return "\n".join(result)

    def _format_transformations(self, transforms: List[Dict]) -> str:
        """格式化变换步骤"""
        result = []
        for i, tr in enumerate(transforms, 1):
            result.append(f"""
            {i}. **{tr['type']}**
               - 操作：{tr['description']}
               - 影响：{tr['input_rows']} → {tr['output_rows']} 行
               - 执行时间：{tr['execute_time_ms']}ms
            """)
        return "\n".join(result)
```

**关键特点**：

- ✓ 三层追踪，完整的数据链路
- ✓ 源数据 → 变换 → 模型 → 结果，每步可验证
- ✓ 用户友好的血缘说明
- ✓ 支持数据对比和异常检测
- ✓ 审计和复现能力

---

## 总结

### 架构要点

1. **分层设计**：用户交互层 → 应用编排层 → 数据/配置层
2. **LLM精准使用**：3次关键调用，其他都是计算或API调用
3. **数据为中心**：完整的获取、变换、追踪、验证体系
4. **可扩展**：配置驱动，易于新增场景、模型、数据源

### 核心决策

1. **分层递进的数据规划**：显式基准 + LLM智能 + 动态验证
2. **多维模型选择**：问题类型 + 数据特征 + 历史表现 + 场景约束
3. **全链路血缘追踪**：源数据 → 变换 → 模型，每步可验证和解释

### 优势

- ✓ LLM参与规划和解释，不参与计算（准确性有保障）
- ✓ 完整的数据溯源（可信度高）
- ✓ 灵活的场景适配（可扩展性强）
- ✓ 用户友好的问答体验（可交互性好）
