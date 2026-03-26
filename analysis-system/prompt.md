# LLM 提示词参考

本文档包含AI分析系统中各LLM调用的提示词模板和最佳实践。

---

## 调用1: 分析计划生成

### 核心职责

基于初始数据和场景配置，生成结构化的分析计划。

### 提示词模板

```
你是一个数据分析专家。根据以下信息，生成一份详细的分析计划。

【业务场景】
{scenario_name}
场景描述: {scenario_description}

【初始数据】
{initial_metrics}

示例:
- 转化率: 2.5% (环比↓5%)
- 订单量: 12,450 (环比↑12%)
- 客单价: ¥450 (环比→)

【可用分析框架】
{available_frameworks}

示例:
- 5W2H: 多维度问题诊断
- 对标分析: 竞品对比
- SWOT: 竞争力分析

【可用数据获取能力(Skill)】
{available_skills}

示例:
- user_cohort: 用户分群分析
- funnel: 转化漏斗分析
- benchmark: 竞品对标
- timeseries: 趋势分析
- geo_distribution: 地域分布

【分析目标】
找到导致转化率下降的根本原因，并提出改进建议。

【输出要求】
必须返回有效的JSON格式，结构如下：

{
  "analysisTitle": "string",           // 分析标题
  "reasoning": "string",                // 为什么选择这个计划
  "frameworks": [                       // 要使用的分析框架(1-3个)
    {
      "name": "5W2H" | "benchmark" | "SWOT",
      "purpose": "string"               // 该框架的作用
    }
  ],
  "steps": [                            // 执行步骤
    {
      "step": 1,                        // 步骤序号
      "skillId": "string",              // 使用的Skill ID
      "description": "string",          // 这一步的目的
      "dependencies": [0]               // 依赖的前序步骤序号(0表示无依赖)
    }
  ],
  "expectedDuration": "30s"             // 预计耗时
}

【约束条件】
1. 选择的Skill必须从【可用数据获取能力】中选择
2. 选择的框架必须从【可用分析框架】中选择
3. 总步骤数不超过8个
4. 必须返回标准JSON格式，不要添加其他文本
5. 如果某个Skill不适用，不要强行使用

【示例响应】
{
  "analysisTitle": "转化率下降5%的原因诊断与优化建议",
  "reasoning": "先使用5W2H框架拆解问题要素，再通过对标分析找出与竞品的差距。优先分析新客转化率下降(环比-8%)这个重点问题。",
  "frameworks": [
    {
      "name": "5W2H",
      "purpose": "多维度诊断转化率下降的原因"
    },
    {
      "name": "benchmark",
      "purpose": "对比竞品，找出差距和机会"
    }
  ],
  "steps": [
    {
      "step": 1,
      "skillId": "user_cohort",
      "description": "按新老客、设备类型分群，识别哪些用户群体转化率下降最严重",
      "dependencies": []
    },
    {
      "step": 2,
      "skillId": "funnel",
      "description": "分析转化漏斗各环节，定位卡点",
      "dependencies": []
    },
    {
      "step": 3,
      "skillId": "timeseries",
      "description": "时间序列分析，找出转化率下降的时间节点和规律",
      "dependencies": []
    },
    {
      "step": 4,
      "skillId": "benchmark",
      "description": "对标竞品和行业平均值，找出差距",
      "dependencies": [1]
    }
  ],
  "expectedDuration": "25s"
}

请生成分析计划。
```

### 调用方式

```python
def generate_analysis_plan(scenario: dict, initial_data: dict) -> dict:
    prompt = build_prompt(scenario, initial_data)
    response = llm.chat(
        model="gpt-4o-mini",  # 推荐: GPT-4o-mini (快速便宜)
        messages=[{"role": "user", "content": prompt}],
        temperature=0.3,      # 降低温度，增加确定性
        response_format={"type": "json_object"},  # 强制JSON输出
        max_tokens=1000
    )
    return json.loads(response.content)
```

---

## 调用2: 洞察提取

### 核心职责

将框架处理后的结构化数据转化为人类可理解的洞察。

### 提示词模板

```
你是一位数据分析解释专家。根据以下分析结果，提取关键洞察。

【分析框架】
5W2H 分析

【框架处理结果】
{framework_result}

示例:
{
  "Why": [
    "新客在购物车阶段转化率特别低(8%)",
    "移动端首屏加载速度变慢(+200ms)"
  ],
  "What": [
    "总体转化率从2.6%下降至2.5%",
    "新客转化率下降8%，老客仅下降2%"
  ],
  "When": [
    "周二-周三高峰时段转化率最低",
    "下降最明显的时间是3月15日"
  ],
  "Where": [
    "北方地区受影响最大(↓8%)",
    "华东地区相对稳定(↓2%)"
  ],
  "Who": [
    "新用户群体受影响最大",
    "高消费用户(客单价>500)转化率下降较少"
  ]
}

【数据来源信息】
{data_sources}

示例:
[
  {
    "skill": "user_cohort",
    "source": "user_system",
    "timestamp": "2024-03-17 10:30",
    "confidence": 0.99
  },
  {
    "skill": "funnel",
    "source": "order_system",
    "timestamp": "2024-03-17 10:25",
    "confidence": 0.98
  }
]

【输出要求】
返回JSON格式的洞察结果:

{
  "keyInsights": [
    {
      "rank": 1,              // 重要性排序
      "insight": "string",    // 洞察内容
      "evidence": [           // 支撑数据
        "数据点1",
        "数据点2"
      ],
      "impact": "high|medium|low"  // 影响程度
    }
  ],
  "rootCauses": [             // 根本原因分析
    {
      "cause": "string",
      "affectedUsers": "number|percent",
      "recommendations": ["建议1", "建议2"]
    }
  ],
  "opportunities": [          // 优化机会
    {
      "opportunity": "string",
      "estimatedLift": "x%"
    }
  ]
}

【约束条件】
1. 洞察应该是具体的、有数据支撑的
2. 最多列出5个关键洞察
3. 避免模糊的表述，如"可能"、"似乎"
4. 每个洞察都必须标注数据来源
5. 排序应该按影响程度从大到小

请提取洞察。
```

### 调用方式

```python
def extract_insights(framework_result: dict, data_sources: list) -> dict:
    prompt = build_insights_prompt(framework_result, data_sources)
    response = llm.chat(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.5,      # 允许一定的创意
        response_format={"type": "json_object"},
        max_tokens=1500
    )
    return json.loads(response.content)
```

---

## 调用3: 报告生成

### 核心职责

综合所有洞察和建议，生成结构化的分析报告。

### 提示词模板

```
根据以下分析信息，生成一份完整的分析报告。

【分析题目】
{title}

【关键洞察】
{key_insights}

【根本原因】
{root_causes}

【优化建议】
{recommendations}

【数据来源】
{data_sources_text}

【输出格式要求】
请以Markdown格式生成清晰的分析报告，不要返回JSON。

结构要求:
1. 执行摘要 (2-3句话)
2. 核心发现 (使用📊图表符号)
3. 根本原因分析 (分点叙述)
4. 改进建议 (按优先级排序)
5. 预期效果
6. 数据来源说明

保持专业语气，结果要可信、可操作。
```

### 调用方式

```python
def generate_report(
    insights: dict,
    recommendations: list,
    data_sources: list
) -> str:
    prompt = build_report_prompt(insights, recommendations, data_sources)
    response = llm.chat(
        model="gpt-4o",  # 报告生成用完整模型
        messages=[{"role": "user", "content": prompt}],
        temperature=0.7,
        max_tokens=2000
    )
    return response.content
```

---

## 调用4: 用户问答

### 核心职责

基于分析报告和历史上下文，回答用户的追问。

### 提示词模板

```
你是一位数据分析顾问。用户基于以下分析报告提出了问题。

【原分析报告】
{analysis_report}

【用户问题】
{user_question}

【上下文】
分析场景: {scenario}
分析时间: {timestamp}

【回答要求】
1. 基于报告内容和数据来回答，不要凭空编造
2. 如果问题超出分析范围，明确说明
3. 提供具体的数据支撑
4. 保持逻辑一致性
5. 保持专业语气

请直接给出答复，不要用JSON格式。
```

### 调用方式

```python
def answer_user_question(
    question: str,
    analysis_report: dict,
    conversation_history: list
) -> str:
    prompt = build_qa_prompt(question, analysis_report)

    # 利用对话历史提升准确性
    messages = conversation_history + [
        {"role": "user", "content": prompt}
    ]

    response = llm.chat(
        model="gpt-4o-mini",
        messages=messages,
        temperature=0.6,
        max_tokens=800
    )
    return response.content
```

---

## 最佳实践

### 1. Prompt缓存优化

```python
# 对于可复用的系统Prompt，使用缓存
SYSTEM_PROMPT = """
你是数据分析AI助手。
- 所有分析基于数据，不凭主观推测
- 始终明确标注数据来源
- ...
"""

def call_with_cache(user_prompt):
    return llm.chat(
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": user_prompt}
        ],
        cache_control=[{"type": "ephemeral"}]
    )
```

### 2. 温度参数策略

| 调用     | 温度 | 原因                       |
| -------- | ---- | -------------------------- |
| 分析计划 | 0.3  | 需要符合规则，选择最佳方案 |
| 洞察提取 | 0.5  | 需要洞察力，但基于数据     |
| 报告生成 | 0.7  | 需要表达力和创意           |
| 用户问答 | 0.6  | 平衡准确和表达             |

### 3. 错误处理

```python
def call_llm_with_retry(
    prompt: str,
    max_retries: int = 3,
    expected_json: bool = False
) -> dict | str:
    for attempt in range(max_retries):
        try:
            response = llm.chat(prompt)

            if expected_json:
                return json.loads(response)
            return response

        except json.JSONDecodeError:
            if attempt < max_retries - 1:
                # 追加修正Prompt
                prompt += "\n\n请检查并修正JSON格式错误。"
            else:
                raise
        except Exception as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)  # 指数退避
            else:
                raise
```

### 4. 成本控制

```python
# 使用更便宜的模型进行快速分析
MODEL_STRATEGY = {
    "plan_generation": "gpt-4o-mini",  # ¥0.15/1M
    "insights": "gpt-4o-mini",          # ¥0.15/1M
    "report": "gpt-4o",                 # ¥5/1M (精品报告才用)
    "qa": "gpt-4o-mini"                 # ¥0.15/1M
}

# 平均成本/分析: ~¥0.01
```

---

## 提示词调试检查表

- [ ] Prompt是否包含明确的约束条件?
- [ ] Prompt中的示例是否准确?
- [ ] 是否明确了输出格式?
- [ ] 是否包含了所有必要的上下文?
- [ ] 温度参数是否合理?
- [ ] 是否有错误处理机制?
- [ ] 是否考虑了成本控制?
