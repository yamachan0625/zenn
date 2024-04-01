---
title: 'アプリケーションサービス (Application Service)'
---

# アプリケーションサービスとは

アプリケーションサービス (Application Service) とは、ドメインサービスに次ぐ 2 つ目のサービスで、**ユースケースを実現**するための操作を提供するサービスです。アプリケーションサービスは、**ドメイン層の「エンティティ」「値オブジェクト」「ドメインサービス」などのドメインオブジェクトを利用してユースケースを実現**します。

「ユースケース」とは、ユーザーがシステムを利用する際に実現したい機能や処理のことです。たとえば、在庫管理コンテキストでは、

- 書籍を登録する
- 書籍を取得する
- 在庫を追加する
- 書籍を削除する

などの、いわゆる**CRUD** 操作がユースケースにあたります。

では、それぞれのユースケースを表現するアプリケーションサービスを実装していきましょう。

# 書籍登録アプリケーションサービスの実装

それでは、「書籍を登録する」ユースケースを実現するアプリケーションサービスを実装していきましょう。`Application`ディレクトリ配下に`Book/RegisterBookApplicationService`ディレクトリを作成します。次に`RegisterBookApplicationService.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/Book/RegisterBookApplicationService/RegisterBookApplicationService.ts
import { ITransactionManager } from 'Application/shared/ITransactionManager';
import { Book } from 'Domain/models/Book/Book';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { IBookRepository } from 'Domain/models/Book/IBookRepository';
import { Price } from 'Domain/models/Book/Price/Price';
import { Title } from 'Domain/models/Book/Title/Title';
import { ISBNDuplicationCheckDomainService } from 'Domain/services/Book/ISBNDuplicationCheckDomainService/ISBNDuplicationCheckDomainService';

export type RegisterBookCommand = {
  isbn: string;
  title: string;
  priceAmount: number;
};

export class RegisterBookApplicationService {
  constructor(
    private bookRepository: IBookRepository,
    private transactionManager: ITransactionManager
  ) {}

  async execute(command: RegisterBookCommand): Promise<void> {
    await this.transactionManager.begin(async () => {
      const isDuplicateISBN = await new ISBNDuplicationCheckDomainService(
        this.bookRepository
      ).execute(new BookId(command.isbn));

      if (isDuplicateISBN) {
        throw new Error('既に存在する書籍です');
      }

      const book = Book.create(
        new BookId(command.isbn),
        new Title(command.title),
        new Price({ amount: command.priceAmount, currency: 'JPY' })
      );

      await this.bookRepository.save(book);
    });
  }
}
```

`RegisterBookApplicationService`では、まず`ISBNDuplicationCheckDomainService`を利用して、ISBN の重複チェックを行います。重複している場合は、`Error`をスローします。重複していない場合は、`Book`エンティティを生成し、`BookRepository`を利用して永続化します。

ここで重要なのは、ISBN の重複チェックのビジネスロジックや、`Book`エンティティ生成時のビジネスロジックがドメインオブジェクトに隠蔽されているということです。これにより、アプリケーションサービスの実装はドメイン知識を持たない状態で、ドメインオブジェクトを利用するだけでユースケースを実現することができます。

## 書籍登録アプリケーションサービスのテスト

`RegisterBookApplicationService`ではトランザクション管理オブジェクト`transactionManager`を DI しています。今回はトランザクションを考慮したテストを行わないため、テスト用にモックを作成しましょう。`Application/shared/`配下に`MockTransactionManager.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/shared/MockTransactionManager.ts
export class MockTransactionManager {
  async begin<T>(callback: () => Promise<T>) {
    return await callback();
  }
}
```

`MockTransactionManager`は、`begin`メソッドを実装していますが、引数に渡されたコールバック関数をそのまま実行しています。これにより、トランザクションを考慮したテストを行いたくないケースでは、`MockTransactionManager`を利用することでテストを行うことができます。

それでは、テストを実装していきましょう。`RegisterBookApplicationService.ts`と同じディレクトリ内に`RegisterBookApplicationService.test.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/Book/RegisterBookApplicationService/RegisterBookApplicationService.test.ts
import { InMemoryBookRepository } from 'Infrastructure/InMemory/Book/InMemoryBookRepository';
import {
  RegisterBookApplicationService,
  RegisterBookCommand,
} from './RegisterBookApplicationService';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { MockTransactionManager } from 'Application/shared/MockTransactionManager';
import { bookTestDataCreator } from 'Infrastructure/shared/Book/bookTestDataCreator';

describe('RegisterBookApplicationService', () => {
  it('重複書籍が存在しない場合書籍が正常に作成できる', async () => {
    const repository = new InMemoryBookRepository();
    const mockTransactionManager = new MockTransactionManager();
    const registerBookApplicationService = new RegisterBookApplicationService(
      repository,
      mockTransactionManager
    );

    const command: Required<RegisterBookCommand> = {
      isbn: '9784167158057',
      title: '吾輩は猫である',
      priceAmount: 770,
    };

    await registerBookApplicationService.execute(command);

    const createdBook = await repository.find(new BookId(command.isbn));
    expect(createdBook).not.toBeNull();
  });

  it('重複書籍が存在する場合エラーを投げる', async () => {
    const repository = new InMemoryBookRepository();
    const mockTransactionManager = new MockTransactionManager();
    const registerBookApplicationService = new RegisterBookApplicationService(
      repository,
      mockTransactionManager
    );

    // 重複させるデータを準備
    const bookID = '9784167158057';
    await bookTestDataCreator(repository)({
      bookId: bookID,
    });

    const command: Required<RegisterBookCommand> = {
      isbn: bookID,
      title: '吾輩は猫である',
      priceAmount: 770,
    };
    await expect(
      registerBookApplicationService.execute(command)
    ).rejects.toThrow();
  });
});
```

このテストでは、テスト用のリポジトリとトランザクション管理オブジェクト`MockTransactionManager`を利用し、`RegisterBookApplicationService`をインスタンス化し、実行して書籍が正常に作成できることを確認しています。また、重複書籍が存在する場合はエラーが投げられることを確認しています。

アプリケーションサービスのテストでは、値に関する検証を行っていません。その理由は、値に対する検証はドメインオブジェクトのテストですでに行っているからです。このアプローチにより、アプリケーションサービスのテストはユースケースに集中することができます。その結果テストは、よりシンプルで理解しやすくなります。

:::message
テストで利用するパラメータに `TypeScript` の `Utility Types` である `Required` を利用している点に注目してください。これは、`RegisterBookCommand`のプロパティをすべて必須にするために利用しています。

```js
const command: Required<RegisterBookCommand> = {
  isbn: '9784167158057',
  title: '吾輩は猫である',
  priceAmount: 770,
};
```

コマンド系アプリケーションサービスが実行時に受け取るパラメーターは、ビジネスルールの変更により任意のプロパティが追加される可能性があります。任意のプロパティの追加はテストファイルでコンパイルエラーが出ず、**悪い意味でテストに影響を与えることはありません**。

```js
// application service
export type RegisterBookCommand = {
  isbn: string,
  title: string,
  priceAmount: number,
  author?: string, // 任意のプロパティの追加
};

// test
// テストでは、実装側で任意のプロパティが追加されてもコンパイルエラーが出ない
const command: RegisterBookCommand = {
  isbn: '9784167158057',
  title: '吾輩は猫である',
  priceAmount: 770,
};
```

そこで、`RegisterBookCommand`のプロパティをすべて必須にするために `Required` を利用しています。これにより、新しいプロパティの追加やドメインモデルの変更が行われた際、既存のテストケースがコンパイルエラーを引き起こすことで、開発者はこれらの変更をテストケースに組み込む必要性に気づきます。これにより、新しいプロパティや変更されたビジネスロジックが適切にテストされることが保証され、テストが常に最新の実装を正確に反映するようになります。結果として、ソフトウェアの品質が継続的に向上します。

:::

`jest` コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest RegisterBookApplicationService.test.ts
```

エラーなく成功すれば OK です。

# 書籍取得アプリケーションサービスの実装

次は、「書籍を取得する」ユースケースを実現するアプリケーションサービスを実装していきましょう。`Application/Book/GetBookApplicationService`ディレクトリを作成ます。次に、`GetBookApplicationService.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/Book/GetBookApplicationService/GetBookApplicationService.ts
import { Book } from 'Domain/models/Book/Book';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { IBookRepository } from 'Domain/models/Book/IBookRepository';

export class GetBookApplicationService {
  constructor(private bookRepository: IBookRepository) {}

  async execute(isbn: string): Promise<Book | null> {
    const book = await this.bookRepository.find(new BookId(isbn));

    return book
  }
}
```

`GetBookApplicationService`では、`BookRepository`を利用して`Book`エンティティを取得し、`Book`エンティティをそのまま返却しています。

この実装には問題があります。アプリケーションサービスのクライアントはプレゼンテーション層です。現在の実装では Book 集約をそのまま`return`しています。これでは、アプリケーションサービスのクライアントであるプレゼンテーション層にドメインオブジェクトが漏れてしまい、プレゼンテーション層でドメインオブジェクトを操作できてしまいます。ドメインオブジェクトに依存するレイヤーが増えると、ドメインオブジェクトの変更により、影響範囲が広がり、変更容易性が低下します。

この問題に DTO (Data Transfer Object) を利用することで対応します。

## DTO (Data Transfer Object)

DTO とはデザインパターンの一つで、一般的には関連データをまとめて転送するためのデータ構造です。ドメイン駆動設計においてはドメインオブジェクトのデータのみをプレゼンテーション層に渡すためのデータ構造と定義することができます。

まずは DTO を作成していきましょう。`Application/Book/`配下に`BookDTO.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/Book/BookDTO.ts
import { Book } from 'Domain/models/Book/Book';
import { StatusLabel } from 'Domain/models/Book/Stock/Status/Status';

export class BookDTO {
  public readonly bookId: string;
  public readonly title: string;
  public readonly price: number;
  public readonly stockId: string;
  public readonly quantityAvailable: number;
  public readonly status: StatusLabel;

  constructor(book: Book) {
    this.bookId = book.bookId.value;
    this.title = book.title.value;
    this.price = book.price.value.amount;
    this.stockId = book.stockId.value;
    this.quantityAvailable = book.quantityAvailable.value;
    this.status = book.status.toLabel();
  }
}
```

実装自体は非常にシンプルで、`Book`エンティティを受け取り、プロパティに値をセットします。これはただのデータを保持するためのクラスです。しかし、このクラスには重要な役割があります。それは、ドメインオブジェクトをプレゼンテーション層に渡すためのデータ構造としての役割です。このクラスを利用することで、プレゼンテーション層はドメインオブジェクトを知ることなく、データのみを利用させるように制御することができます。`BookDTO`はデータ取得系のアプリケーションサービスで使い回すことができます。

`GetBookApplicationService`を修正し、`BookDTO`を返却するように実装しましょう。

```js:.../Application/Book/GetBookApplicationService/GetBookApplicationService.ts
  async execute(isbn: string): Promise<BookDTO | null> {
    const book = await this.bookRepository.find(new BookId(isbn));

    return book ? new BookDTO(book) : null;
  }
```

これで、`GetBookApplicationService`は`BookDTO`を返却するようになり、ドメインオブジェクトがアプリケーション層から漏れ出すことを防ぐことができます。

## 書籍取得アプリケーションサービスのテスト

それでは、テストを実装していきましょう。`GetBookApplicationService`と同じディレクトリ内に`GetBookApplicationService.test.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/Book/GetBookApplicationService/GetBookApplicationService.test.ts
import { InMemoryBookRepository } from 'Infrastructure/InMemory/Book/InMemoryBookRepository';
import { GetBookApplicationService } from './GetBookApplicationService';
import { bookTestDataCreator } from 'Infrastructure/shared/Book/bookTestDataCreator';
import { BookDTO } from '../BookDTO';

describe('GetBookApplicationService', () => {
  it('指定されたIDの書籍が存在する場合、DTOに詰め替えられ、取得できる', async () => {
    const repository = new InMemoryBookRepository();
    const getBookApplicationService = new GetBookApplicationService(repository);

    // テスト用データ作成
    const createdBook = await bookTestDataCreator(repository)({});

    const data = await getBookApplicationService.execute(
      createdBook.bookId.value
    );

    expect(data).toEqual(new BookDTO(createdBook));
  });

  it('指定されたIDの書籍が存在しない場合、nullが取得できる', async () => {
    const repository = new InMemoryBookRepository();
    const getBookApplicationService = new GetBookApplicationService(repository);

    const data = await getBookApplicationService.execute('9784167158057');

    expect(data).toBeNull();
  });
});
```

このテストでは、`GetBookApplicationService`を利用して、指定された ID の書籍が存在する場合は`BookDTO`に詰め替えられ、取得できることを確認しています。また、指定された ID の書籍が存在しない場合は`null`が取得できることを確認しています。

`jest` コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest GetBookApplicationService.test.ts
```

エラーなく成功すれば OK です。
在庫を増やすために必要なドメ

# 在庫追加アプリケーションサービスの実装

次は、「在庫を追加する」ユースケースを実現するアプリケーションサービスを実装していきましょう。`Application/Book/IncreaseBookStockApplicationService`ディレクトリを作成します。次に、`IncreaseBookStockApplicationService.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/Book/IncreaseBookStockApplicationService/IncreaseBookStockApplicationService.ts
import { ITransactionManager } from 'Application/shared/ITransactionManager';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { IBookRepository } from 'Domain/models/Book/IBookRepository';

export type IncreaseBookStockCommand = {
  bookId: string;
  incrementAmount: number;
};

export class IncreaseBookStockApplicationService {
  constructor(
    private bookRepository: IBookRepository,
    private transactionManager: ITransactionManager
  ) {}

  async execute(command: IncreaseBookStockCommand): Promise<void> {
    await this.transactionManager.begin(async () => {
      const book = await this.bookRepository.find(new BookId(command.bookId));

      if (!book) {
        throw new Error('書籍が存在しません');
      }

      book.increaseStock(command.incrementAmount);

      await this.bookRepository.update(book);
    });
  }
}
```

`IncreaseBookStockApplicationService`では、`BookRepository`を利用して`Book`エンティティを一度取得します。その後、`Book`エンティティの`increaseStock`メソッドを利用して在庫を増やします。最後に、`BookRepository`を利用して永続化します。

更新系のアプリケーションサービスで特徴的なのが、一度リポジトリから`Book`エンティティを取得し、その後に`increaseStock`メソッドを利用して在庫を増やしてる点です。これは、`Book`エンティティの`increaseStock`メソッドが、在庫を増やすために必要なビジネスロジックを持っているためです。このような実装にすることで、在庫の増減に関するドメイン知識がドメイン層に集約され、アプリケーションサービスの実装はドメイン知識を持たない状態で、ドメインオブジェクトを利用するだけでユースケースを**安全に**実現することができます。

## 在庫追加アプリケーションサービスのテスト

それでは、テストを実装していきましょう。`IncreaseBookStockApplicationService`と同じディレクトリ内に`IncreaseBookStockApplicationService.test.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/Book/IncreaseBookStockApplicationService/IncreaseBookStockApplicationService.test.ts
import { InMemoryBookRepository } from 'Infrastructure/InMemory/Book/InMemoryBookRepository';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { MockTransactionManager } from 'Application/shared/MockTransactionManager';
import { bookTestDataCreator } from 'Infrastructure/shared/Book/bookTestDataCreator';
import {
  IncreaseBookStockApplicationService,
  IncreaseBookStockCommand,
} from './IncreaseBookStockApplicationService';

describe('IncreaseBookStockApplicationService', () => {
  it('書籍の在庫を増加することができる', async () => {
    const repository = new InMemoryBookRepository();
    const mockTransactionManager = new MockTransactionManager();
    const increaseBookStockApplicationService =
      new IncreaseBookStockApplicationService(
        repository,
        mockTransactionManager
      );

    // テスト用データ準備
    const bookId = '9784167158057';
    await bookTestDataCreator(repository)({
      bookId,
      quantityAvailable: 0,
    });

    const incrementAmount = 100;
    const command: Required<IncreaseBookStockCommand> = {
      bookId,
      incrementAmount,
    };
    await increaseBookStockApplicationService.execute(command);

    const updatedBook = await repository.find(new BookId(bookId));
    expect(updatedBook?.quantityAvailable.value).toBe(incrementAmount);
  });

  it('書籍が存在しない場合エラーを投げる', async () => {
    const repository = new InMemoryBookRepository();
    const mockTransactionManager = new MockTransactionManager();
    const increaseBookStockApplicationService =
      new IncreaseBookStockApplicationService(
        repository,
        mockTransactionManager
      );

    const bookId = '9784167158057';
    const incrementAmount = 100;
    const command: Required<IncreaseBookStockCommand> = {
      bookId,
      incrementAmount,
    };
    await expect(
      increaseBookStockApplicationService.execute(command)
    ).rejects.toThrow();
  });
});
```

このテストでは、`IncreaseBookStockApplicationService`を利用して、書籍の在庫を増加することができることを確認しています。また、更新対象の書籍が存在しない場合はエラーを投げることを確認しています。

`jest` コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest IncreaseBookStockApplicationService.test.ts
```

エラーなく成功すれば OK です。

# 書籍削除アプリケーションサービスの実装

最後は、「書籍を削除する」ユースケースを実現するアプリケーションサービスを実装していきましょう。`Application/Book/DeleteBookApplicationService`ディレクトリを作成します。次に、`DeleteBookApplicationService.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/Book/DeleteBookApplicationService/DeleteBookApplicationService.ts
import { ITransactionManager } from 'Application/shared/ITransactionManager';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { IBookRepository } from 'Domain/models/Book/IBookRepository';

export type DeleteBookCommand = {
  bookId: string;
};
export class DeleteBookApplicationService {
  constructor(
    private bookRepository: IBookRepository,
    private transactionManager: ITransactionManager
  ) {}

  async execute(command: DeleteBookCommand): Promise<void> {
    await this.transactionManager.begin(async () => {
      const book = await this.bookRepository.find(new BookId(command.bookId));

      if (!book) {
        throw new Error('書籍が存在しません');
      }

      book.delete();

      await this.bookRepository.delete(book.bookId);
    });
  }
}
```

`DeleteBookApplicationService`では、`BookRepository`を利用して`Book`エンティティを一度取得します。その後、`Book`エンティティの`delete`メソッドを利用して書籍削除時の処理を行います。ここでは、書籍削除可能かどうかの判定を`Book`エンティティの`delete`メソッド内で行っています。最後に、`BookRepository`を利用して永続化します。

## 書籍削除アプリケーションサービスのテスト

それでは、テストを実装していきましょう。`DeleteBookApplicationService`と同じディレクトリ内に`DeleteBookApplicationService.test.ts`ファイルを作成し、以下のように実装します。

```js:.../Application/Book/DeleteBookApplicationService/DeleteBookApplicationService.test.ts
import { InMemoryBookRepository } from 'Infrastructure/InMemory/Book/InMemoryBookRepository';
import { BookId } from 'Domain/models/Book/BookId/BookId';
import { MockTransactionManager } from 'Application/shared/MockTransactionManager';
import { bookTestDataCreator } from 'Infrastructure/shared/Book/bookTestDataCreator';
import {
  DeleteBookApplicationService,
  DeleteBookCommand,
} from './DeleteBookApplicationService';

describe('DeleteBookApplicationService', () => {
  it('書籍を削除することができる', async () => {
    const repository = new InMemoryBookRepository();
    const mockTransactionManager = new MockTransactionManager();
    const deleteBookApplicationService = new DeleteBookApplicationService(
      repository,
      mockTransactionManager
    );

    // テスト用データ作成
    const bookId = '9784167158057';
    await bookTestDataCreator(repository)({
      bookId,
    });

    const command: Required<DeleteBookCommand> = { bookId };
    await deleteBookApplicationService.execute(command);

    const deletedBook = await repository.find(new BookId(bookId));
    expect(deletedBook).toBe(null);
  });
});
```

このテストでは、`DeleteBookApplicationService`を利用して、書籍を削除することができることを確認しています。

`jest` コマンドでテストを実行し、テストが成功することを確認します。

```bash:StockManagement/
$ jest DeleteBookApplicationService.test.ts
```

以上で、CRUD 操作を実現するアプリケーションサービスの実装は完了です。

基本的なユースケースはこれらの実装をベースに表現することが可能です。アプリケーションサービスの実装で重要なのは、ドメイン知識をドメインオブジェクト内に閉じ込め、ドメインオブジェクトを利用してユースケースを実現することです。アプリケーションサービス内にドメイン知識が漏れていないかを常に意識して実装するようにしましょう。

# まとめ

- アプリケーションサービスは、ユースケースを実現するための操作を提供するサービス
- アプリケーションサービスは、ドメインオブジェクトを利用してユースケースを実現する

本章では、アプリケーションサービスの説明と実装を行いました。
次章では、プレゼンテーション層の実装を行います。API を利用してアプリケーションサービスを呼び出し、システムを構築していきましょう。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/application-service
