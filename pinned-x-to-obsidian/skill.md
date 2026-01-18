# pinned-x-to-obsidian

**从 X/Twitter pinned URL 生成 Obsidian 文章**

## 适用场景

当你需要：
- 从 X/Twitter 的 pinned 推文收集整理信息到 Obsidian
- 批量提取推文中的提示词、图片等内容
- 将内容保存为结构化的 Obsidian markdown 文件

## 工作流程

### 1. 初始设置

```bash
# 创建 Obsidian vault 目录（如果不存在）
mkdir -p ~/Documents/ObsidianVault/images
```

### 2. 使用 Playwright 访问目标 URL

```javascript
// 在 Playwright 中执行
await page.navigate('https://x.com/username/status/123456');
await page.waitForTimeout(3000);

// 滚动加载所有内容
for (let i = 0; i < 5; i++) {
  await page.evaluate(() => window.scrollBy(0, 2000));
  await page.waitForTimeout(2000);
}
```

### 3. 提取推文信息和图片链接

```javascript
// 提取所有图片 URL
const imageSrcs = await page.evaluate(() => {
  const imgs = Array.from(document.querySelectorAll('img[alt="Image"]'));
  return imgs.map(img => img.src);
});

// 提取推文文本
const tweetTexts = await page.evaluate(() => {
  const tweets = Array.from(document.querySelectorAll('[data-testid="tweetText"]'));
  return tweets.map(t => t.innerText);
});
```

### 4. 批量下载图片

```bash
# 为每个提示词创建有序文件名
curl -L -o "3-coloring-1.jpg" "https://pbs.twimg.com/media/xxx?format=jpg&name=medium"
curl -L -o "3-coloring-2.jpg" "https://pbs.twimg.com/media/yyy?format=jpg&name=medium"
```

**关键点**：
- 使用英文文件名（避免编码问题）
- 按提示词编号 + 序号命名（如 `3-coloring-1.jpg`）
- 保存到 `ObsidianVault/images/` 目录

### 5. 访问引用推文获取完整内容

```javascript
// 对每个引用推文重复上述步骤
for (const tweet of referencedTweets) {
  await page.navigate(tweet.url);
  await page.waitForTimeout(3000);

  // 提取完整提示词内容
  const fullPrompt = await page.evaluate(() => {
    return document.querySelector('[data-testid="tweetText"]')?.innerText;
  });
}
```

### 6. 生成 Obsidian Markdown

使用以下结构：

```markdown
# [文章标题]

> 来源：[原推文链接](https://x.com/...)
> 整理时间：YYYY-MM-DD

---

## 1. [提示词名称] ([英文名])

**作用**：[提示词功能描述]

**提示词**：
```
[完整提示词内容，保留代码块格式]
```

**效果图**：
![[images/3-coloring-1.jpg]]
![[images/3-coloring-2.jpg]]
![[images/3-coloring-3.jpg]]
![[images/3-coloring-4.jpg]]

---

## 2. [下一个提示词]
...
```

**关键格式要求**：
- ✅ 使用 `![[ ]]` wiki-link 语法引用图片
- ❌ **不要**用代码块 ``` 包裹图片列表（会导致无法显示）
- ✅ 每个图片独占一行
- ✅ 图片路径相对于 vault 根目录（如 `images/xxx.jpg`）

### 7. 验证和修复

```bash
# 统计图片引用
grep -o "!\[\[images/" "文件名.md" | wc -l

# 检查提示词结构
grep -n "^## " "文件名.md"

# 验证图片文件存在
ls -lh ~/Documents/ObsidianVault/images/
```

**常见问题修复**：

| 问题 | 原因 | 解决方案 |
|------|--------|----------|
| 图片无法显示 | 用 ``` 包裹了图片列表 | 删除 ``` 符号 |
| 文件名乱码 | 使用了中文文件名 | 重命名为英文 |
| 内容丢失 | 正则替换范围过大 | 改用字符串精确替换 |

### 8. Git 版本控制

```bash
cd ~/Documents/ObsidianVault

# 添加文件
git add "文件名.md" images/

# 提交
git commit -m "Add [描述]"

# 推送（如需要）
git push
```

## 关键技术点

### 图片下载策略

```bash
# 批量下载脚本示例
cat > download-images.sh << 'EOF'
#!/bin/bash
curl -L -o "prompt1-image1.jpg" "URL1"
curl -L -o "prompt1-image2.jpg" "URL2"
curl -L -o "prompt2-image1.jpg" "URL3"
EOF

chmod +x download-images.sh
./download-images.sh
```

### URL 替换技巧

**错误方式**（会误删内容）：
```python
# 危险！正则范围过大
re.sub(r'## 3.*?(?=## 4)', replacement, content)
```

**正确方式**（精确替换）：
```python
# 安全！只替换特定 URL
url_to_images = {
    'https://x.com/status/12345': '![[images/prompt1-1.jpg]]\n![[images/prompt1-2.jpg]]'
}

for url, images in url_to_images.items():
    old_text = f'[查看原推文]({url})'
    new_text = images
    content = content.replace(old_text, new_text)
```

### Playwright 超时处理

```javascript
// 等待元素出现
await page.waitForSelector('[data-testid="tweet"]', { timeout: 10000 });

// 处理动态加载
const isLoadComplete = await page.evaluate(() => {
  return document.readyState === 'complete';
});

// 多次滚动确保加载所有内容
for (let i = 0; i < 10; i++) {
  await page.evaluate(() => window.scrollBy(0, 1000));
  await page.waitForTimeout(1000);
}
```

## 实际案例参考

### 案例：宝玉的 nano banana 提示词

**输入**：https://x.com/dotey/status/1996285439867556304

**输出结构**：
- 19 个提示词
- 104 张效果图
- 45 张图片已引用

**文件结构**：
```
~/Documents/ObsidianVault/
├── 宝玉的 nano banana 提示词.md
└── images/
    ├── 3-coloring-1.jpg
    ├── 3-coloring-2.jpg
    ├── 4-home-office-1.jpg
    ├── 4-home-office-2.jpg
    └── ...
```

**统计数据**：
```
提示词总数: 19
图片引用总数: 45
包含"作用"的提示词: 19
包含"提示词"的提示词: 19
```

## 常见问题和解决方案

### 问题 1：图片引用格式错误

**错误**：
```markdown
**效果图**：
```
![[images/prompt1-1.jpg]]
![[images/prompt1-2.jpg]]
```
```

**正确**：
```markdown
**效果图**：
![[images/prompt1-1.jpg]]
![[images/prompt1-2.jpg]]
```

### 问题 2：中文文件名导致编码问题

**解决方案**：
```bash
# 使用英文文件名
mv "1-天气卡片.jpg" "1-weather-card-1.jpg"
mv "2-卡通信息图.jpg" "2-cartoon-infographic-1.jpg"
```

### 问题 3：Git 提交前忘记检查

**验证清单**：
- [ ] 所有图片文件已下载
- [ ] 图片路径正确（相对路径）
- [ ] Obsidian 可以正常显示图片
- [ ] Markdown 结构完整
- [ ] 使用正确的 wiki-link 语法

### 问题 4：动态内容未完全加载

**解决方案**：
```javascript
// 多次滚动确保加载
async function scrollToBottom() {
  let lastHeight = 0;
  while (true) {
    await page.evaluate(() => window.scrollBy(0, 2000));
    await page.waitForTimeout(2000);

    const height = await page.evaluate(() => document.body.scrollHeight);
    if (height === lastHeight) break;
    lastHeight = height;
  }
}
```

## 工具和依赖

### 必需工具
- **Playwright MCP**：浏览器自动化
- **curl**：下载图片
- **Git**：版本控制

### Python 库（可选）
```python
import re        # 正则表达式
import os        # 文件操作
import json       # JSON 处理
```

### Bash 工具
```bash
# 文本处理
grep, sed, awk

# 统计工具
wc -l, wc -c

# 文件管理
ls, mkdir, cp, mv
```

## 最佳实践

1. **分阶段工作**
   - 先收集所有 URL
   - 再批量下载图片
   - 最后生成 markdown

2. **使用有意义的命名**
   - 提示词编号 + 英文描述
   - 顺序编号（1, 2, 3...）

3. **保持文件结构清晰**
   - markdown 文件在根目录
   - 图片在子目录 `images/`
   - 使用相对路径引用

4. **版本控制**
   - 提交前验证
   - 写清晰的提交信息
   - 定期备份

5. **错误恢复**
   - 使用 Git 历史恢复误删内容
   - 检查前先查看差异
   - 小步提交，快速发现错误

## 性能优化

### 并行下载

```bash
# 使用 xargs 并行下载
cat urls.txt | xargs -P 4 -I {} curl -L -o {}.jpg {}

# 或使用 GNU parallel
cat urls.txt | parallel -j 4 curl -L -o {}.jpg {}
```

### 缓存策略

```python
# 检查文件是否已存在
import os

def download_image(url, filename):
    if os.path.exists(filename):
        print(f"✓ Skipped: {filename}")
        return
    # 下载逻辑...
```

## 扩展功能

### 自动化完整流程

```python
def pinned_x_to_obsidian(url, vault_path):
    """完整的自动化流程"""
    # 1. 访问 URL 并收集数据
    data = collect_from_x(url)

    # 2. 下载所有图片
    images = download_images(data['image_urls'], vault_path)

    # 3. 生成 markdown
    markdown = generate_markdown(data, images)

    # 4. 保存文件
    save_markdown(markdown, vault_path)

    return markdown
```

### 批量处理

```bash
# 处理多个 pinned URL
for url in $(cat urls.txt); do
  pinned-x-to_obsidian "$url" ~/Documents/ObsidianVault
done
```

## 相关资源

- [Obsidian Wiki Links](https://help.obsidian.md/How+to/Link+notes)
- [Playwright Documentation](https://playwright.dev/)
- [Twitter API](https://developer.twitter.com/)

---

**最后更新**：2026-01-18
**维护者**：AI Assistant
**版本**：1.0.0
