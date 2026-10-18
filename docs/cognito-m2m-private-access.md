# Cognito M2M アクセストークンを閉域で取る検討

- 文書の日付: 2026-10-18
- 根拠の時点: Amazon Cognito 開発者ガイド 2026-10-17 版、GetClientToken API リファレンス 2026-10-07 版、Private API の呼び出し手順 2026-10-17 版
- 対象: すでに Cognito の M2M（機械同士の認証）を使っていて、トークンは公開の経路で取っている。API は Private API Gateway の REST API。Salesforce とは NLB の先の VPC エンドポイント経由でつながっている。
- このフォルダの現状: Cognito や API Gateway の定義ファイルは置かれていない。本文は、上の構成を前提にした設計判断である。実アカウントでの疎通試験は未実施。
- 対になる表: `docs/cognito-m2m-private-access-review.xlsx`（Meiryo UI、モノトーン）

## 今日の結論

SDK を使っただけでは、今のトークン取得は閉域にならない。

今の M2M は、ユーザープールのドメインにある `/oauth2/token` へ HTTPS で合言葉（クライアント ID とクライアントシークレット）を送ってアクセストークンを受け取っている。このドアは公開の名前で、VPC エンドポイントの私道には入らない。AWS SDK にも、このドアを叩く専用の命令は無い。SDK の中から同じ URL を呼んでも、行き先は同じ公開のドアである。

2026年8月31日に、別のドアができた。名前は `GetClientToken`。これは Cognito のユーザープール API（`cognito-idp`）の命令で、ドメインが要らない。AWS CLI、各言語の SDK、素の AWS JSON で呼べる。2025年11月7日からの `cognito-idp` 用インターフェース VPC エンドポイント（PrivateLink）を通る。

したがって、次の条件が揃えば、アクセストークンの取得と Private API の呼び出しは、公衆回線の IP を宛先にせずに通せる。

1. JMeter が VPC の中で動く。または、専用線か VPN の先にいて、DNS が両方の VPC エンドポイントのプライベート IP を返す。
2. トークンは `GetClientToken` で取る。`/oauth2/token` は使わない。
3. そのユーザープールにはドメインを付けない。公式の表は、ドメインが付いたプールを PrivateLink 非互換としている。今のプールはドメインが要る使い方なので、ドメインは消さず、閉域用のプールを新しく分ける。
4. アプリクライアントはクライアントシークレットを持ち、認証フローは `ALLOW_CLIENT_TOKEN_AUTH` だけにする。
5. API 呼び出しは `execute-api` の VPC エンドポイントへ直接出す。Salesforce 用の NLB には乗せない。
6. 取得の直前に DNS の答えがプライベート IP であることを確認する。VPC エンドポイントを作っただけでは、名前が公開 IP のままのことがある。

Salesforce の既存の道は、これとは別である。NLB がインターネット向けなら、Salesforce から NLB までの区間は公衆回線である。その先の VPCE から Private API までは AWS の中である。Salesforce との区間まで公衆回線を無くすには、Salesforce Private Connect（相手側の PrivateLink）が要る。JMeter の閉域化と、Salesforce の入口の公開可否は、同時に片づく話ではない。

## 小学生にもわかるたとえ

町に、関係者だけの建物がある。これが Private API である。建物の門は、大通りではなく私道にしか無い。私道の門が、API Gateway の VPC エンドポイントである。

建物に入るには通行証が要る。通行証がアクセストークンである。通行証には「この機械は、この棚を読んでよい」と書いてある。棚の名前がスコープである。

通行証を売る場所が、昔は一つだった。大通りに面した看板のある店である。看板の名前が、ユーザープールのドメイン（`xxxxx.auth.リージョン.amazoncognito.com`）である。機械は店の窓口 `/oauth2/token` で合言葉を言い、通行証を受け取る。店は私道の中には無い。私道をどれだけ整えても、この店のドアは大通りのままである。

2026年8月に、市役所の窓口でも同じ通行証を売るようになった。窓口の名前が `GetClientToken` である。市役所は私道（`cognito-idp` の VPC エンドポイント）からも行ける。SDK や AWS CLI は、市役所を呼ぶ呼び鈴である。呼び鈴を市役所に向ければ私道で買える。呼び鈴を看板の店に向けると、SDK を使っていても大通りに出る。

点検の人が JMeter である。点検の人が私道の中にいれば、市役所で通行証を受け取り、そのまま建物の門を叩ける。点検の人が町の外にいて、大通りの通用門（インターネット向けの NLB）から入ると、その一歩は公衆回線である。

お客さんである Salesforce は、町の外にいる。通用門が大通りに開いていると、お客さんは大通りを歩いて来る。通用門を私道専用（Private Connect）にすると、お客さんの道も大通りを使わない。点検の人は、お客さんの通用門を借りなくても、建物の門へ直接行ける。

合言葉（クライアントシークレット）は通行証そのものより大事である。道を私道にしても、合言葉が漏れれば、公開の市役所窓口が残っているあいだは外から通行証を買える。私道を増やすことと、大通りの窓口を閉じることは、別の作業である。

## 何を聞かれているか

| 問い | 答え |
| --- | --- |
| SDK を使えば、公開のインターネットへ出ずにアクセストークンを取れるか | `GetClientToken` を、ドメインの無いプールに対して、`cognito-idp` の VPC エンドポイント経由で呼ぶ場合だけ、取れる。今の `/oauth2/token` を SDK や curl で呼ぶ方式は、公開の名前のままである |
| Private API への JMeter も、トークン取得も、公衆回線をまったく通さないでよいか | JMeter の置き場所、DNS、2 種類の VPC エンドポイント、ドメイン無しプールが揃えば可能。NLB の公開リスナーを経由する呼び方では不可能 |
| 今の Salesforce 用 NLB → VPCE を JMeter にも使えば閉域になるか | NLB がインターネット向けなら、そこへ出すパケットは公衆回線へ向かう。閉域の JMeter は NLB を経由せず `execute-api` の VPC エンドポイントを直接使う |

## 歴史と、その都度必要になった理由

人のログインと、機械の通行証と、私道は、別々の時期に足された。一つに見えるのは、後から同じ Cognito という言葉で呼ばれたからである。

### 2014年〜2016年: 人と、AWS の一時キー

Amazon Cognito は、最初はモバイルアプリが AWS の一時的なアクセスキーを受け取るための仕組みだった。こちらは ID プール（`cognito-identity`）である。返ってくるのは AWS の認証情報で、API Gateway に見せるアクセストークンではない。

2016年にユーザープールが一般提供された。人のユーザー名とパスワードを預かるサービスである。アプリは AWS SDK から `InitiateAuth` などを呼ぶ。行き先のホスト名は `cognito-idp.リージョン.amazonaws.com` である。この時点の通行証は、人が自分の情報を見るためのもので、外部 API の細かい権限（カスタムスコープ）はまだ主役ではない。

### 2017年8月: 大通りの看板

フェデレーション、組み込みのログイン画面、OAuth 2.0 がユーザープールに載った。ユーザープールにドメインを付けると、ブラウザで開くログインページと、トークンを渡すエンドポイントが公開される。

ドメインが必要になった理由は、人がブラウザで Google や SAML の画面へ行き来するには、戻ってくるための公開の URL が要るからである。機械同士にはブラウザが無い。それでもトークンの発行口は、この公開ドメインの上に置かれた。

### 2018年5月: Salesforce は「人の IdP」として先に文書化された

ユーザープールが、Salesforce や Ping のような OpenID Connect の IdP を使って人を受け入れる、という文書が追加された。ここでの Salesforce は、API の呼び出し相手というより、人のログインの相手である。

今の構成（Salesforce が Private API を呼ぶ、またはその逆で、間に NLB と VPCE がある）は、この 2018 年のログイン連携とは別の回線の話である。名前が同じでも、ドアは別である。

### リソースサーバーと client credentials

OAuth のカスタムスコープは、2017年の OAuth 対応に含まれて広がった。API を「リソースサーバー」として登録し、`資源の名前/操作` というスコープをアクセストークンへ載せる。機械向けの client credentials は、人がログインしない代わりに、アプリクライアントの ID とシークレットで `/oauth2/token` を叩く形に定着した。

この形が残った理由は、標準の OAuth 2.0 に合わせると、Cognito 以外のライブラリ（Salesforce、JMeter、各種の API クライアント）が同じ HTTPS のフォームでトークンを買えるからである。標準に合わせた代償として、発行口が公開ドメインに固定された。

client credentials のアプリクライアントは、人のログイン（認可コードや暗黙）とは同じクライアントに同居できない。機械用と人用は、当時から別クライアントである。

### 2017年〜2018年: Private API という「私道だけの建物」

API Gateway の Private REST API は、インターフェース VPC エンドポイント（サービス名 `com.amazonaws.リージョン.execute-api`）からしか呼べない。リソースポリシーで、許可する VPC エンドポイントを指定する。

必要になった理由は、API をインターネットの IP に晒さず、VPC と専用線の中だけに置きたかったからである。

呼び出し側の不便も同時に生まれた。プライベート DNS を使わない場合、呼び出しは VPC エンドポイントの長い名前に対して、`Host` または `x-apigw-api-id` ヘッダーで API を指定する。Salesforce のような相手は、このヘッダーを自由に付けられないことがある。そこで、NLB を VPC エンドポイントの ENI の前に置き、相手には一つの名前と証明書だけを見せる形が広まった。NLB は中身の HTTP ヘッダーを書き換えられない。名前を合わせるプロキシや、後年のプライベートカスタムドメインが要る、という複雑さはここから来ている。

VPC Link は逆向きである。API Gateway から、VPC の中の NLB（後に ALB）へ出ていく道である。Salesforce から入ってくる道とは向きが逆なので、VPC Link を足してもトークン取得は閉域にならない。

### 2024年5月9日: 機械の通行証が有料になった

client credentials のトークン要求と、M2M 用アプリクライアントが、ユーザープールの人数課金とは別に課金される、と発表された。それまで機械のトークンは事実上のただに近く、マイクロサービスが呼ぶたびにトークンを取り、クライアントを増やしていた。

このとき必要になったのは、私道ではなく、回数を減らすことである。アクセストークンは有効期限まで再利用する。JMeter がスレッドのたびにトークンを取ると、負荷試験そのものがトークン課金になる。

### 2024年11月: 機能プランと、プライベートな独自ドメイン

Lite / Essentials / Plus という機能プランと、マネージドログインが並んだ。ログイン画面は新しくなった。トークンエンドポイントが公開のドメインであることは変わらない。

同じ頃、Private API にプライベートカスタムドメインが追加された。`Host` ヘッダーの工夫や、名前を偽装するための NLB が、名前の問題だけでは必須ではなくなった。すでに動いている NLB → VPCE を、ドメインができたからといって急いで壊す必要はない。新規や整理のときに、NLB が「相手が VPC の外にいるための入口」なのか「ヘッダーを隠すための中継」なのかを分ける。

### 2025年3月3日と4月30日: 通行証の中身

Essentials または Plus で、M2M のアクセストークンへ独自のクレームを足せるようになった。4月30日には、トークン要求へ `ClientMetadata`（機械の名前や環境などの文脈）を添えられるようになった。クライアントを環境の数だけ増やさなくてよくするための機能である。

`ClientMetadata` は保存も暗号化もされない。Lambda トリガーへ渡るだけである。シークレットや個人情報は載せない。

### 2025年11月7日: 市役所まで私道が来た。看板の店は対象外

ユーザープールが AWS PrivateLink に対応した。VPC から `cognito-idp` の API を、インターネットゲートウェイも NAT も無しに呼べる。

発表文は、対象外を明示している。マネージドログイン、Hosted UI、認可コード、SAML / OIDC のフェデレーション、そして client credentials（当時の M2M）は、VPC エンドポイントを通らない。

ここで「SDK ならプライベートになる」という期待と実装がずれる。SDK が呼ぶ `InitiateAuth` や管理 API は私道を通れる。M2M が実際に使っていた `/oauth2/token` は私道の対象外のままだった。PrivateLink の追加は、機械の通行証の閉域化をまだ完成させていない。

同じ 2025年11月、REST API の VPC Link v2 が ALB へ直接つながるようになった。これは API の背後の統合の話である。JMeter がトークンを取る道とは無関係である。

### 2026年8月31日: GetClientToken

`GetClientToken` が追加された。アプリクライアントが ID とシークレットを渡し、リソースサーバーのカスタムスコープが付いたアクセストークンを受け取る。ドメインは不要である。SDK、CLI、API から呼べる。AWS WAF と VPC エンドポイントの対象である、と発表されている。従来の `/oauth2/token` も残る。

開発者ガイド（2026-10-17）の PrivateLink の表は、こう分けている。

- トークンエンドポイントでの M2M: VPC エンドポイント経由は不可
- `GetClientToken`: SDK と REST API の未認証リクエストの行に含まれる。VPC エンドポイント経由の対象

「未認証」は、IAM の署名（SigV4）で誰かを認可しない、という意味である。合言葉が無いという意味ではない。公式 API は、この命令では IAM ポリシーを評価しない、と書いている。クライアント ID とシークレットが認可の本体である。VPC エンドポイントポリシーとリソースコントロールポリシー（RCP）は、この未認証の命令も許可と拒否の対象にできる。

### 文書の中の矛盾（判断を分けた理由）

同じ PrivateLink のページに、次の二つがある。

1. 表の脚注。ドメインが付いたユーザープールは PrivateLink 非互換。ドメインがあると、VPC エンドポイントを通るユーザープールへの要求は完了しない。SDK の行にも「ドメインが付いていてはならない」とある。完全に私道だけにする運用モードも「ドメインを削除する」から始まる。
2. サポートされる操作の節。ドメインそのものは PrivateLink を通らない。一方で、`CreateUserPoolDomain` を除く SDK の API は PrivateLink を通る、とも読める。

2026-10-17 版の時点で、この二つは同じページの中で食い違っている。本番の設計は、厳しい方の脚注に従う。閉域用プールにはドメインを付けない。既存のドメイン付きプールのドメインは消さない。切替前に、VPC の中から `GetClientToken` を呼び、DNS の答えがプライベート IP であることと、トークンが Private API で受理されることを試験で残す。

## 二つのドア

```text
機械（JMeter / バッチ / SDK）
        │
        ├─ ドアA  今のやり方
        │    https://<ドメイン>/oauth2/token
        │    grant_type=client_credentials
        │    ホストはユーザープールドメイン
        │    VPC エンドポイントのプライベート DNS は、この名前を私道へ向けない
        │    行き先は公開の経路
        │
        └─ ドアB  2026-08-31 以降
             GetClientToken
             ホストは cognito-idp.<リージョン>.amazonaws.com
             プライベート DNS 付きの VPC エンドポイントがあれば、
             同じ名前が VPC 内のプライベート IP になる
             ドメイン不要
             プールにドメインが付いていると、公式の表では私道側が非互換

通行証を持ったあと
        │
        └─ Private REST API
             サービス名 com.amazonaws.<リージョン>.execute-api
             プライベート DNS なら
             https://<api-id>.execute-api.<リージョン>.amazonaws.com/<stage>/...
             この VPC エンドポイントは、Cognito の VPC エンドポイントとは別物
```

SDK は転送装置ではない。SDK は、どのホストへ HTTPS を出すかを決める呼び出し方の名前である。`get_client_token` はドア B へ行く。`requests` や curl で `/oauth2/token` を叩くコードを SDK の関数の中に書いても、ドア A のままである。

ID プールの `GetCredentialsForIdentity` が返すアクセスキーも、ここでいうアクセストークンではない。Private API の Cognito オーソライザーが期待するのは、ユーザープールが発行した JWT である。

## ドア B のトークンの中身

公開の実装例（2026年9月、東京リージョン）では、`GetClientToken` の応答は次の形だった。

```json
{
  "ClientAuthenticationResult": {
    "AccessToken": "<JWT>",
    "ExpiresIn": 3600,
    "TokenType": "Bearer"
  }
}
```

JWT の中身の例は次のとおりである。ID トークンは返らない。リフレッシュトークンも返らない。期限が切れたら、もう一度 `GetClientToken` を呼ぶ。

```json
{
  "sub": "<クライアントID>",
  "token_use": "access",
  "scope": "https://api.example.com/read",
  "iss": "https://cognito-idp.ap-northeast-1.amazonaws.com/<ユーザープールID>",
  "version": 2,
  "client_id": "<クライアントID>",
  "exp": 1788210941,
  "iat": 1788207341
}
```

発行者（`iss`）は、看板のドメインではなく `cognito-idp` の URL である。ユーザープールの通常のアクセストークンと同じ系統である。`aud` が無い。REST API でスコープを見るなら、メソッドの認可スコープと、この `scope` を対応させる。

HTTP API の JWT オーソライザーは audience が必須で、このトークンとは相性が悪い。今回の API は REST なので、REST の認可の話として扱う。

独自クレームを Pre token generation で足している場合は、ドア A で取ったトークンとドア B で取ったトークンのペイロードを試験で並べる。公式は両者を同じ M2M 機能と説明している。オーソライザーが独自クレームに依存するなら、この比較を切替条件にする。

## 閉域にするための構成

### プールを分ける

| プール | ドメイン | トークンの取り方 | 誰が使う | PrivateLink |
| --- | --- | --- | --- | --- |
| 既存（P） | 付けたまま。消さない | `/oauth2/token` | いまドメインを必要としている呼び出し元。Salesforce が自分でトークンエンドポイントを叩いているなら、それもここ | 公式の表では非互換。閉域化の対象にしない |
| 新設（Q） | 付けない。作成も RCP で拒否する | `GetClientToken` のみ | VPC 内の JMeter、閉域のバッチ | 対象にする |

既存プールのドメインを削除すると、Hosted UI、フェデレーション、`/oauth2/token` が同時に止まる。Salesforce の現行経路がドア A に依存しているなら、削除は障害になる。

REST の Cognito オーソライザーは、紐づくユーザープールが一つである。P と Q の両方のトークンを同じメソッドで受けるなら、Lambda オーソライザーで発行者を複数見るか、試験用の API またはステージを分ける。JMeter の閉域試験だけが目的なら、Q のトークンだけを受ける試験用ステージが安全である。Salesforce の本番ステージへ負荷試験を流さない。

### アプリクライアント（プール Q）

公式の作成例に沿う。

```bash
aws cognito-idp create-resource-server \
  --user-pool-id "<プールQのID>" \
  --identifier "https://api.example.internal" \
  --name "private-api" \
  --scopes ScopeName=read,ScopeDescription="read" \
           ScopeName=write,ScopeDescription="write"

aws cognito-idp create-user-pool-client \
  --user-pool-id "<プールQのID>" \
  --client-name "jmeter-private-m2m" \
  --generate-secret \
  --explicit-auth-flows ALLOW_CLIENT_TOKEN_AUTH \
  --allowed-o-auth-scopes "https://api.example.internal/read"
```

条件は次のとおりである。

- クライアントシークレットが必須である。
- `ALLOW_CLIENT_TOKEN_AUTH` は、そのアプリクライアントの唯一の認証フローである。人のログイン用フローとは同居できない。
- `GetClientToken` の引数にユーザープール ID は無い。クライアント ID からプールが決まる。
- スコープを省略すると、クライアントに付いたスコープがすべて載る。JMeter では必要なスコープだけを明示する。
- シークレットは最大 2 つまで同時に有効にできる。ローテーション時は新しいシークレットを足してから古いものを消す。JMeter のプロパティファイルを先に入れ替え、成功を見てから古いシークレットを無効にする。

人のログイン用クライアントへ、あとからこのフローだけを足す、という寄せ方はしない。専用クライアントを新規に作る。

### ネットワーク

VPC にインターフェースエンドポイントを 2 つ作る。

| 用途 | サービス名 | プライベート DNS |
| --- | --- | --- |
| トークン | `com.amazonaws.<リージョン>.cognito-idp` | 有効 |
| Private API | `com.amazonaws.<リージョン>.execute-api` | 有効。有効にすると、同じ VPC から公開 API の標準エンドポイント名は、この私道へ吸い込まれる。公開 API を同じ VPC から叩いている処理が無いかを先に確認する |

エンドポイントのセキュリティグループは、JMeter がいるサブネットから TCP 443 を許可する。サブネットのルートテーブルに、この宛先のための NAT ゲートウェイ経由の特別扱いが不要であることを確認する。VPC エンドポイント宛ては、VPC の中の ENI へ届く。

`execute-api` のプライベート DNS を有効にすると、公開 REST API の `*.execute-api.リージョン.amazonaws.com` も VPC 内では私道側の名前解決になる。公開 API を併用している VPC では、先に影響を洗う。

オンプレミスの JMeter を使う場合は、専用線または VPN に加えて、Route 53 Resolver のインバウンドエンドポイントで次の名前を VPC の DNS へ転送する。

- `cognito-idp.<リージョン>.amazonaws.com`
- `execute-api.<リージョン>.amazonaws.com`

転送が無いと、社内 DNS は公開 IP を返す。VPC エンドポイントが存在していても、パケットは公開のアドレスへ向かう。回線がインターネットに出ていれば「なんとなく成功」し、出ていなければ失敗する。成功だけを見ると閉域になったように見える。

管理用の EC2 にパブリック IP を付けない。操作は Systems Manager Session Manager にする。その場合、閉域を崩さないために `ssm`、`ssmmessages`、`ec2messages` の VPC エンドポイントも要る。シークレットを実行時に取るなら `secretsmanager` と、CMK を使うなら `kms` も要る。閉域の宣言は、トークンと API だけでなく、試験ランナーが起動時に触る先すべてに対して行う。

### エンドポイントポリシー

`GetClientToken` は IAM のプリンシパルを持たない。エンドポイントポリシーで絞るときは、主体を `*` にする。管理 API だけを書いたポリシーへ差し替えると、`GetClientToken` は暗黙の拒否で落ちる。初期状態のフルアクセスから、必要な命令だけを残す変更は、試験プールで先にやる。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": ["cognito-idp:GetClientToken"],
      "Resource": "arn:aws:cognito-idp:<リージョン>:<アカウントID>:userpool/<プールQのID>"
    }
  ]
}
```

ドメイン作成を私道の中から行わせない制御は、公式が例示している。閉域用プールだけに掛ける。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": ["cognito-idp:CreateUserPoolDomain"],
      "Resource": "arn:aws:cognito-idp:<リージョン>:<アカウントID>:userpool/<プールQのID>"
    }
  ]
}
```

アカウント全体の `cognito-idp:*` を VPC 以外で拒否する RCP は、既存プール P の公開経路も巻き込む。P を残すあいだは掛けない。

VPC エンドポイントは私道を増やす。公開の `cognito-idp` エンドポイントは、別途 RCP で拒否するまで残る。クライアント ID とシークレットがあれば、公開経路からも `GetClientToken` は呼べる。シークレットの保管と、公開経路を拒否する RCP は、DNS がプライベートになったあとの次の段である。RCP は試験で「VPC 内は成功、VPC 外は拒否」を見てから本番のプールへ掛ける。

### Private API の呼び方

リソースポリシーの例である。

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:<リージョン>:<アカウントID>:<api-id>/*",
      "Condition": {
        "StringEquals": {
          "aws:SourceVpce": "vpce-xxxxxxxx"
        }
      }
    }
  ]
}
```

プライベート DNS が有効な VPC の中から:

```bash
curl --silent --show-error \
  --header "Authorization: Bearer ${ACCESS_TOKEN}" \
  "https://${API_ID}.execute-api.${AWS_REGION}.amazonaws.com/${STAGE}/resource"
```

プライベート DNS が無い、またはオンプレミスからエンドポイント固有の名前を使う場合は、公式のどちらかである。

```bash
curl --silent --show-error \
  --header "Host: ${API_ID}.execute-api.${AWS_REGION}.amazonaws.com" \
  --header "Authorization: Bearer ${ACCESS_TOKEN}" \
  "https://${VPCE_DNS}.execute-api.${AWS_REGION}.vpce.amazonaws.com/${STAGE}/resource"
```

```bash
curl --silent --show-error \
  --header "x-apigw-api-id: ${API_ID}" \
  --header "Authorization: Bearer ${ACCESS_TOKEN}" \
  "https://${VPCE_DNS}.execute-api.${AWS_REGION}.vpce.amazonaws.com/${STAGE}/resource"
```

2024年後半以降は、プライベートカスタムドメインを VPC エンドポイントへ関連付けると、通常のホスト名だけで呼べる。JMeter と社内ランナーには、こちらを優先する。NLB は、VPC の外にいる Salesforce がまだ必要とする場合に限って残す。

Direct Connect の公式手順では、オンプレミスから公開 DNS 名でプライベート API に到達する経路も案内されている。この「公開 DNS 名」は、VPC エンドポイントの公開 DNS ホスト名がプライベート IP を返す、という PrivateLink の仕組みを指す。名前の形が公開用に見えても、答えの IP がプライベートなら宛先は私道である。判定基準は名前の見た目ではなく、解決された IP である。

## Salesforce と NLB と VPCE

今の「NLB より VPCE 経由」は、Private API を、VPC の外の相手へ渡すための入口である。典型的には次の順である。

```text
Salesforce
  → NLB（インターネット向け、または Private Connect のエンドポイントサービス）
    → ターゲット（execute-api VPC エンドポイントの ENI、またはヘッダーを補うプロキシ）
      → Private REST API
```

NLB は L4 である。`x-apigw-api-id` を足さない。ターゲットが VPC エンドポイントの IP そのもののとき、Salesforce が使うホスト名は、API Gateway が受理する名前と一致している必要がある。プライベートカスタムドメインの関連付け、または中間プロキシが、その一致を作っている。

| NLB の置き方 | Salesforce から NLB まで | NLB から API まで | JMeter を同じ NLB に乗せるべきか |
| --- | --- | --- | --- |
| インターネット向け | 公衆回線 | VPC 内 | 乗せない。試験が公開リスナーと Salesforce の経路に混ざる |
| 社内向け NLB。専用線の先からのみ | 専用線の中 | VPC 内 | 技術的には通る。試験負荷を本番の入口から分離するなら、JMeter は VPCE 直がよい |
| Salesforce Private Connect のエンドポイントサービス | AWS と Salesforce の PrivateLink。公衆インターネットの IP を宛先にしない | VPC 内 | Salesforce の入口である。JMeter の通り道にしない |

JMeter の閉域試験が証明したいのは「トークンも API もプライベート IP だけ」である。証明の呼び方は、NLB を経由しない。

Salesforce 自身が `/oauth2/token` を叩いているなら、そのトークン取得は Salesforce の出ロから公開ドメインへ向かう。GetClientToken へ移しても、Salesforce の実行環境が VPC の私道を DNS で見ていない限り、Salesforce のトークン取得は閉域にならない。閉域に寄せるなら、トークン取得は VPC 内の中継に置き、Salesforce には業務 API だけを Private Connect で見せる、という分け方が合う。中継をどこに置くかは、Salesforce 側の現行の認証方式を見てから決める。この文書の時点では、Salesforce がトークンを自分で取っているのか、中継が取っているのかは、リポジトリ上の定義が無く未確認である。

## 呼び出し方

どの例も、シークレットとトークンを画面へ出さない。プレースホルダのままでは動かない。実値は Secrets Manager か、権限を絞ったローカルファイル（`umask 077`）から読む。

### AWS CLI

CLI はドア B をそのまま呼べる。2026年8月以降の AWS CLI v2 が要る。古い CLI は `get-client-token` を知らない。`aws cognito-idp help` に `get-client-token` が出ることを先に見る。

VPC の中で、プライベート DNS が有効なら、エンドポイント URL の指定は不要である。通常の名前がプライベート IP に解決される。

```bash
aws cognito-idp get-client-token \
  --region "${AWS_REGION}" \
  --client-id "${CLIENT_ID}" \
  --secret "${CLIENT_SECRET}" \
  --scopes "https://api.example.internal/read" \
  --query "ClientAuthenticationResult.AccessToken" \
  --output text
```

`--endpoint-url` に `vpce-....vpce.amazonaws.com` を渡すと、証明書の名前とホスト名が一致せず TLS 検証で落ちることがある。名前の上書きより、プライベート DNS か、次のシェルの `--resolve` を使う。

この命令は IAM では認可されない。CLI に IAM 資格情報が付いていても、通る理由はクライアントシークレットである。逆に、IAM 権限を足してもシークレットの代わりにはならない。

### シェル（VPC 内の Linux）

このスクリプトは、名前がプライベート IP に解決されないときはトークンを取りに行かない。公開 IP へ落ちる成功を失敗として扱う。

```bash
#!/bin/bash
# get-m2m-token.sh
# 使い方:
#   CLIENT_ID=... CLIENT_SECRET=... AWS_REGION=ap-northeast-1 \
#   SCOPE=https://api.example.internal/read \
#   ./get-m2m-token.sh
set -euo pipefail
umask 077

: "${CLIENT_ID:?}"
: "${CLIENT_SECRET:?}"
: "${AWS_REGION:?}"
: "${SCOPE:?}"

HOST="cognito-idp.${AWS_REGION}.amazonaws.com"

is_private() {
  local ip="$1"
  [[ "${ip}" =~ ^10\. ]] && return 0
  [[ "${ip}" =~ ^192\.168\. ]] && return 0
  [[ "${ip}" =~ ^172\.(1[6-9]|2[0-9]|3[0-1])\. ]] && return 0
  return 1
}

mapfile -t IPS < <(getent ahostsv4 "${HOST}" | awk '{print $1}' | sort -u)
if [[ "${#IPS[@]}" -eq 0 ]]; then
  echo "DNS の答えがありません: ${HOST}" >&2
  exit 2
fi

RESOLVE_ARGS=()
for ip in "${IPS[@]}"; do
  if ! is_private "${ip}"; then
    echo "公開アドレスに解決されました: ${HOST} -> ${ip}" >&2
    echo "VPC エンドポイントのプライベート DNS、または Resolver の転送を確認してください。" >&2
    exit 3
  fi
  RESOLVE_ARGS+=(--resolve "${HOST}:443:${ip}")
done

REQ="$(mktemp)"
RESP="$(mktemp)"
trap 'rm -f "${REQ}" "${RESP}"' EXIT

python3 - "${CLIENT_ID}" "${CLIENT_SECRET}" "${SCOPE}" > "${REQ}" <<'PY'
import json, sys
client_id, secret, scope = sys.argv[1:]
json.dump({"ClientId": client_id, "Secret": secret, "Scopes": [scope]}, sys.stdout)
PY

HTTP_CODE="$(
  curl --silent --show-error \
    --connect-timeout 5 --max-time 20 \
    --output "${RESP}" --write-out "%{http_code}" \
    "${RESOLVE_ARGS[@]}" \
    --header "Content-Type: application/x-amz-json-1.1" \
    --header "X-Amz-Target: AWSCognitoIdentityProviderService.GetClientToken" \
    --data-binary @"${REQ}" \
    "https://${HOST}/"
)"

if [[ "${HTTP_CODE}" != "200" ]]; then
  echo "GetClientToken が HTTP ${HTTP_CODE} を返しました。応答本文はシークレットを含むことがあるため出しません。" >&2
  exit 4
fi

python3 - "${RESP}" <<'PY'
import base64, json, sys, time
body = json.load(open(sys.argv[1], encoding="utf-8"))
token = body["ClientAuthenticationResult"]["AccessToken"]
expires_in = body["ClientAuthenticationResult"]["ExpiresIn"]
part = token.split(".")[1]
part += "=" * ((4 - len(part) % 4) % 4)
payload = json.loads(base64.urlsafe_b64decode(part))
print(f"token_chars={len(token)}")
print(f"expires_in={expires_in}")
print(f"iss={payload.get('iss')}")
print(f"scope={payload.get('scope')}")
print(f"client_id={payload.get('client_id')}")
print(f"token_use={payload.get('token_use')}")
print(f"exp={payload.get('exp')}")
remain = int(payload.get("exp", 0)) - int(time.time())
print(f"remaining_seconds={remain}")
# 後段の API 呼び出しへ渡すときだけ、ファイルへ書く
open("/tmp/m2m-access-token", "w", encoding="utf-8").write(token)
PY
chmod 600 /tmp/m2m-access-token
```

`X-Amz-Target` の接頭辞 `AWSCognitoIdentityProviderService` は、`cognito-idp` の JSON 1.1 プロトコルで `InitiateAuth` などと同じサービス接頭辞である。`GetClientToken` 向けの公式の生 HTTP 例は、確認した範囲では CLI と SDK が主で、ターゲット文字列の公式掲載は見当たらなかった。最初の 1 回は HTTP ステータスとエラーコードだけを見て接頭辞を確定する。CLI が使えるランナーでは、上の curl より `aws cognito-idp get-client-token` を優先する。CLI はターゲット文字列を自分で組み立てる。

トークンファイルを読む API 呼び出し:

```bash
#!/bin/bash
set -euo pipefail
: "${API_ID:?}"
: "${AWS_REGION:?}"
: "${STAGE:?}"
TOKEN="$(cat /tmp/m2m-access-token)"
HOST="${API_ID}.execute-api.${AWS_REGION}.amazonaws.com"

is_private() {
  local ip="$1"
  [[ "${ip}" =~ ^10\. ]] && return 0
  [[ "${ip}" =~ ^192\.168\. ]] && return 0
  [[ "${ip}" =~ ^172\.(1[6-9]|2[0-9]|3[0-1])\. ]] && return 0
  return 1
}

mapfile -t IPS < <(getent ahostsv4 "${HOST}" | awk '{print $1}' | sort -u)
[[ "${#IPS[@]}" -gt 0 ]] || { echo "API の DNS が空です" >&2; exit 2; }
ARGS=()
for ip in "${IPS[@]}"; do
  is_private "${ip}" || { echo "API が公開アドレスです: ${ip}" >&2; exit 3; }
  ARGS+=(--resolve "${HOST}:443:${ip}")
done

curl --silent --show-error \
  "${ARGS[@]}" \
  --header "Authorization: Bearer ${TOKEN}" \
  "https://${HOST}/${STAGE}/resource"
```

### PowerShell（VPN の先の Windows ランナー）

ランナーが Windows でも、判定は同じである。解決結果がプライベート IP でなければ先へ進まない。

```powershell
$ErrorActionPreference = "Stop"
$Region = $env:AWS_REGION
$HostName = "cognito-idp.$Region.amazonaws.com"
$ips = [System.Net.Dns]::GetHostAddresses($HostName) |
  Where-Object { $_.AddressFamily -eq "InterNetwork" } |
  ForEach-Object { $_.IPAddressToString }

function Test-PrivateIpv4([string]$Ip) {
  return ($Ip -like "10.*") -or ($Ip -like "192.168.*") -or ($Ip -match "^172\.(1[6-9]|2[0-9]|3[0-1])\.")
}

if (-not $ips) { throw "DNS の答えがありません: $HostName" }
foreach ($ip in $ips) {
  if (-not (Test-PrivateIpv4 $ip)) {
    throw "公開アドレスに解決されました: $HostName -> $ip"
  }
}

aws cognito-idp get-client-token `
  --region $Region `
  --client-id $env:CLIENT_ID `
  --secret $env:CLIENT_SECRET `
  --scopes $env:SCOPE `
  --query "ClientAuthenticationResult.AccessToken" `
  --output text |
  Set-Content -Path "$env:TEMP\m2m-access-token" -Encoding ascii -NoNewline
```

シークレットをコマンドラインに置くと、実行中のプロセス一覧に残ることがある。Windows の試験機では、可能なら CLI の `--cli-input-json` と、ACL を絞ったファイルを使う。使い終わりにファイルを消す。

### SDK

SDK のクライアントは、資格情報プロバイダが IAM 署名を付けても、この API の認可には使われない。リージョンを明示する。VPC 内でプライベート DNS が有効なら、エンドポイントの上書きはしない。

Python（boto3。`get_client_token` を持つ版）:

```python
import os
import boto3

client = boto3.client("cognito-idp", region_name=os.environ["AWS_REGION"])
result = client.get_client_token(
    ClientId=os.environ["CLIENT_ID"],
    Secret=os.environ["CLIENT_SECRET"],
    Scopes=[os.environ["SCOPE"]],
)
token = result["ClientAuthenticationResult"]["AccessToken"]
expires_in = result["ClientAuthenticationResult"]["ExpiresIn"]
```

Java（AWS SDK for Java v2。JMeter に同梱するなら、`GetClientToken` が入った 2026年8月以降の版）:

```java
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.cognitoidentityprovider.CognitoIdentityProviderClient;
import software.amazon.awssdk.services.cognitoidentityprovider.model.GetClientTokenRequest;

try (var cognito = CognitoIdentityProviderClient.builder()
        .region(Region.of(System.getenv("AWS_REGION")))
        .build()) {
    var response = cognito.getClientToken(GetClientTokenRequest.builder()
            .clientId(System.getenv("CLIENT_ID"))
            .secret(System.getenv("CLIENT_SECRET"))
            .scopes(System.getenv("SCOPE"))
            .build());
    String accessToken = response.clientAuthenticationResult().accessToken();
    int expiresIn = response.clientAuthenticationResult().expiresIn();
}
```

SDK の版が古いと、メソッドが無くてコンパイルまたは実行が失敗する。失敗したら URL を `/oauth2/token` に戻してはいけない。CLI の版を上げるか、次の素の HTTP を使う。

### 素の HTTP（SDK が古くてもドア B を使う）

```http
POST / HTTP/1.1
Host: cognito-idp.ap-northeast-1.amazonaws.com
Content-Type: application/x-amz-json-1.1
X-Amz-Target: AWSCognitoIdentityProviderService.GetClientToken

{"ClientId":"<クライアントID>","Secret":"<クライアントシークレット>","Scopes":["https://api.example.internal/read"]}
```

署名ヘッダーは必須ではない。付けても IAM では認可されない。

### JMeter

JMeter の HTTP リクエストは AWS SDK ではない。ドア B を使うには、上の素の HTTP をサンプラーにする。SDK の JAR を `lib` へ足して JSR223 から呼ぶ方法もある。ランナーの Java と JAR の版管理が増える。閉域試験では HTTP サンプラーを本線にする。

配置:

- GUI の操作 PC がインターネット側でもよい。その PC からトークンを取らない。
- リモートエンジン、または `jmeter -n` の実行先を、プライベート IP だけの EC2 にする。
- JMeter のプロキシ欄は空にする。会社の SSL インスペクションを通すと、宛先の意味が変わる。
- 分散実行の RMI をインターネットへ開けない。結果の回収は、S3 のゲートウェイ型 VPC エンドポイントか、Session Manager のファイル転送にする。

テスト計画:

1. setUp スレッドグループを 1 スレッド、1 ループにする。
2. JSR223 プリプロセッサで DNS を調べ、プライベート IP でなければ `AssertionResult` を失敗にして止める。トークン用ホストと API 用ホストの両方を見る。
3. HTTP リクエスト「GetClientToken」。
   - プロトコル `https`
   - サーバ `cognito-idp.${__P(AWS_REGION)}.amazonaws.com`
   - メソッド POST
   - パス `/`
   - ボディは `${__P(CLIENT_ID)}` と `${__P(SCOPE)}` を使った JSON。シークレットはテスト計画に直書きしない。`-JCLIENT_SECRET` もコマンドライン履歴に残る。セットアップスレッドの中で、権限を絞ったファイルから読み、変数に置く。
4. ヘッダーマネージャ。
   - `Content-Type: application/x-amz-json-1.1`
   - `X-Amz-Target: AWSCognitoIdentityProviderService.GetClientToken`
5. JSON Extractor。
   - 変数名 `access_token`
   - JSON Path `$.ClientAuthenticationResult.AccessToken`
   - `$.ClientAuthenticationResult.ExpiresIn` も取る
6. JSR223 ポストプロセッサ。

```groovy
props.put("ACCESS_TOKEN", vars.get("access_token"))
props.put("ACCESS_TOKEN_DEADLINE_MS",
    String.valueOf(System.currentTimeMillis() +
        (Long.parseLong(vars.get("expires_in")) - 60L) * 1000L))
```

7. 本スレッドグループの HTTP ヘッダーマネージャ。
   - `Authorization: Bearer ${__P(ACCESS_TOKEN)}`
8. HTTP リクエスト「Private API」。
   - サーバ `${__P(API_ID)}.execute-api.${__P(AWS_REGION)}.amazonaws.com`
   - パス `/${__P(STAGE)}/resource`
   - プロキシ無し
9. 試験時間がトークン期限を超える場合だけ、期限の 60 秒前に setUp と同じ取得を 1 回やり直す。スレッドごとに取り直さない。

JSON Extractor の後に、応答本文をログへ残すリスナーを付けない。失敗時のアサーションメッセージには、HTTP ステータスと Cognito の `__type` だけを出す。

DNS を確認する Groovy の骨子:

```groovy
import java.net.InetAddress

def mustBePrivate = { String host ->
    def addrs = InetAddress.getAllByName(host)
    if (addrs.length == 0) {
        throw new IllegalStateException("DNS が空です: " + host)
    }
    addrs.each { addr ->
        def hostAddr = addr.hostAddress
        def privateAddr = hostAddr.startsWith("10.") ||
            hostAddr.startsWith("192.168.") ||
            hostAddr ==~ /^172\.(1[6-9]|2[0-9]|3[0-1])\..*/
        if (!privateAddr) {
            throw new IllegalStateException("公開アドレスです: " + host + " -> " + hostAddr)
        }
        log.info("private dns " + host + " -> " + hostAddr)
    }
}

mustBePrivate("cognito-idp." + props.get("AWS_REGION") + ".amazonaws.com")
mustBePrivate(props.get("API_ID") + ".execute-api." + props.get("AWS_REGION") + ".amazonaws.com")
```

負荷の掛け方。1000 リクエスト毎秒で 1 時間、リクエストのたびにトークンを取ると、トークン要求は 360 万回になる。M2M の料金はトークン要求数とアプリクライアント数で階段になる。最新の単価は Cognito の料金ページの Machine-to-machine の表をその月のリージョンで見る。試験クライアントは本番クライアントと分け、期限まで同じトークンを再利用し、取得回数の上限を setUp に閉じ込める。

## 運用提案

### やること

1. プール Q を新設する。ドメインは付けない。リソースサーバーと、`ALLOW_CLIENT_TOKEN_AUTH` だけのアプリクライアントを作る。
2. プール P のドメイン、既存アプリクライアント、Salesforce の経路は変更しない。
3. `cognito-idp` と `execute-api` の VPC エンドポイントを、JMeter 用サブネットに置く。プライベート DNS の影響（公開 API の名前が吸い込まれること）を先に確認する。
4. Private API の試験用ステージまたは試験用 API で、プール Q の発行者とスコープを受ける。本番の Salesforce ステージには付けない。
5. パブリック IP を持たない EC2 で、この文書のシェルを実行する。DNS がプライベートであること、`GetClientToken` が 200 であること、Private API が 200 または業務上正しい応答であることを記録する。
6. 同じ EC2 で `jmeter -n` を短時間流す。リスナーにトークンが出ないことをログで確認する。
7. ルートテーブルから、この試験サブネットの NAT を外した状態で、同じ試験が成功することを確認する。これが「公衆回線を使っていない」の実証になる。NAT が残ったままの成功は、ドア A へ逃げる道が残っている。
8. 成功のあとで、プール Q に対する `CreateUserPoolDomain` の拒否を入れる。公開経路からの `GetClientToken` 拒否は、VPC 内成功と VPC 外拒否の両方を試験してから入れる。
9. トークンの再利用、シークレットの 2 本ローテーション、取得失敗のアラーム、M2M 料金の予算アラームを本番手順に書く。

### やらないこと

- 既存プールのドメインを、試験の途中で削除しない。
- JMeter を Salesforce 用のインターネット向け NLB へ向けない。
- `/oauth2/token` が VPC の中から NAT 経由で成功したことを、閉域化の完了とみなさない。NAT の先は公開のアドレスである。AWS のネットワーク内に留まることはあっても、PrivateLink ではない。
- アカウント全体の Cognito RCP を、既存の公開クライアントが残った日に掛けない。
- クライアントシークレットを JMX、Git、チャット、JMeter のログへ置かない。
- ID プールの一時アクセスキーを、Private API の Bearer トークンの代わりに使わない。

### 段階

| 段階 | 期間の目安 | 完了の条件 |
| --- | --- | --- |
| 0. 現状の宛先を記録する | 半日 | 今のトークン URL の解決 IP と、JMeter の API 宛先が NLB なのか VPCE 直なのかを、コマンド出力で残す |
| 1. プール Q と 2 つの VPC エンドポイント | 1〜2 日 | NAT 無しサブネットから `GetClientToken` と Private API 試験ステージが成功する |
| 2. JMeter の setUp 化 | 1 日 | トークン取得が試験全体で数回に収まり、ログに JWT が無い |
| 3. 入口の分離 | Salesforce の現行方式の確認後 | Salesforce がドア A を自分で叩いているなら、その経路は P のまま残す。閉域が必要なら中継方式を別設計にする。JMeter は VPCE 直のまま |
| 4. 公開窓口を閉じる | 段階 1 の証拠のあと | プール Q だけ、VPC 外の `GetClientToken` が拒否される。プール P は従来どおり |

段階 3 が空欄なのは、Salesforce がトークンをどこで取っているかが、この作業環境のファイルからは確認できないからである。ここを埋めずに NLB を撤去しない。

## 検証手順（実施者用）

試験用サブネットの EC2（パブリック IP 無し）で、上から順に行う。

```bash
# 1. エンドポイントが見えること
aws ec2 describe-vpc-endpoints \
  --filters "Name=service-name,Values=com.amazonaws.${AWS_REGION}.cognito-idp,com.amazonaws.${AWS_REGION}.execute-api" \
  --query "VpcEndpoints[].{Id:VpcEndpointId,Service:ServiceName,State:State,PrivateDns:PrivateDnsEnabled}"

# 2. 名前がプライベート IP であること
getent ahostsv4 "cognito-idp.${AWS_REGION}.amazonaws.com"
getent ahostsv4 "${API_ID}.execute-api.${AWS_REGION}.amazonaws.com"

# 3. ドアB
./get-m2m-token.sh

# 4. Private API
./call-private-api.sh

# 5. ドアAをこのサブネットから叩くと、NAT が無い場合は失敗すること
#    失敗することが、閉域の証拠になる
curl --connect-timeout 5 --max-time 10 \
  "https://${POOL_DOMAIN}/oauth2/token" || echo "ドアAは閉域サブネットから到達できない（想定どおり）"
```

記録に残すもの: コマンドの終了コード、解決された IP、HTTP ステータス、トークンの `iss` と `scope` と `exp`。JWT 全文とシークレットは残さない。

## 費用と壊れ方

- インターフェース VPC エンドポイントは、時間と処理データ量で課金される。トークン用と API 用で 2 つ。可用性と AZ 障害を見るなら 2 AZ 以上にする。
- M2M はアプリクライアント数とトークン要求数で課金される。キャッシュを手順に含めないと、閉域化のあとに請求だけが増える。
- 独自クレームは Essentials または Plus の機能である。トークンを取るだけの `GetClientToken` は、発表文ではユーザープールがあるリージョンで使え、標準の M2M 料金が適用される。プランの現行条件は料金ページで確認する。
- プライベート DNS を `execute-api` で有効にすると、同じ VPC の公開 API 呼び出しが私道側へ変わる。公開 API 用のリソースポリシーが VPCE を許していないと、その公開 API が VPC 内から失敗する。
- ドメイン付きプールへ、脚注を見ずに VPC エンドポイント経由の SDK 呼び出しを移すと、ページ内の矛盾がそのまま障害になる。プール Q を分けておけば、既存は止まらない。
- `CreateUserPoolDomain` は VPC エンドポイント経由では拒否される。ドメインの管理作業は、私道の中のランナーでは行わない。
- シークレットをコマンドラインに出すと、プロセス一覧とシェル履歴に残る。ファイルか Secrets Manager を使う。
- トークンの有効期限は時間単位であることが多い（実測例は 3600 秒）。長い負荷試験で期限切れを再取得しないと、途中から 401 になる。再取得は 1 本のセットアップにまとめる。

## 出典

- PrivateLink と認証フローの表（2026-10-17）: https://docs.aws.amazon.com/cognito/latest/developerguide/vpc-interface-endpoints.html
- PrivateLink 発表。client credentials は対象外と明記（2025-11-07）: https://aws.amazon.com/about-aws/whats-new/2025/11/amazon-cognito-user-pools-private-connectivity-aws-privatelink/
- GetClientToken 発表（2026-08-31）: https://aws.amazon.com/about-aws/whats-new/2026/08/amazon-cognito-get-client-token/
- GetClientToken API（2026-10-07）: https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_GetClientToken.html
- スコープ、M2M、リソースサーバー。CLI の作成例を含む（2026-10-16）: https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html
- トークンエンドポイント（2026-10-17）: https://docs.aws.amazon.com/cognito/latest/developerguide/token-endpoint.html
- ドメインは任意。M2M は GetClientToken でも可（2026-10-15）: https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-assign-domain.html
- SDK の使用例（2026-10-15）: https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-identity-provider_example_cognito-identity-provider_GetClientToken_section.html
- Private API の呼び出し方（2026-10-17）: https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-api-test-invoke-url.html
- Private API のカスタムドメイン: https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-custom-domains.html
- M2M アクセストークンのカスタムクレーム（2025-03-03）: https://aws.amazon.com/about-aws/whats-new/2025/03/amazon-cognito-access-token-m2m-authorization-flows/
- M2M の文脈情報（2025-04-30）: https://aws.amazon.com/about-aws/whats-new/2025/04/amazon-cognito-context-machine-to-machine-flows/
- 実応答の形の確認記事（2026-09-01）: https://dev.classmethod.jp/articles/amazon-cognito-get-client-token/

文書履歴の「2017年8月に OAuth と Hosted UI」は、Cognito 開発者ガイドの文書履歴の 2017-08-10 の項に沿っている。client credentials がカスタムスコープと同じ日に一般提供されたかまでは、今回参照した文書履歴の抜粋だけでは日付を特定できていない。定着した形（ドメイン上の `/oauth2/token`）については、現行のトークンエンドポイントの文書で確認できる。
