---
description: RedNote Research V4.0 - Step 2.5 图片理解与补充生成模块
---

# Step 2.5: 图片理解与补充生成 (Image Understanding & Generation)

**目标**：确保发布到小红书的图片质量和数量达标

> [!IMPORTANT]
> **小红书平台特性**：图片重要性 > 文字重要性
> - 用户首先被图片吸引，然后才阅读文字
> - 图片质量直接影响点击率和停留时间
> - 建议图片数量：5-9 张（最佳：5-7张）

---

## 2.5.1 图片理解与筛选

**目标**：使用视觉模型理解并筛选高质量图片

### 执行流程

```python
def analyze_and_filter_images(notes_data, topic):
    """
    对笔记中的所有图片进行理解和筛选
    """
    all_images = []
    
    # 收集所有图片 URL
    FOR note in notes_data:
        FOR image_url in note['image_urls']:
            all_images.append({
                "url": image_url,
                "note_title": note['title'],
                "note_id": note['noteId']
            })
    
    # 批量理解图片（使用视觉模型）
    image_analysis_results = []
    
    FOR image in all_images:
        analysis = analyze_image_with_vision_model(
            image_url=image['url'],
            analysis_aspects=[
                "content_description",  # 内容描述
                "relevance_to_topic",   # 与主题相关性
                "visual_quality",       # 视觉质量
                "xiaohongshu_style",    # 小红书风格匹配度
                "emotional_appeal"      # 情感吸引力
            ]
        )
        
        image_analysis_results.append({
            **image,
            "analysis": analysis,
            "quality_score": calculate_quality_score(analysis),
            "relevance_score": calculate_relevance_score(analysis, topic)
        })
    
    # 筛选高质量图片
    filtered_images = filter_high_quality_images(
        image_analysis_results,
        min_quality_score=0.7,
        min_relevance_score=0.6,
        target_count=7  # 目标7张图
    )
    
    return filtered_images
```

### 视觉模型分析 Prompt

**基础模板**：
```
分析这张图片，用于小红书发布：

请从以下维度评分（0-1）：
1. **内容相关性**：图片内容与"{topic}"的相关程度
2. **视觉质量**：清晰度、构图、色彩、光线
3. **风格匹配**：是否符合小红书风格（真实、生活化、有质感）
4. **情感吸引**：是否能引发共鸣、吸引眼球

输出 JSON 格式：
{
  "content_description": "图片内容简短描述",
  "relevance_score": 0.0-1.0,
  "quality_score": 0.0-1.0,
  "style_score": 0.0-1.0,
  "emotional_score": 0.0-1.0,
  "is_suitable": true/false,
  "reason": "适合/不适合的原因"
}
```

**示例分析**（美食主题）：
```json
{
  "content_description": "烤肉特写，炭火烟雾缭绕，肉质鲜嫩金黄",
  "relevance_score": 0.95,
  "quality_score": 0.88,
  "style_score": 0.92,
  "emotional_score": 0.90,
  "is_suitable": true,
  "reason": "高清美食特写，色彩诱人，符合小红书美食风格"
}
```

### 筛选规则

```python
def filter_high_quality_images(analysis_results, min_quality_score, min_relevance_score, target_count):
    """
    筛选高质量图片
    """
    # 综合打分
    FOR result in analysis_results:
        result['composite_score'] = (
            result['analysis']['relevance_score'] * 0.4 +
            result['analysis']['quality_score'] * 0.3 +
            result['analysis']['style_score'] * 0.2 +
            result['analysis']['emotional_score'] * 0.1
        )
    
    # 过滤低分图片
    qualified_images = [
        img for img in analysis_results
        if img['analysis']['is_suitable'] 
        and img['quality_score'] >= min_quality_score
        and img['relevance_score'] >= min_relevance_score
    ]
    
    # 按综合分排序
    qualified_images.sort(key=lambda x: x['composite_score'], reverse=True)
    
    # 去重（避免同一笔记的相似图片）
    deduplicated_images = deduplicate_by_note(qualified_images)
    
    return deduplicated_images[:target_count]
```

---

## 2.5.2 图片补充生成

**触发条件**：筛选后的图片数量 < 5 张

### 执行流程

```python
def supplement_images_if_needed(filtered_images, notes_data, analysis_data, target_count=7):
    """
    如果筛选图片不足，生成补充图片
    """
    current_count = len(filtered_images)
    
    IF current_count >= target_count:
        LOG_INFO(f"已有 {current_count} 张高质量图片，无需补充")
        return filtered_images
    
    # 计算需要生成的图片数量
    needed_count = target_count - current_count
    LOG_WARNING(f"仅有 {current_count} 张合格图片，需要生成 {needed_count} 张")
    
    # 基于报告章节生成图片
    generated_images = []
    
    # 从分析数据中提取章节主题
    chapters = extract_chapter_topics(analysis_data)
    
    FOR i, chapter in enumerate(chapters[:needed_count]):
        # 生成章节配图
        image_prompt = build_chapter_image_prompt(
            chapter_topic=chapter['topic'],
            key_points=chapter['key_points'],
            style_reference=analyze_existing_images_style(filtered_images),
            platform="xiaohongshu"
        )
        
        generated_image = generate_image(
            Prompt=image_prompt,
            ImageName=f"chapter_{i+1}_{chapter['topic']}"
        )
        
        # 保存到 publish 目录
        image_path = f"{PUBLISH_DIR}/generated_chapter_{i+1}.png"
        save_image(generated_image, image_path)
        
        generated_images.append({
            "url": image_path,
            "source": "ai_generated",
            "chapter": chapter['topic'],
            "composite_score": 0.85  # 生成图片默认高分
        })
    
    # 合并筛选图片和生成图片
    all_images = filtered_images + generated_images
    
    return all_images
```

### 章节图片生成 Prompt

**模板**：
```
为小红书笔记生成章节配图

章节主题：{chapter_topic}
核心要点：
- {key_point_1}
- {key_point_2}

风格参考（来自已筛选图片）：
- 色调：{dominant_colors}
- 构图：{composition_style}
- 视觉元素：{visual_elements}

小红书平台要求：
✨ 真实感强，避免过度 AI 痕迹
🎨 色彩鲜艳但不俗气，符合生活美学
📐 构图简洁，主体突出
🔥 能引发共鸣，激发互动欲望

技术规格：
- 尺寸：1080x1440px（3:4竖版）
- 风格：摄影级真实感
- 质量：高清，无水印
```

**美食类示例**：
```
为小红书笔记生成章节配图

章节主题：烤肉店推荐
核心要点：
- 炭火现烤，肉质鲜嫩
- 酱料秘制，香气四溢
- 人均150元，性价比高

风格参考：
- 色调：暖橙色 + 金黄色（食欲感）
- 构图：45°俯视角（展示全貌）
- 视觉元素：烤架、火焰、新鲜食材

小红书平台要求：
✨ 真实烤肉场景，冒热气的视觉效果
🎨 暖色调为主，配合木质餐桌纹理
📐 突出烤肉主体，背景虚化
🔥 让人看了就想去吃

技术规格：
- 尺寸：1080x1440px
- 风格：自然光摄影 + 饱和度+15%
- 质量：8K 渲染
```

---

## 2.5.3 图片质量保障策略

### 三级质量标准

| 等级 | 数量 | 要求 | 处理方式 |
|------|------|------|---------|
| **优秀** | ≥7张 | 综合分 ≥0.8 | 直接使用，无需生成 |
| **良好** | 5-6张 | 综合分 ≥0.7 | 补充生成 1-2 张 |
| **不足** | <5张 | 综合分 <0.7 | 补充生成 3-5 张 |

### 图片顺序优化

```python
def optimize_image_order(all_images):
    """
    优化图片顺序，确保最佳视觉效果
    """
    # 原则1: 首图必须是吸引力最强的
    first_image = max(all_images, key=lambda x: x['analysis']['emotional_score'])
    
    # 原则2: 笔记原图和生成图片交替排列
    original_images = [img for img in all_images if img['source'] != 'ai_generated']
    generated_images = [img for img in all_images if img['source'] == 'ai_generated']
    
    # 原则3: 色调变化，避免视觉疲劳
    optimized_order = interleave_by_color_contrast(original_images, generated_images)
    
    # 确保首图在第一位
    optimized_order.remove(first_image)
    optimized_order.insert(0, first_image)
    
    return optimized_order
```

---

## 2.5.4 输出格式

### 图片清单

**文件**：`{DATA_DIR}/images_final.json`

```json
{
  "total_count": 7,
  "original_count": 5,
  "generated_count": 2,
  "images": [
    {
      "order": 1,
      "url": "http://sns-webpic-qc.xhscdn.com/...",
      "source": "note_original",
      "note_title": "深圳必吃烤肉店",
      "composite_score": 0.92,
      "analysis": {
        "content_description": "烤肉特写，炭火烟雾缭绕",
        "relevance_score": 0.95,
        "quality_score": 0.88
      }
    },
    {
      "order": 2,
      "url": "D:/work/workspace/.../publish/generated_chapter_1.png",
      "source": "ai_generated",
      "chapter": "海鲜餐厅推荐",
      "composite_score": 0.85
    }
  ]
}
```

### 控制台输出

```markdown
## 🖼️ 图片处理结果

**筛选统计**：
- 原始图片总数：35 张
- 视觉模型分析：35 张
- 筛选合格图片：5 张（综合分 ≥0.75）
- AI 补充生成：2 张

**最终图片清单**：
| # | 来源 | 描述 | 综合分 |
|---|------|------|--------|
| 1 | 笔记原图 | 烤肉特写，炭火烟雾缭绕 | 0.92 |
| 2 | AI生成 | 海鲜餐厅场景 | 0.85 |
| 3 | 笔记原图 | 甜品店环境，网红打卡 | 0.88 |
| 4 | AI生成 | 美食大合集 | 0.85 |
| 5 | 笔记原图 | 餐厅外观，装修精致 | 0.80 |
| 6 | 笔记原图 | 特色菜品，摆盘精美 | 0.86 |
| 7 | 笔记原图 | 用餐氛围，情侣约会 | 0.83 |

**质量等级**：优秀（7张图片，综合分均 ≥0.80）
```

---

## 2.5.5 错误处理

| 错误场景 | 处理策略 |
|---------|---------|
| 所有图片 URL 失效 | 全部使用 AI 生成（7张） |
| 视觉模型调用失败 | 降级到基于文本关键词筛选 |
| 图片生成失败 | 重试3次，失败则跳过 |
| 图片数量仍不足 | 最低保证 3 张（1封面+2配图） |

---

*V4.0 更新时间：2025-12-28*
*图片优先策略：确保小红书发布成功的关键*
