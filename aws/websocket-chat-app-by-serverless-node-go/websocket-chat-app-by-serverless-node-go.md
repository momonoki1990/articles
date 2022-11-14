もうすぐ 2 歳になる次男がよく寝てくれるようになったせいか、ひさしぶりに早起きできるようになりました w
エンジニアの熊瀬川です。

AWS のチュートリアルで作成した API Gateway の WebSocket API を使ったチャットアプリを、Serverless Framework で再現してみました。
言語は、普段から使用している Node.js と勉強中の Go の 2 種類使ってみました。

AWS のチュートリアル
[チュートリアル: WebSocket API、Lambda、DynamoDB を使用したサーバーレスチャットアプリケーションの構築](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/websocket-api-chat-app.html)

## つくったもの

リポジトリ
[aws-tutorial-chat-app](https://github.com/momonoki1990/aws-tutorial-chat-app)

元々のチュートリアルのファイル(Cloudformation)
[tutorial-ver](https://github.com/momonoki1990/aws-tutorial-chat-app/tree/main/tutorial-ver)

Node.js x Serverless Framework
[node-ver](https://github.com/momonoki1990/aws-tutorial-chat-app/tree/main/node-ver)

Go x Serverless Framework
[go-ver](https://github.com/momonoki1990/aws-tutorial-chat-app/tree/main/go-ver)

## WebSocket とは

> WebSocket プロトコルは仕様 RFC 6455 で説明されており、これは永続的な接続を介してブラウザとサーバ間でデータを交換する方法を提供します。接続の切断や追加の HTTP リクエストをすることなく、データを “パケット” として双方向に渡すことができます。
> WebSocket は継続的にデータ交換を必要とするようなサービスに特に適しています。例えば、オンラインゲームやリアルタイムの取引システムなどです。
> https://ja.javascript.info/websocket

チャットでのメッセージのやりとりのように、リアルタイムで接続者に対してデータの変更を反映させたいといった場合に使用されます。

`ws://`と`wss://`プロトコルがあって、`wss`(WebSocket over TLS)では HTTPS と同じようにデータが暗号化されるようです。

## アーキテクチャ

[](/images/architecture.png)

([チュートリアル](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/websocket-api-chat-app.html)より転載)

今回はチュートリアルの構成そのままですが、接続・切断のタイミングで接続 ID(`connectionID`)を DynamoDB に反映させて、`sendMessage`イベントが発生した際に、受信したメッセージをサーバーから各接続者に送信しています。

Chrome のネットワークタブで確認したところ、最初の接続確認を行なっている GET リクエストだけ確認できました。

(`101 Switching Protocols`が返ってきている)
[](/images/101_Switching_Protocols.png)

※[Go 言語による Web アプリケーション開発](https://www.amazon.co.jp/Go%E8%A8%80%E8%AA%9E%E3%81%AB%E3%82%88%E3%82%8BWeb%E3%82%A2%E3%83%97%E3%83%AA%E3%82%B1%E3%83%BC%E3%82%B7%E3%83%A7%E3%83%B3%E9%96%8B%E7%99%BA-Mat-Ryer/dp/4873117526)で作成した[チャットアプリ](https://github.com/momonoki1990/go-web-app-orreilly-book/tree/main/chat-app)を動かして確認しました。

## APIGateway でのルーティング

[API Gateway での WebSocket API について](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/apigateway-websocket-api-overview.html)
[WebSocket API のルートの操作](https://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/websocket-api-develop-routes.html#apigateway-websocket-api-route-selection-expressions)

既定のルーティングが 3 つあって、それぞれ接続時(`$connect`)・切断時(`$disconnect`)・どのルーティングにも当てはまらない時(`$default`)に使用されます。

それ以外の独自のルーティングは、リクエストボディに含まれる特定の key(ルート選択式)の値を元に行います。

公式ドキュメントだと、`request.body.action`がルート選択式(REST で言うところのルーティングの path だと思います)に指定されていて、今回のアプリでも同様に、`{"action": "sendmessage", "message: "hello"}`というリクエストボディを受け取ったら、他の接続者に対してメッセージ(`hello`)が転送されるようになっています。

複数の値で判断したり変数を含めることもできるようです。

[](/images/route_expression.png)

## Serverless Framework

[Serverless Framework: Websocket](https://www.serverless.com/framework/docs/providers/aws/events/websocket)

Lambda 関数及び AWS の関連リソースをサクッと作れるフレームワークです。

Lambda を起点としたサーバーレスな構成のアプリケーションを作る際によく使用される印象です。

使ってみたこともないので実際にどのくらい使用されているのかわかりませんが、ドキュメントを見ると、GCP などの AWS 以外のクラウドにも利用できます。

[Serverless Google Cloud Functions Provider Documentation](https://www.serverless.com/framework/docs/providers/google)

> Serverless Framework の開発自体も AWS 向けがメインになっているみたいなので、AWS と同じ様な便利さを想像して使うには現段階では少し厳しいかなと感じました。
> [Google Cloud Functions を Serverless Framework で環境構築してみる](https://dev.classmethod.jp/articles/google-cloud-functions-with-serverless-framework/)

なるほどですね！

## 動作

[](/images/demo_slg_gif.gif)

AWS のドキュメント内でも紹介されていますが、npm ライブラリの`wscat`をクライアントとして使用し、API Gateway に接続しています。

`wscat -c {エンドポイント}`で接続後、`{"action": "sendmessage", "message": "hello"}`を入力して Enter を押すと、メッセージがサーバーに送信され、`sendmessage`ハンドラでメッセージを他の接続者に転送しています。

[wscat](https://github.com/websockets/wscat)のソースコードを見てみると、npm ライブラリの[ws](https://github.com/websockets/ws)を使っているようですね。

## リソースの作成

`serverless.ts`(go バージョンは`.yml`)に DynamoDB や IAM などのリソースを定義しています。

また、Node バージョンと Go バージョンで微妙に違いますが、`connect/`など関数名のディレクトリ以下で Lambda 関数を定義し、API Gateway との紐付けを記載しています(Go は`serverless.yml`に)

IAM(DynamoDB へのアクセス許可)

```ts:serverless.ts
iam: {
      role: {
        statements: [
          {
            Effect: "Allow",
            Action: [
              "dynamodb:Get*",
              "dynamodb:Query",
              "dynamodb:Scan",
              "dynamodb:Delete*",
              "dynamodb:Update*",
              "dynamodb:PutItem",
            ],
            Resource: [{ "Fn::GetAtt": ["connectionsTable", "Arn"] }],
          },
        ],
      },
    },
```

DynamoDB

```ts:serverless.ts
resources: {
    Resources: {
      // Table for recording connectionId
      connectionsTable: {
        Type: "AWS::DynamoDB::Table",
        Properties: {
          TableName: "${self:custom.DB_TABLE_NAME}",
          KeySchema: [
            {
              AttributeName: "connectionId",
              KeyType: "HASH",
            },
          ],
          AttributeDefinitions: [
            { AttributeName: "connectionId", AttributeType: "S" },
          ],
          // NOTE: for on demand capacity, not provisioned
          BillingMode: "PAY_PER_REQUEST",
        },
        UpdateReplacePolicy: "Delete",
        DeletionPolicy: "Delete",
      },
    },
  },
```

Lambda 関数と WebSocket API の紐付け

```ts:src/functions/send-message/index.ts
export default {
  handler: `${handlerPath(__dirname)}/handler.main`,
  events: [
    {
      websocket: {
        route: "sendmessage",
      },
    },
  ],
};
```

デプロイ

```
$ npx sls deploy --stage {app stage e.g. production, dev} --region {region e.g. ap-northeast-1} --aws-profile {your aws profile name}
```

stage や region などは`serverless.ts(yml)`内で設定することもできます。

[Serverless.yml Reference](https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml)
[Serverless Framework: Websocket](https://www.serverless.com/framework/docs/providers/aws/events/websocket)

## APIGatewayMagegementAPI

接続者へのデータ送信は APIGatewayMagegementAPI を使って行います。
DynamoDB の`connections`テーブルから現在接続されているクライアントの接続者 ID を取得し、それとメッセージを APIGatewayMagegementAPI に渡してデータを送信しています。

Node バージョン

```ts:src/functions/send-message/handler.ts
// Send message to all clients
const sendMessages = connections.Items.map(async ({ connectionId }) => {
  // exclude the sender of the message
  if (connectionId === event.requestContext.connectionId) return;
  try {
      await callbackAPI
      .postToConnection({ ConnectionId: connectionId, Data: message })
      .promise();
  } catch (e) {
      console.log(e);
  }
});

try {
  await Promise.all(sendMessages);
} catch (err) {
  return handleError(err);
}
```

Go バージョン

```go:send_message/main.go
// Send message to all clients
var body Body
if err := json.Unmarshal([]byte(req.Body), &body); err != nil {
    return Response{StatusCode: 500}, err
}

managementAPI := apigatewaymanagementapi.New(session, &aws.Config{
    Endpoint: aws.String(req.RequestContext.DomainName + "/" + req.RequestContext.Stage),
})

for _, id := range connectionIDs {
    if id == connectionID {
        continue
    }
    if _, err := managementAPI.PostToConnection(&apigatewaymanagementapi.PostToConnectionInput{ConnectionId: &id, Data: []byte(body.Message)}); err != nil {
        return Response{StatusCode: 500}, err
    }
}
```

aws-cli を使って、ローカルから APIGatewayMagegementAPI 経由で接続者にメッセージを送信することもできます。

```shell
$ aws apigatewaymanagementapi post-to-connection \
--data $(echo '{ "name": "hello" }' | base64) \
--endpoint-url 'https://324v05mks9.execute-api.ap-northeast-1.amazonaws.com/dev' \
--connection-id 'bZYNAdtjNjMCKCw=' \
--profile {AWSプロファイル名}

別ターミナル
$ wscat -c wss://324v05mks9.execute-api.ap-northeast-1.amazonaws.com/dev
Connected (press CTRL+C to quit)
< { "name": "hello" }
```

[AWS API Gateway の WebSocket API をちゃんと理解する](https://zenn.dev/mryhryki/articles/2020-12-01-aws-api-gateway-websocket)
[AWS Lambda 関数の呼び出しが AWS CLI v2 にアップデートすると失敗する](https://dev.classmethod.jp/articles/aws-cli-v2-blob-default-base64/)

## Node バージョン

```md
3rd party libraries

- [json-schema-to-ts](https://github.com/ThomasAribart/json-schema-to-ts) - uses JSON-Schema definitions used by API Gateway for HTTP request validation to statically generate TypeScript types in your lambda's handler code base
- [middy](https://github.com/middyjs/middy) - middleware engine for Node.Js lambda. This template uses [http-json-body-parser](https://github.com/middyjs/middy/tree/master/packages/http-json-body-parser) to convert API Gateway `event.body` property, originally passed as a stringified JSON, to its corresponding parsed object
```

自動生成された README に記載されていましたが、デフォルトでは`json-schema-to-ts`と`middy`というライブラリがバリデーションやミドルウェアとして使われていて、ハンドラ関数が export する際にラップされています。

```ts:/functions/hello/handler.ts
import { middyfy } from '@libs/lambda';
export const main = middyfy(hello);
```

今回は WebSocket API ではどう使うのか調べるのが面倒で、エラーも出てしまったので外してしまいましたが、便利そうなので使ってみたいですね。

[Serverless Framework の aws-nodejs-typescript のテンプレートを詳しく見る](https://zenn.dev/merutin/articles/588b5f02f16f5c#%E5%88%A9%E7%94%A8%E3%81%97%E3%81%A6%E3%81%84%E3%82%8B%E3%83%A9%E3%82%A4%E3%83%96%E3%83%A9%E3%83%AA%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)

## Go バージョン

Serverless Framework での Go プロジェクトの作り方は公式のブログ記事があったので、こちらを参考にしました。

[Serverless Framework example for Golang and Lambda](https://www.serverless.com/blog/framework-example-golang-lambda-support/)

最初、記事で使われている`aws-go-dep`というテンプレートを使ったらビルドエラーが出て、地味にハマりました。

> There are a couple Go templates already included with the Framework as of v1.26—aws-go for a basic service with two functions, and aws-go-dep for the basic service using the dep dependency management tool. Let's try the aws-go-dep template. You will need dep installed.

記事にちゃんと書いてありましたが、こちらのテンプレートは、Go Modules 以前のデファクトであった`dep`という依存ライブラリ管理ツールを使用するのが前提だったようです。

[テンプレート一覧](https://www.serverless.com/framework/docs/providers/aws/cli-reference/create)に載っていた`aws-go-mod`を使ったところ、無事にビルドが通るようになりました。

## 感想

Node.js と Go x Serverless Framework x Websocket API(API Gateway)を使って、簡易なチャットアプリを再現できました。

Go x Serverless Framework の組み合わせも初めて試してみましたが、問題なく開発することができました。

個別のチャットルームを作ったり、過去のメッセージを表示したりなど、本格的なチャットアプリを作ってみたくなりますね。
