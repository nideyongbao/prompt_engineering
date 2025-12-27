---
description: RedNote Research V4.0 - Step 3 分析模块
---

# Step 3: 分析阶段 (Analysis Phase)

**目标**：从收集的笔记中提取洞察

> [!NOTE]
> V4 版本：分析逻辑与 MCP 工具无关，保持 v3 版本即可

---

## 3.1 分批分析

**问题**：笔记量大时，一次性送入 LLM 可能超出 token 限制

**解决方案**：
```python
batch_size = 10  # 每批 10 篇笔记
all_insights = []

FOR batch in chunks(documents, batch_size):
    batch_insights = analyze_batch(batch)
    all_insights.append(batch_insights)

# 汇总所有批次的洞察
final_insights = synthesize_insights(all_insights)
```

---

## 3.2 结构化输出

**问题**：JSON 手动解析容易出错

**解决方案**：在 LLM 调用时使用 `response_format`
```python
llm_response = call_llm(
    prompt=analysis_prompt,
    response_format={"type": "json_object"},
    temperature=0.5,
    max_tokens=4000
)
```

**输出 JSON 结构定义**：
```json
{
  "topic": "研究主题",
  "key_findings": [
    {"finding": "发现内容", "source": "来源笔记标题", "confidence": 0.9}
  ],
  "user_pain_points": [
    {"pain_point": "痛点描述", "frequency": "高/中/低"}
  ],
  "recommendations": [
    {"recommendation": "建议内容", "priority": "高/中/低"}
  ],
  "top_keywords": ["关键词1", "关键词2", "关键词3"],
  "image_relevance_scores": {
    "image_url_1": 0.95,
    "image_url_2": 0.88
  },
  "needs_more_data": false,
  "additional_keywords": [],
  "confidence": 0.85
}
```

> [!IMPORTANT]
> **V4 新增字段**：
> - `topic`: 研究主题（供封面图生成使用）
> - `top_keywords`: 高频关键词（供表情符号插入使用）
> - `image_relevance_scores`: 图片相关性打分（供发布时选图使用）

---

## 3.3 反思循环

```python
iteration = 0
max_iterations = complexity_based_limit  # 来自 Step 0

WHILE iteration < max_iterations:
    insights = analyze(documents)
    
    IF insights.needs_more_data AND insights.additional_keywords:
        new_docs = search(insights.additional_keywords)
        documents.extend(new_docs)
        iteration += 1
    ELSE:
        BREAK
```

---

## 3.4 输出格式

```markdown
## 🧠 内容分析

**分析统计**：
- 分析笔记数：[N] 篇
- 分批数量：[M] 批
- 迭代次数：[K] 次
- 总体置信度：[X]%

### 核心发现
1. [发现1] — 来源：[笔记标题]（置信度：[X]%）
2. [发现2] — 来源：[笔记标题]（置信度：[X]%）

### 用户痛点
- [痛点1]（出现频率：高）
- [痛点2]（出现频率：中）

### 行动建议
- [建议1]（优先级：高）
- [建议2]（优先级：中）

### 高频关键词
- [关键词1], [关键词2], [关键词3]
```

---

## 3.5 网络搜索增强 [可选]

**触发条件**：用户需求涉及业界对比或专业数据

```python
mcp_tavily_tavily-search(
    query="[相关查询]",
    max_results=5,
    search_depth="basic"
)
```

**整合方式**：
- 将搜索结果与笔记分析对比
- 在报告中标注"业界参考数据"

---

*V4.0 更新时间：2025-12-28*
