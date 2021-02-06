---
title: "意識低めのTED視聴"
emoji: "📺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [Chrome]
published: true
---

# 導入

- かつて意識が高めだった頃、ted.comの動画を見るのにハマっていた時があった。
- 久しぶりに見たらやっぱり面白いのだが、自分の意識が低くなっていて、15分以上の動画なんて見てられない(ゲーム実況動画なら見てしまうのだが)。
- そこで文字情報として摂取することにする。

## やりかた

- chromeでted.comのサイトにアクセスし、見たい動画を探す
- 例えば <https://www.ted.com/talks/linus_torvalds_the_mind_behind_linux>
- おもむろにDev Toolsのコンソールを開いて(Ctrl+Shift+J)、下記の呪文をペーストする

```javascript
  fetch("https://www.ted.com/talks/"+document.head.querySelector("[name^=twitter\\:app\\:url]").attributes.content.value.replace(/[^0-9]/g,"")+"/transcript.json?language=ja").then(r=>r.json()).then(j=>console.log(j.paragraphs.map(p=>p.cues.map(c=>c.text)).flat().join("\n")))
```

- 動画を再生しながら、consoleに現れたテキストを流し読みする。
- テキストを読み終えて動画の残り時間を確認すると、時間を節約できたような気になる。
- (笑)とかあって映像が気になるところは、動画をシークして確認する。

