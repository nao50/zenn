---
title: "HonoとDrizzleとPostgreSQL"
emoji: "🔥"
type: "tech"
topics: ["typescript", "hono", "drizzleorm", "postgresql"]
published: false
---

# はじめに

```sh
$ npm create hono@latest
Need to install the following packages:
create-hono@0.14.3
Ok to proceed? (y) 

> npx
> create-hono

create-hono version 0.14.3
? Target directory zenn-hono-drizzle
? Which template do you want to use? nodejs
? Do you want to install project dependencies? yes
? Which package manager do you want to use? npm
✔ Cloning the template
✔ Installing project dependencies
🎉 Copied project files
Get started with: cd zenn-hono-drizzle
```


npm install drizzle-orm pg dotenv
npm i -D drizzle-kit @types/pg

npm i zod
npm i @hono/zod-validator
npm i drizzle-zod
npm i @hono/zod-openapi


---
$ docker compose up -d
npx drizzle-kit push




{
  data: { name: 'nao4', age: 43 },
  success: false,
  error: ZodError: [
    {
      "code": "invalid_type",
      "expected": "string",
      "received": "undefined",
      "path": [
        "email"
      ],
      "message": "Required"
    },
    {
      "code": "invalid_type",
      "expected": "string",
      "received": "undefined",
      "path": [
        "password"
      ],
      "message": "Required"
    }
  ]
}