# mongolia19's Blog Space

Welcome to my personal blog space! This is a modern, responsive Single Page Application (SPA) static blog website built with pure HTML, CSS, and JavaScript. It runs on GitHub Pages and requires **zero build steps** to publish new articles.

欢迎来到我的个人博客空间！这是一个使用纯前端技术（HTML, CSS, JS）构建的现代化单页应用（SPA）静态博客，直接运行在 GitHub Pages 上，发布新文章**无需任何编译或构建步骤**。

---

## Features / 特性

- ⚡ **Zero Build Step / 零构建**: Simply drop your Markdown file and push. / 放入 Markdown 文件并推送即可发布。
- 🔍 **Instant Search / 实时搜索**: Search posts instantly by title, summary, or tags. / 支持标题、摘要和标签的实时关键字过滤。
- 🌐 **Language Filter / 多语言过滤**: Toggle between English and Chinese posts. / 支持中英文一键过滤。
- 🎨 **Premium Responsive Design / 极简美学**: Outfit & Lora typography with transition animations. / 完美适配手机与桌面端，提供流畅的动画效果。
- 🌗 **Auto Dark Mode / 自动暗黑模式**: Supports manual toggle and respects system preferences. / 支持手动切换，并能自动适配系统暗黑模式。

---

## File Structure / 目录结构

```text
mongolia19.github.io/
├── index.html        # SPA Blog Engine & Styles / 博客单页应用引擎与全局样式
├── posts.json        # Post Database / 文章注册数据库 (JSON)
├── README.md         # Documentation / 本说明文档
└── posts/            # Markdown Files & Covers / 文章内容与封面图片存放目录
    ├── post-1.md
    └── cover-1.jpg
```

---

## How to Add New Articles / 如何添加新文章

To publish a new article or blog post, follow these simple steps:

要发布一篇新博文，只需按以下步骤操作：

### Step 1: Copy Markdown & Images / 准备文件
Place your Markdown file (e.g. `my-new-post.md`) and any cover images (e.g. `my-cover.jpg`) inside the `posts/` directory.

将您的 Markdown 文章（如 `my-new-post.md`）和封面配图（如 `my-cover.jpg`）复制到 `posts/` 目录中。

### Step 2: Register in `posts.json` / 注册文章信息
Open the root `posts.json` file and append a new JSON object to the array:

打开根目录下的 `posts.json`，在 JSON 数组的最前面或后面追加一个对象：

```json
  {
    "id": "my-new-post",
    "title": "Your Article Title / 您的文章标题",
    "date": "2026-07-02",
    "summary": "A brief summary of your article... / 文章简短摘要...",
    "cover": "posts/my-cover.jpg",
    "lang": "en", // "en" for English, "cn" for Chinese
    "file": "posts/my-new-post.md",
    "tags": ["Tag1", "Tag2"]
  }
```

#### Fields Description / 字段说明：
- `id`: A unique URL slug for the article (e.g., `?post=my-new-post`). / 唯一的 URL 标识符。
- `title`: The title displayed on the card and detail page. / 卡片和详情页显示的文章标题。
- `date`: Publication date (YYYY-MM-DD). / 发布日期。
- `summary`: Short excerpt for the card snippet. / 显示在列表中的卡片摘要。
- `cover`: Path to the cover image inside the repository. / 封面图片在仓库中的相对路径。
- `lang`: Language code (`cn` or `en`). / 语言代码。
- `file`: Path to the Markdown file. / Markdown 文章在仓库中的相对路径。
- `tags`: An array of keywords for search filtering. / 用于搜索过滤的标签数组。

### Step 3: Git Commit & Push / 提交并推送
Push the changes to your GitHub repository:

在终端运行命令提交并推送到 GitHub 仓库：

```bash
git add .
git commit -m "feat: add my new post"
git push origin main
```

Within a minute, GitHub Pages will compile and automatically showcase your new article!

大约一分钟后，GitHub Pages 会自动更新并展示您的最新博文！

---

## Technical Stack / 技术栈

- Markdown Parser: [marked.js](https://github.com/markedjs/marked)
- Syntax Highlighter: [highlight.js](https://github.com/highlightjs/highlight.js)
- Typography: Inter & Lora (Google Fonts)
- Styling: Custom Vanilla CSS variables
