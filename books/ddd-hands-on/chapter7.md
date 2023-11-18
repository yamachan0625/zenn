---
title: '第2部 戦術的設計 (コード実装)'
---

# 1 部のまとめと 2 部で何するか入れる

# 戦術的設計とは

戦術的設計とは、ビジネスの複雑性をコードにどのように反映させるか、具体的なアプローチやパターンに焦点を当てたものです。第 1 部で行ったドメインモデリングでの成果物を DDD のアプローチやパターン (値オブジェクト、エンティティ、集約、リポジトリなど) を紹介すると共に実際にコードに反映していきます。

# アーキテクチャ

DIP 依存関係逆転の原則

# 環境のセットアップ

次の章に進む前に、TypeScript の環境をセットアップしましょう。
node のバージョンは 「18.15.0」 です。

```bash
$ node -v
v18.15.0
```

### npm プロジェクトの初期化

```bash:StockManagement/
$ npm init -y
```

### src ディレクトリの作成

src ディレクトリを作成し Domain ディレクトリを移動します。

```bash:StockManagement/
$ mkdir src && mv Domain/ src/
```

### パッケージのインストール

必要なパッケージをインストールしていきます。

```bash:StockManagement/
$ npm i -D typescript ts-node tsconfig-paths @types/node jest ts-jest @types/jest
```

### TypeScript の設定

tsconfig.json を作成し以下のコードをコピーします。オプションはお好みで追加してください。

```json:StockManagement/tsconfig.json
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
    "esModuleInterop": true
  },
  "include": ["src/**/*.ts", "src/**/*.js"],
  "exclude": ["node_modules"]
}

```

### TypeScript の動作確認

「sayHello.ts」 を作成し以下のコードをコピーします。

```js:StockManagement/src/sayHello.ts
export const sayHello = (name: string): void => {
  console.log(`Hello ${name}!`);
};

sayHello('World');

```

```bash:StockManagement/
$ ts-node src/sayHello.ts
```

ts-node で実行しターミナルに「Hello World!」が表示されば OK です。

### Jest の設定

「jest.config.js」 を作成し以下のコードをコピーします。

```js:StockManagement/jest.config.js
/** @type {import('ts-jest').JestConfigWithTsJest} */
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  moduleDirectories: ['node_modules', 'src'],
  transformIgnorePatterns: ['/node_modules'],
};
```

### Jest の動作確認

テストには jest を利用します。
「sayHello.test.ts」 を作成し以下のコードをコピーします。

```js:StockManagement/src/sayHello.test.ts
import { sayHello } from './sayHello';

test('sayHello', () => {
  expect(sayHello('World')).toBe('Hello World');
});

```

```bash:StockManagement/
$ jest src/sayHello.test.ts
```

テストに成功すれば OK です。
「sayHello.ts」と「sayHello.test.ts」は以降使用しないため削除しましょう。

# まとめ

以上で環境のセットアップは完了です。それでは次章から戦術的設計のアプローチを詳しく見ていきましょう。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/setup
