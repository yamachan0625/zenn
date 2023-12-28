---
title: 'ESLintで不正な依存関係を防ぐ'
---

# 不正な依存関係

第 2 部では、ドメイン層を独立して実装するために、「**ドメイン層がインフラストラクチャや特定の技術に依存しないようにしましょう**」と繰り返し言ってきました。具体的な対応としては、オニオンアーキテクチャと依存性逆転の原則 (DIP) を適用し、ドメイン層への依存を防ぐような構成を用いました。

しかし、これらの制約は口約束でしかなく、強制力がありません。チームで開発している場合、コードレビューで指摘することも可能ですが、レビュー漏れは起こりえます。極端な例ですが以下のようにドメイン層でインフラストラクチャのコードを呼び出してしまうことも可能です。

```js:.../src/Domain/models/Book/Book.ts
import prisma from 'Infrastructure/Prisma/prismaClient';

export class Book {
  private constructor(
    private readonly _bookId: BookId,
    private _title: Title,
    private _price: Price,
    private readonly _stock: Stock
  ) {}

  // ドメイン層のエンティティでprismaを利用しDBを操作できてしまう
  async insertDB(book: Book) {
    await prisma.book.create({
      data: {
        bookId: book.bookId.value,
        title: book.title.value,
        priceAmount: book.price.value.amount,
      },
    });
  }
}
```

これでは、せっかく保っていたドメイン層の独立性が崩れてしまいます。そこで`ESLint`でのアプローチを利用することで、このような不正な依存関係を防ぐことができます。

## 環境のセットアップ

まずは、`ESLint` の環境をセットアップしましょう。詳しくは[こちら](https://eslint.org/docs/latest/use/getting-started)を参照してください。

### パッケージのインストール

必要なパッケージをインストールしていきます。

```bash
$ npm install --save-dev eslint eslint-plugin-import eslint-import-resolver-typescript @typescript-eslint/eslint-plugin
```

### 初期設定

ルートディレクトリに `.eslintrc.js` ファイルを作成し、以下のように設定します。ここでは最低限の設定のみ行います。

```js:stockManagement/.eslintrc.js
module.exports = {
  env: {
    node: true,
    es6: true,
  },
  extends: [
    'plugin:import/recommended',
    'plugin:import/typescript',
    'plugin:@typescript-eslint/recommended',
  ],
  parserOptions: {
    ecmaVersion: 'latest',
    sourceType: 'module',
  },
  // TypeScriptの場合に必要な設定
  settings: {
    'import/resolver': {
      typescript: {},
    },
  },
};
```

これでセットアップは完了です。

## 不正な依存関係の洗い出し

設定の前に、再度オニオンアーキテクチャの図を確認し、不正な依存関係を洗い出しておきましょう。

![](https://storage.googleapis.com/zenn-user-upload/ea8459dbb430-20231121.png)

**ドメイン層が依存してはいけない層**

- アプリケーション層
- インフラストラクチャ層
- プレゼンテーション層

**アプリケーション層が依存してはいけない層**

- インフラストラクチャ層
- プレゼンテーション層

これらが不正な依存関係となります。

実装において、依存してはいけないということは、import してはいけないということに言い換えることができます。ですが、import すること自体を防ぐことはできないため、不正な import がされた時にエラーを出し、検知できるような仕組みを`ESLint`を利用して構築していきます。

# ESLint で不正な依存関係を防ぐ

`ESLint`には、さまざまなプラグインが用意されています。その中でも
`eslint-plugin-import`というプラグインの`import/no-restricted-paths`というルールを利用し、以下のように設定します。

```js:stockManagement/.eslintrc.js
module.exports = {
  (省略)
  rules: {
    'import/no-restricted-paths': [
      'error',
      {
        zones: [
          // Domain層が依存してはいけない領域
          {
            from: './src/Application/**/*',
            target: './src/Domain/**/!(*.spec.ts|*.test.ts)',
            message: 'Domain層でApplication層をimportしてはいけません。',
          },
          {
            from: './src/Presentation/**/*',
            target: './src/Domain/**/!(*.spec.ts|*.test.ts)',
            message: 'Domain層でPresentation層をimportしてはいけません。',
          },
          {
            from: './src/Infrastructure/**/*!(test).ts',
            target: './src/Domain/**/!(*.spec.ts|*.test.ts)',
            message: 'Domain層でInfrastructure層をimportしてはいけません。',
          },
          // Application層が依存してはいけない領域
          {
            from: './src/Presentation/**/*',
            target: './src/Application/**/!(*.spec.ts|*.test.ts)',
            message: 'Application層でPresentation層をimportしてはいけません。',
          },
          {
            from: './src/Infrastructure/**/*',
            target: './src/Application/**/!(*.spec.ts|*.test.ts)',
            message:
              'Application層でInfrastructure層をimportしてはいけません。',
          },
        ],
      },
    ],
  },
};
```

`import/no-restricted-paths` ルールで使用される 「from」, 「target」, 「message」 の各フィールドは、次のような役割を持っています。
| フィールド | 説明 |
|------------|------|
| `from` | このフィールドは、制限を設けたいソースの場所 (ディレクトリまたはファイル) を指定します。この設定では、特定のディレクトリからの import を監視したい場合にこのフィールドを使用します。 |
| `target` | このフィールドは、`from`で指定されたソースから import されるべきでないターゲットの場所（ディレクトリまたはファイル）を定義します。たとえば、ドメイン層のファイルがインフラストラクチャ層のファイルを import してはならない場合、ドメイン層を`target`として指定します。 |
| `message` | このフィールドは、ルールに違反した際に表示されるエラーメッセージをカスタマイズします。このメッセージは、開発者に対して何が問題であるかを具体的に説明し、アーキテクチャの設計原則を遵守するよう促すために使用されます。 |

設定が完了したので、実際に不正な依存関係の import を行い動作確認をしてみましょう。

```js:stockManagement/src/Domain/models/Book/Book.ts
import prisma from 'Infrastructure/Prisma/prismaClient';
```

ドメイン層の`Book.ts`にインフラストラクチャ層の`PrismaClient`の import を追加しました。赤い波線が出てくるのでマウスカーソルを合わせると、`Unexpected path "Infrastructure/Prisma/prismaClient" imported in restricted zone. Domain層でInfrastructure層をimportしてはいけません。`というエラーが表示されます。

![](https://storage.googleapis.com/zenn-user-upload/256e76220aa0-20231210.png)

`ESLint` の `import/no-restricted-paths` ルールを適切に設定することで、誤って不正な依存関係を持つ実装が行われた際にエラーが発生するようになります。これにより、開発者は実装時に即座に問題を検知できます。さらに、連続的インテグレーション (CI) 環境で `ESLint` を実行することにより、不正な依存関係を持つコードがリポジトリにマージされることを効果的に防ぐことが可能です。このような手法により、プロジェクトのアーキテクチャと品質を維持するための重要な安全網を提供することができます。

# まとめ

本章では、`ESLint`を利用して不正な依存関係を防ぐ方法、つまり不正な import を防ぐ方法を学びました。`ESLint`は、依存関係を制御し、ドメイン層の独立性を保つために必須のツールと言えるでしょう。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/eslint
