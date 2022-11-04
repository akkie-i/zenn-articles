---
title: "TypeScriptの型定義まとめ【Reactも対応】"
emoji: "🗂"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "ts"]
published: true
---

# はじめに

プログラムを始めて間もない時に以下のような体験をした人は少なくないのではないでしょうか。

静的型付け言語を始める → むずい。もう嫌 → 動的型付け言語サイコー！→ いや、やっぱ静的型付け言語大事だわ

少なくとも僕はこの一人でした。
加えてこんなことも思ってる人もいるのではないでしょうか。

静的型付け言語やりたいけど、業務で使う言語は動的型付け言語での開発がメイン。かといって今使ってない Java とかをガッツリやるのもなぁ。

なんて思ってたら TypeScript が誕生。歓喜。
よく使う言語を続けつつも静的型付けを取り入れることが可能になりました。

そんなこんなで最近は TypeScript の学習を始めましたのでアウトプットとして型の定義の方法まとめです。

並走で React の学習も進めているので通常の TypeScript の型定義と React 特有の型定義を本記事では取り扱っていければと思ってます。

それではよろしくお願いします。

## この記事でわかること

:::message
- TypeScriptでの型定義
  - プリミティブ
  - リテラル
  - オブジェクト
  - ユニオン
  - 型推論
  - 関数
  - ジェネリクス
  - 列挙
  - クラス
  - エイリアス
- React × TypeScript
  - コンポーネント
  - props
  - イベント
  - useState
:::

## この記事を読む上での前提条件

:::message alert
TypeScript の型に関わる部分しか取り扱いません。
なので、クラスやアロー関数、React の基礎知識、特有の言葉などの説明の補足はしませんのである程度は JavaScript や React を触ったことがあるとスムーズに読めるかと思います。
:::

## この記事で取り扱わないこと

:::message alert

- 実際に触ってもらうための環境構築の手順
- TypeScript の型定義以外の情報(型の中でも主観でよく使うものに絞ってます)
- React のクラスコンポーネントでの記載例
  :::

## 環境情報

- npm: v8.9.0
- node: v18.2.0
- typeScript: v4.4.2
- react: v18.0.0

# プリミティブ値の型定義

TypeScript で扱うことのできるプリミティブ値は = JavaScript のプリミティブ値です。
JavaScript のプリミティブ値は [MDN](https://developer.mozilla.org/ja/docs/Glossary/Primitive) によると「プリミティブはオブジェクトでなく、メソッドを持たないデータのことです。」とのことです。

具体的には以下のような値になります。

- 文字列(string)
- 数値(number)
- 数値で扱えない範囲の数値(bigint)
- 真偽値(boolean)
- null(null)
- undefined(undefined)
- シンボル(symbol)

上記のプリミティブにはそれぞれ()内の型があり、TypeScript を使用することで型の異なる値は相互に代入ができなくなります。

以下サンプルです。それぞれ型の一致する値は問題なく代入できて型の異なる値を代入するとエラーメッセージが出力されます。(エラーメッセージは vscode の自動補完で確認できます)

```js
let str: string = "hello";
str = 0; // Type 'number' is not assignable to type 'string'.

let num: number = 0;
num = "0"; // Type 'string' is not assignable to type 'number'.

let big: bigint = 10n;
big = 0; // Type 'number' is not assignable to type 'bigint'.

let bool: boolean = true;
bool = 1; // Type 'number' is not assignable to type 'boolean'.

let n: null = null;
n = undefined; // Type 'undefined' is not assignable to type 'null'.

let u: undefined = undefined;
u = null; // Type 'null' is not assignable to type 'undefined'.

let sym: symbol = Symbol();
sym = ""; // Type 'string' is not assignable to type 'symbol'.
```

:::details ※ bigint 型で BigInt literals are not available when targeting lower than ES2020 と表示されたら場合
`tsconfig.json`の値を以下に変更する修正します。

```diff json
{
  "compilerOptions": {
-   "target": "es5",
+   "target": "es2020",
    "lib": [
      "dom",
      "dom.iterable",
      "esnext",
+     "es2020"
    ],
    "allowJs": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "strict": true,
    "forceConsistentCasingInFileNames": true,
    "noFallthroughCasesInSwitch": true,
    "module": "esnext",
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "isolatedModules": true,
    "noEmit": true,
    "jsx": "react-jsx"
  },
  "include": [
    "src"
  ]
}
```

:::

# リテラル型定義

プリミティブ値に使用した型の中で更に特定の値のみしか代入できないようにした型をリテラル型と言います。

以下サンプルです。

```js
let str: "hello" = "hello";
str = "world"; // Type '"world"' is not assignable to type '"hello"'.

let num: 123 = 123;
num = 321; // Type '321' is not assignable to type '123'.

let big: 10n = 10n;
big = 20n; // Type '20n' is not assignable to type '10n'.

let bool: true = true;
bool = false; // Type 'false' is not assignable to type 'true'.
```

特定の値のみの型定義って使い道なくね？なんて思うかもしれませんが、以降に紹介します「ユニオン型定義」と組み合わせることで威力を発揮します。

# Object 型定義

Object 定義は基本的にオブジェクトリテラルで `{property: type}`のように定義します。

以下サンプルです。型の違うプロパティ値の代入と定義してないプロパティは追加できないことが確認できます。

```js
const obj: { name: string, age: number } = { name: "taro", age: 20 };
obj.name = 20; // Type 'number' is not assignable to type 'string'.
obj.age = "taro"; // Type 'string' is not assignable to type 'number'.
obj.gender = "male"; // Property 'gender' does not exist on type '{ name: string; age: number; }'.
```

毎度変数のところにオブジェクトリテラルで型を定義するのは冗長で可読性も悪いです。
以降で紹介する「型エイリアス」を組み合わせるとスッキリした Object 型の定義が可能です。

## オプションプロパティ

先ほどのオブジェクトリテラルでの型定義ですが、設定したプロパティを必ず設定しないとエラーになります。

```js
const obj: { name: string, age: number } = { name: "taro" };
// Property 'age' is missing in type '{ name: string; }' but required in type '{ name: string; age: number; }'.
```

例えば、`name`は必ず必要だけど `age`はなくても良い。そんな時にオプションプロパティを使用できます。

以下サンプルです。プロパティ名の後に `?`付与することで任意のプロパティになります。

```js
const obj: { name: string, age?: number } = { name: "taro" };
```

# 配列型定義

配列の型定義にはプリミティブで扱っていた型に `[]`を付けることで定義できます。(以降の「ジェネリクス型定義」でも定義することが可能です)

以下サンプルです。`number[]`では数値のみの配列、`string[]`では文字列のみの配列しか許可しないことを確認できます。

```js
const strArray: string[] = ["a", "b", "c"];
strArray.push(0); // Argument of type 'number' is not assignable to parameter of type 'string'.

const numArray: number[] = [1, 2, 3];
numArray.push("a"); // Argument of type 'string' is not assignable to parameter of type 'number'.
```

従来の JavaScript では配列の中に文字列や数値をごちゃ混ぜで追加するなんてことよくあったことかと思います。
慣れてしまった人にとっては配列の中に一つの型の値しか追加できないのは少々不便に感じるかと思います。

配列に複数の型を追加したい場合は以降の「ユニオン型定義」を参照ください。

# 配列 Object 型定義

配列の中に Object が格納されているパターンです。
この場合は `Objectリテラル[]`の形式で定義すれば OK です。

以下サンプルです。

```js
const obj: { name: string, age?: number }[] = [
  { name: "taro" },
  { name: "hanako", age: 24 },
];
```

# ユニオン型定義

ここまではプリミティブ値に単一の型を、一つの配列に一つの型を定義する方法などを見てきました。
ユニオン型を用いることで複数の型を組み合わせて柔軟な型を定義することができます。

使い方は簡単で型に続けて `|`で繋いでいくだけです。
以下サンプルです。ここまで見てきた型定義にユニオンを組み合わせて複数パターン試してみます。

```js
// プリミティブで複数の型を定義
let strOrNum: string | number;
strOrNum = "string";
strOrNum = 123;
strOrNum = true; // Type 'boolean' is not assignable to type 'string | number'.

// リテラルとの組み合わせての定義
let gender: "male" | "female" | "other";
gender = "male";
gender = "female";
gender = "other";
gender = "test"; // Type '"test"' is not assignable to type '"male" | "female" | "other"'.

// 配列に複数の型を定義
const strOrNumArray: (string | number)[] = ["a", "b", 1, 2];
strOrNumArray.push(true); // Argument of type 'boolean' is not assignable to parameter of type 'string | number'.

// 配列オブジェクトのプロパティに対し複数の型を定義
const persons: { name: string, age: string | number }[] = [
  { name: "taro", age: 20 },
  { name: "hanako", age: "24" },
];
```

また、先頭に `|`をつけてもコード上に問題はないです。
なので見栄え的に以下のように書くことも可能です。

```js
let week:
  | "sun"
  | "mon"
  | "tue"
  | "wed"
  | "thu"
  | "fri"
  | "stu";
```

# 型推論

型推論とは型が明確な時にある程度 TypeScript が自動で変換してくれる機能になります。
通常のプリミティブ値は初期値の値によって暗黙的に型が定義されますし、Object の型推論もある程度は自動でやってくれます。

また、`let`宣言での変数にはプリミティブ値への型推論がされますが、`const`宣言の場合はリテラル型の型推論が行われます。
それぞれ変換された結果が以下のキャプチャになります。

![](https://storage.googleapis.com/zenn-user-upload/a0674e8ac9aa-20221102.jpg)

# 関数の引数の型定義

変数に格納する値に型を定義するだけでなく関数の引数と戻り値にも型を定義することができます。
まずは引数の型定義についてみていきます。

書き方はこれまでと大差ないです。ただ引数の部分に型を定義してあげれば OK です。

以下サンプルです。簡単な足し算をする関数の引数に `number`の型を定義して、型の不一致、引数が足りない場合はあエラーが発生することを確認できます。

```js
const sum = (x: number, y: number) => {
  return x + y;
};

console.log(sum(1, 2));
console.log(sum(1, "2")); // Argument of type 'string' is not assignable to parameter of type 'number'.
console.log(sum(1)); // Expected 2 arguments, but got 1.
```

引数の数が一致しない場合は先ほどのオプションプロパティとして設定するかデフォルト引数を設定することで回避します。

```js
const sum1 = (x: number, y?: number) => {
  return x + y;
};

const sum2 = (x: number, y: number = 0) => {
  return x + y;
};
```

※ オプションプロパティの場合 `y`の引数の値は `undefined`になりますので `number + undefined`で戻り値は `NaN`になり、不具合の原因になりますの使用する際の制御をお忘れなく

もう 1 パターンで可変長引数の場合です。
可変長引数は引数をまとめて配列で受け取るのでこれまでの配列での型定義と同じ方法で指定してあげれば OK です。

```js
const sum = (...numbers: number[]) => {
  return numbers.reduce((accumulator, current) => {
    return accumulator + current;
  }, 0);
};

console.log(sum(1, 2));
console.log(sum(1, 2, 3, 4, 5));
```

# 関数の戻り値の型定義

次に関数の戻り値の型定義です。
先ほど引数の型で書いた `sum()`関数を確認すると戻り値に `number`が指定されていることが確認できます。

![](https://storage.googleapis.com/zenn-user-upload/b46cde903d44-20221102.png)

これは TypeScript の型推論で暗黙的に戻り値が指定されているためです。
ここを暗黙的ではなく明示的に戻り値を指定してみます。

以下サンプルです。アローの前に型を指定してあげるだけで OK です。

```js
const sum = (x: number, y: number): number => {
  return x + y;
};
```

## 戻り値がない場合の void

関数によっては戻り値がないパターンもあります。
そんな時は `void`を指定することでこの関数には戻り値がないと明示的に定義できます。

```js
const logger = (): void => {
  console.log("log");
};
```

## 非同期処理の Promise の戻り値

非同期の関数に戻り値の型を定義する時は `Promise<type>`で指定します。

以下サンプルです。細かい処理は置いておいて非同期関数が終了した結果として戻り値に `string`が返る場合のイメージです。

```js
async function asyncFn(): Promise<string> {
  // 非同期処理
  return "executed";
}

console.log(await asyncFn());
```

## タプル型で Promise の戻り値を複数受け取る

TypeScript にはタプル型という型の順番を定義する方法があります。
以下サンプルは `string` `number`の順番で値が入った配列しか許容しない定義になります。

```js
let tuple: [string, number] = ["hoge", 1];
tuple = [1, "hoge"]; // Type 'string' is not assignable to type 'number'.
```

この性質と `Promise.all()`の非同期処理の戻り値の順番を担保してくれる性質を組み合わせて型定義をするケースなどがあります。

以下サンプルです。

```js
async function asyncFn1(): Promise<string> {
  // 非同期処理
  return "executed";
}

async function asyncFn2(): Promise<number> {
  // 非同期処理
  return 1;
}

const tuple: [string, number] = await Promise.all([asyncFn1(), asyncFn2()]);
```

もはや何を持ってタプル？となりますが可変長引数でもタプル型の定義も可能です。
例えば、戻り値が `string`が返る非同期処理を複数回繰り返す時に以下のような定義が可能です。

```js
const apiReturnValues = ["a", "b", "c"];
const tuple: [...string[]] = await Promise.all(
  apiReturnValues.map(async (v) => {
    // 非同期処理
    return v;
  })
);
```

# ジェネリクス型定義

ジェネリクス型は `<>`の中に型の引数を渡して型の定義ができる方法です。

まずはよく見かける Array のジェネリクス型です。
以下サンプルです。こちらは `string[]`で型定義するのと同義です。

```js
const str: Array<string> = ["a", "b", "c"];
```

ユニオン型を組み合わせて複数の型の配列を生成することもできます。
以下サンプルです。こちらは `(string|number)[]`で型定義するのと同義です。

```js
const strOrNum: Array<string | number> = ["a", "b", 1, 2];
```

次に Promise のジェネリクス型です。
先ほどの間「非同期処理の Promise の戻り値」でも触れてた内容ですが Promise の場合は以下で戻り値の型を定義できます。

```js
const promise: Promise<string> = new Promise((resolve, reject) => {
  resolve("test");
});
```

最後に型に `<S, T, U>`のように一時的な型引数の名前を付け、関数側では具体的な型の定義を持たず、関数の呼び出し元で型を指定する方法です。

使用例を確認してみます。まず与えた引数をもとに `string[]`の配列を生成してリターンするという関数を作成してみます。

```js
const addKeys = (key1: string, key2: string): string[] => {
  return [key1, key2];
};

addKeys("a", "b");
```

ではこちらと同じロジックで `number[]`と `boolean[]`と複数組み合わせで配列を生成する関数を作成してくださいと言われました。

引数と戻り値の型を変えただけの同じロジックの関数を複製するのも一つの手段ですが、やや冗長です。

こんな時にジェネリクス型を使うといい感じにまとめられたりします。

以下サンプルです。まずはコードを確認してみましょう。

```js
const addKeys = <T, U>(key1: T, key2: U): Array<T | U> => {
  return [key1, key2];
};

addKeys < string, string > ("a", "b");
addKeys < number, number > (1, 2);
addKeys < boolean, boolean > (true, false);
addKeys < string, number > ("a", 1);
```

関数側では `<T, U>(key1: T, key2: U): Array<T | U>`として呼び出し元で渡された型を変数 `T, U`に格納。
引数 `key1`と `key2`に渡された型変数 `T, U`を適応する。

関数の呼び出し側では `addKeys < type, type >(arg1, arg2)`として渡す引数に対しての型を `<>`に定義します。

このような流れです。一部だけに使うことなどももちろん OK です。

```js
const addKeys = <T>(key1: T, key2: string): Array<T | string> => {
  return [key1, key2];
};

addKeys < number > (1, "b");
```

このように共通のロジックやプロパティは使いまわしたいけど型だけはある程度動的に変更できるようにしたい。
そんな時にジェネリクス型が使えます。

# 列挙型定義

列挙型(enum)を用いると、複数の定数に値を持たせたリストを生成できます。
数値の列挙と文字列の列挙の 2 つがありますのでそれぞれみていきます。

## 数値列挙型定義

数値列挙は定数の変数名に対して連番の番号を値として保持する型定義の方法になります。
以下サンプルです。方角を数値列挙型として定義するとすると以下のような挙動になります。

```js
enum Directions {
  NORTH,
  SOUTH,
  EAST,
  WEST,
}

console.log("north", Directions.NORTH); // north 0
console.log("south", Directions.SOUTH); // south 1
console.log("east", Directions.EAST); // east 2
console.log("west", Directions.WEST); // west 3
```

初期値を設定するとそこからスタートする連番になります

```js
enum Directions {
  NORTH = 1,
  SOUTH,
  EAST,
  WEST,
}

console.log("north", Directions.NORTH); // north 1
console.log("south", Directions.SOUTH); // south 2
console.log("east", Directions.EAST); // east 3
console.log("west", Directions.WEST); // west 4
```

## 文字列挙型定義

先ほどは定数の名前、および初期値に数値を設定すると連番になる方法をみましたが、文字列でそれぞれに設定することも可能です。

以下サンプルです。先ほどの方角に対して、日本語での方角の値を定数の値として設定する方法です。

```js
enum Directions {
  NORTH = "北",
  SOUTH = "南",
  EAST = "東",
  WEST = "西",
}

console.log("north", Directions.NORTH); // north 北
console.log("south", Directions.SOUTH); // south 南
console.log("east", Directions.EAST); // east 東
console.log("west", Directions.WEST); // east 西
```

# class 型定義

ES6 から導入された class 構文ですが、こちらの class も型として指定できます。

まずは基本系です。`Person`のインスタンスは class 型で受け取ることができるのが確認できます。

```js
class Person {
  private name: string;
  private age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  public greeting() {
    return `Hello My name is ${this.name}. I'm ${this.age} years old.`;
  }
}

const person: Person = new Person("taro", 20);
console.log(person.greeting()); // // Hello My name is taro. I'm 20 years old.
```

次に別の class へのプロパティとして渡すパターンです。
いわゆる DI するときの `constructor`で class 型を受け取ることができます。

```js
class Person {
  private name: string;
  private age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  public greeting(): string {
    return `Hello My name is ${this.name}. I'm ${this.age} years old.`;
  }
}

class Profile {
  private person: Person;
  constructor(_person: Person) {
    this.person = _person;
  }

  public getMessage(): string {
    return `profile message. "${this.person.greeting()}"`;
  }
}

const person: Person = new Person("taro", 20);
const profile: Profile = new Profile(person);
console.log(profile.getMessage()); // profile message. "Hello My name is taro. I'm 20 years old."
```

継承してるパターンも確認してみます。
パラメータによってサブクラスが動的に変わるケースはよくあるかと思いますが、そういう時に継承されてるスーパークラスを戻り値の型として設定することができます。

以下サンプルです。

```js
abstract class Profile {
  protected name: string;
  protected age: number;

  constructor(name: string, age: number) {
    this.name = name;
    this.age = age;
  }

  public greeting(): string {
    return `Hello My name is ${this.name}. I'm ${this.age} years old.`;
  }
}

class PersonTaro extends Profile {
  constructor(name: string, age: number) {
    super(name, age);
  }
}

class PersonHanako extends Profile {
  constructor(name: string, age: number) {
    super(name, age);
  }
}

const handlePerson = (name: string): Profile | undefined => {
  if (name === "taro") {
    return new PersonTaro(name, 20);
  } else if (name === "hanako") {
    return new PersonHanako(name, 20);
  }
};

console.log(handlePerson("taro")?.greeting()); // Hello My name is taro. I'm 20 years old.
console.log(handlePerson("hanako")?.greeting()); // Hello My name is hanako. I'm 20 years old.
console.log(handlePerson("hoge")?.greeting()); // undefined
```

# 型エイリアス

ここまでの型定義ですが全部変数や関数の引数、戻り値のところに直接書いていましたが、エイリアスとして型定義だけを宣言したり別ファイルに切り分けたりすることができます。

特に Object なんかは実際のソースコードでは結構な量のプロパティを扱いますので変数のとこに直接定義したらコードの可読性が著しく落ちます。

そんな時はエイリアスとして別で定義してあげるとスッキリします。

## type の型エイリアス

```js
// ユニオン型を組み合わせた文字列型定義
type Gender = "male" | "female" | "other";
const gender: Gender = "male";

// Objectの型定義
type Person = { name: string, age: number };
const person: Person = { name: "taro", age: 20 };

// 配列の型定義
type Fruits = string[];
const fruits: Fruits = ["apple", "banana", "orange"];

// 関数の型定義
type Sum = (x: number, y: number) => number;
const sum: Sum = (x, y) => x + y;

// ジェネリクス型を組み合わせたObject型定義
type Hoge<T, U> = {
  hoge: T,
  foo: U,
};
const hoge: Hoge<string, number> = { hoge: "hoge", foo: 123 };
```

## interface の型定義

Object は`interface`で定義しても OK です。

```js
interface Person {
  name: string;
  age: number;
}
const person: Person = { name: "taro", age: 20 };

interface Hoge<T, U> {
  hoge: T;
  foo: U;
}
const hoge: Hoge<string, number> = { hoge: "hoge", foo: 123 };
```

## export してファイルを分割

`interface`は `export`して別ファイルで切り分けられます。
また `enum`も `export`して別ファイルで切り分けられます。

```js
export interface Person {
  name: string;
  age: number;
}
```

```js
export interface Hoge<T, U> {
  hoge: T;
  foo: U;
}
```

```js
export enum Directions {
  NORTH = "北",
  SOUTH = "南",
  EAST = "東",
  WEST = "西",
}
```

```js
import { Person } from "./@types/Person";
import { Hoge } from "./@types/Hoge";
import { Directions } from "./enums/Directions.enum";

const person: Person = { name: "taro", age: 20 };
const hoge: Hoge<string, number> = { hoge: "hoge", foo: 123 };
const directions: Directions[] = [
  Directions.NORTH,
  Directions.SOUTH,
  Directions.EAST,
  Directions.WEST,
];
```

この辺りが TypeScript を実務で使っていて使用頻度の高い型定義になるかと思います。
ここまでの内容を組み合わせればある程度は柔軟に型定義ができると思いますので色々試してみると良いと思います。

TypeScript の基礎的な知識はみれたと思いますので、次からは React 特有の型定義の方法を確認してみます。

# React コンポーネントの型定義

React のコンポーネントの型定義には以下で定義します。

```js
import React from "react";

const Hello: React.FC = () => {
  return <h1>Hello TypeSctipt × React</h1>;
};

export default Hello;
```

`React.FC`という React のモジュール内に定義されている型を使っています。
`import { FC } from "react";`として直接 import して`React.`を省略することも可能です。

::: message
React.FC は React18 系から導入されました。
18 以前は React.VFC としていましたが 18 系からは非推奨になってます。
ほぼ一緒ですが、Children が暗黙的に渡ってこなくなったという違いがあります。
:::

# props の型定義

props の受け取り方は以下です。こちらは `message`というプロパティが props として渡ってくる例です。

```js
import React from "react";

type PropsTypeTestProps = {
  message: string,
};

const PropsTypeTest: React.FC<PropsTypeTestProps> = (props) => {
  return (
    <>
      <p>propsから渡ってきた値は「{props.message}」です。</p>
    </>
  );
};

export default PropsTypeTest;
```

```js
import React from "react";
import Hello from "./components/Hello";

const App: React.FC = () => {
  return (
    <>
      <PropsTypeTest message="Appコンポーネントからのpropsメッセージです" />
    </>
  );
};

export default App;
```

`type`で渡ってくる props のプロパティの型を定義して `React.FC<Type>`のジェネリクス型で受け取っています。

props を分割代入で受け取っても OK です。

```js
const PropsTypeTest: React.FC<PropsTypeTestProps> = ({ message }) => {};
```

## props children の型定義

コンポーネントのタグに何かを入れ子にしたらそれは `children`というプロパティになります。
この `children`が渡ってきたときの型の宣言方法です。

まずは簡易な文字列を渡した場合のサンプルです。

```js
import React from "react";
type PropsChildren1Props = {
  children: React.ReactNode,
};

const PropsChildren1: React.FC<PropsChildren1Props> = (props) => {
  return <p>{props.children}</p>;
};

export default PropsChildren1;
```

```js
import React from "react";
import PropsChildren1 from "./components/PropsChildren1";

const App: React.FC = () => {
  return (
    <>
      <PropsChildren1>
        親コンポーネントからchildrenの値をpropsへ渡す
      </PropsChildren1>
    </>
  );
};

export default App;
```

`type`の中に `children`というプロパティを設定し、`React.ReactNode`の型を定義します。
React.v18 以降の `React.FC`の定義で明示的に `children`は定義するようになりました。(以前は暗黙的に渡ってきてた)

文字列だけでなくコンポーネントなどを渡すときも同じです。
以下ボーダーを入れ子にして子コンポーネントを展開しているサンプルです。

```js
import React from "react";
import styled from "styled-components";

type BorderProps = {
  color: string,
  children?: React.ReactNode,
};

const BorderStyle = styled.div`
  border: solid 4px ${({ color }) => color};
  border-radius: 8px;
  padding: 16px;
`;

const Border: React.FC<BorderProps> = (props) => {
  const { color, children } = props;
  return <BorderStyle color={color}>{children}</BorderStyle>;
};

export default Border;
```

```js
import React from "react";
type PropsChildren2Props = {
  children?: React.ReactNode,
};

const PropsChildren2: React.FC<PropsChildren2Props> = (props) => {
  return <p>{props.children}</p>;
};

export default PropsChildren2;
```

```js
import React from "react";
import PropsChildren2 from "./components/PropsChildren2";
import Border from "./components/Border";

const App: React.FC = () => {
  return (
    <>
      <Border color="green">
        <Border color="blue">
          <PropsChildren2>子コンポーネント</PropsChildren2>
        </Border>
      </Border>
    </>
  );
};

export default App;
```

## props function の型定義

props に function を渡した時の型定義の方法です。
以下サンプルです。ボタンを押したらアラートが出る関数を渡しています。

```js
import React from "react";

type PropsFncProps = {
  fnc: (text: string) => void,
};
const PropsFnc: React.FC<PropsFncProps> = (props) => {
  return <button onClick={() => props.fnc("hello")}>click</button>;
};

export default PropsFnc;
```

```js
import React from "react";
import PropsFnc from "./components/PropsFnc";

const App: React.FC = () => {
  return (
    <>
      <PropsFnc fnc={(text) => alert(text)} />
    </>
  );
};

export default App;
```

`type`に渡ってくる関数名 `fnc`を設定し引数の型を `(text: string)`とし、戻り値を定義します。今回はアラートするだけなので `void`です。
このように関数を渡す場合は `propsFunctionName: (arg: type) => returnType`の形式で定義してあげてください。

# event の型定義

`onClick`や `onChange`などのイベントハンドラを使用するとイベントオブジェクトを引数として受け取ります。
この時のイベントに対する型の定義の仕方です。

ある程度のイベントを一覧で載せますが、基本はイベント名をタイプして `()`で関数を定義するときにエディタの補完機能で適切な型を出してくれるので基本はそちらを参考にすれば良いです。

![](https://storage.googleapis.com/zenn-user-upload/1ef2a63eaac3-20221104.png)

以下、各種イベントの型定義のサンプルです。

```js
import React from "react";

const App: React.FC = () => {
  return (
    <>
      <h1>イベントの型定義</h1>

      <h2>クリック系</h2>
      <button
        onClick={(event: React.MouseEvent<HTMLButtonElement>) =>
          alert("button")
        }
      >
        button
      </button>
      <br />
      <button
        onDoubleClick={(event: React.MouseEvent<HTMLButtonElement>) =>
          alert("double click")
        }
      >
        double click
      </button>
      <br />
      <button
        onMouseDown={(event: React.MouseEvent<HTMLButtonElement>) =>
          alert("mouse click(down)")
        }
      >
        mouse click(down)
      </button>
      <br />
      <button
        onMouseUp={(event: React.MouseEvent<HTMLButtonElement>) =>
          alert("mouse click(up)")
        }
      >
        mouse click(up)
      </button>
      <br />

      <h2>フォーム操作系</h2>
      <form
        style={{ border: "solid 1px grey", padding: "8px" }}
        onSubmit={(event: React.FormEvent<HTMLFormElement>) => {
          event.preventDefault();
          alert("sumit");
        }}
      >
        <label htmlFor="inp">input(confim.console)</label>
        <br />
        <input
          id="inp"
          type="text"
          onChange={(event: React.ChangeEvent<HTMLInputElement>) =>
            console.log(event.target.value)
          }
        />
        <br />
        <label htmlFor="area">textarea(confim.console)</label>
        <br />
        <textarea
          id="area"
          onChange={(event: React.ChangeEvent<HTMLTextAreaElement>) =>
            console.log(event.target.value)
          }
        />
        <br />
        <label>radio(confim.console)</label>
        <br />
        <input
          type="radio"
          id="radio1"
          onChange={(event: React.ChangeEvent<HTMLInputElement>) =>
            console.log(event.target.value)
          }
        />
        <label htmlFor="radio1">radio1</label>
        <input
          type="radio"
          id="radio2"
          onChange={(event: React.ChangeEvent<HTMLInputElement>) =>
            console.log(event.target.value)
          }
        />
        <label htmlFor="radio2">radio2</label>
        <br />
        <label htmlFor="check">checkBox(confim.console)</label>
        <br />
        <input
          type="checkbox"
          id="check"
          onChange={(event: React.ChangeEvent<HTMLInputElement>) =>
            console.log(event.target.value)
          }
        />
        <br />
        <label>selectBox(confim.console)</label>
        <br />
        <select
          onChange={(event: React.ChangeEvent<HTMLSelectElement>) =>
            console.log(event.target.value)
          }
        >
          <option value="a">a</option>
          <option value="b">b</option>
          <option value="c">c</option>
        </select>
        <br />
        <label>focus in(confim.console)</label>
        <br />
        <input
          type="text"
          onFocus={(event: React.ChangeEvent<HTMLInputElement>) =>
            console.log("focus in")
          }
        />
        <br />
        <label>focus out(confim.console)</label>
        <br />
        <input
          type="text"
          onBlur={(event: React.ChangeEvent<HTMLInputElement>) =>
            console.log("focus out")
          }
        />
        <br />
        <button type="submit">submit</button>
      </form>
      <h2>マウス操作系</h2>
      <label>confim.console</label>
      <br />
      <p
        style={{
          border: "solid 1px grey",
          width: "120px",
          height: "60px",
        }}
        onMouseOver={(event: React.MouseEvent<HTMLParagraphElement>) =>
          console.log("mouse on")
        }
        onMouseOut={(event: React.MouseEvent<HTMLParagraphElement>) =>
          console.log("mouse out")
        }
        onMouseMove={(event: React.MouseEvent<HTMLParagraphElement>) =>
          console.log("mouse move")
        }
      >
        マウス操作範囲
      </p>
      <h2>キー操作系</h2>
      <label>confim.console</label>
      <br />
      <input
        style={{
          border: "solid 1px grey",
          width: "120px",
          height: "60px",
        }}
        onKeyDown={(event: React.KeyboardEvent<HTMLParagraphElement>) =>
          console.log("key down")
        }
        onKeyUp={(event: React.KeyboardEvent<HTMLParagraphElement>) =>
          console.log("key up")
        }
        onKeyPress={(event: React.KeyboardEvent<HTMLParagraphElement>) =>
          console.log("key press")
        }
      />
    </>
  );
};

export default App;
```

# useState の型定義

useState での型定義は `useState<type>(initValue)`のジェネリクス型で定義してあげます。
何パターンかサンプルで確認します。

まず、useState のジェネリクス型に文字列配列を定義するケースです。

```js
import React, { useState } from "react";

type Person = {
  name: string;
  age: number;
};

const App: React.FC = () => {
  const [fruits] = useState<string[]>(["apple", "banana", "orange"]);
  return (
    <>
      <p>フルーツ一覧：　{fruits.join(",")}</p>
    </>
  );
};

export default App;
```

`useState<string[]>`とジェネリクス型で定義して初期値は型に合わせて `["apple", "banana", "orange"]`の配列であることが確認できます。

次に useState のジェネリクス型にオブジェクトリテラルを定義するケースです。

```js
import React, { useState } from "react";

const App: React.FC = () => {
  const [fruitPrice] = useState<{ name: string; price: number }>({
    name: "banana",
    price: 100
  });

  return (
    <>
      <p>
        {fruitPrice.name} ${fruitPrice.price}
      </p>
    </>
  );
};

export default App;
```

こちらも useState の型定義に合わせて初期値を設定してます。

最後に useState のジェネリクス型にエイリアスを定義するケースです。

```js
import React, { useState } from "react";

type Person = {
  name: string,
  age: number,
};

const App: React.FC = () => {
  const [name, setName] = useState < string > "taro";
  const [age, setAge] = useState < number > 20;
  const [person, setPerson] = useState < Person > { name, age };

  const changePerson = (): void => {
    setPerson({ name, age });
  };

  return (
    <>
      <label htmlFor="name">名前を入力</label>
      <br />
      <input
        id="name"
        type="text"
        value={name}
        onChange={(event: React.ChangeEvent<HTMLInputElement>) =>
          setName(event.target.value)
        }
      />
      <br />
      <label htmlFor="age">年齢を入力</label>
      <br />
      <input
        id="age"
        type="number"
        value={age}
        onChange={(event: React.ChangeEvent<HTMLInputElement>) =>
          setAge(Number(event.target.value))
        }
      />
      <br />
      <button onClick={changePerson}>変更</button>
      <p>
        {person.name}'s age: {person.age}
      </p>
    </>
  );
};

export default App;
```

せっかくなのでステートの更新も確認するためにフォーム風にしました。
`name`と `age`のプロパティの型定義をしたエイリアス `Person`を定義します。

それを `useState < Person >`とジェネリクス型の引数に渡して定義してます。
プロパティのそれぞれの値を input で取得するため、別々のステートに格納し初期値は `{name, age}`とします。

input の内容を変更して `変更`ボタンを押すとステートが更新され画面の値も更新されることが確認できます。

# サンプル

ここまでの React のサンプルのコードを codesandbox に作成しましたのでよろしければご参考にしてください。

@[codesandbox](https://codesandbox.io/embed/react-typescript-type-6trmiq?fontsize=14&hidenavigation=1&theme=dark)

# any 型

ここまでいろんなパターンで型定義の方法を確認しましたが、全てを any 型に変更することができます。
any は一言で言えばどんな型定義でも許容します。という意味合いになります。

なんでもいいと言うことなので TypeScript のトランスパイル時に型の違いによるエラーがなくなりますし、型を定義していることで自動補完で型のプロパティが候補に出てこなくなったりしてしまいます。

プログラムとしてもバグを生みやすい原因にもなりますし、自動補完されないのは長い目で見たら作業効率が悪いので出来ることならあまり多様したくない型定義の方法です。

ですが any が完全に悪かというとそうとも言い切れないかと思います。

まず、型定義は慣れないと結構難しいです。
React は関数型プログラミングの手法に完全に舵を切ったと思うのでそれほどかもしれませんが、TypeScript のみのコードでオブジェクトが細かく分割され抽象化されまくっていたりしていると戻り値の型を判定するのが難しいなどは結構あると思います。

これまでの JavaScript のコードを TypeScript に移行するなどを検討したときにまずは素早くリリースしたいのにこの型の定義がうまいこと進まなくて時間的コストが高くなる可能性があります。

これは本末転倒です。

静的型付けは新しい便利な機能を提供したり処理速度を向上したりするものではなく、あくまでプログラムの安全性を向上するためのものです。

ですのでプロジェクトにおいてまずは何を優先すべきかをしっかりと決めた上で、まずはリリースという結論が出たのであれば最初は any 型でエラーは回避して後から少しずつ型を正確に定義していくという流れも良いと思います。

プロジェクトでは臨機応変に、学習の時は any 禁止令を課してコードを書くなどしたりするのが個人的には良いアプローチだと思います。

# さいごに

さいごまで読んでいただきありがとうございます。
頻出の部分に絞ろうとは思ってたのですが、書いてくうちに結構長くなってしまい読みづらかったりしたらすみません。

間違いの指摘やリクエストなどありましたら加筆していきたので是非、ご意見をいただけたらと思います。

それではまた次の記事でお会いしましょう。

PS. Zenn の処女作です。
