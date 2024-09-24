# ECS でアプリケーションサーバを起動する

## アプリケーションサーバのコンテナイメージを作成する
Cloud 9 でアプリケーションサーバのコンテナイメージを作成します。

1. アプリケーションサーバを作成する  
[アプリケーションサーバを作成する](./app.md) を参照してアプリケーションサーバを作成します。

2. コンテナイメージを作成する  
以下の内容で `Dockerfile` を作成します。

```dockerfile
FROM node:20-slim
WORKDIR /opt/app    
COPY package.json tsconfig.json ./
COPY src/ ./src/
RUN yarn
CMD ["npm", "run", "dev"]
```

3. コンテナイメージをビルドする  
以下のコマンドを実行してコンテナイメージをビルドします。

```bash
docker build -t myapp .
```

4. コンテナを起動する  
以下のコマンドを実行してコンテナを起動します。

```bash
docker run --name myapp -p 3000:3000 -d --rm myapp
```
curl コマンドでリクエストを送信してレスポンスを確認する。  
以下のレスポンスが帰ってくれば成功。  
```json
{
  "message": "Hello Hono!"
}
```

5. コンテナを終了する
以下のコマンドを実行してコンテナを終了します。

```bash
docker stop myapp
```

## コンテナイメージを Elastic Container Registry に登録する
1. リポジトリを作成する  
Elasitc Container Registry(ECR) サービスに移動してコンテナイメージを登録するリポジトリを作成します。
    - Repository name: `myapp`
    - [Create] をクリックする

2. コンテナイメージをプッシュする  
作成したリポジトリの詳細画面で [View push commands] をクリックして表示される手順でコンテナイメージをリポジトリにプッシュします。

## Elastic Container Service でコンテナを起動する
Elastic Container Service(ECS) サービスに移動してコンテナを起動します。

1. ECS クラスタを作成する  
[Create cluster] をクリックして以下の内容でクラスタを作成します。
    - Cluster name: `myapp`
    - Infrastructure: AWS Fargate (serverless)
    - [Create] をクリックする

2. Task Definition を作成する  
左のナビゲーションペインから `Task definitions` を選択して [Create new Task Definition] をクリックして以下の内容で Task Definition を作成します。
    - Task definition configuration
        - Task definition family: `myapp-def`
    - Infrastructure requirements
        - Launch type: AWS Fargate
        - Task role: LabRole
        - Task execution role: LabRole
    - Container - 1
        - Name: `myapp`
        - Image URI: ECR に登録したコンテナイメージの URI
        - Container port: 3000
    - [Create] をクリックする

3. Service を作成する
左のナビゲーションペインから `Clusters` を選択して作成したクラスタを選択して Service タブから [Create] をクリックして以下の内容で Service を作成します。
    - Deployment configuration
        - Family: `myapp-def`
        - Service name: `myapp-svc`
    - Networking
        - Security group: app  
          __default のセキュリティグループは削除する__
    - Load balancing
        - Load balancer type: Application Load Balancer
        - Application Load Balancer: Use an existing load balancer
        - Load balancer: `alb`
        - Listener: Create new listener
            - Port: 81
            - Protocol: HTTP

    サービスが起動するまで待ちます。

## 動作確認する
Web ブラウザから ALB の 81 番ポートを経由してアプリケーションサーバにアクセスできることを確認する

ALB の DNS 名を Web ブラウザに入力してアクセスする  
例：`http://alb-268898749.us-east-1.elb.amazonaws.com:81`

Web ブラウザに `{"message":"Hello Hono!"}` と表示されれば成功です。

### 課題
ここまでの手順だけではアプリケーションサーバにアクセスできません。  
どこに問題があるのかを確認して修正してください。
