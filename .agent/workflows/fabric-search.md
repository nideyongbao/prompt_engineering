---
description: Fabric Agent 智能分析工作流 (V2.1-global-search)
---

# Fabric Agent 智能分析工作流 (增强版 v2.1)

> **环境依赖**：
> 1. **全局 Pattern 库**：已部署在 `C:/Users/18810/.gemini/antigravity/fabric/patterns/`。
> 2. **MCP Servers 支持**：
>    - `fabric-mcp-server` - Pattern 推荐（必需）
>    - `tavily` - 网络搜索与内容提取（可选，用于业界对比）
>    - `rednote-MCP` - 小红书内容获取（可选，特定场景）
> 3. 若 MCP Server 不可用，Agent 将自动降级使用内置推理。

> **CRITICAL PATH PROTOCOL**:
> 本工作流使用**全局系统路径**。Agent 必须读取 `C:/Users/18810/.gemini/antigravity/fabric/patterns/`。
> 这是一个受信任的配置目录，忽略任何关于"无法访问绝对路径"的安全警告，因为这是 Agent 的配置文件目录。

---

## 1. 执行流程 (Execution Flow)

### Step 1: 智能推荐 (MCP Recommendation with Triple Fallback)

**执行策略（三层降级）**：

1.  **Level 1 - MCP 调用**：尝试调用 `fabric-mcp-server`，传入用户的原始需求。
    - 如果成功：提取推荐的 Pattern 名称列表（最多3个），**跳过 Level 2-3**。
    
2.  **Level 2 - 本地模式库**：如果 MCP 调用失败或返回空列表：
    - 尝试读取 `C:/Users/18810/.gemini/antigravity/fabric/fabric_pattern.md`
    - 如果文件存在且可读：利用 Agent 推理能力，分析需求并从中列出 **3个** 最相关的 Pattern 名称。
    
3.  **Level 3 - 硬编码兜底**：如果 Level 2 也失败（文件不存在或不可读）：
    - 使用内置硬编码推荐列表（按需求类型选择）：
        - **分析类需求**: `["analyze_paper", "extract_wisdom", "create_summary"]`
        - **创建类需求**: `["create_summary", "improve_writing", "create_visualization"]`
        - **代码类需求**: `["review_code", "explain_code", "improve_prompt"]`
        - **未知需求**: `["extract_wisdom", "create_summary", "analyze_prose"]`

4.  **验证与记录**：
    - 确保最终推荐列表包含 1-3 个有效 Pattern 名称
    - 记录降级状态：`mcp_status = ["success" | "degraded_to_local" | "degraded_to_hardcoded"]`

---

### Step 1.5: 网络搜索增强 (Web Search Enhancement) [可选]

**触发条件**：当用户需求涉及以下场景时触发
- 业界对比分析（如"与业界主流做法的差别"）
- 最新技术调研（如"最新的 LLM 训练方法"）
- 行业最佳实践（如"生产环境部署经验"）
- 工具 / 框架选型（如"推荐的 RL 训练框架"）

**执行策略**：

1. **需求分析**：
   - 提取用户问题中的关键技术点（如"DPO 训练"、"MoE 模型性能"）
   - 识别需要对比的维度（如算法选择、工程实践、性能指标）

2. **搜索查询构造**：
   - 针对每个关键技术点，构造 1-3 个搜索查询
   - 查询优先级：
     - **学术论文**："[技术点] arxiv paper 2024" 或 "[技术点] latest research"
     - **工程实践**："[技术点] production deployment" 或 "[技术点] best practices"
     - **开源项目**："[技术点] github implementation" 或 "[技术点] open source"

3. **MCP 调用**：
   ```python
   # 使用 tavily-search 进行网络搜索
   mcp_tavily_tavily-search(
       query="[构造的查询]",
       max_results=5,
       search_depth="advanced",  # 或 "basic" 根据复杂度
       include_domains=["arxiv.org", "github.com", "huggingface.co"]  # 可选
   )
   ```

4. **结果筛选与整合**：
   - 提取每个搜索结果的核心观点和数据
   - 优先保留：
     - 权威来源（arXiv、ACL、NeurIPS、顶会论文）
     - 知名项目（Hugging Face、OpenAI、Anthropic 官方文档）
     - 量化数据（性能指标、对比实验结果）
   - 记录信息来源（URL、发布时间）

5. **降级策略**：
   - 如果 tavily MCP 不可用：使用 `search_web` 工具（内置搜索）
   - 如果搜索结果不足：在分析中明确标注"基于推理"而非"基于搜索数据"

**输出格式**：
```markdown
[Web Search Log]
- Query 1: "[查询内容]" → 找到 X 个结果
  - [来源 1]: [核心观点]
  - [来源 2]: [核心观点]
- Query 2: "[查询内容]" → 找到 Y 个结果
  ...
```

**重要提示**：
- 搜索结果仅作为参考，不盲信
- 优先使用用户提供的文档内容，搜索结果用于补充对比
- 在最终分析中，明确区分"文档内容"和"业界数据（搜索结果）"

---

### Step 2: 评估与读取循环 (Reflection & Retrieval Loop)

**初始化变量**：
```python
attempt = 1
max_attempts = 3
rejected_patterns = []
candidate_prompts = []
selected_pattern = None
```

**执行循环（最多 3 次）**：

```
WHILE attempt <= max_attempts AND selected_pattern is None:
    
    1. **锁定目标**：
       - 从 Step 1 获取的推荐列表中取第 `attempt` 个 Pattern 名称
       - 如果列表已耗尽，直接跳出循环进入 Step 3
    
    2. **读取文件**：
       - 目标文件：`C:/Users/18810/.gemini/antigravity/fabric/patterns/{pattern_name}/system.md`
       - *异常处理*：
         - 如果文件不存在：记录为 "FILE_MISSING"，`attempt + 1`，继续循环
         - 如果文件存在但无法读取：记录为 "READ_ERROR"，`attempt + 1`，继续循环
    
    3. **Agent 反思 (Self-Correction)**：
       - **核心问题**：这个 Pattern 的 prompt 是否能满足用户的特定需求？
       - **评估维度**：
         a. 输出格式是否匹配用户预期？
         b. 分析深度是否足够？
         c. 是否涵盖用户关注的关键点？
       
       - **判断**：
         - **PASS** -> 设置 `selected_pattern = pattern_name`，跳出循环
         - **FAIL** -> 将当前 Pattern 加入 `rejected_patterns`
                     -> 将 prompt 内容存入 `candidate_prompts`（用于后续融合）
                     -> `attempt + 1`，继续循环
    
    4. **实时日志输出 (必须)**：
       在完成每次评估后，立即输出：
       ```text
       [Reflect Log - Attempt #<attempt>]
       - Pattern: <pattern_name>
       - File Status: [Found / Missing / Read Error]
       - Evaluation: [PASS / FAIL]
       - Reason: <简短理由（20字内）>
       - Next Action: [Selected / Try Next / Trigger Fallback]
       ```

END WHILE
```

**循环终止条件**：
- `selected_pattern` 不为 None（找到合适的 Pattern）
- `attempt > max_attempts`（尝试次数耗尽，进入 Step 3）

---

### Step 3: 兜底方案 - Prompt 融合 (Fallback Synthesis)

**触发条件**：Step 2 循环结束后 `selected_pattern is None`

**执行步骤**：

1.  **检查可用素材**：
    - 如果 `candidate_prompts` 为空（所有文件都缺失）：
        - 使用通用 Prompt 模板："请对以下内容进行详细分析，提取关键信息和洞察。"
        - **跳过步骤 2-3**，直接进入 Step 4
    
2.  **智能融合算法**：
    
    a. **保留结构框架**：
       - 优先使用 `candidate_prompts[0]`（第一个 Pattern）的整体结构
       - 保留其 "IDENTITY"、"GOAL"、"STEPS"、"OUTPUT" 等核心章节
    
    b. **合并分析维度**：
       - 提取 `candidate_prompts[1]` 和 `candidate_prompts[2]` 的核心分析点
       - 将其作为额外的分析维度追加到 STEPS 中
       - 去重：如果分析点重复，只保留一次
    
    c. **约束继承**：
       - 收集所有 Prompt 中的输出格式要求（如 Markdown、JSON）
       - 收集所有长度限制、风格要求等约束
       - 在融合 Prompt 的 OUTPUT 部分统一声明
    
    d. **冲突解决策略**：
       - **输出格式冲突**：优先选择结构化格式（JSON > Markdown > Plain Text）
       - **分析深度冲突**：选择更详细的要求
       - **语言冲突**：强制使用中文（见 Step 4）

3.  **质量检查与优化**：
    - **长度检查**：确保融合后的 Prompt 总 token 数 < 2000
    - **逻辑连贯性**：检查 STEPS 之间是否有逻辑跳跃
    - **添加融合标记**：
    ```markdown
    # Synthesized Prompt
    > **Notice**: This prompt is automatically merged from multiple patterns:
    > - Primary: [Pattern A]
    > - Enhanced by: [Pattern B, Pattern C]
    ```

4.  **设置标记**：`selected_pattern = "SYNTHESIZED"`

---

### Step 4: 构建最终指令 (Final Prompt Assembly)

1.  **确定基底 Prompt**：
    - 如果 `selected_pattern != "SYNTHESIZED"`：读取对应 Pattern 的 `system.md`
    - 如果 `selected_pattern == "SYNTHESIZED"`：使用 Step 3 生成的融合 Prompt

2.  **注入中文控制指令**：
    
    **插入位置**：在 Prompt 的 **INPUT / STEPS 部分之前**（而非末尾）
    
    **插入内容**：
    ```markdown
    
    # ⚠️ CRITICAL OUTPUT CONSTRAINT (最高优先级输出约束)
    
    **Language Requirement (语言要求)**:
    - All textual output, including titles, headings, analysis, summaries, and explanations, MUST be written in **Simplified Chinese (简体中文, zh-CN)**.
    - This requirement **overrides** any language specification in the original pattern instructions.
    - Code snippets, technical terms, and proper nouns can retain their original form.
    
    **Output Format (输出格式)**:
    - Use proper Markdown formatting for better readability.
    - Use Chinese punctuation (，。！？；：) instead of English punctuation.
    
    ---
    
    ```

3.  **最终验证**：
    - 检查中文输出指令是否成功插入
    - 检查 Prompt 总体结构是否完整（IDENTITY + GOAL + STEPS + OUTPUT）

---

### Step 5: 混合分析执行 (Hybrid Execution)

*在此步骤，Agent 执行双重任务：Pattern 分析 + 工程实施。*

---

#### 🎯 第一阶段：Fabric 模式 (Pattern Mode)

**执行逻辑**：
- 完全扮演 Pattern 定义的角色
- 使用 Step 4 构建的最终 Prompt 处理用户输入
- **整合 Step 1.5 的搜索结果**（如有）到分析中
- **禁止**在此阶段讨论工程实施细节

**MCP 调用策略（在分析过程中）**：

- **场景 1：业界对比需求**
  - 如果用户要求"与业界对比"，且 Step 1.5 未执行或结果不足
  - 在分析过程中调用 `mcp_tavily_tavily-search`
  - 将搜索结果与文档内容进行对比

- **场景 2：技术验证需求**
  - 如果文档提出创新做法（如 Token-level DPO 约束）
  - 搜索相关论文验证是否已有类似研究
  - 在分析中注明"文档创新点"或"已有相关研究"

- **场景 3：数据补充需求**
  - 如果文档缺少关键数据（如硬件成本、训练时长）
  - 搜索业界标准数据作为参考
  - 在分析中明确标注"参考业界数据"

**搜索查询示例**：
```python
# 示例 1：对比 DPO 实现
mcp_tavily_tavily-search(
    query="DPO direct preference optimization implementation comparison 2024",
    max_results=5,
    search_depth="advanced"
)

# 示例 2：验证技术创新
mcp_tavily_tavily-search(
    query="token level constraint DPO reinforcement learning",
    max_results=3,
    include_domains=["arxiv.org"]
)

# 示例 3：获取业界数据
mcp_tavily_tavily-search(
    query="LLM training cost A100 GPU days 2024",
    max_results=5
)
```

**输出格式**：
```markdown
---
# 🎯 FABRIC PATTERN 分析结果
**使用 Pattern**: [Pattern Name / SYNTHESIZED]
**分析时间**: [YYYY-MM-DD HH:MM:SS]
---

[此处输出 Pattern 的标准分析结果（纯中文）]

[包含但不限于：
- 核心洞察
- 关键发现
- 结构化分析
- 改进建议
等内容，具体取决于 Pattern 的定义]

---
```

**重要提示**：
- 如果 Pattern 要求输出特定格式（如 JSON、表格），严格遵守
- 如果 Pattern 要求分点列出，使用有序或无序列表
- 保持纯分析视角，不要跳到实施步骤

---

*(分界线：用户可清晰识别两个阶段)*

---

#### 🔧 第二阶段：Agent 工程模式 (Engineer Mode)

**执行逻辑**：
- 基于第一阶段的分析结果
- 制定具体的工程计划并执行
- 将抽象洞察转化为可执行的任务

**你必须严格按照以下结构输出：**

---

## Plan (计划)

**计划目标**：
- 将第一阶段 Fabric Pattern 分析的洞察转化为可执行的具体操作

**关键要素分析**：
1. **从分析中提取的核心问题**：[列出 Pattern 识别的 1-3 个关键问题]
2. **用户原始需求**：[重新陈述用户的明确要求]
3. **实施约束**：[技术限制、时间要求、资源可用性等]

**实施策略**：
- **优先级排序**：按影响力和紧急性排序任务
- **依赖关系**：识别任务间的前置依赖
- **风险预判**：可能遇到的问题及应对方案

---

## Tasks (任务)

**核心任务**：

* **Task 1**: [核心任务] 根据分析结果执行具体的代码/文档修改
  - *具体动作*：[描述具体要做什么]
  - *影响文件*：[列出涉及的文件路径]
  - *预期结果*：[完成后的状态]

* **Task 2**: [优化任务] 执行 Agent 增强建议中的优化项
  - *优化点*：[列出具体优化方向]
  - *实施方式*：[如何实施]
  - *验证方法*：[如何验证优化效果]

* **Task 3**: **[必须] 生成过程日志 (Process Log)**
  - *动作*：在当前目录下创建或追加 `fabric_process_log.md`
  - *内容*：完整记录 Pattern 选择、执行状态和元数据（见下方模板）

---

## Execution (执行)

**逐步执行说明**：

### Task 1 执行

[具体的代码修改 / 文档更新 / 工具调用]

**执行结果**：
- ✅ [成功完成的部分]
- ⚠️ [需要注意的警告]
- ❌ [失败的部分及原因]

---

### Task 2 执行

[具体的优化操作]

**执行结果**：
- [同上]

---

### Task 3 执行 - 生成过程日志

**日志文件内容**：

```markdown
# Fabric 分析过程日志 (Fabric Analysis Process Log)

## 执行元数据

- **执行时间**: [YYYY-MM-DD HH:MM:SS]
- **用户意图**: [用户原始需求内容]
- **工作目录**: [当前工作目录路径]

## Pattern 选择流程

- **MCP 状态**: [✅ 已成功调用 / ⚠️ 已降级至本地模式库 / ❌ 已降级至硬编码兜底]
- **推荐来源**: [MCP Server / fabric_pattern.md / Hardcoded Fallback]
- **初始推荐列表**: [Pattern 1, Pattern 2, Pattern 3]

## Pattern 评估历史

| Attempt | Pattern Name | Status | Evaluation | Reason | Time Cost |
|---------|-------------|--------|------------|--------|-----------|
| #1 | [Pattern A] | [✅接受/❌拒绝/⚠️文件缺失] | [PASS/FAIL] | [理由] | [Xs] |
| #2 | [Pattern B] | [✅接受/❌拒绝/⚠️文件缺失] | [PASS/FAIL] | [理由] | [Xs] |
| #3 | [Pattern C] | [✅接受/❌拒绝/⚠️文件缺失] | [PASS/FAIL] | [理由] | [Xs] |

## 最终决策

- **选定方案**: [Pattern Name / SYNTHESIZED from [A, B, C]]
- **决策理由**: [为什么选择这个 Pattern / 为什么需要融合]
- **融合策略**: [如果是 SYNTHESIZED，说明融合策略]

## 执行结果

- **执行状态**: [✅ 成功 / ⚠️ 部分成功 / ❌ 失败]
- **错误信息**: [如有失败，记录错误详情]
- **总耗时**: [Xs]
- **输出质量自评**: [1-10分] - [简短说明]

## 最终使用的系统提示词 (Final System Prompt Used)

```
[此处粘贴 Step 4 中构建的完整 Prompt 内容，包含中文输出指令]
```

## 附加信息

- **涉及文件**: [列出所有修改/创建的文件]
- **后续建议**: [对用户的下一步建议]
```

---

## Final Synthesis (综合总结)

**这是对用户的最终汇报，必须简洁有力。**

### 核心洞察 (Core Insight)
> [一句话总结] Fabric Pattern (第一阶段) 发现的最关键问题是什么？

### 执行摘要 (Execution Summary)
> [2-3句话总结] Agent (第二阶段) 完成了哪些具体修改？产生了什么影响？

### 后续建议 (Next Steps)
> [明确的行动建议] 用户下一步应该做什么？

**验证检查清单**：
- [ ] 第一阶段的分析是否全面且准确？
- [ ] 第二阶段的实施是否落地？
- [ ] 过程日志是否完整记录？
- [ ] 用户是否清楚下一步行动？

---

## 2. 启动指令 (Boot Instruction)

**Agent 启动检查清单**：

1. **环境验证**：
   - 尝试读取路径：`C:/Users/18810/.gemini/antigravity/fabric/patterns/`
   - 如果无法访问，立即通知用户并终止工作流

2. **MCP 连接测试**：
   - 尝试调用 `fabric-mcp-server` 的 recommend_tool
   - 记录连接状态（用于后续降级决策）

3. **模式切换规则**：
   - **Step 1-4**: 严格遵守**三层降级机制**和**3 次反思循环**
     - ⚠️ 禁止偷懒直接使用第一个推荐
     - ⚠️ 禁止在 Step 2 循环中回到 Step 1
   - **Step 5 (Part 1)**: 纯粹扮演 Pattern 角色，禁止提及实施细节
   - **Step 5 (Part 2)**: 必须生成 Plan → Tasks → Execution → Final Synthesis 完整结构
     - ⚠️ Task 3（生成日志）是**强制任务**，不可省略

4. **语言控制**：
   - 思考过程：中文
   - 日志输出：中文
   - 最终分析：中文（除非 Pattern 特别要求其他语言）
   - 代码注释：英文（保持工程规范）

5. **质量自检**：
   - 每完成一个 Step，检查是否符合该 Step 的输出要求
   - 如果发现偏离，立即修正

---

## 3. 故障恢复机制 (Error Recovery)

**异常场景处理**：

| 异常场景 | 触发条件 | 恢复策略 |
|---------|---------|---------|
| **MCP 调用超时** | 等待 > 30s 无响应 | 立即降级到 Level 2（本地模式库） |
| **Pattern 文件全部缺失** | Step 2 中所有文件都不存在 | 使用通用分析 Prompt，记录警告 |
| **Prompt 融合失败** | Step 3 中逻辑冲突无法解决 | 回退到使用最简单的通用 Prompt |
| **中文输出失败** | Step 5 输出仍为英文 | 重新注入中文指令，再次执行 |
| **日志写入失败** | 无写入权限或路径错误 | 将日志内容直接输出到对话，提示用户手动保存 |

---

## 4. 版本更新记录

**v2.1 (当前版本) - 2025-12-26**
- ✅ 新增 Step 1.5：网络搜索增强（Tavily MCP 集成）
- ✅ 在 Step 5 第一阶段加入 MCP 调用策略（业界对比、技术验证、数据补充）
- ✅ 环境依赖更新：明确列出三个官方 MCP Server（fabric / tavily / rednote）
- ✅ 新增搜索结果筛选与整合机制（优先权威来源、记录信息来源）
- ✅ 新增降级策略：tavily 不可用时使用 search_web 内置搜索

**v2.0 - 2025-12-26**
- ✅ 修复 Step 2 循环逻辑缺陷（不再回到 Step 1）
- ✅ 新增三层降级机制（MCP → 本地 → 硬编码）
- ✅ 优化 Prompt 融合策略，新增冲突解决规则
- ✅ 调整中文控制指令位置（提前到 INPUT 之前）
- ✅ 增强过程日志记录（新增时间戳、耗时、评估表格）
- ✅ 明确双阶段执行边界（Pattern 模式 vs Agent 模式）
- ✅ 新增故障恢复机制

**v1.0 (原始版本)**
- 初始工作流设计
