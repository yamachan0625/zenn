---
title: 'エンティティ'
---

# エンティティ (Entity) とは

エンティティは、値オブジェクトと並びドメインモデル(ドメインオブジェクト)の中心的な要素で、ドメイン内のさまざまな**ビジネスの実体**の概念をモデル化するのに用いられます。例えば書籍、在庫、ユーザー、履歴などが挙げられます。

## 値オブジェクトとの違い

エンティティと値オブジェクトは、どちらもドメイン駆動設計（DDD）におけるドメインモデルの中心的な要素ですが、別物です。それらを区別する概念は **同一性** (Identity) にあります。

### エンティティの同一性

エンティティは「誰であるか」や「何であるか」という概念によって同一性が定義されます。エンティティを説明するのによく「人」が例に挙げられます。人には名前や住所、年齢などの属性がありますが、それらの属性が変わっても、その人は同じ人であり続けます。たとえば、誕生日を迎え、年齢が変わってもその人は同じ人です。エンティティの同一性は、それを構成する属性の値に依存しません。たとえ属性が変わっても、そのエンティティの同一性は変わりません。

そして、エンティティは同一であるという概念を、一意に識別する 「ID」 を割り当てることで表現します。この 「ID」 によってエンティティのインスタンスは区別されます。たとえ属性が時間の経過と共に変わっても、その「ID」 が同じであれば、それは同一のエンティティと見なされます。

```js
class Person {
  constructor(
    public readonly personId: string,
    public name: string,
    public age: number,
    public address: string
  ) {}
}

// 山田太郎さんが誕生
const person1 = new Person('1', '山田太郎', 0, '東京都');
person1.age = 1; // 東京の山田太郎さんが誕生日を迎え、年齢が1歳になった。

// 一意な識別子「personId」が同一であるため、同一のエンティティと見なされる
console.log(person1); // Person { personId: '1', name: '山田太郎', age: 1, address: '東京都' }
```

### 値オブジェクトの同一性

一方で、値オブジェクトはその属性によって同一性が定義されるものです。値オブジェクトには識別子がなく、その属性の値がすべてであり、それらの値が同じであれば、それは同じ値オブジェクトと見なされます。値オブジェクトの同一性は、それを構成する属性の値の組み合わせに依存します。たとえば、`BookId`を表す値オブジェクトが「9784167158057」という属性を持ち、もしほかの`BookId`値オブジェクトも全く同じ属性を持っていれば、それらは区別されず同じものとして扱われます。詳しくは前章 (値オブジェクト) で説明しています。

```js
const bookId1 = new BookId('9784167158057');
const bookId2 = new BookId('9784167158057');

// 同一であることの確認
console.log(bookId1.equals(bookId2)); // true
```

## エンティティの特徴

値同様エンティティにもいくつかの特徴があります。

- 一意な識別子によって区別される
- 可変である
- ライフサイクルがある

ではそれぞれ確認していきましょう。

### 一意な識別子によって区別される

さきほど説明した通り、エンティティはその一意な識別子によって区別されます。この識別子はエンティティが生成された瞬間に割り当てられ、そのライフサイクルの終わりまで変わることはありません。この識別子のおかげで、属性が時間と共に変化しても、エンティティの同一性は保たれ続けます。

### 可変である

エンティティは値オブジェクトとは反対に、その状態が変更可能です。属性や関連するオブジェクトが変更されることがあり、エンティティの状態はその内容を反映できます。

### ライフサイクルがある

エンティティには明確なライフサイクルが存在します。生成、変更、そして場合によっては削除というプロセスを経ることで、エンティティは時間の経過と共にビジネスプロセスに沿って変化します。

# エンティティの実装

エンティティの特徴が確認できたので、実際に「Stock」エンティティを例にエンティティを実装していきましょう。まずはドメインモデリングで作成した 「Stock」 エンティティを振り返ってみましょう。Stock エンティティが持つ属性やビジネスルールは以下の通りです。

```plantuml:StockManagement/Domain/models/Book/Stock/Stock.pu
@startuml Stock

!include ./Status/Status.pu
!include ./QuantityAvailable/QuantityAvailable.pu
!include ./StockId/StockId.pu

class "Stock(在庫)" as Stock << (E,green) Entity >> {
    StockId: StockId
    QuantityAvailable: 在庫数
    Status: ステータス
}

Stock *-down- StockId
Stock *-down- QuantityAvailable
Stock *-down- Status

note bottom of Stock
    - 初回作成時、ステータスは「在庫切れ」から始まる。
    - 在庫数は0の場合は在庫切れ。10以下の場合は残りわずか。それ以外は在庫あり。
end note

@enduml

```

## 実装

それでは`Stock.ts`ファイルを作成し実装していきましょう。以下がエンティティのベースになります。

```js:StockManagement/Domain/models/Book/Stock/Stock.ts
import { QuantityAvailable } from './QuantityAvailable/QuantityAvailable';
import { Status, StatusEnum } from './Status/Status';
import { StockId } from './StockId/StockId';

export class Stock {
  private constructor(
    private readonly _stockId: StockId, // 識別子は変更不可のためreadonlyにする
    private _quantityAvailable: QuantityAvailable,
    private _status: Status
  ) {}

  // 新規エンティティの生成
  static create() {
    const defaultStockId = new StockId(); // 自動ID採番
    const defaultQuantityAvailable = new QuantityAvailable(0);
    const defaultStatus = new Status(StatusEnum.OutOfStock);

    return new Stock(defaultStockId, defaultQuantityAvailable, defaultStatus);
  }

  delete() {
    if (this.status.value !== StatusEnum.OutOfStock) {
      throw new Error('在庫がある場合削除できません。');
    }
  }

  private changeStatus(newStatus: Status) {
    this._status = newStatus;
  }

  private changeQuantityAvailable(newQuantityAvailable: QuantityAvailable) {
    this._quantityAvailable = newQuantityAvailable;
  }

  // エンティティの再構築
  static reconstruct(
    stockId: StockId,
    quantityAvailable: QuantityAvailable,
    status: Status
  ) {
    return new Stock(stockId, quantityAvailable, status);
  }

  get stockId(): StockId {
    return this._stockId;
  }

  get quantityAvailable(): QuantityAvailable {
    return this._quantityAvailable;
  }

  get status(): Status {
    return this._status;
  }
}

```

:::message
**private constructor** にしている理由は、**create メソッドと reconstruct メソッドのみでエンティティを生成することを強制**するためです。`create` メソッドでは、エンティティの生成時の制御 (ビジネスルールの適用) を行います。`reconstruct` メソッドは、データベースなどから読み込んだデータをもとにエンティティを再構築する際に使用します。`reconstruct` メソッド は本章では使用しません。詳しくはリポジトリの章 TODO で説明します。

:::

それでは、Stock クラスを用いて、エンティティの特徴がどのように実装されているかを確認しましょう。

### 一意な識別子によって区別される

`StockId` は一意な識別子です。この `StockId` はエンティティの生成時に割り当てられ、そのライフサイクルの終わりまで変わることはありません。そのため、`StockId` はエンティティのコンストラクタでのみ設定され、その後変更されることはありません。readonly 修飾子を用いて、コンストラクタ以外での変更を防ぎます。

```js
const stock: Stock = Stock.create(
  new StockId(),
  new QuantityAvailable(0),
  new Status(StatusEnum.PreSale)
);

stock.stockId = new StockId('stockId2'); // StockIdはreadonlyなので変更できない
```

### 可変である

メソッドを用いて、エンティティの状態を変更することができます。

```js
const stock = Stock.create(
  new StockId('stockId'),
  new QuantityAvailable(0),
  new Status(StatusEnum.PreSale)
);

stock.changeStatus(new Status(StatusEnum.OnSale));
console.log(stock.status); // Status { _value: 'OnSale' }

stock.changeQuantityAvailable(new QuantityAvailable(100));
console.log(stock.quantityAvailable); // QuantityAvailable { _value: 100 }
```

### ライフサイクルがある

`create`、`change`、`delete` メソッドを用いて、エンティティのライフサイクルを表現することができます。

```js
const stock = Stock.create(省略);
stock.changeStatus(省略);
stock.delete(省略);
```

## ビジネスルールの適用

実装したエンティティはまだ未完成です。今のままではビジネスルールに反したエンティティのライフサイクルが発生してしまいます。例えば`在庫数が0`の状態でステータスが`在庫あり`のエンティティが生成できてしまいます。そこで、エンティティのライフサイクルにビジネスルールを適用する必要があります。それでは、ビジネスルールを適用していきましょう。

```js:StockManagement/Domain/models/Book/Stock/Stock.ts
import { QuantityAvailable } from './QuantityAvailable/QuantityAvailable';
import { Status, StatusEnum } from './Status/Status';
import { StockId } from './StockId/StockId';

export class Stock {
  private constructor(
    private readonly _stockId: StockId,
    private _quantityAvailable: QuantityAvailable,
    private _status: Status
  ) {}

  // 新規エンティティの生成
  static create() {
    const defaultStockId = new StockId(); // 自動ID採番
    const defaultQuantityAvailable = new QuantityAvailable(0);
    const defaultStatus = new Status(StatusEnum.OutOfStock);

    return new Stock(defaultStockId, defaultQuantityAvailable, defaultStatus);
  }

  delete() {
    if (this.status.value !== StatusEnum.OutOfStock) {
      throw new Error('在庫がある場合削除できません。');
    }
  }

  private changeStatus(newStatus: Status) {
    this._status = newStatus;
  }

  // 在庫数を増やす
  increaseQuantity(amount: number) {
    if (amount < 0) {
      throw new Error('増加量は0以上でなければなりません。');
    }

    const newQuantity = this.quantityAvailable.increment(amount).value;

    // 在庫数が10以下ならステータスを残りわずかにする
    if (newQuantity <= 10) {
      this.changeStatus(new Status(StatusEnum.LowStock));
    }
    this._quantityAvailable = new QuantityAvailable(newQuantity);
  }

  // 在庫数を減らす
  decreaseQuantity(amount: number) {
    if (amount < 0) {
      throw new Error('減少量は0以上でなければなりません。');
    }

    const newQuantity = this.quantityAvailable.decrement(amount).value;
    if (newQuantity < 0) {
      throw new Error('減少後の在庫数が0未満になってしまいます。');
    }

    // 在庫数が10以下ならステータスを残りわずかにする
    if (newQuantity <= 10) {
      this.changeStatus(new Status(StatusEnum.LowStock));
    }

    // 在庫数が0になったらステータスを在庫切れにする
    if (newQuantity === 0) {
      this.changeStatus(new Status(StatusEnum.OutOfStock));
    }

    this._quantityAvailable = new QuantityAvailable(newQuantity);
  }

  // エンティティの再構築
  static reconstruct(
    stockId: StockId,
    quantityAvailable: QuantityAvailable,
    status: Status
  ) {
    return new Stock(stockId, quantityAvailable, status);
  }

  get stockId(): StockId {
    return this._stockId;
  }

  get quantityAvailable(): QuantityAvailable {
    return this._quantityAvailable;
  }

  get status(): Status {
    return this._status;
  }
}

```

`create` メソッドでは、デフォルトの値を設定しています。このデフォルトの値は、例えば在庫数は 0、ステータスは在庫切れというように、ビジネスルールによって決まった値です。このように、ビジネスルールとの整合性を保つことができます。

```js
 static create() {
    const defaultStockId = new StockId(); // 自動ID採番
    const defaultQuantityAvailable = 0;
    const defaultStatus = new Status(StatusEnum.PreSale);

    return new Stock(
      defaultStockId,
      new QuantityAvailable(defaultQuantityAvailable),
      defaultStatus
    );
  }
```

`delete` メソッドでは、「ステータスが在庫切れの在庫は削除できない」というビジネスルールを適用しています。このように、エンティティの状態を変更するメソッドの中で、ビジネスルールを適用することで、エンティティのライフサイクルにビジネスルールを適用することができます。

```js
  delete() {
    if (this.status.value !== StatusEnum.OutOfStock) {
      throw new Error('在庫がある場合削除できません。');
    }
  }
```

`changeQuantityAvailable`メソッドは、`increaseQuantity`、`decreaseQuantity`メソッドに変更されました。エンティティのメソッドはドメインの振る舞いを反映したものであるべきです。この変更でより直感的に在庫数を増減できるようになりました。そして「在庫数が 0 になったらステータスを在庫切れに変更する」というビジネスルールや、在庫数の整合性のルールを適用しています。

```js
  // 在庫数を増やす
  increaseQuantity(amount: number) {
    if (amount < 0) {
      throw new Error('増加量は0以上でなければなりません。');
    }

    const newQuantity = this.quantityAvailable.increment(amount).value;

    // 在庫数が10以下ならステータスを残りわずかにする
    if (newQuantity <= 10) {
      this.changeStatus(new Status(StatusEnum.LowStock));
    }
    this._quantityAvailable = new QuantityAvailable(newQuantity);
  }

  // 在庫数を減らす
  decreaseQuantity(amount: number) {
    if (amount < 0) {
      throw new Error('減少量は0以上でなければなりません。');
    }

    const newQuantity = this.quantityAvailable.decrement(amount).value;
    if (newQuantity < 0) {
      throw new Error('減少後の在庫数が0未満になってしまいます。');
    }

    // 在庫数が10以下ならステータスを残りわずかにする
    if (newQuantity <= 10) {
      this.changeStatus(new Status(StatusEnum.LowStock));
    }

    // 在庫数が0になったらステータスを在庫切れにする
    if (newQuantity === 0) {
      this.changeStatus(new Status(StatusEnum.OutOfStock));
    }

    this._quantityAvailable = new QuantityAvailable(newQuantity);
  }
```

:::message
エンティティが持つ属性は private にして、必ずメソッドを通して変更するようにしましょう。属性を変更するメソッドにビジネスルールを適用することで、ビジネスルールの整合性が崩れるのを防ぐことができます。

:::

これらの実装により、エンティティにビジネスルールを適用することができました。

# エンティティのテスト

値オブジェクト同様、ビジネスルールが正しく実装されているかを保証するためにはテストは必須です。
それではテストを書いていきましょう。「Stock.test.ts」を作成し、書いていきます。

```js:StockManagement/Domain/models/Book/Stock/Stock.test.ts
import { Stock } from './Stock';
import { QuantityAvailable } from './QuantityAvailable/QuantityAvailable';
import { Status, StatusEnum } from './Status/Status';
import { StockId } from './StockId/StockId';

// nanoid() をモックする
jest.mock('nanoid', () => ({
  nanoid: () => 'testIdWithExactLength',
}));

describe('Stock', () => {
  const stockId = new StockId('abc');
  const quantityAvailable = new QuantityAvailable(100);
  const status = new Status(StatusEnum.InStock);

  describe('create', () => {
    it('デフォルト値で在庫を作成する', () => {
      const stock = Stock.create();

      expect(
        stock.stockId.equals(new StockId('testIdWithExactLength'))
      ).toBeTruthy();
      expect(
        stock.quantityAvailable.equals(new QuantityAvailable(0))
      ).toBeTruthy();
      expect(
        stock.status.equals(new Status(StatusEnum.OutOfStock))
      ).toBeTruthy();
    });
  });

  describe('delete', () => {
    it('在庫ありの場合はエラーを投げる', () => {
      const stock = Stock.reconstruct(stockId, quantityAvailable, status);

      expect(() => stock.delete()).toThrow('在庫がある場合削除できません。');
    });

    it('在庫なしなしの場合はエラーを投げない', () => {
      const notOnSaleStatus = new Status(StatusEnum.OutOfStock);
      const stock = Stock.reconstruct(
        stockId,
        quantityAvailable,
        notOnSaleStatus
      );

      expect(() => stock.delete()).not.toThrow();
    });
  });

  describe('increaseQuantity', () => {
    it('在庫数を増やす', () => {
      const stock = Stock.reconstruct(stockId, quantityAvailable, status);
      stock.increaseQuantity(5);

      expect(
        stock.quantityAvailable.equals(new QuantityAvailable(105))
      ).toBeTruthy();
    });

    it('増加量が負の数の場合はエラーを投げる', () => {
      const stock = Stock.reconstruct(stockId, quantityAvailable, status);

      expect(() => stock.increaseQuantity(-1)).toThrow(
        '増加量は0以上でなければなりません。'
      );
    });
  });

  describe('decreaseQuantity', () => {
    it('在庫数を減らす', () => {
      const stock = Stock.reconstruct(stockId, quantityAvailable, status);
      stock.decreaseQuantity(5);

      expect(
        stock.quantityAvailable.equals(new QuantityAvailable(95))
      ).toBeTruthy();
    });

    it('減少量が負の数の場合はエラーを投げる', () => {
      const stock = Stock.reconstruct(stockId, quantityAvailable, status);

      expect(() => stock.decreaseQuantity(-1)).toThrow(
        '減少量は0以上でなければなりません。'
      );
    });

    it('減少後の在庫数が0未満になる場合はエラーを投げる', () => {
      const stock = Stock.reconstruct(stockId, quantityAvailable, status);

      expect(() => stock.decreaseQuantity(101)).toThrow();
    });

    it('在庫数が0になったらステータスを在庫切れにする', () => {
      const stock = Stock.reconstruct(stockId, quantityAvailable, status);
      stock.decreaseQuantity(100);

      expect(
        stock.quantityAvailable.equals(new QuantityAvailable(0))
      ).toBeTruthy();
      expect(
        stock.status.equals(new Status(StatusEnum.OutOfStock))
      ).toBeTruthy();
    });

    it('在庫数が10以下になったらステータスを残りわずかにする', () => {
      const stock = Stock.reconstruct(stockId, quantityAvailable, status);
      stock.decreaseQuantity(90);

      expect(
        stock.quantityAvailable.equals(new QuantityAvailable(10))
      ).toBeTruthy();
      expect(stock.status.equals(new Status(StatusEnum.LowStock))).toBeTruthy();
    });
  });
});

```

ビジネスロジック、振る舞い(メソッド)、例外処理などを網羅するようにテストを書きます。
なるべくテストのカバレッジが 100%に近づけるようにしましょう。

jest コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest Stock.test.ts
```

お疲れ様でした、以上で`Stock`エンティティの実装は完了です。ビジネスルールをクラス内にカプセル化し、`Stock`エンティティ自身がドキュメントの役割を果たすようになりました。さらに、ライフサイクルにおける整合性が保てるようになりました。

# まとめ

- エンティティを利用することで、ドメインの振る舞いの整合性を担保できる
- エンティティ自身がドキュメントになる

本章では、値オブジェクトとエンティティの違いとエンティティの実装方法について学びました。
次章は、DDD において非常に重要で難しい概念である**集約**の説明を行い、**集約ルート**である**Book**ルートエンティティを実装していきます。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/entity
