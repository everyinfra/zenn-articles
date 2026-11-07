---
title: "Snowflake Cortex Agents MCP：SNS上の不満の声を返品データの横に取り込む方法"
emoji: "📊"
type: "tech"
topics: ["mcp", "datascience", "aiagents", "snowflake", "api"]
published: true
---

:::message
この記事は [EveryInfra ブログ](https://everyinfra.com/ja/blog/snowflake-cortex-agents-mcp-social-reviews) からの転載です（原文が正本です）。
:::

返品は、消費者向け小売業で最も大きな隠れたコストの1つです。全米小売業協会（NRF）は、2025年に米国の買い物客が返品する商品が約$849.9 billionにのぼると推計しています。これは売上の15.8%にあたり、オンライン購入では返品率が19.3%に達します（[NRFのプレスリリース](https://nrf.com/media-center/press-releases/consumers-expected-to-return-nearly-850-billion-in-merchandise-in-2025)）。多くのブランドは、販売データと返品データをすでにデータウェアハウスに取り込んでいます。一方、顧客が製品について公の場で発信している声まで取り込んでいるブランドはずっと少数です。この記事では、Snowflake上でこの2つを結び付けます。

きっかけはSnowflake Expedition 2026です。2026年11月3日、SnowflakeはSnowflake Intelligenceの新名称であるCoWorkと、Cortex Agentsの新機能を発表しました（[プレスリリース](https://www.snowflake.com/en/news/press-releases/expedition-2026-coco-cowork/)）。私たちが注目したのはMCPコネクタです。MCPコネクタを使うと、エージェントが外部のMCPサーバー上のツールを呼び出せます（[MCPコネクタのドキュメント](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors)）。ブランドチームなら、すぐに用途が思い浮かぶはずです。Amazonレビュー、Redditの投稿、TikTokのコメントを販売・返品テーブルの横に並べます。そのうえで、返品率と連動して増えている苦情のテーマはどれかをエージェントに尋ねます。

データ側は実際にテストしました。2026-11-06のシンガポール時間（SGT）00:16から00:36の間に、EveryInfra APIを12回呼び出し、すべてのリクエストとレスポンスを保存しています。本記事のSnowflake側は設定例のみです。このマシンにはSnowflakeアカウントがないため、SQL、プロシージャ、エージェントは実行していません。Snowflakeに関する事実はすべてsnowflake.comまたはdocs.snowflake.comに基づいており、閲覧日は参考文献に記載しています。

結論を先に述べます。現時点では、EveryInfraのMCPサーバーをSnowflakeのMCPコネクタとして追加することはできません。

SnowflakeはOAuthのみに対応しており、特定の方式でのクライアント登録が必要ですが、EveryInfraのサーバーはまだそれを提供していません。仮にOAuthに対応しても、コネクタではこの課題を解決できません。コネクタの結果は1つのチャット内にとどまり、テーブルには保存されないためです。実際に使えるのは、Snowflakeの外部アクセス統合を使う方法です。Pythonのストアドプロシージャが定期的にAPIを呼び出し、レビューをテーブルに書き込みます。そのうえでCortex Agentが、それらのテーブルを売上データや返品データと結合します。本記事で示していない内容も明確にしておきます。その夜のAmazonレビューの呼び出しはデータを返さず、返品データも手元にありません。そのため本記事で示すのは構築プランとコスト見積もりであり、どの不満が返品を引き起こしているかという分析結果ではありません。EveryInfraはSnowflakeと提携関係にありません。ここでのSnowflakeに関する事実はすべて、同社の公開ドキュメントとプレスリリースに基づいています。

> **要点**
>
> - SnowflakeのMCPコネクタはOAuthのみに対応しています。リモートサーバーが動的クライアント登録に対応している必要があります。対応していない場合は、コールバックURLに`https://identity.snowflake.com/oauth2/callback`を指定してクライアントを登録する必要があります。EveryInfraは現時点でどちらの条件も満たしていません。
> - 実現可能な方法は、ネットワークルール、APIキーを保存するシークレット、外部アクセス統合、Pythonプロシージャ、タスクの5つの要素で構成されます。エージェントはその後、外部ツールをその場で呼び出すのではなく、テーブルを読み取ります。
> - 12回の実際の呼び出しにかかった費用は合計$0.005016でした。Amazonレビューの呼び出し5回はすべてデータを返さず（503「一時的に利用不可」が4回、空の結果が1回）、いずれも課金されていません。
> - 2本の動画に付いたTikTokコメント273件のうち、製品の問題を述べたものは17件でした。最も多かったのはフタと中身のこぼれに関する問題（6件）です。比較動画のコメント200件では、キーワードルールが42件にマッチしましたが、実際に製品の問題だったのはそのうち4件（9.5%）だけでした。
> - 1ブランドあたりのAPI料金は月約$1.26です。Snowflake側で最も大きいコストは、HTTPレスポンスを待つ間も稼働するウェアハウスです。逐次実行では月約$40.54、10並列で呼び出すと月約$4.05になります。

![イラスト：左側には施錠された扉があり、その前に鍵穴に合わない鍵が置かれています。右側ではティール色の通用口が開いていて、ベルトコンベアがそこを通り、白紙のレビューカードの束をガラス製の保管キューブへ運んでいます。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/img-explain.webp)

*MCPコネクタの扉を開けるには登録済みのOAuthクライアントが必要で、APIキーでは開きません。外部アクセスの通用口は固定された1つのホストだけを許可し、データを直接テーブルに運びます。*

## SnowflakeがExpedition 2026で発表したSnowflake CoWorkとCortex Agents

このイベントの目玉はSnowflake CoWorkです。プレスリリースでは、ai.snowflake.comで提供されるビジネスユーザー向けのエージェントワークスペースと説明されています。Cortex Code（CoCo）は、開発者とデータエンジニア向けの対になるツールです（[Expedition 2026のプレスリリース](https://www.snowflake.com/en/news/press-releases/expedition-2026-coco-cowork/)、[CoWorkの製品ページ](https://www.snowflake.com/en/product/snowflake-cowork/)）。

同じ日に出た別のプレスリリースでは、Business OntologyとCortex AI Gatewayが発表されました。Business Ontologyは承認済みのビジネス用語の意味を保存し、その定義をエージェントに提供します。Cortex AI Gatewayは、エージェントがモデルにアクセスし、ツールを呼び出し、社内システム全体でアクションを実行する方法を統制します（[Business OntologyとAI Gatewayのプレスリリース](https://www.snowflake.com/en/news/press-releases/expedition-2026-business-ontology-ai-gateway/)）。どちらも後で扱う本題に関係します。「返品率」のような用語には合意済みの計算式が1つ必要です。そうでなければ、2つのエージェントが異なる答えを出してしまいます。

本記事の文脈では、CoWorkは人が質問する場所であり、実際の処理はCortex Agentsが担います。

CoWorkの各アシスタントは、Cortex Agentオブジェクトです。エージェントの仕様には、使用するモデル、指示、ツールが記載されます。ツールには、セマンティックビューに対してSQLを実行するCortex Analystや、テキストを検索するCortex Searchがあります。ほかにもコード実行、カスタムツール、MCPコネクタ、Web検索があります（[Cortex Agents overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)、[Create and manage agents](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-manage)）。つまり回答の品質は、主にテーブルとセマンティックビューで決まります。エージェントは質問をSQLに変換し、その結果を説明します。

以下の表は、ここで重要な2つの機能について、2026-11-06時点で確認したステータスを示しています。「記載なし」は、ドキュメントにもプレスリリースにもステータスが明記されていないことを意味します。

| 機能 | 2026-11-06時点のステータス | 出典 |
| --- | --- | --- |
| CoWork Automations | 記載なし | [Automationsのドキュメント](https://docs.snowflake.com/en/user-guide/snowflake-cortex/snowflake-cowork/automations) |
| Cortex AgentsのMCPコネクタ | ドキュメントにステータスの記載なし、OAuthのみ | [コネクタのドキュメント](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors) |

CoWorkのブログ記事では、プレビュー中の分析検索（analytical search）にも触れています。その例は本記事の課題に近く、先月最も多く挙がった顧客の問題は何か、というものです（[CoWorkのブログ](https://www.snowflake.com/en/blog/snowflake-cowork-profiles-automations-dashboards)）。Automationsの制限とアクセスルールについては、後述の週次メールのセクションで説明します。

分析検索は、すでにSnowflakeにあるドキュメントを対象に動作します。Cortex Searchで対象を絞り込み、AI関数とSQLで集計・比較します。本記事の課題には、もう1つステップが加わります。顧客の声はウェアハウスの外、つまりAmazon、Reddit、TikTokにあります。そのため最初の作業は、それを売上・返品と結合できる形で、スケジュールに沿って取り込むことです。

## Snowflake Cortex AgentsはMCPでEveryInfraに直接接続できるか

最初に思いつくのはこの方法でしょう。EveryInfraはすでにリモートMCPサーバーを運用しています（[EveryInfra MCPサーバー](https://everyinfra.com/ja/mcp-server)）。Cortex Agentsは外部のMCPサーバーを呼び出せるようになりました。それなら、サーバーのアドレスをSnowflakeに貼り付ければよいのではないでしょうか。Snowflakeの要件とEveryInfraのサーバーの仕様を1項目ずつ比較しました。答えは「できない」で、障害となるのはOAuthクライアントの登録方法です。

Snowflakeのドキュメントはこの点を明言しており、MCPサーバー接続でサポートするのはOAuthのみと記載しています。管理者はまず、OAuth設定を保持する`API_PROVIDER = external_mcp`のAPI統合を作成します。次に、その統合を参照する外部MCPサーバーオブジェクトを作成します（[MCPコネクタのドキュメント](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors)）。OAuth設定の指定方法は2つあります。

第1の方法は標準のOAuthです。リモートの認可サーバーにアプリを登録し、クライアントIDを取得します。クライアントにはシークレットを持たせることも、シークレットなしのパブリッククライアントにすることもできます。

そのアプリには、コールバックURL`https://identity.snowflake.com/oauth2/callback`も登録する必要があります。第2の方法は動的クライアント登録で、Snowflakeが自らリモートサーバーにクライアントIDを要求します。この場合、リモートサーバーがRFC 7591を実装している必要があります（[RFC 7591](https://datatracker.ietf.org/doc/html/rfc7591)）。あらかじめ用意されているコネクタはAtlassian、GitHub、Glean、Linear、Salesforceで、Atlassianの例では動的登録を使っています。

![SnowflakeのMCPコネクタに関するドキュメントページの「Key considerations and best practices」セクションのスクリーンショット。ティール色の枠で2行を囲んでいます。1つはCustom MCP serversで、利用可能なコネクタ一覧にないMCPエンドポイントにはカスタムサーバーを使うよう説明しています。もう1つはOAuth authenticationで、SnowflakeはMCPサーバー接続にOAuthのみをサポートすると説明しています。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/img-doc-oauth.webp)

*SnowflakeのMCPコネクタのドキュメントのスクリーンショット（2026-11-06、シンガポール時間に閲覧）。ティール色の枠は当社が追加したものです。* [docs.snowflake.com「Cortex Agents MCP connectors」](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors)

EveryInfraには2つのMCPアドレスがあります。通常使う`https://api.everyinfra.com/mcp`は、BearerヘッダーのAPIキーで認証します。2026-11-06の00:43（SGT）に、このアドレスのProtected Resource Metadataと認可サーバーメタデータをリクエストしたところ、どちらも404が返りました。つまりこのアドレスはOAuthディスカバリーに対応していません。フル版のアドレス`https://api.everyinfra.com/mcp/all`は挙動が異なります。同じ時刻に認証情報なしでリクエストすると、`WWW-Authenticate`ヘッダー付きの401が返りました。

このヘッダーはRFC 9728形式のリソースメタデータを指しており、そのメタデータでは`auth.everyinfra.com/oidc`が認可サーバーとして指定されています（[RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728)）。次に、RFC 8414に準拠した認可サーバーメタデータを読み取りました（[RFC 8414](https://datatracker.ietf.org/doc/html/rfc8414)）。このサーバーはS256によるPKCEと、シークレットを持たないパブリッククライアントに対応しています。しかし`registration_endpoint`の値が空のため、動的登録はできません。また、当社の公開ドキュメントには、顧客がOAuthクライアントを自分で登録できるセルフサービスのページもありません。

| Snowflakeの要件 | EveryInfra `/mcp`（APIキー） | EveryInfra `/mcp/all`（2026-11-06 00:43 SGTにテスト） |
| --- | --- | --- |
| OAuthのみ | いいえ、BearerのAPIキーを使用 | はい、RFC 9728メタデータ付きの401 |
| 動的クライアント登録（RFC 7591） | 該当なし | いいえ、メタデータに登録エンドポイントなし |
| クライアントとコールバックを登録する標準のOAuth | 該当なし | 公開ドキュメントにセルフサービス登録なし |
| ツールの機能のみ | はい、すべてのツールがMCPツール | 可能 |
| アンダースコアを含まないホスト名 | 可能 | 可能 |

MCPの仕様自体はOAuthを必須としていません。バージョン2025-06-18では、認可は任意とされています。サーバーが認可に対応する場合はRFC 9728とRFC 8414のメタデータを公開しなければなりませんが、動的登録は「MUST」ではなく「SHOULD」です（[MCP authorization, 2025-06-18](https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization)）。つまり、APIキーを使うサーバーも、OAuthのみのクライアントも仕様に準拠しています。両者が噛み合わないだけです。

Snowflakeには、外部MCPサーバーを追加できる場所がもう1つあります。Cortex AI Gatewayでは、管理者がカスタムMCPサーバーを追加できます。こちらのドキュメントでも、OAuth、`/mcp`で終わるURL、ACCOUNTADMINロールが必要とされています（[AI Gateway custom apps](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-ai-gateway/tools/admin-custom-apps)）。EveryInfraのOAuth対応アドレスは`/mcp/all`で終わり、`/mcp`で終わるアドレスはキーしか受け付けません。両方の条件を満たすアドレスはありません。

仮に登録の問題が解消されたとしても、コネクタはこの用途に合いません。各ユーザーがCoWorkで「Connect」をクリックし、接続先のページでログインしてアクセスを承認します。その後、エージェントはチャット中にリモートのツールを呼び出し、結果はそのチャット内にとどまります（[Use MCP connectors in Snowflake CoWork](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors)）。質問する人は全員、自分のEveryInfraアカウントが必要です。さらに重要なのは、レビューが一切テーブルに保存されないことです。そのためエージェントは、3か月分のレビューと3か月分の返品データを結合できません。

トレンドに関する質問には履歴が必要で、履歴はテーブルにあります。コネクタが向いているのは別の種類の質問で、たとえば「この動画のコメント欄で今、何が議論になっているか」といったものです。こうした質問は一度きりで、データの保存は必要ありません。

Snowflakeを外部のデータAPIと接続する方法は4つ見つかりました。以下の表では、「データがテーブルに保存されるか」「その方法にOAuthが必要か」という2つの基準で比較しています。

| 方法 | データがテーブルに保存される | OAuthが必要 | 用途 | 出典 |
| --- | --- | --- | --- | --- |
| MCPコネクタ（外部MCPサーバーオブジェクト） | いいえ、チャット内のみ | 可能 | 一度きりの質問 | [コネクタのドキュメント](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors) |
| 外部アクセス統合、ストアドプロシージャ、タスク | 可能 | いいえ、キーはシークレットに保存 | 日次ロード（本記事の推奨） | [External network access](https://docs.snowflake.com/en/developer-guide/external-network-access/external-network-access-overview) |
| Snowflake外部で収集し、Snowpipeで取り込む | 可能 | 不要 | すでにコレクターを運用しているチーム | [Snowpipe](https://docs.snowflake.com/en/user-guide/data-load-snowpipe-intro) |
| Cortex Code Agent SDKでの外部MCP | いいえ、コードで書き戻さない限り | いいえ、ヘッダーを利用可能 | コードでテストする開発者 | [Agent SDK custom tools](https://docs.snowflake.com/en/user-guide/cortex-code-agent-sdk/mcp-custom-tools) |

第4の方法では、Snowflakeの開発者向けツールを使い、リクエストヘッダーにAPIキーを入れられます（[Agent SDKのドキュメント](https://docs.snowflake.com/en/user-guide/cortex-code-agent-sdk/mcp-custom-tools)）。この方法は開発者が書くコード内で使うもので、CoWorkのビジネス向けインターフェースからは使えません。本記事では以降、第2の方法を採用します。Snowflakeの通常のアクセス制御下で、データを定期的にテーブルへ書き込めるためです。

## 外部アクセスとストアドプロシージャで実現する方法（設定例）

全体の流れは1枚の図に収まります。EveryInfra APIが、Amazon、Reddit、TikTokから公開レビューとコメントを収集します。Snowflake内では、定期実行タスクが1日1回ストアドプロシージャを呼び出します。プロシージャは外部アクセス統合を通じてAPIを呼び出し、生のレスポンスをそれぞれ保存したうえで、レビューテーブルに展開します。自社の売上データとAmazon FBAの返品レポートは、現在と同じように、さらに2つのテーブルへ読み込まれます。その上にセマンティックビューとCortex Agentを置き、ビジネスユーザーはCoWorkで質問します。

![左から右へ5つの段階を示すアニメーションのフロー図。第1段階はEveryInfra APIで、その下にamazon/reviews、reddit/search、tiktok/commentsが並んでいます。第2段階はSnowflakeの外部アクセス統合で、api.everyinfra.comのみを許可するルールと、キーを保持するシークレットが示されています。第3段階は2つの定期実行タスクで、一方が非同期リクエストを送信し、もう一方が結果を回収します。第4段階は4つのテーブルで、レビュー、ソーシャルメディア上の言及、売上、返品です。第5段階はセマンティックビューとCortex Agentで、その横に「返品率の上昇と連動して増える不満のテーマは何か」と尋ねるチャットの吹き出しがあります。矢印に沿って小さなティール色の点が動きます。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/fig-02-load-flow.ja.light.svg)

外部アクセス統合は、プロシージャや関数からインターネットにアクセスするためのSnowflake標準の仕組みです。3つのオブジェクトを使います。ネットワークルールは、コードがアクセスできるホストとポートを指定します。シークレットは認証情報を保持します。統合がこの2つを結び付けます。この統合の使用権限を特定のロールに付与します（[External network access overview](https://docs.snowflake.com/en/developer-guide/external-network-access/external-network-access-overview)、[Creating and using an external access integration](https://docs.snowflake.com/en/developer-guide/external-network-access/creating-using-external-network-access)）。

このルールで許可するのは`api.everyinfra.com`のみです。誰かがプロシージャのコードを書き換えても、ほかのホストにはアクセスできません。キーはシークレットオブジェクトに保存されるため、コードに平文で含まれることはありません。プロシージャを実行するロールも、キーを読み取れません。以下の例では、`voc`がデータベース名です。「voice of the customer（顧客の声）」の略です。

```text
-- 1. Allow outbound traffic to one host only
CREATE OR REPLACE NETWORK RULE voc.raw.everyinfra_egress
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('api.everyinfra.com');

-- 2. Store the API key. Create a separate key in the EveryInfra console
--    and give it its own per-minute call limit.
CREATE OR REPLACE SECRET voc.raw.everyinfra_key
  TYPE = GENERIC_STRING
  SECRET_STRING = '<EveryInfra API key>';

-- 3. Tie the rule and the secret together
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION everyinfra_access
  ALLOWED_NETWORK_RULES = (voc.raw.everyinfra_egress)
  ALLOWED_AUTHENTICATION_SECRETS = (voc.raw.everyinfra_key)
  ENABLED = TRUE;

-- 4. Grant it only to the loading role
GRANT USAGE ON INTEGRATION everyinfra_access TO ROLE voc_loader;
GRANT READ ON SECRET voc.raw.everyinfra_key TO ROLE voc_loader;
```

構文はSnowflakeのSQLリファレンスに従っています（[CREATE NETWORK RULE](https://docs.snowflake.com/en/sql-reference/sql/create-network-rule)、[CREATE SECRET](https://docs.snowflake.com/en/sql-reference/sql/create-secret)、[CREATE EXTERNAL ACCESS INTEGRATION](https://docs.snowflake.com/en/sql-reference/sql/create-external-access-integration)）。統合の作成にはアカウントレベルの権限が必要なため、通常は管理者が一度だけ行います。その後、ロード用のロールに必要なのは、統合のUSAGE権限とシークレットのREAD権限だけです。Snowflakeのサンプルページには、シークレットを使って外部APIを呼び出すプロシージャの完全な例が掲載されています（[External network access examples](https://docs.snowflake.com/en/developer-guide/external-network-access/external-network-access-examples)）。

プロシージャには、請求額を左右する設計上の選択が1つあります。各結果を待つか、リクエストを送信して後で取りに戻るかです。EveryInfra APIはどちらにも対応しています。

同期呼び出しでは、データが届くまで接続を開いたままにします。テストでは、最も遅かったAmazonレビューの呼び出しで67秒間接続が保持されました。非同期呼び出しでは、リクエストボディに`"mode": "async"`を追加します。APIは約2秒でステータス202とジョブIDを返します。結果は後から`GET /api/v1/jobs/{job_id}`で取得します（[ドキュメント](https://everyinfra.com/ja/docs)）。

Snowflakeでは、この差がそのままコストになります。プロシージャはウェアハウス上で実行され、ウェアハウスは稼働した秒数すべてに課金されます。待っている1秒1秒が課金対象です。そこで処理を2つのプロシージャと2つのタスクに分けました。一方のプロシージャは送信だけを行い、数秒で終わります。もう一方は完了したジョブを回収します。

次の表は、この構成で使うSnowflakeオブジェクトと、それぞれの役割、通常の作成者をまとめたものです。

| オブジェクト | 用途 | 通常の作成者 |
| --- | --- | --- |
| ネットワークルール`everyinfra_egress` | 1つのホストへの送信トラフィックを許可 | アカウント管理者 |
| シークレット`everyinfra_key` | EveryInfraのAPIキーを保持 | アカウント管理者 |
| 外部アクセス統合`everyinfra_access` | ルールとシークレットを結び付ける | アカウント管理者 |
| プロシージャ`submit_pulls` | ウォッチリストの各行について非同期ジョブを送信 | データエンジニア |
| プロシージャ`collect_results` | 完了したジョブを生データテーブルに回収 | データエンジニア |
| タスク`t_submit`と`t_collect` | 2つのプロシージャをスケジュール実行 | データエンジニア |
| セマンティックビュー`voc_returns` | テーマ、週、返品指標を定義 | アナリティクスエンジニア |
| `VocAnalyst`を設定したエージェント | CoWorkで質問に回答 | アナリティクスエンジニア |

```text
CREATE OR REPLACE PROCEDURE voc.raw.submit_pulls()
  RETURNS VARIANT
  LANGUAGE PYTHON
  RUNTIME_VERSION = '3.11'
  PACKAGES = ('snowflake-snowpark-python', 'requests')
  HANDLER = 'run'
  EXTERNAL_ACCESS_INTEGRATIONS = (everyinfra_access)
  SECRETS = ('ei' = voc.raw.everyinfra_key)
AS
$$
import json, requests, _snowflake

API = "https://api.everyinfra.com/api/v1/social"

def run(session):
    key = _snowflake.get_generic_secret_string("ei")
    headers = {"Authorization": f"Bearer {key}", "Content-Type": "application/json"}
    # watchlist: one row per product URL, video URL or search keyword
    rows = session.sql(
        "SELECT watch_id, platform, action, params FROM voc.raw.watchlist WHERE active"
    ).collect()
    submitted = 0
    for r in rows:
        body = {"platform": r["PLATFORM"], "action": r["ACTION"],
                "params": json.loads(r["PARAMS"]), "mode": "async"}
        resp = requests.post(API, headers=headers, json=body, timeout=30)
        payload = resp.json()
        session.sql(
            """INSERT INTO voc.raw.jobs
               (watch_id, job_id, http_status, charged_usd, first_payload, submitted_at)
               SELECT ?, ?, ?, ?, PARSE_JSON(?), CURRENT_TIMESTAMP()""",
            params=[r["WATCH_ID"], payload.get("job_id"), resp.status_code,
                    (payload.get("billing") or {}).get("charged_usd"), json.dumps(payload)],
        ).collect()
        submitted += 1
    return {"submitted": submitted}
$$;
```

このプロシージャは、SnowflakeのPythonプロシージャのガイドに沿っています。`SECRETS`句でシークレットオブジェクトを名前に対応付け、コードでは`_snowflake.get_generic_secret_string`で読み取ります。`PACKAGES`句には外部パッケージを列挙します（「[Python stored procedures overview](https://docs.snowflake.com/en/developer-guide/stored-procedure/python/procedure-python-overview)」）。取り入れる価値のある点が2つあります。第1に、最初のレスポンスは全体を保存してください。正確な課金額がドルで記載されているため、毎月EveryInfraの請求ページと照合できます。

第2に、プロシージャ内で同じ対象を再試行しないでください。APIは不正なパラメータ、一時的に利用できない対象、空の結果には課金せず、その理由をレスポンスに記載します。再試行は翌日の実行に任せましょう。実際、Amazonの503レスポンスにも、同じ商品ですぐに再試行しても成功する見込みは低いと書かれていました。

回収側も同じ構成です。`voc.raw.jobs`から未回収のジョブIDを読み取り、それぞれのステータスを問い合わせます。`running`または`finalizing`のジョブはスキップします。完了した結果は`voc.raw.api_responses`に書き込み、回収済みとしてマークします。私たちのテストでは、当初`running`だけを未完了として扱っていました。その結果、まだfinalizing中のジョブを1件、早すぎるタイミングで読み取ってしまいました。チェックには両方の状態を含めてください。

送信タスクは毎日シンガポール時間の02:00に実行するようスケジュールします。回収タスクは02:20と03:20に実行し、1回目で取りこぼしたものを2回目で拾います。回収タスクを数分おきに実行するのは避けてください。ウェアハウスは再開のたびに最低60秒分が課金されるため、頻繁に実行しても大半はアイドル時間への支払いになります。自動サスペンドも、デフォルトの10分ではなく60秒にすることをおすすめします。

```text
CREATE OR REPLACE TASK voc.raw.t_submit
  WAREHOUSE = voc_xs_wh
  SCHEDULE = 'USING CRON 0 2 * * * Asia/Singapore'
AS
  CALL voc.raw.submit_pulls();

CREATE OR REPLACE TASK voc.raw.t_collect
  WAREHOUSE = voc_xs_wh
  SCHEDULE = 'USING CRON 20 2,3 * * * Asia/Singapore'
AS
  CALL voc.raw.collect_results();

ALTER TASK voc.raw.t_collect RESUME;
ALTER TASK voc.raw.t_submit RESUME;
```

タスクの構文は「[CREATE TASK](https://docs.snowflake.com/en/sql-reference/sql/create-task)」と「[introduction to tasks](https://docs.snowflake.com/en/user-guide/tasks-intro)」を参照してください。`WAREHOUSE`を削除してサーバーレスタスクにすることもでき、その場合Snowflakeは実際のコンピュート量で課金します。Service Consumption Tableでは、サーバーレスタスクの乗数は0.9とされています（「[Service Consumption Table](https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf)」）。

## 4つのテーブル：レビュー、ソーシャル上の言及、売上、返品

テーブル設計では3つのルールに従います。生のレスポンスは受け取ったままの形で保存します。展開後のテーブルには、分析に必要なフィールドだけを残します。個人データは可能な限り削除します。APIはレビュー投稿者の名前とアバターを返しますが、返品率の分析では一切使いません。削除すればストレージを節約でき、プライバシー上の懸念もなくなります。以下は4つのコアテーブルと1つのマッピングテーブルです。売上と返品は自社システムから取得するデータなので、構造のみを示します。

```text
-- Raw responses: one row per API response, stored as VARIANT
CREATE TABLE IF NOT EXISTS voc.raw.api_responses (
  job_id        STRING,
  platform      STRING,
  action        STRING,
  fetched_at    TIMESTAMP_TZ,
  payload       VARIANT
);

-- Amazon reviews (expanded from amazon/reviews)
CREATE TABLE IF NOT EXISTS voc.core.reviews (
  review_id            STRING PRIMARY KEY,
  asin                 STRING,
  country              STRING,
  rating               NUMBER(3,1),
  title                STRING,
  body                 STRING,
  posted_at            TIMESTAMP_TZ,
  is_verified_purchase BOOLEAN,
  helpful_count        NUMBER,
  variant              STRING,
  fetched_at           TIMESTAMP_TZ
);

-- Social mentions: TikTok comments and Reddit posts in one table
CREATE TABLE IF NOT EXISTS voc.core.social_mentions (
  platform      STRING,          -- tiktok or reddit
  source_id     STRING,          -- comment or post ID
  source_url    STRING,          -- video URL or post URL
  community     STRING,          -- Reddit community, empty for TikTok
  product_key   STRING,          -- key from product_map
  body          STRING,
  like_count    NUMBER,
  reply_count   NUMBER,
  is_reply      BOOLEAN,
  posted_at     TIMESTAMP_TZ,
  fetched_at    TIMESTAMP_TZ,
  themes        ARRAY,           -- labels from AI_CLASSIFY
  PRIMARY KEY (platform, source_id)
);

-- Daily sales from your order system
CREATE TABLE IF NOT EXISTS voc.core.sales_daily (
  sale_date     DATE,
  sku           STRING,
  asin          STRING,
  units_ordered NUMBER,
  PRIMARY KEY (sale_date, sku)
);

-- FBA customer returns report (GET_FBA_FULFILLMENT_CUSTOMER_RETURNS_DATA)
CREATE TABLE IF NOT EXISTS voc.core.fba_returns (
  return_date           TIMESTAMP_TZ,
  order_id              STRING,
  sku                   STRING,
  asin                  STRING,
  fnsku                 STRING,
  quantity              NUMBER,
  fulfillment_center_id STRING,
  detailed_disposition  STRING,
  reason                STRING,
  status                STRING,
  customer_comments     STRING,
  return_themes         ARRAY
);

-- Product map: which ASINs, SKUs, videos and keywords belong to one product
CREATE TABLE IF NOT EXISTS voc.core.product_map (
  product_key   STRING,
  asin          STRING,
  sku           STRING,
  tiktok_urls   ARRAY,
  reddit_terms  ARRAY
);
```

返品テーブルは、Amazon SP-APIのFBAカスタマー返品レポートの構成をそのまま写しています。レポートタイプは`GET_FBA_FULFILLMENT_CUSTOMER_RETURNS_DATA`です。フィールドには、返品日、注文ID、SKU、ASIN、FNSKU、商品名、数量、フルフィルメントセンター、詳細な処分状況、理由、ステータス、ライセンスプレート番号、顧客コメントが含まれます。ドキュメントによると、Amazonはこのレポートを毎日更新しています（「[SP-API FBA report types](https://developer-docs.amazon/sp-api/docs/report-type-values-fba)」）。

ここで最も重要なのは顧客コメントの列です。TikTokのコメントと同じく自由記述のテキストです。両方を同じテーマリストで分類すれば、2つのデータをテーマ単位で突き合わせられます。理由の列はAmazonの固定コードで、「蓋から漏れる」のようなテーマに比べてはるかに粒度が粗くなります。理由コードだけでは、どのテーマが返品につながっているかはわかりません。

次の表は、APIのフィールドとテーブルの列の対応です。フィールド名は、2026年11月6日に`GET /api/v1/social/catalog`で参照したEveryInfraのカタログに基づいています（[カタログ](https://api.everyinfra.com/api/v1/social/catalog)）。

| 機能 | APIフィールド | テーブルの列 | 備考 |
| --- | --- | --- | --- |
| `amazon/reviews` | `review_id`、`product_asin`、`country` | `reviews.review_id`、`asin`、`country` | レビューIDで重複排除 |
| `amazon/reviews` | `rating`、`title`、`text`、`posted_at` | `rating`、`title`、`body`、`posted_at` | 評価は1〜5 |
| `amazon/reviews` | `is_verified_purchase`、`helpful_count`、`variant` | 同じ名前 | 色とサイズは`variant`に含まれる |
| `amazon/reviews` | `author_name`、`author_id`、`author_avatar_url` | 保存しない | 分析に不要な個人データ |
| `tiktok/comments` | `id`、`text`、`like_count`、`reply_count`、`is_reply`、`posted_at` | `social_mentions`の対応する列 | 各トップレベルコメントに最大2件の返信が付く |
| `tiktok/comments` | `author_username`、`author_id` | 保存しない | 同上 |
| `reddit/search` | `id`、`url`、`title`と`text`、`community`、`like_count`、`comment_count` | `source_id`、`source_url`、`body`、`community`、`like_count`、`reply_count` | タイトルと本文を1つのテキストに結合 |
| `reddit/search` | `author_username` | 保存しない | 同上 |

展開処理はSQLだけで実装できます。`api_responses`からプラットフォームごとに行を選択し、`payload:results`に`LATERAL FLATTEN`を適用して、IDをキーに対象テーブルへ`MERGE`します。同じ動画を再取得すると、既存のコメントはいいね数が更新され、新しいコメントは追加されます。この記事の後半で見るように、数か月後に同じ動画を再取得することが、この構成で最も役に立つステップです。

## 実際の1往復を追う：REST、非同期、MCP

Snowflake側は実行できませんでした。ただし、プロシージャは、私たちが手元のマシンで使ったのと同じAPIに同じリクエストを送り、同じ形式のレスポンスを受け取ります。2026年11月6日00:16〜00:36（SGT）に、テスト用のAPIキーで12回の呼び出しを行いました。この記事で扱う3つの機能に加え、対象選びに使った2回の検索も含みます。各呼び出しについて、リクエスト、開始・終了時刻、HTTPステータス、所要時間、課金額を記録しました。

| # | SGT | 機能 | モード | 結果 | 所要時間 | 課金額 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 00:16:52 | `amazon/search`、キーワード：portable blender | 同期 | 商品10件、うち9件がNinja | 42.14 s | $0.002236 |
| 2 | 00:17:41 | `amazon/reviews`、商品B0C2FF4JD1 | 同期 | 503、一時的に利用不可（課金なし） | 60.12 s | $0 |
| 3 | 00:18:41 | `amazon/reviews`、商品B0CN3S2SW2 | 同期 | 503 | 58.57 s | $0 |
| 4 | 00:19:47 | `amazon/reviews`、商品B09B8V1LZ3 | 同期 | 0件、空の結果のため返金 | 67.25 s | $0 |
| 5 | 00:20:55 | `reddit/search`、portable blender、新着順 | 同期 | 投稿14件 | 6.56 s | $0.000556 |
| 6 | 00:21:13 | `tiktok/search`、クエリ「ninja blast portable blender review」 | 同期 | 動画10本 | 3.78 s | $0.000556 |
| 7 | 00:21:21 | `tiktok/comments`、安物と高級品の比較テスト動画 | 同期 | コメント200件 | 52.30 s | $0.000556 |
| 8 | 00:22:13 | `walmart/reviews`、商品ページ | 同期 | 0件、空の結果のため返金 | 54.34 s | $0 |
| 9 | 00:24:22 | `amazon/reviews`、商品B0D2YYHN1B | 同期 | 503 | 19.48 s | $0 |
| 10 | 00:25:26 | `reddit/search`、ninja blast | MCP | 投稿7件 | 7.62 s | $0.000556 |
| 11 | 00:27:36 | `tiktok/comments`、1製品の試用動画 | 非同期 | 202、約100秒後に完了、コメント73件 | 送信1.95秒 | $0.000556 |
| 12 | 00:36:23 | `amazon/reviews`、商品B0C2FF4JD1（2回目） | 同期 | 503 | 38.42 s | $0 |

12回の呼び出しのうち6回が課金対象で、合計は$0.005016でした。残りの6回は503または空の結果を返し、いずれも課金されませんでした。呼び出し8のWalmart商品URLは商品名から推測したものなので、空の結果は私たちのミスです。それでも、空の結果に対する返金が[Walmartレビュー](https://everyinfra.com/ja/walmart-api/reviews)にも適用されることを確認できました。

![当社APIの実際のレスポンスのスクリーンショット。明るい配色のコードカードで、3つのパートに分かれています。パート1は非同期での送信です。platformにtiktok、actionにcomments、modeにasyncを指定してPOST https://api.everyinfra.com/api/v1/socialを呼び出すと、1.95秒でステータス202が返り、レスポンスにはstate running、ジョブID、ポーリングURLが含まれています。青緑色の枠は課金額0.000556米ドルを示しています。パート2は完了したジョブの抜粋で、state succeeded、コメント73件、最初の2件のコメントの投稿時刻、いいね数、本文が表示され、authorフィールドは非表示になっています。パート3はAmazonレビューのレスポンスで、ステータス503、reason access_blocked、900秒後の再試行指示が示され、課金額は0です。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/img-api-async.webp)

*2026年11月6日00:27（SGT）の非同期送信と結果、および00:17の503レスポンスのスクリーンショット。投稿者データは非表示にしています。* [EveryInfra APIレスポンス](https://api.everyinfra.com/api/v1/social)

スクリーンショットのパート1は、送信プロシージャが受け取る内容です。1.95秒で202とジョブIDが返り、レスポンスには送信時点の課金額$0.000556が記載されています。パート2は回収プロシージャが受け取る内容です。stateは`running`から`finalizing`、`succeeded`へと遷移しました。約8秒ごとに計12回ポーリングし、ジョブは送信から約100秒後に完了しました。

パート3は失敗例です。ステータスは503、reasonは`access_blocked`、推奨される再試行は900秒後、課金額は0です。プロシージャは、ステータスコードとこれらのフィールドで分岐するだけで済みます。

呼び出し10は比較用です。同じReddit検索を、MCPツール`everyinfra_call_api`経由で実行しました。7.62秒で7件の投稿が返り、ツール結果は18,619文字、レスポンス全体は19,490バイトでした。Cortex AgentがMCPコネクタ経由でこの呼び出しを行うと、この18,619文字が入力トークンとしてモデルのコンテキストに入ります。同じデータをテーブルにロードしておけば、エージェントのコストは読み取る数件分だけです。私たちが「チャットのたびにライブで取得する」より「先にロードし、後で質問する」を選ぶのはこのためです。

![横棒グラフ。実際の呼び出し1回につき1本、計12本の棒が0〜120秒の軸上に並んでいます。青緑色の6本は課金された呼び出しで、Amazon検索42.14秒、Reddit検索6.56秒、TikTok検索3.78秒、TikTokコメント（同期）52.30秒、MCP経由のReddit検索7.62秒、TikTokコメントの非同期送信1.95秒です。灰色の6本は課金されなかった呼び出しで、19〜67秒のAmazonレビューのエラー4件とAmazonの空の結果1件、54.34秒のWalmartの空の結果です。サブタイトルには2026年11月6日00:16〜00:36（シンガポール時間）と記載されています。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/fig-03-call-latency-bar.ja.light.svg)

このレイテンシのグラフが、Snowflakeのコストモデルを決めます。データサイズで見れば、3つの機能はどれも軽量です。コメント200件を取得したTikTokコメントの呼び出し1回で71,228バイト、Reddit検索1回で約30,000バイトでした。一方、待ち時間で見ると、レビューやコメントの同期呼び出しには50〜70秒かかることがよくあります。

ウェアハウスは秒単位で課金され、再開のたびに最低60秒分が課金されます（「[Warehouses overview](https://docs.snowflake.com/en/user-guide/warehouses-overview)」、「[Understanding compute cost](https://docs.snowflake.com/en/user-guide/cost-understanding-compute)」）。同期呼び出しの間、ウェアハウスはほぼすべての時間をネットワーク待ちに費やします。これをドルに換算した結果は、コストのセクションで示します。

## テスト1：製品の問題に言及したTikTokコメントは何件あるか

対象には、多くの読者になじみのあるカテゴリーとしてポータブルブレンダーを選びました。呼び出し1では`amazon/search`で「portable blender」を検索しました。上位10件のうち9件がNinja製品で、価格は$19.95〜$59.99でした（[AmazonデータAPI](https://everyinfra.com/ja/amazon-api)、[Amazon検索](https://everyinfra.com/ja/amazon-api/search)）。呼び出し6では`tiktok/search`で「ninja blast portable blender review」を検索しました。返ってきた10本の動画からコメント数の多い2本を選び、`tiktok/comments`でコメントを取得しました（[TikTokデータAPI](https://everyinfra.com/ja/tiktok-api)、[TikTokコメントAPI](https://everyinfra.com/ja/tiktok-api/comments)、[TikTok検索API](https://everyinfra.com/ja/tiktok-api/search)）。

1本目は、2025年7月27日に投稿された「安物と高級品」のブレンダー比較テスト動画です。取得時点で再生回数は2,586,703回、コメントは251件でした。受け取ったのは、1回の呼び出しの上限である200件です。2本目は2026年1月20日に投稿された1製品の試用動画で、再生回数は208,916回、コメントは98件でした。受け取ったのは73件で、うち17件は返信でした。この記事では、どちらのクリエイターも名前を伏せています。

| 動画 | 投稿日 | ビュー | 動画のコメント数 | 取得したコメント数 | リプライ | 製品の問題 | 割合 |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 安物と高級品の比較テスト | 2025-07-27 | 2,586,703 | 251 | 200 | 23 | 4 | 2.0% |
| 1製品の試用 | 2026-01-20 | 208,916 | 98 | 73 | 17 | 13 | 17.8% |

まず、よく使われる簡易的な手法として、キーワードリストを試しました。キーワードはbattery、leak、broke、returned、expensive、too smallです。いずれかの語に一致したコメントを不満として数えました。1本目の動画では、200件中42件がリストに一致しました。次に、すべてのコメントを手作業で読みました。42件のうちブレンダーの問題を述べていたのは4件だけで、キーワードルールの正解率は9.5%でした。

理由はすぐにわかります。「安いものと高いものを比べる」動画のコメント欄は、大半がその企画へのジョークや、ほかの製品でも安いものと高いものを試してほしいというリクエストです。「expensive」や価格に関する語に一致したコメントは30件ありましたが、そのうちブレンダーが価格に見合わないと述べていたのは7件だけでした。Snowflakeでは、テーマのラベルはモデルに付けさせる必要があり、その前に「製品の問題」の定義を文章で決めておく必要があります。

表は、1本目の動画でキーワードリストに一致した42件のコメントに私たちが手作業で付けたラベルを示しています。

| 手作業で付けたラベル | コメント |
| --- | --- |
| 製品に関するシグナルではない（ジョーク、ほかの話題） | 29 |
| 価格への不満のみ | 7 |
| 製品の問題 | 4 |
| 判断できない | 2 |
| キーワード一致の合計 | 42 |

手作業でのラベル付けには厳格なルールを設けました。この製品に問題があったと述べているコメントだけをカウントします。問題の種類は、フタの不具合や液漏れ、バッテリーの持ちの悪さ、氷を砕く力の弱さ、部品の欠品、洗いにくさ、短期間での故障です。「買う価値がない。返品した」といった全般的な否定もカウントします。価格への不満、ブランド間の比較、ジョーク、絵文字はカウントしません。このルールで見ると、2本の動画の結果は大きく異なります。

![2本のバーで構成された横向きの積み上げ棒グラフです。上のバーは再生回数2.59Mの「安い商品と高い商品の比較テスト」動画で、コメント200件中4件（2.0%）が商品の問題に言及しています。下のバーは再生回数209Kの「単品の試着」動画で、コメント73件中13件（17.8%）が商品の問題に言及しています。ティール色は商品の問題に関するコメント、灰色はそれ以外のすべてのコメントを示します。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/fig-04-issue-share-bar.ja.light.svg)

再生回数が10倍の動画では、製品の問題に関するコメントは2.0%にとどまります。規模の小さい試用動画では17.8%です。理由は視聴者層にあります。比較動画には娯楽として見る人が集まります。試用動画には製品を持っている人や購入を検討している人が集まり、「うちのも漏れた」と書くのはそうした人たちです。Snowflakeの設計では、1本あたりのコメント数よりも、`product_map`でどの動画を選ぶかのほうが重要です。

1回の呼び出しあたり$0.000556という同じ料金で、試用動画からは約9倍のシグナルが得られました。この選定も自動化できます。`tiktok/search`で検索し、キャプションに「review」「trying」「honest」などの語を含み、コメント数が数十〜数百件の動画を残します。

2本の動画を合わせると、273件中17件のコメントが製品の問題について述べていました。テーマ別の内訳は次のとおりです。

| テーマ | コメント | 例（原文、投稿者は非表示） | いいね |
| --- | --- | --- | --- |
| フタ・液漏れ | 6 | The way I just gasped when it spilled. Were you supposed to open it from the top instead? | 79 |
| バッテリー | 3 | the batteries do not last long enough to make a full smoothie | 9 |
| 全般的な否定、返品 | 2 | Not worth it, returned after a week | 0 |
| 氷や冷凍フルーツに弱い | 2 | It's not. I returned mine. Barely could get through frozen fruit | 97 |
| 部品の欠品 | 2 | ours both didn't come with the blade cap | 0 |
| すぐ壊れた | 1 | they are shit, lasts 2 spins | 1 |
| 洗いにくい | 1 | Hard to wash | 0 |

![7つのテーマをコメント数の多い順に並べた横棒グラフ。フタ・液漏れ6件（ティール）、バッテリー3件、全般的な否定2件、冷凍フルーツに弱い2件、部品の欠品2件、すぐ壊れた1件、洗いにくい1件です。サブタイトルには、フタに関するコメントの多くはフタの開け方がわからないという内容であること、2件のコメントで購入者が製品を返品したと述べていることが記されています。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/fig-05-issue-theme-bar.ja.light.svg)

フタと液漏れの問題が最大のグループです。このグループで最も「いいね」が多いコメント（79件）は、フタは上から開けるべきだったのかと尋ねています。別のコメントでは、投稿者も最初に同じようにこぼしてしまい、キャップを使うことに思い至らなかったと述べています。2026-01-22のコメントは、フタのボタンは壊れるうえ交換部品が販売されていないので使わないようにと、ほかの人に注意を促しています。

この種の問題は、バッテリーやモーターへの不満とは性質が異なります。説明書、パッケージ、製品写真を改善すれば解決できることが多く、そうすれば返品率もすぐに下がる可能性があります。「どの不満テーマが返品とともに増えるか」という問いが見つけようとしているのは、まさにこの点です。あるテーマが返品の急増と重なれば、取るべきアクションは明確です。

表には「いいね」数を残しておきます。97件の「いいね」が付いた「返品した」というコメントと、「いいね」0件の不満とでは、代弁している人数が違います。セマンティックビューでは、コメント数と「いいね」で重み付けしたコメント数の2つのメジャーを定義し、エージェントに両方を報告させます。

Snowflakeでは、`AI_CLASSIFY`でラベル付けができます。7つのテーマに「製品の問題ではない」を加えたものをカテゴリにします。出力はマルチラベルに設定し、価格への不満とジョークを除外するタスク説明を追加します。ドキュメントでは、カテゴリはおおむね20個以下が推奨されており、例を与えることもできます（[AI_CLASSIFY documentation](https://docs.snowflake.com/en/sql-reference/functions/ai_classify)）。

このドキュメントにある利用上の注意のうち、重要な点が1つあります。入力が英語の場合に最も良い結果が得られ、ほかの言語では精度が下がることがあります。TikTokやAmazon USのコメントの大半は英語なので、米国のブランドにとっては小さな問題です。スペイン語圏などほかの市場も追跡する場合は、まず手作業でラベル付けしたサンプルで確認してください。また、タスク説明とカテゴリも1件ごとにトークンとして数えられます。そのため、1件あたりの実際のトークン数はコメントの長さより多くなります。

```text
UPDATE voc.core.social_mentions
SET themes = AI_CLASSIFY(
      body,
      ['lid_or_spill', 'battery', 'weak_on_ice', 'missing_part',
       'breaks_quickly', 'hard_to_clean', 'general_negative', 'not_a_product_issue'],
      {'output_mode': 'multi',
       'task_description': 'Label only comments that report a problem with this blender. Price complaints, jokes and brand comparisons are not_a_product_issue.'}
    ):labels
WHERE themes IS NULL;
```

返品テーブルの`customer_comments`列にも同じ分類を実行し、結果を`return_themes`に書き込みます。こうすれば両方が同じテーマ、同じモデル、同じ説明を使うので、テーマ別の比較が成り立ちます。`AI_SENTIMENT`で各コメントをネガティブからポジティブまでスコア付けすることもできます（[AI_SENTIMENT documentation](https://docs.snowflake.com/en/sql-reference/functions/ai_sentiment)）。ただし、ここで問うのはどれだけ不満かではなくどの問題かなので、あまり役に立ちません。

## テスト2：Redditを検索すると何が返ってくるか

Redditを率直なユーザーの声が集まる場所と考えるチームは多いため、TikTokを補えるかどうかを確認しました。5回目の呼び出しでは`reddit/search`で「portable blender」を新しい順に検索し、14件の投稿が返りました。10回目の呼び出しではMCP経由で「ninja blast」を検索し、7件の投稿が返りました（[RedditデータAPI](https://everyinfra.com/ja/reddit-api)、[Reddit検索API](https://everyinfra.com/ja/reddit-api/search)）。この機能は常に直近1か月を検索対象とし、1回の呼び出しで最大25件の投稿を返します。14件の投稿は2026-10-30〜2026-11-05（UTC）の範囲しかカバーしておらず、7件の投稿は11月3日〜11月5日の範囲でした。

すべての投稿を読みましたが、21件のうちブレンダーの性能を論じたものは1件もありませんでした。「portable blender」の14件のうち10件はセール情報の投稿でした。大半はr/priceglitchなどのセール情報コミュニティに投稿されたもので、「Ninja Blast Max Portable Blender, Twist & Go, 22oz」に色と価格を付けたようなタイトルでした。セール情報の投稿は2026-10-31、2026-11-01、2026-11-05に、Amazonの日替わりセールのまとめは2026-11-02と2026-11-03に見られました。

残りの4件はブレンダーとは無関係でした。1件はポータブルなThinkPadについての質問で、3DソフトウェアBlenderのコミュニティでノートPCのスペックを尋ねる投稿が2件ありました。「ninja blast」の7件のうち4件はセール情報でした。残りの3件は偶然語句が一致しただけで、1件はビデオゲーム、2件はアニメシリーズ「ニンジャゴー」のコミュニティの投稿でした。

| クエリ | 投稿数 | セール情報の投稿 | 無関係 | 製品に関する話題 | 期間（UTC） |
| --- | --- | --- | --- | --- | --- |
| portable blender（REST） | 14 | 10 | 4 | 0 | 2026-10-30〜2026-11-05 |
| ninja blast（MCP） | 7 | 4 | 3 | 0 | 2026-11-03〜2026-11-05 |

![2本のバーからなる横向きの積み上げ棒グラフ。portable blenderは14件で、セール情報の投稿10件、無関係4件、製品に関する話題0件です。MCP経由のninja blastは7件で、セール情報の投稿4件、無関係3件、製品に関する話題0件です。サブタイトルには、検索は新しい順で、14件の投稿は10月30日〜11月5日の範囲しかカバーしていないこと、無関係な投稿は3DソフトウェアのBlender、ニンジャゴーシリーズ、ゲームなどの語句一致によるものであることが記されています。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/fig-06-reddit-kind-bar.ja.light.svg)

これはRedditが役に立たないという意味ではありません。短いキーワードを新しい順で検索するというクエリが間違っているのです。Snowflakeに取り込むなら、3つの変更を加えます。第1に、関連性はタイトルだけで判定し、「blender」の前にportable、personal、smoothie、またはブランド名が付いていることを条件にします。最初のルールでは投稿本文中の「blender」にも一致させていたため、3Dソフトウェアの投稿を製品の投稿として数えていました。

第2に、セール情報の投稿は有用なデータなので、独立したカテゴリとして残します。あるブレンダーが1か月にセール情報コミュニティに何回、いくらで登場したかを、売上テーブルと並べて見られます。そうすれば、セール週の後に返品が急増するかどうかがわかります。第3に、実際の製品に関する話題は、スムージーやフィットネス食を扱うグループなど、少数のコミュニティに集まる傾向があります。サイト全体を検索するのではなく、そうしたコミュニティの関連スレッドに`reddit/comments`を使います。

## テスト3：Amazonレビューでは503が返ったが、課金は発生しなかった

最も欲しかったデータはAmazonレビューでした。レビューと返品は、同じプラットフォームの同じ購入者から生まれるからです。しかし、`amazon/reviews`を5回呼び出しても、レビューは1件も返ってきませんでした（[AmazonレビューAPI](https://everyinfra.com/ja/amazon-api/reviews)）。SGTの00:17、00:18、00:24に、NinjaとHamilton Beachの異なる3製品へのリクエストが、それぞれ理由として`access_blocked`を示す503を返しました。これは、その商品のレビューが一時的に取得できず、課金は発生しておらず、レスポンスでは900秒後の再試行が推奨されていることを意味します。

SGTの00:19には、より古い製品を試しました。結果は200で0件となり、この呼び出しは空の結果として返金されました。SGTの00:36、最初の試行から19分後に1つ目の製品を再度呼び出しましたが、やはり503でした。5回の呼び出しはいずれも`charged_usd`が0です。

| SGT | 商品 | HTTPステータス | 理由 | 所要時間 | 課金額 |
| --- | --- | --- | --- | --- | --- |
| 00:17:41 | B0C2FF4JD1 | 503 | `access_blocked` | 60.12 s | $0 |
| 00:18:41 | B0CN3S2SW2 | 503 | `access_blocked` | 58.57 s | $0 |
| 00:19:47 | B09B8V1LZ3 | 200 | 空の結果（返金済み） | 67.25 s | $0 |
| 00:24:22 | B0D2YYHN1B | 503 | `access_blocked` | 19.48 s | $0 |
| 00:36:23 | B0C2FF4JD1 | 503 | `access_blocked` | 38.42 s | $0 |

毎日実行するSnowflakeのタスクでも同じことが起きるため、起きたとおりに報告します。EveryInfraのルールでは、一時的に利用できない対象やログインが必要な対象は、理由と再試行のヒントを付けて503を返します。レビューがないと主張する空の200を返すことはありません。この違いはローダーにとって重要です。503は「今日はデータがないので明日再試行」、空の200は「本当に何もない」という意味です。

プロシージャがこの2つを区別しないと、セマンティックビューで一部の製品に偽の「不満ゼロ」の日が表示されます。そしてエージェントはそこから誤った結論を導きます。HTTPステータスと理由を`jobs`テーブルに保存し、ステータス200のレスポンスだけを展開します。セマンティックビューに「データ欠損日数」のメジャーを追加し、どの日にデータがなかったかをエージェントが答えられるようにします。

カタログには、`amazon/reviews`が返すのは商品ページに表示される注目のレビューのみだとも記載されています。通常は1製品あたり十数件で、1回の呼び出しあたり最大100件、対象は11のAmazonマーケットプレイスです。

つまり、うまくいった日でもすべてのレビューが取れるわけではありません。取れるのは、Amazonが表示を選んだレビューの集合です。この集合は「最も目立つ不満は何を言っているか」を知るには十分ですが、不満をすべて数えるには足りません。完全な件数が必要なら、Seller Centralのデータや、FBA返品レポートの顧客コメントのほうが網羅的です。このレポートは上記の`fba_returns`テーブルのデータ元です。

## Cortex Agentに聞く：返品率とともに増える不満テーマはどれか

テーブルがそろったら、週次のビューを作ります。各行は1製品・1テーマ・1週を表し、ソーシャルでの言及数、返品数、販売数を持ちます。テーマ別返品率は、その週のそのテーマの返品数を、その週の販売数で割った値です。

```text
CREATE OR REPLACE VIEW voc.core.theme_returns_weekly AS
WITH mentions AS (
  SELECT s.product_key,
         DATE_TRUNC('week', s.posted_at)::DATE AS week_start,
         t.value::STRING AS theme,
         COUNT(*) AS mentions,
         SUM(1 + COALESCE(s.like_count, 0)) AS weighted_mentions
  FROM voc.core.social_mentions s,
       LATERAL FLATTEN(input => s.themes) t
  WHERE t.value::STRING <> 'not_a_product_issue'
  GROUP BY 1, 2, 3
),
returns AS (
  SELECT pm.product_key,
         DATE_TRUNC('week', f.return_date)::DATE AS week_start,
         COALESCE(t.value::STRING, 'unclassified') AS theme,
         SUM(f.quantity) AS returned_units
  FROM voc.core.fba_returns f
  JOIN (SELECT DISTINCT product_key, asin FROM voc.core.product_map) pm
    ON pm.asin = f.asin,
       LATERAL FLATTEN(input => f.return_themes, OUTER => TRUE) t
  GROUP BY 1, 2, 3
),
sales AS (
  SELECT pm.product_key,
         DATE_TRUNC('week', d.sale_date)::DATE AS week_start,
         SUM(d.units_ordered) AS units_sold
  FROM voc.core.sales_daily d
  JOIN (SELECT DISTINCT product_key, asin FROM voc.core.product_map) pm
    ON pm.asin = d.asin
  GROUP BY 1, 2
),
theme_weeks AS (
  SELECT COALESCE(m.product_key, r.product_key) AS product_key,
         COALESCE(m.week_start, r.week_start) AS week_start,
         COALESCE(m.theme, r.theme) AS theme,
         COALESCE(m.mentions, 0) AS mentions,
         COALESCE(m.weighted_mentions, 0) AS weighted_mentions,
         COALESCE(r.returned_units, 0) AS returned_units
  FROM mentions m
  FULL OUTER JOIN returns r
    ON r.product_key = m.product_key
   AND r.week_start = m.week_start
   AND r.theme = m.theme
)
SELECT tw.product_key, tw.week_start, tw.theme,
       tw.mentions, tw.weighted_mentions, tw.returned_units,
       s.units_sold,
       tw.returned_units / NULLIF(s.units_sold, 0) AS theme_return_rate
FROM theme_weeks tw
LEFT JOIN sales s
  ON s.product_key = tw.product_key AND s.week_start = tw.week_start;
```

このビューでは、返品が抜け落ちないように3つの工夫をしています。第1に、言及と返品は製品・週・テーマで完全外部結合し、販売はその後に製品と週で結合します。そのため、ある週のソーシャル投稿がフタの問題にしか言及していなくても、その週のバッテリーによる返品は独自の行として残ります。第2に、`OUTER => TRUE`により、コメントやテーマのない返品も`unclassified`というラベルで残します。第3に、`product_map`では1つのASINが複数のSKUに対応することがあるため、ビューでは製品とASINの重複のない組み合わせで結合します。そうしないと、返品や販売が重複して数えられます。1点注意があります。1つのコメントが2つのテーマを持つことがあるため、テーマ別の行を合計すると実際の返品数を上回ることがあります。全体の返品率を出すには、テーマ別の行を足し合わせるのではなく、`fba_returns`を直接合計してください。

続いて、その上にセマンティックビューを作ります。セマンティックビューは、各列の意味とどの列がメジャーかをCortex Analystに伝えます（[Semantic views overview](https://docs.snowflake.com/en/user-guide/views-semantic/overview)、[CREATE SEMANTIC VIEW](https://docs.snowflake.com/en/sql-reference/sql/create-semantic-view)）。メジャーは、テーマ別言及数、「いいね」で重み付けした言及数、テーマ別返品数、テーマ別返品率の少なくとも4つを定義します。ディメンションは製品、テーマ、週です。シノニムも追加します。「leak」「spill」「cap」を`lid_or_spill`に対応させ、ビジネスユーザーが実際に入力する言葉をエージェントが理解できるようにします。

エージェントの仕様はYAMLで記述します。最小構成は、このセマンティックビューを対象にしたCortex Analystツール1つです。私たちは、製品とテーマを指定すると最も「いいね」の多いコメントを返すカスタムツールを1つ追加しています。構造は[CREATE AGENT](https://docs.snowflake.com/en/sql-reference/sql/create-agent)に従います。Cortex Agentsのドキュメントによると、カスタムツールのパラメータにはobject型を使えないため、パラメータはシンプルにしてください（[Cortex Agents overview](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)）。

```text
models:
  orchestration: claude-sonnet-5-5
orchestration:
  budget:
    seconds: 120
    tokens: 40000
instructions:
  orchestration: >
    For questions about complaint themes and returns, use VocAnalyst on weekly data.
    Compare each theme's mentions with the same week and the next 1 to 4 weeks of
    theme_return_rate. Report weeks with missing review data. Never treat a missing
    week as zero complaints.
  response: >
    Give the theme, the weeks, both raw and like-weighted mention counts, and the
    return rate change. Quote at most two comments from TopComments, without author names.
  sample_questions:
    - question: "Which complaint theme rises together with the return rate?"
    - question: "Did lid complaints for the Ninja Blast track returns over the last four weeks?"
tools:
  - tool_spec:
      type: cortex_analyst_text_to_sql
      name: VocAnalyst
      description: Weekly social complaint themes, returns and sales by product.
  - tool_spec:
      type: generic
      name: TopComments
      description: Returns the most liked comments for one product and one theme.
      input_schema:
        type: object
        properties:
          product_key: {type: string, description: Product key from product_map}
          theme: {type: string, description: One theme label}
        required: [product_key, theme]
tool_resources:
  VocAnalyst:
    semantic_view: voc.core.voc_returns
  TopComments:
    type: procedure
    identifier: voc.core.top_comments
    execution_environment:
      type: warehouse
      warehouse: voc_xs_wh
```

「ともに増える」には厳密な定義が必要です。インストラクションでは、不満の発生数を、同じ週およびその後の1〜4週目の返品数と比較するようエージェントに指示しています。その理由は私たち自身のデータが示しています。不満と返品は同時には起きません。下のグラフは、製品の問題に関する17件のコメントを月別に示したものです。

比較動画は2025-07-27に公開され、問題に関する4件のコメントは6日後、44日後、245日後、245日後に投稿されました。試用動画は2026-01-20に公開されました。問題に関する13件のコメントのうち最初の1週間に投稿されたのは2件だけで、残りは3月から9月にかけて分散しています。返品に触れた2件のコメントはどちらも2026年3月に投稿されたもので、1件は動画の245日後、もう1件は52日後でした。

| コメント（投稿者は非表示） | テーマ | コメント日 | 動画公開からの日数 | いいね |
| --- | --- | --- | --- | --- |
| フタは上から開けるべきなのかと質問 | フタ・液漏れ | 2026-01-20 | 0 | 79 |
| フタのボタンは壊れるので使わないように | フタ・液漏れ | 2026-01-22 | 2 | 5 |
| このブレンダーはみんなにひどいと言われていると発言 | 全般的な否定 | 2026-03-01 | 40 | 23 |
| 返品した、冷凍フルーツに弱い | 氷に弱い | 2026-03-13 | 52 | 97 |
| バッテリーがスムージー1回分もたない | バッテリー | 2026-03-14 | 53 | 9 |
| 2台とも刃のキャップが付属していなかった | 部品の欠品 | 2026-07-31 | 192 | 0 |
| 予備のフタ目当てに2台目を購入 | フタ・液漏れ | 2026-09-12 | 235 | 0 |

![積み上げ縦棒グラフ。横軸はコメントの月で、2025年8月、9月、2026年1月、3月、4月、7月、9月です。縦軸は製品の問題に関するコメント数で、0〜8です。最も多いのは2026年3月の8件で、内訳はすぐ壊れた、全般的な否定、冷凍フルーツに弱い、バッテリー、フタ、洗いにくいです。2026年1月はフタに関するコメントが2件、4月は3件、そのほかの月は各1件です。フタ・液漏れはティールで示しています。サブタイトルには、コメントの大半は動画の1〜2か月後に投稿されるため、1回の取得では後から寄せられる不満を見逃すと記されています。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/fig-07-issue-monthly-bar.ja.light.svg)

このグラフを受けて、Snowflakeの設計を2点変えます。第1に、動画を1回だけ取得して終わりにしないことです。最初の1週間だけ取得していたら、1本目の動画では問題に関するコメントを3件、2本目では11件見逃していたことになります。そのため、`watchlist`では同じ動画を毎週または毎月取得し、`MERGE`で新しいコメントを追加します。

第2に、不満と返品を同じ週で結合しても、おそらく何も見えてきません。購入者はしばらく製品を使ってから不満を書き込み、返品します。ラグを決め打ちせず、データから特定してください。

量についても正直に言っておく必要があります。2本の動画から得られた問題に関するコメントは14か月で17件なので、大半の週はゼロです。これほどまばらなデータで相関を取っても意味がありません。

エージェントにしっかりした土台を与えるには、製品ごとに試用系の動画を20〜30本追跡し、Amazonレビューと返品コメントも加えます。まずは月次の粒度で始め、データが十分にそろったら週次に移行します。この制約はSnowflakeではなくデータ量に起因するもので、どのツールでも同じです。インストラクションでデータ欠損週を報告するようエージェントに指示しているのもそのためです。3件のコメントからでっち上げた傾向より、正直な「データ不足」のほうが価値があります。

## 1か月のコストはいくらか：API料金とSnowflake料金

まず具体的な計画を立て、その費用を見積もります。あるブランドが20個のASINと20本のTikTok動画を追跡し、それぞれについてレビューまたはコメントを1日1回取得するとします。新しい動画を見つけるため、Redditでは5個のキーワードを毎日、TikTokでは5個のキーワードを毎週検索します。30日間で1,370回のAPI呼び出しになります。API料金は2026-11-06時点で確認したカタログ価格に基づくため、実際の価格です。

Snowflakeの項目はすべて見積もりで、算出方法は表に示しています。価格は2026-11-04発効のService Consumption Tableと、AWS US EastにおけるオンデマンドのStandardエディションの価格に基づきます。プラットフォームクレジットは$2.00、AIクレジットはグローバルレートで$2.00です（[Service Consumption Table](https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf)、[料金ページ](https://www.snowflake.com/en/pricing-options/)、[Cortex AI pricing](https://docs.snowflake.com/en/user-guide/snowflake-cortex/pricing)）。

| 項目 | 方法 | 月あたり |
| --- | --- | --- |
| EveryInfra API料金 | 600件の`amazon/reviews` × $0.001389 + 600件の`tiktok/comments` × $0.000556 + 150件の`reddit/search` × $0.000556 + 20件の`tiktok/search` × $0.000556 | $1.26 |
| ウェアハウスがレスポンスを待機（同期・逐次） | 実測の所要時間はAmazonレビュー67.25秒（5回の呼び出しで最も遅く、結果は空。503を返した呼び出しの19.48〜60.12秒で計算すると、この行は$24.61〜$38.16）、TikTokコメント52.3秒、Reddit 7.62秒、TikTok検索3.78秒。合計20.27時間 × XSで1時間あたり1クレジット × $2 | $40.54 |
| ウェアハウスが待機（同期・10並列呼び出し） | 上記を10で割った値 | $4.05 |
| ウェアハウス（非同期・2タスク） | 1日約46回のジョブ送信 × 2秒、最低課金60秒の収集処理の実行2回、さらに3回の実行それぞれの後に自動サスペンドまでのアイドル60秒。1日約392秒、月3.3時間 | 約$6.5 |
| `AI_CLASSIFY`によるテーマラベル | 10,000件のコメント × 約150トークン = 1.5Mトークン × 1Mトークンあたり1.62 AIクレジット × $2 | $4.86 |
| Cortex Agentへの質問（Sonnet 5.5） | 質問50件 × 入力20,000トークン・出力1,500トークン、100万トークンあたり入力1.30・出力6.50 AIクレジット | $3.58 |
| Cortex Agentへの質問（Haiku 5.5） | 同上、入力0.065・出力0.325 | $0.18 |
| エージェントがウェアハウスで実行するSQL | クエリ50件 × 最低60秒 × 1クレジット時間あたり$2 | $1.67 |
| データの保存 | 1 GB、1 TBあたり月$23 | 約$0.02 |

![対数スケールの横棒グラフで、月額コスト8項目を示しています。EveryInfra API料金1.26米ドル（ティール）、逐次待機するウェアハウス40.54ドル（最長の棒）、10並列呼び出し4.05ドル、AI_CLASSIFYラベル4.86ドル、Sonnet 5.5によるCortex Agentへの質問50件3.58ドル、Haiku 5.5では0.18ドル、ウェアハウス上でのエージェントのSQL 1.67ドル、1 GBのストレージ0.022ドルです。サブタイトルには、API料金は実際の価格で、それ以外はSnowflakeの消費テーブルに基づく推定値であり実測ではないと記載されています。](https://everyinfra.com/articles/snowflake-cortex-agents-mcp-social-reviews/fig-08-monthly-cost-bar.ja.light.svg)

注目すべきは2行目です。API自体の費用は月$1.26です。しかしプロシージャを最も単純なコード、つまり同期呼び出しを1回ずつ順に行う形で書くと、ウェアハウスは月20時間以上稼働します。これは約$40.54で、API料金の32倍です。10並列呼び出しか非同期の2タスク構成にすれば、この行は$4〜$7に下がります。

今回の試算では並列呼び出しが最も安価ですが、スレッドとタイムアウトをPythonで自前で管理する必要があります。非同期構成は2〜3ドル高くなります。その代わり、待ち時間が長すぎて個々のリクエストが打ち切られることはなく、失敗したジョブは翌日に再実行されるだけです。私たちは非同期構成を選びます。

合計すると、逐次構成は月約$51.93、並列構成は月約$15.44です。シンガポールリージョンではStandardエディションのクレジットが$2.50なので、ウェアハウスの行を1.25倍してください。Enterpriseエディションの場合、US Eastのオンデマンド価格は1クレジットあたり$3.00で、同様に換算します。

2つ目の調整ポイントはモデルです。Cortex Agentのオーケストレーションはトークン単位で課金され、消費テーブル上のモデル価格には大きな差があります。同じ50件の質問で、Sonnet 5.5は約$3.58、Haiku 5.5は約$0.18です。Opus 5.5は入力100万トークンあたり2.60、出力100万トークンあたり13.00で、Sonnetの2倍です。下の表は、本記事に関係する価格を100万トークンあたりのAIクレジットで示したものです。AI関数は入力と出力を区別せず、合計トークン数で課金されます。

| モデルまたは関数 | 入力 | 出力 | この記事での使い方 |
| --- | --- | --- | --- |
| claude-sonnet-5-5（CoWork、Agents） | 1.30 | 6.50 | エージェントのオーケストレーション、コスト表のデフォルト |
| claude-haiku-5-5（CoWork、Agents） | 0.065 | 0.325 | 低コストのオーケストレーション |
| claude-opus-5-5（CoWork、Agents） | 2.60 | 13.00 | 定型レポートには不要 |
| openai-gpt-6.1-sol（CoWork、Agents） | 1.30 | 6.50 | Sonnetと同価格 |
| `AI_CLASSIFY` | 1.62（合計） |  | コメントと返品メモのテーマラベル |
| `AI_SENTIMENT` | 1.60（合計） |  | 感情スコア（任意） |
| `AI_AGG` | 1.85（合計） |  | テーマ別のテキスト要約（任意） |

質問1件あたり入力20,000トークンという数値は私たちの想定です。セマンティックビュー、指示、1〜2回分のツール結果を含みます。1回のチャットでエージェントがツールを何度も呼び出したり、長いコメントをコンテキストに読み込んだりすると、実際の数値は増えます。運用開始後は`CORTEX_AGENT_USAGE_HISTORY`ビューで実数を確認できます。CoWorkでの質問は`SNOWFLAKE_COWORK_USAGE_HISTORY`に記録され、どちらのビューにももう一方のリクエストは含まれません（[Cortex Agentの使用状況ビュー](https://docs.snowflake.com/en/sql-reference/account-usage/cortex_agent_usage_history)、[CoWorkの使用状況ビュー](https://docs.snowflake.com/en/sql-reference/account-usage/snowflake_cowork_usage_history_view)）。

プロシージャから外部へのリクエストは、それぞれ`EXTERNAL_ACCESS_HISTORY`ビューに記録されます。このビューは最大3時間遅れることがあります（[外部アクセス履歴](https://docs.snowflake.com/en/sql-reference/account-usage/external_access_history)）。`voc.raw.jobs`の`charged_usd`値と時刻で突き合わせれば、呼び出し回数と支出を毎月両側で確認できます。Snowflakeではエージェントごとのリソース予算やユーザーごとのクォータも設定できます（[Cortex Agentsのリソース予算](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-resource-budgets)）。EveryInfra側では、このキーに専用の1分あたり呼び出し上限を設定してください。両側に上限を設けておけば、不具合のあるタスクが一晩で数万回の呼び出しを行うことはありません。

消費テーブルには誤読しやすい行が1つあります。「Cortex Tools」は、サードパーティのエージェントがSnowflake以外のエンドポイントにアクセスする場合に1,000回のツール呼び出しあたり8 AIクレジットを課金し、Snowflakeのエンドポイントには課金しません。本構成でEveryInfra APIを呼び出すのはエージェントではなく、Snowflakeのストアドプロシージャです。エージェントはSnowflakeのテーブルを読むだけなので、この行は含めていません。

MCPコネクタに切り替え、チャット中にエージェントに外部ツールを呼び出させる場合を考えます。この行が適用されるか、どう計上されるかはドキュメントに記載がありません。運用開始前にSnowflakeのアカウントチームに確認してください。新規アカウントならトライアルで構成全体を試せます。料金ページには30日間・$400分のクレジットと記載されています。

## Snowflake CoWork Automationsで週次メールを配信する

データとエージェントが揃ったら、最後は適切な人に適切なタイミングで回答を届けます。CoWork Automationsはまさにそのための機能です。質問をスケジュール実行するよう設定します。たとえば毎週月曜09:00です。よい質問の例は「直近4週間で、返品率とともに増えた不満のテーマはどれか？」です。結果はメールで届きます（[Automationsのドキュメント](https://docs.snowflake.com/en/user-guide/snowflake-cortex/snowflake-cowork/automations)）。

まず、ドキュメントに記載された制限を把握しておきましょう。オートメーションの実行頻度は最大で1時間に1回です。結果の送信先はメールのみで、各受信者は事前にSnowflakeでメールアドレスを認証する必要があります。実行履歴の保持期間は60日です。各オートメーションは作成者の権限で実行されるため、作成者が読めるテーブルはどれもメールに含まれる可能性があります。

| ドキュメント記載の制限 | 値 | 週次の不満レポートへの影響 |
| --- | --- | --- |
| 最短の実行間隔 | 1時間に1回 | 影響なし（レポートは週次） |
| 配信 | メールのみ、認証済みアドレス宛 | 受信者は事前に認証が必要 |
| 実行履歴 | 60日 | より長い履歴が必要なら回答をテーブルに保存 |
| 権限 | 作成者の権限 | 作成者が読めるテーブルはすべてメールに含まれうる |
| EXECUTE AGENT TASK | デフォルトでPUBLICに付与 | 必要に応じて取り消し、ロール単位で付与 |

本ユースケースを妨げる制限はありません。不満や返品は週単位で見るテーマなので、1時間ごとの実行で十分すぎるほどです。配信がメールのみなので、Slackへの直接投稿はできません。プレスリリースによると、Snowflake for Slackは現在パブリックプレビュー中です（[Expedition 2026のプレスリリース](https://www.snowflake.com/en/news/press-releases/expedition-2026-coco-cowork/)）。

本当に注意すべきはアクセス権です。EXECUTE AGENT TASK権限はデフォルトでPUBLICロールに付与されているため、アカウント内の誰でも自分用のオートメーションを作成できます。オートメーションは作成者の権限で実行されます。そのため、全売上データを読める人は、そのデータを自分で入力した任意のアドレスに送れます。これが許容できない場合は、管理者がPUBLICから権限を取り消し、ロール単位で付与してください。

## このテストの制約

Snowflake側の内容はすべて設定例です。私たちはSnowflakeアカウントを持っておらず、このSQL、プロシージャ、エージェント仕様は実行していません。構文は2026-11-06時点のdocs.snowflake.comに基づいています。Snowflakeは今月大規模なリリースを行ったため、構文や制限は今後も変わる可能性があります。運用で頼る前に、各ブロックをご自身のアカウントで実行してください。

コスト表のうち実際の価格はAPI料金だけです。Snowflakeの行はすべて、公開価格と私たちの想定に基づく推定値です。最大の不確定要素は、分類するコメント1件あたりの実際のトークン数と、エージェントへの質問1件あたりの実際のトークン数の2つです。

データ面では、今回Amazonレビューの呼び出し5回はいずれもレビューを返しませんでした。そのため、テーマ分析にはTikTokコメント273件のみを使っています。ラベル付けは筆者1人が上記のルールに従って手作業で行い、誰も再確認していません。問題のあるコメント17件から、蓋とこぼれの問題に注意すべきだとは言えます。しかし、この製品の本当の問題発生率を推定するには足りません。

本記事で示すのは売上テーブルと返品テーブルの構造だけで、実際の数値も架空の数値も載せていません。「返品とともに増える不満のテーマはどれか」の答えは、ご自身のデータが揃ったときに得られます。

EveryInfra自体にも制限が1つあります。上記の理由により、現時点ではEveryInfraのMCPサーバーをSnowflakeのコネクタとして追加できません。外部アクセスのルートはこれに依存せず、私たちが推奨するのもこのルートです。ただし、CoWork内でワンクリックで接続したい場合、それはまだできません。この状況は本記事の公開時に改めて確認します。

## これまでの記事との違い

これまでのレビュー関連記事では、データをエージェントやファイルの中に置いていました。[Claude Coworkのスケジュールタスク](https://everyinfra.com/ja/blog/claude-cowork-scheduled-tasks-competitor-price-reviews)、[Claude Managed Agentsによる夜間のレビューチェック](https://everyinfra.com/ja/blog/claude-managed-agents-nightly-review-monitoring)、[Qwen Agent MCPの実践](https://everyinfra.com/ja/blog/qwen-agent-mcp-ecommerce-review-analyst)、[Claude CodeでDeepSeek V4を使ったレビューのクリーニング](https://everyinfra.com/ja/blog/deepseek-v4-claude-code-comment-cleaning)です。これらが答えるのは「最近どんな不満が出ているか」です。本記事では、レビューを社内のデータウェアハウスに取り込み、売上や返品と同じアクセスルールとビジネス定義のもとで扱います。そのため問いは「実際にコストにつながっている不満はどれか」に変わります。代償はウェアハウスの請求とテーブル一式が増えることで、上の試算では1ブランドあたり月約15ドルです。売上と返品がまだSnowflakeにないなら、これまでの方法のほうが手軽です。すでにSnowflakeにあるなら、この構成で初めて不満と財務の数値を突き合わせられます。

## 最近の変更点

2026-11-04に、Snowflakeの新しいService Consumption Tableが発効しました。本記事のモデル価格、AI関数の価格、「Cortex Tools」の行はすべてこの版に基づいています（[Service Consumption Table](https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf)）。

同日、SnowflakeマネージドMCPサーバーにバックグラウンド実行を追加するリリースノートが公開されました。`CORTEX_AGENT_RUN`ツールと`CODING_AGENT`ツールに`background`引数が加わり、即座にハンドルを返すようになりました。これらのツールを持つサーバーには、`ASYNC_AGENT_RUN_STATUS`、`ASYNC_AGENT_RUN_CANCEL`、`AGENT_RUN_RESULT`の3つのツールが自動的に追加されます。新しい`content_cel_filter`パラメータを使うと、クライアントはまずアウトラインを読み、必要な部分だけを取得できます（[2026年11月4日のリリースノート](https://docs.snowflake.com/en/release-notes/2026/other/2026-11-04-mcp-async-agent-tools)）。

## よくある質問：Snowflake Cortex AgentsのMCPとSnowflake CoWork

### Snowflake Cortex AgentsのMCPサーバーは、MCPコネクタと同じものですか？

いいえ。両者は向きが逆です。出典：「snowflake cortex agents mcp server」のGoogleオートコンプリート。SnowflakeマネージドMCPサーバーでは、Snowflakeがサーバーになります。外部のクライアントがMCP経由で、エージェント、Cortex Search、Cortex Analyst、SQLを呼び出します。ツールタイプは`CORTEX_AGENT_RUN`、`CODING_AGENT`、`CORTEX_SEARCH_SERVICE_QUERY`、`CORTEX_ANALYST_MESSAGE`、`SYSTEM_EXECUTE_SQL`、`GENERIC`です。ドキュメントでは、ハードコードしたトークンよりOAuthを推奨しています（[SnowflakeマネージドMCPサーバー](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp)、[CREATE MCP SERVER](https://docs.snowflake.com/en/sql-reference/sql/create-mcp-server)）。

MCPコネクタではSnowflakeがクライアントになり、Cortex AgentsとCoWorkから外部のMCPサーバーを呼び出せます。認証はOAuthのみです（[コネクタのドキュメント](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors)）。本記事で必要なのは後者の向きですが、最終的にはMCPではなく外部アクセスを使っています。

### Snowflake CoWorkとCoCo（Cortex Code）の違いは何ですか？

出典：「snowflake cowork vs coco」のGoogleオートコンプリート。2026-11-03のプレスリリースによると、CoWorkはビジネスユーザー向けです。ユーザーはai.snowflake.comで普段の言葉で質問し、ダッシュボードを開き、オートメーションを設定します。CoCoは開発者やデータエンジニア向けで、コードの作成やデータオブジェクトの構築に使います。アナリストがCoCoにダッシュボードの内容を説明して作成させることもできます（[Expedition 2026のプレスリリース](https://www.snowflake.com/en/news/press-releases/expedition-2026-coco-cowork/)、[Cortex Codeのドキュメント](https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code)）。

本記事では、データエンジニアがCoCoまたは通常のSQLでテーブル、プロシージャ、セマンティックビューを構築します。ビジネスユーザーはCoWorkで質問し、週次レポートを受け取ります。

### Cortex AgentsはOAuthではなくAPIキーで外部APIを呼び出せますか？

はい。ただしMCPコネクタ経由ではできません。出典：Snowflakeのコネクタのドキュメントにある「OAuthのみ」という記述から生じる追加の疑問。キーは外部アクセス統合を通じて、`GENERIC_STRING`型のシークレットに保存します。APIを呼び出すPythonのストアドプロシージャを書き、そのプロシージャをカスタムツールとしてエージェントに追加します。エージェントが呼び出すと、プロシージャはウェアハウス上で実行され、ウェアハウスの稼働時間で課金されます（[外部ネットワークアクセス](https://docs.snowflake.com/en/developer-guide/external-network-access/creating-using-external-network-access)、[Cortex Agentsの概要](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents)）。私たちは、質問のたびにAPIの待ち時間が発生しないよう、スケジュールタスクでテーブルに読み込む方式を選びます。

### AI_CLASSIFYは英語以外のコメントにも使えますか？

使えますが、事前に確認してください。出典：SnowflakeのAI_CLASSIFYドキュメントの使用上の注意。ドキュメントには、英語の入力で最も良い結果が出て、他の言語では精度が下がる場合があると記載されています（[AI_CLASSIFYのドキュメント](https://docs.snowflake.com/en/sql-reference/functions/ai_classify)）。スペイン語圏などの市場を追跡する場合は、約100件のコメントを手作業でラベル付けして比較してください。カテゴリが一致しない箇所では、タスクの説明と例をより具体的にします。コストは上がりますが、Cortex Agentsでより高性能なモデルに各コメントを判定させることもできます。

### EveryInfraは失敗した呼び出しや結果が空の呼び出しにも課金しますか？

いいえ。出典：私たちが実際に行った12回の呼び出しのレスポンス。不正なパラメータ、一時的に利用できない対象（ステータス503）、空の結果は課金されず、課金されたレスポンスにはすべて正確な金額が記載されます。本記事でデータが返らなかった6回の呼び出しは、いずれも`charged_usd`が0です。非同期モードでは送信時に課金され、後で失敗したジョブはその課金分が返金されます。

## 始め方

1. EveryInfraに新規登録し、[クイックスタート](https://everyinfra.com/ja/docs#quickstart)に従います。Snowflake専用のAPIキーを作成し、専用の1分あたり上限を設定します。
1. 各商品URL、動画URL、キーワードを`curl`で1回ずつ呼び出します。`watchlist`テーブルに対象を追加する前に、フィールドと件数を確認します。
1. Snowflakeで、上記の例に沿ってネットワークルール、シークレット、外部アクセス統合を作成します。続いて2つのプロシージャと2つのタスクを作成します。
1. 数日分のデータを読み込んだら、週次ビューとセマンティックビューを構築します。コメントと返品メモには同じ`AI_CLASSIFY`のテーマでラベルを付けます。
1. Cortex Agentを作成し、CoWorkでいくつか質問します。欠けている週を正しく報告できたら、週次のオートメーションを設定します。

データ収集の全機能は[EveryInfraカタログ](https://api.everyinfra.com/api/v1/social/catalog)に掲載しています。リモートMCPの設定は[MCPサーバーのページ](https://everyinfra.com/ja/mcp-server)、呼び出しごとの料金は[料金ページ](https://everyinfra.com/ja/pricing)をご覧ください。

## 参考文献

1. Snowflakeプレスリリース「Expedition 2026: CoCo and CoWork」（2026-11-06閲覧）：https://www.snowflake.com/en/news/press-releases/expedition-2026-coco-cowork/
1. Snowflakeプレスリリース「Business Ontology and Cortex AI Gateway」（2026-11-06閲覧）：https://www.snowflake.com/en/news/press-releases/expedition-2026-business-ontology-ai-gateway/
1. Snowflakeブログ「AI agents, governed data and business context」（2026-11-06閲覧）：https://www.snowflake.com/en/blog/ai-agents-governed-data-business-context/
1. Snowflakeブログ「Snowflake CoWork Adds Profiles, Dashboards and Automations」（2026-11-06閲覧）：https://www.snowflake.com/en/blog/snowflake-cowork-profiles-automations-dashboards
1. Snowflakeブログ「Snowflake CoCo cloud agents and automations」（2026-11-06閲覧）：https://www.snowflake.com/en/blog/snowflake-coco-cloud-agents-automations/
1. Snowflake CoWork製品ページ（2026-11-06閲覧）：https://www.snowflake.com/en/product/snowflake-cowork/
1. Snowflake「Pricing Options」ページ（2026-11-06閲覧）：https://www.snowflake.com/en/pricing-options/
1. Snowflake「Service Consumption Table」（2026-11-04発効、2026-11-06閲覧）：https://www.snowflake.com/legal-files/CreditConsumptionTable.pdf
1. Snowflakeドキュメント「Cortex Agents MCP connectors」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors
1. Snowflakeドキュメント「Cortex Agents MCP connectors」の「Limitations」セクション（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp-connectors#limitations
1. Snowflakeドキュメント「Cortex Agents」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents
1. Snowflakeドキュメント「Create and manage agents」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-manage
1. Snowflakeドキュメント「CREATE AGENT」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/sql/create-agent
1. Snowflakeドキュメント「Cortex Agents Run API」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-run
1. Snowflakeドキュメント「Cortex Agents resource budgets」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-resource-budgets
1. Snowflakeドキュメント「Cortex Agents code execution tool」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-code-execution-tool
1. Snowflakeドキュメント「Snowflake-managed MCP server」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-agents-mcp
1. Snowflakeドキュメント「CREATE MCP SERVER」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/sql/create-mcp-server
1. Snowflakeリリースノート「November 4, 2026: Background runs and additional supplementary tools for agents」（2026-11-06閲覧）：https://docs.snowflake.com/en/release-notes/2026/other/2026-11-04-mcp-async-agent-tools
1. Snowflakeドキュメント「Cortex AI Gateway custom apps and MCP servers」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/cortex-ai-gateway/tools/admin-custom-apps
1. Snowflakeドキュメント「Cortex Code Agent SDK, MCP and custom tools」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/cortex-code-agent-sdk/mcp-custom-tools
1. Snowflakeドキュメント「Cortex Code」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/cortex-code/cortex-code
1. Snowflakeドキュメント「Snowflake CoWork」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/snowflake-cowork
1. Snowflakeドキュメント「CoWork Automations」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/snowflake-cowork/automations
1. Snowflakeドキュメント「Cortex AI pricing」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/pricing
1. Snowflakeドキュメント「Cortex AI Functions」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql
1. Snowflakeドキュメント「AI_CLASSIFY」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/functions/ai_classify
1. Snowflakeドキュメント「AI_SENTIMENT」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/functions/ai_sentiment
1. Snowflakeドキュメント「Semantic views overview」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/views-semantic/overview
1. Snowflakeドキュメント「CREATE SEMANTIC VIEW」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/sql/create-semantic-view
1. Snowflakeドキュメント「External network access overview」（2026-11-06閲覧）：https://docs.snowflake.com/en/developer-guide/external-network-access/external-network-access-overview
1. Snowflakeドキュメント「Creating and using an external access integration」（2026-11-06閲覧）：https://docs.snowflake.com/en/developer-guide/external-network-access/creating-using-external-network-access
1. Snowflakeドキュメント「External network access examples」（2026-11-06閲覧）：https://docs.snowflake.com/en/developer-guide/external-network-access/external-network-access-examples
1. Snowflakeドキュメント「CREATE NETWORK RULE」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/sql/create-network-rule
1. Snowflakeドキュメント「CREATE SECRET」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/sql/create-secret
1. Snowflakeドキュメント「CREATE EXTERNAL ACCESS INTEGRATION」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/sql/create-external-access-integration
1. Snowflakeドキュメント「Writing stored procedures in Python」（2026-11-06閲覧）：https://docs.snowflake.com/en/developer-guide/stored-procedure/python/procedure-python-overview
1. Snowflakeドキュメント「CREATE TASK」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/sql/create-task
1. Snowflakeドキュメント「Introduction to tasks」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/tasks-intro
1. Snowflakeドキュメント「Snowpipe」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/data-load-snowpipe-intro
1. Snowflakeドキュメント「Overview of warehouses」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/warehouses-overview
1. Snowflakeドキュメント「Understanding compute cost」（2026-11-06閲覧）：https://docs.snowflake.com/en/user-guide/cost-understanding-compute
1. Snowflakeドキュメント「CORTEX_AGENT_USAGE_HISTORY view」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/account-usage/cortex_agent_usage_history
1. Snowflakeドキュメント「SNOWFLAKE_COWORK_USAGE_HISTORY view」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/account-usage/snowflake_cowork_usage_history_view
1. Snowflakeドキュメント「EXTERNAL_ACCESS_HISTORY view」（2026-11-06閲覧）：https://docs.snowflake.com/en/sql-reference/account-usage/external_access_history
1. MCP仕様「Authorization」（バージョン2025-06-18、2026-11-06閲覧）：https://modelcontextprotocol.io/specification/2025-06-18/basic/authorization
1. IETF RFC 9728「OAuth 2.0 Protected Resource Metadata」（2026-11-06閲覧）：https://datatracker.ietf.org/doc/html/rfc9728
1. IETF RFC 8414「OAuth 2.0 Authorization Server Metadata」（2026-11-06閲覧）：https://datatracker.ietf.org/doc/html/rfc8414
1. IETF RFC 7591「OAuth 2.0 Dynamic Client Registration Protocol」（2026-11-06閲覧）：https://datatracker.ietf.org/doc/html/rfc7591
1. Amazon SP-APIドキュメント「FBA report type values」（2026-11-06閲覧）：https://developer-docs.amazon/sp-api/docs/report-type-values-fba
1. National Retail Federation「Consumers expected to return nearly $850 billion in merchandise in 2025」（2026-11-06閲覧）：https://nrf.com/media-center/press-releases/consumers-expected-to-return-nearly-850-billion-in-merchandise-in-2025
1. Analytics India Magazine「Snowflake introduces Business Ontology and Cortex AI Gateway」（2026-11-06閲覧）：https://analyticsindiamag.com/ai-news/snowflake-introduces-business-ontology-cortex-ai-gateway-to-bring-ai-agents-into-enterprise-workflows
1. EveryInfra機能カタログ、GET /api/v1/social/catalog（2026-11-06閲覧）：https://api.everyinfra.com/api/v1/social/catalog
1. EveryInfraのリモートMCPアドレス（2026-11-06に検証）：https://api.everyinfra.com/mcp
1. EveryInfraのフルMCPアドレス用の保護リソースメタデータ（2026-11-06に検証）：https://api.everyinfra.com/.well-known/oauth-protected-resource/mcp/all

---

:::message
この記事は EveryInfra ブログの記事の転載です。原文：[Snowflake Cortex Agents MCP：SNS上の不満の声を返品データの横に取り込む方法](https://everyinfra.com/ja/blog/snowflake-cortex-agents-mcp-social-reviews)

EveryInfra は AI エージェントと開発者向けの従量課金データ API です：[everyinfra.com/ja](https://everyinfra.com/ja)
:::
