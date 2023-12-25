## Prismaを使う
プロジェクトを作成する
```bash
nest new プロジェクト名
```

[Prisma公式](https://www.prisma.io/docs/orm/prisma-client/setup-and-configuration/introduction)

[NestJSのPrismaのドキュメント](https://docs.nestjs.com/recipes/prisma)


prisma clientをインストール
```bash
npm install @prisma/client
```

prismaをインストール
```bash
npm install prisma --save-dev
npx prisma
```

- コントローラーとサービスを削除する.
- bookディレクトリを作成する。

Prismaをセットアップする
```bash
npx prisma init
```

スキーマーを定義したらマイグレーションをする
```
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

datasource db {
 provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model Book {
  id        Int      @id @default(autoincrement())
  title String
  description String?
}
```

コマンドを実行

```bash
npx prisma migrate dev --name init
```

```bash
npx prisma generate
```

--------

## Installation

```bash
$ npm install
```

## Running the app

```bash
# development
$ npm run start

# watch mode
$ npm run start:dev

# production mode
$ npm run start:prod
```

HTTTP POSTするときは登録されているユーザーのメールあアドレスと一致している必要がある。
![Alt text](<スクリーンショット 2023-12-26 8.02.59.png>)

## Test

```bash
# unit tests
$ npm run test

# e2e tests
$ npm run test:e2e

# test coverage
$ npm run test:cov
```

