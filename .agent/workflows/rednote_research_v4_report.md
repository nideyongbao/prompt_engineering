---
description: RedNote Research V4.0 - Step 4-5 报告生成模块
---

# Step 4-5: 大纲与报告生成

> [!NOTE]
> V4 版本：报告生成逻辑基本保持 v3，同时在 Step 5 结束后为 Step 6 准备发布数据

---

## Step 4: 大纲生成

**目标**：基于分析结果生成结构化报告大纲，并进行智能图片匹配

### 4.1 章节规划
- 根据研究维度确定主要章节（4-6 个）
- 每章节确定 2-3 个要点

### 4.2 图片匹配（V4 简化）

> [!IMPORTANT]
> **V4 优化**：由于图片已在 notes_detail.json 中保存为 URL，简化为直接选择相关图片。

**执行逻辑**：
```python
FOR each section in outline:
    # 从笔记图片中选择最相关的
    section_images = select_relevant_images(
        section_content=section,
        available_images=notes_data['image_urls'],
        relevance_scores=analysis_data['image_relevance_scores'],
        max_count=2  # 每章节最多2张图
    )
    
    IF not section_images:
        # 回退：生成图片
        section.needs_generation = True
        section.image_prompt = generate_image_prompt(section)
```

### 4.3 大纲输出格式

```markdown
## 📋 报告大纲

1. **引言**
   - 研究背景
   - 核心问题

2. **[维度1标题]**
   - 要点 A
   - 要点 B
   - 📷 图片来源：笔记原图（2张）

3. **[维度2标题]**
   - 要点 A
   - 要点 B
   - 🎨 图片来源：AI生成

4. **结论与建议**
   - 关键结论
   - 行动建议
```

---

## Step 5: 报告生成

**目标**：生成图文交错的研究报告

### 5.1 逐章节生成
- 按大纲结构逐章撰写
- 每章节 150-250 字，整体 1500-2000 字
- 引用原始笔记作为数据支撑

### 5.2 图片处理

```python
FOR each section in outline:
    IF section.section_images:
        FOR image_url in section.section_images:
            embed_image(image_url, caption="来源：小红书笔记")
    
    ELSE IF section.needs_generation:
        generated_image = generate_image(
            Prompt=section.image_prompt,
            ImageName=f"section_{section.index}_generated"
        )
        embed_image(generated_image, caption="来源：AI生成")
```

> [!NOTE]
> V4 直接嵌入图片 URL，无需下载到本地。

### 5.3 报告输出格式

```markdown
# [研究主题] 深度研究报告

> 📅 生成时间：[日期时间]
> 📊 数据来源：小红书 [N] 篇笔记
> 🎯 复杂度等级：[低/中/高]

## 📋 摘要

[150字以内的核心发现总结]

---

## 1. [章节1标题]

[150-250字正文内容，包含数据引用]

![章节1配图](http://sns-webpic-qc.xhscdn.com/...)
*图片来源：笔记原图*

---

## 2. [章节2标题]

[150-250字正文内容]

![章节2配图](http://sns-webpic-qc.xhscdn.com/...)
*图片来源：笔记原图*

---

## 💡 结论与建议

### 核心结论
1. [结论1]
2. [结论2]

### 行动建议
- [建议1]
- [建议2]

---

## 📚 参考笔记

| # | 笔记标题 | 作者 | 点赞数 | 链接 |
|---|----------|------|--------|------|
| 1 | [标题] | @作者 | N | [查看](URL) |
| 2 | [标题] | @作者 | N | [查看](URL) |

---

*本报告由 RedNote Research Agent V4.0 自动生成*
*支持一键发布到小红书，详见 Step 6*
```

**格式规范**：
- 总字数控制在 1500-2000 字
- 每章节 150-250 字
- 图片使用在线 URL，必须标注来源
- 数据引用需注明原始笔记

---

## Step 5.4: 为发布准备数据（V4 新增）

```python
# 报告生成完成后，准备发布所需数据
publish_preparation = {
    "report_md": report_content,
    "analysis_data": analysis_json,
    "notes_data": notes_detail_json
}

# 保存到 DATA_DIR 供 Step 6 使用
save_json(f"{DATA_DIR}/publish_preparation.json", publish_preparation)
```

---

*V4.0 更新时间：2025-12-28*
