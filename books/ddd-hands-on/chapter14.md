---
title: 'プレゼンテーション'
---

# プレゼンテーションとは

プレゼンテーションとは、アプリケーションの**ユーザーインターフェース**部分です。MVC で言うと、プレゼンテーション層は `Controller` に相当します。ユーザーの入力を受け取り、処理結果をわかりやすい形で返却します。
ユーザーインターフェースにはたくさんの種類があります。たとえば Web アプリケーションにおいては、 `HTML`、`API` 、`CLI` などがこれに該当します。**ユーザー**とは人間を指しているわけではありません。Web アプリケーションの場合はブラウザ、CLI の場合はコマンドライン、API の場合はリクエストを送信するクライアントなどです。

プレゼンテーションは、アプリケーションサービスのクライアントであり、ユーザーからの入力をアプリケーションサービスに渡します。そしてアプリケーションサービスからの処理結果を、ユーザーがわかりやすい形で返却します。「わかりやすい形」とは、たとえば表示に適した日付のフォーマットや、ユーザーが理解しやすいエラーメッセージなどです。

また、ドメイン層やアプリケーション層から投げられた例外をキャッチするのもプレゼンテーションの役割です。例外をキャッチしたら、ユーザーインターフェースや例外の内容によって適切なハンドリングを行います。

:::message
プレゼンテーション層は、特定の技術や Web アプリケーションフレームワークに依存することが許可されます。それらの依存をドメイン層やアプリケーション層に持ち込まないように注意し、プレゼンテーション層に閉じ込めることが重要です。
:::

# Express.js を利用した実装

それでは、`Express.js`を利用した`API`の実装を行い、`API` を叩いてデータベースにデータを登録するまでの流れを確認します。

## 環境のセットアップ

まずは、`Express.js`を利用するための環境をセットアップします。

### パッケージのインストール

必要なパッケージをインストールしていきます。

```bash:StockManagement/
$ npm install express
$ npm i --save-dev @types/express
```

### 動作確認

`src` ディレクトリ配下に`Presentation/Express`ディレクトリを作成します。次に、`index.ts`ファイルを作成し、以下のように実装します。

```typescript:StockManagement/src/Presentation/Express/index.ts
const express = require('express')
const app = express()
const port = 3000

app.get('/', (_, res) => {
  res.send('Hello World!')
})

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`)
})
```

サーバの起動を行います。

```bash:StockManagement/
$ npx ts-node src/Presentation/Express/index.ts
```

ブラウザで `http://localhost:3000` にアクセスし、`Hello World!` が表示されれば環境のセットアップは完了です。

## 書籍登録 API の実装

それでは、書籍登録 API の実装を行います。まずは、`API` のエンドポイントを作成します。`index.ts`ファイルに以下の実装を追加します。

```js:StockManagement/src/Presentation/Express/index.ts
import express from 'express';

import {
  RegisterBookApplicationService,
  RegisterBookCommand,
} from 'Application/Book/RegisterBookApplicationService/RegisterBookApplicationService';
import { PrismaBookRepository } from 'Infrastructure/Prisma/Book/PrismaBookRepository';
import { PrismaClientManager } from 'Infrastructure/Prisma/PrismaClientManager';
import { PrismaTransactionManager } from 'Infrastructure/Prisma/PrismaTransactionManager';

const app = express();
const port = 3000;

app.get('/', (_, res) => {
  res.send('Hello World!');
});

app.listen(port, () => {
  console.log(`Example app listening on port ${port}`);
});

// JSON形式のリクエストボディを正しく解析するために必要
app.use(express.json());
app.post('/book', async (req, res) => {
  try {
    const requestBody = req.body as {
      isbn: string;
      title: string;
      priceAmount: number;
    };

    const clientManager = new PrismaClientManager();
    const transactionManager = new PrismaTransactionManager(clientManager);
    const bookRepository = new PrismaBookRepository(clientManager);
    const registerBookApplicationService = new RegisterBookApplicationService(
      bookRepository,
      transactionManager
    );

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

`API`のエンドポイントは`/book`です。`POST`メソッドでリクエストを受け取り、`RegisterBookApplicationService`のコマンドの型に合わせてリクエストボディを変換しています。
`RegisterBookApplicationService`には、`Prisma`を利用した`BookRepository`と`TransactionManager`を注入しています。その後実行し、結果をレスポンスとして返却しています。

### 動作確認

それでは、実際に`API`のエンドポイントにリクエストを送信し、動作確認を行います。`DB` が立ち上がっていない場合は、`docker-compose up -d`であらかじめ立ち上げてください。
ここでは`curl`コマンドを利用してリクエストを送信します。

```bash:StockManagement/
$ curl -X POST -H "Content-Type: application/json" -d '{"isbn":"9784167158057","title":"吾輩は猫である","priceAmount":770}' http://localhost:3000/book
{"message":"success"}
```

`{"message":"success"}`というレスポンスが返却されれば、正しく登録が完了しています。
`prisma`では、`DB`のデータを確認するための GUI が用意されています。`npx prisma studio`を実行すると、` http://localhost:5555`で GUI が立ち上がります。`Book`テーブルを確認し、登録されていることを確認します。

![](https://storage.googleapis.com/zenn-user-upload/3ab24c2c1f17-20231209.png)

次に、再度同じリクエストを送信してみます。

```bash:StockManagement/
$ curl -X POST -H "Content-Type: application/json" -d '{"isbn":"9784167158057","title":"吾輩は猫である","priceAmount":770}' http://localhost:3000/book
{"message":"既に存在する書籍です"}
```

`{"message":"既に存在する書籍です"}`というレスポンスが返却されデータの登録に失敗します。これは、`isbn`が一意であることを保証するために、`RegisterBookApplicationService`内でドメインサービスを利用しチェックを行っているためです。エラーのメッセージは、ドメインサービスで定義したものがそのまま返却されています。

レスポンスは、`API`の仕様に合わせて適切に定義する必要があります。ここでは、考慮せずに簡易的に実装していますが、実際には仕様に合わせてレスポンスを定義する必要があります。

以上で、`Express.js`を利用した`API`の実装は完了とします。ここでは省略しますが、その他のエンドポイントも同様に実装して行くことができます。

# まとめ

- プレゼンテーション層は、アプリケーションのユーザーインターフェースである

本章では、プレゼンテーション層の実装を行いました。`Express.js`を利用した`API`の実装を行い、`API` を叩いてデータベースにデータを登録するまでの流れを確認しました。この本ではプレゼンテーションの一例として`API`の実装を行いましたが、`CLI`などのその他プレゼンテーションの実装も同様です。リクエストを受け取り、データを変換しアプリケーションサービスを呼び出し、ユーザーに沿った結果を返却するという流れは同じです。

以上でドメイン駆動設計の基本的な戦術的設計 (コード実装)は終了です。お疲れ様でした！ここまでの実装を通して、ドメイン駆動設計の特有な考え方や実装方法を学びました。実際の開発現場でドメイン駆動設計を利用する際に、この本で学んだことを活かしていただければ幸いです。

次章からは、より実践的な内容を学んでいきます。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/presentation
