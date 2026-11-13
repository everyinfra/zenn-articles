---
title: "Qwen Agent MCPで、Taobao、JD、Xianyuのレビューを分析するアナリストを作る"
emoji: "📊"
type: "tech"
topics: ["mcp", "datascience", "aiagents", "api", "ai"]
published: true
---

:::message
この記事は [EveryInfra ブログ](https://everyinfra.com/ja/blog/qwen-agent-mcp-ecommerce-review-analyst) からの転載です（原文が正本です）。
:::

XianyuでLabubuのブラインドボックスを検索したところ、50件の結果が返ってきました。50件すべてが株式投資の本でした。次にLabubuの中国語名でもう一度検索しましたが、ツールから返されたのはまた同じ50冊の本でした。

どちらの呼び出しも成功し、どちらも課金されました。言語モデルがこの結果を検証せずに読み取ると、投資本の価格を中古Labubuの価格として報告してしまいます。この記事では、それを防ぐ方法を説明します。

目標は「ECレビューアナリスト」です。Qwenに「今DJI Pocket 3を買うならどこがいい？」のような質問を1つするだけで、Qwenが中国の3つのマーケットプレイスにまたがる6つのデータツールを呼び出します。Taobaoは大規模な消費者向けマーケットプレイスです。

JDは自社倉庫と配送の速さで知られる小売業者です。Xianyuは中古品のマーケットプレイスです。Qwenは結果を読み、1ページの購入判断レポートを書きます。モデルとデータツールの接続にはMCP（Model Context Protocol。モデルが外部ツールを呼び出せるようにするオープンなプロトコル）を使います。

構築方法は2つ紹介します。1つ目はAlibaba Cloud Model StudioとそのManaged Agentsサービスで、中国語のドキュメントではAgent APIと呼ばれています。2つ目はオープンソースのQwen-Agentフレームワークです。なお、テスト環境にはModel Studioのアカウントがありません。

そのため、Model Studioのパートは公式ヘルプページに基づく設定例で、実際には実行していません。Qwen-Agentのパートは実際に動かしたものです。Qwen-Agent 0.0.34をEveryInfraのリモートMCPエンドポイントに接続し、2026-11-03のシンガポール時間00:20からライブで呼び出しました。

> **要点**
>
> - Model StudioでカスタムMCPサーバーを使うには、`type: customer`とそのサービスIDに加え、各ツールを有効にする`mcp_toolkit`エントリを指定します。
> - 「MCP Squareの終了」で2026年に削除されたのは名前の挙がった11のサービス（2026-06-30に7つ、2026-04-01に4つ）で、MCP Square全体ではありません。
> - Qwen-Agent 0.0.34には`"type": "streamable-http"`と、バージョン2.0未満の`mcp`パッケージが必要です（今回は`mcp==1.30.0`を使用）。
> - DJI Pocket 3の価格の中央値は、TaobaoとJDの新品で2,399 CNY、Xianyuの中古で1,499 CNY（37.5%安）でした。
> - 比較1回あたりのコストは、データ呼び出しが$0.0243、qwen3.8-maxのトークンが約$0.2192（9倍）でした。

## Qwen Agent MCPレビューアナリストの役割

タスクはあえて狭く絞っています。ユーザーは1つの商品について1つの質問をし、アナリストは決まった順序で呼び出しを実行します。まずTaobaoを検索し、次にTaobaoのレビュー取得を試みます。

レビューが空の場合は、TaobaoのQ&Aセクションを読みます。ここでは購入を検討している人が質問し、過去の購入者が回答しています。続いてJDを検索してJDの商品ページを開き、Xianyuを検索します。最後に、価格帯、中央値、中古品の割引率、購入者によくある不安、出品者に聞くべき3つの質問をまとめた短いレポートを書きます。

モデルだけではこの仕事はできません。今日の価格を知らないからです。データAPIだけでもうまくいきません。Xianyuの出品者は商品の状態を自由記述で書き、「未開封」「メーカー交換品」「検品サービス付き」といった表現を使います。

誰かがそのタイトルを読んで仕分ける必要があります。MCPはこの2つをつなぎます。モデルはどのツールをどのパラメータで呼び出すかを決め、ツールはマーケットプレイスからデータを取得して構造化された行を返します。

![アイソメトリックのイラスト。右側では小さなロボットが机に向かい、虫眼鏡でレビューカードと値札を読んでいます。机の上のクリップボードには棒グラフとチェックマークが描かれています。左側では、ロボットのケーブルが小型サーバーのソケットの1つにつながっています。サーバーからはさらに3本のケーブルが、オンラインショップ、大型の小売倉庫、中古品の露店へ伸びています。](https://everyinfra.com/articles/qwen-agent-mcp-ecommerce-review-analyst/img-explain.webp)

*1つのモデル、1つのMCPエンドポイント、3つのマーケットプレイス* [この記事のために生成したイラスト](https://everyinfra.com/ja/mcp-server)

下の表に、実際の呼び出しで返された結果を示します。「行数」は返された行の数、「コスト」はEveryInfraのカタログ価格に基づく1回あたりの実際の請求額です。何も返らなかった呼び出しは自動的に返金されます。

| 機能 | 実行結果 | 1回あたりのコスト | レイテンシ |
| --- | --- | --- | --- |
| `taobao/search` | Pocket 3で20件、フィールドは9つのみ | $0.0135 | 1.8–3.7 s |
| `taobao/reviews` | 7商品とも0件、返金済み | $0 | 11.8–19.8 s |
| `taobao/questions` | Q&Aが20件（2026-03-01〜2026-10-21） | $0.001389 | 7.3 s |
| `jd/search` | 5件、うち4件がPocket 3 | $0.002472 | 6.1 s |
| `jd/product_detail` | 公式ストアのページは在庫切れ、価格は非表示 | $0.001389 | 11.4–14.0 s |
| `jd/reviews` | カタログで一時停止中、HTTP 503、課金なし | $0 | 0.1 s |
| `xianyu/search` | 中古出品50件、1,199–1,900 CNY | $0.005556 | 8.9–9.2 s |

この表から、すでにこのアナリストの限界が見えます。価格と状態のデータはすべて返ってきましたが、Taobaoのレビューは今回は何も返らず、JDのレビューは一時停止中です。以降では、まずModel Studioのルール、次にオープンソースでのセットアップ、最後に実際のデータと請求額を説明します。

## Model StudioのエージェントとMCP：モデル、フィールド、制限、4つのカスタムタイプ

Model Studioには「エージェント」と呼ばれる製品が2つあり、多くの記事で混同されています。1つ目はコンソールの「エージェントアプリケーション」です。アプリケーションページで作成し、フォームでMCPサービスにチェックを入れます。2つ目がManaged Agentsです。

`POST /agents`でエージェントを作成し、`POST /sessions`で実行を開始します。エージェントはModel Studioがホストするサンドボックス内で実行されます。API概要に記載されているベースアドレスは`https://{workspace_id}.{region}.maas.aliyuncs.com/api/v1/agentstudio`です。現在利用できるリージョンはcn-beijingのみです（[API概要](https://help.aliyun.com/zh/model-studio/managed-agents-api-overview)）。

モデル一覧は「エージェントの定義」ページ（2026-11-03に確認）によるものです。qwen3.8-max、qwen3.8-flash、qwen3.7-max、qwen3.7-plus、qwen3.6-plus、qwen3.6-flashが掲載されており、deepseek-v4.1-flash、deepseek-v4-pro、glm-5.3、kimi-k3などのサードパーティモデルもあります。実行モードはauto、performance、balanced、economyの4つです（[エージェントの定義](https://help.aliyun.com/zh/model-studio/managed-agents-agent-definition)）。

エージェントはアーカイブできますが、削除はできません。更新するとエージェント全体が置き換えられ、現在のバージョン番号を送る必要があります。セッションは作成時にエージェントのバージョンを固定するため、後から編集しても実行中のセッションには影響しません（[エージェントAPI](https://help.aliyun.com/zh/model-studio/agent-api)）。

`mcp_servers`フィールドは配列で、各要素には`type`と`name`の2つのキーがあります。typeは、MCPマーケットのサービスなら`official`、自分で登録したサービスなら`customer`です。値は`custom`ではなく`customer`なので注意してください。

実際にツールを有効にするのは`tools`フィールドです。`type`を`mcp_toolkit`に設定した要素を追加し、その`mcp_server_name`にサービス名を再度指定して、`configs`リストで各ツールを名前で有効にします（[エージェントの作成](https://help.aliyun.com/zh/model-studio/agent-create)）。

| フィールド | 値 | 備考 | 参照ページ（確認日） |
| --- | --- | --- | --- |
| `model` | 必須の`id`を持つオブジェクト（例：`qwen3.8-max`） | 任意で`enable_thinking`、`max_tokens`、`temperature`を指定可能 | エージェントの作成、2026-11-03 |
| `mcp_servers[].type` | `official`または`customer` | 独自のサーバーには`customer`を使用 | エージェントの作成、2026-11-03 |
| `mcp_servers[].name` | サービスID（例：`mcp-xxxxxxxx`） | 表示名を指定するとエラー`AGENT_010`になる | カスタムMCPとスキル、2026-11-03 |
| `tools[].type` | `builtin_toolkit`は1回、`mcp_toolkit`は複数回 | 個別に有効化していないツールは警告なしで欠落 | エージェントの作成、2026-11-03 |
| `permission_policy` | `{"type": "always_allow"}`または`{"type": "always_ask"}` | オブジェクトのみ、メインエージェントのみ | 組み込みツール、2026-11-03 |

2つのタイプは、チェックされるタイミングが正反対です。ここが最も間違えやすいポイントです。`official`の場合、作成時の呼び出しでは名前がチェックされません。タイプミスがあってもエージェントは作成されます。実行時にはツールが存在せず、モデルは「使えるツールがない」と返すだけです。

`customer`の場合、作成時の呼び出しで名前がチェックされ、サービスが登録されていなければHTTP 400と`AGENT_010`が返ります。そのため、自前のサーバーのほうがデバッグしやすくなります。まずコンソールで登録し、そのIDをエージェントに設定してください。ドキュメントには、カスタムMCPサービスの追加や編集はコンソールでしか行えないとも記載されています。この手順にはAPIがありません（[カスタムMCPとスキル](https://help.aliyun.com/zh/model-studio/managed-agents-custom-mcp-skills)）。

![Model StudioのカスタムMCPとスキルに関するヘルプページのスクリーンショット。ページには更新日時2026-10-16 10:41:23が表示されています。表では、外部エージェントツールを組み込みツールキット、typeがofficialのマーケットMCP、typeがcustomerの自前のアセットという3つのレベルに分けています。最後の行はティール色の枠で囲まれています。](https://everyinfra.com/articles/qwen-agent-mcp-ecommerce-review-analyst/img-shot-aliyun-mcp-types.webp)

*Model Studioのドキュメントでは、自前のMCPサーバーはtype customerに対応します* [Alibaba Cloudヘルプセンターのスクリーンショット](https://help.aliyun.com/zh/model-studio/managed-agents-custom-mcp-skills)

エージェントあたりのMCPサーバー数の上限は、コンソールアプリとManaged Agentsで異なります。両方の答えは末尾のFAQにあります。コンソールの上限は[公式MCPサービス](https://help.aliyun.com/zh/model-studio/official-and-third-party-mcp)のページに基づいています。

カスタムMCPのタイプも、2つのページで異なります。古い「カスタムMCP」のページには、スクリプトデプロイ、AIゲートウェイ、Alibaba Cloud OpenAPIの3つのデプロイ方法が記載されています（[カスタムMCP](https://help.aliyun.com/zh/model-studio/custom-mcp)）。新しいManaged Agentsのページには、プラグイン、スクリプトデプロイ、AIゲートウェイ、Alibaba Cloud OpenAPIの4つのアクセスタイプが記載されています。

費用に関する注記があるのはスクリプトデプロイだけです。`npx`または`uvx`を使う場合、サーバーはFunction Computeにデプロイされ、そちらで課金されます。`http`を使う場合は、すでに稼働しているエンドポイントを登録するだけなので、デプロイ費用はかかりません。AIゲートウェイタイプは、すでにAlibaba Cloud AIゲートウェイでMCPを管理しているチームに向いています。OpenAPIタイプは、Alibaba Cloud製品のAPIをMCPツールに変換します。プラグインタイプについては、ページに名前が記載されているだけです。

EveryInfraのようなサービスは、すでにリモートMCPエンドポイントを運用しています。そのため、スクリプトデプロイと`http`インストール方式を選ぶのが適切です。ドキュメントによると、この手順ではエンドポイントのアドレスと認証方式をModel Studioに伝えるだけです。何もデプロイせず、デプロイ費用もかかりません。Function Computeのドキュメントも、外部でホストされているサーバーについて同じ方法を勧めています（[Function Compute MCP](https://help.aliyun.com/zh/functioncompute/mcp-server)）。Alibaba Cloud自身の製品APIについては、OpenAPI MCPサーバーを扱う別のガイドがあります（[OpenAPI MCPサーバー](https://help.aliyun.com/zh/openapi/user-guide/openapi-mcp-server-guide)）。

## 「MCP Squareの終了」で実際に削除されたもの

Model StudioのMCP Squareについて検索すると、Squareが終了するという情報をよく目にします。私たちは元の告知を確認しました。オフラインになったのは特定のサービスだけで、Square全体ではありません。2026-01-05に公開された告知によると、7つのサービスが北京時間2026-06-30 00:00にSquareから削除されます。

対象はEver Art、旧Bocha AI検索、旧Xiaosuスマート検索です。残りはDouyinアシスタント、Qiankun Guadong、Wanxing Tianmu、株価データサービスです。BochaとXiaosuには新バージョンがあります（[告知117858](https://www.aliyun.com/notice/117858)）。

2026-03-26の2つ目の告知では、旧Qibaikeの4つのサービスが2026-04-01にオフラインになるとされています。対象は知的財産、企業登記、入札、リスクの各データです。新バージョンがあります（[告知118111](https://www.aliyun.com/notice/118111)）。3つ目の告知は削除ではなく課金に関するものです。

Managed Agentsの課金は2026-08-17 09:00に開始されました。セッションランタイムは稼働中1時間あたり0.5 CNYで、アイドル時間は無料です。新規ユーザーには30日間有効の無料枠10時間が付与されます。モデルのトークンと、ツールまたはMCPの呼び出しは別途課金されます（[告知118456](https://www.aliyun.com/notice/118456)）。

| 告知 | 公開日 | 発効日 | 対象範囲 |
| --- | --- | --- | --- |
| 117858 | 2026-01-05 | 2026-06-30 | 7サービス、うち2つは新バージョンあり |
| 118111 | 2026-03-26 | 2026-04-01 | 旧Qibaikeの4サービス、新バージョンあり |
| 118456 | 2026-07-16 | 2026-08-17 | Managed Agentsの課金開始、削除なし |

レビューアナリストにとって、2つの削除告知はほとんど影響しません。今回の設計では、自分で登録する`customer`サーバーを使います。Square内のどのサービスにも依存しません。課金開始の告知のほうが重要で、その数値は後述のコストセクションで使います。

## 1回の比較実行の流れ

![9秒ループのアニメーションによるフロー図。「あなたの質問」から「Qwenモデル」、「EveryInfraリモートMCP」へと進みます。続いてティール色の点が6つの機能のボックスを順に回ります。taobao/search（20件、$0.0135）、taobao/reviews（0件、無料）、taobao/questions（20件、$0.0014）。次にjd/search（5件、$0.0025）、jd/product_detail（在庫切れ、$0.0014）、xianyu/search（50件、$0.0056）です。点はQwenに戻り、「1ページの結論」が点灯します。jd/reviewsは一時停止中で課金されないという注記があります。](https://everyinfra.com/articles/qwen-agent-mcp-ecommerce-review-analyst/fig-02-agent-flow.ja.light.svg)

このフローの主経路は1本です。第1に、ユーザーが質問をQwenに送ります。システムプロンプトでは、アナリストに呼び出しの順序と出力形式を指示します。第2に、QwenがMCP経由でツール一覧を取得します。EveryInfraのリモートMCPは12個のツールを公開していますが、アナリストに必要なのは2つだけです。

`everyinfra_list_capabilities`はプラットフォーム、機能、パラメータを表示します。`everyinfra_call_api`は機能を実行します。第3に、Qwenは`platform`、`action`、`params`の値を変えて`everyinfra_call_api`を6回呼び出します。第4に、6つの結果がモデルに戻り、モデルが結論を書きます。

この流れには見落としやすい点が2つあります。1つ目は結果のサイズです。Xianyuの50件はQwenでは21,710トークンに相当します。空のTaobaoレビューの結果は432トークンだけです。モデルはターンごとに、それまでのすべての結果を読み直します。

そのため、最も長い結果は最後に置き、モデルが1回だけ読むようにします。2つ目は空の結果です。Taobaoのレビューが0件のとき、レスポンスには`"charged": false`と`"reason": "empty_result_refunded"`が含まれます。これにより、モデルはレビューを取得できないことをユーザーに伝えられます。空の結果を「悪いレビューはない」と解釈してはいけません。

## Model Studioの設定例（未実行）

以下の3つのブロックは、Model Studioのヘルプページに従ってフィールドごとに作成しています。名前、値、順序はドキュメントと照合済みです。Model Studioのアカウントを持っていないため、実行はしていません。テスト済みのコードではなく、出発点として扱ってください。実行してドキュメントと異なる点があれば、ぜひお知らせください。

ブロック1では、EveryInfraのリモートMCPをコンソールで登録します。Managed AgentおよびMCPの下にある「カスタム」タブを開きます。MCPサービスを作成するボタンをクリックします。スクリプトデプロイと`http`インストール方式を選択します。以下のJSONをサービス設定欄に貼り付けます。

Model Studio特有の注意点が1つあります。トランスポートはキャメルケースで`streamableHttp`と書きます。オープンソースのQwen-Agentでは、ハイフン付きの`streamable-http`を使います。混同しないでください。キーには動的ヘッダーのプレースホルダーを使っているため、セッションごとに独自のキーを渡せます。

```text
{
  "mcpServers": {
    "everyinfra": {
      "type": "streamableHttp",
      "url": "https://api.everyinfra.com/mcp",
      "headers": {
        "Authorization": "${Authorization}"
      }
    }
  }
}
```

ブロック2ではエージェントを作成します。登録後、MCP一覧からサービスIDをコピーします。形式は`mcp-xxxxxxxx`です。このIDを`mcp_servers[].name`と`tools[].mcp_server_name`の両方に指定する必要があります。

有効にするツールは2つだけです。呼び出しのたびに費用が発生する可能性があるため、ツール一覧では`everyinfra_call_api`に`destructiveHint: true`を付けています。そのためデフォルトのポリシーは`always_ask`で、無人で実行する場合は`always_allow`に切り替えられます。ドキュメントによると、承認はメインエージェントでのみ機能し、サブエージェントでは機能しません。

```bash
curl -X POST "https://${WORKSPACE_ID}.cn-beijing.maas.aliyuncs.com/api/v1/agentstudio/agents" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ecommerce-review-analyst",
    "description": "Compare one product on Taobao, JD and Xianyu",
    "model": {"id": "qwen3.8-max"},
    "system": "You are an ecommerce review analyst. First call everyinfra_list_capabilities to confirm parameters. Then call taobao/search, taobao/reviews (if 0 rows, call taobao/questions), jd/search, jd/product_detail and xianyu/search. Do not call jd/reviews. Write one page: price range and median per marketplace, used discount, top buyer worries, three questions for the seller.",
    "mcp_servers": [{"type": "customer", "name": "mcp-xxxxxxxx"}],
    "tools": [{
      "type": "mcp_toolkit",
      "mcp_server_name": "mcp-xxxxxxxx",
      "default_config": {"enabled": false, "permission_policy": {"type": "always_ask"}},
      "configs": [
        {"name": "everyinfra_list_capabilities", "enabled": true, "permission_policy": {"type": "always_allow"}},
        {"name": "everyinfra_call_api", "enabled": true}
      ]
    }]
  }'
```

ブロック3ではセッションを開始し、実際のキーを渡します。`mcp_configs`は`POST /sessions`のリクエストボディのトップレベルに置きます。その`mcp_server_name`には同じサービスIDを指定します。ドキュメントでは2つの点に注意を促しています。

サービス名が間違っていてもエラーにはなりませんが、設定は反映されません。また、`mcp_configs`は作成時のレスポンスに含まれません。確認には`GET /sessions/{id}`を使ってください（[セッションの作成](https://help.aliyun.com/zh/model-studio/session-create)）。

```bash
curl -X POST "https://${WORKSPACE_ID}.cn-beijing.maas.aliyuncs.com/api/v1/agentstudio/sessions" \
  -H "Authorization: Bearer $DASHSCOPE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "agent": {"id": "agent_xxx"},
    "environment_id": "env_xxx",
    "title": "Pocket 3 comparison",
    "mcp_configs": [{
      "mcp_server_name": "mcp-xxxxxxxx",
      "headers": {"Authorization": "Bearer <your EveryInfra API key>"}
    }]
  }'
```

Model StudioのVaultは、bashなどのサンドボックスツールからのリクエストを対象としています（[Vaultとエグレスゲートウェイ](https://help.aliyun.com/zh/model-studio/managed-agents-credential-security)）。カスタムMCPのリクエストはプラットフォーム自体が送信するため、Vaultは適用されません。キーは登録時のヘッダーか、前述のセッションの`mcp_configs`で送信してください。

## 同じMCPサーバーをオープンソースのQwen-Agentで使う（実際に実行した部分）

ホスト型のサービスを使いたくない場合もあるでしょう。アナリストを自分のマシンで動かしたいこともあります。オープンソースのQwen-Agentフレームワークは、リモートMCPサーバーに直接接続できます。QwenチームがGitHubで管理しています（[Qwen-Agentリポジトリ](https://github.com/QwenLM/Qwen-Agent)）。PyPIの最新リリースは、2026年2月16日公開の0.0.34です（[PyPIのqwen-agent](https://pypi.org/project/qwen-agent/)）。

クリーンなPython 3.11の仮想環境にインストールしました。MCPマネージャーで`https://api.everyinfra.com/mcp`に接続し、この部分は実際に動かしています。モデルの呼び出しにはDashScopeキーが必要ですが、手元にありませんでした。そのため、以下の`llm_cfg`と`bot.run`は設定例です。

```text
# pip install "qwen-agent[mcp]==0.0.34" "mcp==1.30.0" numpy soundfile tqdm python-dateutil
import os
from qwen_agent.agents import Assistant

tools = [{
    "mcpServers": {
        "everyinfra": {
            "type": "streamable-http",          # without "type", a bare url connects over sse
            "url": "https://api.everyinfra.com/mcp",
            "headers": {"Authorization": "Bearer " + os.environ["EVERYINFRA_API_KEY"]},
        }
    }
}]

# Configuration example: we did not call the model in this article
llm_cfg = {"model": "qwen3.8-flash", "model_server": "dashscope",
           "api_key": os.environ["DASHSCOPE_API_KEY"]}
bot = Assistant(llm=llm_cfg, function_list=tools,
                system_message="You are an ecommerce review analyst ... (same as the Model Studio example)")
for reply in bot.run(messages=[{"role": "user", "content": "Where should I buy a DJI Pocket 3 now?"}]):
    pass
print(reply[-1]["content"])
```

インストールコマンドに並ぶ依存パッケージは、どれも実際に出たエラーをもとに追加したものです。`qwen-agent[mcp]`だけでは、`qwen_agent.agents`のインポートが4回失敗しました。必要だったのはnumpy、soundfile、tqdm、python-dateutilです。numpyとpython-dateutilは、パッケージの別のオプショングループに含まれています。soundfileとtqdmは、宣言された依存関係のどこにも見つかりませんでした。これらを入れた後もインポートは`ImportError`で失敗し、原因は`mcp`パッケージでした。

`mcp` Pythonパッケージは2026年7月28日に2.0.0をリリースし、2026年10月2日には2.3.0になりました（[PyPIのmcp](https://pypi.org/project/mcp/)）。2.0から`streamablehttp_client`は`streamable_http_client`に改名されています。移行ガイドには、旧名は削除されたと書かれています（[MCP Python SDK移行ガイド](https://py.sdk.modelcontextprotocol.io/migration/)）。Qwen-Agent 0.0.34は今も旧名をインポートしています。

2026年8月25日に、あるユーザーがGitHubでこの問題を報告しています（[issue #949](https://github.com/QwenLM/Qwen-Agent/issues/949)）。私たちは`mcp==1.30.0`に固定して対処しました。その後はすべて正常に動作しました。

接続すると、Qwen-AgentがMCPツールをモデルにどう見せているかがわかります。各ツールは「server-tool」の形式で登録されます。そのため`everyinfra_call_api`は`everyinfra-everyinfra_call_api`になります。さらに`everyinfra-list_resources`と`everyinfra-read_resource`の2つのリソースツールが追加されます。ツールは合計14個です。

各ツールのスキーマからは`type`、`properties`、`required`だけが残ります。パラメータの説明とenum値は保持されるため、モデルは`mode`が`sync`か`async`であることを把握できます。ツール一覧は小さくありません。私たちの`tools/list`レスポンスは16,634文字で、Qwen-Agentに同梱されているQwenトークナイザーでは4,211トークンです。モデルはこれを毎ターン読み込みます。

| 項目 | Model Studio Managed Agents | オープンソース版Qwen-Agent 0.0.34 |
| --- | --- | --- |
| トランスポートの値 | `streamableHttp`または`sse` | `streamable-http`または`sse`（デフォルトは`sse`） |
| サーバーの指定方法 | コンソールで登録し、サービスIDを取得 | URLを直接記述 |
| 認証 | 登録時のヘッダー、またはセッションの`mcp_configs` | 設定内の`headers` |
| ツール名 | 元の名前のまま、1つずつ有効化 | 「server-tool」形式、すべて登録 |
| 承認 | `always_ask`で人に確認 | 対応なし、独自のチェックを追加 |
| 実行場所 | Model Studioのサンドボックス、セッション時間で課金 | 自分のマシン |
| 本記事での扱い | 設定例（未実行） | MCP部分は実行済み、モデル部分は例 |
| 重要な日付 | ガイド更新：2026年10月16日 | リリース：2026年2月16日 |
| 確認日 | 2026-11-03 | 2026-11-03 |

![暗いターミナルに2つのJSONパネルが並ぶレスポンスのスクリーンショット。左はシンガポール時間00:22:36のtaobao/reviewsで、countは0、chargedはfalse、reasonはempty_result_refunded、返金は成功。右は00:22:55のxianyu/searchで、countは50、先頭の商品は1299 CNYで「欲しい」が42000件、charged_usdは0.005556。主要なフィールドはティール色の枠で囲まれています。](https://everyinfra.com/articles/qwen-agent-mcp-ecommerce-review-analyst/img-shot-mcp-response.webp)

*Qwen-AgentのMCPマネージャー経由のライブ呼び出し。残高フィールド部分を切り取り、出品者名を非表示にしています。* [EveryInfraリモートMCPのレスポンスのスクリーンショット](https://api.everyinfra.com/mcp)

今回のテストでは、ツールの選択をモデルに任せていません。MCPマネージャーから各呼び出しを順番に送ったため、どのリクエストも簡単に確認できます。モデル主導の実行との違いは、ツールを選ぶのが誰かという点だけです。リクエストの内容は同じです。

`everyinfra_call_api`への`tools/call` 1回で、リクエストボディは`{"platform": "xianyu", "action": "search", "params": {"keyword": "...", "limit": 50}}`のようになります。すべてのプラットフォームとツールは、[MCPサーバーのページ](https://everyinfra.com/ja/mcp-server)と[リモートMCPセットアップガイド](https://everyinfra.com/ja/build/remote-mcp-setup)で確認できます。最初の手順は[クイックスタート](https://everyinfra.com/ja/docs#quickstart)にあります。

## ライブ検証：同じDJI Pocket 3を3つのマーケットプレイスで調べる

メインの商品にDJI Osmo Pocket 3を選んだ理由は3つあります。3つのマーケットプレイスすべてで大量に出品されていること。新品価格がほぼ揃っているため、中古の値引き率を計算しやすいこと。そして、後継機のPocket 4が2026年4月16日に発売されたことです。

Pocket 4はスタンダードキットが2,999 CNY、フルキットが3,799 CNYです（[DJI Pocket 4の発表](https://www.dji.com/cn/media-center/announcements/dji-release-osmo-pocket-4)）。そのため、旧モデルの中古価格はまさに今動いています。このセクションのCNY価格は、すべてマーケットプレイス上の表示価格です。2026年11月2日の対ドル基準値1 USD = 6.7146 CNYで換算すると、2,399 CNYは約$357です。

シンガポール時間2026年11月3日00:20に、`taobao/search`は20件を返しました。タイトルで分類すると、Pocket 3だけを記載したものが14件、Pocket 4など他のモデルも記載したものが5件、中古品が1件でした。

単一モデルの14件では、中央値が2,399 CNY、最安値が1,508 CNY、最高値が4,823 CNYでした。ちょうど2,399 CNYの出品が7件あります。両端の価格はどちらも相場ではありません。安い価格は部品取り用や整備済み品の可能性があります。高い価格はたいていセット品です。

このTaobao検索で返ったフィールドは、ID、リンク、タイトル、価格、元の価格、通貨、所在地、画像、プラットフォームの9つだけでした。販売数とショップ名はありません。Tmall限定のフィルターで再度検索すると、ショップ名や送料無料を含む14のフィールドが返りました。2つの呼び出しで結果が異なる理由はわかっていません。そのため、プロンプトには「フィールドが欠けている場合は、その旨を明記し、推測しない」という1つのルールが必要です。

Taobaoのタイトル20件のうち4件が「国家補助金」をうたっています。中国商務部による2026年の買い替え補助に関する通知では、国レベルの対象となるデジタル製品は4カテゴリーだけです。スマートフォン、タブレット、スマートウォッチ・スマートバンド、スマートグラスです。対象は価格6,000 CNY未満の商品で、補助率は15%、上限は1点あたり500 CNYです。カメラは対象に含まれていません。

この通知は、販売者が政府補助金の名称を偽って使うことも禁じています（[商務部の通知](https://www.mofcom.gov.cn/zcjd/gnmy/art/2026/art_c367aa0834354884844962df2dceebab.html)）。これだけで、これらの出品が誤りだと証明されるわけではありません。地方の制度が適用される可能性もあります。ただし、アナリストは販売者への質問として「どの補助金か、最終価格はいくらか」を加えるべきです。

シンガポール時間00:20:32に、`jd/search`は5件を返しました。そのうち4件がPocket 3です。JD直営の公式ストアと2つの正規販売店は、いずれも2,399 CNYで出品しています。フルキットは3,199 CNYです。

5件目は中古ショップが出品したPocket 4で、価格は1,489 CNYです。JDの検索結果では、レビュー数が「500,000+」「100,000+」のようなページ上のテキストで表示されます。高評価率は98%〜99%です。これらの数値は丸められているため、厳密な比較には向きません。

シンガポール時間00:21:57に、`jd/product_detail`で公式ストアの商品ページを開きました。在庫切れでした。JDは価格を「2??9」とマスクしており、フィールド`price_masked`はtrueでした。ページにはバリエーションが7つあり、保証は1年、重量は0.35 kgでした。

カタログにあるこのフィールドの説明は明確です。価格がマスクされている場合、表示価格を実際の価格として扱ってはいけません。そのため、アナリストは検索結果の2,399 CNYを使い、公式ストアが在庫切れであることを記録すべきです。

![横向きのドットプロット。1つのドットが1件の出品を表し、価格は1,000〜5,000 CNY。Taobaoは単一モデルの14件で、中央値2,399、その他のドットは1,508、2,799、2,987、3,199、3,499と、4,800付近に2つ。JDはPocket 3の4件で、中央値2,399と、3,199に1つ。Xianyuは中古の50件で、中央値1,499、ドットは1,199〜1,900に密集。破線は新品価格の2,399とPocket 4の価格2,999を示しています。](https://everyinfra.com/articles/qwen-agent-mcp-ecommerce-review-analyst/fig-03-price-ladder.ja.light.svg)

シンガポール時間00:22:55に、`xianyu/search`はMCP経由で50件を返しました。価格は1,199〜1,900 CNYです。中央値は1,499 CNYで、四分位数は1,388.5 CNYと1,607.25 CNYです。この中央値は、一般的な新品価格2,399 CNYより37.5%安くなっています。

これはPocket 4スタンダードキットの新品価格のちょうど半分です。50件中31件は2026年10月16日以降に出品されており、出品商品の大半は出品後3週間未満です。先頭の出品には「欲しい」が42,000件付いています。タイトルは「未開封・未アクティベート・メーカー交換品」で、価格は1,299 CNYです。

Xianyuでは、出品者が自分で状態タグを選びます。カードにタグがある場合はそれを使いました。ない場合は、「未開封」や「メーカー交換品」といった語をタイトルから読み取りました。結果は以下の表のとおりです。

| 条件 | 件数 | 中央値（CNY） | 新品価格2,399との比較 |
| --- | --- | --- | --- |
| 新品 | 4 | 1,749 | 27.1%安い |
| 未開封・メーカー交換品（タイトルより） | 3 | 1,350 | 43.7%安い |
| 新品同様 | 20 | 1,499.5 | 37.5%安い |
| わずかな使用感 | 8 | 1,549.5 | 35.4%安い |
| 軽い傷 | 2 | 1,288 | 46.3%安い |
| 目立つ損傷 | 1 | 1,350 | 43.7%安い |
| 状態表記なし | 12 | 1,449 | 39.6%安い |

意外なのは2行目です。「未開封、工場交換品」と書かれた3件の出品は、中央値が1,350 CNYです。これは「新品」と書かれた4件の中央値1,749 CNYより23%低い価格です。工場交換品は通常、アフターサービスで返品された製品です。保証、アクティベーション、DJI Care Refreshプランの扱いは未確認です。

これらの話題はXianyuのタイトルに繰り返し登場します。50件のうち18件がCare Refreshに、3件がメーカー交換品に言及しており、17件が有料の検品サービスを利用し、26件は`free_shipping`がtrueです。新品より安い未開封品に下すべき判定は「最もお得」ではなく、「最も多く質問すべき」です。

![状態別のXianyu価格中央値（CNY）を示す横棒グラフ。新品1,749（4件）、未開封・メーカー交換品1,350（3件、ティール）、新品同様1,499.5（20件）。わずかな使用感1,549.5（8件）、軽い傷1,288（2件）、目立つ損傷1,350（1件）、状態表記なし1,449（12件）。未開封のメーカー交換品は新品より23%安くなっています。](https://everyinfra.com/articles/qwen-agent-mcp-ecommerce-review-analyst/fig-04-xianyu-condition-bar.ja.light.svg)

今回のテストで最大のデータの欠落はレビューです。シンガポール時間00:22:36から、MCP経由で7つの商品について`taobao/reviews`を呼び出しました。1つはTmallのストアの商品で、1つはこの記事の後半で扱うLabubuの商品です。7回の呼び出しはすべて0件でした。

どのレスポンスにも`empty_result_refunded`とあり、課金はありませんでした。0件は「悪いレビューがない」という意味ではありません。この7商品で0件だった理由はわかりません。そこで、TaobaoのQ&Aセクションである`taobao/questions`に切り替えました。

シンガポール時間00:25:53に、`taobao/questions`は2026-03-01から2026-10-21までに投稿された20件を返しました。最も新しい2026-10-21の質問は、カメラが使いやすいかを尋ねるものです。次の2026-10-17の質問は、メモリーカードを自分で買う必要があるかを尋ねています。各質問にトピックのタグを付けました。1つの質問に複数のタグが付くこともあります。画質と写真の品質に関する質問は6件でした。

コンサート、車載、一人称視点の動画などの用途は5件でした。真贋、正規販売店、アクティベーションは5件、アクセサリーとストレージは4件、使いやすさは2件、価格は1件でした。

Q&Aの回答を事実として判定に書き写してはいけません。20件の質問はいずれも回答が1件しか返ってきませんでした。しかし`answer_count`を見ると、そのうち14件には2〜4件の回答があります。つまり、アナリストに見えているのは各質問の最初の回答だけです。この20件の回答のうち、少なくとも3点は公式スペックとの照合が必要です。

1点目です。2026-04-19に、購入者がマイクを内蔵しているかを尋ねました。回答は「マイクは購入が必要」でした。DJIのスペックページによると、Pocket 3はマイクを3基搭載しています（[Osmo Pocket 3の仕様](https://www.dji.com/cn/osmo-pocket-3/specs)）。購入者はおそらく外付けのワイヤレスマイクのことを指していたのでしょう。

2点目は、メモリーカードは自分で購入する必要があるという2026-10-17の回答です。この回答は正しいです。スペックページによると、カメラに内部ストレージはなく、最大1 TBのmicroSDカードに対応しています。

3点目です。コンサートについて尋ねた質問が2件あり、回答が食い違っています。2026-07-08の質問への回答は「使えるが、期待したほどではない」です。2026-04-11の質問への回答は「向いていない」です。回答が食い違う場合、アナリストは「購入者の意見が分かれている」と書く必要があります。どちらか一方の回答を判定として採用してはいけません。

もう1件、価格に関する回答があります。2026-03-04に購入者がほかの人の購入価格を尋ね、回答は2,700 CNYでした。今回のテストでは、TaobaoとJDの新品価格の中央値は2,399 CNYでした。ただし、1件の回答だけでは市場価格とはいえません。

ただし時期は2026-04-16のPocket 4発売と重なっており、旧モデルは半年で約300 CNY値下がりしたことになります。ほかに2件の回答では、対面でのアクティベーションは不要で、届いた個体は「未アクティベート・未開封」だったとしています。Xianyuのタイトルにも「未アクティベート」が繰り返し登場します。このカメラについて購入者と販売者が最も気にしているのは、明らかにアクティベーションの状態です。

| 質問（日付） | 購入者の回答 | 公式スペックまたはその他のデータ | アナリストが書く内容 |
| --- | --- | --- | --- |
| マイク内蔵？（2026-04-19） | 自分で購入 | 本体にマイク3基 | 回答が誤り、または外付けマイクを指している |
| メモリーカード付属？（2026-10-17） | 自分で購入 | 内部ストレージなし、最大1 TBのmicroSD | 正しい。カード代を予算に含める |
| コンサート向き？（2026-07-08、2026-04-11） | 「使える」と「向いていない」 | 該当するスペックなし | 購入者の意見が分かれている |
| 購入価格は？（2026-03-04） | 2,700 CNY | 今回のテストでの中央値は2,399 CNY | 1件のデータであり、市場価格ではない |

Q&AをXianyuのタイトルと並べると、購入者が気にしている点は同じ3つです。1つ目は、本物かどうか、どうアクティベーションするか。2つ目は、メモリーカードなどの部品を買う必要があるか。3つ目は、特定の用途に合うかです。この3点が、判定に含める販売者への質問になります。

JDのレビューについては、機能カタログで`jd/reviews`が利用不可とされています。シンガポール時間00:20:53に1回呼び出したところ、HTTP 503が返り、課金はなく、900秒後に再試行するよう案内がありました。そのため判定には「JDのレビューは取得不可」とだけ記載します。各機能の詳細は、[TaobaoレビューAPI](https://everyinfra.com/ja/taobao-api/reviews)、[Taobao Q&A API](https://everyinfra.com/ja/taobao-api/questions)、[JDレビューAPI](https://everyinfra.com/ja/jd-api/reviews)の各ページをご覧ください。

## アナリストが書くべき1ページの判定

モデルにデータを渡す前に、期待される回答を自分たちで書いておきます。その回答をもとにモデルの出力を評価します。以下がその回答です。数値はすべて、2026-11-03のシンガポール時間00:20〜00:29に実際に行った呼び出しから得たものです。

**新品を買う場合**。中央値は、Taobao（単一モデルの14件）でもJD（4件）でも2,399 CNY（約$357）です。JDの公式ストアは在庫切れです。新品価格は2つのマーケットプレイスでほとんど変わりません。領収書と正規保証を提供するストアを選んでください。商品名に補助金とある場合は、どの補助金で、最終価格がいくらかを確認してください。

**中古を買う場合**。Xianyuの中央値は1,499 CNY（約$223）で、新品より37.5%安くなっています。「ほぼ新品」の20件は中央値が1,499.5 CNYで、主な選択肢になります。「未開封、工場交換品」の3件は中央値が1,350 CNYです。最も安く見えますが、まずシリアル番号を確認してください。

**販売者に確認したい3つの質問**。シリアル番号から、DJIのサイトでアクティベーション日と保証期間を確認できますか。Care Refreshプランはまだ有効で、新しい所有者に引き継げますか。価格には、メモリーカードと商品名に記載されたすべての部品が含まれていますか。**データの欠落**。今回、Taobaoのレビューは何も返りませんでした。JDのレビューは一時停止中です。ここでの購入者の声は、TaobaoのQ&Aの20件だけです。

販売者向けには、判定の視点が変わります。Xianyuの50件の価格のうち半数は1,388.5〜1,607.25 CNYの範囲にあります。これが購入者が最もよく目にする価格帯です。1,388.5 CNY未満の価格は最も安い4分の1に入るため、購入者に理由を聞かれるでしょう。

100 CNYの値下げよりも、わかりやすいタイトルのほうが効果があるかもしれません。シリアル番号を確認できること、部品がすべて揃っていること、検品に対応していることを明記しましょう。TaobaoやJDで新品を販売する場合、2,399 CNYがすでに一般的な価格です。新品のPocket 4標準セットとの差は600 CNYしかないため、旧モデルの利幅は薄くなりつつあります。

モデルが判定を正しく書けたかどうかは、どう確認すればよいでしょうか。モデルの出力を上記の回答と1行ずつ比較します。モデル自身に採点させることはしません。チェック項目は5つです。3つの中央値は2,399、2,399、1,499 CNYになっているか。

中古の値引き率は37.5%前後か。レビューを「良好」とせず、2つのレビューの欠落を明記しているか。補助金をうたう記載に注意を促しているか。未開封の交換品を個別に取り上げているか。1つでも満たさない場合は、より大きなモデルを試す前にプロンプトを修正してください。

数値の誤りの多くは、モデルが中央値を頭の中で計算することから生じます。より安全な設計では、ツールが結果を返した後に数行のコードで価格を並べ替えます。四分位数はコードで計算し、モデルはその数値を文章にするだけです。これにより判定の再現性も確保できます。同じデータで2回実行すれば、言い回しが変わっても数値は同じになります。

## プロンプト設計：先に仕分け、次に判定する

アナリスト用のプロンプトは3回書き直しました。最終的に残ったルールは3つです。ルール1：仕分けてから数える。モデルは各行について、まず商品を確認します。

対象のモデルで合っているか、新品か中古か、単品かセットか部品かを確認します。確認を通過した件だけを価格の計算に使います。TaobaoのPocket 3の結果では、5件がほかのモデル名も含み、1件が中古でした。仕分けしないと中央値が下がります。

ルール2：欠けているものを明記する。フィールドの欠落、空の結果、一時停止中の機能は、必ず判定に記載します。その欠落によってどの判断の確度が下がるかも、判定に書く必要があります。レビューの結果が空でも、決して「不満なし」と扱ってはいけません。

ルール3：アドバイスは最後にだけ書く。それまでのターンでは収集と仕分けだけを行います。最後のターンで、ユーザー向けの文章を書きます。各ターンが短くなるため、トークン費用も抑えられます。

これらのルールによって、呼び出しの順序も決まります。最も安く速い呼び出しであるTaobao検索を最初に行います。これで商品名と、出品者が型番をどう表記しているかを確認します。出力が最も長い呼び出しであるXianyu検索は最後に回すため、その21,710トークンは一度しか読み込まれません。TaobaoレビューはTaobao Q&Aより前に置くため、結果が空ならそのままフォールバックに移れます。同じ6回の呼び出しでも、順序が違うと入力が数万トークン変わることがあります。

## 2つ目の商品：検索結果を鵜呑みにできない理由

別のカテゴリーでもアナリストを試すため、シンガポール時間00:27〜00:29に人気のコレクティブルに切り替えました。選んだのは、Pop MartのLabubu第3弾ビニール製ぬいぐるみシリーズです。中国語のシリーズ名は、おおよそ「この先ハイエナジー」という意味です。Taobaoの検索は20件を返しました。12件はチャームかブラインドボックスで、中央値は104.9 CNY、価格帯は60.9〜268 CNYでした。4件は大型ドール、4件はディスプレイケースなどのアクセサリーでした。

JDの検索は5件を返しました。Pop Martの旗艦店では、ブラインドボックス1個が99 CNY、6個入りのケースが594 CNYです。3つの転売ショップは109 CNY、1,146 CNY、2,399 CNYを提示していました。JDの詳細ページはまたも在庫切れで、価格は「9?」と伏せられていました。これらの結果は正常で、アナリストは通常どおり比較できます。JDでは、転売ショップの単品ボックスは旗艦店より10 CNY高くなっています。

問題はXianyuでした。最初のクエリは、ブランド名に中国語のシリーズ名と「第3弾」を加えたものです。50件が返りましたが、すべて株式トレードに関する本か講座のタイトルでした。Labubuは1件もありませんでした。2つ目のクエリでは、おもちゃの中国語名、シリーズ名、「ブラインドボックス」を使いました。

このクエリは、同じ50件の商品IDを順序だけ変えて返しました。タイトルの一例は、「価格と出来高」と「トレード」を扱った2冊セットの本です。どちらの呼び出しも通常の結果として、それぞれ$0.005556で課金されました。

![結果ごとにタイトルで分類した積み上げ横棒グラフ。Taobao検索の20件は、チャームまたはブラインドボックスが12件、大型ドールが4件、アクセサリーが4件です。XianyuのクエリAとクエリBはそれぞれ50件で、どちらもすべて株式トレードの本で、同じ商品でした。Xianyuの2つのクエリでは、一致する結果が0件でした。](https://everyinfra.com/articles/qwen-agent-mcp-ecommerce-review-analyst/fig-07-labubu-relevance-share.ja.light.svg)

アナリストにとっての教訓は明確です。50件を返したツールが、関連する50件を返したとは限りません。プロンプトには関連性を確認するステップが必要です。各タイトルに製品名が含まれているかを確認し、一致する件が半分未満ならクエリを変えるか、ユーザーに伝えます。このステップがないと、モデルは株式トレードに関する本50冊からLabubuの中古価格を計算してしまいます。

Pocket 3では、Xianyuの50件すべてが関連していました。Labubuでは1件もありませんでした。同じツール、同じ呼び出し方でも、品質は大きく違います。そのため、モデルが気づくことを期待するのではなく、関連性チェックを処理の流れに組み込むべきです。

## 比較1回あたりのコスト

まず、データ呼び出しから見ていきます。EveryInfraでは、呼び出しごとにカタログ価格で課金されます。`taobao/search`は1回あたり$0.0135、`xianyu/search`は$0.005556、`jd/search`は$0.002472です。`jd/product_detail`、`taobao/reviews`、`taobao/questions`はいずれも$0.001389です。

空の結果は返金されます。Taobaoレビューが空でQ&Aにフォールバックする場合、このフローの1回の実行コストは$0.024306です。Taobaoレビューが行を返した場合は$0.001389が加わり、合計$0.025695になります。最新の価格は[料金ページ](https://everyinfra.com/ja/pricing)と各機能のページでご確認ください。たとえば[Taobao検索API](https://everyinfra.com/ja/taobao-api/search)、[JD検索API](https://everyinfra.com/ja/jd-api/search)、[JD商品詳細API](https://everyinfra.com/ja/jd-api/product_detail)、[Xianyu検索API](https://everyinfra.com/ja/xianyu-api/search)です。

| 機能 | 1回あたりの料金 | 実行あたりの呼び出し回数 | 小計 |
| --- | --- | --- | --- |
| `taobao/search` | $0.0135 | 1 | $0.0135 |
| `taobao/reviews` | $0.001389、空の場合は返金 | 1 | $0 |
| `taobao/questions` | $0.001389 | 1 | $0.001389 |
| `jd/search` | $0.002472 | 1 | $0.002472 |
| `jd/product_detail` | $0.001389 | 1 | $0.001389 |
| `xianyu/search` | $0.005556 | 1 | $0.005556 |
| 合計 |  | 6 | $0.024306 |

モデルのコストは結果の長さで決まります。Qwen-Agentに同梱されているQwenトークナイザーで、実際の6件のレスポンスのトークン数を数えました。50件のXianyuは21,710トークン、TaobaoのQ&Aは4,923トークン、Taobaoの検索は3,961トークン、JDの検索は1,955トークンでした。

JD詳細は1,831、空だったTaobaoレビューの結果は432でした。合計は34,812トークンで、Xianyuだけで62%を占めます。このトークナイザーは、Model Studioが課金に使うものと異なる可能性があります。そのため、以下のモデルコストは推定値です。

前提条件は次のとおりです。ツール一覧は4,211トークン、システムプロンプトは500トークン、質問は100トークンです。ターンは7回で、ツール呼び出しが6回、最終回答が1回です。各ターンで、モデルはそれまでのすべての内容を読み込みます。

各ツール呼び出しの出力は80トークン、最終回答は1,500トークンです。キャッシュは使いません。この数値では、1回の実行で116,703トークンを入力として読み込み、1,980トークンを出力します。

![1回の実行における、機能ごとの結果のトークン数を示す横棒グラフ。taobao/search 3,961（11%）、taobao/questions 4,923（14%）、taobao/reviews 432（1%）。jd/search 1,955（6%）、jd/product_detail 1,831（5%）、xianyu/search 21,710（62%、ティール色）。Xianyuの50件がツール出力全体の62%を占めています。](https://everyinfra.com/articles/qwen-agent-mcp-ecommerce-review-analyst/fig-05-token-share-bar.ja.light.svg)

北京リージョンでは、qwen3.8-maxは入力100万トークンあたり12 CNY、出力100万トークンあたり36 CNYです。キャッシュヒットは1.5 CNYです。このページは2026年10月10日に更新されています（[qwen3.8-maxのモデルページ](https://help.aliyun.com/zh/model-studio/qwen3-8-max)）。qwen3.8-flashは入力0.8 CNY、出力2.7 CNY、キャッシュヒット0.1 CNYです。こちらのページは2026年10月29日に更新されています（[qwen3.8-flashのモデルページ](https://help.aliyun.com/zh/model-studio/qwen3-8-flash)）。換算レートは1 USD＝6.7146 CNYで、中国の外国為替管理当局が2026年11月2日付で公表した基準レートです（[SAFE基準レート](https://www.safe.gov.cn/AppStructured/hlw/RMBQuery.do)）。

qwen3.8-maxでは、1回の実行コストは約1.4717 CNY（約$0.2192）で、データコストの9.02倍です。qwen3.8-flashでは、約0.0987 CNY（約$0.0147）です。国際版サイトでは、シンガポールリージョンのqwen3.8-maxは100万トークンあたり入力$2、出力$6と記載されています。このページは2026年10月11日に更新されています（[qwen3.8-maxの国際版ページ](https://www.alibabacloud.com/help/en/model-studio/qwen3-8-max)）。換算後の北京の価格よりやや高くなります。

![キャッシュなしで1つの商品を6つの機能で処理した場合のUSD建てコストを示す横棒グラフ。EveryInfraのデータ呼び出し$0.0243、qwen3.8-maxのトークン$0.2192（ティール）、qwen3.8-flashのトークン$0.0147、3分間のセッション実行時間$0.0037。qwen3.8-maxでは、モデルのコストがデータ呼び出しの9倍になります。](https://everyinfra.com/articles/qwen-agent-mcp-ecommerce-review-analyst/fig-06-cost-per-run-bar.ja.light.svg)

| 利用形態 | データ | モデル | セッション実行時間 | 1回あたり | 1日100商品 | 30日間 |
| --- | --- | --- | --- | --- | --- | --- |
| Model Studio上のqwen3.8-max | $0.0243 | $0.2192 | $0.0037 | $0.2472 | $24.72 | $741.62 |
| Model Studio上のqwen3.8-flash | $0.0243 | $0.0147 | $0.0037 | $0.0427 | $4.27 | $128.12 |
| データ呼び出しのみ | $0.0243 | — | — | $0.0243 | $2.43 | $72.92 |

セッション実行コストは、料金を1時間あたり0.5 CNY、実行時間を1回あたり3分として計算しています。1回あたり約0.025 CNY（$0.0037）です。組み込みの`web_search`ツールは1回あたり0.03 CNY（約$0.00447）ですが、このフローでは使いません（[組み込みツール](https://help.aliyun.com/zh/model-studio/managed-agents-builtin-tools)）。

この表からわかることは2つあります。第1に、主なコストはデータではなくモデルです。qwen3.8-maxでは、各実行の費用の89%をモデルが占めます。第2に、費用を抑えるには、データの呼び出しを減らすのではなく、モデルが読む量を減らすことです。そこで、Xianyuを50件から20件に減らしてみました。

qwen3.8-maxのコストは$0.2192から$0.1959へと、約11%しか下がりませんでした。Xianyuは最後に呼び出されるため、その長い結果が読み込まれるのは1回だけだからです。より効果的な方法は2つあります。1つは、qwen3.8-flashで行を整理し、その要約だけをqwen3.8-maxに渡すことです。もう1つは、キャッシュを有効にすることです。

## アナリストにまだできないこと

| 問題 | 確認した内容 | 対処法 | 発生箇所 |
| --- | --- | --- | --- |
| Taobaoレビューなし | 7商品とも0件、返金済み | Taobao Q&Aを使い、欠落を明記する | 2026-11-03 |
| JDレビューが一時停止 | `jd/reviews`が利用不可、HTTP 503 | 「JDレビューは利用できません」と記載する | 2026-11-03 |
| JDの価格が伏せ字 | 詳細ページに「2??9」や「9?」と表示され、在庫切れ | 検索結果の価格を使い、在庫状況を併記する | 2026-11-03 |
| Taobaoのフィールドが一定しない | 9フィールドのときもあれば、14フィールドのときもある | 欠けているフィールドを報告するようモデルに指示する | 2026-11-03 |
| Xianyuのクエリのずれ | Labubuのクエリ2件で、在庫品の書籍が50件返された | 関連性を確認し、クエリを変更する | 2026-11-03 |
| 出品者の自己申告による状態 | 「新品」のほうが「未開封の交換品」より高い | シリアル番号と保証について質問する | 2026-11-03 |
| Model Studioでは未実行 | テストマシンにアカウントがない | 私たちの設定は一例として扱う | 2026-11-03 |
| MCPサーバーの上限 | コンソールアプリではMCPサーバーは5つまで、Managed Agentsでは上限の記載なし | ご自身のワークスペースでテストする | 2026-11-03 |
| トークナイザーの不一致 | オープンソースのトークナイザーで推定 | Model Studioの実際の請求額を確認する | 2026-11-03 |

本番運用の前に、注意すべき点があと2つあります。1つ目は返品です。オンライン購入の7日間理由なし返品に関する中国の暫定規則は、事業として運営するオンライン販売者に適用されます（[7日間返品ルール](https://www.cac.gov.cn/2017-03/15/c_1120630716.htm)）。Xianyuで中古品を売る個人の場合は、出品者とプラットフォームの規則によります。私たちのデータには、「開封後は7日間返品不可」と書かれたTaobaoの商品タイトルもありました。判定ではこの点を警告するべきです。

2つ目はレート制限です。私たちのテスト用キーでは、1分あたり10回まで呼び出せます。JDの詳細取得は初回でHTTP 429となり、7秒後の再試行で成功しました。本番環境では、フローに再試行を組み込むか、非同期モードを使ってください。同期レスポンスには毎回ヒントが含まれています。リクエストボディに`"mode": "async"`を追加すると、すぐにジョブIDが返されます。その後、`everyinfra_get_job`ツールまたは`GET /api/v1/jobs/{job_id}`で結果を取得します。

このアナリストでは、JDの詳細取得に11〜14秒、Taobaoのレビュー取得に12〜20秒かかります。1回の実行で12件の商品を扱うと、同期呼び出しを待つ間にセッションが長くなります。Model Studioでは、セッションが長いほど費用がかかります。私たちが比較したのは2商品だけで、非同期モードは使っていません。そのため、これはテスト結果ではなく助言です。

## QwenとMCPをめぐる最近の変更

最大の変更はオープンソース側にあります。`mcp` Pythonパッケージは2026-07-28に2.0.0をリリースしました。このリリースで`streamablehttp_client`が`streamable_http_client`に名称変更されました。その後、パッケージは2026-10-02に2.3.0になりました。Qwen-Agentは2026-02-16の0.0.34のままです。今日新しく環境を構築してバージョンを固定しないと、issue #949のインポートエラーが発生します。Qwen-Agentのセクションにある`pip install`の行では、`mcp==1.30.0`を残してください。

モデル側では、2026-10-29に更新されたqwen3.8-flashのページに、100万トークンあたり入力0.8 CNY、出力2.7 CNYと記載されています。2026-10-10に更新されたqwen3.8-maxのページには、コンテキスト1,000,000トークン、最大出力131,072トークンと記載されています。スナップショットはqwen3.8-max-0902です。

これはアナリストに2つの影響を与えます。1回の実行で使う入力トークンは116,703で、コンテキスト上限を大きく下回るため、ウィンドウに収めるためにXianyuの行を削る必要はありません。また、flashの料金はmaxの10分の1未満です。そのため、費用のセクションでは作業を分けています。flashで行を仕分けし、maxで判定を書きます。2026-10-11に更新された国際版のページには、qwen3.8-maxのシンガポールでの料金が米ドルで記載されています。

| 日付 | 変更内容 | このアナリストへの影響 |
| --- | --- | --- |
| 2026-07-28 | `mcp` 2.0.0でHTTPクライアントの名称が変更 | Qwen-Agent 0.0.34でインポートが失敗 |
| 2026-10-02 | PyPIで`mcp` 2.3.0が公開 | 新規インストールでは引き続き失敗するため、1.30.0に固定 |
| 2026-10-10 | qwen3.8-maxのページ：コンテキスト1M、出力131,072 | 1回の実行が余裕をもって収まる |
| 2026-10-11 | qwen3.8-maxの国際版ページ | シンガポールでの料金、100万トークンあたり$2と$6 |
| 2026-10-16 | カスタムMCPとスキルのガイドを更新 | `customer`、サービスID、`mcp_toolkit`を使用 |
| 2026-10-29 | qwen3.8-flashのページ | 仕分けステップ用の安価なモデル |

プラットフォーム側では、Model Studioが2026-10-16にカスタムMCPとスキルのガイドを更新しました。`customer`、サービスID、`mcp_toolkit`に関するルールは、この版に基づいています。この日付より古く、`custom`や表示名を使っているチュートリアルがあれば、新しいガイドに従ってください。私たちが参照したページや2026-11-03に取得したデータは、1か月以内に変わる可能性があります。そのため、アナリストは1回きりではなく定期的に実行してください。

商品を追加する場合は、まず機能一覧を確認してください。一覧は[TaobaoデータAPI](https://everyinfra.com/ja/taobao-api)、[JDデータAPI](https://everyinfra.com/ja/jd-api)、[XianyuデータAPI](https://everyinfra.com/ja/xianyu-api)の各ページにあります。ほかのエージェントフレームワークについては、[レビュー分析向けのエージェントスキル](https://everyinfra.com/ja/blog/agent-skills-competitor-review-analysis)を参照してください。[OpenAI Agents APIによるレビュー監視エージェント](https://everyinfra.com/ja/blog/openai-agents-api-review-monitoring-agent)もあわせてご覧ください。ModelScopeでのMCPについては、[XiaohongshuとDouyin向けのModelScope MCP](https://everyinfra.com/ja/build/modelscope-mcp-xiaohongshu-douyin)を参照してください。

## FAQ

### Qwen Agent MCPはリモートのMCPサーバーにどのように接続しますか？

Qwen-Agentの`function_list`に`mcpServers`ブロックを追加します。リモートサーバーの場合は、`"type": "streamable-http"`、`url`、および`headers`内の認証情報を設定します。`type`を指定しない場合、フレームワークはSSE経由で接続します。ツール名は「server-tool」の形式になります。

Model StudioのManaged Agentsでは、まずコンソールでサーバーを登録し、トランスポートを`streamableHttp`と記述します。質問の出典：「qwen agent mcp server」の検索候補（2026-11-03に確認）。根拠：[Qwen-Agentリポジトリ](https://github.com/QwenLM/Qwen-Agent)、[カスタムMCPとスキル](https://help.aliyun.com/zh/model-studio/managed-agents-custom-mcp-skills)。

### Qwen-AgentでImportError: streamablehttp_clientが発生するのはなぜですか？

バージョン2.0以降、`mcp`パッケージでは`streamablehttp_client`の名称が`streamable_http_client`に変更されました。Qwen-Agent 0.0.34はまだ古い名前を使っています。`mcp`のバージョンを2.0未満に固定してください。本記事では`mcp==1.30.0`を使用しました。質問の出典：[GitHub issue #949](https://github.com/QwenLM/Qwen-Agent/issues/949)（2026-08-25に作成）。根拠：[MCP Python SDK移行ガイド](https://py.sdk.modelcontextprotocol.io/migration/)。

### Model StudioのMCP Squareは終了したのですか？

いいえ。2026-06-30に、対象として指定された7つのサービスが提供終了しました。2026-04-01には、Qibaikeの4つの旧サービスが提供終了しました。その他のサービスは引き続き利用でき、自身のカスタムMCPサーバーにも影響はありません。質問の出典：Model Studio MCP Squareの中国語名での検索候補（2026-11-03に確認）。根拠：[お知らせ117858](https://www.aliyun.com/notice/117858)、[お知らせ118111](https://www.aliyun.com/notice/118111)。

### Model Studioのエージェント1つで、MCPサーバーはいくつ使えますか？

コンソールのエージェントアプリケーションのページには、同時に最大5つと記載されています。Managed Agentsの作成ページによると、`mcp_toolkit`は複数回指定でき、上限は記載されていません。コミュニティ記事にある10という数字は、ヘルプセンターには記載されていません。質問の出典：ヘルプセンターと開発者コミュニティの記事で記述が食い違っていたこと（2026-11-03に確認）。根拠：[公式MCPサービス](https://help.aliyun.com/zh/model-studio/official-and-third-party-mcp)、[エージェントの作成](https://help.aliyun.com/zh/model-studio/agent-create)。

### Qwen Agent APIとオープンソースのQwen-Agentは同じものですか？

いいえ。Model StudioのAgent API（Managed Agentsとも呼ばれます）はホスト型のサービスです。エージェントはModel Studioのサンドボックスで動作し、セッション時間とモデルのトークンに応じて課金されます。現在利用できるのはcn-beijingのみです。オープンソースのQwen-Agentは自分のマシンで動かすPythonフレームワークで、支払いはモデルの呼び出し分だけです。どちらもMCPを使えますが、設定は異なります。質問の出典：「qwen agent api」の検索候補（2026-11-03に確認）。根拠：[APIの概要](https://help.aliyun.com/zh/model-studio/managed-agents-api-overview)、[PyPIのqwen-agent](https://pypi.org/project/qwen-agent/)。

## 参考文献

1. [Managed Agents応用編：カスタムMCPとスキル（Alibaba Cloudヘルプセンター、2026-10-16更新）](https://help.aliyun.com/zh/model-studio/managed-agents-custom-mcp-skills)
1. [エージェント作成API：POST /agents（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/agent-create)
1. [Managed AgentsのMCP（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/managed-agents-mcp)
1. [エージェントの定義：モデルと実行モード（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/managed-agents-agent-definition)
1. [Managed Agents APIの概要と認証（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/managed-agents-api-overview)
1. [Agent API：操作とバージョン（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/agent-api)
1. [カスタムMCP（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/custom-mcp)
1. [公式MCPサービス：エージェントアプリケーション1つにつき最大5つ（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/official-and-third-party-mcp)
1. [セッションの作成：mcp_configs（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/session-create)
1. [Managed Agentsの組み込みツールと承認（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/managed-agents-builtin-tools)
1. [Vaultとエグレスゲートウェイ（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/managed-agents-credential-security)
1. [本番環境でのManaged Agents（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/managed-agents-production)
1. [Managed Agentsのマルチエージェント（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/managed-agents-multiagent)
1. [Managed Agentsのスキル（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/managed-agents-skill)
1. [qwen3.8-maxモデルページ（Alibaba Cloudヘルプセンター、2026-10-10更新）](https://help.aliyun.com/zh/model-studio/qwen3-8-max)
1. [qwen3.8-flashモデルページ（Alibaba Cloudヘルプセンター、2026-10-29更新）](https://help.aliyun.com/zh/model-studio/qwen3-8-flash)
1. [Model Studioのモデル料金（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/model-pricing)
1. [Model Studioの新規ユーザー向け無料枠（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/new-free-quota)
1. [Function Compute MCPサーバー（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/functioncompute/mcp-server)
1. [OpenAPI MCPサーバーガイド（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/openapi/user-guide/openapi-mcp-server-guide)
1. [Model Studioとは（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/what-is-model-studio)
1. [Model Studioのモデル一覧（Alibaba Cloudヘルプセンター）](https://help.aliyun.com/zh/model-studio/models)
1. [Alibaba Cloudお知らせ117858：MCP Squareの7サービスが提供終了（2026-01-05）](https://www.aliyun.com/notice/117858)
1. [Alibaba Cloudお知らせ118111：旧Qibaikeサービスの提供終了（2026-03-26）](https://www.aliyun.com/notice/118111)
1. [Alibaba Cloudお知らせ118456：Managed Agentsの課金（2026-07-16）](https://www.aliyun.com/notice/118456)
1. [qwen3.8-maxの国際向けモデルページ（2026-10-11更新）](https://www.alibabacloud.com/help/en/model-studio/qwen3-8-max)
1. [QwenLM/Qwen-Agent（GitHub）](https://github.com/QwenLM/Qwen-Agent)
1. [Qwen-Agent issue #949：streamablehttp_clientのインポートエラー（GitHub、2026-08-25）](https://github.com/QwenLM/Qwen-Agent/issues/949)
1. [qwen-agent 0.0.34（PyPI、2026-02-16）](https://pypi.org/project/qwen-agent/)
1. [mcp 2.3.0（PyPI、2026-10-02）](https://pypi.org/project/mcp/)
1. [MCP Python SDK移行ガイド](https://py.sdk.modelcontextprotocol.io/migration/)
1. [MCP仕様（2025-06-18）：トランスポート](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports)
1. [MCP仕様（2025-06-18）：ツール](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)
1. [MCP仕様（2025-06-18）：ライフサイクル](https://modelcontextprotocol.io/specification/2025-06-18/basic/lifecycle)
1. [EveryInfraリモートMCPエンドポイント](https://api.everyinfra.com/mcp)
1. [EveryInfra機能カタログ（無料、price_usd付き）](https://api.everyinfra.com/api/v1/social/catalog)
1. [SAFE：人民元レートの基準値（2026-11-02）](https://www.safe.gov.cn/AppStructured/hlw/RMBQuery.do)
1. [中国商務部：2026年消費財買い替え促進に関する通知](https://www.mofcom.gov.cn/zcjd/gnmy/art/2026/art_c367aa0834354884844962df2dceebab.html)
1. [オンライン購入における7日間の理由を問わない返品に関する暫定規則（中国国家インターネット情報弁公室）](https://www.cac.gov.cn/2017-03/15/c_1120630716.htm)
1. [DJI Osmo Pocket 4の発表（2026-04-16）](https://www.dji.com/cn/media-center/announcements/dji-release-osmo-pocket-4)
1. [DJI Osmo Pocket 3の発表](https://www.dji.com/cn/media-center/announcements/dji-release-osmo-pocket-3)
1. [Osmo Pocket 3の仕様（DJI）](https://www.dji.com/cn/osmo-pocket-3/specs)

---

:::message
この記事は EveryInfra ブログの記事の転載です。原文：[Qwen Agent MCPで、Taobao、JD、Xianyuのレビューを分析するアナリストを作る](https://everyinfra.com/ja/blog/qwen-agent-mcp-ecommerce-review-analyst)

EveryInfra は AI エージェントと開発者向けの従量課金データ API です：[everyinfra.com/ja](https://everyinfra.com/ja)
:::
