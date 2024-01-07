---
title: 'DI コンテナで依存関係を切り替える'
---

# DI コンテナとは

DI コンテナとは、オブジェクトのインスタンス化と依存関係の解決を行うコンテナのことです。

これまで、依存性逆転の原則 (DIP) に従って、依存関係を注入 (DI) することで、オブジェクトの拡張性やテストの容易性を向上させることについて学んできました。
たとえば、アプリケーションサービスでは、リポジトリをコンストラクタインジェクションで注入することで、特定の技術への依存を排除し、アプリケーションサービスのテストを容易にしました。

これまでの実装では、オブジェクトのインスタンス化を行う際にコンストラクタに依存オブジェクトを明示的に渡していました。たとえば、アプリケーションサービスではリポジトリを DI する必要があり、テストでは明示的にインメモリのリポジトリをインスタンス化し、コンストラクタに渡していました。しかし、この方法では、依存オブジェクトのインスタンス化と依存関係の解決をアプリケーションサービスのコンストラクタの外側で行う必要があります。このため、依存オブジェクトのインスタンス化と依存関係の解決を行うコードが散在してしまい、コードの可読性が低下してしまいます。

また、開発時にはインメモリデータベースを利用し、本番環境では `PostgreSQL` を利用するように切り替えたい場合、リポジトリのインスタンス化を行っている箇所をすべて洗い出し、差し替えていく必要がありあます。このような作業は、ヒューマンエラーを誘発する原因となります。

**DI コンテナ**を利用することで、依存オブジェクトのインスタンス化と依存関係の解決を行うコードを集約することができます。また、差し替えも DI コンテナの設定を変更するだけで済むため、容易になります。

# tsyringe を利用して DI コンテナを作成する

それでは実際に DI コンテナを利用してみましょう。ここでは、`Microsoft`が提供している`TypeScript/JavaScript `用の DI コンテナライブラリ[tsyringe](https://github.com/microsoft/tsyringe)を利用します。

## 環境のセットアップ

まずは`tsyringe`の環境をセットアップしましょう。

### パッケージのインストール

必要なパッケージをインストールしていきます。

```bash
$ npm install --save tsyringe reflect-metadata
```

### 初期設定

次の設定を含むように `tsconfig.json` を変更します

```diff json:StockManagement/tsconfig.json
  {
    "ts-node": {
      "require": ["tsconfig-paths/register"]
      // ts-nodeがtsconfigのpathsを解決できるようにします。
    },
    "compilerOptions": {
      "outDir": "./dist",
      "strict": true,
      "resolveJsonModule": true,
      "noUnusedLocals": true,
      "noUnusedParameters": true,
      "baseUrl": "./",
      "paths": {
        "*": ["./src/*"]
      },
      "esModuleInterop": true,
+     "experimentalDecorators": true,
+     "emitDecoratorMetadata": true
    },
    "include": ["src/**/*.ts", "src/**/*.js"],
    "exclude": ["node_modules"]
  }
```

テストでポリフィルの影響を受けるため、テストが開始される前にポリフィルを読み込むような設定を行います。`setupJest.ts`ファイルを作成し、以下のように実装します。

```ts:setupJest.ts
import 'reflect-metadata';
```

`jest` では`jest.config.js`の`setupFilesAfterEnv`を利用してテストが実行される前に適当な処理を行うことができます。`setupJest.ts`を読み込むように設定を変更します。

```diff js:StockManagement/jest.config.js
  /** @type {import('ts-jest').JestConfigWithTsJest} */
  module.exports = {
    preset: 'ts-jest',
    testEnvironment: 'node',
    moduleDirectories: ['node_modules', 'src'],
    transformIgnorePatterns: ['/node_modules'],
+   setupFilesAfterEnv: ['./setupJest.ts'],
};
```

以上で環境のセットアップは完了です。

`tsyringe`を利用し、依存オブジェクトのインスタンス化と依存関係の解決を行うためには、以下のような設定が必要となります。

- デコレータの設定
- 依存オブジェクトの登録
- 依存オブジェクトの解決

それでは、これらの設定を行なっていきましょう。

## デコレータの設定

まず初めに、依存関係として DI されるクラスに `@injectable()` デコレータを適用します。次にコンストラクタインジェクションする引数に `@inject('Interface')`デコレータを適用します。これにより、特定のインターフェイスに対応する依存オブジェクトを DI コンテナを利用して注入することができます。依存オブジェクトとは、たとえばアプリケーションサービスで実行時に利用したいリポジトリを指します。

それでは 依存関係として DI されるクラスに対して上記の設定を行なっていきましょう。

```diff js:src/Infrastructure/Prisma/PrismaTransactionManager.ts
+ import { injectable, inject } from 'tsyringe';
  ...

+ @injectable()
  export class PrismaTransactionManager implements ITransactionManager {
    constructor(
+    @inject('IDataAccessClientManager')
      private clientManager: PrismaClientManager
    ) {}
    (省略)
  }
```

```diff js:src/Infrastructure/Prisma/PrismaTransactionManager.ts
+ import { injectable, inject } from 'tsyringe';
  ...

+ @injectable()
  export class PrismaBookRepository implements IBookRepository {
    constructor(
+     @inject('IDataAccessClientManager')
      private clientManager: PrismaClientManager
    ) {}
    (省略)
  }
```

```diff js:...src/Application/Book/RegisterBookApplicationService/RegisterBookApplicationService.ts
+ import { injectable, inject } from 'tsyringe';
  ...

+ @injectable()
  export class RegisterBookApplicationService {
    constructor(
+     @inject('IBookRepository')
      private bookRepository: IBookRepository,
+     @inject('ITransactionManager')
      private transactionManager: ITransactionManager
    ) {}
    (省略)
  }
```

そのほかのアプリケーションサービスに対しても同様にデコレータの設定を行なってください。

## 依存オブジェクトの登録

次に、依存オブジェクトの登録を行います。登録には`tsyringe`が提供する`container.register`メソッドを利用します。

`src/`配下に依存オブジェクト登録用に`Program.ts`ファイルを作成し、以下のように実装します。

```typescript:src/Program.ts
import { container, Lifecycle } from 'tsyringe';

import { PrismaBookRepository } from 'Infrastructure/Prisma/Book/PrismaBookRepository';
import { PrismaClientManager } from 'Infrastructure/Prisma/PrismaClientManager';
import { PrismaTransactionManager } from 'Infrastructure/Prisma/PrismaTransactionManager';

// repository
container.register('IBookRepository', {
  useClass: PrismaBookRepository,
});

// transactionManager
container.register('ITransactionManager', {
  useClass: PrismaTransactionManager,
});

// IDataAccessClientManager
container.register(
  'IDataAccessClientManager',
  {
    useClass: PrismaClientManager,
  },
  // The same instance will be resolved for each resolution of this dependency during a single resolution chain
  { lifecycle: Lifecycle.ResolutionScoped }
);
```

`container.register`の第一引数では登録したいオブジェクトのインターフェイスを指定します。第二引数では、登録したいオブジェクトの実装クラスを指定します。第三引数では、生成するオブジェクトのインスタンスのライフサイクルを指定することができます。ライフサイクルについての説明は[こちら](https://github.com/microsoft/tsyringe#scoped)を参照してください。

この設定により、`container.resolve`を利用してインスタンス化を行なった場合、たとえば`IBookRepository`を DI している箇所で`PrismaBookRepository`のインスタンス化が行なわれます。

テスト時にはインメモリのリポジトリを使いたくなるでしょう。その場合は、`Program.ts`で登録した`IBookRepository`の実装クラスを`PrismaBookRepository`から`InMemoryBookRepository`に差し替えることで、リポジトリを利用する側のコードに変更を加えずに、インメモリのリポジトリを利用することができます。もしくはテスト用の`Program`ファイルを作成し、テスト実行前にテスト用の`Program`ファイルを読み込ませることによっても同様のことができます。

## 依存オブジェクトの解決

`container.register`によって登録された依存オブジェクトは、`container.resolve`を利用して解決することができます。
それでは、`RegisterBookApplicationService`のインスタンス化を`container.resolve`を利用して行なってみましょう。

```diff js:src/Express/index.ts
+ // Reflectのポリフィルをcontainer.resolveされる前に一度読み込む必要がある
+ import 'reflect-metadata';
+ import '../../Program';
+ import { container } from 'tsyringe';
 ...

 app.post('/book', async (req, res) => {
   try {
     const requestBody = req.body as {
       isbn: string;
       title: string;
       priceAmount: number;
     };

-    const clientManager = new PrismaClientManager();
-    const transactionManager = new PrismaTransactionManager(clientManager);
-    const bookRepository = new PrismaBookRepository(clientManager);
-    const registerBookApplicationService = new RegisterBookApplicationService(
-      bookRepository,
-      transactionManager
-    );

+    const registerBookApplicationService = container.resolve(
+     RegisterBookApplicationService
+     );

     // リクエストボディをコマンドに変換。今回はたまたま一致しているため、そのまま渡している。
     const registerBookCommand: RegisterBookCommand = requestBody;
     await registerBookApplicationService.execute(registerBookCommand);

     // 実際は詳細なレスポンスを返す
     res.status(200).json({ message: 'success' });
   } catch (error) {
     // 実際はエラーを解析し、詳細なレスポンスを返す。また、ロギングなどを行う。
     res.status(500).json({ message: (error as Error).message });
   }
 });
```

まず初めに、`Reflect`のポリフィルの読み込みと、さきほど定義した`Program.ts`の読み込みを行なう必要があります。

次に、`RegisterBookApplicationService`のインスタンス化を`container.resolve`で行なっています。これにより、`RegisterBookApplicationService`のインスタンス化に必要な依存オブジェクト (リポジトリなど) のインスタンス化と依存関係の解決が`Program.ts`で登録した設定に従って行なわれるようになります。

そして、リポジトリやトランザクション管理オブジェクトを明示的にインスタンス化するコードが無くなったことに注目してください。つまり、依存オブジェクトのインスタンス化と依存関係の解決を行うコードが`Program.ts`に集約されたということです。これにより、依存オブジェクトのインスタンス化と依存関係の解決を行うコードが散在することなく、コードの可読性が向上しました。また、差し替えも`Program.ts`を変更するだけで済むため、容易になりました。

以上で、DI コンテナを利用した実装への変更は完了です。

# まとめ

- DI コンテナを利用することで、散財していた依存オブジェクトのインスタンス化と依存関係の解決を行うコードを集約することができる
- DI コンテナを利用することで、コードの可読性の向上と依存オブジェクトの差し替えを容易にすることができる

本章では、DI コンテナを利用して依存オブジェクトのインスタンス化と依存関係の解決を行う方法について学びました。DI コンテナを利用することで、依存オブジェクトのインスタンス化と依存関係の解決を行うコードを集約することができ、コードの可読性が向上しました。また、差し替えも`Program.ts`を変更するだけで済むため、容易になりました。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/DIContainer
