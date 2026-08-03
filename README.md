# deep-read-canvas

> 微信读书书籍精读分析 → 主题自适应翻页式 HTML 演示文稿

输入一本书名，自动从微信读书获取完整数据（章节、划线、点评），进行深度分析，最终生成一份可浏览器翻页演示的 HTML PPT——包含知识图谱、章节分析、读者评论、行业交叉对比和总结提炼。

---

## 安装

### 1. 获取 Skill

```bash
git clone git@github.com:Polardong66/deep-read-canvas.git ~/.workbuddy/skills/deep-read-canvas
```

### 2. 配置微信读书 API Key

**获取 API Key：**

1. 打开 [微信读书 Skill 配置页](https://weread.qq.com/r/weread-skills)
2. 点击「快速配置」，扫码登录微信读书账号
3. 页面会生成一串 `wrk-` 开头的 API Key，点击复制
4. ⚠️ Key 绑定你的个人账号，请勿泄露给他人

> 除非主动重置，Key 长期有效。

**配置 API Key（二选一）：**

**方式一：环境变量（当前终端有效）**

```bash
export WEREAD_API_KEY=wrk-your-key-here
```

**方式二：持久化保存（推荐，跨会话生效）**

将以下内容写入 `~/.workbuddy/MEMORY.md`：

```
WEREAD_API_KEY: wrk-your-key-here
```

Skill 启动时会自动从环境变量和 MEMORY.md 中检测。均未配置时会暂停并提示。

### 3. 验证安装

在 WorkBuddy 中输入「帮我看看书架里有什么书」——如果 skill 正确加载，会自动调用微信读书 API 获取书架数据。

---

## 使用方式

Skill 通过自然语言触发，无需手动调用命令。在 WorkBuddy 中说以下任意一句即可：

| 你说 | 效果 |
|------|------|
| 「帮我精读书架里的《XXX》」 | 完整五阶段流程：采集 → 分析 → 调研 → 生成 PPT → 交付 |
| 「分析一下《XXX》这本书」 | 同上，标准流程 |
| 「把《XXX》做成翻页 PPT」 | 侧重输出环节 |
| 「生成《XXX》的知识图谱」 | 可只看图谱部分 |

**执行过程分三个检查点，每个节点等待你的确认**：
1. 数据采集完 → 展示书名、评分、章节数，确认分析范围
2. 结构设计完 → 展示 PPT 页数和各页摘要，确认是否调整
3. PPT 生成完 → 展示文件路径，等你验收

输出文件始终为 `outputs/book-deep-read.html`，浏览器打开后用 **← → 方向键翻页**。

---

## 功能特征

### 📊 数据采集
- 调用微信读书 Agent Gateway API，同时拉取：书籍元数据、章节目录、热门划线（含读者人数）、推荐点评
- 接口并行调用减少等待，单个失败有 10+ 种 fallback 兜底
- 原始 JSON 保存到 `/tmp/` 便于排查

### 🧠 内容分析
- 按书籍自然篇章结构组织，每篇章生成 200-400 字核心论述
- 精选每条划线的读者人数（如「899人划线」），标注章节来源
- 整合读者点评（标注作者 + 星级），增强可信度

### 🔬 行业交叉
- 纵向：搜索领域发展脉络，构建里程碑时间线
- 横向：搜索同类书籍/方法论，形成对比表
- 交汇：标注本书在同领域的独特定位

### 🎨 输出设计
- **主题自适应配色**：技术类蓝绿冷色 / 人文类暖色 / 亲子类柔和紫粉
- **字号阶梯**：标题 48-72px，正文 22px+，卡片 16px+
- **内容密度**：每页 1 核心 + 3-4 辅助 + 1 视觉主角，超过自动拆页
- **视觉节奏**：8 种页面类型轮换（封面/图谱/过渡页/双栏/卡片/引用/表格/总结）
- **SVG 知识图谱**：环形放射状布局，164×70px 节点，14px 最小字号
- **毛玻璃导航栏**：底部固定，显示 ← → 按钮 + 页码 + 圆点导航
- **全键盘控制**：方向键翻页、Home/End 首尾页、空格下一页
- **触屏滑动**：左右滑动手势，阈值 60px
- **动画克制**：350ms cubic-bezier 过渡，`prefers-reduced-motion` 自动取消

---

## 文件结构

```
deep-read-canvas/
├── SKILL.md                         # 核心指令：5阶段流程 + 设计规范 + 异常处理
├── README.md                        # 本文件
├── references/
│   └── weread-api.md                # 微信读书 API 接口文档（7 个接口 + 数据处理要点）
└── assets/
    └── ppt-template.html            # HTML PPT 模板（CSS 变量 + 交互 JS + SVG 示例）
```

---

## 常见问题

### Q: 提示「WEREAD_API_KEY 未设置」？

按上方「配置微信读书 API Key」步骤检查。可运行 `echo ${#WEREAD_API_KEY}` 验证——输出 `0` 表示未设置。

### Q: 搜不到书？

Skill 会自动尝试去副标题重搜。仍失败会提示并提供手动指定 bookId 的选项。bookId 可在微信读书网页版 URL 中找到。

### Q: 生成的 PPT 字体显示异常？

模板使用 Google Fonts（Noto Serif SC + Inter），需要网络加载。首次打开等待 2-3 秒让字体下载完成。离线环境可用系统回退字体。

### Q: 数据不准？

所有章节、评分、划线数据均直接从微信读书 API 实时获取，不编造、不修改。若与 APP 显示有差异，可能是 API 缓存延迟。

### Q: 如何自定义输出风格？

在 WorkBuddy 中对 skill 说「用科技感深色风格」即可触发 `ui-ux-pro-max` 重新生成设计系统。

---

## 技术栈

- 微信读书 Agent Gateway API（REST JSON）
- WorkBuddy Skill 引擎
- SVG 内联绘图（feDropShadow / radialGradient）
- Vanilla JS 幻灯片引擎（opacity + scale 过渡）
- Google Fonts（Noto Serif SC + Inter）
- Cubic-bezier 缓动函数 + backdrop-filter 毛玻璃
- prefers-reduced-motion 无障碍适配

---

## License

MIT
