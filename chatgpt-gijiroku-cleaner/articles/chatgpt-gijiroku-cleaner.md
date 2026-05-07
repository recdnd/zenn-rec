---
title: "ChatGPT議事録の出力、毎回手で整形してませんか？"
emoji: "🌀"
type: "tech"
topics: ["chatgpt", "productivity", "text-processing", "workflow"]
published: true
---

## はじめに

社内会議の議事録を ChatGPT に整理してもらう運用、最近めちゃくちゃ増えてきてると思います。
私も例に漏れず、文字起こし → ChatGPT で要約・整形 → Obsidian / 共有ドキュメントに貼り付け、という流れで毎日やってます。

ただ、これ毎日やってる人なら絶対共感してもらえると思うんですが —— **ChatGPT の出力って、地味に"汚い"んですよね**。

- 謎の空行が 2〜3 行入る
- 箇条書きの記号がバラバラ（`-` と `・` と `*` が混在、たまに `- •` みたいな崩れも）
- 同じ内容の文が微妙に表現を変えて重複してる
- なぜかコピペすると全角スペースが混入してる

毎回これを手で直すの、ぶっちゃけ「議事録を整える時間」のほうが「会議の中身を読む時間」より長くなってる日があります。

ということで、**この後処理を全部自動化するブラウザツールを作りました**。記事の最後にリンク貼っておきます（無料・登録不要）。今回は「なぜ作ったのか」と「どう設計したのか」、それから業務でどう使ってるかを共有します。

![sample-case-1](/images/sample-case-1.png)

## 既存ツールで解決できなかった理由

最初は VS Code の正規表現置換と `sed` ワンライナーで頑張ってました。が、すぐ限界がきます。

理由は単純で、**汚れ方のパターンが多すぎる**から。一つの正規表現を増やすと、別のケースが壊れる。会議資料には日本語・英語・コード・URL・数式が混ざるので、「全部に効くルール」を 1 行で書くのが現実的に無理。

LLM（ChatGPT に「整形して」と頼む）もダメでした。理由は 2 つ：

1. **再現性がない** — 同じ入力でも毎回少しずつ違う出力になる
2. **勝手に内容を改変する** — 「整形だけして」と頼んだのに要約まで始めることがある

業務で使うなら、**入力 X に対して出力 Y が常に一意に定まる "決定論的" な処理**じゃないとダメだなと思ったのが出発点です。レビューする側も結果を信頼できるし、後から「なぜこう変換されたのか」も説明できる。

## 設計：小さなモジュールのパイプライン

ちなみにここから "モジュール" とか "パイプライン" って単語が出てきますが、UI 用語というより、TeX や Unix pipe あたりの古典的な意味のほうで使ってます。詳しくは末尾の「観察可能な決定論的パイプライン」で。

最終的に、こういう構成に落ち着きました。

```
入力 → parse → normalize → transform → render → 出力
```

各ステップは独立した小さなモジュールです。

| モジュール | 役割 |
|---|---|
| `boiler` | ヘッダー/フッター/ノイズ行の除去 |
| `pageNoiseStrip` | ページ番号・改ページ由来のノイズ行を除去 |
| `joinlines` | PDF 由来の不自然な改行を結合 |
| `punct` | 句読点・全半角スペースの正規化 |
| `dedupe` | 重複行/重複文の検出と削除 |
| `limit` | 文字数 / 文 / 段落単位の上限を適用 |
| `trimmax` | 連続する空行を最大 N 行に圧縮 |
| `commaspacing` | 句読点周りのスペース整形 |

この上に 3 つのプリセットを用意していて、**ほとんどの場合これで足ります**：

- `clean` — 軽い整形（議事録・社内資料向け）
- `notes` — メモや箇条書きの構造を保ったまま整形
- `strong` — 容赦ない正規化（PDF 抽出・論文の本気の清書）

## 実例：mixed AI / meeting log cleanup

例えばこんな入力：

```txt
# Q3 営業戦略ミーティング 議事録

- • 開催日時: 2026/05/06
• - 参加者: 田中 / 山田 / 鈴木
* • 場所: online


- 議題1: 新規 onboarding flow について
- 議題1: 新規 onboarding flow について

田中さんから報告がありました。

目標達成率は 102% です ​。

「 現状では retention が低い 」

β版リリースは6月中旬を予定
β版リリースは6月中旬を予定


- duplicated line
- duplicated line


# TODO

・LP の copy 修正
・pricing table 更新
・FAQ 追加


本研究では、決定論的テキスト処理パイプラインに
ついて検討する。LLMベースの整形は再現性に
課題があり、業務利用には適さない。


- • ripgrep
• - jq
* • sed


Contact :   support@example.com
```

`clean` プリセットを通すと：

```txt
Q3 営業戦略ミーティング 議事録
- 開催日時: 2026/05/06
- 参加者: 田中 / 山田 / 鈴木
- 場所: online
議題1: 新規 onboarding flow について
田中さんから報告がありました。
目標達成率は 102% です。
「現状では retention が低い」
β版リリースは6月中旬を予定
duplicated line
TODO
・LP の copy 修正
・pricing table 更新
・FAQ 追加
本研究では、決定論的テキスト処理パイプラインに
ついて検討する。LLMベースの整形は再現性に
課題があり、業務利用には適さない。
- ripgrep
- jq
- sed
Contact : support@example. com
```

1つの入力に「箇条書き崩れ」「重複行」「空白ノイズ」「全角スペース混入」「行分割の揺れ」が同時に混在していても、同じルールで安定して整形できるのがポイントです。

## 観察可能な決定論的パイプライン

ここまで「便利な整形ツール」みたいな紹介をしてきましたが、kiln の設計はもうちょっと奥にあります。

kiln は実装的には **TeX / Unix pipe / compiler pass pipeline** の系譜にあるツールです。AI ラッパーや SaaS formatter ではなく、**テキストに対して決定論的な atomic operation を直列に適用する transformation system** として作ってます。

```
text → op → op → op → output
```

`punct`、`bullet-normalize`、`dedupe`、`trimmax` といったモジュールは「機能ボタン」ではなく、テキストに変換を施す atomic op です。それが固定された順序で適用される。

ブラウザのコンソールを開きながら kiln を動かすと、`modules/index.js` のこの 1 行：

```js
console.log(`[PIPELINE:${stage}] ${mod.name} ${changed ? "✓" : "-"}`);
```

から、こういうトレースが流れます：

![consolelog-screenshot](/images/consolelog-screenshot.png)

`✓` = このモジュールが実際にテキストを変えた / `-` = 走ったが変えなかった。

これは debug toy ではなく、**execution observability** という第一級の性質として置いています。compiler の pass trace、ffmpeg の filter chain、build system のステップログ — ああいうレイヤーと同じ意図のものです。

そうしておくと、たとえば「重複行が消えなかった」と言われたときに：

- `dedupe ✓` 出てるけど一部残ってる → 比較ロジックの問題
- `dedupe -` が出てる → モジュールは走ったが何も検出しなかった
- そもそも `dedupe` が出てない → state が off、もしくは bypass パスに乗ってる

…と、**事象がどの層で起きたかが trace から特定できる**。「LLM に頼んだら何かいい感じになった/ならなかった」という不可説明な世界とは、そもそも違うレイヤーで動いてます。

| | AI ラッパー | kiln |
|---|---|---|
| 出力 | stochastic | deterministic |
| 内部 | opaque | observable |
| 失敗時 | "trust me" | pipeline trace |
| 系譜 | LLM API | TeX / Unix pipe / compiler |

**predictability > intelligence** — これが kiln の核にある preference。AI を入れて「なんとなくいい感じ」にする方向ではなく、explicit な op を直列に走らせて inspectable な結果を出す方向です。地味ですが、業務の "配管" 層には、地味であることが効きます。

## まとめ

- ChatGPT 出力 / 箇条書きの崩れ / 重複行 — どれも結局「機械的に正規化できる問題」
- LLM ではなく **決定論的なパイプライン** で処理する方が業務には合う
- モジュールを組み合わせる形にすると、ケースが増えても拡張しやすい

ツールはここで触れます（ブラウザ完結）：

🔥 **[kiln — テキスト整形ツール](https://kiln.ooo/)**

PDF / DOCX / HTML / EPUB をそのまま流し込めるので、コピペ整形のひと手間も省けます。

関連記事：

1. [[LLMなし個人開発] 観察可能な決定論的 text-cleanup pipeline をブラウザで実装した](https://qiita.com/recdnd/items/f35383d3a34f4df8ff72#ui-%E3%82%82-pipeline-%E3%81%A8%E3%81%97%E3%81%A6%E8%A8%AD%E8%A8%88%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B)
2. 現在、kiln の boundary testing 用に weird text corpus を収集中です。  
   broken bullet / invisible char / PDF collapse / Unicode horror など歓迎です 😭  
   [[🔬 boundary testers thread] deterministic cleanup edge cases](https://zenn.dev/recdnd/scraps/f901d5d96526cc)

---

**次回予告**：議事録の文字起こし → ChatGPT 整形 → kiln 後処理 を 1 つのワークフローにまとめた話を書く予定です。フィードバック・要望あれば気軽にコメントください 🙏
