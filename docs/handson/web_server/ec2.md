# EC2 インスタンスで アプリケーションサーバを起動する

## EC2 インスタンスで アプリケーションサーバをセットアップする

1. EC2 インスタンスのセキュリティグループを作成する  
以下の設定でセキュリティグループを作成してください。
    - Security group name: `app-server`
    - Description: `For Application Server`
    - VPC: default
    - Inbound rule1:
      - Type: SSH
      - Source: Anywhere-IPv4
    - Inbound rule2:
      - Type: Custom TCP
      - Port range: 3000
      - Source: Anywhere-IPv4

      __※ Outbound rules は変更しないこと！__

2. EC2 インスタンスを起動する  
以下の設定で EC2 インスタンスを作成してください。
    - Name: `app-server`
    - AMI: Amazon Linux 2023 AMI
    - Instance type: t2.medium
    - Key pair: vockey
    - VPC: default
    - サブネット: default VPC のサブネットであればどれでもOK
    - セキュリティグループ: 作成したセキュリティグループ

3. 作成した EC2 インスタンスに Node.js をセットアップする  
以下のページを参考にして Node.js をセットアップしてください。

    - [EC2 インスタンスで `node.js` をセットアップする](https://docs.aws.amazon.com/ja_jp/sdk-for-javascript/v2/developer-guide/setting-up-node-on-ec2-instance.html)

- EC2 には SSH でログインして作業を行ってください。
- SSH で使用する秘密鍵は、AWS Academy の AWS Defails からダウンロードしてください。

4. 作成した EC2 インスタンスに アプリケーションサーバを構築する  
[hono の公式ページ](https://hono.dev/docs/getting-started/nodejs)
    - 以下のコマンドを実行して `hono` をセットアップします。
        ```
        npm create hono@latest my-app
        ```

    - 以下のコマンドを実行して アプリケーションサーバを起動します。
        ```
        npm run dev
        ```

    - Web ブラウザで EC2 インスタンスの 3000 ポートにアクセスします。  
    Web ブラウザに `{"message":"Hello Hono!"}` と表示されれば成功です。

## ロードバランサをセットアップする
アプリケーションサーバにインターネットからの通信をルーティングするアプリケーションロードバランサ(ALB)をセットアップします。

1. ALB のセキュリティグループを作成する
    - Security group name: `alb`
    - Description: `For ALB`
    - VPC: default
    - Inbound rule1:
      - Type: HTTP
      - Source: Anywhere-IPv4

2. セキュリティグループ(`app-server`) の Inbound rule を修正する  
    - 3000 ポートの Inbound rule を削除
    - Inbound rule を追加
        - Type: Custom TCP
        - Port range: 3000
        - Source: ALB のセキュリティグループ

3. ALB を作成する

    Load Balancers から ALBを作成します。
    - Load balancer name: `app-alb`
    - Scheme: internet-facing
    - Load balancer IP address type: IPv4
    - VPC: default
    - Availability Zones: どれでも良いので 2つ選択する
    - Security group: 作成した ALB のセキュリティグループ  
      __default のセキュリティグループは削除すること__

    Target groups からターゲットグループを作成します。
    - Target type: Instance
    - Target group name: `app-tg`
    - Protocol: HTTP
    - Port: 3000
    - その他の設定はデフォルトのままで [Next] をクリックする
    - Register targets アプリケーションサーバの EC2 インスタンスを選択して [Include as pending below] をクリックする
    - [Create target group] をクリックする

    Health status が `healthy` になるまで待ちます。

## 動作確認する
- Web ブラウザからアプリケーションサーバに直接アクセスできないことを確認する

    アプリケーションサーバの 3000 ポートに Web ブラウザからアクセスする  
    Web ブラウザにエラーが表示されれば成功です。

    以下はChrome の場合のエラーメッセージの例です。  
     `This site can’t be reached`

- Web ブラウザから ALB を経由してアプリケーションサーバにアクセスできることを確認する

    ALB の DNS 名を Web ブラウザに入力してアクセスする  
    Web ブラウザに `{"message":"Hello Hono!"}` と表示されれば成功です。