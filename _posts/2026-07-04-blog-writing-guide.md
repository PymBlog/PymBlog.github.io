---
layout: post
title: "Jekyll 博客写作规范"
date: 2026-07-04 12:00:00 +0800
categories: 教程
---

## 文件命名

在 `_posts` 目录下创建文件，格式为：

```
YYYY-MM-DD-标题.md
```

例如 `2026-07-04-my-new-post.md`。

## Front Matter

每篇文章顶部必须包含 YAML front matter：

```yaml
---
layout: post
title: "文章标题"
date: 2026-07-04 12:00:00 +0800
categories: 分类名
---
```

| 字段 | 说明 |
|------|------|
| `layout` | 固定为 `post` |
| `title` | 文章标题 |
| `date` | 发布时间 |
| `categories` | 分类，多个用空格分隔 |

## Markdown 语法速查

### 标题

```markdown
# 一级标题
## 二级标题
### 三级标题
```

### 文本格式

```markdown
**粗体**
*斜体*
~~删除线~~
`行内代码`
```

### 链接与图片

```markdown
[链接文字](https://example.com)
![图片描述](/assets/image.png)
```

### 列表

```markdown
- 无序列表项
- 另一项

1. 有序列表项
2. 另一项
```

### 代码块

三个反引号加语言名：

````markdown
```python
print("Hello, World")
```
````

### 表格

```markdown
| 列1 | 列2 | 列3 |
|-----|-----|-----|
| A   | B   | C   |
```

### 引用

```markdown
> 这是一段引用文字。
```

## 写作建议

1. **标题简洁**——一眼看出文章主题
2. **开头点题**——前两段说清楚要解决什么问题
3. **小节拆分**——用 `##` 分段，每段讲一个点
4. **代码可运行**——贴代码时加上语言标注，确保能跑
5. **图比文字直观**——复杂流程优先用图

## 发布流程

```bash
# 1. 写文章
vim _posts/2026-07-04-my-post.md

# 2. 本地预览
bundle exec jekyll serve

# 3. 提交推送
git add _posts/2026-07-04-my-post.md
git commit -m "新文章: 文章标题"
git push
```

GitHub Actions 会自动构建部署，几分钟后线上就能看到。
