---
title: '第2部 戦術的設計'
---

# 戦術的設計とは

戦術的設計とは、ビジネスの複雑性をコードにどのように反映させるか、具体的なアプローチやパターンに焦点を当てたものです。第 1 部のメインモデリングでの成果物を DDD のアプローチやパターンを紹介すると共にコードに反映していきます。

# 環境のセットアップ

次の章に進む前に、TypeScript の環境をセットアップしましょう。
node のバージョンは 「18.15.0」 です。

```bash
$ node -v
v18.15.0
```

### npm プロジェクトの初期化

```bash:OnlineBookstore/StockManagementDomain
$ npm init -y
```

### src ディレクトリの作成

src ディレクトリを作成し Domain ディレクトリを移動します。

```bash:OnlineBookstore/StockManagementDomain
$ mkdir src && mv Domain/ src/
```

### パッケージのインストール

必要なパッケージをインストールしていきます。

```bash:OnlineBookstore/StockManagementDomain
$ npm i -D typescript ts-node tsconfig-paths @types/node jest ts-jest @types/jest
```

### TypeScript の設定

tsconfig.json を作成し以下のコードをコピーします。オプションはお好みで追加してください。

```json:OnlineBookstore/StockManagementDomain/tsconfig.json
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

```js:OnlineBookstore/StockManagementDomain/src/sayHello.ts
export const sayHello = (name: string): void => {
  console.log(`Hello ${name}!`);
};

sayHello('World');

```

```bash:OnlineBookstore/StockManagementDomain
$ ts-node src/sayHello.ts
```

ts-node で実行しターミナルに「Hello World!」が表示されば OK です。

### Jest の設定

「jest.config.js」 を作成し以下のコードをコピーします。

```js:OnlineBookstore/StockManagementDomain/jest.config.js
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

```js:OnlineBookstore/StockManagementDomain/src/sayHello.test.ts
import { sayHello } from './sayHello';

test('sayHello', () => {
  expect(sayHello('World')).toBe('Hello World');
});

```

```bash:OnlineBookstore/StockManagementDomain
$ jest src/sayHello.test.ts
```

テストに成功すれば OK です。
「sayHello.ts」と「sayHello.test.ts」は削除しましょう。

# まとめ

以上で環境のセットアップは完了です。それでは次章から戦術的設計のアプローチを詳しく見ていきましょう。

### これまでのコード

https://github.com/yamachan0625/ddd-hands-on/tree/setup
