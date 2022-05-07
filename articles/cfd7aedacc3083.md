---
title: "NestJSで新規プロジェクトを作るときの備忘録"
emoji: "😺"
type: "tech"
topics: ["typescript", "nestjs"]
published: true
---

# はじめに

個人的には最近のバックエンドはサーバレスにNestJSが一択になりつつあります。明日の自分のために、NestJS v8で`nest new`直後に行っている開発前の下準備の備忘録になります。

https://nestjs.com/

https://github.com/himorishige/nestjs-template

# 1.NestJSのプロジェクトを作成

CLIをまだ未インストールの場合はNestJSのCLIをグローバルにインストールします。

```bash
$ npm i -g @nestjs/cli
```

CLIを利用して新規プロジェクトを作成します。
今回はsample-appとしてみました。
最初にパッケージマネージャとしてnpmかyarnを使うか聞かれますが今回はyarnで進めています。

```bash
$ nest new sample-app

? Which package manager would you ❤️  to use? yarn
✔ Installation in progress... ☕

🚀  Successfully created project todo-app
👉  Get started with the following commands:

$ cd sample-app
$ yarn run start
```

# 2.必須パッケージのインストール

個人的にほぼ必ず利用するパッケージ類をインストールします。

## @nestjs/config

NestJSで.envなどの環境設定ファイルを利用するためのパッケージ@nestjs/configをインスールしておきます。

```
$ yarn add @nestjs/config
```

## TypeORM

DBを利用するためにTypeORMと、各種バリデーションを利用するためのパッケージをインストールします。

```bash
$ yarn add typeorm @nestjs/typeorm
$ yarn add class-validator class-transformer
```

NestJS v8以上利用の場合、TypeORMなどを利用の場合下記エラーが出るため、rxjsをv7以上にする必要があります。

`TypeError: rxjs_1.lastValueFrom is not a function`

```bash
$ yarn add rxjs@^7
```

参考記事
https://stackoverflow.com/questions/68317383/typeerror-rxjs-1-lastvaluefrom-is-not-a-function

## データベースの用意

今回は簡易的な開発用にsqlite3を利用します。
私の環境（M1 mac環境）ではsqlite3がうまくインストールできなかったため、mapboxのGitHubリポジトリから拝借しています。

```bash
$ yarn add https://github.com/mapbox/node-sqlite3/archive/refs/tags/v4.2.0.tar.gz
```

参考記事
https://stackoverflow.com/questions/67928866/cannot-download-node-sqlite34-2-0-node-pre-gyp-error-tried-to-download403-a

### PostgreSQL利用の場合

postgres接続用のパッケージをインストール。

```bash
$ yarn add pg
```

PostgreSQL利用の場合は簡易的なDBをDockerでシンプルに。

```yaml:docker-compose.yml
version: '3'
services:
  db:
    image: postgres
    restart: always
    ports:
      - '5432:5432'
    environment:
      POSTGRES_PASSWORD: pass12
```

# 3. `main.ts`の編集

CORSの設定を追加します。

```typescript:/src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableCors(); // CORSを有効化
  await app.listen(3000);
}
bootstrap();
```

# 4.`app.module.ts`の編集

main.tsにコードを追記していくとモジュールごとのテストコードが書きづらくなるため、`main.ts`はほぼそのまま、`app.module.ts`に設定を記載します。
このあたりのベストプラクティスはまだまだ模索中です。。

```typescript:/src/app.module.ts
import { Module, ValidationPipe } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { APP_PIPE } from '@nestjs/core';
import { TypeOrmModule } from '@nestjs/typeorm';
import { AppController } from './app.controller';
import { AppService } from './app.service';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
    }),
    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => {
        return {
          // sqlite利用
          type: 'sqlite',
          database: config.get<string>('DATABASE_NAME'),
          // postgreSQL利用の場合
          // type: 'postgres',
          // host: config.get<string>('DATABASE_HOST'),
          // port: +config.get<string>('DATABASE_PORT'),
          // username: config.get<string>('DATABASE_USER'),
          // password: config.get<string>('DATABASE_PASSWORD'),
          // database: config.get<string>('DATABASE_NAME'),
          entities: [
            // Entitiesを記載
          ],
          synchronize: true,
        };
      },
    }),
  ],
  controllers: [AppController],
  providers: [
    AppService,
    // テストしやすいようにmain.tsから分離
    {
      provide: APP_PIPE,
      useValue: new ValidationPipe({
        whitelist: true,
      }),
    },
  ],
})
export class AppModule {
  // Middlewareを利用の場合はこちらに
}

```

DBの設定は`.env`ファイルに記載します。

```bash:.env
# for sqlite3
DATABASE_NAME=test.sqlite
# for postgres
#DATABASE_USER=postgres
#DATABASE_PASSWORD=pass123
#DATABASE_NAME=postgres
#DATABASE_PORT=5432
#DATABASE_HOST=localhost
```

# 5.その他

テストコードのテンプレートなどを準備中。。📝

# さいごに

NestJSは多機能で何でもできる？パワフルなフレームワークです。最近はバックエンドを作るときはほぼ一択で採用しています。NestJSはExpressよりもコードの秩序がかなり改善されるのでおすすめです。
とはいえ、自分にとってのベストプラクティスへの道のりはまだまだ続きそうです。。🤔