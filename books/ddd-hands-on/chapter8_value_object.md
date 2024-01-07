---
title: '値オブジェクト (Value Object) '
---

# 値オブジェクトとは

値オブジェクト (Value Object) とは、エンティティと並びドメインモデル (ドメインオブジェクト) の中心的な要素で、ドメイン内のさまざまな**値**の概念をモデル化するのに用いられます。値には、名前、年齢、色などがあります。値オブジェクトは、これらの値を表現するために使用されます。

## 値とは

ここで少し値とは何かを考えてみましょう。これは 「xxxxxxxx」 という値で `BookId`という変数に代入されています。このコードは正しいように見えます。当然`TypeScript`では正しい構文ですし、エラーも発生しません。では、変数`BookId`の値として「xxxxxxxx」は正しいでしょうか？

```js
const BookId: string = 'xxxxxxxx';
```

ドメインモデリングで作成した `BookId.pu` を振り返ってみましょう。確認したところ`BookId`として正しい値は **ISBN コード**であることが読み取れます。ISBN コードはいくつものルールの組み合わせによって構成されています。

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

`BookId.pu`の内容に従い値を ISBN コードに修正しました。これで変数 `BookId` を正しい値にすることができました。

```js
const BookId: string = '9774167158057';
```

と思いましたが、ISBN コードは「978」から始まらなければいけないところを間違えて「977」から始めてしまいました。正しくは`'9784167158057'`です。さて、このミスに気づけた方はいるでしょうか？

また、この数字の羅列を見て ISBN コードであると理解できる方がどれほどいるでしょうか？つまり`BookId`は不正な状態で存在することが可能であり、正しい値が何かわからないという状況です。これではバグの温床になってしまいます。

このような問題を値オブジェクトは解決します。

## 値の特徴

値には主に 3 つの特徴があります。そしてこれらは値オブジェクトに対しても同様に当てめる必要があります。

- 不変に保つことができる
- 値同士が等しいか比較できる
- 副作用がない

ではそれぞれ確認していきましょう。

### 不変に保つことができる

値が不変とはどういうことでしょうか？私たちが頻繁に使う値にはプリミティブ型（たとえば、文字列、数値、ブーリアンなど）があります。プリミティブ型は値そのものが直接変数に格納されます。そして、プリミティブ型の値は不変です。不変とは、一度作成されるとその値自体を変更することができないという意味です。

```js
let value = 'value';
// 'value' という文字列自体を変更することはできません。
// 以下の操作は新しい文字列を作成し、それを value に割り当てるだけです。
value = value + ' changed'; // 新しい文字列 'value changed' を作成
```

この例では、`value` 変数は最初に `'value'`という値を参照しています。その後、`value + ' changed'` によって新しい値 `'value changed'` が作成され、`value` 変数はこの新しい値を参照するようになります。重要な点は、元の値 `'value'` そのものが変更されたわけではなく、代わりに新しい値`'value changed'`が作成されたということです。これは値が不変であることを意味します。

### 値同士が等しいか比較できる

「値同士が等しいか比較できる」という特徴は直感的です。プリミティブな値は、その内容によって等価性が判断されます。

```js
let value1 = 'Hello';
let value2 = 'Hello';
let value3 = 'Goodbye';

console.log(value1 === value2); // true
console.log(value1 === value3); // false
```

この例では、`value1 `と `value2` は同じ文字列`Hello`を保持しているため、等価とみなされ、比較結果は `true` になります。一方、`value1` と `value3` は異なる内容を持っているため、等価ではなく、結果は `false` になります。

### 副作用がない

「副作用がない」とはどういうことでしょうか？プログラミングにおいて、「副作用がない」とは、ある操作が他の状態に予期せぬ影響を与えないことを意味します。そして値の操作には副作用がありません。例えばプリミティブな値である文字列は、`String.prototype`から `toUpperCase()`、`toLowerCase()`などのメソッドを継承していますが、これらのメソッドには副作用がありません。

```js
let originalValue = 'Hello';

// 新しい値を生成するが、originalValueは変更されない
let newValue = originalValue.toUpperCase();

console.log(originalValue); // 出力: "Hello"
console.log(newValue); // 出力: "HELLO"
```

この例では、`originalValue` 変数に格納されている文字列 `Hello` に対して `toUpperCase()` メソッドを適用しています。このメソッドは新しい文字列 `HELLO` を生成しますが、元の `originalValue` は変更されません。これは値の操作には副作用がないことを意味します。

# 値オブジェクトの実装

値の特徴が確認できたので、実際に値オブジェクトを実装してみましょう。

## 実装

まずは、 必要なパッケージをインストールしましょう。

```bash:StockManagement/
$ npm i lodash nanoid@3 #バージョンはは3系を指定してください
$ npm i -D @types/lodash
```

`src/Domain/models/Book/BookId/`配下に`BookId.ts`を作成し以下のように実装します。これが基本的な値オブジェクトのベースになります。

```js:src/Domain/models/Book/BookId/BookId.ts
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
値オブジェクトは、プリミティブな値だけでなく、複雑なデータ構造を持つ場合があります。`lodash` の `isEqual` は、オブジェクトや配列などの複雑なデータ構造を持つ値に対しても、深い比較 (Deep Comparison) を実行できます。しかし大量のデータを扱うシステムでは、パフォーマンスに影響する可能性があるため、要件によって取捨選択してください。
:::

それでは`BookId`クラスを用いて、値オブジェクトの特徴がどのように実装されているかを確認してみましょう。

### 不変に保つことができる

BookId クラスの `value` プロパティは `private readonly` で定義されており、一度設定された後は変更できません。これにより、`BookId` インスタンスは生成時に受け取った 値を変更できない、不変のオブジェクトとなります。

```js
const bookId = new BookId('9784167158057');
console.log(bookId.value); // "9784167158057"

bookId.value = '新しいISBN'; // エラー: _value は readonly なので変更できません。
```

### 値同士が等しいか比較できる

`equals` メソッドを用いて、他の `BookId` インスタンスとの等価性を比較できます。

```js
const bookId1 = new BookId('9784167158057');
const bookId2 = new BookId('9784167158057');
const bookId3 = new BookId('9780306406157');

console.log(bookId1.equals(bookId2)); // true
console.log(bookId1.equals(bookId3)); // false
```

:::message
値オブジェクト同士の比較には必ず equals メソッドを利用しましょう。
**値オブジェクトは値です**。以下のコードは`'9784167158057'.value === '9784167158057'.value`と同じで値の値 (value) を取り出して比較を行なっており不自然な実装です。

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

実装した値オブジェクトが値の特徴を満たしていることが確認できました。ですが、まだ未完成です。今のままでは変わらず不正な ISBN で `BookId` を生成することが可能です。値オブジェクトの真髄は値にビジネスルールを適用できる点にあります。それでは、ビジネスルールを適用していきましょう。(バリデーション、変換のロジックの実装は省略します)

```js:src/Domain/models/Book/BookId/BookId.ts
import { isEqual } from 'lodash';

export class BookId {
  private readonly _value: string;

  static MAX_LENGTH = 13;
  static MIN_LENGTH = 10;

  constructor(value: string) {
    this.validate(value);
    this._value = value;
  }

  private validate(isbn: string): void {
    if (isbn.length < BookId.MIN_LENGTH || isbn.length > BookId.MAX_LENGTH) {
      throw new Error('ISBNの文字数が不正です');
    }

    if (!this.isValidIsbn10(isbn) && !this.isValidIsbn13(isbn)) {
      throw new Error('不正なISBNの形式です');
    }
  }

  private isValidIsbn10(isbn10: string): boolean {
    // ISBN-10 のバリデーションロジックを実装
    // 実際の実装ではここにチェックディジットを計算するロジックが必要です。
    return isbn10.length === 10; // ここを実際のチェックディジット計算に置き換える
  }

  private isValidIsbn13(isbn13: string): boolean {
    // ISBN-13 のバリデーションロジックを実装
    // ここでは簡単な例を示しますが、実際にはより複雑なチェックが必要です
    return isbn13.startsWith('978') && isbn13.length === 13;
  }

  equals(other: BookId): boolean {
    return isEqual(this._value, other._value);
  }

  get value(): string {
    return this._value;
  }

  toISBN(): string {
    if (this._value.length === 10) {
      // ISBNが10桁の場合の、'ISBN' フォーマットに変換します。
      const groupIdentifier = this._value.substring(0, 1); // 国コードなど（1桁）
      const publisherCode = this._value.substring(1, 3); // 出版者コード（2桁）
      const bookCode = this._value.substring(3, 9); // 書籍コード（6桁）
      const checksum = this._value.substring(9); // チェックディジット（1桁）

      return `ISBN${groupIdentifier}-${publisherCode}-${bookCode}-${checksum}`;
    } else {
      // ISBNが13桁の場合の、'ISBN' フォーマットに変換します。
      const isbnPrefix = this._value.substring(0, 3); // 最初の3桁 (978 または 979)
      const groupIdentifier = this._value.substring(3, 4); // 国コードなど（1桁）
      const publisherCode = this._value.substring(4, 6); // 出版者コード（2桁）
      const bookCode = this._value.substring(6, 12); // 書籍コード（6桁）
      const checksum = this._value.substring(12); // チェックディジット（1桁）

      return `ISBN${isbnPrefix}-${groupIdentifier}-${publisherCode}-${bookCode}-${checksum}`;
    }
  }
}


```

まずコンストラクタでは `validate` メソッドを使って入力された ISBN をバリデーションします。これにより **BookId は無効な形式の ISBN を受け付けず、適切なフォーマットの ISBN だけ**が `BookId` 値オブジェクトとして作成されます。これにより、不正なデータがシステムに流入するリスクが軽減されます。

```js
  constructor(value: string) {
    this.validate(value);
    this._value = value;
  }
```

:::message
バリデーションでは、構文チェック等の前に文字数のチェックを行うことを推奨します。なぜなら仮に文字列が 10 億桁だった場合、正規表現エンジンが読み込み負荷の重い処理を無駄に実行してしまい、パフォーマンスに影響が出る可能性があります。

また、文字数のチェックを先に行うことで、正規表現のパターンを簡単にすることができます。

```js
  private validate(isbn: string): void {
    if (isbn.length < BookId.MIN_LENGTH || isbn.length > BookId.MAX_LENGTH) {
      throw new Error('ISBNの文字数が不正です');
    }
    (省略)
  }
```

:::

`toISBN`メソッドでは`BookId` の値から ISBN コードの表示用フォーマットへの変換を行なっています。これにより、ISBN フォーマットへの変換ロジックを 値自身で管理 (カプセル化) することができ、保守性が向上します。

```js
  toISBN(): string {
    if (this._value.length === 10) {
      // ISBNが10桁の場合の、'ISBN' フォーマットに変換します。
      const groupIdentifier = this._value.substring(0, 1); // 国コードなど（1桁）
      const publisherCode = this._value.substring(1, 3); // 出版者コード（2桁）
      const bookCode = this._value.substring(3, 9); // 書籍コード（6桁）
      const checksum = this._value.substring(9); // チェックディジット（1桁）

      return `ISBN${groupIdentifier}-${publisherCode}-${bookCode}-${checksum}`;
    } else {
      // ISBNが13桁の場合の、'ISBN' フォーマットに変換します。
      const isbnPrefix = this._value.substring(0, 3); // 最初の3桁 (978 または 979)
      const groupIdentifier = this._value.substring(3, 4); // 国コードなど（1桁）
      const publisherCode = this._value.substring(4, 6); // 出版者コード（2桁）
      const bookCode = this._value.substring(6, 12); // 書籍コード（6桁）
      const checksum = this._value.substring(12); // チェックディジット（1桁）

      return `ISBN${isbnPrefix}-${groupIdentifier}-${publisherCode}-${bookCode}-${checksum}`;
    }
  }
```

これらの実装により、値にビジネスルールを適用することができました。

# 値オブジェクトのテスト

値オブジェクトのビジネスルールが正しく実装されているかを保証するためにはテストは必須です。DDD においてビジネス環境や市場の変化、組織の戦略の変更などにより、ビジネスルールも変わる可能性は十分にあります。テストを書いておくことで、値オブジェクトのビジネスルールを正しく理解し、その振る舞いが変更された場合にもすぐに気づくことができます。

それではテストを書いていきましょう。`BookId.ts`と同じディレクトリに`BookId.test.ts`を作成し、以下のように実装します。

```js:src/Domain/models/Book/BookId/BookId.test.ts
import { BookId } from './BookId';

describe('BookId', () => {
  // 正常系
  test('有効なフォーマットの場合正しい変換結果を期待', () => {
    expect(new BookId('9784167158057').value).toBe('9784167158057');
    expect(new BookId('4167158051').value).toBe('4167158051');
  });

  test('equals', () => {
    const bookId1 = new BookId('9784167158057');
    const bookId2 = new BookId('9784167158057');
    const bookId3 = new BookId('9781234567890');
    expect(bookId1.equals(bookId2)).toBeTruthy();
    expect(bookId1.equals(bookId3)).toBeFalsy();
  });

  test('toISBN() 13桁', () => {
    const bookId = new BookId('9784167158057');
    expect(bookId.toISBN()).toBe('ISBN978-4-16-715805-7');
  });
  test('toISBN() 10桁', () => {
    const bookId = new BookId('4167158051');
    expect(bookId.toISBN()).toBe('ISBN4-16-715805-1');
  });

  // 異常系
  test('不正な文字数の場合にエラーを投げる', () => {
    // 境界値のテスト
    expect(() => new BookId('1'.repeat(101))).toThrow('ISBNの文字数が不正です');
    expect(() => new BookId('1'.repeat(9))).toThrow('ISBNの文字数が不正です');
  });

  test('不正なフォーマットの場合にエラーを投げる', () => {
    expect(() => new BookId('9994167158057')).toThrow('不正なISBNの形式です');
  });
});

```

ビジネスロジック、振る舞い (メソッド) 、例外処理などを網羅するようにテストを書きます。
値オブジェクトのテストではなるべくカバレッジが 100%に近づけるようにしましょう。

`jest`コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest
 PASS  src/Domain/models/Book/BookId/BookId.test.ts
  BookId
    ✓ 有効なフォーマットの場合正しい変換結果を期待 (11 ms)
    ✓ equals (1 ms)
    ✓ toISBN() 13桁
    ✓ toISBN() 10桁
    ✓ 不正な文字数の場合にエラーを投げる (46 ms)
    ✓ 不正なフォーマットの場合にエラーを投げる (4 ms)
```

エラーなく成功すれば OK です。

以上で`BookId`値オブジェクトの実装は完了です。ビジネスロジックを値にカプセル化し、テストによって品質を担保することができました。そして最初に値が抱えていた「**不正な状態で存在することが可能であり、正しい値が何かわからない**」という問題を解決することができました。

# プリミティブな値を値オブジェクトにする基準

値オブジェクトの素晴らしさを実感できたと思います。ですが、値オブジェクトの実装にはコストがかかるため、すべての値を値オブジェクトにするかどうかは慎重に判断する必要があります。そこで、値オブジェクトに適した値の特徴を確認しましょう。

- **意味のある単一の概念を表す値**
  通貨、距離、時間、範囲など、単一の概念や複数の関連する値を一つの単位として表現したい場合、それらの値の一貫性を保持しやすくなります。例えば、金額と通貨を一緒に扱うことで、通貨の変換や計算の一貫性が保たれます。
- **ビジネスルールを持つ**
  オブジェクトが独自のバリデーションルールやドメイン特有のビジネスロジック (メールアドレスの形式、電話番号の形式など) を必要とする場合、それらのデータの正確性が保たれます。
- **再利用性**
  同じ値がドメイン内の複数の箇所で必要な場合、再利用できることにより開発の効率が向上します。これは新しい機能の追加や、既存機能の拡張が容易になります。

上記の特徴やメリットと、実装コストを比較した上で導入するかどうかを判断しましょう。

:::message

> すべての値を値オブジェクトにするかどうかは慎重に判断する必要があります

`TypeScript`においては、安全性の観点でコストを払ってすべての値を値オブジェクトとして実装するメリットがあると考えます。`TypeScript`には**名前付き引数**がないため、プリミティブな型を利用すると以下のような問題に気づくのが難しくなります。

```js
class Person {
  constructor(
    public firstName: string,
    public lastName: string,
    public email: string,
    public phoneNumber: string,
  ) {}
}

const person = new Person(
  'John',
  'Doe',
  '09012341234', // emailとphoneNumberの順番を間違えてしまった。が、エラーが出ないため気づけない。
  'test@test.com',
);

```

すべての値を値オブジェクトとして定義することで、コンパイル時にエラーが出るため、間違いに気づくことができます。

```js
class Person {
  constructor(
    public firstName: FirstName,
    public lastName: LastName,
    public email: Email,
    public phoneNumber: PhoneNumber,
  ) {}
}

new Person(
  new FirstName('John'),
  new LastName('Doe'),
  new PhoneNumber('09012341234'), // エラーが出て順序の間違いに気づく
  new Email('test@test.com'),
);
```

:::

# 値オブジェクトのリファクタリング

次はリファクタリングを行いましょう。ここでは共通処理を抽象化します。たとえば、値オブジェクトの等価性を比較する`equals`メソッドは、すべての値オブジェクトで同じ実装を行う必要があります。これは、値オブジェクトの実装を重複させることになります。そこで、値オブジェクトの共通処理を抽象化することで、値オブジェクトの実装を簡潔にできます。
それではすべての値オブジェクトが継承する 共通クラス`ValueObject` を作成します。

```js:src/Domain/models/shared/ValueObject.ts
import { isEqual } from 'lodash';

export abstract class ValueObject<T, U> {
  // @ts-expect-error
  private _type: U;
  protected readonly _value: T;

  constructor(value: T) {
    this.validate(value);
    this._value = value;
  }

  protected abstract validate(value: T): void;

  get value(): T {
    return this._value;
  }

  equals(other: ValueObject<T, U>): boolean {
    return isEqual(this._value, other._value);
  }
}


```

:::message

どこからも参照されず、利用されることもないメンバ変数`_type`が定義されている理由を補足します。

```js
export abstract class ValueObject<T, U> {
  // @ts-expect-error
  private _type: U;
  (省略)
}
```

`TypeScript` は**構造的型付け** (Structural Typing) を採用しているため、型の互換性はその型が持つ構造 (プロパティやメソッド) に基づいて判断されます。
たとえば、以下のように`CustomerId`、`OrderId`2 つのクラスがあるとします。

```js
class CustomerId {
  constructor(public readonly id: string) {}
}

class OrderId {
  constructor(public readonly id: string) {}
}

const print = (customerId: CustomerId) => {
  console.log(customerId.id);
}

print(new OrderId('1')); // OrderIdを渡してもエラーにならない
```

これらは同じ構造を持っているため、`TypeScript` はこれらの型を同等とみなします。これは意図しないバグを引き起こす可能性があります。たとえば、`CustomerId` を期待する関数に `OrderId` を渡すことができてしまいます。このような型の混同を防ぐために、`_type` のような専用のプロパティを追加します。これにより、構造的には同じでも、このプライベートプロパティのおかげで異なる型として認識させることができます。

:::

次に、`BookId`クラスを`ValueObject`を継承するようにリファクタリングします。共通の処理が`ValueObject`に移動したため、`BookId`クラスは自身のビジネスロジックのみを実装することができ、簡潔になりました。

```js:src/Domain/models/Book/BookId/BookId.ts
import { ValueObject } from 'Domain/models/shared/ValueObject';

export class BookId extends ValueObject<string, 'BookId'> {
  static MAX_LENGTH = 13;
  static MIN_LENGTH = 10;

  constructor(value: string) {
    super(value);
  }
  protected validate(isbn: string): void {
   (省略)
  }
  private isValidIsbn10(isbn10: string): boolean {
   (省略)
  }
  private isValidIsbn13(isbn13: string): boolean {
   (省略)
  }
  toISBN(): string {
   (省略)
  }
}

```

リファクタリングを行ったのでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest
 PASS  src/Domain/models/Book/BookId/BookId.test.ts
```

エラーなく成功すれば OK です。

# すべての値オブジェクトの実装

それではここまでの知識を用いて、すべての値オブジェクトを実装していきましょう。それぞれの細かい説明はここでは省略しますが、基本的な考え方、実装の流れ、テストの書き方は`BookId`と同様です。

:::details Price

```js:src/Domain/models/Book/Price/Price.ts
import { ValueObject } from 'Domain/models/shared/ValueObject';

interface PriceValue {
  amount: number;
  currency: 'JPY'; // USD などの通貨を追加する場合はここに追加します
}

export class Price extends ValueObject<PriceValue, 'Price'> {
  static readonly MAX = 1000000;
  static readonly MIN = 1;

  constructor(value: PriceValue) {
    super(value);
  }

  protected validate(value: PriceValue): void {
    if (value.currency !== 'JPY') {
      throw new Error('現在は日本円のみを扱います。');
    }

    if (value.amount < Price.MIN || value.amount > Price.MAX) {
      throw new Error(
        `価格は${Price.MIN}円から${Price.MAX}円の間でなければなりません。`
      );
    }
  }

  get amount(): PriceValue['amount'] {
    return this.value.amount;
  }

  get currency(): PriceValue['currency'] {
    return this.value.currency;
  }
}

```

```js:src/Domain/models/Book/Price/Price.test.ts
import { Price } from './Price';

describe('Price', () => {
  // 正常系
  it('正しい値と通貨コードJPYで有効なPriceを作成する', () => {
    const validAmount = 500;
    const price = new Price({ amount: validAmount, currency: 'JPY' });
    expect(price.amount).toBe(validAmount);
    expect(price.currency).toBe('JPY');
  });

  // 異常系
  it('無効な通貨コードの場合エラーを投げる', () => {
    const invalidCurrency = 'USD';
    expect(() => {
      // @ts-expect-error テストのために無効な値を渡す
      new Price({ amount: 500, currency: invalidCurrency });
    }).toThrow('現在は日本円のみを扱います。');
  });

  it('MIN未満の値でPriceを生成するとエラーを投げる', () => {
    const lessThanMin = Price.MIN - 1;
    expect(() => {
      new Price({ amount: lessThanMin, currency: 'JPY' });
    }).toThrow(
      `価格は${Price.MIN}円から${Price.MAX}円の間でなければなりません。`
    );
  });

  it('MAX超の値でPriceを生成するとエラーを投げる', () => {
    const moreThanMax = Price.MAX + 1;
    expect(() => {
      new Price({ amount: moreThanMax, currency: 'JPY' });
    }).toThrow(
      `価格は${Price.MIN}円から${Price.MAX}円の間でなければなりません。`
    );
  });
});
```

:::

:::details Title

```js:src/Domain/models/Book/Title/Title.ts
import { ValueObject } from 'Domain/models/shared/ValueObject';

type TitleValue = string;
export class Title extends ValueObject<TitleValue, 'Title'> {
  static readonly MAX_LENGTH = 1000;
  static readonly MIN_LENGTH = 1;

  constructor(value: TitleValue) {
    super(value);
  }

  protected validate(value: TitleValue): void {
    if (value.length < Title.MIN_LENGTH || value.length > Title.MAX_LENGTH) {
      throw new Error(
        `タイトルは${Title.MIN_LENGTH}文字以上、${Title.MAX_LENGTH}文字以下でなければなりません。`
      );
    }
  }
}


```

```js:src/Domain/models/Book/Title/Title.test.ts
import { Title } from './Title';

describe('Title', () => {
  test('Titleが1文字で作成できる', () => {
    expect(new Title('a').value).toBe('a');
  });

  test('Titleが1000文字で作成できる', () => {
    const longTitle = 'a'.repeat(1000);
    expect(new Title(longTitle).value).toBe(longTitle);
  });

  test('最小長以上の値でTitleを生成するとエラーを投げる', () => {
    expect(() => new Title('')).toThrow(
      'タイトルは1文字以上、1000文字以下でなければなりません。'
    );
  });

  test('最大長以上の値でTitleを生成するとエラーを投げる', () => {
    const tooLongTitle = 'a'.repeat(1001);
    expect(() => new Title(tooLongTitle)).toThrow(
      'タイトルは1文字以上、1000文字以下でなければなりません。'
    );
  });
});


```

:::

:::details StockId

```js:src/Domain/models/Book/Stock/StockId.ts
import { ValueObject } from 'Domain/models/shared/ValueObject';
import { nanoid } from 'nanoid';

type StockIdValue = string;
export class StockId extends ValueObject<StockIdValue, 'StockId'> {
  static readonly MAX_LENGTH = 100;
  static readonly MIN_LENGTH = 1;

  constructor(value: StockIdValue = nanoid()) { // デフォルトではnanoidを利用しID生成
    super(value);
  }

  protected validate(value: StockIdValue): void {
    if (
      value.length < StockId.MIN_LENGTH ||
      value.length > StockId.MAX_LENGTH
    ) {
      throw new Error(
        `StockIdは${StockId.MIN_LENGTH}文字以上、${StockId.MAX_LENGTH}文字以下でなければなりません。`
      );
    }
  }
}

```

```js:src/Domain/models/Book/Stock/StockId.test.ts
import { StockId } from './StockId';

// nanoid() をモックする
jest.mock('nanoid', () => ({
  nanoid: () => 'testIdWithExactLength',
}));

describe('StockId', () => {
  test('デフォルトの値でStockIdを生成する', () => {
    const stockId = new StockId();
    expect(stockId.value).toBe('testIdWithExactLength');
  });

  test('指定された値でStockIdを生成する', () => {
    const value = 'customId';
    const stockId = new StockId(value);
    expect(stockId.value).toBe(value);
  });

  test('最小長以下の値でStockIdを生成するとエラーを投げる', () => {
    const shortValue = '';
    expect(() => new StockId(shortValue)).toThrowError(
      new Error(
        `StockIdは${StockId.MIN_LENGTH}文字以上、${StockId.MAX_LENGTH}文字以下でなければなりません。`
      )
    );
  });

  test('最大長以上の値でStockIdを生成するとエラーを投げる', () => {
    const longValue = 'a'.repeat(StockId.MAX_LENGTH + 1);
    expect(() => new StockId(longValue)).toThrowError(
      new Error(
        `StockIdは${StockId.MIN_LENGTH}文字以上、${StockId.MAX_LENGTH}文字以下でなければなりません。`
      )
    );
  });
});

```

:::

:::details QuantityAvailable

```js:src/Domain/models/Book/Stock/QuantityAvailable.ts
import { ValueObject } from 'Domain/models/shared/ValueObject';

type QuantityAvailableValue = number;
export class QuantityAvailable extends ValueObject<
  QuantityAvailableValue,
  'QuantityAvailable'
> {
  static readonly MAX: number = 1000000;
  static readonly MIN: number = 0;

  constructor(value: QuantityAvailableValue) {
    super(value);
  }

  protected validate(value: QuantityAvailableValue): void {
    if (value < QuantityAvailable.MIN || value > QuantityAvailable.MAX) {
      throw new Error(
        `在庫数は${QuantityAvailable.MIN}から${QuantityAvailable.MAX}の間でなければなりません。`
      );
    }
  }

  increment(amount: number): QuantityAvailable {
    const newValue = this._value + amount;

    return new QuantityAvailable(newValue);
  }

  decrement(amount: number): QuantityAvailable {
    const newValue = this._value - amount;

    return new QuantityAvailable(newValue);
  }
}

```

```js:src/Domain/models/Book/Stock/QuantityAvailable.test.ts
import { QuantityAvailable } from './QuantityAvailable';

describe('QuantityAvailable', () => {
  it('許容される範囲内の在庫数を設定できる', () => {
    const validQuantityAvailable = 500;
    const quantity = new QuantityAvailable(validQuantityAvailable);
    expect(quantity.value).toBe(validQuantityAvailable);
  });

  it('MIN未満の値でQuantityAvailableを生成するとエラーを投げる', () => {
    const lessThanMin = QuantityAvailable.MIN - 1;
    expect(() => new QuantityAvailable(lessThanMin)).toThrow(
      `在庫数は${QuantityAvailable.MIN}から${QuantityAvailable.MAX}の間でなければなりません。`
    );
  });

  it('MAX超の値でQuantityAvailableを生成するとエラーを投げる', () => {
    const moreThanMax = QuantityAvailable.MAX + 1;
    expect(() => new QuantityAvailable(moreThanMax)).toThrow(
      `在庫数は${QuantityAvailable.MIN}から${QuantityAvailable.MAX}の間でなければなりません。`
    );
  });

  describe('increment', () => {
    it('正の数を加算すると、在庫数が増加する', () => {
      const initialQuantity = new QuantityAvailable(10);
      const incrementAmount = 5;
      const newQuantity = initialQuantity.increment(incrementAmount);

      expect(newQuantity.value).toBe(15);
    });

    it('最大値を超える加算を試みるとエラーが発生する', () => {
      const initialQuantity = new QuantityAvailable(QuantityAvailable.MAX);
      const incrementAmount = 1;

      expect(() => initialQuantity.increment(incrementAmount)).toThrow(
        `在庫数は${QuantityAvailable.MIN}から${QuantityAvailable.MAX}の間でなければなりません。`
      );
    });
  });

  describe('decrement', () => {
    it('正の数を減算すると、在庫数が減少する', () => {
      const initialQuantity = new QuantityAvailable(10);
      const decrementAmount = 5;
      const newQuantity = initialQuantity.decrement(decrementAmount);

      expect(newQuantity.value).toBe(5);
    });

    it('在庫数を負の数に減算しようとするとエラーが発生する', () => {
      const initialQuantity = new QuantityAvailable(0);
      const decrementAmount = 1;

      expect(() => initialQuantity.decrement(decrementAmount)).toThrow(
        `在庫数は${QuantityAvailable.MIN}から${QuantityAvailable.MAX}の間でなければなりません。`
      );
    });
  });
});

```

:::

:::details Status

```js:src/Domain/models/Book/Stock/Status.ts
import { ValueObject } from 'Domain/models/shared/ValueObject';

export enum StatusEnum {
  InStock = 'InStock',
  LowStock = 'LowStock',
  OutOfStock = 'OutOfStock',
}
export type StatusLabel = '在庫あり' | '残りわずか' | '在庫切れ';

type StatusValue = StatusEnum;
export class Status extends ValueObject<StatusValue, 'Status'> {
  constructor(value: StatusValue) {
    super(value);
  }

  protected validate(value: StatusValue): void {
    if (!Object.values(StatusEnum).includes(value)) {
      throw new Error('無効なステータスです。');
    }
  }

  toLabel(): StatusLabel {
    switch (this._value) {
      case StatusEnum.InStock:
        return '在庫あり';
      case StatusEnum.LowStock:
        return '残りわずか';
      case StatusEnum.OutOfStock:
        return '在庫切れ';
    }
  }
}


```

```js:src/Domain/models/Book/Stock/Status.test.ts
import { Status, StatusEnum } from './Status';

describe('Status', () => {
  it('有効なステータスでインスタンスが生成されること', () => {
    expect(new Status(StatusEnum.InStock).value).toBe(StatusEnum.InStock);
    expect(new Status(StatusEnum.OutOfStock).value).toBe(StatusEnum.OutOfStock);
    expect(new Status(StatusEnum.LowStock).value).toBe(StatusEnum.LowStock);
  });

  it('無効なステータスでエラーが投げられること', () => {
    const invalidStatus = 'invalid' as StatusEnum; // テストのために無効な値を渡す
    expect(() => new Status(invalidStatus)).toThrow('無効なステータスです。');
  });

  describe('toLabel()', () => {
    it('ステータスInStockが「在庫あり」に変換されること', () => {
      const status = new Status(StatusEnum.InStock);
      expect(status.toLabel()).toBe('在庫あり');
    });

    it('ステータスOutOfStockが「在庫切れ」に変換されること', () => {
      const status = new Status(StatusEnum.OutOfStock);
      expect(status.toLabel()).toBe('在庫切れ');
    });

    it('ステータスLowStockが「残りわずか」に変換されること', () => {
      const status = new Status(StatusEnum.LowStock);
      expect(status.toLabel()).toBe('残りわずか');
    });
  });
});


```

:::

:::message
値オブジェクトをすべて手動で作成するのは大変な作業です。そこで、値オブジェクトを自動生成するツールを作成するか、ChatGPT などの生成 AI を用いて、コードを**自動生成**することをオススメします。
:::

最後に全値オブジェクトのテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest
```

以上で全ての値オブジェクトの実装が完了しました。

# まとめ

- 値オブジェクトは値である
- 値オブジェクトを利用することで、不正な値が存在する可能性を減らすことができる
- 値オブジェクト自身がドメイン内の値のドキュメントになる

本章では、値オブジェクトの実装方法と、値オブジェクトのメリットについて学びました。値オブジェクトは、ドメインの知識を表現するために欠かせない重要な概念です。本章では取り扱いませんでしたが、`StringValueObject`、`NumberValueObject`、 `EnumValueObject` など、もう一段抽象的なクラスを作成したりプラス α の機能を追加するなどのカスタマイズも可能です。ぜひ、値オブジェクトを活用してみてください。
次章は、今回作成した値オブジェクトを利用して、エンティティを実装していきます。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/valueObject

# 参考文献

https://qiita.com/suin/items/57cfc0ec9bb1a6995aa5
