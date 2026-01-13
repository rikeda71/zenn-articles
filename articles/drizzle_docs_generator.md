---
title: "drizzle のスキーマ定義からドキュメントを自動生成するツール drizzle-docs-generator"
emoji: "🦁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [drizzle, database, typescript, orm]
published: false
---

drizzle は TypeScript でスキーマ定義が記述できることや軽量であることから注目されている ORM です。
v1.0 がβリリースされており、正式公開も近そうです。

drizzle のスキーマ定義から Markdown 形式のドキュメント及び DBML 形式のテーブル定義を生成する CLI ツールを開発したので紹介します。

https://github.com/rikeda71/drizzle-docs-generator

## 課題感

drizzle はスキーマ定義からドキュメントを生成する方法を備えていません。

drizzle-kit を使うことで DDL を生成できるため、DDL を経由して、間接的にスキーマ定義からドキュメントを作成可能です。
しかし、comment 句に対応していないため、カラムやテーブルの説明を確認したい場合、別途用意したドキュメントや drizzle のスキーマ定義上の JSDoc を確認する必要があります。

:::message
comment 句への対応の要望は Issue に起票されています。

https://github.com/drizzle-team/drizzle-orm/issues/886

ただし、2026/1 時点では v1 Roadmap にも対応に関する記載はなく、今後も対応されるかわかりません。

https://orm.drizzle.team/roadmap
:::


## drizzle-docs-generator の紹介

[drizzle-docs-generator](https://github.com/rikeda71/drizzle-docs-generator) は drizzle のスキーマ定義からカラムやテーブルに関する説明を含めたドキュメントを自動生成でき、上述のような問題を解決できます。

npm からインストールできます。

https://www.npmjs.com/package/drizzle-docs-generator

```bash
$ npm install -D drizzle-docs-generator
# pnpm の場合
$ pnpm add -D drizzle-docs-generator
# global install の場合
$ npm install -g drizzle-docs-generator
```

このツールでは、前述の通り、Markdown 形式のドキュメント または DBML (Database Markup Language) 形式のテーブル定義を生成できます。

それぞれの機能について紹介します。

### Markdown 形式のドキュメント生成

次のようにコマンドを実行することで Markdown 形式のドキュメントを生成できます。

```bash
$ drizzle-docs generate ./src/db/schema.ts -d mysql -f markdown -o ./docs
```

[schema.ts](https://github.com/rikeda71/drizzle-docs-generator/blob/main/examples/mysql/schema.ts) を対象にコマンドを実行した場合、次のようなディレクトリが生成されます。

https://github.com/rikeda71/drizzle-docs-generator/tree/main/examples/mysql/markdown

コマンドで指定したパスにドキュメント用のディレクトリを作成し、`README.md` にテーブル一覧および ER 図が記載されます。
ER 図は mermaid 形式で記載されるため、GitHub などで閲覧可能です。
また、同一ディレクトリ上にテーブルごとのドキュメントである `<tablename>.md` が生成されます。
このドキュメントにはカラム一覧、制約、インデックス、drizzle で設定した relation に基づく他テーブルとの関係が記されます。

[schema.ts](https://github.com/rikeda71/drizzle-docs-generator/blob/main/examples/mysql/schema.ts) のように JSDoc をスキーマ定義を書くことで、JSDoc に記載したコメントをドキュメントに記載する仕組みとなっています。

```ts:schema.ts
/** User accounts table storing basic user information */
export const users = mysqlTable(
  "users",
  {
    /** Auto-generated unique identifier */
    id: serial("id").primaryKey(),
    /** User's display name */
    name: varchar("name", { length: 100 }).notNull(),
    /** Email address (must be unique) */
    email: varchar("email", { length: 255 }).unique().notNull(),
    /** Whether the user account is active */
    active: boolean("active").default(true),
    /** Timestamp when the user was created */
    createdAt: timestamp("created_at").defaultNow(),
  },
  (table) => [index("users_email_idx").on(table.email)],
);
```

MySQL, PostgreSQL, SQLite 形式のスキーマ定義に対応しており、`-d` オプションで利用している DB のスキーマ定義を指定できます。

```bash
# mysql
$ drizzle-docs generate ./src/db/schema.ts -d mysql -f markdown -o ./docs
# postgresql
$ drizzle-docs generate ./src/db/schema.ts -d postgresql -f markdown -o ./docs
# sqlite
$ drizzle-docs generate ./src/db/schema.ts -d sqlite -f markdown -o ./docs
```

単一ディレクトリに複数ファイルでスキーマ定義をまとめている場合、スキーマ定義があるディレクトリを指定することで利用できます。

```bash
# src/db/schema 配下にスキーマ定義をまとめている場合
$ drizzle-docs generate ./src/db/schema/ -d mysql -f markdown -o ./docs
```

単一ファイルにドキュメントをまとめたい場合、 `--single-file` + `-o <filepath>` を利用します。

```bash
$ drizzle-docs generate ./src/db/schema.ts -d mysql -f markdown --single-file -o doc.md
```

非常にテーブル定義が多い場合に ER 図はノイズになり得るため、`--no-er-diagram` オプションを使って ER 図を Markdown 出力から除外できます。

### DBML 形式のテーブル定義

DBML はテーブル定義を記述するための DSL です。
[dbdiagram.io](https://dbdiagram.io/) や [dbdocs.io](https://dbdocs.io/) を使うことで、DBML 形式のファイルから ER 図やテーブル定義を閲覧しやすい形で確認できます。

次のようにコマンドを実行することで DBML 形式のテーブル定義を生成できます。

```bash
$ drizzle-docs generate ./src/db/schema.ts -d mysql -f dbml -o schema.dbml
```

[schema.ts](https://github.com/rikeda71/drizzle-docs-generator/blob/main/examples/mysql/schema.ts) を対象にコマンドを実行した場合、次のようなファイルが生成されます。

https://github.com/rikeda71/drizzle-docs-generator/blob/main/examples/mysql/schema.dbml

この DBML を dbdiagram.io にインポートすると、ER 図とテーブル定義を視覚的に確認できます。

![dbdiagram.io での表示例](/images/drizzle_docs_generator/1.gif)

Markdown 形式と同じく MySQL, PostgreSQL, SQLite のスキーマ定義 及び 複数ファイルによるスキーマ定義に対応しています。

### v1 対応について

drizzle は v1.0 がβリリースされています。
v1.0 では relation 定義の方法が v0 から変更されています。

https://orm.drizzle.team/docs/relations-v1-v2


drizzle-docs-generator ではどちらの relation 定義にも対応しています。
そのため、どちらのバージョンを利用されていてもご利用いただけます。

## 実装について

次のようにスキーマファイルから情報を抽出しています。

- TypeScript Compiler API を利用して、スキーマファイルを AST に変換
- drizzle のテーブル定義に利用される `mysqlTable()` などの関数呼び出しを検出
- 上記の関数呼び出しの第一引数からテーブル名を、第二引数のオブジェクトからカラム名やカラムの設定に関するメソッドチェーンを抽出
- テーブル定義の関数呼び出し及びその引数の JSDoc をそれぞれテーブル及びカラムのドキュメントとする
- relations 関数内の `one()`, `many()` 呼び出しからテーブル間の関係を抽出

その後、これらの情報を整形して Markdown ファイル または DBML ファイルを生成しています。

## 終わりに

drizzle のスキーマ定義からドキュメントを生成できる drizzle-docs-generator を紹介しました。
drizzle を利用されていて、スキーマ定義とドキュメントの同期にお困りの方はぜひご利用ください！

star や PR, Issue などもよければお願いします。
