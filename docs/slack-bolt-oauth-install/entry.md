# Slack Bolt で簡単に複数ワークスペースにインストールできるアプリを開発しよう！（TypeScript, Lambda, Serverless Framework によるサンプルコード付き）

![Slack_User-token-install.gif](assets/Slack_User-token-install.gif)

# はじめに

## おやくそく

この記事は [「Develop fun!」を体現する Works Human Intelligence #2 Advent Calendar 2020](https://qiita.com/advent-calendar/2020/whi-2) の 21 日目の記事です。
昨日は @etnk さんの「[自然言語処理のサービスのAWS上での構成を考えてみる](https://qiita.com/etnk/items/92b2871d332d923236c7)」でした。

本日は私 @irongineer が Slack 開発をより Develop fun! していくような記事をお届けできたらと思います。
記事を書くのはこれが初めてです。どうぞよろしくお願いいたします。

## 概要

Slack でアプリを作成したことがありますか？
Slack にちょっと便利なアプリを作成すると、日頃の業務が楽になったり、コミュニケーションを取るのが楽しくなったりしますよね。
最近は Slack 公式のアプリ開発フレームワーク、[Bolt](https://slack.dev/bolt-js/ja-jp/tutorial/getting-started)（ボルト）が登場したこともあり、
ますます Slack アプリを作成することが簡単になってきました。
ただ、よくある Slack アプリチュートリアルだと **「Bot トークンを環境変数としてコピー&ペーストし、**
**Slack アプリ管理コンソールからアプリをインストールする」** という方法が多く紹介されていないでしょうか？
この方法は開発した Slack アプリを利用するうえで最も手っ取り早い方法ですが、単一のワークスペースに紐付いたトークンしか利用できないため、
違うワークスペースで利用する場合はワークスペースごとに Slack アプリ管理コンソールでアプリを作成する必要があります。
**せっかく作った便利なアプリ、もっと手軽に色んなワークスペースで利用したくありませんか？**
ということで、この記事では Bolt の OAuth フローを利用して簡単に複数ワークスペースにアプリをインストールするための手順を説明します！

## 書くことと書かないこと

### 書くこと

- Bolt の簡単な紹介
- Bolt を使った従来のアプリ作成方法
- Bolt を使った OAuth フローの実装方法
- TypeScript, Lambda, Serverless Framework などを利用したサンプルコード（付録）

### 書かないこと

- Slack アプリの作成方法（[リファレンス](#リファレンス)を参考にしてね）
- OAuth 2.0 についての説明（[リファレンス](#リファレンス)を参考にしてね）
- TypeScript, Lambda, Serverless Framework などの基礎知識（ググってください oOo）

# 本題

## Bolt

Bolt は Slack のアプリを直感的に利用できるフレームワークです。
本来、Slack API を利用するうえで必要な様々な非機能要件を Bolt がラップしてくれるため、
非常に簡単に Slack アプリを実装でき、かつアプリの本質的なロジックに集中できるようになります。
2020/12/21 時点で言語は JavaScript(Node.js), Java, Python をサポートしています。
特に JavaScript 版に関しては[日本語ドキュメント](https://slack.dev/bolt/ja-jp)が現在ではすべて翻訳されていっているため、
比較的新しいフレームワークながらも手を出しやすくなっています。

Bolt を触ったことがない！　という方は、以下の記事を見てみてください。
Gif や動画を用いながら丁寧に説明してくれているので、初めての方でも簡単にアプリを作成できるはずです。

- [Hello World, Bolt! ⚡️ Bolt フレームワークを使って Slack Bot を作ろう](https://qiita.com/tomomi_slack/items/93538f9a69eb4015f951)
- [Bolt を使って Slack でヘルプデスクチームとのやりとりを効率化するアプリをつくってみよう](https://qiita.com/seratch/items/9b5666479ad9f602e5ee)

## アプリをインストールできるようにするためには

上の記事の説明でもありますが、手っ取り早く Slack アプリを作成して、自身が利用するワークスペースにインストールしたい場合は、

1. **Signing Secret** を `.env` にコピー&ペーストするなどして、環境変数として認識させる
2. 左のメニューから Install App へ行き、必要な Bot/User スコープを追加する
3. 緑の **Install App to Workspace** ボタンをクリックしてアプリをワークスペースにインストールする
4. 生成された **Bot Token** を `.env` にコピー&ペーストするなどして、環境変数として認識させる
5. App のインスタンス化時に環境変数から **Signing Secret** や **Bot Token** をセットする

```javascript
const { App } = require("@slack/bolt");

const app = new App({
  signingSecret: process.env.SLACK_SIGNING_SECRET,
  token: process.env.SLACK_BOT_TOKEN,
});
```

といった手順を行います。
この方法はとても簡単ですが、デメリットとして、**「アプリを作成したワークスペースにしかインストールできない」** という制限があります。
（そのアプリを作成したワークスペースのトークンをそのまま利用しているため）

複数のワークスペースにアプリをインストールできるようにするためには、
アプリインストール時にワークスペースごとのトークンを取得・保持し、
Slack API を呼び出すときにそれらのトークンを利用することが必要です。
**ワークスペースごとのトークンを取得できるようにするためには、Slack の OAuth フローを実装する必要があります。**

## OAuth フロー

OAuth は，「認可情報の委譲」のための仕様です。
ユーザーに認可情報の委譲してもらい、本人の代わりにサービスが API 等を通じて何かしらの処理をするための仕組みです。
「OAuth って聞いたことあるけどいまいち仕組みが分からないんだよね」という方は、以下などの記事を見てみてください。

- [一番分かりやすい OAuth の説明](https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be)

Slack では、ユーザーがアプリをインストールするとき、ユーザーの権限を委譲することを許可することを引き換えに、
そのワークスペースでの Bot やユーザーのトークンをアプリに渡しています。
（おそらく Slack を利用されている方は、アプリをインストールするときにこんな画面を見たことがあるんじゃないでしょうか）

![assets/Slack_User-token-install.png](assets/Slack_User-token-install.png)

さて、Slack で OAuth フローを実装するためには、上述の **Bot Token** や **User Token** を利用する代わりに、
**Client ID** と **Client Secret** が必要になります（こちらも Slack アプリ管理コンソールから取得できます）。
次項では、Bolt を使わない場合と使った場合で、どのようにして OAuth フローを実装していくかを説明します。

※ Slack は 2020 年前半に OAuth 2.0 の API の `v2` を公開しています。
二重の v2 について混乱するかもしれませんが、OAuth 2.0 は OAuth 仕様の 2.0 バージョンを指し、
これは Slack の OAuth 2.0 API の 2 番目のバージョンを指しています。
昔から運用しているアプリなどを除いて、基本的に `v2` の OAuth API を利用するようにしましょう。

## Bolt を使わない場合

こんな感じのフローを実装する必要があります。

![Obtaining access tokens with OAuth 2.0](https://a.slack-edge.com/fbd3c/img/api/articles/oauth_scopes_tutorial/slack_oauth_flow_diagram.png)

（引用：[Obtaining access tokens with OAuth 2.0](https://api.slack.com/authentication/oauth-v2#obtaining)）

通信が複数回に渡ったり、通信をやり取りする相手が複数いて複雑に感じますね。

それでも、Slack の OAuth API を利用すると、実際に開発者が意識するのは以下の 3 ステップだけになります。

1. Asking for scopes（スコープを求める）
2. Waiting for a user to approve your requested scopes（ユーザーが要求されたスコープを承認するのを待つ）
3. Exchanging a temporary authorization code for an access token（アクセストークンを取得するための認可コードを交換する）

### 1. Asking for scopes（スコープを求める）

`v2/authorize` API を利用して、認可リクエストの URL を生成してユーザーに実行してもらいます。
このリクエストに `scope` を指定することによって、ユーザーに許可を求める権限を適切に指定できます。

```http
https://slack.com/oauth/v2/authorize?scope=incoming-webhook,commands&client_id=3336676.569200954261
```

ユーザーはアプリのインストールを許可すると、登録した Redirect URL に遷移します。

### 2. Waiting for a user to approve your requested scopes（ユーザーが要求されたスコープを承認するのを待つ）

ここは特に何もする必要はありません。
上記の手順で指定した Redirect URL で HTTP リクエストをリッスンして、ユーザーの復帰に備えるだけです。

### 3. Exchanging a temporary authorization code for an access token（アクセストークンを取得するための認可コードを交換する）

ここでは以下のことをする必要があります。

- 認可コード（`code`）の変換と取得
- state の検証
- 認可コードとアクセストークンの交換

上記を考慮のうえ、`oauth.v2.access` API で以下のようなリクエストを発行します。

```console
curl -F code=1234 -F client_id=3336676.569200954261 -F client_secret=ABCDEFGH https://slack.com/api/oauth.v2.access
```

成功すると以下のようなレスポンスが返ってきます。

```json
{
    "ok": true,
    "access_token": "xoxb-17653672481-19874698323-pdFZKVeTuE8sk7oOcBrzbqgy",
    "token_type": "bot",
    "scope": "commands,incoming-webhook",
    "bot_user_id": "U0KRQLJ9H",
    "app_id": "A0KRD7HC3",
    "team": {
        "name": "Slack Softball Team",
        "id": "T9TK3CUKW"
    },
    "enterprise": {
        "name": "slack-sports",
        "id": "E12345678"
    },
    "authed_user": {
        "id": "U1234",
        "scope": "chat:write",
        "access_token": "xoxp-1234",
        "token_type": "user"
    }
}
```

取得したトークン等を DB にセキュアな方法で保存すれば、あとは API 利用時にトークンを取得してセットすれば OK です。

今までの Slack OAuth フローの実装に関しては、以下の記事も参考になります。

- [SlackのOAuth認証を使ってユーザ情報を取得してみた](https://qiita.com/dbgso/items/a95a3364d9a8c67f3387)

ここまで見て、「まだ難しい」と思った人もいるのではないでしょうか。
アクセストークンを取得するために、複数回リクエストをやり取りする必要があるため、難しく感じます。
また OAuth 2.0 の仕様を最低限知っていないと、各リクエストは何のためにやっているかが分からず、手を出しにくいです。

## Bolt を使った場合

基本的には、公式ドキュメントどおりに実装すれば OK です。

Bolt for JavaScript では、`clientId`、`clientSecret`、`stateSecret`、`scopes` を App 初期化時に指定すると、
OAuth フローのためのルートのセットアップや state パラメーターを検証する機能が有効になります。

```javascript
const app = new App({
  signingSecret: process.env.SLACK_SIGNING_SECRET,
  clientId: process.env.SLACK_CLIENT_ID,
  clientSecret: process.env.SLACK_CLIENT_SECRET,
  stateSecret: 'my-state-secret',
  scopes: ['channels:read', 'groups:read', 'channels:manage', 'chat:write', 'incoming-webhook'],
  installationStore: {
    storeInstallation: async (installation) => {
      // 実際のデータベースに保存するために、ここのコードを変更
      return await database.set(installation.team.id, installation);
    },
    fetchInstallation: async (InstallQuery) => {
      // 実際のデータベースから取得するために、ここのコードを変更
      return await database.get(InstallQuery.teamId);
    },
    storeOrgInstallation: async (installation) => { // Bolt-js v2.5.0 から追加
      // OrG 全体へのインストールに対応する場合はこのメソッドも追加
      // 実際のデータベースから取得するために、ここのコードを変更
      return await database.set(installation.enterprise.id, installation);
    },
    fetchOrgInstallation: async (InstallQuery) => { // Bolt-js v2.5.0 から追加
      // OrG 全体へのインストールに対応する場合はこのメソッドも追加
      // 実際のデータベースから取得するために、ここのコードを変更
      return await database.get(InstallQuery.enterpriseId);
    },
  },
});
```

（引用：[Bolt 入門ガイド#OAuth フローの実装](https://slack.dev/bolt-js/ja-jp/concepts#authenticating-oauth)）

上記のように App 初期化を行うと、Bolt はデフォルトで以下のパスを自動的に生成してくれます。

- アプリのインストールフローを完了した後の遷移先の URL である Redirect URL のためのパス: `slack/oauth_redirect`
- Slack アプリのダイレクトインストールのために Add to Slack ボタンを置く場合に指定できる URL のためのパス：`slack/install`

なお、この方法は組み込みの OAuth サポートモジュールは ExpressReceiver を使用している場合のみ利用可能です。
カスタムのレシーバーを実装して利用している場合は、Slack 公式の [OAuth ライブラリ](https://slack.dev/node-slack-sdk/oauth#slack-oauth) を利用する必要があります（前章 [Bolt を使わない場合](#bolt-を使わない場合) の方法のことです）

なお、[@slack/bolt@2.5.0](https://github.com/slackapi/bolt-js/releases/tag/%40slack/bolt%402.5.0) から **org wide app installations**（組織全体のアプリインストール）のサポートが追加されました。
`storeInstallation` だけでなく `storeOrgInstallation` が用意され、それに応じて `appId`, `teamId`, `enterpriseId` などの型の扱いが変わっているため、以前のバージョンの bolt を使っている方は注意してください。
（`storeInstallation` は `teamId` が含まれる一方、`storeOrgInstallation` は `enterpriseId` が含まれるなど）

## 実際にやってみる

では実際にやってみましょう。
ここでは、既に何らかのアプリを実装し、作成したワークスペースにのみインストールされている状態とします。
（まだ作成していない人は、
私は **スクワットをした回数を任意のチャンネルに報告するアプリ「スクワットしとく？」** を開発しました。

![assets/Slack_Training-result-normal.gif](assets/Slack_Training-result-normal.gif)

**世界中の人に健康になってほしいため**、これを複数ワークスペースにインストールさせるようにしてみましょう。

Slack [アプリ管理コンソール](https://api.slack.com/apps) へアクセスし、複数ワークスペースにインストールできるようにしたいアプリを開いてください。

### 1. Redirect URL の追加

まず、OAuth フローが完了した後の遷移先である Redirect URL を入力します。
左側のメニューから **OAuth & Permissions** を開き、**Redirect URLs** で `Add New Redirect URL` をクリックしてください。
URL に `https://<App サーバーのホスト名>/slack/oauth_redirect` と入力しましょう。
（Serverless Framework などでステージ管理している場合は `://<App サーバーのホスト名>/` の後に `dev/` などのパスが加わることがあるため、注意してください）。
Redirect URL を入力したら、`Done` をクリックし、`Save URLs` をクリックします（忘れやすいので注意！）

![Slack_Redirect-URLs.png](assets/Slack_Redirect-URLs.png)

なお、OAuth フローを有効化しない場合はこの画面でユーザーに要求する Bot スコープと User スコープを追加していました。
今後はアプリのインストール URL を生成する際に要求するスコープをパラメータで指定するため、アプリ管理コンソールでのスコープの指定は不要になります。

### 2. Bot ユーザーの作成

次に、Bot ユーザーを作成します。
左側のメニューから **App Home** を開き、**Your App’s Presence in Slack** で **App Display Name** の `Edit` ボタンをクリックします。
ここで `Display Name (Bot Name)` と `Default username` を適当に入力してください。

![Slack_App-Display-Name.png](assets/Slack_App-Display-Name.png)

### 3. アプリ配布の有効化

アプリの配布を有効化します。
左側のメニューから **Manage Distribution** を開き、**Share Your App with Other Workspaces** で `Activate Public Distribution` をクリックします。
配布にあたりいくつかチェック項目をクリアする必要がありますが、ほとんどは既にクリア済の状態になっています。
**"Remove Hard Coded Information"** にチェックが入っていないため、自身のアプリにシークレット情報などがハードコーディングされていないことを確認したうえで、チェックを入れましょう。
すべての項目にチェックを入れたら、一番下のアプリ有効化のボタンをクリックします。

![Slack_Activate-Public-Distribution.png](assets/Slack_Activate-Public-Distribution.png)

なお、配布に関しては以下のドキュメントを一読することをおすすめいたします。

- [Distributing your app publicly](https://api.slack.com/start/distributing/public#enabling)

### 4. 環境変数のコピー

環境変数をコピーします。
左側のメニューから **Basic Information** を開きます。
**App Credentials** で既にコピーしている **Signing Secret** に加え、**Client ID** と **Client Secret** をコピーします。

![Slack_App-Credentials.png](assets/Slack_App-Credentials.png)

そして以下のように環境変数を記載しているファイル（ここでは `.env`）に値をペーストします。
今まで利用していた Bot Token や User Token はもう不要です。

```diff:.env
export SLACK_SIGNING_SECRET=c6ccbf0f52ab8db407c5d3c5191438c8
- SLACK_BOT_TOKEN=XOXO-------
+export SLACK_CLIENT_ID=1162xxxxxx.1612xxxxxx
+export SLACK_CLIENT_SECRET=924axxxxxx
```

### 5. App 初期化の実装変更

以下のように `App` を初期化しているファイル（ここでは `handler.ts`）を書き換えてください。


```diff:handler.ts
 const app = new App({
   signingSecret: process.env.SLACK_SIGNING_SECRET!,
-  token: process.env.SLACK_BOT_TOKEN,
+  clientId: process.env.SLACK_CLIENT_ID,
+  clientSecret: process.env.SLACK_CLIENT_SECRET,
+  stateSecret: 'my-state-secret',
+  scopes: ['commands', 'chat:write'],
+  installationStore: {
+    storeInstallation: async installation => {
+      // TODO: 実際のデータベースに保存するために、ここのコードを変更
+      token_database[installation.team.id] = installation;
+      return Promise.resolve();
+    },
+    fetchInstallation: async installQuery => {
+      // TODO: 実際のデータベースから取得するために、ここのコードを変更
+      const installation = token_database[installQuery.teamId];
+      return Promise.resolve(installation);
+    },
+  },
 });
```

ここで、`stateSecret` は [CSRF（Cross Site Request Forgery）](https://owasp.org/www-community/attacks/csrf)対策のために、
認可リクエスト時にセッションに紐付けて発行されるパラメータ `state` をエンコードするための値です。
認可コードを受信したときに `state` をデコードするためにも使用され、`state` の値が同じかどうかを検証することで
 フロー中に悪意のある第三者から攻撃を受けたかどうかを検知します。
そのため、`stateSecret` は推測されにくいランダムな値（ただし一連の OAuth フロー中では同じ値）を発行するべきです。
ただ今回は説明を簡単にするために、固定文字列を指定しています。

これで、Slack の OAuth フローの実装は終了です。

あとはソースコードをサーバーにデプロイすれば、Bolt が生成してくれた `slack/install` パスを指定するだけで、
アプリのインストールボタンが表示された画面が表示されます。

`https://<App サーバーのホスト名>/slack/install`

ここでは、`ngrok` を利用して、ソースコードをサーバーにデプロイせずに、ローカルで動作を確認してみましょう。

ローカルでアプリを起動し、`ngrok` も起動した状態で、以下の URL をブラウザのアドレスバーに貼り付けます。

`https://foo.ngrok.io/dev/slack/install`（`foo` の箇所は `ngrok` を起動するときに発行されるサブドメインの文字列です）

すると、`Add to Slack` というボタンが表示されます。

![Slack_Bot-token-install.gif](assets/Slack_Bot-token-install.gif)

このボタンのリンクに、OAuth フローのスタートである認可リクエストが生成されて埋め込まれています。
ボタンをクリックすると、アプリのインストール画面と、アプリが要求するスコープの説明が表示されるはずです。
「許可」をクリックすると、ユーザーがアプリに認可情報を委譲したとみなし、OAuth のフローを通じてトークンを取得できます。

```javascript
{
  team: { id: 'T014XXXXXXX', name: 'irongineer' },
  enterprise: undefined,
  appId: 'A01JXXXXXXX',
  user: { id: 'U014XXXXXXX', token: '', scopes: [] },
  bot: {
    scopes: [ 'commands', 'chat:write' ],
    token: 'xoxb-1162xxxxxxxx',
    userId: 'U01HXXXXXXX',
    id: 'B01HXXXXXXX'
  },
  tokenType: 'bot'
}
```

あとはトークンを暗号化する等してセキュアな方法で DB に保存すれば OK です。
また、`fetchInstallation` で上記のオブジェクトと同じ形を返却する必要があります。
これを指定しておくと、Slack API を呼び出す際、自動で `fetchInstallation` を呼び出し、必要なトークンを取得してきてくれます。

気づいたかと思いますが、ここまでで前章 [Bolt を使わない場合](#bolt-を使わない場合) のような OAuth フロー（認可エンドポイントやトークンエンドポイントへのリクエスト、state の検証など）の実装を一切記述しておりません。
すべて Bolt が裏側でよしなにやってくれるのです。
すごくないですか？？？

Bolt のおかげで、晴れて私が開発した **「スクワットしとく？」** アプリを任意のワークスペースにインストールできるようになりました。
**これで世界中の開発者の QOL 爆上がり間違いなしです。**
めでたしめでたし。

## ユーザースコープが欲しい場合は…？

ある日、ふと追加の要件を思いつきました。

**「スクワットできたかどうかはそのときの調子（ステータス）を表す。スクワットの結果によって Slack のステータスも変更させたい」**

我ながら**天才的なアイデア**を思いついたのですが、Slack のステータスを API で変更するには、Bot スコープではなく User スコープの `users.profile:write`、および User トークンが必要であることが分かりました。
そのため、Bot スコープだけでなく User スコープを OAuth の認可リクエストで指定しなければいけません。

ここで注意が必要なのが、**`App` 初期化時に指定した `scopes` パラメータは、Bot スコープしか指定できないということです。**
では、User スコープを要求したい場合はどうすればいいかというと、Bolt のドキュメントには以下のように記載があります。

> アプリはすでにインストールされていて、さらにユーザーから追加の認可情報（例：ユーザートークンの発行）な場合や、何らかの理由で動的にインストール用の URL を生成したい場合は、ExpressReceiver を自前でインスタンス化し、それを receiver という変数に代入した上で receiver.installer.generateInstallUrl() を呼び出してください。詳細は [OAuth ライブラリのドキュメント](https://slack.dev/node-slack-sdk/oauth#generating-an-installation-url)の generateInstallUrl() を参照してください。

（引用：[Bolt 入門ガイド#OAuth フローの実装](https://slack.dev/bolt-js/ja-jp/concepts#authenticating-oauth)）

説明どおりやってみましょう。

> ExpressReceiver を自前でインスタンス化し、それを receiver という変数に代入

```javascript
const receiver = new ExpressReceiver({
  signingSecret: process.env.SLACK_SIGNING_SECRET!,
  clientId: process.env.SLACK_CLIENT_ID,
  clientSecret: process.env.SLACK_CLIENT_SECRET,
  stateSecret: 'my-state-secret',
  scopes: ['commands', 'chat:write'],
  installationStore: {
    storeInstallation: async installation => {
      // TODO: 実際のデータベースに保存するために、ここのコードを変更
      token_database[installation.team.id] = installation;
      return Promise.resolve();
    },
    fetchInstallation: async installQuery => {
      // TODO: 実際のデータベースから取得するために、ここのコードを変更
      const installation = token_database[installQuery.teamId];
      return installation;
    },
  },
});
const app = new App({
  receiver,
});
```

> receiver.installer.generateInstallUrl() を呼び出してください

```javascript
receiver.router.get('/slack/user_install', async (_req, res) => {
  try {
    // feel free to modify the scopes
    const url = await receiver.installer?.generateInstallUrl({
      scopes: ['commands', 'chat:write'],
      userScopes: ['users.profile:write'],
    });

    res.send(helpers.buildSlackUrl(url || ''));
  } catch (error) {
    console.log(error);
  }
});

const buildSlackUrl = (url: string): string =>
  `<a href=${url}><img alt=""Add to Slack"" height="40" width="139" src="https://platform.slack-edge.com/img/add_to_slack.png" srcset="https://platform.slack-edge.com/img/add_to_slack.png 1x, https://platform.slack-edge.com/img/add_to_slack@2x.png 2x" /></a>`;
}
```

ここでは、`slack/user_install` というカスタム HTTP ルートを追加し、そこで Bot スコープと User スコープを指定した `Add to Slack` ボタンの URL を生成しています。
このパスにアクセスしてみましょう。

`https://foo.ngrok.io/dev/slack/user_install`

先程と同様に `Add to Slack` というボタンが表示されます。
違うのは、このボタンをクリックしたあとに表示される権限リクエスト画面で、User スコープに関する説明が追加されていることです。

![Slack_User-token-install.gif](assets/Slack_User-token-install.gif)

これを許可すると、Bot トークンだけでなく User トークンも取得できます。

```json
{
  team: { id: 'T014XXXXXXX', name: 'irongineer' },
  enterprise: undefined,
  appId: 'A01JXXXXXXX',
  user: {
    id: 'U014XXXXXXX',
    token: 'xoxp-1162xxxxxxxx',
    scopes: [ 'users.profile:write' ]
  },
  bot: {
    scopes: [ 'commands', 'chat:write' ],
    token: 'xoxb-1162xxxxxxxx',
    userId: 'U01HXXXXXXX',
    id: 'B01HXXXXXXX'
  },
  tokenType: 'bot'
}
```

この User トークンと許可された User スコープ `users.profile:write` を利用することで、API によって Slack のカスタムステータスを変更することが可能になりました。
アプリを動かしてみると、**スクワットの結果に応じたステータス**が有効になっていることが分かります。

![Slack_Training-result-status.gif](assets/Slack_Training-result-status.gif)

User スコープの指定以外にも、OAuth の設定をカスタマイズすることは可能です。
気になる方は[OAuth ライブラリのドキュメント](https://slack.dev/node-slack-sdk/oauth#generating-an-installation-url)を読んでみてください。

## まとめ

- Slack アプリを複数ワークスペースにインストールするためには、OAuth フローの実装が必要
- Bolt を使えば OAuth の仕組みを意識せずに OAuth フローの実装が可能
- ユーザースコープ・トークンが欲しい場合は ExpressReceiver を自前でインスタンス化して、`generateInstallUrl()` を呼び出す
- その他のカスタマイズも可能（詳しくは公式ドキュメント参照）

## 感想

Bolt のおかげで OAuth 2.0 の仕組みを意識することなくアプリを複数のワークスペースにインストールできるようになりました。
もちろん、本来であればどういった仕組みで動いているのか理解した方がいいですが、誰でも簡単にできるようにした Bolt の功績は本当にすごいと思います。
みなさんも Bolt を使って楽しく Slack アプリを開発していきましょう！

## サンプルコード

https://github.com/irongineer/slack-bolt-oauth-install

動くサンプルを貼っておきます。
たまにしか利用しない Slack アプリなら常駐サーバーよりサーバレスの方がコストの節約になるため、サーバレス構成としています。
（上記のコード例で TODO としていた DB アクセス部分も AWS Lambda, DynamoDB を使って実装しています）。
README どおりにインストールすれば動くと思います。
Yarn と Serverless Framework がローカルに入っていて、AWS アカウントを持っていれば動くと思います。
また、ローカルで serverless-dynamodb-local を実行する必要があるため、Java のインストールが必要です。

なお、このサンプルでは簡略化のためトークンをそのまま DB に保存していますが、セキュリティを考慮して暗号化してから保存するようにしましょう。
（AWS を利用している場合は、KMS を利用して保存するのがよくある方法です）

サンプルコードについて質問や改善点がある方は、Issue や PR 送っていただけると幸いです。

# おまけ

## Tips

### FaaS で Bolt 製アプリを動かしたいとき

AWS Lambda や Google Cloud Functions のような FaaS (Function as a Service) 環境で Slack アプリを動かしたい場合、Bolt-js v2.0 から導入された `processBeforeResponse` というフラグを true に設定します。
これを設定することでリスナー関数が 3 秒以内に処理を完了できる前提で FaaS 環境で正常に動作させることができるようになります。
ただし、処理が 3 秒以上かかるとき（FaaS の Cold Start 問題を含む）はタイムアウトしてしまいます。
その場合はキューを挟むなどして時間のかかる処理を分け、先に HTTP レスポンスだけ返してからゆっくり処理するといった構成を考えるといいかもしれません。

詳しくは以下の記事が参考になります。

- [Slack アプリフレームワーク Bolt for JavaScript を Cloud Functions for Firebase で動かそう](https://qiita.com/seratch/items/3e44f58a7c8ab41f21f0)
- [Bolt for Python が FaaS での実行のために解決した課題](https://qiita.com/seratch/items/6d142a9128c6831a6718)
- [Bolt for JavaScript を使った Slack アプリ開発で知っておくと捗る 7 つのこと](https://qiita.com/seratch/items/dcedc7476eac5f681d30)

### ブロックの型を知りたいとき

Bolt for JavaScript は TypeScript で作成されているため基本的には型のサポートは手厚いです。
`app.shortcut<GlobalShortcut>()` や `app.action<BlockAction<StaticSelectAction>>()` といった感じに、
ジェネリクスを駆使して汎用的に型付けできるようにライブラリを作成してくれています。
ただし、メソッドと型の対応に関するドキュメントがないので、型を調べたいときはローカルにダウンロードした `@slack/bolt` の型定義ファイルを直接見ています・・・
あと、まだ対応していない型定義もちょいちょいあるので、そういった場合は型定義を自作するようにしています。

（ここらへんみなさんどうやってるんだろう・・・いい方法あったら教えて下さい oOo）

### ローカル環境で Slack アプリを開発したいとき

ローカル開発したい場合、[ngrok](https://ngrok.com/) が便利です。
ローカル PC 上で稼働しているサービスを外部公開できるため、実際にサーバーと通信しないと動作確認できないようなサービスでも通信テストをすることが可能です。
HTTPS 通信にも対応していて素敵。
唯一のデメリットは起動し直すたびにサブドメインが変更されることです・・・（有償プランであれば固定 URL を取得できるようです）

詳しくは以下の記事を見てみてください。

- [ngrokが便利すぎる](https://qiita.com/mininobu/items/b45dbc70faedf30f484e)

### ローカル環境で DynamoDB のテーブルの値を見たいとき

[NoSQL Workbench for DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/workbench.html) がいい感じです。
現在 DB に保持している値などもテーブルに可視化できるので、ローカル開発がはかどります。

![Slack_No-SQL-Workbench.png](assets/Slack_No-SQL-Workbench.png)

詳しい使い方は以下の記事をどうぞ。

- [DynamoDB用NoSQL WorkbenchでDynamoDB Local (Localstack) に接続する](https://kdnakt.hatenablog.com/entry/2019/11/14/080000)

## リファレンス

この記事を書くにあたって参考にしたドキュメント類です。

- Bolt
  - 公式ドキュメント
    - [The Bolt family of SDKs](https://api.slack.com/tools/bolt)
    - [Bolt 入門ガイド](https://api.slack.com/tools/bolt)
    - [Node.js 版 Slack SDK の OAuth フローのドキュメント](https://slack.dev/node-slack-sdk/oauth#generating-an-installation-url)
  - 記事・動画など
    - [Hello World, Bolt! ⚡️ Bolt フレームワークを使って Slack Bot を作ろう](https://qiita.com/tomomi_slack/items/93538f9a69eb4015f951)
    - [Bolt for JavaScript を使った Slack アプリ開発で知っておくと捗る 7 つのこと](https://qiita.com/seratch/items/dcedc7476eac5f681d30)
    - [Bolt を使って Slack でヘルプデスクチームとのやりとりを効率化するアプリをつくってみよう](https://qiita.com/seratch/items/9b5666479ad9f602e5ee)
    - [インタラクティブな連携を簡単に実現する最新の Slack SDK: Bolt for JavaScript / Java / Python](https://2020.bit-valley.jp/session/298) 動画なので取っつきやすいです
- OAuth
  - 公式ドキュメント
    - [Bolt 入門ガイド#OAuth フローの実装](https://slack.dev/bolt-js/ja-jp/concepts#authenticating-oauth)
    - [Installing with OAuth](https://api.slack.com/authentication/oauth-v2)
    - [Distributing your app publicly](https://api.slack.com/start/distributing/public#enabling)
    - [Node.js 版 Slack SDK の OAuth フローのドキュメント](https://slack.dev/node-slack-sdk/oauth#generating-an-installation-url)
  - 記事・動画など
    - [一番分かりやすい OAuth の説明](https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be)
    - [SlackのOAuth認証を使ってユーザ情報を取得してみた](https://qiita.com/dbgso/items/a95a3364d9a8c67f3387)

あと同僚が素敵なモジュールを作っていたので宣伝しておきます。

- [SlackのBlock Kitをより便利にするモジュールを作った話](https://qiita.com/sai2-dev/items/21a6aeec30c042377ae4)

# おわりに

この記事は [「Develop fun!」を体現する Works Human Intelligence #2 Advent Calendar 2020](https://qiita.com/advent-calendar/2020/whi-2) の 21 日目の記事でした。
明日は @harufumiy さんの記事です。尊敬する先輩の 1 人なので、楽しみにしています。
