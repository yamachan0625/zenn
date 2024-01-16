---
title: 'ドメインイベントの活用'
---

# ドメインイベントとは

ドメインイベント (Domain Event) とは、ドメインで「過去に発生した出来事」のことです。この本ではすでにイベントストーミングを通して、ドメインイベントについて触れていました。イベントストーミングでは、ドメインイベントを洗いだし、ドメインイベントを発生させるプロセスを特定し、システムの動作の全体像を明確に理解することができました。以下の図でオレンジ色の図形に記載されている内容がドメインイベントになります。

@[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=843-1791&t=hvtQJVFBGW90SwuN-0)

ドメインイベントはしばしば連鎖的な影響を及ぼします。たとえば、「書籍の在庫が切れた」というイベントが「書籍の発注イベント」や「在庫切れメール送信イベント」を引き起こします。それらのイベントがさらに新しいイベントを発生させる可能性もあります。このように、一つのイベントが次のイベントを誘発するプロセスを「**イベント駆動**」と呼びます。そして、イベント駆動を利用してシステムを設計する方法が「**イベント駆動型アーキテクチャ**」です。

イベント駆動型アーキテクチャでドメインイベントを適切に活用することにより、**イベントストーミングマップ内のプロセスをそのままシステム設計に反映させることができます**。ドメインイベントを、コードで具体的なイベントクラスやメソッドとして表現し、これらのイベントに基づいてシステムの各部分が相互に通信し合うことで、実現します。

本章では、イベント駆動型アーキテクチャについて説明し、ドメインイベントの具体的な実装と、ドメインイベントを活用したシステム設計の方法を紹介します。

## イベント駆動型アーキテクチャ (EDA)

イベント駆動型アーキテクチャは、システムの各コンポーネントが、状態の変更や更新を示すイベントをパブリッシュ、処理するアーキテクチャです。イベント駆動型アーキテクチャは、従来のリクエスト駆動型と異なり、システムの各コンポーネント間の疎結合を実現し、個にスケーリングさせることが容易になります。

イベント駆動型アーキテクチャのメリットとデメリットは以下の通りです。

|                  | 説明                                                             |
| ---------------- | ---------------------------------------------------------------- |
| **メリット**     |                                                                  |
| 疎結合           | コンポーネント間の依存関係が少なく、変更に強い。                 |
| スケーラビリティ | 各コンポーネントを独立してスケールアップ・ダウンできる。         |
| 柔軟性           | 新しいイベントやサービスを容易に統合できる。                     |
| 非同期処理       | システムの全体的なパフォーマンスを向上させる。                   |
| **デメリット**   |                                                                  |
| 複雑性           | イベントの管理や追跡が難しい場合がある。                         |
| デバッグの難しさ | イベントベースのシステムはデバッグが困難になることがある。       |
| データの整合性   | イベントが非同期であるため、データは結果整合性となる場合がある。 |

イベント駆動型アーキテクチャを利用する場合は、これらのメリットとデメリットを理解した上で、利用する必要があります。

<!-- イベント駆動型アーキテクチャでイベントを送信するモデルには、`Pub/Sub`モデル、`イベント・ストリーム`モデルなどがあります。ここでは`Pub/Sub`モデルを利用してドメインイベントを送信する方法を紹介します。 -->

次に、イベント駆動型アーキテクチャにおいて、イベントを送受信するために `Pub/Sub` モデルを利用する方法について説明します。さらに、`Pub/Sub` モデルの `push` 型と `pull` 型のメカニズムについても説明します。

## Pub/Sub モデル

`Pub/Sub` (パブリッシュ/サブスクライブ) とは、アプリケーションのコンポーネント間でメッセージを非同期にやり取りする方法です。この方式では、メッセージを送る側 (パブリッシャー) が特定のトピックにメッセージを送信し、そのトピックを購読している側 (サブスクライバー) がメッセージを受け取ります。これにより、各サービスは他のサービスから独立して互いの動作に影響を与えずに情報を交換できます。
主なメッセージングブローカーには、`RabbitMQ`、`Apache Kafka`、`Amazon SNS`、`Azure Service Bus`、`Google Cloud Pub/Sub`などがあります。

### Push 型と Pull 型

`Pub/Sub` モデル内でのイベント送信には `push` 型と `pull` 型の二つのアプローチがあります。

- **Push 型**
  この方式では、メッセージが生成されると、パブリッシャー がそのメッセージをサブスクライバーにプッシュします。サブスクライバーは待機するだけで、新しいメッセージが到着したら即座に通知されます。リアルタイム性が求められる場合や、サブスクライバーがオフラインになる可能性が低い場合に有効です。
- **Pull 型**
  この方式では、サブスクライバーがアクティブにメッセージングシステムに問い合わせ (ポーリング) し、新しいメッセージがあるかどうかを確認します。サブスクライバーがオフラインの場合は、メッセージは保持され、サブスクライバーがオンラインになったときに配信されます。リアルタイム性が求められない場合や、サブスクライバーがオフラインになる可能性がある場合に有効です。

Pull 型のアプローチは、とくに分散システムやマイクロサービスアーキテクチャにおいて効果的です。サービス間の結合度を低く保ちつつ、柔軟でスケーラブルなイベント処理が実現できます。

## Pub/Sub モデルの主要なコンポーネント

`Pub/Sub` モデルの主要なコンポーネントは以下の 4 つです。

- メッセージ
- トピック
- サブスクライバー
- パブリッシャー

それぞれの紹介とともに、`Pub/Sub` モデルをドメインイベントにどのように適用するか確認していきましょう。

### メッセージ

メッセージは、送信者 (プロデューサー) から受信者 (コンシューマー) に送信される通信データです。利用可能なメッセージのデータ型はさまざまですが、ドメインイベントでは JSON 形式のデータを利用します。JSON には、イベントの ID、イベントの名前、発生した日時、発生時の集約の状態、およびその他の関連するデータなどが含まれます。

### トピック

すべてのメッセージは、特定のトピックに関連付けられています。トピックはメッセージのフィードを表す名前付きリソースで、送信者 (プロデューサー) と受信者 (コンシューマー) の間でメッセージをやり取りするための**中間のチャネル**のような役割を果たします。トピックには名前をつけます。そして、メッセージを送信するときに、トピック名を指定し、そのトピックをサブスクライブしているすべてのサブスクライバーにメッセージを送信します。

ここでは、イベントごとにトピックを分割するパターンを適用します。トピック名にはドメインイベントの名前を利用します。たとえば「書籍の在庫が切れた」というドメインイベントの場合は`BookStockDepleted`のような名前にします。

:::message
トピックの設計は、システムのニーズと制約 (利用するメッセージブローカーによっても異なる) に基づいて検討する必要があります。たとえば、イベントの種類ごとに処理の優先度やリソースの要件が異なる場合は、トピックをイベントごとに分割するパターンが有効です。一方で、イベント同士が密接に関連している場合や、管理の簡素化が優先される場合は、一つのトピックでの管理が適切な選択となるかもしれません。他にも、リソースの制限、アクセス制御なども考慮する必要があります。
:::

### サブスクライバー

サブスクライバーはメッセージの受信者 (コンシューマー) です。サブスクライバーは、関心のあるトピックをサブスクライブします。サブスクライバーは、受信したメッセージを利用して、関連する処理を実行します。たとえば、「書籍の在庫が切れた」というドメインイベントのメッセージを受け取ったサブスクライバーは、書籍の発注を行う処理を実行します。

### パブリッシャー

パブリッシャーは、メッセージを送信するコンポーネントです。特定のドメインイベントが発生したとき、パブリッシャーはこのイベントに関連する情報を含むメッセージを作成します。たとえば「書籍の在庫が切れた」というドメインイベントの場合、パブリッシャーはこのメッセージを特定のトピック (`BookStockDepleted`) に公開し、トピックをサブスクライブしているすべてのサブスクライバーにイベントを通知します。

パブリッシャーの役割は、ドメインイベントがシステム内の他のコンポーネントに伝わることを保証することです。このプロセスにより、イベント駆動での構築が可能となり、各コンポーネントが疎結合でありながら連携して動作することができます。

@[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=3371-854&t=qIagwKy9SOBpSlIE-0)

# ドメインイベントの実装

それでは`Pub/Sub`モデルの`pull`型を利用したドメインイベントの基盤を実装していきましょう。ここでは実装をシンプルにするためにメッセージブローカーの代わりとして`Node.js`の`EventEmitter`を利用します。メッセージブローカーとメカニズムは異なりますが、同様の実装でドメインイベントのパブリッシュ/サブスクライブを実現できます。

ドメインイベントをパブリッシュする側は `EventEmitter` オブジェクトの `emit` メソッドを使用してドメインイベントパブリッシュし、関連するデータを渡すことができます。一方でドメインイベントをサブスクライブする側は、`once`メソッドを利用して特定のドメインイベント名に対してコールバック関数を登録することで、ドメインイベントを受け取り、処理を実行することができます。
このように`EventEmitter`は`Pub/Sub`モデルのパブリッシャーとサブスクライバーの役割を担います。

:::message
`EventEmitter` は、メッセージの配信を保証しません。つまり、メッセージが失われた場合やサブスクライバーがドメインイベントを正常に処理したかどうかを追跡しません。また、`Node.js`の同一プロセスでのみ動作するため、サービスを跨いだメッセージの配信ができません。そのため、本番環境でドメインイベントを利用する場合は、メッセージブローカーを利用することをオススメします。
:::

## DomainEvent クラス

まずはドメインで発生したことに関連する情報を保持する`DomainEvent`クラスを作成します。このクラスは`Pub/Sub`モデルのメッセージとして利用されます。
`src/Domain/shared`配下に`DomainEvent`ディレクトリを作成します。次に`DomainEvent.ts`ファイルを作成し以下のように実装します。

```ts:src/Domain/shared/DomainEvent/DomainEvent.ts
import { nanoid } from 'nanoid';

export class DomainEvent<
  T extends Record<string, unknown> = Record<string, unknown>
> {
  private constructor(
    public readonly eventId: string,
    // イベントの内容
    public readonly eventBody: T,
    // イベントの名前
    public readonly eventName: string,
    // イベントの発生時刻
    public readonly occurredOn: Date
  ) {}

  static create<T extends Record<string, unknown> = Record<string, unknown>>(
    eventBody: T,
    eventName: string
  ): DomainEvent<T> {
    return new DomainEvent(nanoid(), eventBody, eventName, new Date());
  }

  static reconstruct<T extends Record<string, unknown>>(
    eventId: string,
    eventBody: T,
    eventName: string,
    occurredOn: Date
  ): DomainEvent<T> {
    return new DomainEvent(eventId, eventBody, eventName, occurredOn);
  }
}
```

`eventId`はメッセージの一意の ID でメッセージの重複を防ぐために利用することができます。`eventBody`には集約の状態を保持するオブジェクトを渡します。`eventName`にはドメインイベントの名前を渡します。この値を`Pub/Sub`のトピックとして利用します。`occurredOn`にはドメインイベントが発生した日時を渡します。
`create`メソッドは新しいドメインイベントを生成するために利用され、`reconstruct`メソッドは、メッセージブローカーを利用した場合に、受け取ったメッセージをドメインイベントに変換するために利用されます。 (`EventEmitter`を利用する場合は使用しません。)
ドメインイベントはドメインで「過去に発生した出来事」であるため、不変である必要があります。そのため`readonly`修飾子を利用して、内容を変更できないようにしています。

## ドメインイベントを一時的に記録する

ドメインイベントは**ドメイン**で「過去に発生した出来事」であるため、**ドメインオブジェクト** (集約) の操作によって発生するというのが自然でしょう。

たとえば、「書籍の作成イベント」は Book 集約の`create`メソッドによって発生するため、`create`メソッド内でドメインイベントの作成を行います。このタイミングでは、ドメインイベントの作成のみ行い、パブリッシュまでは行いません。なぜなら、このタイミングでドメインイベントのパブリッシュまで行ってしまうと、集約がドメインイベントをパブリッシュするためのコードを持つことになり、集約の責務が増えてしまいます。また、集約を操作し、ドメインイベントがパブリッシュされた後にリポジトリを利用した集約の永続化に失敗する可能性があります。その場合、ドメインイベントがパブリッシュされたにもかかわらず、集約の状態が永続化されていないという状況が発生してしまいます。

そのため、ドメインイベントをパブリッシュするためのコードは集約から分離する必要があります。集約ではドメインイベントの作成後、一度記録しておき、集約の永続化が成功した後にパブリッシュするようにします。そのためにドメインイベント記録用のクラスを作成します。`src/Domain/shared/DomainEvent`配下に`DomainEventStorable.ts`ファイルを作成し以下のように実装します。

```ts:src/Domain/shared/DomainEvent/DomainEventStorable.ts
import { DomainEvent } from './DomainEvent';

export abstract class DomainEventStorable {
  private domainEvents: DomainEvent[] = [];

  protected addDomainEvent(domainEvent: DomainEvent) {
    this.domainEvents.push(domainEvent);
  }

  getDomainEvents(): DomainEvent[] {
    return this.domainEvents;
  }

  clearDomainEvents() {
    this.domainEvents = [];
  }
}
```

`DomainEventStorable`クラスは、ドメインイベントを記録するためのクラスです。`addDomainEvent`メソッドでドメインイベントを記録し、`getDomainEvents`メソッドで記録されたドメインイベントを取得します。`clearDomainEvents`メソッドは、ドメインイベントをパブリッシュした後に呼び出し、記録されたドメインイベントを削除するために利用します。

このクラスはすべての集約で継承し、集約内でドメインイベントを記録できるようにします。`Book.ts`で`DomainEventStorable`クラスを継承するように変更しましょう。

```diff ts:src/Domain/models/Book/Book.ts
+ import { DomainEventStorable } from 'Domain/shared/DomainEvent/DomainEventStorable';
  ...
  // DomainEventStorableを継承する
- export class Book {
+ export class Book extends DomainEventStorable {
    private constructor(
      private readonly _bookId: BookId,
      private _title: Title,
      private _price: Price,
      private readonly _stock: Stock
    ) {
+     super();
    }
    (省略)
}
```

## 集約の操作でドメインイベントを生成する

ドメインイベントの生成には**ファクトリ**を利用すると見通しが良くなります。まずは、ファクトリの作成を行いましょう。`src/Domain/shared/DomainEvent/`配下に`Book`ディレクトリを作成します。次に`BookDomainEventFactory.ts`ファイルを作成し以下のように実装します。

```ts:src/Domain/shared/DomainEvent/Book/BookDomainEventFactory.ts
import { Book } from 'Domain/models/Book/Book';
import { DomainEvent } from '../DomainEvent';
import { StatusLabel } from 'Domain/models/Book/Stock/Status/Status';

export type BookDomainEventBody = {
  BookId: string;
  title: string;
  price: number;
  quantityAvailable: number;
  status: StatusLabel;
};

export const BOOK_EVENT_NAME = {
  CREATED: 'StockManagement.BookCreated',
  DEPLETED: 'StockManagement.BookDepleted',
  DELETED: 'StockManagement.BookDeleted',
} as const;

export class BookDomainEventFactory {
  constructor(private book: Book) {}

  public createEvent(
    eventName: (typeof BOOK_EVENT_NAME)[keyof typeof BOOK_EVENT_NAME]
  ) {
    return DomainEvent.create(this.entityToEventBody(), eventName);
  }

  private entityToEventBody(): BookDomainEventBody {
    return {
      BookId: this.book.bookId.value,
      title: this.book.title.value,
      price: this.book.price.value.amount,
      quantityAvailable: this.book.quantityAvailable.value,
      status: this.book.status.toLabel(),
    };
  }
}
```

`BookDomainEventFactory`は Book 集約のドメインイベントを生成するファクトリです。一例として`Created`、`Depleted`、`Deleted`の 3 つのドメインイベント名を用意しました。集約では`createEvent`メソッドを利用してドメインイベントを生成します。`DomainEvent`の`eventBody`には集約を直接渡すのではなく、`entityToEventBody`メソッドを利用して、ドメインイベントで必要な集約の状態のみを保持するオブジェクトを生成して渡します。これはドメインイベントを不変に扱うために重要です。

それでは Book 集約で`BookDomainEventFactory`を利用してドメインイベントを生成するように実装していきましょう。`Book.ts`ファイルを以下のように修正します。

```diff ts:src/Domain/models/Book/Book.ts
  import { DomainEventStorable } from 'Domain/shared/DomainEvent/DomainEventStorable';
  import { BookId } from './BookId/BookId';
  import { Price } from './Price/Price';
+ import { Status, StatusEnum } from './Stock/Status/Status';
  import { Stock } from './Stock/Stock';
  import { Title } from './Title/Title';
+ import {
+   BOOK_EVENT_NAME,
+   BookDomainEventFactory,
+ } from 'Domain/shared/DomainEvent/Book/BookDomainEventFactory';


  export class Book extends DomainEventStorable {
    (省略)
    static create(bookId: BookId, title: Title, price: Price) {
-     return new Book(bookId, title, price, Stock.create());
+     const book = new Book(bookId, title, price, Stock.create());
+     book.addDomainEvent(
+       new BookDomainEventFactory(book).createEvent(BOOK_EVENT_NAME.CREATED)
+     );

+     return book;
    }

    delete() {
      // stockが削除可能か確認する
      this._stock.delete();
      // Bookを削除する処理があればここに書く

+     this.addDomainEvent(
+       new BookDomainEventFactory(this).createEvent(BOOK_EVENT_NAME.DELETED)
+     );
    }

    decreaseStock(amount: number) {
      this._stock.decreaseQuantity(amount);

+      // 在庫切れになったらイベントを生成する
+      if (this.status.equals(new Status(StatusEnum.OutOfStock))) {
+        this.addDomainEvent(
+          new BookDomainEventFactory(this).createEvent(BOOK_EVENT_NAME.DEPLETED)
+        );
+      }
    }
}
```

`BookDomainEventFactory`でドメインイベントを生成し、継承した`DomainEventStorable`クラスの`addDomainEvent`メソッドを利用してドメインイベントを記録します。集約のテストでは、`getDomainEvents`メソッドを利用して記録されたドメインイベントを取得し、期待するドメインイベントが記録されているかどうかを検証します。

```ts
describe('create', () => {
  it('デフォルト値で在庫を作成し、ドメインイベントが生成される', () => {
    const book = Book.create(bookId, title, price);
    // 期待するドメインイベントが記録されているかどうかを検証する
    expect(book.getDomainEvents()[0].eventName).toBe(BOOK_EVENT_NAME.CREATED);
  });
});
```

記録したドメインイベントをどのように利用するかは後ほど説明します。

## EventEmitter を利用したパブリッシャー/サブスクライバーの実装

メッセージブローカーはデータベースや外部サービスと同様に、アプリケーションのビジネスロジックとは独立した技術的なサービスのためインフラストラクチャとして扱います。インフラ層で扱うということは、`DIP`を適用する必要があります。そのため、まずはインターフェイスの定義から行いましょう。

まずぱパブリッシャーです。`src/Domain/shared/DomainEvent/`配下に`IDomainEventPublisher.ts`ファイルを作成し以下のように実装します。

```ts:src/Domain/shared/DomainEvent/IDomainEventPublisher.ts
import { DomainEvent } from 'Domain/shared/DomainEvent/DomainEvent';

export interface IDomainEventPublisher {
  publish(domainEvent: DomainEvent): void;
}
```

`publish`メソッドは`DomainEvent`を受け取り、受け取った`DomainEvent`を利用して実際にドメインイベントのパブリッシュを行います。

次はサブスクライバーです。`src/Application/shared/`配下に`DomainEvent`ディレクトリを作成します。次に`IDomainEventSubscriber.ts`ファイルを作成し以下のように実装します。

```ts:src/Application/shared/DomainEvent/IDomainEventSubscriber.ts
import { DomainEvent } from 'Domain/shared/DomainEvent/DomainEvent';

export interface IDomainEventSubscriber {
  subscribe<T extends Record<string, unknown>>(
    eventName: string,
    callback: (event: DomainEvent<T>) => void
  ): void;
}
```

`subscribe`メソッドは、イベント名とコールバック関数を受け取り、イベント名に対応するドメインイベントがパブリッシュされたときに、パブリッシュされた`DomainEvent`を利用してコールバック関数を実行します。

では、これらのインターフェイスに対して`EventEmitter`を利用した具体的なパブリッシャーとサブスクライバーを実装しましょう。`EventEmitter`の`once`で登録したイベントをサブスクライブするには同一の`EventEmitter`インスタンスの`emit`によってパブリッシュする必要があります。そのため`EventEmitter`をシングルトンで扱えるように、`EventEmitter`をラップするクラスを作成します。`src/Infrastructure/`配下に`DomainEvent/EventEmitter/`ディレクトリを作成します。次に、`EventEmitterClient.ts`ファイルを作成し以下のように実装します。

```ts:src/Infrastructure/DomainEvent/EventEmitter/EventEmitterClient.ts
import { EventEmitter } from 'events';
import { singleton } from 'tsyringe';

@singleton()
class EventEmitterClient {
  public eventEmitter: EventEmitter;

  constructor() {
    this.eventEmitter = new EventEmitter();
  }
}

export default EventEmitterClient;
```

`tsyringe`の`@singleton`デコレータを利用して、`EventEmitter`をシングルトンで扱えるようにします。`EventEmitterClient`を`container.resolve`で取得し、`EventEmitter`を利用します。

では、パブリッシャーを実装します。`src/Infrastructure/DomainEvent/EventEmitter`配下に`EventEmitterDomainEventPublisher.ts`ファイルを作成し以下のように実装します。

```ts:src/Infrastructure/DomainEvent/EventEmitter/EventEmitterDomainEventPublisher.ts
import { IDomainEventPublisher } from 'Domain/shared/DomainEvent/IDomainEventPublisher';
import EventEmitterClient from './EventEmitterClient';
import { DomainEvent } from 'Domain/shared/DomainEvent/DomainEvent';
import { container } from 'tsyringe';

export class EventEmitterDomainEventPublisher implements IDomainEventPublisher {
  publish(domainEvent: DomainEvent) {
    container
      .resolve(EventEmitterClient)
      .eventEmitter.emit(domainEvent.eventName, domainEvent);
  }
}
```

`EventEmitterDomainEventPublisher`では、`IDomainEventPublisher`の`publish`メソッドを実装します。`publish`メソッドでは、`EventEmitterClient`を利用して`DomainEvent`をパブリッシュします。

次にサブスクライバーを実装します。`src/Infrastructure/DomainEvent/EventEmitter`配下に`EventEmitterDomainEventSubscriber.ts`ファイルを作成し以下のように実装します。

```ts:src/Infrastructure/DomainEvent/EventEmitter/EventEmitterDomainEventSubscriber.ts
import EventEmitterClient from './EventEmitterClient';
import { DomainEvent } from 'Domain/shared/DomainEvent/DomainEvent';
import { IDomainEventSubscriber } from 'Application/shared/DomainEvent/IDomainEventSubscriber';
import { container } from 'tsyringe';

export class EventEmitterDomainEventSubscriber
  implements IDomainEventSubscriber
{
  subscribe<T extends Record<string, unknown>>(
    eventName: string,
    callback: (event: DomainEvent<T>) => void
  ) {
    container
      .resolve(EventEmitterClient)
      .eventEmitter.once(eventName, callback);
  }
}
```

`EventEmitterDomainEventSubscriber`では、`IDomainEventSubscriber`の`subscribe`メソッドを実装します。`subscribe`メソッドでは、`EventEmitterClient`を利用して`DomainEvent`をサブスクライブします。実際にはこのサブスクライバーを利用して特定のドメインイベントをサブスクライブして何かしらの処理を行うような実装になります。後ほど説明します。

次に、作成した`EventEmitterDomainEventPublisher`と`EventEmitterDomainEventSubscriber`を DI コンテナで利用できるように登録を行います。`Program.ts`に以下のように追加しましょう。

```diff ts:src/Program.ts
  import { container, Lifecycle } from 'tsyringe';

  import { PrismaBookRepository } from 'Infrastructure/Prisma/Book/PrismaBookRepository';
  import { PrismaClientManager } from 'Infrastructure/Prisma/PrismaClientManager';
  import { PrismaTransactionManager } from 'Infrastructure/Prisma/PrismaTransactionManager';
+ import { EventEmitterDomainEventPublisher } from 'Infrastructure/DomainEvent/EventEmitter/EventEmitterDomainEventPublisher';
+ import { EventEmitterDomainEventSubscriber } from 'Infrastructure/DomainEvent/EventEmitter/EventEmitterDomainEventSubscriber';

(省略)

+ // DomainEvent
+ container.register('IDomainEventPublisher', {
+   useClass: EventEmitterDomainEventPublisher,
+ });
+ container.register('IDomainEventSubscriber', {
+   useClass: EventEmitterDomainEventSubscriber,
+ });
```

これで`EventEmitterDomainEventPublisher`と`EventEmitterDomainEventSubscriber`を DI コンテナで利用できるようになりました。

## ドメインイベントをパブリッシュする

それでは実際にドメインイベントをパブリッシュする処理を追加しましょう。集約で一時的に記録されたドメインイベントをリポジトリを利用して永続化した後に取り出し、パブリッシュするように実装します。

リポジトリでパブリッシャーを利用するために、リポジトリの各メソッドでパブリッシャーを受け取るようにします。あえてリポジトリでパブリッシャーを受け取るようにすることで、リポジトリでドメインイベントをパブリッシュしなければいけないということを明示的にします。

では、まずはリポジトリのインターフェイスを修正します。`IBookRepository.ts`を以下のように修正します。

```diff ts:src/Domain/models/Book/IBookRepository.ts
  import { Book } from './Book';
  import { BookId } from './BookId/BookId';
+ import { IDomainEventPublisher } from '../../shared/DomainEvent/IDomainEventPublisher';

  export interface IBookRepository {
    save(
      book: Book,
+     domainEventPublisher: IDomainEventPublisher
    ): Promise<void>;
    update(
      book: Book,
+     domainEventPublisher: IDomainEventPublisher
    ): Promise<void>;
    delete(
      book: Book,
+     domainEventPublisher: IDomainEventPublisher
    ): Promise<void>;
    find(bookId: BookId): Promise<Book | null>;
}
```

次に`PrismaBookRepository`でパブリッシャーを受け取るように修正します。そして永続化が成功した後にパブリッシャーを利用してドメインイベントをパブリッシュします。`PrismaBookRepository.ts`を以下のように修正します。

```diff ts:src/Infrastructure/Prisma/Book/PrismaBookRepository.ts
  ---
+ import { IDomainEventPublisher } from 'Domain/shared/DomainEvent/IDomainEventPublisher';

  @injectable()
  export class PrismaBookRepository implements IBookRepository {
    (省略)

-   async save(book: Book) {
+   async save(book: Book, domainEventPublisher: IDomainEventPublisher) {
      const client = this.clientManager.getClient();

      await client.book.create({
        data: {
          bookId: book.bookId.value,
          title: book.title.value,
          priceAmount: book.price.value.amount,
          stock: {
            create: {
              stockId: book.stockId.value,
              quantityAvailable: book.quantityAvailable.value,
              status: this.statusDataMapper(book.status.value),
            },
          },
        },
      });

+     // ここでイベントをパブリッシュする
+     book.getDomainEvents().forEach((event) => {
+       domainEventPublisher.publish(event);
+     });
+     book.clearDomainEvents();
    }

  (省略)
  }
```

`Prisma`による集約の永続化が成功した後に、`getDomainEvents`で一時的に記録されたドメインイベントを取り出し、`domainEventPublisher.publish(event)`でドメインイベントをパブリッシュしています。その後`clearDomainEvents`を呼び出し、一時的に記録されたドメインイベントをクリアしています。
`update`、`delete`メソッドも同様に修正しましょう。リポジトリのテストでは`IDomainEventPublisher`のモックし、`publish`メソッドが呼び出されているかどうかを検証しましょう。

次にアプリケーションサービスで`domainEventPublisher`を DI し、リポジトリに渡すように修正します。今回は例として`RegisterBookApplicationService.ts`を以下のように修正します。

```diff ts:src/Application/Book/RegisterBookApplicationService/RegisterBookApplicationService.ts
---
+ import { IDomainEventPublisher } from 'Domain/shared/DomainEvent/IDomainEventPublisher';

  export type RegisterBookCommand = {
    isbn: string;
    title: string;
    priceAmount: number;
  };

  @injectable()
  export class RegisterBookApplicationService {
    constructor(
      @inject('IBookRepository')
      private bookRepository: IBookRepository,
      @inject('ITransactionManager')
      private transactionManager: ITransactionManager,
+     @inject('IDomainEventPublisher')
+     private domainEventPublisher: IDomainEventPublisher
    ) {}

    async execute(command: RegisterBookCommand): Promise<void> {
      await this.transactionManager.begin(async () => {
        const isDuplicateISBN = await new ISBNDuplicationCheckDomainService(
          this.bookRepository
        ).execute(new BookId(command.isbn));

        if (isDuplicateISBN) {
          throw new Error('既に存在する書籍です');
        }

        // イベントを生成し、集約に記録する
        const book = Book.create(
          new BookId(command.isbn),
          new Title(command.title),
          new Price({ amount: command.priceAmount, currency: 'JPY' })
        );

-       await this.bookRepository.save(book);
+       await this.bookRepository.save(book, this.domainEventPublisher);
      });
    }
  }
```

そのほかのアプリケーションサービスも同様に修正しましょう。

## サブスクライバーを登録する

ドメインイベントのパブリッシュができるようになったので、次はドメインイベントをサブスクライブする処理を実装します。ここでは、パブリッシュされたドメインイベントを受け取って、受け取ったドメインイベントを`console.log`で出力するような処理を実装してみましょう。`src/Application/shared/DomainEvent/`配下に`subscribers`ディレクトリを作成します。次に、`BookLogSubscriber.ts`ファイルを作成し以下のように実装します。

```ts:src/Application/shared/DomainEvent/subscribers/BookLogSubscriber.ts
import { inject, injectable } from 'tsyringe';
import { IDomainEventSubscriber } from '../IDomainEventSubscriber';
import {
  BOOK_EVENT_NAME,
  BookDomainEventBody,
} from 'Domain/shared/DomainEvent/Book/BookDomainEventFactory';

@injectable()
export class BookLogSubscriber {
  constructor(
    @inject('IDomainEventSubscriber')
    private subscriber: IDomainEventSubscriber
  ) {
    this.subscriber.subscribe<BookDomainEventBody>(
      BOOK_EVENT_NAME.CREATED,
      (event) => {
        console.log(event);
      }
    );
  }
}
```

`BookLogSubscriber`では、`IDomainEventSubscriber`を利用して`BOOK_EVENT_NAME.CREATED`に対応するドメインイベントがパブリッシュされたときに、コールバック関数の引数で受け取ったドメインイベントを`console.log`で出力するように実装しています。

次に`BookLogSubscriber`がドメインイベントをサブスクライブできるようにドメインイベントがパブリッシュされる前に実行してあげる必要があります。`Express.js`で作成した API のサーバが起動するタイミング (`app.listen`) で実行してみましょう。`src/Presentation/Express/index.ts`を以下のように修正します。

```diff ts:src/Presentation/Express/index.ts
  ---
+ import { BookLogSubscriber } from 'Application/shared/DomainEvent/subscribers/BookLogSubscriber';

  const app = express();
  const port = 3000;

  app.listen(port, () => {
    console.log(`Example app listening on port ${port}`);

+   // サブスクライバーを登録する
+   container.resolve(BookLogSubscriber);
  });
```

以上でドメインイベントのパブリッシュとサブスクライブの実装と設定は完了です。

## 動作確認

それでは、書籍登録 API を叩いてサブスクライブしたドメインイベントが実際に発火するか、動作確認を行いましょう。

まずは、サーバの起動を行います。

```bash:StockManagement/
$ npx ts-node src/Presentation/Express/index.ts
```

次に`curl`コマンドを利用してリクエストを送信します。

```bash
$ curl -X POST -H "Content-Type: application/json" -d '{"isbn":"9784167158057","title":"吾輩は猫である","priceAmount":770}' http://localhost:3000/book
```

サーバのログを確認すると、以下のようにパブリッシュされた`DomainEvent`がサブスクライバーによって受け取られて、ログが吐きだされていることが確認できます。今回はログを出力するだけですが、実際にはドメインイベントを利用して何かしらの処理を行うことができます。

```bash
$ npx ts-node src/Presentation/Express/index.ts
Example app listening on port 3000
DomainEvent {
  eventId: 'NQx01hsBv1ZPDMMpn3oUK',
  eventBody: {
    BookId: '9784167158057',
    title: '吾輩は猫である',
    price: 770,
    quantityAvailable: 0,
    status: '在庫切れ'
  },
  eventName: 'StockManagement.BookCreated',
  occurredOn: 2024-01-01T12:54:03.656Z
}
```

以上で動作確認は完了です。ドメインイベントに関するテストは、これまでの内容を生かして実装してみてください。

最後に全体的な処理の流れを確認しておきましょう。
![](https://storage.googleapis.com/zenn-user-upload/f7b0c889eac4-20240106.png)

:::message
ドメインイベントを扱う際、実際のシステム設計はより複雑になります。とくに、ドメインイベントの確実なパブリッシングや失敗時の対応、処理の整合性の維持、メッセージの順序付け、重複メッセージの処理など、さまざまな側面を考慮する必要があります。以下に、ドメインイベントをより安全に扱うためのいくつかのパターンを紹介します。

- **OutBox パターン**
  集約がデータベースに永続化された際にドメインイベントを確実にパブリッシュするために、OutBox パターンが使用されます。このパターンでは、ドメインイベントを一時的に「OutBox」と呼ばれる中間ストレージに保存し、そこから非同期でドメインイベントを取得しパブリッシュします。これにより、ドメインイベントのパブリッシュが失敗した場合でも、OutBox 内のイベントを再試行し、確実にイベントを処理できるようになります。

- **Saga パターン**
  ドメインイベントが連鎖的に処理される場合、とくに一連の処理の中で一部が失敗した際に整合性を保つことが重要です。Saga パターンは、長期間にわたるトランザクションや複数のステップを含むビジネスプロセスを管理するために使用されます。このパターンでは、各ステップが成功するか、あるいは失敗した場合に適切な補償トランザクション (巻き戻し操作）を実行することで、全体としての整合性を保ちます。

これらのパターンは、ドメインイベントがシステム全体で一貫した方法で処理され、データの整合性が保たれることを保証するために重要です。メッセージの順序付け、重複メッセージに対しては、メッセージブローカー側の機能で対応することが可能な場合があります。

これらのアプローチを適切に組み合わせることで、システムの信頼性が向上します。
:::

# ドメインイベントの活用

ドメインイベントの基本的な実装が完了しました。次に、ドメインイベントの具体的な活用方法について確認しましょう。ここでは、活用方法を 3 つ紹介します。

- 集約間の結果整合性を保つ
- 非同期処理
- イベントソーシング

それではそれぞれ確認していきましょう。

## 集約間の結果整合性を保つ

イベントストーミングマップの一部を引用します。
@[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=4042-1099&t=6WUO3TJnvErmBTAL-0)
「書籍の在庫が切れた」というドメインイベントが発生した場合に、「書籍の発注書を作成する」というコマンドの実行が必要であることが分かります。このプロセスには、「書籍」と「発注」という二つの集約が関与しており、書籍集約が在庫切れ状態になった場合、発注集約を作成するというような整合性が求められます。これらの異なる集約間で整合性を保つためには、結果整合性を用いる必要があります。詳しくは`chapter10 集約`の「集約の外部では結果整合性を用いる」を確認してください。

このようなケースでドメインイベントを活用することができます。具体的には、書籍集約で「書籍の在庫が切れた」というドメインイベントをパブリッシュし、「書籍の在庫が切れた」というドメインイベントをサブスクライブして、取得したドメインイベントのデータを元に発注集約を作成するという処理を実装します。

:::message
シンプルにするために、アプリケーションサービスで複数の集約を扱って整合性を保つことも可能です。しかし、集約間の結合度が上がり保守性、拡張性が下がるためオススメはしません。
:::

## 非同期処理

再度イベントストーミングマップの一部を引用します。
@[figma](https://www.figma.com/file/g04nAogGCGgM62IKXHUSLT/Online-bookstore?type=whiteboard&node-id=4190-2032&t=6WUO3TJnvErmBTAL-0)

「書籍の在庫が切れた」というドメインイベントに対して、同時に二つの処理が必要であることが分かります。

1. 書籍の発注書を作成する
2. 在庫切れメールを送信

現時点で必要な処理は二つですが将来的に、通知も行いたいという要望や異なるコンテキストとの連携などの処理が増える可能性を考えると、これらを同期的に実行することは、システムのパフォーマンスと可用性に悪影響を及ぼす可能性があります。処理の増加はレスポンス時間の遅延を引き起こし、ユーザー体験を損なうだけでなく、一部の処理に問題が生じた場合、他の処理にも影響を及ぼすリスクがあります。

ということで、非同期で処理できないか考えてみましょう。たとえば、「在庫切れメールを送信」の処理は、ユーザーに対して即時のフィードバックが必要なわけではありません。多少の遅延も許されるでしょう。このようなケースでは、主要なレスポンスフローから分離し、非同期で実行することで、メール送信の遅延や失敗がシステム全体のパフォーマンスに影響を及ぼすことを防ぐことができます。

このような非同期処理を行いたい場合にもドメインイベントを活用することができます。具体的には、発注集約を作成する時と同様に、パブリッシュされたドメインイベントをサブスクライブして、取得したドメインイベントを元にメール送信処理を実行するという処理を実装します。さらに、将来的に処理を増やしたい場合、サブスクライバーが自身のコンテキストで関心のあるドメインイベントをサブスクライブするだけで、処理を簡単に追加することができます。この時パブリッシャー側には、何の変更も加える必要がないということがポイントです。さらに、パブリッシャー側はサブスクライバーによってどのように処理されるかは気にする必要がありません。このようにドメインイベントを利用することでスケーラブルに非同期処理を実装することができます。

## イベントソーシング

イベントソーシングとは、アプリケーションの状態の変化をイベント (「書籍が購入された」「書籍が編集された」「注文が削除された」など) として記録し、これらのイベントを元にアプリケーションの状態を再構築するアプローチです。各イベントは変更の履歴としてイベントストアに永続化され、アプリケーションの状態はこれらのイベントを再生することで任意の時点の状態を再現できます。ドメインイベントをイベントソーシングのイベントとして活用して、イベントソーシングを実装することができます。

では、具体的にイベントソーシングを利用すると何が嬉しいのでしょうか？アプリケーションは、ビジネスの成長に伴って、新しい要件や変更に対応しなければならなくなります。たとえば、在庫管理を最適化するために、過去の在庫変更の履歴を分析して、在庫の変化を予測する必要があるかもしれません。他にも、顧客の購入履歴を分析して、顧客の嗜好を理解し、より良い販売戦略を立てる必要があるかもしれません。しかし、一般的なデータベースのアプローチ (ステートソーシング) では、在庫や購入履歴のデータを上書きすることで現在の状態を表現します。このようなアプローチでは、過去にどのような変更が行われたかを追跡することができず、ビジネス上の要件を満たすことができません。

このような要件を満たすためには、アプリケーションの状態を変更するだけでは不十分で、変更の履歴を追跡する必要があります。イベントソーシングでは、アプリケーションの状態変化をイベントとして記録するため、任意のタイミングの履歴を追跡することができます。このように、イベントソーシングは、アプリケーションの状態を変更するだけでなく、変更の履歴を追跡することができるため、アプリケーションの状態を変更するだけでは不十分な要件を満たすことができます。また、詳細は割愛しますが、CQRS (コマンドクエリ責務分離) などのパターンと組み合わせることで、より柔軟なアーキテクチャを実現することができます。

イベントソーシングやイベントストア、CQRS については以下の記事が参考になります。
https://aws.amazon.com/jp/blogs/news/build-a-cqrs-event-store-with-amazon-dynamodb/

# まとめ

- イベント駆動型アーキテクチャを利用することで、システムの柔軟性、拡張性、可用性、パフォーマンスが向上する
- ドメインイベントを利用することで、イベントストーミングマップの内容を反映した実装を行うことができる

本章では、イベント駆動型アーキテクチャを実現するドメインイベントの基本的な実装と活用方法について確認しました。ドメインイベントを利用することで、イベント駆動型アーキテクチャのメリットを享受することができます。ぜひ、ドメインイベントを活用して、柔軟なシステムを実現してください。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/domain-event

# 参考文献

https://learn.microsoft.com/ja-jp/dotnet/architecture/microservices/microservice-ddd-cqrs-patterns/domain-events-design-implementation
https://kakehashi-dev.hatenablog.com/entry/2023/12/24/091000
https://techblog.zozo.com/entry/implementation-of-cqrs-using-outbox-and-cdc-with-dynamodb
https://aws.amazon.com/jp/what-is/pub-sub-messaging/
