---
title: Denoで型レベルプログラミングの単体テストを書く
tags:
  - JavaScript
  - TypeScript
  - 型レベルプログラミング
  - Deno
private: false
updated_at: '2024-06-28T19:41:35+09:00'
id: bd9f7f47c5fec2260dcf
organization_url_name: primebrains
slide: false
ignorePublish: false
---
# はじめに

Deno公式が公開しているライブラリ `@std/testing`で、Deno環境では型レベルプログラミングのユニットテストを書くことができると知ったので方法を共有します。  

手前味噌ながら型レベルプログラミングについては別の拙稿にて紹介していますのでご参照ください。

https://qiita.com/umiushi_1/items/4aa25df3343c5b2be62d


# 環境構築

Denoを利用できるセットアップが必要です。

https://docs.deno.com/runtime/manual/getting_started/installation

上記のページに従い、インストール等を行ってください。

# 作業ディレクトリを切る

```sh
# 各自の所定の作業ディレクトリから
mkdir deno-type-test
cd deno-type-test
deno init
```

```console
✅ Project initialized

Run these commands to get started

  # Run the program
  deno run main.ts

  # Run the program and watch for file changes
  deno task dev

  # Run the tests
  deno test
```

上記のように表示されればOKです。

ちなみにフォルダ内は以下のようになっています。

```console
.
├── deno.json
├── main.ts
└── main_test.ts
```

# 実際に型レベルプログラミングを行う

```ts:main.ts
/**
 * 型引数に配列を渡したときに、渡した要素の型を返す型レベルプログラミング.
 */
export type ArrayOf<T extends unknown[]> = T extends (infer E)[] ? E : never;

```

これに対するテストを書いてみましょう。まず、テスト用にパッケージを追加します。

`@std/testing`は[JSR](https://jsr.io/@std/testing)で公開されていますので、`deno add`コマンドで追加することができます。

```sh
# deno-type-testにいる状態で
deno add @std/testing
```

以下のように追加された旨コンソール表示されます。

```console
Add @std/testing - jsr:@std/testing@^0.225.3
```

# テストを書く

では実際にテストを書きます。
`@std/testing/types`の`AssertTrue`と`IsExact`を利用します。
`AssertTrue`は型引数の型がtrueかを判定します。
`IsExact`は二つの型引数を取り、それらが一致するかを判定します。

参考）

https://jsr.io/@std/testing@0.225.3/doc/types/~/AssertTrue

https://jsr.io/@std/testing@0.225.3/doc/types/~/IsExact


```ts:main_test.ts
import type { AssertTrue, IsExact } from "@std/testing/types";
import { ArrayOf } from "./main.ts";

Deno.test("ArrayOf", async (t) => {
  await t.step("Tがnumber[]のとき、型がnumberとなること", () => {
    type Expected = number;
    type Actual = ArrayOf<number[]>;
    type _ = AssertTrue<IsExact<Expected, Actual>>;
  });

  await t.step(
    "Tが(string | number)[]のとき、型がstring | numberとなること",
    () => {
      type Expected = string | number;
      type Actual = ArrayOf<(string | number)[]>;
      type _ = AssertTrue<IsExact<Expected, Actual>>;
    }
  );

  await t.step("Tが ('a' | 1)[]のとき、型が 'a'|1となること", () => {
    type Expected = "a" | 1;
    type Actual = ArrayOf<("a" | 1)[]>;
    type _ = AssertTrue<IsExact<Expected, Actual>>;
  });
});

```

# テストを実行する

```sh
# 所定のディレクトリ上で
deno test
```

```console
Check file:///path/to/deno-type-test/main_test.ts
running 1 test from ./main_test.ts
ArrayOf ...
  Tがnumber[]のとき、型がnumberとなること ... ok (1ms)
  Tが(string | number)[]のとき、型がstring | numberとなること ... ok (0ms)
  Tが ('a' | 1)[]のとき、型が 'a'|1となること ... ok (0ms)
ArrayOf ... ok (2ms)

ok | 1 passed (3 steps) | 0 failed (3ms)
```


以上です。
ただし、カバレッジは取れないようです。

# おわりに

今回はDeno環境で型レベルプログラミングの単体テストを書く方法について紹介しました。
上記の方法の場合はDeno環境に限定されますが、IsExactの実装は公開されていますし、LICENSEもMITとなっていますので、同様の型定義を利用してNode環境などでも型レベルプログラミングの単体テストを実施することが可能だと思います。

https://jsr.io/@std/testing/0.225.3/types.ts#L162

ちなみにJSRは(ESM前提なら)Node環境などにも依存関係として追加できますが、このライブラリはDeno環境での利用前提のためうまく動きません。
