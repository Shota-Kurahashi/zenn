---
title: "[React] Comlinkを使ってiframeと通信する"
emoji: "✨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [react, iframe, comlink]
published: false
publication_name: "chot"
---

## Comlink とは

https://github.com/GoogleChromeLabs/comlink

Comlink は、Web Worker などの異なるコンテキスト間での通信を簡単にするためのライブラリです。

公式では、Web Worker を enjoyable なものにするためのライブラリと紹介されています。

> Comlink makes WebWorkers enjoyable. Comlink is a tiny library (1.1kB), that removes the mental barrier of thinking about postMessage and hides the fact that you are working with workers.

`1.1kB` と非常に軽量なのも嬉しいですね。

本来は、`postMessage`を通してメッセージを送信する必要がありますが、Comlink を使うことで、以下のサンプルのように、あたかも別のコンテキストの関数を実行しているかのように扱うことができます。

### サンプル

```ts
// worker.ts
import * as Comlink from "comlink";
const handlers = {
  add: (a: number, b: number) => a + b,
};
expose(handlers);
```

```tsx
// main.ts
import * as Comlink from "comlink";

const handlers = Comlink.wrap(new Worker("worker.js"));

// worker.ts で定義した関数を呼び出したかのように使用できる
const result = await handlers.add(1, 2);

console.log(result); // 3
```

`expose` でオブジェクトを公開し、`wrap` で `expose` したオブジェクトを取得します。
`wrap`で取得したオブジェクトは全て非同期になります。

本来 Comlink は、Web Worker 間で使用することを想定していますが、iframe 間でも同様に使用することができます。

## iframe 間で Comlink を使う

今回の本題です。

基本的な使い方は同じですが、iframe など他のウィンドウと連携する場合は、次のように`windowEndpoint` を活用します。

```tsx
// main.ts
import * as Comlink from "comlink";

const handlers = Comlink.wrap(Comlink.windowEndpoint(iframe.contentWindow));
```

```tsx
// iframe.ts
import * as Comlink from "comlink";

const handlers = {
  add: (a: number, b: number) => a + b,
};
expose(handlers, Comlink.windowEndpoint(self.parent));
```

iframe 側では、Web Worker のときと同様に`expose` を行い関数を受け取れるようにします。
こちら側でも、同様`windowEndpoint` を使用します。

`windowEndpoint` の詳細は、以下のドキュメントを参照してください。

https://github.com/GoogleChromeLabs/comlink?tab=readme-ov-file#comlinkwindowendpointwindow-context--self-targetorigin--

### オリジンが異なる場合

もし、iframe のオリジンが異なる場合は、`Comlink.windowEndpoint(iframe.contentWindow,self, targetOrigin)` のように、`targetOrigin` を指定します。

```tsx
// main.ts
import * as Comlink from "comlink";

const handlers = Comlink.wrap(
  Comlink.windowEndpoint(iframe.contentWindow, self, targetOrigin)
);
```

### 型をつける

もちろん、`expose` するオブジェクトには型をつけることができます。
iframe 側を別のリポジトリで開発するときなどは、 `turborepo`などのモノレポで開発して、型を共有すると便利です。

```tsx


// type.ts
type Handlers = {
  add: (a: number, b: number) => Promise<number>;
};

// iframe.ts
import * as Comlink from "comlink";

const handlers: Handlers = {
  add: (a: number, b: number) => Promise<number>;
};
expose(handlers);

// main.ts
import * as Comlink from "comlink";

const handlers:Comlink.Remote<Handlers> = Comlink.wrap<Handlers>(Comlink.windowEndpoint(iframe.contentWindow));
```

`wrap` したオブジェクトは、`Remote<T>` 型になります。
`Remote<T>` は、`T` の関数がすべて非同期になるため注意が必要です。

## React で Comlink を使う

ほぼ同じですが、`wrap` したオブジェクトを React の `useRef` で管理します。

```tsx
// main.tsx
import { useRef } from "react";
import * as Comlink from "comlink";
import type { Handlers } from "@/packages/types";

function Example() {
  const workerRef = useRef<Comlink.Remote<Handlers> | null>(null);

  const handleOnLoad = (e: React.SyntheticEvent<HTMLIFrameElement>) => {
    const iframe = e.target;
    if (!iframe) return;

    const contentWindow = iframe.contentWindow;
    if (!contentWindow) return;

    const worker = Comlink.wrap<Handlers>(
      Comlink.windowEndpoint(contentWindow)
    );
    workerRef.current = worker;
  };

  return <iframe onLoad={handleOnLoad} />;
}

// iframe.tsx
import * as Comlink from "comlink";

const handlers = {
  add: (a: number, b: number) => a + b,
};
expose(handlers);
```

`onLoad` イベントで `workerRef.current` に `wrap` したオブジェクトをセットします。
これで、他の箇所でも `workerRef.current` から `expose` したオブジェクトを使用することができます。

### 親側から iframe に `expose`する

上記のサンプルでは、iframe 側で `expose` したオブジェクトを親側で `wrap` していました。

反対に、親側から iframe に `expose` することもできます。

```tsx
// main.tsx
import { useRef } from "react";
import * as Comlink from "comlink";
import type { Handlers } from "@/packages/types";

function Example() {
  const getApiData = async (key: string) => {
    // key に対応するデータをCMSなどから取得
    const data = await getCMSData(key);

    return data;
  };

  const handleOnLoad = (e: React.SyntheticEvent<HTMLIFrameElement>) => {
    const iframe = e.target;
    if (!iframe) return;

    const contentWindow = iframe.contentWindow;
    if (!contentWindow) return;

    // expose する関数の一覧を定義
    const parentHandlers = {
      getApiData,
    };

    Comlink.expose(parentHandlers, Comlink.windowEndpoint(contentWindow));
  };

  return <iframe onLoad={handleOnLoad} />;
}
```

```ts
// iframe.ts
import * as Comlink from "comlink";

const parentHandlers = Comlink.wrap(Comlink.windowEndpoint(self.parent));

const result = await parentHandlers.getApiData("/api/data");
```

このように、親側で定義した関数を iframe 側で使用することができます。
iframe 側から親側の関数を呼び出すことができるので、`postMessage` で実装するのが大変だった処理も簡単に実装できます。

## Comlink.Proxy を使う

Comlink では、**シリアライズ可能なデータ** のみ送信できます。
シリアライズできないデータを送信しようとすると、エラーが発生します。

正確な挙動は、Comlink の GitHub 上に記載されています。

[GitHub - Comlink/structured-clone-table.md](https://github.com/GoogleChromeLabs/comlink/blob/fd4b52666b1ec62784f8b45cb1108c7e40bc481d/structured-clone-table.md)

MDN では、シリアライズ可能なデータの一覧が記載されています。
https://developer.mozilla.org/ja/docs/Glossary/Serializable_object

シリアライズできないデータを送信する場合は、Comlink.Proxy を使ってシリアライズ可能なデータに変換します。

```tsx
// iframe.ts

import * as Comlink from "comlink";
type Handlers = {
  callFunction: (fn: () => void) => void;
};

const handlers: Handlers = {
  callFunction: (fn) => {
    fn();
  },
};

Comlink.expose(handlers);

// main.tsx
const handleOnClick = () => {
  const fuga = () => {
    console.log("fuga");
  };
  workerRef.current?.callFunction(Comlink.proxy(fuga));
};
```

上記のように、`Comlink.proxy` を使ってシリアライズ可能なデータに変換します。

## まとめ

Comlink を使うことで、iframe 間での通信を簡単に行うことができます。

それでは、良い Comlink ライフを！

## 参考

https://github.com/GoogleChromeLabs/comlink
https://developer.mozilla.org/ja/docs/Glossary/Serializable_object
