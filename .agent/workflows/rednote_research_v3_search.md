---
description: RedNote Research V3.0 - Step 2 搜索模块
---

# Step 2: 搜索阶段 (Search Phase)

**目标**：通过 rednote-MCP 高效收集小红书笔记的**完整内容、图片和视频**

> [!IMPORTANT]
> **核心策略**：搜索后立即获取详情并下载图片，确保数据完整性优先于效率。

---

## 2.1 MCP 连接检查

```python
# 首先验证 rednote-MCP 可用
mcp_rednote-MCP_login()  # 检查登录状态
```

---

## 2.2 搜索 + 详情获取 (串行执行)

> [!CAUTION]
> **关键改进**：每次搜索后，立即对搜索结果中的每条笔记调用 `get_note_content`，避免 session 失效导致的后续获取失败。

### 执行流程

```python
all_notes = []
request_interval = 2  # 秒，请求间隔

FOR each keyword in keywords:
    # 1. 执行搜索
    search_results = mcp_rednote-MCP_search_notes(keywords=keyword, limit=N)
    wait(request_interval)
    
    # 2. 立即获取每条笔记的详情 (包含图片)
    FOR note_preview in search_results:
        TRY:
            detail = mcp_rednote-MCP_get_note_content(url=note_preview.url)
            
            # 3. 合并数据
            note = {
                "title": detail.title,
                "content": detail.content,
                "author": detail.author,
                "tags": detail.tags,
                "image_urls": detail.imgs,    # 完整图片 URL 列表
                "videos": detail.videos,
                "url": note_preview.url,
                "likes": detail.likes,
                "comments": detail.comments,
                "search_keyword": keyword
            }
            
            # 4. 下载图片到本地
            download_note_images(note, len(all_notes))
            
            all_notes.append(note)
            wait(request_interval)
            
        EXCEPT as e:
            LOG_WARNING(f"获取详情失败: {note_preview.url}, 错误: {e}")
            # 降级：使用搜索结果的预览数据
            note = {
                "title": note_preview.title,
                "content": note_preview.content,  # 搜索结果中的正文
                "author": note_preview.author,
                "image_urls": [],  # 无图片
                "videos": [],
                "url": note_preview.url,
                "search_keyword": keyword,
                "fetch_failed": True
            }
            all_notes.append(note)
    
```

### 实际执行示例

```python
# 关键词1: 搜索 + 即时获取详情
result1 = mcp_rednote-MCP_search_notes(keywords="深圳元旦跨年", limit=5)
# 对 result1 中的每条笔记立即获取详情
FOR note in result1:
    detail = mcp_rednote-MCP_get_note_content(url=note.url)
    download_images(detail.imgs)
    wait(2)

# 关键词2: 搜索 + 即时获取详情
result2 = mcp_rednote-MCP_search_notes(keywords="深圳情侣旅游", limit=5)
# 对 result2 中的每条笔记立即获取详情
FOR note in result2:
    detail = mcp_rednote-MCP_get_note_content(url=note.url)
    download_images(detail.imgs)
    wait(2)
```

> [!TIP]
> **为什么立即获取详情**：`get_note_content` 依赖有效的 session/cookie，连续调用可降低超时风险。分离搜索和详情获取会导致 session 过期。

---

## 2.3 图片下载逻辑

```python
def download_note_images(note, note_index):
    """
    下载笔记的所有图片到本地 IMAGES_DIR
    """
    note["local_images"] = []
    
    FOR img_idx, img_url in enumerate(note["image_urls"]):
        # 生成本地文件名: note_001_img_01.webp
        filename = f"note_{note_index:03d}_img_{img_idx:02d}.webp"
        local_path = f"{IMAGES_DIR}/{filename}"
        
        TRY:
            # 使用 run_command 下载图片 (curl)
            run_command(
                Cwd=OUTPUT_DIR,
                CommandLine=f'curl -s -o "{local_path}" "{img_url}"',
                SafeToAutoRun=True,
                WaitMsBeforeAsync=3000
            )
            
            # 记录下载结果
            note["local_images"].append({
                "url": img_url,
                "local_path": local_path,
                "filename": filename
            })
            
        EXCEPT as e:
            LOG_WARNING(f"图片下载失败: {img_url}, 错误: {e}")
```

> [!TIP]
> `get_note_content` 返回的 `imgs` 字段包含完整的 CDN 图片链接（格式：`https://sns-webpic-qc.xhscdn.com/...`），可直接用 curl 下载。

---

## 2.4 视频处理 (可选)

```python
IF note["videos"] is not empty:
    FOR video_idx, video_url in enumerate(note["videos"]):
        filename = f"note_{note_index:03d}_video_{video_idx:02d}.mp4"
        local_path = f"{IMAGES_DIR}/{filename}"
        
        run_command(
            Cwd=OUTPUT_DIR,
            CommandLine=f'curl -s -o "{local_path}" "{video_url}"',
            SafeToAutoRun=True,
            WaitMsBeforeAsync=10000  # 视频较大，等待更长时间
        )
        
        note["local_videos"].append({"url": video_url, "local_path": local_path})
```

---

## 2.5 输出格式

> [!IMPORTANT]
> 搜索完成后，需将结果保存到 `{DATA_DIR}/search_results.json`。

```markdown
## 🔍 搜索结果

**搜索统计**：
- 关键词数量：[N] 个
- 笔记总数：[M] 篇
- 详情获取成功：[M1] 篇
- 详情获取失败(降级)：[M2] 篇

**图片下载统计**：
- 图片总数：[I1] 张
- 下载成功：[I2] 张
- 保存位置：`{IMAGES_DIR}/`

**笔记列表**：
| # | 标题 | 作者 | 点赞 | 本地图片数 |
|---|------|------|------|------------|
| 1 | [...] | [...] | [...] | [...] |
```

### 数据持久化

```python
# 保存笔记详情（含本地图片路径）到 JSON
save_json(
    filepath=f"{DATA_DIR}/notes_detail.json",
    data={
        "keywords": keywords,
        "notes": all_notes,
        "stats": {
            "total": len(all_notes),
            "detail_success": len([n for n in all_notes if not n.get("fetch_failed")]),
            "images_downloaded": sum(len(n.get("local_images", [])) for n in all_notes)
        }
    }
)
```

---

## 2.6 降级策略

| 异常场景 | 处理方式 |
|---------|---------|
| `get_note_content` 超时 | 使用 `search_notes` 返回的正文（无图片） |
| 图片下载失败 | 记录失败 URL，继续处理其他图片 |
| 某关键词无结果 | 跳过并记录日志 |
| MCP 登录失败 | 提示用户重新登录 |