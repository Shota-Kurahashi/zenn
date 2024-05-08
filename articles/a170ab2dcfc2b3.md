---
title: 'ファイル名をindexにするとディレクトリ構成が綺麗になる話'
emoji: '🗂️'
type: 'tech' # tech: 技術記事 / idea: アイデア
topics: ['javascript', 'react', 'nextjs', 'typescript', 'frontend']
published: false
---

こんにちは！
フロントエンドの開発中に、ファイル名を`index`にすると`test`のファイルや`stories`のファイルなどコンポーネントに関連するファイルをいい感じにまとめることができるという話をしたいと思います。

## ディレクトリ構成

例えば、button コンポーネントを作成して、それらに関連するファイルをまとめると以下のようなディレクトリ構成になります。

```bash
# Good
components/
  button/
    index.tsx
    index.test.tsx
    index.stories.tsx

# Bad
components/
  button/
    button.tsx
    button.test.tsx
    button.stories.tsx

# Bad
components/
    button.tsx
    button.test.tsx
    button.stories.tsx
```

## なにがいいのか

### 1. import した際のファイルパスが短くなる

ファイル名を index ではない場合、以下のようにファイルパスが長くなります。

```ts
// Bad
import { Button } from '@/components/button/button';
```

しかし、ファイル名を`index`にすることで以下のようにファイルパスが短くなります。

```ts
// Good
import { Button } from '@/components/button';
```

### 2. 機能ごとにファイルをまとめることができる

これはファイル名を`index`にしたからというわけではなく、機能ごとに関連したファイルをまとめることができるという点です。
基本同じディレクトリ内のファイルを編集した場合、それらに関連したファイルも編集することが多いと思います。
そのため、関連したファイルをまとめることで、ファイルを探す手間を減らすことができます。

### 3. ファイル名が統一されており綺麗

これは個人的な感覚ですが、ファイル名が統一されていると綺麗に見えると思います。

## デメリット

ディレクトリ構成がきれいになるというメリットがある一方で、デメリットもあります。

### 1. フレームワークでファイル名が決められている場合

Next.js などではルーティングとファイル名が紐づいているため、ファイル名を`index`にすることができない場合があります。この場合は、仕方がないのでファイル名を`index`にすることができないです。

### 2. ディレクトリ構造が深くなる

新しい機能や同じ拡張子のファイルを追加する場合、毎回ディレクトリを作成する必要があるため、ディレクトリ構造が深くなる可能性があります。
その場合、例外として`index`以外のファイル名を使用する手がありますが、、、

## 自分がよく使うディレクトリ構成

最後に 自分が Next.js の App Router を使用している場合のディレクトリ構成を紹介します。

```bash
├── __tests__ # テストファイルのセットアップ用
├── e2e # e2eテスト
├── public # 静的ファイル
├── src
│   ├── app
│   │   ├── (apps) # ルーティング
│   │   │   ├── _features # ページでしか使わないコンポーネント
│   │   │   │   └── hoge
│   │   │   │       ├── api # データ取得のファイル
│   │   │   │       ├── components # コンポーネント (index.tsx)が配置されている。
│   │   │   │       ├── mocks # APIのモックデータ
│   │   │   │       └── utils # ユーティリティ
│   │   │   ├── layout.tsx
│   │   │   └── page.tsx # トップページ
│   │   ├── layout.module.scss # レイアウト用のスタイル
│   │   └── layout.tsx #グローバルレイアウト
│   ├── components # 共通コンポーネント (ui配下以外はスタイルを持つことができる)
│   │   ├── checkbox
│   │   │   ├── index.stories.tsx
│   │   │   ├── index.test.tsx
│   │   │   └── index.tsx
│   │   ├── loader
│   │   │   ├── index.stories.tsx
│   │   │   ├── index.test.tsx
│   │   │   └── index.tsx
│   ├── layouts #グローバルやページごとのレイアウト
│   │   ├── header
│   │   │   ├── index.stories.tsx
│   │   │   └── index.tsx
│   │   ├── index.module.scss
│   │   ├── index.stories.tsx
│   │   └── index.tsx
│   ├── libs # ライブラリや独自のユーティリティを配置
│   │   ├── errors # カスタムのエラーを配置 エラーはここのファイルを使用してエラーを生成する。
│   │   │   ├── index.test.ts
│   │   │   └── index.ts
│   │   └── types # 共通の型を配置
│   │       ├── api
│   │       │   └── hoge
│   │       │       └── index.ts
│   │       └── next # Next.jsの型
│   │           └── index.ts
│   └── tests # vitestに関連しないテストの設定ファイル
└──       └── index.ts
```

### ディレクトリのルール

- ルーティングは基本的に`/app/(apps)`に配置をする。他の group にしたい場合は、`/app/(group name)`に配置する。

- フォルダを作成し`index`を prefix にしてファイルを作成する。

  - (例: `index.tsx` `index.module.scss` `index.stories.tsx`)

- `page`と`layout`は Next.js の関係上名前を変更できないので、それに付随して関連するファイルも prefix を`page.`や`layout.`にする。

  - (例: `page.tsx` `layout.tsx` `layout.module.scss`)

- そのルーティングでしか使わないものは`page.tsx`と同階層の`_features`に配置する。

- `_features`の中には`(フォルダ名)/components`や`(フォルダ名)/hooks`などを配置する。

- 共通コンポーネントは`/components`に配置する。

## おわりに

ファイル名を`index`にすることで、ディレクトリ構成が綺麗になるという話をしました。
正直、ディレクトリ構成が綺麗になるという個人的な感覚が強いですが、ファイルパスが短くなるというメリット？もあるので、よかったら試してみてください！
