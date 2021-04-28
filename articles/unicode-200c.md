---
title: "対人間用コード難読化？unicode-200c"
emoji: "⁉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript", "unicode"]
published: true
---

# tl;dr;

- JavaScriptでは変数は`alphabet`以外の、例えば日本語とかでもOKらしい。
- 試したら、たいていのUnicodeもいけるっぽい。
- つまりこんないたずらも可能(DevTools Consoleなどにコピペしてみて)。
- ヒントは`U+200C`。

```javascript
let a = 1;
    a‌ = 2;
console.log(a) // 1
```