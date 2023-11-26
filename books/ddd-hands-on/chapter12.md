---
title: 'リポジトリ'
---

# リポジトリとは

リポジトリは、**集約の永続化を抽象化**する役割を果たします。リポジトリの主な目的は、ドメインモデルやビジネスロジックをデータ永続化の詳細 (どのデータベースを利用するか、ORM を利用するかなど) から切り離すことです。これにより、ドメインモデルがデータベースのスキーマやデータアクセスなどの特定の技術に依存することなく、ドメイン知識の表現に集中できるようになります。

リポジトリの責務は大きく二つに分かれます。一つは**集約の永続化**で、これにはエンティティの新規作成や既存のエンティティの更新が含まれます。もう一つは**集約の復元**で、これはデータベースからエンティティを取得し、ドメインオブジェクトとしてアプリケーションサービスに渡すプロセスを指します。リポジトリはこれらのプロセスを通して、ドメインモデルとデータベースの間の橋渡しを行います。

# リポジトリの実装

それでは、リポジトリの実装を見ていきましょう。ドメインサービスの章で作成した`ISBNDuplicationCheckDomainService`にリポジトリを適用していきます。リポジトリをより理解するために、最初にリポジトリを利用しせずに実装し、問題点を確認します。その後、リポジトリを利用した実装に修正していきメリットを確認していきます。

:::message
ここで重要なのは、Prisma や PostgreSQL の具体的な実装の内容ではなく、**リポジトリの役割を理解すること**です。リポジトリはデータベースの永続化を抽象化することが目的です。そのためデータの永続化には、システムの要件に合わせて RDB でも NoSQL でも何を利用しても構いません。ここではあくまで一例として、Prisma や PostgreSQL といった技術を採用しています。読者のお気に入りのデータベース、ORM に置き換えて読み進めることも可能です。
:::

## 環境のセットアップ

まずはリポジトリの実装に必要な環境のセットアップを行います。

### PostgreSQL のセットアップ

PostgreSQL は Docker を利用してセットアップします。
まずはルートディレクトリに`docker-compose.yml`を作成し、以下の内容を記述します。

```yaml:StockManagement/docker-compose.yml
version: '3.8'
services:
  localdb:
    image: postgres:14.1-alpine
    restart: always
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=localdb
    ports:
      - '5432:5432'
    volumes:
      - localdb:/var/lib/postgresql/data
volumes:
  localdb:
    driver: local

```

### Prisma のセットアップ

`Prisma` の詳しい説明はは省略します。詳しくは[公式ドキュメント](https://www.prisma.io/docs)を参照ください。

まず、必要なパッケージをインストールします。

```bash:StockManagement/
$ npm install prisma --save-dev
$ npm install @prisma/client
```

次に、`Prisma CLI` の `init` コマンドを使用して `Prisma` をセットアップします。データベースには`postgresql`を指定します。このコマンドを実行すると`prisma/schema.prisma`と`.env`が作成されます。

```bash:StockManagement/
$ npx prisma init --datasource-provider postgresql
```

次に、`.env`の`DATABASE_URL`を`docker-compose.yml`に定義した内容に合わせて変更します。

```bash:StockManagement/.env
DATABASE_URL="postgresql://postgres:password@localhost:5432/localdb"
```

次に、`prisma/schema.prisma`に `Book`と`Stock` のモデルを定義します。`Prisma schema` では、データベースのテーブルをモデルとして直感的に定義することができます。

```js:StockManagement/prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Book {
  bookId          String   @id
  title           String
  priceAmount     Float
  stock           Stock?
}

model Stock {
  stockId           String   @id
  quantityAvailable Int
  status            Status   @default(OUT_OF_STOCK)
  book              Book     @relation(fields: [bookId], references: [bookId],  onDelete: Cascade)
  bookId            String   @unique
}

enum Status {
  IN_STOCK
  LOW_STOCK
  OUT_OF_STOCK
}
```

### 動作確認

まず `docker compose` コマンドを実行して、`PostgreSQL` を起動します。

```bash:StockManagement/
$ docker compose up -d
```

次に `prisma migrate`を実行しマイグレーションを行います。

```bash:StockManagement/
$ npx prisma migrate dev --name init
```

エラーなくマイグレーションが完了したら、セットアップ完了になります。

## リポジトリを適用しないケース

セットアップが完了したので、`ISBNDuplicationCheckDomainService`で、データベースからデータを取得できるように修正していきましょう。

```js:.../ISBNDuplicationCheckDomainService.ts
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

// リポジトリを適用せずに実装
export class ISBNDuplicationCheckDomainService {
  async execute(isbn: BookId): Promise<boolean> {
    // データベースに問い合わせて重複があるか確認する
    const duplicateISBNBook = await prisma.book.findUnique({
      where: {
        bookId: isbn.value,
      },
    });

    const isisDuplicateISBN = duplicateISBNBook !== null;

    return isisDuplicateISBN;
  }
}
```

上記コードでは、直接ドメインサービス内で ORM を利用しデータベースへのアクセスを記述しています。これはオニオンアーキテクチャやドメイン駆動設計の考え方に反しています。DB へのアクセスはインフラストラクチャ層の責務であり、ドメイン層がインフラストラクチャ層に依存することは許容されていません。

では、アーキテクチャに違反することでどのような問題が発生するのでしょうか？具体的には以下のような問題が発生します。

- テストの複雑化
- 変更容易性の低下

それぞれ確認していきましょう。

### テストの複雑化

実際にこの状態でテストを書いてみることで、テストの複雑性を確認していきます。

```js
import { ISBNDuplicationCheckDomainService } from './ISBNDuplicationCheckDomainService';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { PrismaClient } from '@prisma/client';

// PrismaClientをmock
jest.mock('@prisma/client', () => {
  return {
    PrismaClient: jest.fn().mockImplementation(() => ({
      book: {
        findUnique: jest.fn(),
      },
    })),
  };
});

describe('ISBNDuplicationCheckDomainService', () => {
  test('ISBNが重複していない場合、falseを返す', async () => {
    const prisma = new PrismaClient();
    const service = new ISBNDuplicationCheckDomainService();

    prisma.book.findUnique.mockResolvedValue(null);
    const isbn = new BookId('9784167158057');
    expect(await service.execute(isbn)).toBeFalsy();
  });

  // (省略)
});
```

このテストでは、`PrismaClient`をモックする必要があるため、テストのセットアップが複雑になります。モックの設定には多くの注意と詳細な知識が必要で、時間と労力を消費します。たとえばこのテストでは、利用する`PrismaClient`のすべてのメソッドをテストごとに正しくモックする必要があります。これではドメインの知識 (重複のチェック) をテストすることに集中できず、テストの信頼性が低下します。

また、テストが特定の ORM の振る舞いに依存しているため、ORM の変更やバージョンアップによりテストが影響を受ける可能性があります。もし ORM が変更された場合、`PrismaClient`をモックしている箇所をすべて見つけ出し、新い ORM をモックするように修正する必要があります。これは、テストの保守性を低下させます。

### 変更容易性の低下

ドメイン層が特定の ORM やデータベース技術に依存すると、将来的な技術変更が困難になります。たとえば、ORM を Prisma から TypeORM に変更したい場合、ドメイン層のコードも大きく変更する必要が出てきます。ビジネスロジックを崩さずに変更するのは、根気がいる作業です。これは、システムの柔軟性を損ね、新しい技術への移行を困難にします。

また、ドメイン層がデータベースアクセスロジックを含むことで、ビジネスロジックとインフラストラクチャの関心が混在します。これにより、ビジネスロジックの見通しが悪くなり、保守性が低下します。ビジネスルールの変更とデータベースアクセスロジックの変更が相互に影響し合うため、変更の影響範囲が広がります。

また、ドメイン層が特定のインフラストラクチャに依存している場合、リファクタリングや再利用が難しくなります。たとえば、新しいプロジェクトで同じドメインロジックを利用したいが、異なるデータベースを使っている場合、ドメインロジックを再利用するには大幅な変更が必要になります。

## リポジトリを適用するケース

リポジトリを適用しない場合の問題点を確認したので、リポジトリを適用した実装に修正し、どのように問題点を解決できるか確認していきます。

### リポジトリのインターフェイス

ドメインサービスでインフラストラクチャの概念であるリポジトリを直接利用する (依存する) ことはできません。そこで DIP (依存性逆転の原則) に従い、リポジトリの抽象、つまりインターフェイスを定義し、インターフェイスに依存させるようにする必要があります。

それでは、先にリポジトリのインターフェイスを作成しましょう。`/Domain/models/Book`ディレクトリ配下に、`IBookRepository.ts`というファイル作成し、リポジトリのインターフェイスを定義します。

```js:StockManagement/Domain/models/Book/IBookRepository.ts
import { Book } from './Book';
import { BookId } from './BookId/BookId';

export interface IBookRepository {
  save(book: Book): Promise<void>;
  update(book: Book): Promise<void>;
  delete(bookId: BookId): Promise<void>;
  find(bookId: BookId): Promise<Book | null>;
}

```

`ISBNDuplicationCheckDomainService`では`find`を利用します。`find`は`BookId`を受け取り、`Book`を返すメソッドです。`Book`が存在しない場合は`null`を返します。その他にも、`save`、`update`、`delete`といったメソッドを定義しています。これらのメソッドは、次章、`Book`の永続化を行うために利用します。

:::message
リポジトリは実際に利用するタイミング、もしくは利用することが決まっているメソッドのみを定義するようにしましょう。いつか使うだろうと思って何でも定義しておくと、インターフェイスが肥大化した挙句、実際に利用せずに放置されるといったことが起きてしまいます。
:::

### リポジトリのインターフェイスを利用してみる

それでは作成した`IBookRepository`を利用して、`ISBNDuplicationCheckDomainService`にリポジトリを適用していきましょう。

```js:.../ISBNDuplicationCheckDomainService.ts
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { IBookRepository } from 'Domain/models/Book/IBookRepository';

// リポジトリを適用するように変更
export class ISBNDuplicationCheckDomainService {
  constructor(private bookRepository: IBookRepository) {}

  async execute(isbn: BookId): Promise<boolean> {
    // データベースに問い合わせて重複があるか確認する
    const duplicateISBNBook = await this.bookRepository.find(isbn);
    const isisDuplicateISBN = !!duplicateISBNBook;

    return isisDuplicateISBN;
  }
}
```

リポジトリを利用することで、ドメインサービスはデータベースアクセスの詳細を知る必要がなくなり、**ドメインの知識に集中**できるようになりました。また、抽象型である、`IBookRepository`に依存するようにすることで、具体的なデータベースアクセスの実装がなくても、ドメインサービスを実装することができました。つまり、ドメイン層がその他レイヤーに依存することなく、独立して実装できるということです。

### ドメインサービスのテスト

それではリポジトリを利用したケースのテストを書いていきましょう。`ISBNDuplicationCheckDomainService`では、リポジトリは抽象型に依存しているため、テスト用の軽量なリポジトリを用いてテストすることができます。

まずは、インメモリを利用した軽量なリポジトリを実装しましょう。`src`ディレクトリ配下に、` Infrastructure/InMemory/Book`ディレクトリを作成し、`InMemoryBookRepository.ts `というファイルを作成し、以下のように実装します。

```js:StockManagement/src/Infrastructure/InMemory/Book/InMemoryBookRepository.ts
import { Book } from 'Domain/models/Book/Book';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { IBookRepository } from 'Domain/models/Book/IBookRepository';

export class InMemoryBookRepository implements IBookRepository {
  public DB: {
    [id: string]: Book;
  } = {};

  async save(book: Book) {
    this.DB[book.bookId.value] = book;
  }

  async update(book: Book) {
    this.DB[book.bookId.value] = book;
  }

  async delete(bookId: BookId) {
    delete this.DB[bookId.value];
  }

  async find(bookId: BookId): Promise<Book | null> {
    const book = Object.entries(this.DB).find(([id]) => {
      return bookId.value === id.toString();
    });

    return book ? book[1] : null;
  }
}

```

このリポジトリはインメモリ上に集約をそのまま保存し、取得することができるだけの実装です。
それでは、作成した`InMemoryBookRepository`を利用して、テストを書いていきましょう。`ISBNDuplicationCheckDomainService.ts`と同じ階層に`ISBNDuplicationCheckDomainService.test.ts`というファイルを作成し、以下のように実装します。

```js:.../ISBNDuplicationCheckDomainService.test.ts
import { ISBNDuplicationCheckDomainService } from './ISBNDuplicationCheckDomainService';
import { InMemoryBookRepository } from 'Infrastructure/InMemory/Book/InMemoryBookRepository';
import { BookId } from '../../../models/Book/BookId/BookId';
import { Book } from '../../../models/Book/Book';
import { Title } from '../../../models/Book/Title/Title';
import { Price } from '../../../models/Book/Price/Price';

describe('ISBNDuplicationCheckDomainService', () => {
  let isbnDuplicationCheckDomainService: ISBNDuplicationCheckDomainService;
  let inMemoryBookRepository: InMemoryBookRepository;

  beforeEach(() => {
    // テスト前に初期化する
    inMemoryBookRepository = new InMemoryBookRepository();
    isbnDuplicationCheckDomainService = new ISBNDuplicationCheckDomainService(
      inMemoryBookRepository
    );
  });

  test('重複がない場合、falseを返す', async () => {
    const isbn = new BookId('9784167158057');
    const result = await isbnDuplicationCheckDomainService.execute(isbn);
    expect(result).toBeFalsy();
  });

  test('重複がある場合、trueを返す', async () => {
    const isbn = new BookId('9784167158057');
    const title = new Title('吾輩は猫である');
    const price = new Price({
      amount: 770,
      currency: 'JPY',
    });
    const book = Book.create(isbn, title, price);

    await inMemoryBookRepository.save(book);

    const result = await isbnDuplicationCheckDomainService.execute(isbn);
    expect(result).toBeTruthy();
  });

  test('異なるISBNで重複がない場合、falseを返す', async () => {
    const existingIsbn = new BookId('9784167158057');
    const newIsbn = new BookId('9784167158064');
    const title = new Title('テスト書籍');
    const price = new Price({ amount: 500, currency: 'JPY' });
    const book = Book.create(existingIsbn, title, price);

    await inMemoryBookRepository.save(book);

    const result = await isbnDuplicationCheckDomainService.execute(newIsbn);
    expect(result).toBeFalsy();
  });
});

```

`InMemoryBookRepository`を利用してテストを書くことで、テストのために固有のデータベースの実行環境を用意する必要がなく、テストのセットアップが簡単になりました。また、テストの実行速度も向上します。

jest コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest ISBNDuplicationCheckDomainService.test.ts
```

エラーなく成功すれば OK です。

# Prisma を利用したリポジトリの実装

それでは実際の実行環境用に、`Prisma`を利用したリポジトリを実装していきましょう。さきほど作成した`Infrastructure`ディレクトリ配下に、`Prisma/Book`ディレクトリを作成し、`PrismaBookRepository.ts`というファイルを作成します。

```js:StockManagement/src/Infrastructure/Prisma/Book/PrismaBookRepository.ts
import { $Enums, PrismaClient } from '@prisma/client';
import { Book } from 'Domain/models/Book/Book';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { IBookRepository } from 'Domain/models/Book/IBookRepository';
import { Price } from 'Domain/models/Book/Price/Price';
import { QuantityAvailable } from 'Domain/models/Book/Stock/QuantityAvailable/QuantityAvailable';
import { Status, StatusEnum } from 'Domain/models/Book/Stock/Status/Status';
import { Stock } from 'Domain/models/Book/Stock/Stock';
import { StockId } from 'Domain/models/Book/Stock/StockId/StockId';
import { Title } from 'Domain/models/Book/Title/Title';

const prisma = new PrismaClient();

export class PrismaBookRepository implements IBookRepository {
  // DBのstatusの型とドメイン層のStatusの型が異なるので変換する
  private statusDataMapper(
    status: StatusEnum
  ): 'IN_STOCK' | 'LOW_STOCK' | 'OUT_OF_STOCK' {
    switch (status) {
      case StatusEnum.InStock:
        return 'IN_STOCK';
      case StatusEnum.LowStock:
        return 'LOW_STOCK';
      case StatusEnum.OutOfStock:
        return 'OUT_OF_STOCK';
    }
  }
  private statusEnumMapper(status: $Enums.Status): Status {
    switch (status) {
      case 'IN_STOCK':
        return new Status(StatusEnum.InStock);
      case 'LOW_STOCK':
        return new Status(StatusEnum.LowStock);
      case 'OUT_OF_STOCK':
        return new Status(StatusEnum.OutOfStock);
    }
  }

  async save(book: Book) {
    await prisma.book.create({
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
  }

  async update(book: Book) {
    await prisma.book.update({
      where: {
        bookId: book.bookId.value,
      },
      data: {
        title: book.title.value,
        priceAmount: book.price.value.amount,
        stock: {
          update: {
            quantityAvailable: book.quantityAvailable.value,
            status: this.statusDataMapper(book.status.value),
          },
        },
      },
    });
  }

  async delete(bookId: BookId) {
    await prisma.book.delete({
      where: {
        bookId: bookId.value,
      },
    });
  }

  async find(bookId: BookId): Promise<Book | null> {
    const data = await prisma.book.findUnique({
      where: {
        bookId: bookId.value,
      },
      include: {
        stock: true,
      },
    });

    if (!data || !data.stock) {
      return null;
    }

    return Book.reconstruct(
      new BookId(data.bookId),
      new Title(data.title),
      new Price({ amount: data.priceAmount, currency: 'JPY' }),
      Stock.reconstruct(
        new StockId(data.stock.stockId),
        new QuantityAvailable(data.stock.quantityAvailable),
        this.statusEnumMapper(data.stock.status)
      )
    );
  }
}
```

Prisma を利用し、`IBookRepository`のインターフェイスを元に、`save`、`update`、`delete`、`find`を実装しています。ここで重要なのは実装の詳細ではなく、インターフェイス`IBookRepository`の要件を満たす実装ができたことです。これにより、DB アクセスや ORM の詳細をドメイン層から隠蔽し、インフラストラクチャ層に閉じ込めることができました。

# リポジトリのテスト

リポジトリのテストには 2 種類あります。

- リポジトリを利用する側のコードが集約の入出力を正しく行えるか
- リポジトリ自体が正しく動くか

それぞれ確認してみましょう。

### リポジトリを利用する側のコードが集約の入出力を正しく行えるか

一つが`ISBNDuplicationCheckDomainService`のような、リポジトリを利用する側のコードが集約の入出力を正しく行えるか確認するテストです。これは、`ISBNDuplicationCheckDomainService.test.ts`のようにテスト用のリポジトリを利用してテストを行うことで確認できましたね。

### リポジトリ自体が正しく動くか

もう一つが、リポジトリ自体が正しく動くか確認するテストです。このテストでは実際にデータベースにアクセスし、データの永続化や復元が正しく行えるか確認します。

リポジトリを利用したテストを行う際に、多くの箇所でテストデータを作成する必要が出てきます。テスト用にテストデータ作成用のオブジェクトを作成しておくと便利です。`Infrastructure`ディレクトリ配下に、`shared/Book`ディレクトリを作成します。次に`bookTestDataCreator.ts`というファイルを作成し、以下のように実装します。

```js:StockManagement/src/Infrastructure/shared/Book/bookTestDataCreator.ts
import { Book } from 'Domain/models/Book/Book';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { IBookRepository } from 'Domain/models/Book/IBookRepository';
import { Price } from 'Domain/models/Book/Price/Price';
import { QuantityAvailable } from 'Domain/models/Book/Stock/QuantityAvailable/QuantityAvailable';
import { Status, StatusEnum } from 'Domain/models/Book/Stock/Status/Status';
import { Stock } from 'Domain/models/Book/Stock/Stock';
import { StockId } from 'Domain/models/Book/Stock/StockId/StockId';
import { Title } from 'Domain/models/Book/Title/Title';

export const bookTestDataCreator =
  (repository: IBookRepository) =>
  async ({
    bookId = '9784167158057',
    title = '吾輩は猫である',
    priceAmount = 770,
    stockId = 'test-stock-id',
    quantityAvailable = 0,
    status = StatusEnum.OutOfStock,
  }): Promise<Book> => {
    const entity = Book.reconstruct(
      new BookId(bookId),
      new Title(title),
      new Price({ amount: priceAmount, currency: 'JPY' }),
      Stock.reconstruct(
        new StockId(stockId),
        new QuantityAvailable(quantityAvailable),
        new Status(status)
      )
    );

    await repository.save(entity);

    return entity;
  };
```

`bookTestDataCreator`は汎用的に利用できます。対象のテストで利用したいリポジトリに切り替えることができますし、テストデータをデフォルト値として定義しているため、テストデータの作成が簡単になります。複数テストデータを作成したい場合は、デフォルト値を上書きすることで対応できます。
それでは、リポジトリ自体が正しく動くか確認するテストを書いていきましょう。`PrismaBookRepository.ts`と同じ階層に`PrismaBookRepository.test.ts`というファイルを作成し、以下のように実装します。

```js:StockManagement/src/Infrastructure/Prisma/Book/PrismaBookRepository.test.ts
import { PrismaClient } from '@prisma/client';
import { PrismaBookRepository } from './PrismaBookRepository';
import { bookTestDataCreator } from 'Infrastructure/shared/Book/bookTestDataCreator';
import { Book } from 'Domain/models/Book/Book';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { Title } from 'Domain/models/Book/Title/Title';
import { Price } from 'Domain/models/Book/Price/Price';
import { QuantityAvailable } from 'Domain/models/Book/Stock/QuantityAvailable/QuantityAvailable';
import { Status, StatusEnum } from 'Domain/models/Book/Stock/Status/Status';
import { Stock } from 'Domain/models/Book/Stock/Stock';

const prisma = new PrismaClient();

describe('PrismaBookRepository', () => {
  beforeEach(async () => {
    // テストごとにデータを初期化する
    await prisma.$transaction([prisma.book.deleteMany()]);
    await prisma.$disconnect();
  });

  const repository = new PrismaBookRepository();

  test('saveした集約がfindで取得できる', async () => {
    const bookId = new BookId('9784167158057');
    const title = new Title('吾輩は猫である');
    const price = new Price({
      amount: 770,
      currency: 'JPY',
    });
    const book = Book.create(bookId, title, price);
    await repository.save(book);

    const createdEntity = await repository.find(bookId);
    expect(createdEntity?.bookId.equals(bookId)).toBeTruthy();
    expect(createdEntity?.title.equals(title)).toBeTruthy();
    expect(createdEntity?.price.equals(price)).toBeTruthy();
    expect(createdEntity?.stockId.equals(book.stockId)).toBeTruthy();
    expect(
      createdEntity?.quantityAvailable.equals(book.quantityAvailable)
    ).toBeTruthy();
    expect(createdEntity?.status.equals(book.status)).toBeTruthy();
  });

  test('updateできる', async () => {
    const createdEntity = await bookTestDataCreator(repository)({});

    const stock = Stock.reconstruct(
      createdEntity.stockId,
      new QuantityAvailable(100),
      new Status(StatusEnum.InStock)
    );

    const book = Book.reconstruct(
      createdEntity.bookId,
      new Title('吾輩は猫である(改訂版))'),
      new Price({
        amount: 800,
        currency: 'JPY',
      }),
      stock
    );

    await repository.update(book);
    const updatedEntity = await repository.find(createdEntity.bookId);
    expect(updatedEntity?.bookId.equals(book.bookId)).toBeTruthy();
    expect(updatedEntity?.title.equals(book.title)).toBeTruthy();
    expect(updatedEntity?.price.equals(book.price)).toBeTruthy();
    expect(updatedEntity?.stockId.equals(book.stockId)).toBeTruthy();
    expect(
      updatedEntity?.quantityAvailable.equals(book.quantityAvailable)
    ).toBeTruthy();
    expect(updatedEntity?.status.equals(book.status)).toBeTruthy();
  });

  it('deleteできる', async () => {
    const createdEntity = await bookTestDataCreator(repository)({});

    const readEntity = await repository.find(createdEntity.bookId);
    expect(readEntity).not.toBeNull();

    await repository.delete(createdEntity.bookId);
    const deletedEntity = await repository.find(createdEntity.bookId);
    expect(deletedEntity).toBeNull();
  });
});

```

リポジトリに生えているメソッド (save, update, delete, find) が正しく動くか確認しています。テストデータの作成には、先ほど作成した`bookTestDataCreator`を利用しています。

:::message
リポジトリを利用したテストでは、開発環境のデータベースにアクセスします。そのため、テストを実行する前に、`docker-compose.yml`で定義した`localdb`を起動しておく必要があります。
また、`localdb`でリポジトリのテストを行うと、データベースのデータが初期化されてしまいます。本来であればテスト用にデータベースを用意することをオススメします。
:::

jest コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest PrismaBookRepository.test.ts
```

エラーなく成功すれば OK です。

# トランザクション管理

データベースを扱うリポジトリで忘れてはいけないのが、**トランザクション管理**です。トランザクション管理を適切に行わなければ、データの整合性が保つことができません。一般的に、ドメインモデルの永続化に関する**トランザクションを管理するのはアプリケーション層の責務**です。 (アプリケーションサービスの詳細は chapter13 アプリケーションサービスの章で説明します)
アプリケーション層は、ドメイン層の永続化を行う際に、トランザクションを開始し、コミットまたはロールバックする責務を持ちます。

トランザクションの管理は利用しているアプリケーションフレームワークや ORM によって異なりますが、実装のベースは以下のような形になります。

```js:sample
class ApplicationService {
  constructor(
    private repository: IDomainRepository,
  ) {}
  execute() {
    try {
      transaction.start();
      // ここでドメインモデルを操作し、データベースへアクセスする
      this.repository.save();
      transaction.commit();
    } catch (error) {
      transaction.rollback();
    }
  }
}
```

最初にトランザクションをスタートし、その後、ドメインモデルを操作し、データベースへコミットします。エラーが発生した場合は、トランザクションをロールバックします。この本では、`Prisma`を利用しているため、`Prisma`を利用したトランザクション管理について説明します。

## Prisma のトランザクション管理

Prisma では、`PrismaClient`の`$transaction`メソッドを利用することでトランザクションの管理を行うことができます。
それでは`ApplicationService`に適用してみましょう。

```js:sample
class ApplicationService {
  constructor(
    private repository: IDomainRepository,
  ) {}
  execute() {
   await prisma.$transaction(async (transaction) => {
      // ここでドメインモデルを操作し、データベースへアクセスする
      this.repository.save(transaction); // transaction用のクライアントを渡す必要がある
    });
  }
}
```

これで`Prisma`を利用したトランザクション管理ができるようになりました。しかし、この実装には問題があります。それは、アプリケーション層の`ApplicationService`がインフラストラクチャ層の`Prisma`に依存していることです。オニオンアーキテクチャでは、アプリケーション層はドメイン層に依存することができますが、インフラストラクチャ層に依存することはできません。このような依存がもたらす問題はさきほど確認しましたね。依存することで、アプリケーション層のテストが複雑になり、アプリケーション層の変更が困難になります。

それでは、アプリケーション層がインフラストラクチャ層に依存しないように、トランザクション管理の実装を行います。

### インターフェイスの定義

まずは、トランザクション管理用オブジェクトのインターフェイスを定義します。`src`ディレクトリに`Application/shared`ディレクトリを作成します。次に`ITransactionManager.ts`というファイルを作成し、以下のように実装します。

```js:StockManagement/src/Application/shared/ITransactionManager.ts
export interface ITransactionManager {
  begin<T>(callback: () => Promise<T>): Promise<T | undefined>;
}
```

このインターフェイスはアプリケーションサービスで DI して利用することになります。

```js:sample
import { ITransactionManager } from 'Application/shared/ITransactionManager';

class ApplicationService {
  constructor(
    private repository: IDomainRepository,
    private transactionManager: ITransactionManager
  ) {}
  execute() {
    await this.transactionManager.begin(async () => {
      // ここでドメインモデルを操作し、データベースへアクセスする
      this.repository.save()
    });
  }
}
```

これで、アプリケーションサービスから`Prisma`への依存を取り除くことができました。後はこのインターフェイスに合わせて、利用している`ORM`やデータベースを利用して実装を行うだけです。

### Prisma のトランザクション管理の実装

それでは、`ITransactionManager`に合わせて`Prisma`を利用したトランザクション管理の実装を行います。

ここからはドメイン駆動設計の本筋と外れた、技術的な実装の話になるため、詳細な説明は省略しコードと簡単な説明のみを記載します。ここで大事なのは、アプリケーション層からトランザクションの具体的な実装を隠蔽することです。`Prisma`の詳細に興味がない方はコピペして次に進みましょう。

:::details PrismaClient
PrismaClient はさまざまなところで利用するため、共通のインスタンスを利用するようにします。

```js:StockManagement/src/Infrastructure/Prisma/prismaClient.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();
export default prisma;
```

:::

:::details IDataAccessClientManager
ORM のデータアクセス用インスタンスを抽象化します。

```js:StockManagement/src/Infrastructure/shared/IDataAccessClientManager.ts
export interface IDataAccessClientManager<T> {
  setClient(client: T): void;
  getClient(): T;
}
```

:::

:::details PrismaClientManager
`IDataAccessClientManager`の`Prisma`を利用した実装です。

```js:StockManagement/src/Infrastructure/Prisma/PrismaClientManager.ts
import { Prisma, PrismaClient } from '@prisma/client';
import prisma from './prismaClient';
import { IDataAccessClientManager } from 'Infrastructure/shared/IDataAccessClientManager';

type Client = PrismaClient | Prisma.TransactionClient;
export class PrismaClientManager implements IDataAccessClientManager<Client> {
  private client: Client = prisma;

  setClient(client: Client): void {
    this.client = client;
  }

  getClient() {
    return this.client;
  }
}

```

:::

:::details PrismaTransactionManager
`ITransactionManager`の`Prisma`を利用した実装です。

```js:StockManagement/src/Infrastructure/Prisma/PrismaTransactionManager.ts
import { ITransactionManager } from 'Application/shared/ITransactionManager';
import prisma from './prismaClient';
import { PrismaClientManager } from './PrismaClientManager';

export class PrismaTransactionManager implements ITransactionManager {
  constructor(private clientManager: PrismaClientManager) {}

  async begin<T>(callback: () => Promise<T>): Promise<T | undefined> {
    return await prisma.$transaction(async (transaction) => {
      // トランザクション用のクライアントをセットする
      this.clientManager.setClient(transaction);

      try {
        return await callback();
      } catch (error) {
        // リトライ処理やロギング処理が必要であれば入れる
      } finally {
        this.clientManager.setClient(prisma);
      }
    });
  }
}


```

:::

:::details PrismaBookRepository
`ClientManager` を DI します。
リポジトリで利用する`Prisma Client`を`PrismaClientManager`から取得するように変更します。

```js:StockManagement/src/Infrastructure/Prisma/Book/PrismaBookRepository.ts
export class PrismaBookRepository implements IBookRepository {
  // ClientManagerをDIする
  constructor(private clientManager: PrismaClientManager) {}

  async save(book: Book) {
    // prismaクライアントをclientManagerから取得するように変更
    const client = this.clientManager.getClient();

    await client.book.create({
     (省略)
    });
  }
}

```

テストでは `PrismaBookRepository` に `clientManager` を渡すように変更します。

```js:StockManagement/src/Infrastructure/Prisma/Book/PrismaBookRepository.test.ts
describe('PrismaBookRepository', () => {
  (省略)
  const clientManager = new PrismaClientManager();
  // PrismaBookRepositoryにclientManagerを渡すように変更
  const repository = new PrismaBookRepository(clientManager);
  (省略)
});

```

:::

これらの実装によって、`Prisma`を利用したトランザクション管理の詳細を`PrismaTransactionManager`に隠蔽することができました。
簡単に処理の流れを説明します。

1. アプリケーションサービスで`PrismaTransactionManager`の`begin`メソッドが呼ばれる
2. `begin`メソッドの最初に`PrismaClientManager`の`setClient`メソッドが呼ばれ、`PrismaClientManager`にトランザクション用のクライアントがセットされる
3. トランザクションが開始される
4. リポジトリで`PrismaClientManager`の`getClient`で 2 でセットしたトランザクション用のクライアントが取得される
5. リポジトリでトランザクション用のクライアントを利用して、データベースへのアクセスが行われる
6. エラーがなければ、トランザクションがコミットされる

このアプローチにより、アプリケーション層は Prisma に依存せずにトランザクションを管理できます。また、トランザクションの管理が一箇所に集約されるため、エラーハンドリングやトランザクションのライフサイクル管理が容易になります。

# まとめ

- リポジトリは、集約の永続化を行うためのインターフェイス
- リポジトリを利用することで、ドメイン層はデータベースアクセスの詳細を知る必要がなくなり、ドメインの知識に集中できるようになる
- 抽象型である、`IBookRepository`に依存するようにすることで、具体的なデータベースアクセスの実装がなくても、ドメインサービスを実装することができる

本章では、リポジトリついて学び、`ISBNDuplicationCheckDomainService`を完成させテストまで行いました。また、トランザクション管理について学び、`Prisma`を利用したトランザクション管理の実装を行いました。
次章では、ユースケースの表現やドメインサービスのクライアントであるアプリケーションサービスについて学びます。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/repository
