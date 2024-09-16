# いろいろな方法で Web サーバを起動する
AWS が提供している以下のコンピューティング環境を使用して Web API を構築してみる。

- EC2
- ECS
- Lambda

## Cloud9 の環境を用意する
1. Cloud9 を開く
2. [Create environment] をクリックする
3. Name に `dev` と入力する
4. Network settings で Secure Shell (SSH) を選択する
5. [Create] をクリックする

## Cloud9 に yarn をインストールする
以下のコマンドを実行して yarn をインストールする。

```bash
sudo npm install -g yarn
```

## Seucirty Group を用意する
1. VPC サービスを開く
2. 左のナビゲーションメニューから [Security groups] をクリックする
3. [Create security group] をクリックして以下の 
3-1. ALB 用
  - Security group name: `alb`
  - Description: `Application Load Balancer`
  - VPC: default
  - Inbound rules: 
    - Type: HTTP, Source: Anywhere-IPv4
3-2. Web API 用
  - Security group name: `web-api`
  - Description: `Web API`
  - VPC: default
  - Inbound rules: 
    - Type: Custom TCP, Port range: 3000, Source: Anywhere-IPv4

## アプリケーションを作成する
Node.js と Lambda を使用して Web API を作成する。

### Node.js
1. プロジェクトを作成する
```bash
yarn create hono myapp
```

以下の設定でプロジェクトを作成する。
```bash
? Which template do you want to use? nodejs
? Do you want to install project dependencies? yes
? Which package manager do you want to use? yarn
```

2. index.ts を編集する

```typescript
import { serve } from '@hono/node-server'
import { Hono } from 'hono'

const app = new Hono()

app.get('/', (c) => {
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

3. 起動する
以下のコマンドを実行してサーバを起動する。

```bash
yarn dev
```

curl コマンドでリクエストを送信してレスポンスを確認する。

以下のレスポンスが帰ってくれば成功。
```json
{
  "message": "Hello Hono!"
}
```

コンテナイメージを作成する
1. Dockerfile を作成する
```dockerfile
FROM node:20-slim
WORKDIR /opt/app

COPY package.json tsconfig.json ./
COPY src/ ./src/

RUN yarn

CMD ["yarn", "dev"]
```

2. コンテナイメージをビルドする
```bash
docker build -t myapp .
```

3. コンテナを起動する
```bash
docker run --name myapp -p 3000:3000 myapp
```

curl コマンドでリクエストを送信してレスポンスを確認する。

以下のレスポンスが帰ってくれば成功。
```json
{
  "message": "Hello Hono!"
}
```

### Lambda
ALB からリクエストを受け取る Lambda 関数を作成する。
1. ディレクトリを作成する

```bash
mkdir lambda
cd lambda
```

2. 初期化する
```bash
yarn init
yarn add typescript @types/aws-lambda esbuild --dev
yarn run tsc --init
```

3. package.json を編集する
以下の定義を追加する。

```json
"scripts": {
  "prebuild": "rm -rf dist",
  "build": "esbuild index.ts --bundle --minify --sourcemap --platform=node --target=es2020 --outfile=dist/index.js",
  "postbuild": "cd dist && zip -r index.zip index.js*"
}
```

4. index.ts を作成する
index.ts に以下のコードを記述する。

```typescript
import { Handler, ALBEvent, Context } from 'aws-lambda'

export const handler: Handler = async (event: ALBEvent, context: Context) => {
  console.log('Received event:', JSON.stringify(event, null, 2))

  return lambdaHandler(event)
}

function lambdaHandler(event: ALBEvent) {
  return {
    statusCode: 200,
    body: JSON.stringify({
      'statusCode': 200,
      'statusDescription': '200 OK',
      'isBase64Encoded': false,
      'headers': {
        'Content-Type': 'application/json',
      },
      'body': {
        'message': 'Hello from Lambda!'
      }
    }),
  }
}
```

5. ビルドする
以下のコマンドを実行してビルドする。

```bash
yarn build
```

`dist` の下に `index.zip` が作成される。

## デプロイする
### EC2
#### EC2 インスタンスを作成する
1. EC2 サービスを開く
2. [Launch instance] をクリックする
3. Name に `web-api` と入力する
4. Amazon Machine Image (AMI) で Amazon Linux 2023 を選択する
5. Instance type で t2.micro を選択する
6. Key pair (login) で 'vockey' を選択する

### ECS

### Lambda
Lambda 関数を作成する。
1. Lambda サービスを開く
2. 左のナビゲーションメニューから [Functions] をクリックする
3. [Create function] をクリックする
4. Function name に `myLambda` と入力する
5. Runtine で `Node.js 20.x` を選択する
6. Architecture で `x86_64` を選択する
7. Change default execution role を開く
8. Use an existing role を選択する
9. Existing role で `LabRole` を選択する
10. Upload from から 作成した Lambda の zip ファイルをアップロードする

ALB から Lambda へリクエストを転送するように設定する。
1. EC2 サービスを開く
2. 左のナビゲーションメニューから [Target Groups] をクリックする
3. [Create target group] をクリックする
4. Target type で `Lambda function` を選択する
5. Target group name に `myLambdaTG` と入力する
6. Health checks を有効にする
7. [Next] をクリックする
8. Lambda function で `myLambda` を選択する
9. [Create target group] をクリックする
10. Actions から Associate with a new load balancer を選択する

ALB を作成する
1. Load balancer name に `myLb` と入力する
2. Scheme で `internet-facing` を選択する
3. Network mapping で以下を選択する
   - VPC: default
   - Availability Zones: 2 つ選択する
4. Security group で作成した ALB 用の security group を選択する
5. HTTP:80 の Listener の Default action で `myLambdaTG` を選択する
6. [Create load balancer] をクリックする
