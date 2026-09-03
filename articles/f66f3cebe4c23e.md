---
title: "Node.jsのfetchは「タイムアウトしない」わけではない"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript","nodejs"]
published: false
---
# Node.jsのfetchは「タイムアウトしない」わけではない

※この記事は昨日気付いた表記の件について、ほとんどAIにまとめてもらったものです

## 「fetchにはタイムアウトがない」はブラウザの話

「fetch APIにはタイムアウトオプションがない」という話をよく見かけます。これはブラウザの話であり、Node.jsには当てはまりません。

Node.jsの`fetch`は[undici](https://undici.nodejs.org/)というHTTPクライアントベースで実装されており、**デフォルトで5分（300秒）のヘッダタイムアウト**が設定されています。

### 実演

サーバーが5分10秒（310秒）応答しないAPIを用意します。

https://github.com/marudedameo2019/the-fetch-timeout/blob/main/server.js

ブラウザから http://localhost:3000/ にアクセスすると、310秒後に「hello」が表示されます（タイムアウトなし）。

一方、Node.jsからfetchすると：

https://github.com/marudedameo2019/the-fetch-timeout/blob/main/cli.js

```
$ node cli.js
fetch failed 300790ms
```

5分でタイムアウトします。

## タイムアウトを変更する方法

undiciの`Agent`（Dispatcher）を渡すことで、fetchごとのタイムアウトを変更できます。

```js
import { fetch, Agent } from "undici";

const dispatcher = new Agent({ headersTimeout: 0, bodyTimeout: 0 });

const r = await fetch("http://localhost:3000/api/hello", { dispatcher });
const s = await r.text(); // "hello"（310秒後）
```

^[ただし`fetch`は`undici`のモノを使っています]

`headersTimeout: 0`で無制限になります。

## バージョン不整合の問題

ここで問題があります。Node.jsのグローバル`fetch`は**Node.jsに同梱された内部undici**で動いており、npmからインストールしたundiciとは別物です。

```
$ node -e "console.log(process.versions.undici)"
6.27.0
```

^[nodejs 22の場合]

npmのundici v8の`Agent`を、内部undici v6.27.0のグローバル`fetch`に渡すと：

```
InvalidArgumentError: invalid onRequestStart method
```

というエラーになります。バージョンが合わないだけで動かさないのです。

### 整合を保つには

`fetch`と`Agent`を**同じundiciパッケージからimport**すれば整合が取れます。

```js
import { fetch, Agent } from "undici"; // 同じバージョンなので問題なし

const dispatcher = new Agent({ headersTimeout: 0, bodyTimeout: 0 });
const r = await fetch(url, { dispatcher });
```

この場合、グローバル`fetch`（内部undici）とは無関係に、npmのundici自己的なfetchが完結するので、Node.jsのバージョンに依存しません。

### 複数バージョンで比較テストする場合

npmのalias機能を使えば、同じパッケージの複数バージョンを別名でインストールできます。

```bash
npm install undici6@npm:undici@6.27.0 undici8@npm:undici@8.10.1
```

```js
import { fetch as fetch6, Agent as Agent6 } from "undici6";
import { fetch as fetch8, Agent as Agent8 } from "undici8";
```

実際に3つのケースを試してみました：

| ケース | 組み合わせ | 結果 |
|--------|-----------|------|
| バージョン不整合 | fetch6 + Agent8 | `invalid onRequestStart method`（即エラー） |
| バージョン一致 | fetch6 + Agent6 | OK: hello (310065ms) |
| 最新で揃え | fetch8 + Agent8 | OK: hello (310016ms) |

## 次のリスク：挙動への暗黙の依存

ここまでの話はまだ「知っている人は対策できる」範囲です。もしあると厄介なのは、**コードがfetchのデフォルト挙動を前提に設計されている場合**です。

例えば：

- 「fetchは5分でタイムアウトするから、その後にリトライロジックを発火させる」
- 「5分以内に返ってこなかったら代替APIを使う」
- 「タイムアウトしたとみなしてユーザーにエラー画面を出す」

このような実装は、undici/Node.jsの将来のバージョンでデフォルト値が変わったら**サイレントに壊れます**。APIの仕様（引数や返り値）ではなく、実装の挙動（デフォルトのタイムアウト値）に依存しているため、型検査でも静的解析でも検出できません。

逆に「fetchにはタイムアウトがない」と信じている開発者が書いたコードに、実は5分のタイムアウトが隠れていて、それがいつか障害として顕在化することもあるでしょう。

## まとめ

| 項目 | ブラウザ | Node.js |
|------|---------|---------|
| fetchのタイムアウト | なし | 5分（ヘッダ） |
| 変更方法 | なし | undiciのDispatcher |
| 標準APIで変更可能か | - | 不可（undiciが必要） |
| 挙動变更のリスク | 低い | 高い（内部undiciに依存） |

Node.jsでfetchを使う際は、「タイムアウトがない」前提で設計しないこと、そしてデフォルト値に依存した実装をしないことが重要だと思います。
