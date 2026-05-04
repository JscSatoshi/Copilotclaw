---
name: web
description: "Use this skill for all web access in this stack. Native web/browser toolsets are intentionally disabled, so use terminal commands with curl against http://skillserver:3000/."
triggers:
- web
- browser
- search
- navigate
- website
- url
- internet
- online
- fetch
- crawl
- screenshot
---
# Local Web Stack

这个仓库的 Hermes 配置故意禁用了内建 `web` 和 `browser` toolset，统一走本地 `skillserver`。

唯一可用的 Web 访问路径：在终端里调用 `curl http://skillserver:3000/...`。

基础规则：

1. 所有请求都带 `-s --max-time <seconds>`。
2. 非 ASCII 查询先 URL 编码，再拼到参数里。
3. 实时信息、价格、文档、新闻、网页内容都先搜再答。
4. 返回给用户时要整理成自然语言，不要直接贴原始 JSON。

## Search

```bash
curl -s --max-time 15 "http://skillserver:3000/search?q=YOUR+QUERY" | head -c 8000
```

## Deep Search

```bash
curl -s --max-time 60 "http://skillserver:3000/deep_search?q=YOUR+QUERY&max_results=3" | head -c 16000
```

## Navigate

```bash
curl -s --max-time 30 "http://skillserver:3000/navigate?url=https%3A%2F%2Fexample.com" | head -c 12000
```

## Screenshot

```bash
curl -s --max-time 30 "http://skillserver:3000/screenshot?url=https%3A%2F%2Fexample.com&full_page=false"
```

返回示例：

```json
{"url":"https://example.com","format":"png","media":"MEDIA:/tmp/screenshots/screenshot_xxx.png"}
```

如果用户要求看页面、确认视觉效果、或者“给我截图”，调用 `/screenshot`，并把返回里的 `MEDIA:...` 原样输出一行。

## URL Encoding

```bash
Q=$(node -e "process.stdout.write(encodeURIComponent('贵州茅台 股价 今日'))")
curl -s --max-time 15 "http://skillserver:3000/search?q=${Q}&language=zh-CN" | head -c 8000
```
