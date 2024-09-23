# Lambda で Web API を起動する

## アプリケーションを作成する
[Hono Web application framework](https://hono.dev/) を使用して Lambda で実行する Web API を作成します。

1. 以下のコマンドを実行して `hono` をセットアップします。  

    ```bash
    npm create hono@latest my-lambda
    cd my-lambda
    npm i
    ```

    以下の設定でプロジェクトを作成する。
    ```bash
    ? Which template do you want to use? aws-lambda
    ? Do you want to install project dependencies? yes
    ? Which package manager do you want to use? npm
    ```

2. index.ts を編集する  
src/index.ts を以下のように編集します。

    ```typescript
    import { Hono } from 'hono'
    import { handle } from 'hono/aws-lambda'
    import { logger } from 'hono/logger'

    const app = new Hono()

    // ロガーを使用してログを出力する
    app.use(logger())

    app.get('/', (c) => {
      return c.json({
          'message': 'Hello Hono!'
      })
    })

    export const handler = handle(app)
    ```

3. ビルドする  
以下のコマンドを実行して Web API をビルドします。

    ```bash
    npm run build
    npm zip
    ```

    `lambda.zip` が作成されます。

## Lambda 関数を作成する
Lambda サービスに移動して Lambda 関数を作成します。

1. 左のナビゲーションメニューから [Functions] を選択し、以下の
    - Author from scratch
    - Basic information
      - Function name: `myLambda`
      - Runtime: `Node.js 20.x`
      - Architecture: `x86_64`
      - Change default execution role
        - `Use an existing role`
        - Existing role: `LabRole`
    - [Create function] をクリックする
2. Upload from から 作成した Lambda の zip ファイルをアップロードする  

    __[注意] アップロードする Lambda の zip ファイルは Cloud9 上に作成されています。__

3. Lambda 関数の動作を確認する
    Lambda 関数の画面から [Test] -> [Configure test event] をクリックして、テストイベントを作成します。

    参考: [Process Application Load Balancer requests with Lambda](https://docs.aws.amazon.com/lambda/latest/dg/services-alb.html)    

    - Event name: `test`
    - Template: Application Load Balancer(alb-request)
      - httpMethod: "GET"
      - path: "/"
      - body: ""
    - [Invoke] をクリックする

    以下のようなレスポンスが返ってくれば成功です。

    ```
    Test Event Name
    myEvent

    Response
    {
      "body": "{\"message\":\"Hello Hono!\"}",
      "headers": {
        "content-type": "application/json; charset=UTF-8"
      },
      "statusCode": 200,
      "isBase64Encoded": false
    }

    Function Logs
    2024-09-22T05:34:40.639Z	14cd3316-d9ff-45ff-a1e6-4cca9275c325	    INFO	<-- GET /
    START RequestId: 14cd3316-d9ff-45ff-a1e6-4cca9275c325 Version:    $LATEST
    2024-09-22T05:34:40.640Z	14cd3316-d9ff-45ff-a1e6-4cca9275c325	    INFO	--> GET / [32m200[0m 1ms
    END RequestId: 14cd3316-d9ff-45ff-a1e6-4cca9275c325
    REPORT RequestId: 14cd3316-d9ff-45ff-a1e6-4cca9275c325	Duration: 7.    48 ms	Billed Duration: 8 ms	Memory Size: 128 MB	Max Memory Used: 75     MB

    Request ID
    14cd3316-d9ff-45ff-a1e6-4cca9275c325
    ```

## ロードバランサをセットアップする
ALB から Lambda へリクエストを転送するように設定します。

1. ターゲットグループを作成する
  - Basic configuration
    - Choose target type: `Lambda function`
    - Target group name: `myLambdaTG`
  - Health checks: Enable をチェックする
  - [Next] をクリックする
  - Lambda function:
    - Select a Lambda function: チェック
    - Lambda function: `myLambda`
  - [Create target group] をクリックする

2. ロードバランサーにリスナーを追加する
  - ロードバランサーの詳細画面で [Add listener] をクリックする
  - Listener details
    - Protocol: HTTP
    - Port: 82
    - Routing actions: Forward to target group
      - Target group: `myLambdaTG`
    - [Add] をクリックする

## 動作確認する
Web ブラウザから ALB の 82 番ポートを経由してアプリケーションサーバにアクセスできることを確認する

ALB の DNS 名を Web ブラウザに入力してアクセスする  
例：`http://alb-268898749.us-east-1.elb.amazonaws.com:82`

Web ブラウザに `{"message":"Hello Hono!"}` と表示されれば成功です。

### 課題
ここまでの手順だけではアプリケーションサーバにアクセスできません。  
どこに問題があるのかを確認して修正してください。
