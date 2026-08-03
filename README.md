# deep-read-canvas

微信读书书籍精读分析 → 翻页式 HTML 演示文稿，一站式生成。

## 做什么

输入一本书名，自动从微信读书获取完整数据，生成一份主题自适应的翻页式 HTML 演示文稿。输出包含：全书知识图谱、章节深度分析、读者热门划线、公开点评、同领域行业交叉对比、总结提炼。

## 怎么用

1. 配置微信读书 API Key：`export WEREAD_API_KEY=<your-key>`
2. 对 WorkBuddy 说：「帮我精读书架里的《XXX》」
3. 确认分析范围后，自动生成 `outputs/book-deep-read.html`
4. 浏览器打开，键盘方向键翻页演示

## 输出效果

- 环形 SVG 知识图谱，展示全书结构关系
- 每篇章过渡页 + 内容页，自动轮换双栏/卡片/引用/表格布局
- 热门划线标注读者人数，精选点评增强可信度
- 22px 正文、48px+ 标题，适合屏幕观看
- 底部毛玻璃导航栏，键盘/触屏双模翻页
- 配色根据书籍主题自适应（技术冷色 / 人文暖色 / 亲子柔和）
- 尊重 `prefers-reduced-motion`，关闭动画

## 技术栈

微信读书 Agent Gateway API · SVG 内联绘图 · Vanilla JS 幻灯片引擎 · Google Fonts (Noto Serif SC + Inter)

## 安装

```bash
# 将 skill 目录放入 WorkBuddy skills 路径
cp -r deep-read-canvas/ ~/.workbuddy/skills/
```

## 前置依赖

- WorkBuddy 运行环境
- 微信读书 API Key（格式 `wrk-xxxxxxxx`）
- 网络连接（API 调用 + Google Fonts 加载）
