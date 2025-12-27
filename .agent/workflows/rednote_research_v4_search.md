---
description: RedNote Research V4.0 - Step 2 搜索模块
---

# Step 2: 搜索阶段 (Search Phase)

**目标**：通过 xiaohongshu-mcp 高效收集小红书笔记的**完整内容和图片 URL**

> [!IMPORTANT]
> **核心策略**：搜索后立即获取详情，确保 xsec_token 在有效期内使用。
> **V4 优化**：不下载图片，仅保存在线 URL，提升速度并避免手动确认。

---

## 2.1 MCP 连接检查

```python
# 验证 xiaohongshu-mcp 可用
mcp_xiaohongshu-mcp_check_login_status()
```

> [!WARNING]
> 如果未登录，提示用户：
> "请在浏览器中登录小红书账号，xiaohongshu-mcp 服务会自动获取 session cookie"

---

## 2.2 搜索 + 详情获取 (串行执行)

> [!CAUTION]
> **关键改进**：每次搜索后，立即对搜索结果中的每条笔记调用 `get_feed_detail`。
> **xsec_token 管理**：从搜索结果中提取 token，立即使用，降低过期风险。

### 执行流程

```python
all_notes = []
request_interval = 2  # 秒，请求间隔

FOR each keyword in keywords:
    # 1. 执行搜索
    search_results = mcp_xiaohongshu-mcp_search_feeds(
        keyword=keyword,
        filters={
            "sort_by": "综合",      # 根据复杂度动态调整
            "publish_time": "半年内" if complexity == "高" else "不限",
            "note_type": "不限",
            "location": "不限",
            "search_scope": "不限"
        }
    )
    wait(request_interval)
    
    # 2. 提取 feeds 列表
    feeds = search_results['feeds']
    
    # 3. 立即获取每条笔记的详情
    FOR feed in feeds[:notes_per_keyword]:  # 限制数量
        feed_id = feed['id']
        xsec_token = feed['xsecToken']  # 关键：从搜索结果提取
        
        TRY:
            detail = mcp_xiaohongshu-mcp_get_feed_detail(
                feed_id=feed_id,
                xsec_token=xsec_token  # 立即使用，降低过期风险
            )
            
            # 4. 提取数据
            note = {
                "noteId": detail['data']['note']['noteId'],
                "xsec_token": xsec_token,  # 保存供后续使用
                "title": detail['data']['note']['title'],
                "content": detail['data']['note']['desc'],
                "author": detail['data']['note']['user']['nickname'],
                "image_urls": [  # ✅ 仅保存 URL，不下载
                    img['urlDefault'] 
                    for img in detail['data']['note']['imageList']
                ],
                "likes": detail['data']['note']['interactInfo']['likedCount'],
                "comments": detail['data']['note']['interactInfo']['commentCount'],
                "collects": detail['data']['note']['interactInfo']['collectedCount'],
                "url": f"https://www.xiaohongshu.com/explore/{feed_id}",
                "search_keyword": keyword,
                "fetch_failed": False
            }
            
            all_notes.append(note)
            wait(request_interval)
            
        EXCEPT as e:
            LOG_WARNING(f"详情获取失败: {feed_id}, 错误: {e}")
            # 降级：使用搜索结果的预览数据
            note = {
                "noteId": feed['id'],
                "title": feed['noteCard']['displayTitle'],
                "author": feed['noteCard']['user']['nickname'],
                "image_urls": [feed['noteCard']['cover']['urlDefault']],  # 预览图
                "url": f"https://www.xiaohongshu.com/explore/{feed['id']}",
                "search_keyword": keyword,
                "fetch_failed": True
            }
            all_notes.append(note)
```

### 实际执行示例

```python
# 关键词1: 搜索 + 即时获取详情
result1 = mcp_xiaohongshu-mcp_search_feeds(
    keyword="深圳元旦跨年",
    filters={"sort_by": "综合", "publish_time": "不限"}
)

# 对 result1 中的每条笔记立即获取详情
FOR feed in result1['feeds'][:5]:
    detail = mcp_xiaohongshu-mcp_get_feed_detail(
        feed_id=feed['id'],
        xsec_token=feed['xsecToken']
    )
    # 保存图片 URL（不下载）
    note = extract_note_data(detail)
    all_notes.append(note)
    wait(2)

# 关键词2: 搜索 + 即时获取详情
result2 = mcp_xiaohongshu-mcp_search_feeds(keyword="深圳情侣旅游")
FOR feed in result2['feeds'][:5]:
    detail = mcp_xiaohongshu-mcp_get_feed_detail(
        feed_id=feed['id'],
        xsec_token=feed['xsecToken']
    )
    note = extract_note_data(detail)
    all_notes.append(note)
    wait(2)
```

> [!TIP]
> **为什么立即获取详情**：`xsec_token` 是动态令牌，有时效性。搜索后立即使用可降低过期风险。

---

## 2.3 数据持久化（仅 JSON，不下载图片）

> [!IMPORTANT]
> **V4 优化**：不下载图片到本地，仅保存在线 URL。
> - ✅ 优点：速度快，无需手动确认，节省存储空间
> - ⚠️ 依赖：发布时需要图片 URL 可访问（xiaohongshu-mcp 支持）

**输出格式**：
```markdown
## 🔍 搜索结果

**搜索统计**：
- 关键词数量：[N] 个
- 笔记总数：[M] 篇
- 详情获取成功：[M1] 篇
- 详情获取失败(降级)：[M2] 篇

**笔记列表**：
| # | 标题 | 作者 | 点赞 | 图片数 |
|---|------|------|------|--------|
| 1 | [...] | [...] | [...] | [...] |
```

### 数据持久化

```python
# 保存笔记详情（含在线图片 URL）到 JSON
save_json(
    filepath=f"{DATA_DIR}/notes_detail.json",
    data={
        "keywords": keywords,
        "notes": all_notes,
        "stats": {
            "total": len(all_notes),
            "detail_success": len([n for n in all_notes if not n.get("fetch_failed")]),
            "total_images": sum(len(n.get("image_urls", [])) for n in all_notes)
        }
    }
)
```

**JSON 结构示例**：
```json
{
  "keywords": ["深圳元旦美食", "深圳情侣餐厅"],
  "notes": [
    {
      "noteId": "68f89d9a0000000007035439",
      "xsec_token": "AB2GqbX_2TQbB38ueCPv_...",
      "title": "只需3种食材，解锁下饭硬菜",
      "content": "#深秋三件套 #豆腐抱蛋 ...",
      "author": "日食记",
      "image_urls": [
        "http://sns-webpic-qc.xhscdn.com/202512272338/4362b9751745b739ff98370e515c5dc3/...",
        "http://sns-webpic-qc.xhscdn.com/..."
      ],
      "likes": "49.4万",
      "comments": "7581",
      "collects": "20.5万",
      "url": "https://www.xiaohongshu.com/explore/68f89d9a0000000007035439",
      "search_keyword": "深圳元旦美食",
      "fetch_failed": false
    }
  ],
  "stats": {
    "total": 10,
    "detail_success": 9,
    "total_images": 35
  }
}
```

---

## 2.4 降级策略

| 异常场景 | 处理方式 |
|---------|---------|
| `get_feed_detail` 超时 | 使用 `search_feeds` 返回的预览数据（title + cover） |
| `xsec_token` 过期 | 重新搜索获取新 token |
| 某关键词无结果 | 跳过并记录日志 |
| MCP 登录失效 | 提示用户重新登录 |

---

## 2.5 搜索筛选器优化

根据复杂度动态调整筛选器：

| 复杂度 | sort_by | publish_time | 说明 |
|--------|---------|--------------|------|
| 低 | 综合 | 不限 | 获取综合质量高的内容 |
| 中 | 最多点赞 | 不限 | 优先热门内容 |
| 高 | 综合 | 半年内 | 平衡时效性和质量 |

**实现示例**：
```python
filters = {
    "sort_by": "综合" if complexity in ["低", "高"] else "最多点赞",
    "publish_time": "半年内" if complexity == "高" else "不限",
    "note_type": "不限",  # V4 主要支持图文
    "location": "不限",
    "search_scope": "不限"
}
```

---

*V4.0 更新时间：2025-12-28*
