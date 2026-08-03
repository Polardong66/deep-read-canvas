---
name: deep-read-canvas
description: 微信读书书籍精读分析与PPT生成。调用微信读书API获取书籍数据，结合横纵分析法调研行业实践，生成主题自适应的翻页式HTML PPT演示文稿。当用户要求分析、精读、研究微信读书中的某本书、或要求生成书籍知识图谱和分析报告PPT时使用。
agent_created: true
---

# 微信读书书籍精读分析与PPT生成

## 概述

从微信读书获取指定书籍的完整数据，进行深度分析，最终生成一份风格根据书籍主题自适应、支持翻页交互的HTML PPT演示文稿。报告包含：知识图谱、章节分析、读者评论、行业交叉对比、总结提炼。

## 触发条件

- 用户要求分析/精读/研究微信读书中的某本书
- 用户要求生成某本书的知识图谱或分析报告
- 用户要求将书籍内容制作成PPT演示文稿
- 「帮我分析下书架里的XXX」「精读XXX这本书并做PPT」

## 🔴 反例黑名单 — 禁止做的事

| # | 禁止行为 | 原因 |
|---|---------|------|
| 1 | **使用 `params` 包裹 API 业务参数** | curl body 必须是 `{"api_name":"...", "bookId":"...", "skill_version":"..."}` 平铺格式 |
| 2 | **编造书籍章节、评分或划线数据** | 所有数据必须来自 API 真实返回，搜不到标注「信息暂缺」 |
| 3 | **修改划线原文或点评内容** | 只能截取长度，不得改写原文措辞 |
| 4 | **行业交叉分析凭空编造竞品信息** | 必须通过 WebSearch/WebFetch 获取真实市场数据 |
| 5 | **Write 大文件时不验证完整性** | 每次写入后用 `tail -3` 检查末尾是否为 `</html>`，确保无截断 |
| 6 | **在 API 回包出现 `upgrade_info` 时继续执行** | 必须暂停并按提示升级 skill_version 后再重试 |
| 7 | **跳过 WEREAD_API_KEY 检测直接调 API** | 先检测环境变量和 MEMORY.md，均无则提示用户设置 |
| 8 | **对评分字段不转换直接展示** | `newRating=846` 必须转为 84.6 分；`star=100` 转为 5 星 |
| 9 | **忽略 `content` 为空的情况** | 点评同时检查 `content` 和 `htmlContent`，回退取值 |
| 10 | **使用日文/西文引号或弯引号** | 全文统一使用中文直角引号「」和『』 |
| 11 | **正文字号 < 22px** | 幻灯片是给屏幕观看的，正文最小 22px，卡片最小 16px |
| 12 | **每页塞入超过 5 个信息块** | 1 核心 + 3-4 辅助 + 1 视觉，超过就拆页 |
| 13 | **所有页面用同一种布局** | 轮换双栏/卡片/引用/表格等类型，保持视觉节奏 |
| 14 | **嵌套元素圆角不协调** | 外层 radius = 内层 radius + 元素间 padding |

## 🛑 CHECKPOINT · 关键决策点

执行到以下节点时**暂停，等待用户确认**后再继续：

| 检查点 | 触发时机 | 确认内容 |
|--------|---------|---------|
| **CHECKPOINT-1** | 数据采集完成后 | 展示书籍元数据（书名、作者、评分、章节数），确认分析范围 |
| **CHECKPOINT-2** | 分析结构设计完成后 | 展示 PPT 页数和各页内容摘要，确认结构是否调整 |
| **CHECKPOINT-3** | HTML PPT 生成完成后（showcase 或完整版） | 展示文件路径 + 明确告知当前页数。若为 2 页 showcase → 说清「这是样板，剩余 N 页待确认后生成」，等待用户验收；若为完整版 → 展示页数统计 |

## 前置条件：微信读书API配置

微信读书API通过Agent Gateway调用。首次使用前需按以下步骤配置：

**API Key 获取与配置**

1. 微信读书API Key 格式为 `wrk-xxxxxxxx`，在微信读书开放平台获取
2. 设置为环境变量：`export WEREAD_API_KEY=<your-api-key>`
3. 将API Key持久化保存到 `~/.workbuddy/MEMORY.md`，避免每次询问

**检测API Key是否已配置：**

```bash
echo ${#WEREAD_API_KEY}
```

若输出为 `0`，依次检查：
1. 环境变量：`echo $WEREAD_API_KEY`
2. 持久化文件：`grep WEREAD_API_KEY ~/.workbuddy/MEMORY.md`
3. 均无 → 🔴 提示用户设置，流程暂停

**API调用规范**

- 统一入口：`POST https://i.weread.qq.com/api/agent/gateway`
- 鉴权Header：`Authorization: Bearer $WEREAD_API_KEY`
- Content-Type：`application/json`
- 所有请求body必须包含 `"skill_version": "1.0.4"`
- 业务参数与 `api_name`、`skill_version` 平铺在body顶层，**不要**包在 `params` 里

**请求示例（搜索书籍 → 获取 bookId）：**

```bash
curl -s -X POST "https://i.weread.qq.com/api/agent/gateway" \
  -H "Authorization: Bearer $WEREAD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"api_name":"/store/search","keyword":"书名","count":5,"skill_version":"1.0.4"}'
```

回包中取 `results[0].books[0].bookInfo.bookId` 即为后续接口所需的 bookId。

详细接口文档见 `references/weread-api.md`。

---

## ⚠️ 失败模式与异常处理

| 触发条件 | 一线修复 | 仍失败兜底 |
|---------|---------|-----------|
| `WEREAD_API_KEY` 未设置 + MEMORY.md 无记录 | 提示用户执行 `export WEREAD_API_KEY=<key>` | 终止流程，等待用户提供 Key |
| `/store/search` 返回 0 结果 | 尝试书名简写/去副标题重搜 | 告知用户「未找到该书」，建议手动提供 bookId |
| `/book/chapterinfo` 返回空 chapters | 检查 bookId 是否正确，重试一次 | 降级为仅基于简介和点评的分析模式，跳过章节分析 |
| `/book/bestbookmarks` 返回 0 条 | 跳过划线模块，仅用点评填充 | 在 PPT 中标注「本书暂无热门划线」 |
| `/review/list` 返回 0 条 | 跳过点评模块 | 在 PPT 中标注「本书暂无公开点评」 |
| curl 请求超时（>30s） | 重试一次，curl 加 `--connect-timeout 15 --max-time 30` | 该接口数据标注「获取超时」，继续其他接口 |
| 回包 JSON 解析失败 | 保存原始响应到 `/tmp/weread_error_{api}.txt` 供排查 | 该接口标注「数据异常」 |
| 回包出现 `upgrade_info` 字段 | 🔴 立即暂停，按 `upgrade_info.message` 指引升级 `skill_version` | 升级完成后重新执行用户请求 |
| HTML 写入被截断（文件行数不符合预期） | 用 `tail -3` 检查末尾是否为 `</html>` | 用 Edit 工具从截断点重新写入 |
| 单页幻灯片内容超出视口 | 按内容密度规则精简（1核心+3-4辅助），用 `overflow-y: auto` 兜底 | 拆分长页为双页 |

## 执行流程

### 阶段一：数据采集

**输入**：书名（用户提供）

**输出**：`bookId` + 元数据 + 目录 + 热门划线 + 点评的完整数据集

**执行步骤**（接口可并行调用，减少等待时间）：

1. 调用 `/store/search`，keyword=书名，获取 `bookId`
2. 并行调用以下 5 个接口：
   - `/book/info` → 提取 `title, author, cover, intro, newRating, newRatingCount, publishTime`
   - `/book/chapterinfo` → 提取 `chapters[].title, chapters[].level, chapters[].wordCount, chapters[].anchors[]`
   - `/book/bestbookmarks` (chapterUid=0) → 提取 `items[].markText, items[].totalCount, items[].chapterUid`
   - `/review/list` (reviewListType=1, count=10) → 提取 `reviews[].review.review.content/htmlContent, star, author.name`
   - `/book/bookmarklist`（可选，若用户有笔记）→ 提取 `updated[].markText`
3. 保存各接口原始 JSON 到 `/tmp/` 供后续解析

**失败处理**：任一步失败 → 参考「⚠️ 失败模式与异常处理」表对应的 fallback

🔴 **CHECKPOINT-1**：采集完成后展示元数据，等待用户确认分析范围

### 阶段二：信息分析

**输入**：阶段一的全量数据集

**输出**：结构化章节分析内容（每个章节一段，供 PPT 直接使用）

**分析模板**（每个章节/每个Part套用）：

```
【Part X · 篇章名】(含N节)
核心论述：[200-400字摘要，逻辑完整，从问题→论证→结论]
关键划线：[选2-3条划线人数最高的，标注章节来源和人数]
读者点评：[选取最相关推荐点评，标注作者和星级]
可视化建议：[该章节适合的页面类型：双栏/卡片/引用/表格]
```

**注意事项**：
- `newRating=846` → 展示为 84.6 分（除以10）
- `star=100` → 展示为 ★★★★★（除以20）
- 点评 `content` 空 → 回退取 `htmlContent`
- 章节可能有锚点嵌套（如 CHAPTER 5 的 anchors 在 CHAPTER 4 的 uid 下），遍历 `anchors[]` 获取完整结构

🔴 **CHECKPOINT-2**：完成全书结构设计和各页内容摘要后，展示 PPT 页数规划，等待用户确认是否调整

### 阶段三：行业交叉分析

**输入**：阶段二的章节分析结果 + 书籍所属领域

**输出**：纵向时间线数据 + 横向竞品对比表

**执行步骤**：
1. **纵向调研**：搜索「[书籍所属领域] 发展脉络 演变 历史」，构建关键里程碑时间线（5-6 个节点）
2. **横向调研**：搜索「[书籍所属领域] 同类书籍 对比 推荐」，获取与本书同领域的其他经典书籍或方法论
3. **交汇判断**：基于调研结果，标注本书在同领域中的独特定位和价值差异

**失败处理**：搜索无结果 → 标注「该领域市场信息暂缺」，仅做书中方法论阐述

### 阶段四：生成HTML PPT

**输入**：阶段二的分析内容 + 阶段三的行业数据

**输出**：完整的 `outputs/book-deep-read.html`

**生成规则**：

🛑 **批量制作前：先做 2 页 showcase 定 grammar**

deck ≥ 5 页时，先选视觉差异最大的 2 页（封面 + 任意一页内容页）做出来供用户确认方向，通过后再批量推进剩余页。避免方向错导致全局返工。

**设计系统（风格根据书籍主题自适应）**：

对每本书根据其主题选择匹配的配色和氛围，但必须遵守以下硬性规格：

- **字号阶梯**：标题 48-72px，正文 ≥22px，卡片文字 ≥16px。幻灯片在屏幕上观看，字号不能小
- **内容密度**：每页 = 1 个核心信息 + 3-4 个辅助点 + 1 个视觉主角。超过就拆分新页
- **呼吸感**：正文行高 ≥1.7，卡片间距 ≥16px，段落间留白 ≥20px。信息不要挤在一起
- **字体**：标题用衬线（Noto Serif SC），正文用无衬线（Inter），`text-wrap: balance` 用于标题，`text-wrap: pretty` 用于正文防孤行
- **渲染**：`-webkit-font-smoothing: antialiased` 加在 body 上
- **圆角协调**：外层 radius = 内层 radius + padding，避免嵌套元素圆角错位
- **阴影优先**：卡片用多层半透明 `box-shadow` 替代实色边框，产生自然层次感

**视觉节奏 — 页面类型轮换**：

如果所有页面都是「文字+卡片」会视觉疲劳。按内容自然轮换以下类型：

| 类型 | 适用场景 | 视觉特征 |
|------|---------|---------|
| 大字封面 | 封面页 | 巨大标题 + 渐淡背景 + 最小信息 |
| 知识图谱 | 总览页 | 全宽 SVG 节点关系图 |
| 过渡页 | 篇章切换 | 深色满版 + 篇章号 + 一行摘要 |
| 双栏对比 | 论点展开 | 左论证 + 右论证，中间分隔 |
| 卡片阵列 | 要点罗列 | 3-6 张卡片均匀分布 |
| 引用页 | 高价值划线 | 留白 + 大号引文 + 来源标注 |
| 数据表格 | 行业对比 | 全宽对比表 |
| 总结页 | 收束 | 卡片 + 核心引用 |

**交互体验**：

- **翻页**：JS 控制 slide 显隐，`opacity + scale(0.97→1)` 过渡，duration 350ms，easing `cubic-bezier(0.2, 0, 0, 1)`
- **导航栏**：底部固定，半透明毛玻璃背景（`backdrop-filter: blur(16px)`），含 ← → 按钮 + 页码 + 圆点指示器
- **键盘**：← → 翻页、Home/End 首尾页、空格下一页。preventDefault 阻止页面滚动
- **触屏**：`touchstart/touchend` 检测左右滑动，阈值 60px
- **圆点**：当前页圆点变长条（width 22px，border-radius 4px），其他为小圆点
- **过渡**：slide 入场 `opacity 0→1 + scale 0.97→1`，退场反向。尊重 `prefers-reduced-motion`：检测到则取消动画
- **溢出兜底**：单页超过视口时 `overflow-y: auto`，滚动条宽度 ≤6px 低调处理

**内容编排**：

- 封面页：书名 + 一句话核心主张 + 评分 + 出版信息
- 知识图谱页：SVG 绘制全书结构关系网络 + 3 个数字统计卡片
- 每篇过渡页：深色背景 + 篇号 + 篇名 + 章节范围 + 一句话摘要
- 每篇内容页：核心论述(200-400字) + 2-3 条热门划线卡片 + 1 条读者点评
- 行业交叉页：时间线 SVG + 对比表格 + 关键判断
- 总结页：6 张核心观点卡片 + 全书主张引用

**SVG 知识图谱绘制规范**：

- **布局**：采用环形/放射状布局，中心圆形节点 + 周边矩形卡片围绕。对于6篇以上内容，采用三行两列网格；4篇采用田字布局。避免节点散落、大面积空白
- **画布尺寸**：viewBox 控制在 `0 0 880 400` 内，不过度拉伸
- **节点尺寸**：矩形卡片 164×70px，包含 3 行文字紧凑排列；中心节点圆形 48px 半径
- **字号底线**：节点标题 ≥14px、副标题 ≥12px、说明文字 ≥11px。SVG 文字不能像 CSS 一样缩放，必须写死足够大的 px 值
- **配色**：中心节点用渐变色（`radialGradient`），外部节点用实色填充头部 + 白色卡片身，主色/辅色交替区分篇章类型。连接线用对应淡色半透明
- **连线**：从中心节点边缘精确计算到每个外部卡片中心，不用手动估算坐标。直线优于折线，避免过长跨画布连线
- **装饰**：中心外围可加一条虚线圆环辅助视觉层次，底部不需要单独的文字条——核心信息在中心节点中表达
- **阴影**：所有节点加 `feDropShadow` 滤镜（`dy="1" stdDeviation="3" flood-opacity=".06"`），轻量不厚重

**输出文件**：`outputs/book-deep-read.html`

### 阶段五：交付

**输入**：完整的 HTML PPT 文件

**输出**：`present_files` 调用结果

**步骤**：
1. 将生成的 HTML 写入：`outputs/book-deep-read.html`
2. 用 `tail -3` 验证文件末尾为 `</html>`，确认写入完整
3. 调用 `present_files` 展示结果

🔴 **CHECKPOINT-3**：展示文件路径和页数统计。若当前仅生成了 2 页 showcase，必须明确告知用户「这是样板，共展示封面+XX页，剩余 N 页待你确认视觉方向后继续生成」；若为完整版，则直接等待用户验收。

---

## PPT 结构模板

根据书籍的实际章节数量灵活调整，结构示例（以4篇10章为例）：

```
Slide 01 — 封面
Slide 02 — 全局总览（知识图谱 + 统计卡片）
Slide 03 — Part I 过渡页
Slide 04-06 — Part I 内容页（每章或合并章节，1-2页）
Slide 07 — Part II 过渡页
Slide 08-09 — Part II 内容页
Slide 10 — Part III 过渡页
Slide 11-13 — Part III 内容页
Slide 14 — Part IV 过渡页
Slide 15-16 — Part IV 内容页
Slide 17 — 行业交叉分析
Slide 18 — 总结提炼
```

**页码弹性规则**：
- 6篇以上书籍：每篇1过渡页 + 1内容页，共约 14-16 页
- 4篇书籍：每篇1过渡页 + 1-2内容页，共约 16-18 页
- 无篇结构书籍：按主题分组，约 12-14 页

---

## 关键约束

**数据真实性**：
- 所有章节数量、评分等数据必须基于API返回的真实数据，不得编造
- 热门划线卡片必须标记划线人数（如「899人划线」）
- 点评和划线内容不得修改原文，只做截取
- 行业交叉分析必须基于真实搜索调研，不可凭空编造

**阅读体验**：
- 每张幻灯片内容量：1 个核心信息 + 3-4 个辅助点 + 1 个视觉主角，通过 `overflow-y: auto` 兜底
- 正文最小 22px，卡片文字最小 16px，标题 48px+——幻灯片是给人看的，不是打印文档
- 视觉类型轮换：连续 3 页不能全用同一种布局（卡片→对比表→引用→双栏轮流）
- 配色根据书籍主题自适应：技术类用蓝绿冷色调，人文类用暖色调，亲子类用柔和紫粉
- `text-wrap: balance` 用于标题防止单字孤行，`text-wrap: pretty` 用于正文
- `-webkit-font-smoothing: antialiased` 必须加在 body 上

**外部 Skill 调用**：
- 当用户明确要求横纵分析时，调用 hv-analysis skill
- 当用户要求特定设计风格时，调用 ui-ux-pro-max 获取设计系统
- 默认情况使用本 skill 的内置设计规范，无需额外加载外部 skill
