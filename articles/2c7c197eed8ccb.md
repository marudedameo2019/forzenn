---
title: "node.jsで待機中の非同期処理を一覧する"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nodejs", "async", "debug"]
published: true
---

**この記事はほとんどAIに作成してもらっています**。

# 問題提起

非同期処理を書いていると、何かどこかが終わっていなくて、プログラムが終了できないことがあったりします。

例えばこんな例…

```js:problem.mjs
import readline from 'node:readline/promises';

const delay = ms => new Promise(resolve => setTimeout(resolve, ms));
const p = delay(10000); // ココ

const askQuestion = async () => {
  let rl;
  try {
    rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout
    });
    const answer = await rl.question("一行入力: ");
    console.log(`読み込み結果: ${answer}`);
  }
  finally {
    if (rl) rl.close();
  }
};

await askQuestion();
console.log("終了");
```

コンソールから1行読み込む処理ですが、その前にawaitしていない行儀の悪い非同期処理がポツンと1つ残っています。
そのため、このコードは終了表示してから最大10秒しないと実際には終わりません。

このコードは短いのですぐにどこに問題があるか分かりますが、通信処理など待ち合わせが多く、比較的規模の大きいプログラムを書いていると、どこで非同期処理が動いているのか把握しにくくなるものです。自分で書いたものならともかく最近だと特に他人やAIが書いたコードを読むことなども多いと思います。そんなときどうやってasync/awaitなどで並行して動作している処理を把握すればいいのでしょうか？

# 解. async_hooksを使用する

並列処理と同様、スレッド一覧ならぬ、非同期処理(リソース)一覧を見る方法です。非同期処理というのは本質的には外部の待ち合わせが必要な状況である間に他の処理を進めることです。外部の待ち合わせには原則待ち合わせ対象となるリソースが必要で、タイマーなり、ファイルなり、ソケットなり、メモリ以外のリソース(デバイス)が該当します。

Node.jsの非同期処理はそれぞれ`asyncId`というIDが振られていて、`node:async_hooks`モジュールを使えばこれらの非同期処理の誕生・死滅のタイミングでコールバックを受け取れます。これをうまく使えば「待機中の非同期処理の一覧」が作れます。

## [async_hooks](https://nodejs.org/api/async_hooks.html)の仕組み

`createHook`は非同期リソースのライフサイクルに紐づくコールバックを登録します。

```js
import { createHook } from 'node:async_hooks';

const hook = createHook({
  init(asyncId, type, triggerAsyncId) {
    // 新しい非同期処理が生まれたとき
    // typeは'Timeout'や'PROMISE'、'TCPWRAP'、'TTYWRAP'、'PIPEWRAP'などの文字列
    // triggerAsyncIdは、この誕生を引き起こした非同期処理のasyncId
  },
  before(asyncId) {
    // 非同期処理が実行される直前
  },
  after(asyncId) {
    // 非同期処理の実行が終わった直後
  },
  destroy(asyncId) {
    // 非同期処理が破棄されたとき
  },
});

hook.enable();
```

注意: `before` / `after` / `destroy`は`asyncId`しか受け取らず`type`は付きません。typeを知りたければ、`init`で自分で作ったテーブルにasyncIdから引き出す必要があります。

## 待機中の非同期処理を追跡する

考え方としては、

- `init`でリソースをMapに記録しておく(その際のスタックトレースも記録)
- `destroy`でMapから削除する
- Mapに残っているものが「待機中の非同期処理」

です。beforeとafter(とresolved)は使い道がないわけではないのですが、各イベントによる待ち合わせ状態の変更がリソースによって違うため、ここでは使用しません。

では先ほどの問題提起の例に、トラッカーと一覧を表示する`listPending`関数を足してみます。

```js
import { createHook } from 'node:async_hooks';
import readline from 'node:readline/promises';

const pending = new Map();

const hook = createHook({
  init(asyncId, type, triggerAsyncId) {
    if (type === 'PROMISE' || type === 'TickObject') return; // ノイズになるのはスキップ
    pending.set(asyncId, { type, stack: new Error().stack });
  },
  destroy(asyncId) {
    pending.delete(asyncId);
  },
});

hook.enable();

const delay = ms => new Promise(resolve => setTimeout(resolve, ms));
const p = delay(10000); // ココ

const askQuestion = async () => {
  let rl;
  try {
    rl = readline.createInterface({
      input: process.stdin,
      output: process.stdout
    });
    const answer = await rl.question("一行入力: ");
    console.log(`読み込み結果: ${answer}`);
  }
  finally {
    if (rl) rl.close();
  }
};

const listPending = () => {
  console.log('=== 待機中の非同期リソース ===');
  for (const [id, info] of pending) {
    console.log(`[asyncId=${id}] ${info.type}`);
    for (const line of info.stack.split('\n').slice(1)
      .filter(l => !/node:internal|async_hooks|AsyncHook\.init/.test(l))) {
      console.log(`    ${line}`);
    }
  }
  console.log('==========================');
};

await askQuestion();
console.log("終了");
listPending();
```

これを実行して`hello`と入力すると…(パスは省略しています)

```text
一行入力: 読み込み結果: hello
終了
=== 待機中の非同期リソース ===
[asyncId=3] Timeout
        at setTimeout (node:timers:135:19)
        at file:///.../demo.mjs:18:44
        at new Promise (<anonymous>)
        at delay (file:///.../demo.mjs:18:21)
        at file:///.../demo.mjs:19:11
[asyncId=5] TTYWRAP
        at new ReadStream (node:tty:58:15)
        at askQuestion (file:///.../demo.mjs:25:22)
        at file:///.../demo.mjs:48:7
[asyncId=6] TTYWRAP
        at new WriteStream (node:tty:95:15)
        at askQuestion (file:///.../demo.mjs:26:23)
        at file:///.../demo.mjs:48:7
[asyncId=7] SIGNALWRAP
        at process.emit (node:events:531:35)
        at _addListener (node:events:562:14)
        at process.addListener (node:events:611:10)
        at askQuestion (file:///.../demo.mjs:26:23)
        at file:///.../demo.mjs:48:7
==========================
```

「終了」が表示されたのに、`Timeout`がまだ残っていることが分かります。スタックトレースを見ると、18行目の`delay(10000);`で生まれたものであることが分かります。問題提起で述べた、プロセスを終わらせない犯人そのものです。

`PIPEWRAP`の2つは、`readline.createInterface`でstdin/stdoutにアクセスした際(25・26行目)に生まれたリソースです。この種のstdio系リソースは長い間(往々にしてプロセスの生存期間中ずっと)リストに残り続けますが、プロセスの終了は妨げません。実際、この例ではTimeoutが発火した瞬間にちょうどプロセスは終了します。`SIGNALWRAP`はCtrl+Cなどのシグナルを待ち合せるもので、これもプロセス終了を妨げません。

### 注意点

**1. 非推奨APIである**

残念ながら…しかし私が使っているnode.js 22だと他の代替手段がありません。

**2. フックのコールバック内で同期的に非同期処理を生み出さない**

例えば`init`の中で`console.log`を書き込もうとしても、初回の`console.log`はstdoutのパイプ(これも非同期リソース)を作成するので、それがまた`init`を呼び、さらに`console.log`して…と無限再帰に陥り、スタックオーバーフローでクラッシュします。安全なパターンは、フックの中ではMapのようなデータ構造に記録するだけにして、表示は外でするというものです。

**3. `PROMISE`などのノイズになるリソースはスキップする**

Promiseのリソースはresolveされてもdestroyされないようで、残しておくとMapがどんどん増える一方です。イベントループの生存を保つものでもないですから、この用途では無視して差し支えありません。

**4. `destroy`は思ったより少し遅い**

タイマーの`destroy`はコールバックの実行が終わった後に発火します。同じタイマーのPromiseチェーンの続きの中で一覧を取ると、そのタイマー自体がまだリストに載っていることがあります。大きな問題ではありませんが、驚かないように。

**5. 性能コストがある**

あらゆる非同期処理でコールバックが呼ばれるので、何らかの減速は避けられません。本番コードに常駐させるものではなく、デバッグのために一時的に取り付けるツールとして使うのが良いです。

**6. deno/bunなど他のランタイムでは使えない**

多くの場合、こういう挙動不審な現象が起きるのはnode.jsではありません。bunなど~~不安定な~~速度重視なランタイムを使う~~安定志向でない~~挑戦的なプログラムで起きがちです。しかし、これらのランタイムではasync_hookは使用できず、代替手段もありません。

### 補足

自分で書かなくても[wtfnode](https://www.npmjs.com/package/wtfnode)などを使っても同じようなことが出来ます。

# まとめ

- `node:async_hooks`を使えば、`asyncId`単位で非同期処理の誕生・死滅を追跡できる
- `init`で記録し`destroy`で削除すると、「待機中の非同期処理の一覧」が得られる
- 作成時のスタックトレースも記録しておくと、問題の処理がどこで生まれたか分かる
- async_hooksには性能コストがあるため、デバッグ用に一時使うのが望ましい
- **非推奨APIであり、他のランタイムでは使用できません**
