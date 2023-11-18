---
title: '集約'
---

# 集約とは

集約とは、関連するオブジェクト群を 1 つのユニットとして管理するための手法です。集約は、1 つの**ルートエンティティ**（集約ルート）と、それに関連するエンティティや値オブジェクトで構成されます。集約は、ビジネスルールとデータの**整合性**を維持するために設計されます。たとえば、「書籍」と「在庫」には整合性が必要です。書籍を削除できるのは在庫が存在しない場合のみで、書籍を削除したら在庫データも削除されなければデータに矛盾が生じてしまいます。このような整合性を保つためには、書籍と在庫を集約として定義する必要があります。

また、集約は**リポジトリ** の入出力の単位になります。集約の設計によって整合性が保たれたデータを確実に管理することができます。集約の設計が適切であれば、データベースに保存されているデータは、ビジネスルールに従って常に一貫性を保つことが可能になります。

:::message
リポジトリとはデータの永続化を行うコンポーネントで、入出力の単位は**集約**となります。集約で整合性が保たれたデータをそのままデータベース反映するために、集約内のデータは同じトランザクション内ですべて更新されます。
リポジトリの詳細は chapter11 リポジトリの章で学びます。
:::

# Book 集約の実装

それでは、集約ルートである 「Book」 ルートエンティティを実装していきましょう。まずはドメインモデリングで作成した 「BookAggregation」 を振り返ってみましょう。値オブジェクトとエンティティ (Stock)を属性として持っていることがわかります。

## デメテルの法則

コードの実装に入る前に集約にとって重要な**デメテルの法則**（Law of Demeter）について説明します。デメテルの法則は、オブジェクト指向プログラミングにおける設計原則の一つです。この法則は、とくにオブジェクト間の相互作用に焦点を当てており、「最小知識の原則」とも呼ばれます。この原則に従うことで、システム内の異なるオブジェクト間の密な結合を減らし、より保守しやすく、理解しやすいコードを作成できます。

### デメテルの法則の基本原則

[wikipedia](https://ja.wikipedia.org/wiki/%E3%83%87%E3%83%A1%E3%83%86%E3%83%AB%E3%81%AE%E6%B3%95%E5%89%87) ではデメテルの法則では、オブジェクト O 上のメソッド M が呼び出してもよいメソッドは以下のオブジェクトに属するメソッドのみに限定されると説明されています。

1. O それ自身
2. M の引数に渡されたオブジェクト
3. M の内部でインスタンス化されたオブジェクト
4. O を直接的に構成するオブジェクト（O のインスタンス変数）

文章だけでは理解しづらいので、簡単な例を用いて確認しましょう。

```js
// デメテルの法則適用前のコード例
class Stock {
  constructor(public _quantityAvailable: number) {}

  get quantityAvailable(): number {
    return this._quantityAvailable;
  }
}

class Book {
  constructor(public stock: Stock) {}
}

const quantityAvailable = new Book(new Stock(100)).stock.quantityAvailable;

```

この例では、`Book` クラスは `Stock` クラスを通じて `quantityAvailable` にアクセスしています。これは`Book` クラスが `Stock` クラス の内部構造（Stock が quantityAvailable を持っていること）に依存しており、結合度が高いと考えられます。

```js
// デメテルの法則適用後のコード例
class Stock {
  constructor(public _quantityAvailable: QuantityAvailable) {}

  get quantityAvailable(): QuantityAvailable {
    return this._quantityAvailable;
  }
}

class Book {
  constructor(private stock: Stock) {}

  getQuantityAvailable() {
    return this.stock.quantityAvailable;
  }
}

const quantityAvailable = new Book(new Stock(100)).getQuantityAvailable();
```

この例では、`Book` クラス は 自身のメソッドを通じて `Stock` クラスの `quantityAvailable` を取得しています。これにより、`Book` クラスは `Stock` クラス の内部構造に依存することなく、より疎結合なコードになります。

### 集約とデメテルの法則

デメテルの法則は集約実装のガイドラインとなります。集約は、関連オブジェクトを 1 つのユニットとして管理します。集約の内部オブジェクトが密接に関連し合っている一方で、集約の外部からの操作は必ず集約ルートを介して行われなければいけません。上記の例で言うと、`Stock` エンティティは `Book`集約ルートでのみ操作できるように制御する必要があります。これはデメテルの法則に沿っています。このようにすることで、集約外のオブジェクトが集約内の詳細について知る必要がなくなるため、疎結合になります。

また、集約の内部構造や状態に直接アクセスすることができなくなるため、ドメインルールの漏洩や整合性が破壊されることを防ぐことができます。

## 実装

それでは`Book.ts`ファイルを作成し、デメテルの法則にしたがって `Book` ルートエンティティを実装していきましょう。

```js:StockManagement/Domain/models/Book/Book.ts
import { BookId } from './BookId/BookId';
import { Price } from './Price/Price';
import { StatusEnum } from './Stock/Status/Status';
import { Stock } from './Stock/Stock';
import { Title } from './Title/Title';

export class Book {
  private constructor(
    private readonly _bookId: BookId,
    private _title: Title,
    private _price: Price,
    private readonly _stock: Stock
  ) {}

  static create(bookId: BookId, title: Title, price: Price) {
    return new Book(bookId, title, price, Stock.create());
  }

  static reconstruct(bookId: BookId, title: Title, price: Price, stock: Stock) {
    return new Book(bookId, title, price, stock);
  }

  delete() {
    // stockが削除可能か確認する
    this._stock.delete();
    // Bookを削除する処理があればここに書く
  }

  changeTitle(newTitle: Title) {
    this._title = newTitle;
  }

  changePrice(newPrice: Price) {
    this._price = newPrice;
  }

  // 販売可能かどうか
  isSaleable() {
    return (
      this._stock.quantityAvailable.value > 0 &&
      this._stock.status.value !== StatusEnum.OutOfStock
    );
  }

  increaseStock(amount: number) {
    this._stock.increaseQuantity(amount);
  }

  decreaseStock(amount: number) {
    this._stock.decreaseQuantity(amount);
  }

  get bookId(): BookId {
    return this._bookId;
  }

  get title(): Title {
    return this._title;
  }

  get price(): Price {
    return this._price;
  }

  get stockId() {
    return this._stock.stockId;
  }

  get quantityAvailable() {
    return this._stock.quantityAvailable;
  }

  get status() {
    return this._stock.status;
  }
}

```

基本的な設計方針は前章で学んだエンティティと同じです。違いは stock エンティティの値やメソッドへの参照がすべて Book ルートエンティティを介して行われている点です。これにより、デメテルの法則に沿ったコードになっています。

# Book 集約のテスト

それではテストを書いていきましょう。「Book.test.ts」を作成し、書いていきます。

```js:StockManagement/Domain/models/Book/Book.test.ts
import { Book } from './Book';
import { BookId } from './BookId/BookId';
import { Title } from './Title/Title';
import { Price } from './Price/Price';
import { Stock } from './Stock/Stock';
import { StockId } from './Stock/StockId/StockId';
import { QuantityAvailable } from './Stock/QuantityAvailable/QuantityAvailable';
import { Status, StatusEnum } from './Stock/Status/Status';

// nanoid() をモックする
jest.mock('nanoid', () => ({
  nanoid: () => 'testIdWithExactLength',
}));

describe('Book', () => {
  const stockId = new StockId('abc');
  const quantityAvailable = new QuantityAvailable(100);
  const status = new Status(StatusEnum.InStock);
  const stock = Stock.reconstruct(stockId, quantityAvailable, status);

  const bookId = new BookId('9784167158057');
  const title = new Title('吾輩は猫である');
  const price = new Price({
    amount: 770,
    currency: 'JPY',
  });

  describe('create', () => {
    it('デフォルト値で在庫を作成する', () => {
      const book = Book.create(bookId, title, price);

      expect(book.bookId.equals(bookId)).toBeTruthy();
      expect(book.title.equals(title)).toBeTruthy();
      expect(book.price.equals(price)).toBeTruthy();
      expect(
        book.stockId.equals(new StockId('testIdWithExactLength'))
      ).toBeTruthy();
      expect(
        book.quantityAvailable.equals(new QuantityAvailable(0))
      ).toBeTruthy();
      expect(
        book.status.equals(new Status(StatusEnum.OutOfStock))
      ).toBeTruthy();
    });
  });

  describe('delete', () => {
    it('在庫ありの場合はエラーを投げる', () => {
      const book = Book.reconstruct(bookId, title, price, stock);

      expect(() => book.delete()).toThrow('在庫がある場合削除できません。');
    });

    it('在庫なしの場合はエラーを投げない', () => {
      const notOnSaleStatus = new Status(StatusEnum.OutOfStock);
      const notQuantityAvailable = new QuantityAvailable(0);
      const stock = Stock.reconstruct(
        stockId,
        notQuantityAvailable,
        notOnSaleStatus
      );
      const book = Book.reconstruct(bookId, title, price, stock);

      expect(() => book.delete()).not.toThrow();
    });
  });

  describe('isSaleable', () => {
    it('在庫あり、在庫数が整数の場合はtrueを返す', () => {
      const stock = Stock.reconstruct(stockId, quantityAvailable, status);
      const book = Book.reconstruct(bookId, title, price, stock);
      expect(book.isSaleable()).toBeTruthy();
    });

    it('在庫なし、在庫数0の場合はfalseを返す', () => {
      const notOnSaleStatus = new Status(StatusEnum.OutOfStock);
      const notQuantityAvailable = new QuantityAvailable(0);
      const stock = Stock.reconstruct(
        stockId,
        notQuantityAvailable,
        notOnSaleStatus
      );
      const book = Book.reconstruct(bookId, title, price, stock);
      expect(book.isSaleable()).toBeFalsy();
    });
  });

  describe('increaseStock', () => {
    it('stock.increaseQuantityが呼ばれる', () => {
      const book = Book.reconstruct(bookId, title, price, stock);
      const spy = jest.spyOn(stock, 'increaseQuantity');
      book.increaseStock(10);
      expect(spy).toHaveBeenCalled();
    });
  });

  describe('decreaseStock', () => {
    it('stock.decreaseQuantityが呼ばれる', () => {
      const book = Book.reconstruct(bookId, title, price, stock);
      const spy = jest.spyOn(stock, 'decreaseQuantity');
      book.decreaseStock(10);
      expect(spy).toHaveBeenCalled();
    });
  });

  describe('changeTitle', () => {
    it('titleを変更する', () => {
      const book = Book.reconstruct(bookId, title, price, stock);
      const newTitle = new Title('坊ちゃん');
      book.changeTitle(newTitle);
      expect(book.title.equals(newTitle)).toBeTruthy();
    });
  });

  describe('changePrice', () => {
    it('priceを変更する', () => {
      const book = Book.reconstruct(bookId, title, price, stock);
      const newPrice = new Price({
        amount: 880,
        currency: 'JPY',
      });
      book.changePrice(newPrice);
      expect(book.price.equals(newPrice)).toBeTruthy();
    });
  });
});

```

:::message
`increaseStock`メソッドではデメテルの法則に従い、`Stock` エンティティの`increaseQuantity`メソッドを呼び出しています。`increaseQuantity`メソッドのロジックのテストは`Stock.test.ts`で行っているため、`Book` ルートエンティティの`increaseStock`メソッドのテストでは、`Stock` エンティティの `increaseQuantity` メソッドが呼び出されていることだけ確認します。そのようにテストすることで、`Stock` エンティティの`increaseQuantity`メソッドのロジックが変更されても、修正は`Stock.test.ts`だけで済み、`Book.test.ts`の修正は不要になります。
メソッドが呼び出されているか確認するためには、`jest.spyOn`を使用します。`jest.spyOn`は、オブジェクトのメソッドが呼び出されたかどうかを確認するためのモック関数を作成します。`jest.spyOn`を使用することで、`Stock` エンティティの`increaseQuantity`メソッドが呼び出されていることを確認することができます。

```js
it('stock.increaseQuantityが呼ばれる', () => {
  const book = Book.reconstruct(bookId, title, price, stock);
  const spy = jest.spyOn(stock, 'increaseQuantity');
  book.increaseStock(10);
  expect(spy).toHaveBeenCalled();
});
```

:::

なるべくテストのカバレッジが 100%に近づけるようにしましょう。
jest コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest Book.test.ts
```

# 集約の大きさ

集約の設計をする上で一番難しいのが、集約の範囲を決めることです。基本的に、集約の範囲はなるべく小さくしたほうが良いとされています。章のはじめに、集約はビジネスルールとデータの**整合性**を維持するために設計すると説明しましたが、「整合性」だけに焦点を当てて設計をしてしまうと集約の範囲が肥大しがちです。たとえば、**書籍販売コンテキスト**の**書籍**には「著者」「出版社」「レビュー」などの要素があります。それらをまとめて 1 つの集約にすると以下のようになります。

```js
class Book {
  constructor(
    private readonly _bookId: BookId, // 値オブジェクト
    private  _title: Title, // 値オブジェクト
    private  _author: Author, // エンティティ
    private  _publisher: Publisher, // エンティティ
    private  _reviews: Review[], // エンティティ
  ) {}
}
```

これらすべてを一つの集約として捉えようとすると、その集約は複雑になり、管理が困難になります。集約は 1 集約 1 トランザクションでデータベースに反映しなければなりません。このため、書籍の一部データを更新するために集約内のすべての要素を一度データベースから読み込み、すべて同時に更新する必要が生じます。これは明らかな**オーバーヘッド**です。すべての関連要素を一括で扱うことで、システムの**パフォーマンスに悪影響**を及ぼします。

また、集約の範囲が大きいと**トランザクションのロック**時間が長くなり、データベースのパフォーマンスが悪化したり障害が発生しやすくなります。たとえば「書籍」とレ「ビュー」は 1 対 n の関係性にあります。仮にレビュー数が 10000 件ある書籍データを更新する場合、10000 件のレビューを更新する処理が同じトランザクション内で行われます。レビューの更新処理が一つでも失敗した場合、書籍の更新処理も失敗するため、書籍の更新ができなくなってしまいます。

これらの理由から集約はなるべく小さく区切ることが望ましいです。

## 集約をどこで区切るか

大きな集約がもたらす問題を避けるためには、集約の範囲をどこで区切るかを考える必要があります。集約の範囲に迷った際には、2 つの基準を参考にしてみてください。

- ルートエンティティ 1 に対して n の関係性にあるエンティティがある場合、保有数の上限が適切か
- 強整合性 (トランザクション整合性) が必要か

それでは、それぞれ説明します。

:::message
ベストな集約の範囲を見つけるのは難しく正解は複数あると考えています。迷ったときはとりあえず実装してみるのも一つの手です。よくない集約は実装してみることで、扱いにくかったりパフォーマンスに問題があったりすることがわかります。その結果を踏まえて集約の範囲を見直し、ブラッシュアップしていきましょう。
:::

### ルートエンティティ 1 に対して n の関係性にあるエンティティがある場合、保有数の上限が適切か

ルートエンティティに対する関係性は重要な要素です。ルートエンティティと保有するエンティティの間の関係が 1 対 n の場合、その n の数には上限があるかどうかによって同一集約に含めた方が良いかどうかが決まります。
たとえば、「書籍」と「レビュー」の関係性は 1 対 n ですが、レビューの数に上限があるでしょうか？それはビジネスの要件や非機能要件によって変動します。仮にレビュー数の上限が 100 件までと決まっているとします。MAX でも 100 件程度であれば、「書籍」と「レビュー」を同一集約に含めることによる問題よりも、集約によって整合性を確実に担保するメリットが勝るかもしれません。

```js
class Book {
  (省略)
  addReview(review: Review) {
    //レビューの追加時にレビュー数の上限のチェックを行い、整合性を保つことができる
    if (this._reviews.length >= 100) {
      throw new Error('レビューは100件までです。');
    }

    this._reviews.push(review);
  }
}
```

しかし、レビュー数の上限が非常に大きい場合やそもそも上限が決められていない場合は、パフォーマンスやトランザクションのロックの問題が発生する可能性が高くなります。そのため、「書籍」と「レビュー」を別の集約にすることを検討する必要があります。

### 強整合性 (トランザクション整合性) が必要か

集約の範囲を決定する際、もう一つの重要な考慮事項は**強整合性**、つまり**トランザクション整合性**の必要性です。すべてのデータが常に正確で最新の状態である必要がある場合、それらのデータは同じ集約内に含めるべきです。しかし、すべての場合において強い整合性が必要というわけではありません。

たとえば、「書籍」と「レビュー」の関係で考えてみましょう。書籍情報は、タイトルや著者など重要なビジネスデータを含んでおり、これらの情報の整合性は重要です。一方で、レビューはユーザーによる個人的な意見や評価を含み、書籍情報の整合性と直接関連するわけではありません。

また、レビュー数に上限があった場合はどうでしょう。レビュー数の整合性も重要になります。しかし、仮に整合性が崩れ上限を超えてレビューが投稿されたとしても、ビジネス的にクリティカルな問題にはならないでしょう。

このように本当に強整合性が必要なのかどうかをいくつかの要素で判断することができます。強整合性の必要性が低い、または必要ない場合はデータを別の集約として扱い整合性の要求を緩和することが可能です。「書籍」と「レビュー」の例で言えば、別の集約として扱ってもよいという判断ができます。

# 集約のルール

複数の集約同士には、ルールがあります。

- 集約間は 識別子 (ID)で 参照する
- 集約の外部では結果整合性を用いる

それでは、それぞれ説明します。

## 集約間は 識別子(ID)で 参照する

id 参照にすることで集約のメソッドを呼び出せなくなるため、集約の不整合を防ぐことができる。

## 集約の外部では結果整合性を用いる

「書籍」と「レビュー」の上限の整合が取れないわけではない。

# まとめ

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/domain-modeling
