---
title: "HonoとDrizzleでDB~API~Frontendと型を繋げる"
emoji: "🔥"
type: "tech"
topics: ["hono", "typescript", "javascript", "drizzleorm", "postgresql"]
published: true
---

# はじめに
[Hono](https://hono.dev/)と[drizzle](https://orm.drizzle.team/)を使って、DBのテーブル定義を「API ServerのScheme」と「Frontendの型」まで伝搬させる方法をまとめます。

また、[@hono/zod-openapi](https://github.com/honojs/middleware/tree/main/packages/zod-openapi)を使ったバリデーションとOpenAPIの整備を合わせて行います。

![hono with drizzle](/images/hono-drizzle-todo01.png)


# Honoのプロジェクト作成
まずはHonoのプロジェクトを作成します。  
今回は `zenn-hono-drizzle-todo` という名前のプロジェクトとしました。  

```sh
$ npm create hono@latest

create-hono version 0.16.0
✔ Target directory zenn-hono-drizzle-todo
✔ Which template do you want to use? nodejs
✔ Do you want to install project dependencies? Yes
✔ Which package manager do you want to use? npm
✔ Cloning the template
✔ Installing project dependencies
🎉 Copied project files

$ tree -I 'node_modules'
.
├── README.md
├── compose.yaml
├── package-lock.json
├── package.json
├── src
│   └── index.ts
└── tsconfig.json
```

# 開発用DBの準備
今回はPostgreSQLを使います。  
Docker Composeを使って開発環境用のDBを立ち上げます。  

```yaml:compose.yaml
version: "3.9"
services:
  postgres:
    image: postgres:17.2
    environment:
      - POSTGRES_USER=myuser
      - POSTGRES_PASSWORD=mypassword
    volumes:
      - ./db/data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
```

```sh
$ mkdir db
$ docker compose up -d
```

# Drizzleの設定
drizzleの設定をします。  
まずは必要なパッケージのインストールします。  

```sh
npm i drizzle-orm pg drizzle-zod
npm i -D drizzle-kit @types/pg
```

drizzleの設定ファイルを作成します。  
規模が大きくなると[schemaファイルを分割](https://orm.drizzle.team/docs/sql-schema-declaration)できるので便利です。  

```ts:drizzle.config.ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  out: './drizzle',
  schema: './src/schema.ts',
  dialect: 'postgresql',
  dbCredentials: {
    url: 'postgres://myuser:mypassword@localhost:5432/',
  },
});
```

次のテーブル定義を行います。  
`table.ts`を作成し、以下のようなTODOを管理するテーブルを考えます。  

:::message
あまり現実的な設計ではないですが、勉強のため`todo`にユニーク制約を設定しています
:::

```ts:table.ts
import { pgTable, uuid, varchar, timestamp, boolean  } from "drizzle-orm/pg-core";

export const todoTable = pgTable("todo", {
  id: uuid('id').defaultRandom().primaryKey(),
  todo: varchar({ length: 255 }).notNull().unique(),
  done: boolean().notNull().default(false),
  createdAt: timestamp("created_at").defaultNow().notNull(),
  updatedAt: timestamp("updated_at").defaultNow().notNull(),
});
```

作成したテーブル定義は以下のコマンドで簡単にDBへ反映できます。  
商用環境などには[ドキュメント](https://orm.drizzle.team/docs/kit-overview)のように段階的にmigrationすることになるでしょう。  

```sh
$ npx drizzle-kit push
```

# API ServerのSchema定義
DBのテーブル定義ができたので、このテーブル情報を極力活かしてAPI ServerのSchemaを定義していきます。  

必要なパッケージをインストールします。  

```sh
npm i zod @hono/zod-openapi
```

`schema.ts`というファイルを作成しSchemaを定義します。  
テーブル定義した `table.ts` をインポートし必要のないパラメータをomitします。  
Error Schemaは`hono/zod-openapi`から定義します。  

:::details schema.ts
```ts:schema.ts
import { createSchemaFactory } from 'drizzle-zod';
import { z } from '@hono/zod-openapi';

import { todoTable } from './table.js';

const { createInsertSchema, createSelectSchema, createUpdateSchema } = createSchemaFactory({ zodInstance: z });

////////////////////////////////////////////////////////////////
// Create
export const createTodoReqBodySchema = createInsertSchema(todoTable, {
  todo: (schema) => schema.max(255, "'todo' must be 255 characters or less").openapi({ example: 'todo' }),
}).omit({
  id: true,
  done: true,
  createdAt: true,
  updatedAt: true,
}).required();

export const createTodoResBodySchema = createInsertSchema(todoTable, {
  id: (schema) => schema.openapi({ example: 'a415d538-88d5-4db4-8da2-628826a15b9f' }),
  todo: (schema) => schema.openapi({ example: 'todo' }),
  done: (schema) => schema.openapi({ example: false }),
  createdAt: (schema) => schema.openapi({ example: '2023-10-01T00:00:00.000Z' }),
  updatedAt: (schema) => schema.openapi({ example: '2023-10-01T00:00:00.000Z' }),
});

////////////////////////////////////////////////////////////////
// Read
export const readTodoReqPathParamSchema = createSelectSchema(todoTable, {
  id: (schema) => schema.openapi({ example: 'a415d538-88d5-4db4-8da2-628826a15b9f' }),
}).omit({
  todo: true,
  done: true,
  createdAt: true,
  updatedAt: true,
}).required();

export const readTodoResBodySchema = createSelectSchema(todoTable, {
  id: (schema) => schema.openapi({ example: 'a415d538-88d5-4db4-8da2-628826a15b9f' }),
  todo: (schema) => schema.openapi({ example: 'todo' }),
  done: (schema) => schema.openapi({ example: false }),
  createdAt: (schema) => schema.openapi({ example: '2023-10-01T00:00:00.000Z' }),
  updatedAt: (schema) => schema.openapi({ example: '2023-10-01T00:00:00.000Z' }),
});

export const readTodoListResBodySchema = createSelectSchema(todoTable, {
  id: (schema) => schema.openapi({ example: 'a415d538-88d5-4db4-8da2-628826a15b9f' }),
  todo: (schema) => schema.openapi({ example: 'todo' }),
  done: (schema) => schema.openapi({ example: false }),
  createdAt: (schema) => schema.openapi({ example: '2023-10-01T00:00:00.000Z' }),
  updatedAt: (schema) => schema.openapi({ example: '2023-10-01T00:00:00.000Z' }),
}).array();

////////////////////////////////////////////////////////////////
// Update
export const updateTodoReqPathParamSchema = createUpdateSchema(todoTable, {
  id: (schema) => schema.openapi({ example: 'a415d538-88d5-4db4-8da2-628826a15b9f' }),
}).omit({
  todo: true,
  done: true,
  createdAt: true,
  updatedAt: true,
}).required();

export const updateTodoReqBodySchema = createUpdateSchema(todoTable, {
  todo: (schema) => schema.openapi({ example: 'todo' }),
  done: (schema) => schema.openapi({ example: false }),
}).omit({
  id: true,
  createdAt: true,
  updatedAt: true,
});

export const updateTodoResBodySchema = createUpdateSchema(todoTable, {
  id: (schema) => schema.openapi({ example: 'a415d538-88d5-4db4-8da2-628826a15b9f' }),
  todo: (schema) => schema.openapi({ example: 'todo' }),
  done: (schema) => schema.openapi({ example: false }),
  createdAt: (schema) => schema.openapi({ example: '2023-10-01T00:00:00.000Z' }),
  updatedAt: (schema) => schema.openapi({ example: '2023-10-01T00:00:00.000Z' }),
});

////////////////////////////////////////////////////////////////
// Delete
export const deleteTodoReqPathParamSchema = createSelectSchema(todoTable, {
  id: (schema) => schema.openapi({ example: 'a415d538-88d5-4db4-8da2-628826a15b9f' }),
}).omit({
  todo: true,
  done: true,
  createdAt: true,
  updatedAt: true,
}).required();

////////////////////////////////////////////////////////////////
// Error
export const errorResBodySchema = z.object({
  id: z.string().openapi({
    example: 'a415d538-88d5-4db4-8da2-628826a15b9f',
  }),
  code: z.number().openapi({
    example: 400,
  }),
  message: z.string().openapi({
    example: "Bad Request",
  }),
});
```

:::

これでテーブル定義からAPI ServerのSchemaを生成することができました。  

# ルーティング設定
次にルーティングの設定を行います。
OpenAPI仕様に則り、どのパスに対し、どういうレスポンスが生じ、それぞれどのスキーマを受け取り、送り返すか、を定義します。  

:::details route.ts

```ts:route.ts
import { createRoute } from '@hono/zod-openapi'
import { 
  createTodoReqBodySchema,
  createTodoResBodySchema,
  readTodoReqPathParamSchema,
  readTodoResBodySchema,
  readTodoListResBodySchema,
  updateTodoReqPathParamSchema,
  updateTodoReqBodySchema,
  updateTodoResBodySchema,
  deleteTodoReqPathParamSchema,
  errorResBodySchema
} from './schema.js';

////////////////////////////////////////////////////////////////
// Create
export const createTodoRoute = createRoute({
  method: "post",
  path: "/",
  request: {
    body: {
      content: {
        'application/json': {
          schema: createTodoReqBodySchema,
        },
      },
    },
  },
  responses: {
    200: {
      description: 'OK',
      content: {
        'application/json': {
          schema: createTodoResBodySchema,
        },
      },
    },
    400: {
      content: {
        "application/json": {
          schema: errorResBodySchema,
        },
      },
      description: "Bad Request",
    },
    409: {
      content: {
        "application/json": {
          schema: errorResBodySchema,
        },
      },
      description: "Conflict",
    }
  },
});


////////////////////////////////////////////////////////////////
// Read
export const readTodoRoute = createRoute({
  method: 'get',
  path: "/{id}",
  request: {
    params: readTodoReqPathParamSchema,
  },
  responses: {
    200: {
      content: {
        'application/json': {
          schema: readTodoResBodySchema,
        },
      },
      description: 'Retrieve the user',
    },
    404: {
      content: {
        "application/json": {
          schema: errorResBodySchema,
        },
      },
      description: "Not Found",
    },
  },
});

export const readTodoListRoute = createRoute({
  method: 'get',
  path: "/",
  responses: {
    200: {
      content: {
        'application/json': {
          schema: readTodoListResBodySchema,
        },
      },
      description: 'Retrieve the user',
    }
  },
});

////////////////////////////////////////////////////////////////
// Update
export const updateTodoRoute = createRoute({
  method: "put",
  path: "/{id}",
  request: {
    params: updateTodoReqPathParamSchema,
    body: {
      content: {
        'application/json': {
          schema: updateTodoReqBodySchema,
        },
      },
    },
  },
  responses: {
    200: {
      content: {
        'application/json': {
          schema: updateTodoResBodySchema,
        },
      },
      description: 'Retrieve the user',
    },
    400: {
      content: {
        "application/json": {
          schema: errorResBodySchema,
        },
      },
      description: "Bad Request",
    },
    404: {
      content: {
        "application/json": {
          schema: errorResBodySchema,
        },
      },
      description: "Not Found",
    },
  },
});

////////////////////////////////////////////////////////////////
// Delete
export const deleteTodoRoute = createRoute({
  method: "delete",
  path: "/{id}",
  request: {
    params: deleteTodoReqPathParamSchema,
  },
  responses: {
    204: { 
      description: 'Successfully deleted'
    },
    404: {
      content: {
        "application/json": {
          schema: errorResBodySchema,
        },
      },
      description: "Not Found",
    },
  }
});
```
:::

# HonoのAPIの実装プラクティス
APIの実装を行なっていきます。  
Honoの[Best Practices](https://hono.dev/docs/guides/best-practices#building-a-larger-application)にある通り、APIの実装を別ファイルに分割し `index.ts` に集約する方法を取ります。  

https://hono.dev/docs/guides/best-practices#building-a-larger-application

DBの変数定義やrequestIdの設定なども `index.ts` に実装します。  

```ts:index.ts
import { serve } from '@hono/node-server'
import { OpenAPIHono } from "@hono/zod-openapi";
import { requestId } from 'hono/request-id'
import { NodePgDatabase, drizzle } from 'drizzle-orm/node-postgres';
import api from "./api.js"; // api.ts でAPI実装をこれから行う

export type Variables = {
  db: NodePgDatabase
}

const app = new OpenAPIHono<{ Variables: Variables }>()
  .use(requestId())
  .use(logger())
  .use(async (c, next) => {
    const db = drizzle('postgres://myuser:mypassword@localhost:5432/');
    c.set('db', db)
    await next()
  })
  .route('/todo', api) // 各ルートをここで集約

serve({
  fetch: app.fetch,
  port: 3000
}, (info) => {
  console.log(`Server is running on http://localhost:${info.port}`)
})

export default app
export type AppType = typeof app
```

## defaultHookによるエラーハンドリング
HonoにはZodのエラー（つまりバリデーションエラー）をまとめてハンドリングできる [defaultHook](https://github.com/honojs/middleware/tree/main/packages/zod-openapi#a-dry-approach-to-handling-validation-errors) という仕組みがあります。  

この仕組みを使うとパスごとに同じバリデーションエラーのハンドリングを書かずに済むため記述量が減るため、積極的に活用していきます。  

https://github.com/honojs/middleware/tree/main/packages/zod-openapi#a-dry-approach-to-handling-validation-errors


## API実装
それではAPIを実装します。  
実装としてはバリデーションエラーを`defaultHook`でハンドリングしています。  
また、今回 `todo` にユニーク制約を入れました。ユニーク制約のエラーは現状`db.insert()`の返り値として返却されないため `try catch` でエラーを拾います。  

このissueで議論されていますが返り値でDBのエラーも返してほしい気持ちになりますね。  

https://github.com/drizzle-team/drizzle-orm/issues/376

:::details api.ts

```ts:api.ts
import { OpenAPIHono } from "@hono/zod-openapi";
import { NodePgDatabase } from 'drizzle-orm/node-postgres';
import { createTodoRoute, readTodoRoute, readTodoListRoute, updateTodoRoute, deleteTodoRoute  } from "./route.js";
import { todoTable } from './table.js';
import pg from "pg";
import { eq } from "drizzle-orm";

type Variables = {
  db: NodePgDatabase
}

const app = new OpenAPIHono<{ Variables: Variables }>({
  defaultHook: (result, c) => {
    if (!result.success) {
      return c.json({
        id: c.get('requestId'),
        code: 400,
        message: result.error.errors.map(error => error.message).join(', '),
      }, 400);
    }
  },
}).openapi(createTodoRoute, async (c) => {
  const data = c.req.valid('json')
  const db = c.get('db')
  try {
    const result = await db.insert(todoTable).values({...data, done: false}).returning();
    return c.json(result[0], 200);
  } catch (error) {
    if (error instanceof pg.DatabaseError) {
      if(error.constraint === "table_todo_unique") {
        return c.json({id: c.get('requestId'), code: 409, message: "Todo already exists"}, 409);
      }
    }
    return c.json({id: c.get('requestId'), code: 400, message: error}, 400)
  }
}).openapi(readTodoRoute, async (c) => {
  const { id } = c.req.valid('param')
  const db = c.get('db')
  const result = await db.select().from(todoTable).where(eq(todoTable.id, id));
  if (result.length === 0) {
    return c.json({id: c.get('requestId'), code: 404, message: 'Not Found'}, 404)
  }
  return c.json(result[0], 200);
}).openapi(readTodoListRoute, async (c) => {
  const db = c.get('db')
  const result = await db.select().from(todoTable);
  return c.json(result, 200);
}).openapi(updateTodoRoute, async (c) => {
  const { id } = c.req.valid('param')
  const data = c.req.valid('json')
  const db = c.get('db')
  const result = await db.update(todoTable).set(data).where(eq(todoTable.id, id)).returning();
  if (result.length === 0) {
    return c.json({id: c.get('requestId'), code: 404, message: 'Not Found'}, 404)
  }
  return c.json(result[0], 200);
}).openapi(deleteTodoRoute, async (c) => {
  const { id } = c.req.valid('param')
  const db = c.get('db')
  const result = await db.delete(todoTable).where(eq(todoTable.id, id)).returning();
  if (result.length === 0) {
    return c.json({id: c.get('requestId'), code: 404, message: 'Not Found'}, 404)
  }
  return c.text('Successfully deleted')
});

export default app
```
:::

# RPCによるクライアント
DBのテーブル情報からAPI Schemaまで型情報を伝搬してきました。  
次はフロントエンドです。  

Honoには[RPC](https://hono.dev/docs/guides/rpc)という仕組みが準備されています。  

今回は以下のスクリプトから`hc<AppType>`とルーティング情報、型情報をimportしAPIの型情報付きでAPI Callすることができます。  

```ts
import { hc } from 'hono/client'
import type { AppType } from './index.js'

const client = hc<AppType>('http://localhost:3003')

const fetchApi = async () => {
  const postRes = await client.todo.$post({
    json: {
      todo: 'test todo 01',
    },
  })
  const postData = await postRes.json()

  const getRes = await client.todo.$get({})
  const getData = await getRes.json()
  return getData
}

fetchApi()
  .then((data) => {
    for(const todo of data) {
      console.log(`ID: ${todo.id}, Todo: ${todo.todo}`)
    }
  })
  .catch((error) => {
    console.error(`Error fetching data: ${error}`)
  });  
```

# まとめ
[Hono](https://hono.dev/)と[drizzle](https://orm.drizzle.team/)を使って、DBのテーブル定義を「API ServerのScheme」と「Frontendの型」まで伝搬させる方法をまとめました。  

https://github.com/nao50/zenn-hono-drizzle-todo