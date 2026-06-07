---
name: skill-orchestrator
description: |
  Skill 体系智能路由器。根据用户意图自动匹配最优 skill，编排多 skill 流水线，提供"该用哪个 skill"的决策。
  
  适用场景:
  - 用户描述了任务但不确定用哪个 skill
  - 复杂任务需要多个 skill 协同（先A后B再C）
  - 用户说"帮我做X"但 X 可能对应多个 skill
  - 新任务首次出现，需要路由决策
  
  不适用场景:
  - 用户已明确指定 skill 名称 → 直接使用指定 skill，不干预
  - 简单问答/闲聊 → 不需要 skill
  - 单步操作（读文件/查命令） → 不需要 skill
  
  触发词: 该用哪个skill, skill路由, 选skill, 技能选择, 用什么skill,
           技能匹配, skill pipeline, 技能流水线, 帮我选skill, 哪个技能
  
  边界: skill-orchestrator 只做路由决策（返回 skill 名+理由），不执行任务本身。
        skill-creator 创建新 skill，skill-review 评测质量，skill-orchestrator 路由调用。
  
  正例:
  - "我想写一篇深度文章，该用哪些skill？" → 触发
  - "帮我做内容质量优化全流程" → 触发（需要多 skill 流水线）
  - "这个任务太复杂了，帮我规划用哪些技能" → 触发
  
  反例:
  - "用 dbs-content 诊断这篇文章" → 不触发（用户已指定 skill）
  - "帮我写一段代码" → 不触发（明确的技术任务，直接用 coding skill）
  - "今天天气怎么样" → 不触发（闲聊）
---

# Skill Orchestrator — Skill 体系智能路由器 v1.0

## 核心原则
1. **不执行，只路由**: 返回 skill 名称+调用顺序+理由，不代替目标 skill 执行
2. **用户指定优先**: 用户明确命名某个 skill → 跳过路由，直接使用
3. **只读不写**: 不修改任何 skill 文件
4. **降级兜底**: 无匹配 skill → 返回最接近的 2-3 个候选 + 建议"考虑创建新 skill"

## 路由流程

### Step 1: 解析用户意图
   - 提取: 核心动词(写/改/评/查/发/同步/创建) + 目标对象(文章/skill/代码/知识库/社交媒体) + 约束(平台/格式/质量要求)
   - 输出: 1 句意图摘要

### Step 2: Skill 注册表扫描
   - 扫描路径(按优先级):
     1. `C:\Users\董辉\.agents\skills\*\SKILL.md`
     2. `C:\Users\董辉\.codex\skills\*\SKILL.md`
     3. `D:\_ai\skills\*\SKILL.md`
   - 对每个 skill: 只读 frontmatter(name + description)，不读 body
   - 提取: name + 触发词(从 description 中解析触发词列表) + 适用/不适用场景

### Step 3: 意图-技能匹配
   匹配算法(按优先级):
   a. **精确匹配**: 用户意图关键词 = skill 的触发词 → 置信度 100%
   b. **语义匹配**: 用户意图关键词 ∩ skill description 中的适用场景 → 置信度 70-90%
   c. **模糊匹配**: 用户意图动词+对象在 skill 适用场景中出现 → 置信度 40-70%
   d. **无匹配**: 置信度 <40% → 返回最接近候选

### Step 4: 流水线编排(仅当多 skill 时)
   已知高频流水线(硬编码，根据你的实际工作流):

   | 任务类型 | 流水线 | 
   |---------|--------|
   | 深度内容创作 | dbs-content-system → dbs-content → khazix-writer → humanizer-zh → compile-and-verify |
   | 内容质量优化 | dbs-content → humanizer-zh → compile-and-verify |
   | 小红书发布 | khazix-writer → baoyu-xhs-images → baoyu-post-to-wechat |
   | 知识库维护 | knowledge-base-health → neat-freak |
   | Skill 优化 | skill-review → skill-creator |
   | 代码审查 | preflight-reviewer → security-review → diff-reviewer |
   | 多平台分发 | khazix-writer → crosspost → baoyu-post-to-wechat/weibo/x |
   | 飞书同步 | [内容创作流水线] → feishu |
   | 研究任务 | deep-research → exa-search → compile-and-verify |

   对于不在上述列表的复杂任务: 分析意图 → 拆分子任务 → 为每个子任务匹配 skill → 输出 linear pipeline

### Step 5: 输出路由决策
   严格按以下格式输出:

```markdown
## 路由决策

**意图**: [1句摘要]

**推荐 Skill**:
| 顺序 | Skill | 置信度 | 作用 |
|------|-------|--------|------|
| 1 | skill-A | XX% | [一句话] |
| 2 | skill-B | XX% | [一句话] |

**替代方案**(如有):
| Skill | 置信度 | 适用条件 |
|-------|--------|---------|

**不推荐**:
| Skill | 原因 |
|-------|------|
```

## 失败模式与兜底
| 失败 | 兜底 |
|------|------|
| 完全无匹配(置信度<40%) | 输出"未找到匹配 skill。最接近候选: [2-3个]。建议: ①检查 skill 注册表 ②考虑用 skill-creator 创建新 skill" |
| 多 skill 冲突(≥3 个候选置信度接近) | 列出所有候选+各自的适用的具体条件，让用户选择 |
| 注册表扫描失败(目录不存在) | 输出"skill 目录不可达: [path]，请检查路径" |
| 单个 skill 的 SKILL.md 读取失败 | 跳过该 skill，继续扫描 |

## 约束
- 不加载 SKILL.md body(节省 token)——只在匹配阶段读 frontmatter
- 流水线长度 ≤5 个 skill(过长=意图需要拆分)
- 输出必须含"不推荐"列表(帮助用户避免误用)

---
## 版本
- v1.1 | 2026-06-07 | R2: 使用统计+自适应学习
- v1.0 | 2026-06-07 | 初始版本(终极形态) | 原因: 108+skill体系缺少统一路由器，每次任务浪费token在"猜用哪个skill"
## R2增强: 使用统计 + 自适应学习

### 使用频率追踪
每次被调用时，在 `D:\KnowledgeBase\_logs\ledger\SKILL_USAGE.md` 追加一行:
```
[YYYY-MM-DD HH:MM] skill-orchestrator → 路由结果: [skill-A → skill-B] | 置信度: XX%
```
定期分析：哪些流水线最常被触发？哪些 skill 从未被路由到(可能是死 skill)？

### 自适应匹配优化
- 如果某 skill 在相似意图下被用户手动覆盖 ≥3 次 → 更新该意图的默认路由
- 如果某流水线连续 5 次置信度 >90% → 标记为"已验证流水线"，下次跳过匹配直接使用
- 如果某 skill 被路由但用户随后立即切换到另一个 skill → 记录为"路由错误"，下次降权