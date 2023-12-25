## NestJS プロジェクトを作成する
[公式チュートリアル](https://docs.nestjs.com/recipes/prisma)

PostgreSQLはこちらを参考に環境構築する:

https://zenn.dev/joo_hashi/articles/3702238384488f

まず、NestJS CLI をインストールし、次のコマンドを使用してアプリのスケルトンを作成します。
```bash
$ npm install -g @nestjs/cli
$ nest new hello-prisma
```

このコマンドで作成されるプロジェクト・ファイルの詳細については、最初のステップのページを参照してください。アプリケーションを起動するためにnpm startを実行できることにも注意してください。http://localhost:3000/ で実行されている REST API は、現在 src/app.controller.ts で実装されている単一のルートを提供しています。このガイドの過程で、ユーザーや投稿に関するデータを保存したり取得したりする追加のルートを実装することになります。

## Prisma#のセットアップ
まず、Prisma CLIを開発依存としてプロジェクトにインストールします：
```bash
$ cd hello-prisma
$ npm install prisma --save-dev
```

以下の手順では、Prisma CLIを使用します。ベストプラクティスとして、npxをプレフィックスとしてCLIをローカルに起動することをお勧めします：

```bash
npx prisma
```

Prisma CLIのinitコマンドを使用して、Prismaの初期設定を作成します：

```bash
npx prisma init
```

このコマンドは、以下の内容で新しいprismaディレクトリを作成する：

schema.prisma: データベース接続を指定し、データベース・スキーマを含みます。
.env： dotenvファイル。通常、データベースの認証情報を環境変数のグループに格納するために使用します。

データベース接続を設定する#。
データベース接続はschema.prismaファイルのdatasourceブロックで設定します。デフォルトではpostgresqlに設定されていますが、このガイドではSQLiteデータベースを使用しているので、データソースブロックのプロバイダフィールドをsqliteに調整する必要があります：

**SQLiteを使う場合**
```
datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

次に、.envを開き、DATABASE_URL環境変数を以下のように調整する：

```
# Environment variables declared in this file are automatically made available to Prisma.
# See the documentation for more detail: https://pris.ly/d/prisma-schema#accessing-environment-variables-from-the-schema

# Prisma supports the native connection string format for PostgreSQL, MySQL, SQLite, SQL Server, MongoDB and CockroachDB.
# See the documentation for all the connection string options: https://pris.ly/d/connection-strings

# DATABASE_URL="postgresql://johndoe:randompassword@localhost:5432/mydb?schema=public"
DATABASE_URL="postgresql://postgres:postgres@localhost:5432/mydb?schema=public"
```

PostgreSQLを使用している場合は、以下のようにschema.prismaと.envファイルを調整する必要があります：

```
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}
```

`.env`
```
DATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?schema=SCHEMA"
```

### Prisma Migrate#を使用して2つのデータベーステーブルを作成する

Prismaをセットアップする
```bash
npx prisma init
```

このセクションでは、Prisma Migrateを使用してデータベースに2つの新しいテーブルを作成します。Prisma Migrateは、Prismaスキーマ内の宣言型データモデル定義に対してSQL移行ファイルを生成します。これらの移行ファイルは完全にカスタマイズ可能なので、基礎となるデータベースの追加機能を構成したり、シーディングなどの追加コマンドを含めることができます。

以下の2つのモデルをschema.prismaファイルに追加します：

```
model User {
  id    Int     @default(autoincrement()) @id
  email String  @unique
  name  String?
  posts Post[]
}

model Post {
  id        Int      @default(autoincrement()) @id
  title     String
  content   String?
  published Boolean? @default(false)
  author    User?    @relation(fields: [authorId], references: [id])
  authorId  Int?
}
```

Prismaモデルを配置して、SQL移行ファイルを生成し、データベースに対して実行できます。ターミナルで次のコマンドを実行します：

```bash
npx prisma migrate dev --name init
```

## Prisma Client#のインストールと生成
Prisma Clientは、Prismaモデル定義から生成されるタイプセーフのデータベースクライアントです。このアプローチにより、Prisma Clientはモデル専用のCRUD操作を公開できます。

プロジェクトにPrisma Clientをインストールするには、ターミナルで次のコマンドを実行します：

```bash
npm install @prisma/client
```

インストール中、Prismaは自動的にprisma generateコマンドを起動します。今後は、Prismaモデルを変更するたびにこのコマンドを実行して、生成されたPrismaクライアントを更新する必要があります。

### 注意
prisma generateコマンドはPrismaスキーマを読み込み、node_modules/@prisma/client内の生成されたPrismaクライアント・ライブラリを更新します。

NestJSサービス#でPrisma Clientを使う
これで、Prisma Clientでデータベースクエリを送信できるようになりました。Prisma Clientを使ったクエリの構築について詳しく知りたい方は、APIドキュメントをご覧ください。

NestJSアプリケーションをセットアップするとき、サービス内のデータベースクエリのためにPrisma Client APIを抽象化したいと思うでしょう。まず、PrismaClientのインスタンス化とデータベースへの接続を行う新しいPrismaServiceを作成します。

srcディレクトリ内に`prisma.service.ts`という新しいファイルを作成し、以下のコードを追加します：

```ts
import { Injectable, OnModuleInit } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit {
  async onModuleInit() {
    await this.$connect();
  }
}
```

### 注意
onModuleInitはオプションです。これを省略すると、Prismaはデータベースへの最初の呼び出し時に遅延接続します。

次に、PrismaスキーマからUserモデルとPostモデルのデータベースを呼び出すためのサービスを記述します。

srcディレクトリ内に`user.service.ts`という新しいファイルを作成し、以下のコードを追加します：

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { User, Prisma } from '@prisma/client';

@Injectable()
export class UserService {
  constructor(private prisma: PrismaService) {}

  async user(
    userWhereUniqueInput: Prisma.UserWhereUniqueInput,
  ): Promise<User | null> {
    return this.prisma.user.findUnique({
      where: userWhereUniqueInput,
    });
  }

  async users(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.UserWhereUniqueInput;
    where?: Prisma.UserWhereInput;
    orderBy?: Prisma.UserOrderByWithRelationInput;
  }): Promise<User[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.user.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createUser(data: Prisma.UserCreateInput): Promise<User> {
    return this.prisma.user.create({
      data,
    });
  }

  async updateUser(params: {
    where: Prisma.UserWhereUniqueInput;
    data: Prisma.UserUpdateInput;
  }): Promise<User> {
    const { where, data } = params;
    return this.prisma.user.update({
      data,
      where,
    });
  }

  async deleteUser(where: Prisma.UserWhereUniqueInput): Promise<User> {
    return this.prisma.user.delete({
      where,
    });
  }
}
```

サービスによって公開されるメソッドが適切に型付けされるように、Prisma Clientの生成された型を使用していることに注意してください。そのため、モデルを型付けしたり、追加のインターフェースファイルやDTOファイルを作成したりする手間が省けます。

Postモデルも同じようにします。

srcディレクトリの中に`post.service.ts`という新しいファイルを作成し、以下のコードを追加します：

```ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from './prisma.service';
import { Post, Prisma } from '@prisma/client';

@Injectable()
export class PostService {
  constructor(private prisma: PrismaService) {}

  async post(
    postWhereUniqueInput: Prisma.PostWhereUniqueInput,
  ): Promise<Post | null> {
    return this.prisma.post.findUnique({
      where: postWhereUniqueInput,
    });
  }

  async posts(params: {
    skip?: number;
    take?: number;
    cursor?: Prisma.PostWhereUniqueInput;
    where?: Prisma.PostWhereInput;
    orderBy?: Prisma.PostOrderByWithRelationInput;
  }): Promise<Post[]> {
    const { skip, take, cursor, where, orderBy } = params;
    return this.prisma.post.findMany({
      skip,
      take,
      cursor,
      where,
      orderBy,
    });
  }

  async createPost(data: Prisma.PostCreateInput): Promise<Post> {
    return this.prisma.post.create({
      data,
    });
  }

  async updatePost(params: {
    where: Prisma.PostWhereUniqueInput;
    data: Prisma.PostUpdateInput;
  }): Promise<Post> {
    const { data, where } = params;
    return this.prisma.post.update({
      data,
      where,
    });
  }

  async deletePost(where: Prisma.PostWhereUniqueInput): Promise<Post> {
    return this.prisma.post.delete({
      where,
    });
  }
}
```

UserServiceとPostServiceは現在、Prisma Clientで利用可能なCRUDクエリをラップしています。実際のアプリケーションでは、サービスはアプリケーションにビジネスロジックを追加する場所でもあります。たとえば、UserService内にupdatePasswordというメソッドを用意して、ユーザーのパスワードを更新することができます。

メインアプリのコントローラにREST APIルートを実装する
最後に、前のセクションで作成したサービスを使用して、アプリのさまざまなルートを実装します。このガイドでは、すべてのルートを既に存在する AppController クラスに実装します。

`app.controller.ts`ファイルの内容を以下のコードに置き換えてください：

```ts
import {
  Controller,
  Get,
  Param,
  Post,
  Body,
  Put,
  Delete,
} from '@nestjs/common';
import { UserService } from './user.service';
import { PostService } from './post.service';
import { User as UserModel, Post as PostModel } from '@prisma/client';

@Controller()
export class AppController {
  constructor(
    private readonly userService: UserService,
    private readonly postService: PostService,
  ) {}

  @Get('post/:id')
  async getPostById(@Param('id') id: string): Promise<PostModel> {
    return this.postService.post({ id: Number(id) });
  }

  @Get('feed')
  async getPublishedPosts(): Promise<PostModel[]> {
    return this.postService.posts({
      where: { published: true },
    });
  }

  @Get('filtered-posts/:searchString')
  async getFilteredPosts(
    @Param('searchString') searchString: string,
  ): Promise<PostModel[]> {
    return this.postService.posts({
      where: {
        OR: [
          {
            title: { contains: searchString },
          },
          {
            content: { contains: searchString },
          },
        ],
      },
    });
  }

  @Post('post')
  async createDraft(
    @Body() postData: { title: string; content?: string; authorEmail: string },
  ): Promise<PostModel> {
    const { title, content, authorEmail } = postData;
    return this.postService.createPost({
      title,
      content,
      author: {
        connect: { email: authorEmail },
      },
    });
  }

  @Post('user')
  async signupUser(
    @Body() userData: { name?: string; email: string },
  ): Promise<UserModel> {
    return this.userService.createUser(userData);
  }

  @Put('publish/:id')
  async publishPost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.updatePost({
      where: { id: Number(id) },
      data: { published: true },
    });
  }

  @Delete('post/:id')
  async deletePost(@Param('id') id: string): Promise<PostModel> {
    return this.postService.deletePost({ id: Number(id) });
  }
}
```

### seed.tsの作成
Prismaを使用してデータベースにシードデータを挿入するには、まずPrisma Clientを使用してデータを作成するスクリプトを作成します。このスクリプトは通常、prisma/seed.ts（またはprisma/seed.js）という名前のファイルに配置します。

以下に、あなたのデータベーススキーマに基づいてユーザーと投稿を作成するシードスクリプトの例を示します：

```ts
// prisma/seed.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

async function main() {
  const user1 = await prisma.user.create({
    data: {
      email: 'alice@example.com',
      name: 'Alice',
      posts: {
        create: {
          title: 'Hello World',
          content: 'Welcome to Prisma',
          published: true,
        },
      },
    },
    include: {
      posts: true,
    },
  });

  console.log({ user1 });
}

main()
  .catch((e) => {
    console.error(e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

このスクリプトは、新しいユーザーを作成し、そのユーザーに対して新しい投稿を作成します。

次に、このシードスクリプトを実行するためのnpmスクリプトをpackage.jsonに追加します：

```json
{
  "scripts": {
    "seed": "ts-node prisma/seed.ts"
  }
}
```

これで、npm run seedコマンドを実行することでシードデータをデータベースに挿入できます。ただし、このコマンドを実行する前に、npx prisma migrate devコマンドを実行してデータベースのマイグレーションを適用する必要があります。

```bash
$ npm run seed
$ npx prisma migrate dev
```

prisma studioを実行して確認する
```bash
npx prisma studio
```

`app.modules.ts`を設定する
```ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { PostService } from './post.service';
import { PrismaService } from './prisma.service';
import { UserService } from './user.service';

@Module({
  imports: [],
  controllers: [AppController],
  providers: [PostService, PrismaService, UserService],
})
export class AppModule {}
```

------

NestJSアプリケーションはデフォルトでポート3000で起動します。したがって、あなたのローカルマシン上でアプリケーションを実行している場合、以下のURLで各エンドポイントにアクセスできます：

投稿をIDで取得：http://localhost:3000/post/{id}

公開された投稿を取得：http://localhost:3000/feed

検索文字列でフィルタリングされた投稿を取得：http://localhost:3000/filtered-posts/{searchString}

ドラフトを作成：http://localhost:3000/post（POSTリクエスト）

ユーザー登録：http://localhost:3000/user（POSTリクエスト）

投稿を公開：http://localhost:3000/publish/{id}（PUTリクエスト）

投稿を削除：http://localhost:3000/post/{id}（DELETEリクエスト）

ただし、これらのエンドポイントはHTTPメソッド（GET、POST、PUT、DELETE）によって異なる動作をしますので、適切なメソッドを使用してリクエストを送信してください。また、{id}や{searchString}は具体的な値に置き換えてください。
