---
title: '値オブジェクト'
---

# 値オブジェクト(Value Object)とは

値オブジェクトは、エンティティと並びドメインモデル(ドメインオブジェクト)の中心的な要素で、ドメイン内の様々な**値**の概念をモデル化するのに用いられます。

# 値とは

ここで少し値とは？を考えてみましょう。これは 「xxxxxxxx」 という値で BookId という変数に代入されています。このコードは正しいように見えます。当然エラーも発生しません。では、変数 BookId の値として「xxxxxxxx」は正しいでしょうか？

```js
const BookId: string = 'xxxxxxxx';
```

ドメインモデリングで作成した BookId.pu を振り返ってみましょう。確認したところ「BookId」として正しい値は **ISBN コード**であることが読み取れます。ISBN コードはいくつものルールの組み合わせによって構成されています。

```plantuml:StockManagement/Domain/models/Book/BookId/BookId.pu
@startuml BookId

class 'BookId' as BookId {
    + value: string
}

note bottom of BookId
    ISBNコードを適用する。
    ISBNコードは、ISBNのあとに数字で「978」、
    さらにグループ（国・地域）番号（日本は4）、出版社番号、書名番号、の合計12桁の数字を並べ、
    最後にこの12桁の数字を特定の計算式で演算して得た1桁のチェック用の数を付け加えたコード。
end note

@enduml
```

BookId.pu の内容に従い値を ISBN コードに修正しました。これで変数 BookId は正しい値になりました...。

```js
const BookId: string = '9774167158057';
```

と思いましたが、ISBN コードは「978」から始まらなければいけないところを間違えて「977」から始めてしまいました。正しくは`'9784167158057'`こちらです。さて、このミスに気づけた方はいるでしょうか？当然エラーも発生しません。また、この数字の羅列を見て ISBN コードであると理解できる方がどれほどいるでしょうか？つまり「BookId」は不正な状態で存在することが可能であり、正しい値が何かわからないという状況です。これではバグの温床になってしまいます。

この問題を値オブジェクトは解決します。

## 値の特徴

値には主に 3 つの特徴があります。そしてこれらは値オブジェクトに対しても当てはまります。

- 不変に保つことができる
- 値同士が等しいか比較できる
- 副作用がない

ではそれぞれ確認していきましょう。

### 不変に保つことができる

値が不変とはどういうことでしょうか？私たちが頻繁に使う値にはプリミティブ型（例えば、文字列、数値、ブーリアン）があります。プリミティブ型は値そのものが直接変数に格納されます。そして、プリミティブ型の値は不変です。これは、一度作成されると、その値自体を変更することができないという意味です。

```js
let value = 'value';
// 'value' という文字列自体を変更することはできません。
// 以下の操作は新しい文字列を作成し、それを value に割り当てるだけです。
value = value + ' changed'; // 新しい文字列 'value changed' を作成
```

この例では、value 変数は最初に 'value' という値を参照しています。その後、value + ' changed' によって新しい値 'value changed' が作成され、value 変数はこの新しい値を参照するようになります。重要な点は、元の値 'value' そのものが変更されたわけではなく、代わりに新しい値が'value changed'が作成されたということです。これは値が不変であることを意味します。

### 値同士が等しいか比較できる

「値同士が等しいか比較できる」という特徴は直感的です。プリミティブな値は、その内容によって等価性が判断されます。

```js
let value1 = 'Hello';
let value2 = 'Hello';
let value3 = 'Goodbye';

console.log(value1 === value2); // true
console.log(value1 === value3); // false
```

この例では、value1 と value2 は同じ文字列「Hello」を保持しているため、等価とみなされ、比較結果は true になります。一方、value1 と value3 は異なる内容を持っているため、等価ではなく、結果は false になります。

### 副作用がない

「副作用がない」とはどういうことでしょうか？プログラミングにおいて、「副作用がない」とは、ある操作が他の状態に予期せぬ影響を与えないことを意味します。そして値の操作には副作用がありません。例えばプリミティブな値である文字列は、`String.prototype`から `toUpperCase()`、`toLowerCase()`などのメソッドを継承していますが、これらのメソッドには副作用がありません。

```js
let originalValue = 'Hello';

// 新しい値を生成するが、originalValueは変更されない
let newValue = originalValue.toUpperCase();

console.log(originalValue); // 出力: "Hello"
console.log(newValue); // 出力: "HELLO"
```

この例では、originalValue 変数に格納されている文字列 "Hello" に対して toUpperCase() メソッドを適用しています。このメソッドは新しい文字列 "HELLO" を生成しますが、元の originalValue は変更されません。これは値の操作には副作用がないことを意味します。

# 値オブジェクトの実装

値の特徴が確認できたので、値オブジェクトを実装してみましょう。

まずは、 lodash を利用するためインストールしましょう。

```bash:StockManagement/
$ npm i lodash
$ npm i -D @types/lodash
```

「BookId.ts」を作成し書いていきます。こちらが値オブジェクトのベースになります。

```js:StockManagement/src/Domain/models/Book/BookId/BookId.ts
import isEqual from 'lodash/isEqual';

export class BookId {
  private readonly _value: string;

  constructor(value: string) {
    this._value = value;
  }

  equals(other: BookId): boolean {
   return isEqual(this._value, other._value);
  }

  get value(): string {
    return this._value;
  }
}
```

:::message
値オブジェクトは、プリミティブな値だけでなく、複雑なデータ構造を持つ場合があります。lodash の isEqual は、オブジェクトや配列などの複雑なデータ構造を持つ値に対しても、深い比較（Deep Comparison）を実行できます。しかし大量のデータを扱うシステムでは、パフォーマンスに影響する可能性があるため、要件によって取捨選択してください。
:::

BookId クラスを用いて、値オブジェクトの特徴がどのように実装されているかを確認しましょう。

### 不変に保つことができる

BookId クラスの `value` プロパティは `private readonly` で定義されており、一度設定された後は変更できません。これにより、BookId インスタンスは生成時に受け取った 値を変更できない、不変のオブジェクトとなります。

```js
const bookId = new BookId('9784167158057');
console.log(bookId.value); // "9784167158057"

bookId.value = '新しいISBN'; // エラー: _value は readonly なので変更できません。
```

### 値同士が等しいか比較できる

`equals` メソッドを用いて、他の BookId インスタンスとの等価性を比較できます。

```js
const bookId1 = new BookId('9784167158057');
const bookId2 = new BookId('9784167158057');
const bookId3 = new BookId('9780306406157');

console.log(bookId1.equals(bookId2)); // true
console.log(bookId1.equals(bookId3)); // false
```

:::message
値オブジェクトの比較では必ず equals メソッドを利用しましょう。
**値オブジェクトは値です**。以下のコードは`'9784167158057'.value === '9784167158057'.value`と同じで値の値(value)を取り出して比較を行なっており不自然です。

```js
const bookId1 = new BookId('9784167158057');
const bookId2 = new BookId('9784167158057');

console.log(bookId1.value === bookId2.value); // NG
```

:::

### 副作用がない

BookId クラス内で行われる操作は、他のオブジェクトや外部状態に影響を与えるような副作用を持ちません。例えば、`equals` メソッドは比較のために内部状態を変更することはなく、単純な値の比較のみを行います。

```js
// 使用例
const bookId1 = new BookId('9784167158057');
const bookId2 = new BookId('9784167158057');

bookId1.equals(bookId2);
console.log(bookId1.value); // "9784167158057"
```

## ビジネスルールの適用

実装した値オブジェクトが値の特徴を満たしていることが確認できました。ですが、まだ未完成です。値オブジェクトの真髄は値にビジネスルールを値に適用できる点にあります。「BookId」に対してジネスルールを適用していきましょう。(バリデーション、変換のロジックの説明はここでは省略します。ロジックを読み解く必要はありません。)

```js:StockManagement/src/Domain/models/Book/BookId/BookId.ts
import { isEqual } from 'lodash';

export class BookId {
  private readonly _value: string;

  constructor(value: string) {
    this._value = this.validateAndConvert(value); // バリデーションのチェックと値の変換を行う
  }

  private MAX_LENGTH = 100;
  private MIN_LENGTH = 10;

  private validateAndConvert(isbn: string): string {
    if (isbn.length < this.MIN_LENGTH || isbn.length > this.MAX_LENGTH) {
      throw new Error('ISBNの文字数が不正です');
    }

    // ISBNプレフィックス（もしあれば）とハイフンを除去
    const cleanedIsbn = isbn.replace(/^ISBN-?|-/g, '');

    if (cleanedIsbn.length === 10) {
      // ISBN-10 を ISBN-13 に変換
      return this.convertIsbn10ToIsbn13(cleanedIsbn);
    } else if (cleanedIsbn.length === 13) {
      // ISBN-13 のバリデーション
      if (this.isValidIsbn13(cleanedIsbn)) {
        return cleanedIsbn;
      }
    }

    throw new Error('不正なISBNの形式です');
  }

  private convertIsbn10ToIsbn13(isbn10: string): string {
    const prefix = '978';
    const core = isbn10.substring(0, 9); // 最初の9文字を取得
    const checksum = this.calculateIsbn13Checksum(prefix + core);
    return prefix + core + checksum;
  }

  private isValidIsbn13(isbn13: string): boolean {
    return isbn13.startsWith('978') || isbn13.startsWith('979');
  }

  private calculateIsbn13Checksum(isbn13: string): string {
    let sum = 0;
    for (let i = 0; i < 12; i++) {
      const digit = parseInt(isbn13.charAt(i), 10);
      sum += i % 2 === 0 ? digit : digit * 3;
    }
    const checksum = 10 - (sum % 10);
    return checksum === 10 ? '0' : checksum.toString();
  }

  equals(other: BookId): boolean {
    return isEqual(this._value, other._value);
  }

  get value(): string {
    return this._value;
  }

  toISBN(): string {
    // この例では 'ISBN978-4-16-715805-7' のような特定のフォーマットに限定
    const isbnPrefix = this._value.substring(0, 3); // 最初の3桁 (978 または 979)
    const groupIdentifier = this._value.substring(3, 4); // 国コードなど（1桁）
    const publisherCode = this._value.substring(4, 6); // 出版者コード（2桁）
    const bookCode = this._value.substring(6, 12); // 書籍コード（6桁）
    const checksum = this._value.substring(12); // チェックディジット（1桁）

    return `ISBN${isbnPrefix}-${groupIdentifier}-${publisherCode}-${bookCode}-${checksum}`;
  }
}

```

まずコンストラクタでは `validateAndConvert` メソッドを使って入力された ISBN をバリデーションし、必要に応じて変換します。これにより **BookId は無効な ISBN を受け付けず、適切なフォーマットの ISBN だけ**が BookId オブジェクトとして作成されます。これにより、不正なデータがシステムに流入するリスクが軽減されます。

```js
  constructor(value: string) {
    this._value = this.validateAndConvert(value);
  }
```

:::message
バリデーションを行う場合構文チェック等の前に文字数のチェックを先に行うことを推奨します。仮に文字列が 10 億桁だった場合、正規表現エンジンが読み込み負荷の重い処理を無駄に実行してしまい、パフォーマンスに影響が出る可能性があります。また、文字数のチェックを先に行うことで、正規表現のパターンを簡単にすることができます。

```js
  private validateAndConvert(isbn: string): string {
    if (isbn.length < this.MIN_LENGTH || isbn.length > this.MAX_LENGTH) {
      throw new Error('ISBNの文字数が不正です');
    }
    ...
```

:::

BookId の値から ISBN コードへの変換を行う振る舞いを追加しています。これにより、**ISBN フォーマットへの変換ロジックを 値自身で管理(カプセル化)**することができ、保守性が向上します。

```js
 toISBN(): string {
    // この例では 'ISBN978-4-16-715805-7' のような特定のフォーマットに限定
    const isbnPrefix = this._value.substring(0, 3); // 最初の3桁 (978 または 979)
    const groupIdentifier = this._value.substring(3, 4); // 国コードなど（1桁）
    const publisherCode = this._value.substring(4, 6); // 出版者コード（2桁）
    const bookCode = this._value.substring(6, 12); // 書籍コード（6桁）
    const checksum = this._value.substring(12); // チェックディジット（1桁）

    return `ISBN${isbnPrefix}-${groupIdentifier}-${publisherCode}-${bookCode}-${checksum}`;
  }
```

これらの実装により、値にビジネスルールを適用することができました。

# 値オブジェクトのテスト

値オブジェクトのビジネスルールが正しく実装されているかを保証するためにはテストは必須です。DDD においてビジネス環境や市場の変化、組織の戦略の変更などにより、ビジネスルールも変わる可能性があります。テストを書いておくことで、値オブジェクトのビジネスルールを正しく理解し、その振る舞いが変更された場合にもすぐに気づくことができます。

それではテストを書いていきましょう。テストは「BookId.test.ts」に書いていきます。

```js:StockManagement/src/Domain/models/Book/BookId/BookId.test.ts

import { BookId } from './BookId';

describe('BookId', () => {
  // 正常系
  test('有効なフォーマットの場合正しい変換結果を期待', () => {
    expect(new BookId('ISBN978-4-16-715805-7').value).toBe('9784167158057');
    expect(new BookId('978-4-16-715805-7').value).toBe('9784167158057');
    expect(new BookId('9784167158057').value).toBe('9784167158057');
  });

  test('旧ISBN(10桁)で入力された場合978を頭に追加した新ISBNに変換される', () => {
    const isbn10 = '4167158051';
    const bookId = new BookId(isbn10);
    expect(bookId.value).toBe('9784167158057'); // 正しい変換結果を期待
  });

  test('equals', () => {
    const bookId1 = new BookId('9784167158057');
    const bookId2 = new BookId('9784167158057');
    const bookId3 = new BookId('9781234567890');
    expect(bookId1.equals(bookId2)).toBeTruthy();
    expect(bookId1.equals(bookId3)).toBeFalsy();
  });

  test('toISBN', () => {
    const bookId = new BookId('9784167158057');
    expect(bookId.toISBN()).toBe('ISBN978-4-16-715805-7');
  });

  // 異常系
  test('不正な文字数の場合にエラーを投げる', () => {
    // 境界値のテスト
    expect(() => new BookId('1'.repeat(101))).toThrow('ISBNの文字数が不正です');
    expect(() => new BookId('1'.repeat(9))).toThrow('ISBNの文字数が不正です');
  });

  test('不正なフォーマットの場合にエラーを投げる', () => {
    expect(() => new BookId('187437538537852742')).toThrow(
      '不正なISBNの形式です'
    );
  });
});

```

ビジネスロジック、振る舞い(メソッド)、例外処理などを網羅するようにテストを書きます。
なるべくテストのカバレッジが 100%に近づけるようにしましょう。

jest コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest

 PASS  src/Domain/models/Book/BookId/BookId.test.ts
  BookId
    ✓ 有効なフォーマットの場合正しい変換結果を期待 (6 ms)
    ✓ 旧ISBN(10桁)で入力された場合978を頭に追加した新ISBNに変換される (1 ms)
    ✓ equals (1 ms)
    ✓ toISBN
    ✓ 不正な文字数の場合にエラーを投げる (17 ms)
    ✓ 不正なフォーマットの場合にエラーを投げる (6 ms)
```

以上で「BookId」値オブジェクトの実装は完了です。ビジネスロジックを値にカプセル化し、テストによって品質を担保することができました。そして最初に値が抱えていた「**不正な状態で存在することが可能であり、正しい値が何かわからない**」という問題が解決されます。

# その他の値オブジェクトの実装

- 値オブジェクトにする基準
- 値オブジェクトの実装に型によるメリットの説明入れる

# まとめ

```

```
