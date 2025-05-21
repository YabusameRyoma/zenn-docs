---
title: "Obsidianデイリーノートのテンプレート"
emoji: "📝"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Obsidian","task"]
published: false
---

```text
---
tags: ""
datetimeCreate: {{date}} {{time}}
---
## 📝 今日のTASK
- [ ] 
---
## 🚨 割り込みTASK
- [ ] 
---
## ❓ 今日のASK
- [ ] 
---
## 🗣️分報
---
## 📅 日報
- 
---
## 🔄 KPTふりかえり
### ✅ Keep（良かったこと・継続したいこと）
- 
### 🛑 Problem（課題・困ったこと）
- 
### 🚀 Try（次に試したいこと・改善案）
- 
---
## 💡 気づき・学び・面白かったこと（Insights）
- 
---
# 🔗 関連ノート

```dataviewjs
var maxLoop = Math.min(dv.current().file.tags.length, 3);
for(let i=0;i<maxLoop;i++){
dv.span(dv.current().file.tags[i]);
dv.list(dv.pages(dv.current().file.tags[i]).sort(f=>f.file.mtime.ts,"desc").limit(15).file.link);
}

for (let outgo of dv.pages('outgoing([[' + dv.current().file.name + ']])')) {
    dv.header(4, outgo.file.name);
    dv.list(outgo.file.inlinks.sort());
}
```
```