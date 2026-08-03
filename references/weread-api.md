# 微信读书 API 参考

## 接口总览

所有接口通过统一网关调用，需要设置 `WEREAD_API_KEY` 环境变量。

### 通用规则

| 规则 | 说明 |
|------|------|
| 网关地址 | `POST https://i.weread.qq.com/api/agent/gateway` |
| 鉴权 | Header: `Authorization: Bearer $WEREAD_API_KEY` |
| 参数位置 | 业务参数与 `api_name`、`skill_version` 平铺在body顶层 |
| 版本 | 每次请求必带 `"skill_version": "1.0.4"` |
| API Key 格式 | `wrk-xxxxxxxx` |

### 请求示例

```bash
curl -s -X POST "https://i.weread.qq.com/api/agent/gateway" \
  -H "Authorization: Bearer $WEREAD_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"api_name":"/store/search","keyword":"书名","count":5,"skill_version":"1.0.4"}'
```

### 错误示范

❌ **不要把业务参数包在 `params` 里**——这会导致参数被忽略，后端按默认值返回。

```json
// 错误
{"api_name":"/store/search","params":{"keyword":"书","count":5},"skill_version":"1.0.4"}
// 正确
{"api_name":"/store/search","keyword":"书","count":5,"skill_version":"1.0.4"}
```

---

## 接口列表

### 1. `/store/search` — 搜索书籍

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| keyword | string | 是 | 搜索关键词 |
| count | int | 否 | 返回数量，默认20 |

**回包关键字段：**

| 字段路径 | 说明 |
|----------|------|
| `results[].books[].bookInfo.bookId` | 书籍ID（后续所有接口需要） |
| `results[].books[].bookInfo.title` | 书名 |
| `results[].books[].bookInfo.author` | 作者 |
| `results[].books[].bookInfo.newRating` | 评分（百分制，846=84.6分） |
| `results[].books[].bookInfo.newRatingCount` | 评分人数 |
| `results[].books[].bookInfo.deepLink` | 跳转链接 |

### 2. `/book/info` — 书籍详情

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| bookId | string | 是 | 书籍ID |

**回包关键字段：** `title`, `author`, `cover`, `intro`, `publishTime`, `newRating`, `newRatingDetail`（含good/fair/poor计数）, `category`, `publisher`

### 3. `/book/chapterinfo` — 章节目录

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| bookId | string | 是 | 书籍ID |

**回包关键字段：**

| 字段 | 说明 |
|------|------|
| `chapters[].chapterUid` | 章节UID，用于查询划线 |
| `chapters[].title` | 章节标题 |
| `chapters[].level` | 层级（1=篇, 2=章） |
| `chapters[].wordCount` | 字数 |
| `chapters[].anchors[]` | 锚点/子标题数组 |
| `chapters[].anchors[].title` | 锚点标题 |

**注意**：有些章节（如第4-5章、第6章）的锚点可能包含在父章节中，需遍历 `anchors` 获取完整结构。

### 4. `/book/bestbookmarks` — 热门划线

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| bookId | string | 是 | 书籍ID |
| chapterUid | int | 否 | 章节UID，0=全部章节 |

**回包关键字段：**

| 字段 | 说明 |
|------|------|
| `totalCount` | 热门划线总数 |
| `items[].markText` | 划线原文 |
| `items[].totalCount` | 划线人数 |
| `items[].chapterUid` | 所在章节UID |
| `chapters[]` | 章节信息数组，用于定位 |

### 5. `/review/list` — 书籍公开点评

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| bookId | string | 是 | 书籍ID |
| reviewListType | int | 否 | 0=全部, 1=推荐, 2=差评, 3=最新, 4=一般 |
| count | int | 否 | 每页数量 |

**回包关键字段：**

点评数据嵌套两层 `review`：`reviews[].review.review.xxx`

| 字段 | 说明 |
|------|------|
| `reviews[].review.review.content` | 点评文本内容 |
| `reviews[].review.review.htmlContent` | 点评HTML内容（content为空时使用） |
| `reviews[].review.review.star` | 评分（100=五星, 80=四星, 60=三星, 40=二星, 20=一星） |
| `reviews[].review.review.author.name` | 评论者昵称 |
| `reviews[].review.review.isFinish` | 是否读完 |

**注意**：`content` 可能为空，需回退到 `htmlContent`。`star` 值除以20得星级。

### 6. `/book/bookmarklist` — 个人划线

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| bookId | string | 是 | 书籍ID |

回包中 `updated[]` 包含划线数据，`markText` 为划线原文。

### 7. `/review/list/mine` — 个人想法与点评

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| bookid | string | 是 | 书籍ID |
| count | int | 否 | 每页数量 |

---

## 数据处理注意事项

1. **评分转换**：`newRating` 为百分制整数，展示时除以10；`star` 除以20得星级
2. **双层review嵌套**：`/review/list` 返回 `reviews[].review.review.xxx`，注意取内层数据
3. **content为空**：点评的 `content` 可能为空，同时检查 `htmlContent`
4. **章节锚点**：部分章节标题作为父级章节的 `anchors` 存在（如CHAPTER 5出现在uid=11的anchors中）
5. **时间戳**：Unix时间戳（如 `createTime`, `updateTime`）展示时转为 `YYYY-MM-DD` 格式
6. **划线人数**：`/book/bestbookmarks` 的 `items[].totalCount` 表示划线人数，展示时标注「X人划线」
7. **升级检测**：回包中若出现 `upgrade_info` 字段，暂停操作并按提示升级skill版本
