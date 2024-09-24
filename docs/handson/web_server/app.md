# アプリケーションサーバを作成する
[Hono Web application framework](https://hono.dev/) を使用してアプリケーションサーバを作成します。

1. 以下のコマンドを実行して `hono` をセットアップします。  
```bash
npm create hono@latest my-app
```

以下の設定でプロジェクトを作成する。
```bash
? Which template do you want to use? nodejs
? Do you want to install project dependencies? yes
? Which package manager do you want to use? npm
```

2. index.ts を編集する  
src/index.ts を以下のように編集します。

```typescript
import { serve } from '@hono/node-server'
import { Hono } from 'hono'
import { logger } from 'hono/logger'
const app = new Hono()
// ロガーを使用してログを出力する
app.use(logger())
app.get('/', (c) => {
  // json のレスポンスを返す
  return c.json({
    'message': 'Hello Hono!'
  })
})
const port = 3000
console.log(`Server is running on port ${port}`)
serve({
  fetch: app.fetch,
  port
})
```

3. アプリケーションサーバの動作を確認する  
以下のコマンドを実行してサーバを起動します。

```bash
npm run dev
```

curl コマンドでリクエストを送信してレスポンスを確認する。  
以下のレスポンスが帰ってくれば成功。
```
{
  "message": "Hello Hono!"
}
```

4. アプリケーションサーバを終了する  
`CTL + C` を押してアプリケーションサーバを終了します。
