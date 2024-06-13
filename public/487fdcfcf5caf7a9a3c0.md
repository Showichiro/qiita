---
title: JavaScriptの便利な記法や関数の紹介およびそれらの注意点について
tags:
  - JavaScript
  - TypeScript
private: false
updated_at: '2024-06-07T09:11:39+09:00'
id: 487fdcfcf5caf7a9a3c0
organization_url_name: primebrains
slide: false
ignorePublish: false
---
# はじめに

JavaScript初学者を抜けたあたりの方にむけて、便利な記法や関数、その注意点について紹介します。  
初歩的な文法やデータ型などの知識は前提として解説を省きます。  
JavaScriptの巨大なテーマとしては非同期処理などもあるのですが、巨大すぎるために本稿では割愛させていただきます。

## let/constの使い分けについて

変数は不変な`const`および可変な`let`を利用することができます。原則的には`const`を使い、再代入が必要な個所のみ`let`を使うのが標準的です。
`let`を利用している時点で **「処理のどこかで再代入される」** と処理内容の推論を働かせてコードを読む人が多いと思います。このようなコードの読み方をするという前提を踏まえてコードの可読性を高めるうえでも、再代入されるかされないかを意識して`const`/`let`を使い分けることが重要です。

```js
const name = "John Doe";
const age = 25;
```

```js
let name = "Jane Doe";
let age = 25;
```

## アロー関数

`function`を利用した関数の簡易的な代替構文で、一部仕様の違いがあります。

```js
function Foo() {}
const Foo = () => {}
```

具体的にはアロー関数はthisへのバインドがない(functionを利用する場合とthisの挙動が異なる)などの違いがあります。  
モダンな環境ではどちらでもよい(ので一貫性や簡潔さのためにアロー関数にそろえる)ということが多いと思いますが、ライブラリによってはfunctionでなければならないケースも存在する(例えば内部的にthisを利用するため)ので要注意です。

## 分割代入

変数を宣言する、配列やオブジェクトからデータを取り出して代入するという操作を簡潔に表現する記法です。

```js
// 配列
const [first, second, , fourth] = [1, 2, 3, 4]; // first = 1, second = 2, fourth = 4

const person = { name: "John Doe", age: 25 };
const { name, age } = person; // name = "John Doe", age = 25
```

## スプレッド構文

複数の引数や要素を配列やオブジェクトして扱うことで、より簡潔に表現できる記法です。以下のようなケースで利用できます。

### 配列の操作

配列の抽出や結合などに使うことができます。
```js
const [first, second, ...rest] = [1, 2, 3, 4, 5, 6, 7] // first = 1, second = 2, rest = [3, 4, 5, 6, 7]
const array = [...rest, 8, 9] // array = [3, 4, 5, 6, 7, 8, 9]
```

これらを活用することにより、pushやpopなどの破壊的な操作をするメソッドを避けることができます。

例えば、

```js
const array = [1, 2, 3, 4, 5];
array.push(6);
```

というコードを

```js
const array = [1, 2, 3, 4, 5];
const pushed = [...array, 6];
```

と書くことができます。

可変長引数の関数の引数も扱うことができます。
```js
const foo = (a, ...rest) => {
  console.log(a);
  console.log(rest);
}

foo(1, 2, 3, 4, 5); // a = 1, rest = [2, 3, 4, 5] 
```

第n引数までが特別な意味を持つような関数などで見通しよくコードを書くことができます。

### オブジェクトのプロパティの抽出・代入

```js
const person = { name: "John Doe", age: 25, role: "admin" }
const { name, ...rest } = person; // name = "John Doe" rest = {age: 25, role: "admin"}
const person2 = {...rest, name: "Jane Doe" }
```

注意点としては同じプロパティ名の代入をする場合はあと勝ちになることです。

```js
const person = { name: "John Doe", age: 25, role: "admin" }
const person2 = { name: "Jane Doe", ...person } // あと勝ちで、{ name: "John Doe", age: 25, role: "admin" }になる。
```

## テンプレート文字列

バッククオートで囲うことで文字列をテンプレートとして利用できます。`${}`を利用することで変数を埋め込むこともできます。`+`による文字列連結より可読性が高いです。

```js
const name = "John Doe";
const message = `Hello, ${name}!`; // Hello, John Doe!
```

URLの整形など文字通りテンプレートがある場合に強力に作用します。

```js
const url = `https://${domain}`;
```

## オプショナルチェーン

JavaScriptはチェーン(`.`)でプロパティにアクセスすることができます。また、存在するかわからない(undefined/nullの可能性がある)プロパティに対しては`?`を利用することで安全に(存在しない場合は実行時エラーではなくundefinedを返す)アクセスできます。関数に対しても利用できます。

```js
const person = {
  name: "John Doe",
  age: 25,
  cat: {
    name: "Tom"
  }
};

const dogName = person?.dog?.name; // dogName = undefined
person?.someMethod?.(); // undefined
```

## null合体演算子(??)の短絡評価を利用した代入

論理演算子の一つである`??`はnullishな値(nullまたはundefined)の場合に次(右辺)の値を代入するのでnullishな値の場合はデフォルトの値を代入するという処理を簡潔に表現できます。

このようなコードを
```js
let name = "John Doe";
if(person.name != null) {
  name = person.name;
}
```
以下のように書くことができます。

```js
const name = person?.name ?? "John Doe";
```

## 論理和の短絡評価を利用した代入

同様に、論理和(`||`)を利用することでfalsyな値の場合にデフォルト値を代入するという処理を簡潔に表現できます。

このようなコードを
```js
let name = "John Doe";
if(person.name !== "") {
  name = person.name;
}
```

以下のように書くことができます。

```js
const name = person.name || "John Doe";
```

`??`と`||`の違いとしてはnullishかfalsyな値かという違いがあります。JavaScriptは暗黙にキャストする仕様であるため、例えば、空文字などがfalseに変換されえます。このように暗黙のキャストでfalseになりうる値のことをfalsyな値と呼びます。つまり、`??`は空文字などはtrue判定(「nullまたはundefined」ではないため)となるのに対して`||`はfalseとして次の値を参照するということです。

ただし、「どのような値がそもそもfalsyになるのか」というJavaScriptの細かめな知識が求められるという意味で、falsyであることを前提としたコーディングを排除しよう(eslintで禁止しよう)という意思決定もあり得ます。このあたりは各プロジェクトのルール次第かと思います。

## 配列操作

### map: 配列の置換処理

配列の各要素に共通の処理を施して変換をかける場合は、`map`メソッドが有用です。`map`は各要素の処理を関数として渡します。

```js
const array = [1, 2, 3, 4, 5];
const square = array.map(val => val * val); // [1, 4, 9, 16, 25]
```

indexを取りたい場合は第二引数からindexを取ることができます。

```js
const array = [1, 2, 3, 4 ,5];
const mapped = array.map((val, index) => val * index);
```

### filter

配列の各要素に対して絞り込みの処理をかけたい場合は`filter`メソッドが有用です。`filter`に渡した処理の結果がtrueな要素だけが残ります。

```js
const array = [1, 2, 3, 4, 5];
const evenArray = array.filter(val => val % 2 === 0); // [2, 4]
```

indexを取りたい場合は第二引数からindexを取ることができます。

```js
const array = [1, 2, 3, 4, 5];
const filtered = array.filter((val, index) => val === index);
```

### reduce

配列の値を加工して特定の処理結果を得たい場合は、`reduce`メソッドが有用です。少しわかりづらいのですが、第一引数に直前の値と現在の値を引数にとり、第二引数に初期値を与えます。

```js
const array = [1, 2, 3, 4, 5];
const sum = array.reduce((prev, current) => prev + current, 0); // 15
```

ただし、reduceは複雑な処理を実装すると途端に可読性が下がる可能性があり、注意が必要です。

そのほか、配列をflatにする`flat`や`flat`しつつ配列の置換も行う`flatMap`など配列周りの便利なメソッドが様々存在します。
