---
title: "ChatGPTのスケジュールタスクとMCPで週次の需要レーダーを作る"
emoji: "📊"
type: "tech"
topics: ["mcp", "openai", "api", "ai", "webscraping"]
published: true
---

:::message
この記事は [EveryInfra ブログ](https://everyinfra.com/ja/blog/chatgpt-scheduled-tasks-demand-radar) からの転載です（原文が正本です）。
:::

はい、構築できます。データも安価で、週1回の「需要レーダー」にかかるEveryInfraのAPI料金は$0.007226、52週分でも約$0.38です。計画すべきなのはお金ではなく承認です。2026年10月26日17:23（シンガポール時間、SGT）に、EveryInfraのリモートMCPサーバーのツール一覧を確認しました。

25個のツールのうち、データを取得する`everyinfra_call_api`と`everyinfra_search`の2つには、呼び出しのたびに料金が発生するため`readOnlyHint=false`と`destructiveHint=true`が設定されています。OpenAIの開発者ガイド「[Add custom MCP server](https://developers.openai.com/api/docs/guides/custom-mcp-server)」によると、ChatGPTは読み取りツールと書き込みツールの両方に対応しています。`readOnlyHint`のないツールは書き込みアクションとして扱われ、書き込みアクションにはデフォルトで承認が必要です。そのため、各チャットの最初の呼び出しでは確認画面が表示されます。許可を選択すると、そのチャットではその選択を記憶させることもできます。[スケジュールタスクに関する記事](https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt)には、アクションに承認が必要な場合はタスクが一時停止すると書かれています。

画面の前にいれば、クリックが1回増えるだけです。無人で実行するには、BusinessまたはEnterpriseワークスペースでこのアプリの承認要件を解除するか、作業を2つに分けます。データの取得は別のスケジューラーが行い、ChatGPTはそれを読んでレポートを書くだけにします。

> **要点**
>
> - 週1回のレーダーで使う呼び出しは10回です。内訳は、コミュニティごとの週間トップ投稿の取得1回、キーワード検索2回、Quoraの質問取得2回、コメントの読み込み5回です。APIコストは$0.007226です。1つのキーワードに対する`reddit/painpoint`の呼び出しは1回で$0.072かかり、約10倍です。
> - EveryInfra MCPの25個のツールのうち、11個は読み取り専用です。2つのデータツールは料金が発生するため、「読み取り専用ではなく、破壊的である可能性がある」と設定されています。OpenAIの開発者ガイドによると、ChatGPTはこうしたツールを使用できます。各チャットの最初の呼び出しで確認画面が表示され、許可を選択すると、そのチャットではその選択を記憶できます。一方、ヘルプセンターのFAQには、ProではカスタムMCPが読み取り・フェッチ専用であるとも書かれています。2つのページの記述は食い違っており、本記事では開発者ガイドに従います。
> - EveryInfraのURLにキーを含める接続（`?key=`）は2026-11-04に停止し、401を返します。ChatGPTにはリクエストヘッダーでキーを渡す機能がないため、接続URLを`https://api.everyinfra.com/mcp/all`にして認証でOAuthを選び、EveryInfraにログインしてください。キーの入力は不要です。ログイン経由の接続は、設定内容は確認済みですが、ChatGPT上ではまだ動作確認が取れていません。
> - `sort=newest`を指定した`reddit/search`が返す25件の投稿は、0.2〜24.4時間前の範囲しかカバーしません。コミュニティごとの週間トップ投稿の取得（`reddit/subreddit_posts`）は約7日間をカバーします。
> - 2026年10月4日以降、公式フォーラムでは、スケジュールタスク内の書き込みに関する問題がユーザーから報告されています。表示されるメッセージは「blocked by OpenAI's safety checks」ですが、同じプロンプトは通常のチャットでは動作します。読み取り専用のステップは引き続き動作します。OpenAIはこれらのスレッドで回答しておらず、同社のドキュメントには現在も、書き込みアクションは承認後に実行されると記載されています。

本記事でいう「需要レーダー」とは、毎週月曜の朝に実行される短いレポートです。キーワードと対象コミュニティは固定です。RedditとQuoraから、その週の不満と質問を拾い出します。不満と質問のそれぞれについて、最も有力な投稿を5件ずつ選び、リンクと1行の要約を添えます。

プロダクトマネージャー、SaaSマーケター、コンテンツプランナーが、1つの問いに答えるためのものです。その問いは「今週、ユーザーは何について話していたか」だけです。売上を予測することも、誰が正しいかを判断することもありません。

![需要レーダーのデスクを描いたアイソメトリックイラスト。左奥にある2つのリングが付いたブロックはカレンダーで、月曜のトリガーを表します。左側の低い台に載った幅広で平たい皿はレーダーで、キーワードでコミュニティをスキャンします。中央の長いトレイには縦に立ったカードが一列に並んでおり、APIが返す投稿と質問を表します。トレイの右端にある、横木を渡した2本の柱は権限ゲートで、読み取り専用ではないツールはここを通過する必要があります。手前の2つの小さなトレイは、カードを質問と不満に仕分けます。右手前の薄い紙の束は週次レポートで、一番上の紙はティール色です。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/fig-01-iso-radar-desk.light.svg)

見落とされがちなのが権限ゲートです。ChatGPTのスケジュールタスクは固定のスクリプトではありません。設定した時刻になると、モデルが指示を改めて読み、自らツールを呼び出します。ツールを呼び出す前には毎回、ChatGPTがツールのアノテーションとユーザーの権限設定を確認します。

そのうえで、ツールを呼び出すか、先にユーザーに確認を求めます。OpenAIの開発者ガイドによると、読み取り専用と指定されていないツールは、各チャットで最初に呼び出されたときに確認が表示され、「許可」を1回クリックすれば実行されます。画面の前にいれば、「確認を求める」はクリックが1回増えるだけです。誰も見ていない月曜の9:00では、「確認を求める」はタスクが止まることを意味します。以下の各セクションでは、次の問いに1つずつ答えます。どのプランのどの設定でこのゲートに人の操作が必要になるのか、そしてスケジュールタスクが止まらないようにするにはどうすればよいのか、という問いです。

## 結論：ChatGPTのスケジュールタスクはプランごとにどこまで動くか

下の表は、OpenAIのドキュメントから今回の用途に関係する制限をまとめたものです。各セルは、2026年10月26日時点で私たちが確認したOpenAIヘルプセンターの記載に基づいています。今回のテストではChatGPTアカウントを用意していないため、表はドキュメントの記載内容を示すもので、実際に動かした結果ではありません。

| 方式 | 有効なスケジュールタスク数 | 最短の繰り返し間隔 | カスタムMCP | スケジュールタスクでのデータツール（読み取り専用ではない） |
| --- | --- | --- | --- | --- |
| Free、Go | 3 | 1日1回（朝や夜などの時間帯で指定） | ヘルプセンターに記載なし。フォーラムのユーザー1人によると、Freeでは引き続き開発者モードが必要 | ドキュメントに記載なし。前提にしないでください |
| Plus | 5 | 1時間ごと、正確な時刻で指定可能 | ヘルプセンターに記載なし | ドキュメントに記載なし。前提にしないでください |
| Pro（$100 / $200 / $500） | 15 | 1時間ごと、正確な時刻で指定可能 | 可。ヘルプセンターのFAQでは読み取り（read）と取得（fetch）の権限のみ | 開発者ガイドでは読み取り・書き込みツールに対応し、書き込みには承認が必要。承認待ちになるとタスクは一時停止 |
| Business | 10 | 1時間ごと、正確な時刻で指定可能 | 管理者とオーナーのみ | 利用可。承認が必要な場合があり、承認を求められるとタスクは一時停止 |
| Enterprise、Edu | Enterprise 15、Edu 10 | 1時間ごと、正確な時刻で指定可能 | 管理者がロールごとに「Create plugins with MCPs」を付与可能 | 同上。加えてアクションごとの制御が可能 |

タスク数の上限、繰り返しのルール、Freeの時間帯は、[Scheduled tasks in ChatGPT](https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt)の「Review scheduled task limits」セクションに基づいています。カスタムMCPの対応範囲は[Connecting custom MCP servers in ChatGPT](https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt)のFAQに基づいています。Proの料金は[About ChatGPT Pro tiers](https://help.openai.com/articles/9793128)に基づいています。[What is ChatGPT Plus?](https://help.openai.com/en/articles/6950777-what-is-chatgpt-plus)によると、Plusは月額$20です。

FreeアカウントでカスタムMCPサーバーを追加できるかどうかは、ヘルプセンターに記載がありません。[2026年10月7日のフォーラムの返信](https://community.openai.com/t/1402157)では、Freeユーザーは引き続き設定の「Security and login」で開発者モードをオンにする必要があるとされています。これはユーザーによる報告のため、「未確認」としています。

![OpenAIヘルプセンターのページ「Scheduled tasks in ChatGPT」のスクリーンショット。「Review scheduled task limits」という見出しの下で、ティール色の枠が2つの段落を囲んでいます。スケジュールタスクは対象のChatGPTモデルに対応し、制限はタスク、プラン、ワークスペースによって異なります。有効なタスク数の上限は、FreeとGoが3、Plusが5、BusinessとEduが10、ProとEnterpriseが15です。Freeユーザーは、1回限りのタスク、または1日1回までの繰り返しタスクを、朝・午後・夜などの柔軟な時間帯で設定できます。1時間ごとのスケジュールや正確な時刻の指定には有料プランが必要です。枠の下には、スケジュールタスクが音声チャット、GPT、ロックされたチャットに対応していないことが記載されています。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/img-help-limits.webp)

*スケジュールタスクのプラン別の制限。ティール色の枠は、表の根拠にした2つの段落を示しています。2026年10月26日 17:47 SGTに取得* [OpenAIヘルプセンター・ChatGPTのスケジュールタスク](https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt)

この表から判断できることは二つあります。まず、PlusやProでは「ChatGPTが当社のデータツールを呼び出す」構成は手軽な方法ではありません。手軽なのは「三つの設計」セクションで紹介した分割設計です。管理者がこのアプリのアクションを許可してくれるBusinessまたはEnterpriseのワークスペースであれば、MCPに直接接続するのが最短の方法です。次に、週次のレーダーであれば、どの有料プランでも繰り返し間隔の制限に抵触することはありません。注意すべき数値は有効なタスク数です。Plusの枠は5つなので、製品カテゴリごとにタスクを一つずつ作ると、5カテゴリでプランの上限に達します。

## 1回の実行、6つのステップ

![フローアニメーション（8秒ループ）。上部の破線の枠は初回のみの設定を示します。Pluginsページで「Add custom MCP server」を選び、URLに /mcp/all を入力し、認証でOAuthを選んでログインし、このアプリの権限を設定します。続いて6枚のカードが時計回りに点灯します。1. スケジュール起動：月曜09:00、有料プランでは正確な時刻を指定可能。2. タスクを読む：ツールとキーワードは固定。3. MCPツールを呼び出す：tools/call everyinfra_call_api。4. 権限ゲート：読み取り専用ではないツールは確認を求められるか、ブロックされる場合がある。5. データを取得：rowsとcharged_usd。6. レポートを作成：不満と質問の上位5件。中央には、API費用は週$0.007226（呼び出し10回）、ChatGPTの利用分は別、と表示されています。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/fig-02-radar-flow.ja.light.svg)

ステップ1と6はChatGPTの担当、ステップ3と5はEveryInfraの担当です。ステップ2と4は、毎週同じように処理が実行されるかどうかを左右します。ステップ2では値を固定する必要があります。モデルは実行のたびに指示を読み直します。指示に選択の余地があると、モデルは週ごとに別のキーワード、並び順、件数の上限を選ぶ可能性があります。

そうなると、費用も結果も変わります。[スケジュールタスクに関するOpenAIの開発者ページ](https://developers.openai.com/codex/app/automations)も同じことを勧めています。ワークフローを自動のツール選択に頼らせたくない場合は、タスクのプロンプトでスキルやツールを名指しするよう書かれています。また、スケジュールを設定する前に、通常のチャットでプロンプトをテストするよう勧めています。ステップ4については、以下で別にセクションを設けて説明します。

## 手順1：ChatGPTにカスタムMCPコネクタを追加する（現在は開発者モード不要）

2026年10月1日、OpenAIのプラグインプラットフォーム責任者が[公式フォーラム](https://community.openai.com/t/1402157)に投稿しました。カスタムMCPサーバーに開発者モードは不要になり、対象となるすべてのユーザーがPluginsページを開いて「+」を選び、メニューから「Add custom MCP server」を選べるようになりました。以前「Developer mode」という名前だったワークスペース権限は名称が変わり、ヘルプセンターでは現在「Create plugins with MCPs」と呼ばれています。開発者ページ[Connect and test your plugin](https://developers.openai.com/plugins/deploy/connect-chatgpt)に手順が掲載されています。名前と説明を入力します。「Connection」で、`/mcp`パスを含む公開HTTPSのURLを入力します。

認証を設定します。リスクに関する警告を承諾します。「Create as a plugin」を選択します。その後、ChatGPTが検出したツールを確認します。

まず認証について考えます。[認証の章](https://developers.openai.com/plugins/build/auth)では、アカウントが必要なMCPサーバーに対し、OAuth 2.1を使用し、`/.well-known/oauth-protected-resource`でメタデータを公開するよう求めています。サードパーティのガイドである[AnythingMCPのChatGPTページ](https://anythingmcp.com/guides/clients/chatgpt)には、フォームの選択肢が3つ記録されています。順に「OAuth」「No authentication」「OAuth or no authentication」です。APIキーのヘッダーを入力するフィールドは表示されていません。2026年11月8日22:20（SGT）に、関連するアドレスを1つずつ確認しました。いずれも無料のリクエストです。

| アドレス | ステータス | ChatGPTにとっての意味 |
| --- | --- | --- |
| `POST /mcp/all`（認証情報なし） | 401。`WWW-Authenticate`に`resource_metadata`あり | MCPの認可仕様が定めるディスカバリーの方法。ChatGPTはここからログイン設定を見つける |
| [`/.well-known/oauth-protected-resource/mcp/all`](https://api.everyinfra.com/.well-known/oauth-protected-resource/mcp/all) | 200 | 保護リソースメタデータ（RFC 9728）。認可サーバーは`https://auth.everyinfra.com/oidc` |
| `https://auth.everyinfra.com/oidc/.well-known/openid-configuration` | 200 | `client_id_metadata_document_supported`は`true`、PKCEは`S256`のみ受け付ける |
| `POST /mcp?key=…` | 401 | URLにキーを含める接続は2026-11-04に停止 |
| User-Agentが`curl/8.5.0`の`POST /mcp`（10月26日に確認） | 200, 0.119 s | 当社はChatGPTの確認リクエストをブロックしない |

そのためChatGPTでは、認証にOAuthを選び、接続URLに`https://api.everyinfra.com/mcp/all`を入力します。画面の案内に従ってEveryInfraにログインし、アクセスを許可してください。キーの入力は不要です。以前は「No authentication」を選び、キーをURLの`?key=`に含める方法でしたが、この方法は2026-11-04に停止しました。2026年11月8日22:20（SGT）にこの形式のURLで`initialize`を送信したところ、サーバーは401を返し、エラーメッセージには「The EveryInfra API key in this connector's URL (?key=) stopped working on 2026-11-04」とありました。新しいURLも2つ示されています。ヘッダーにキーを入れる`https://api.everyinfra.com/mcp`と、ログイン方式の`https://api.everyinfra.com/mcp/all`です。ChatGPTはキーをヘッダーに入れてくれないため、使えるのは後者だけです。旧方式で追加したコネクタは、削除してから追加し直してください。ログイン経由の接続は、上の表の設定を確認しただけで、ChatGPT上で動作を確認してはいません。つながらない場合は、ヘッダーを自分で書けるクライアント（CodexやClaude Code）で`/mcp`に接続してください。

最後の行は意図的に検証しました。2026年9月3日、[ある開発者が公式フォーラムで報告](https://community.openai.com/t/feature-request-use-an-identifiable-user-agent-for-custom-mcp-server-validation/1394731)したところによると、ChatGPTの確認リクエストはUser-Agentに`curl/8.5.0`を使っていました。その開発者のファイアウォールはこれをスクリプトとしてブロックし、ChatGPTには「does not implement OAuth.」とだけ表示されました。当社が同じUser-Agentでinitialize呼び出しを送ったところ、200が返りました。

`?key=`方式を廃止した理由です。[RFC 6750のセクション2.3](https://www.rfc-editor.org/rfc/rfc6750)では、トークンを含むURLはログに記録されやすいとされています。ヘッダーでトークンを送れない場合を除き、この方法は使わないよう求めています。ChatGPTはこのURLを保存し、アプリ設定を閲覧できるワークスペースメンバーもこのURLを見ることができます。ログイン経由の接続では、ChatGPTが受け取るのはAPIキーではなく、EveryInfraがこのアプリ専用に発行するトークンです。接続後は、コンソールの「設定 → 接続済みアプリ」で、そのトークンと最後の利用、利用したAPIを確認でき、そこで接続をまとめて解除することもできます。

費用には注意してください。このログイン接続で消費されるのはお客様のアカウントのウォレット残高で、現時点ではこの接続だけに個別のUSD上限を設定することはできません。このレシピなら1年間のAPI料金は$0.40未満です。支出を確実に抑えたい場合は、Codexから`/mcp`に接続し、そのキーに`reddit/*`と`quora/*`の機能のみを許可し（`scope_mode=selected`）、USD建ての合計上限（`spend_limit_usd`。上限に達すると403 `key_spend_limit_reached`を返し、課金されません）と有効期限を設定してください。これらのフィールドは[当社のOpenAPI仕様](https://api.everyinfra.com/openapi.json)に記載されています。

![OpenAIヘルプセンターのページ「Connecting custom MCP servers in ChatGPT」にあるFAQのスクリーンショット。ティール色の枠が「Are apps and full MCP beta available to Pro users?」への回答を囲んでいます。回答の内容は、ProユーザーはApps SDKでアプリを構築できる、フルMCPはBusinessおよびEnterprise/Eduユーザーのみが利用できる、Proユーザーはread/fetch権限でカスタムMCPサーバーに接続できる、というものです。枠の上には別の回答があり、deep researchはカスタムアプリをread/fetchアクションにのみ使用でき、writeアクションには使用できないと書かれています。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/img-help-pro-readfetch.webp)

*ヘルプセンターのFAQ：Proでは、カスタムMCPはread/fetch権限のみ。2026年10月26日17:48（SGT）に取得* [OpenAIヘルプセンター · Connecting custom MCP servers in ChatGPT](https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt)

この記述は、OpenAIの開発者ガイドと一致しません。[Add custom MCP server](https://developers.openai.com/api/docs/guides/custom-mcp-server)の冒頭には「ChatGPTは、サーバーの読み取りツールと書き込みツールの両方をサポートしています」とあります。「Reviewing and confirming tool calls」のセクションでは、ChatGPTは`readOnlyHint`を使って読み取り専用のツールを判別し、これがないツールは書き込みアクションとして扱うと説明しています。書き込みアクションはデフォルトで承認が必要です。1つのチャット内ではツールごとに承認を記憶させることができますが、新しいチャットを開くか再読み込みすると、再び確認されます。

このルールに従えば、当社のデータツールはChatGPTで動作します。各チャットの最初の呼び出しで確認が表示されるので、許可を選択します。ブロックされるわけではありません。なお、Proアカウントではテストしていません。Proで異なる結果になった場合は、実際に確認した結果を信頼してください。

## 実際の1回のラウンドトリップ：ChatGPTの送信内容と当社の返却内容

ChatGPTはカスタムMCPサーバーのツールを呼び出す際、JSON-RPCの`tools/call`リクエストを1つ送信します。当社は17:34:47（SGT）に、同じリクエストをcurlで送信しました。変更点は1つだけで、キーを`Authorization`ヘッダーに入れたことです。

```bash
curl -s https://api.everyinfra.com/mcp \
  -H "Authorization: Bearer $EVERYINFRA_API_KEY" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":4,"method":"tools/call",
       "params":{"name":"everyinfra_call_api",
                 "arguments":{"platform":"reddit","action":"search",
                              "params":{"keyword":"robot vacuum keeps getting stuck","sort":"top","limit":10}}}}'
```

![当社APIの実際のリクエストとレスポンスを1組示したスクリーンショット（ダークなコードカード）。リクエストはPOST https://api.everyinfra.com/mcpで、AuthorizationヘッダーにはBearerと環境変数名が入っています。リクエストボディはツールeveryinfra_call_apiに対するtools/callで、reddit search、keywordはrobot vacuum keeps getting stuck、sortはtop、limitは10です。レスポンスは5.9秒で200が返っています。最初のテキストブロックのidはreq_41e9cf6d378040b2です。結果の最初の3件のタイトルは出会い系の投稿、副業の投稿、試験の解答用紙の投稿で、残り7件は省略されています。countは10です。billingではchargedがtrueで、charged_usd 0.000556がティール色の枠で囲まれています。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/img-api-roundtrip.webp)

*tools/callによる実際の1回のラウンドトリップ：費用は$0.000556で、10件の結果のうちロボット掃除機に関するものは1件もありません。2026年10月26日17:34:47（SGT）に呼び出し* [EveryInfra APIレスポンス](https://api.everyinfra.com/mcp)

呼び出しには5.9秒かかり、10件が返り、費用は$0.000556でした。これは[カタログ](https://api.everyinfra.com/api/v1/social/catalog)にある`reddit/search`の`price_usd`と一致します。返された10個のフィールド（`id`、`url`、`title`、`text`、`author_username`、`community`、`like_count`、`comment_count`、`posted_at`、`platform`）は、すべてカタログの`response_fields`に含まれています。カタログにないフィールドはありません。ただし、10件の結果のどれも「vacuum」に触れていません。1件目は63,584ポイントの出会い系の投稿でした。

理由はカタログを見ればわかります。`reddit/search`の対象期間は直近1か月に固定されており、`sort=top`はスコア順に並べ替えます。5語のフレーズは、いずれか1語が一致すれば投稿にマッチします。その結果、Reddit全体でその月に最もスコアが高かった投稿のうち、たまたま「stuck」や「getting」を含むものが返されます。

17:35:11（SGT）に、同じフレーズを`sort=relevance`で送信しました。10件すべてに「vacuum」が含まれ、そのうち2件は5語すべてを含んでいました。つまり、APIは正しく動作していました。問題はモデルが選んだパラメータです。

チャットであれば、おかしな結果を見て修正できます。スケジュールされたタスクでは誰も確認しないため、モデルが毎週的外れな投稿に料金を払い続けるおそれがあります。そのため、ステップ2では`sort`を固定し、2〜3語のキーワードフレーズを使う必要があります。また、レポートには呼び出しごとに返された行数と一致した行数を記載させる必要があります。そうすれば、その週のデータに問題があったかどうかが一目でわかります。

![横棒グラフ。7回の呼び出しについて、キーワードに一致した結果の件数を示しています。一致とは、キーワードのすべての語がタイトルまたは本文に含まれることを指します。robot vacuum（newest）：25件中22件。Roborock：25件中18件。robot vacuum mop：25件中6件。password manager：25件中12件。Dreame：25件中3件（ティール色）。robot vacuum keeps getting stuck（relevance）：10件中2件。同じフレーズをtopで検索したもの（MCPの呼び出し）：10件中0件（ティール色）。サブタイトルには、ブランド名Dreameは「dream」のように検索されること、長いフレーズをtopで検索するとサイト全体のその月のトップ投稿が返されることが書かれています。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/fig-05-relevance.ja.light.svg)

もう1本のティール色のバーは、ロボット掃除機ブランドのDreameです。17:31:36（SGT）の時点で、25件の結果のうちDreameに関するものは3件だけでした。残りはr/Dreamsの夢に関する投稿や小説の広告でした。検索が「Dreame」をdreamという単語のように扱ったためです。一般的な単語に見えない名前のRoborockは、25件中18件とはるかに良い結果でした。ブランド名が一般的な英単語に似ている場合、キーワード検索の結果はノイズだらけになります。`reddit/subreddit_posts`でブランド独自のコミュニティを指定するか（Dreameにはr/Dreame_Techがあります）、キーワードにカテゴリーを表す語を加えてください。

## データツールが承認を求める理由：25個のツールのアノテーション

![当社APIの実際のレスポンスのスクリーンショット（ダークなコードカード）。リクエストはメソッドtools/listを指定したPOST https://api.everyinfra.com/mcpです。レスポンスは200で、25個のツールのうち3個が表示されています。everyinfra_list_capabilitiesのannotationsはreadOnlyHint true、openWorldHint false、destructiveHint falseです。everyinfra_call_apiはreadOnlyHint false、openWorldHint true、destructiveHint trueで、ティール色の枠で囲まれています。everyinfra_searchも同じ値で、同じくティール色の枠で囲まれています。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/img-api-tools.webp)

*実際のtools/listレスポンス：どちらのデータツールも読み取り専用とはマークされていません。2026年10月26日17:23（SGT）に取得、キー不要* [EveryInfra MCPレスポンス](https://api.everyinfra.com/mcp)

MCP仕様では、各ツールにいくつかのアノテーションを付けられます。[2025-11-25版仕様のToolAnnotations定義](https://modelcontextprotocol.io/specification/2025-11-25/schema#toolannotations)によると、`readOnlyHint`がtrueの場合はツールが環境を変更しないことを意味し、デフォルトはfalseです。`destructiveHint`はツールが読み取り専用でない場合にのみ意味を持ち、デフォルトはtrueです。`openWorldHint`は、ツールが外部の不特定のエンティティとやり取りすることを意味し、デフォルトはtrueです。[OpenAIのプラグインドキュメント](https://developers.openai.com/plugins/plan/tools)では、ツールが状態を変更できない場合にのみ`readOnlyHint`をtrueに設定するよう求めています。

同ドキュメントによると、アノテーションはChatGPTとCodexが確認や安全に関する挙動を決める際に役立ちます。アノテーション自体は何も行いませんが、ホストはアノテーションを使っていつ確認を求めるかを判断します。たとえば、CodexにはMCPの設定`default_tools_approval_mode`があります。[OpenAIのMCPページ](https://learn.chatgpt.com/docs/extend/mcp)によると、値を`writes`にすると「読み取り専用とマークされていないツールについて確認を求めます」。ChatGPTも同様です。[Add custom MCP server](https://developers.openai.com/api/docs/guides/custom-mcp-server)によると、ChatGPTは`readOnlyHint`を使って読み取り専用のツールを判別し、これがないツールを書き込みアクションとして扱い、書き込みアクションにはデフォルトで承認を求めます。

![EveryInfra MCPの25個のツールをアノテーションで3グループに分けた横棒グラフ。読み取り専用（readOnlyHint true）：11個、グレー。読み取り専用ではなく、破壊的な可能性があるもの：5個、ティール色。2つのデータツール（everyinfra_call_apiとeveryinfra_search）を含みます。読み取り専用ではないが破壊的でもないもの（キーやチケットの作成など）：9個、グレー。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/fig-03-tool-annotations.ja.light.svg)

当社の25個のツールのうち、11個は読み取り専用です。これらのツールは、機能、CAPTCHAの種類、キー、チケットを一覧表示します。また、ジョブ、アカウント、使用量、リクエストログ、請求情報、1件のチケットを読み取ります。1つはチャージ用のリンクを取得します。

残りの14個は読み取り専用ではなく、そのうち5個は破壊的な可能性があります。5個のうち2個は、データツールの`everyinfra_call_api`と`everyinfra_search`です。残りの3個は今回の作業とは関係ありません（そのうち1個はキーを削除します）。では、なぜデータツールは「読み取り専用」と表示されないのでしょうか。

これらのツールはReddit上では何も変更しません。しかし、ウォレットから料金が差し引かれるので、環境は変化します。私たちのルールはシンプルです。外部サイトにアクセスするツールには`openWorldHint=true`を設定します。有料の呼び出しには必ず`destructiveHint=true`を設定します。

私たちはこれが正しいと考えています。お金を使うツールが「読み取り専用」を名乗ると、ホストは確認なしで実行を許可します。そうなると、モデルが誤ってループしたときに止める機会がありません。

![OpenAIの開発者向けページ「Define tools」のスクリーンショット。見出し「Plan safety annotations」の下で、ティール色の枠が3つの箇条書きを囲んでいます。readOnlyHintは、ツールが状態を変更できない場合にのみtrueです。destructiveHintは、ツールが元に戻せない、または元に戻すのが難しい結果を引き起こしうる場合にtrueです。openWorldHintは、ウェブ検索のような読み取り専用アクションを含め、ツールが公開インターネットや範囲の定まらない外部エンティティにアクセスする場合にtrueです。枠の下には、アノテーションはサーバー側の認可、入力の検証、影響の大きいアクションに対する確認の代わりにはならないと書かれています。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/img-dev-annotations.webp)

*OpenAIによる3つのアノテーションの定義。ティール色の枠は、私たちが使用している3つの箇条書きを示しています。2026年10月26日17:49 SGTに取得* [OpenAI Developers · Define tools](https://developers.openai.com/plugins/plan/tools)

アノテーションに加えて、ChatGPTにはユーザー自身が設定する権限があります。[ChatGPTでのアプリ権限の管理](https://help.openai.com/articles/20001495)には4つのレベルが記載されています。「常に確認」は、読み取りと変更の前に確認します。「読み取りアクションを許可」は、読み取りは確認なしで行い、変更の前に確認します。「低リスクのアクションを許可」は、低リスクのアクションを承認し、より高リスクのアクションは承認が必要になるか、拒否される場合があります。

「すべてのアクションを許可」では、対応しているアクションがそれ以降確認なしで実行されます。この最後のレベルは、1つのアプリまたは1つのアカウントに対してのみ設定できます。アカウント全体やワークスペース全体のセレクターでは選択できません。同じページによると、「常に許可」ボタンは、管理対象ワークスペースのメンバーには表示されず、追加の安全性レビューが必要なアクションにも表示されません。[プラグインの変更履歴](https://developers.openai.com/plugins/changelog)によると、これらの「確認のタイミング」の設定は2026年6月12日に公開されました。

| 権限設定 | 当社のデータツール（読み取り専用ではなく、破壊的な可能性あり）の動作 | スケジュールタスクでの結果 |
| --- | --- | --- |
| 常に確認 | 毎回確認されるので、許可を選択 | タスクはユーザーの操作を待つ |
| 読み取りアクションを許可 | 読み取りではないため確認される | 同上 |
| 低リスクのアクションを許可（デフォルト） | 破壊的と指定されているため、低リスクとは見なされない可能性が高く、確認される | 同上 |
| すべてのアクションを許可（1つのアプリのみ） | 対応しているアクションは以降確認なし | 無人で実行可能。ただし、スケジュール実行が安全性チェックでブロックされたというフォーラムでの報告あり |

2列目と3列目は、ドキュメントに基づく私たちの解釈です。ChatGPTは「低リスク」の判定ルールを公開していません。通常のチャットでは、どのレベルでも、確認は最大で初回呼び出し時の1回だけです。許可を選択してそのチャットで記憶させれば、そのチャットでは再び確認されません。スケジュールタスクには、もう1つ知っておくべき点があります。以下のフォーラムの報告はその点に関するものです。

![生成されたイラスト。夜空の下に2つの浮かぶ台座があり、それぞれに丸い頭の小さなロボットがいます。左側では、ロボットが建物を変化させるレバーに手を伸ばしていますが、レバーの前には半透明の琥珀色のガラスゲートが立っています。ロボットの手はゲートで止まり、近くには止まった砂時計が浮かんでいます。右側では、同じロボットが開いた窓から本棚に手を伸ばし、ティール色に光る文書をそっと引き出しています。その横のトレイには1枚の紙が現れています。画像に文字はありません。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/img-explain.webp)

*コンセプト：スケジュールタスクでは、承認が必要なアクションは人の操作を待ち、読み取りは完了する。生成されたイラストであり、データではない* [この記事のためにgpt-image-2.5で生成。仕組みの出典：OpenAI Help Center · Scheduled tasks in ChatGPT](https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt)

2026年10月4日、あるユーザーがOpenAIフォーラムに[リグレッションの報告](https://community.openai.com/t/1403242)を投稿しました。このユーザーの週次スケジュールタスクは、Gmailを読み、調査を行い、どのメールを送るかを決めます。Gmailは「すべてのアクションを許可」に設定され、タスクのプロンプトでも事前に承認を与えていました。しかし、メールを送信してGoogleスプレッドシートに書き込むステップで、呼び出しがブロックされました。

表示されたメッセージは「This tool call was blocked by OpenAI's safety checks」だけで、それ以外の詳細はありませんでした。報告には9月30日から10月3日までの失敗が記載されています。10月5日には、別のユーザーが同じ種類のタスクプロンプトを通常のチャットに貼り付けたところ、Trelloへの書き込みが8回中8回成功しました。

スケジュール実行では、書き込みがブロックされました。10月8日の時点でも、GitHubへの書き込みが日によって成功したり失敗したりするという返信が続いていました。一方、メールの読み取りと仕分けは引き続き動作していました。[10月6日から8日にかけての別の報告](https://community.openai.com/t/chatgpt-scheduled-tasks-fail-to-write-to-github-and-automatically-disable-themselves-after-failed-executions-without-user-authorization/1404882)はさらに深刻です。GitHubへの書き込みがブロックされた後、毎時実行のタスクが実行から約64〜65秒後に無効化され、次の実行は行われませんでした。これらはユーザーによる報告であり、OpenAIはこれらのスレッドで調査結果を示していません。

OpenAIのドキュメントには、書き込みアクションは承認が必要で、承認後に実行されると書かれています。スケジュールタスクで書き込みができないとは書かれていません。これらの報告からわかるのは、スケジュール実行で、承認済みの書き込みが安全性チェックによってブロックされた例を一部のユーザーが経験したということです。ChatGPTは当社のデータツールを書き込みアクションとして扱います。したがって、レーダーを無人で確実に実行する必要がある場合は、データ取得をスケジュールタスクの中に入れないでください。

## 3つの設計：自分に合うのはどれか

ChatGPTではデータツールに承認が必要ですが、スケジュール実行時には承認する人がいません。そこで選択肢は3つあります。それが許可されているワークスペースでゲートを開く方法、データ取得をChatGPTの外に移してChatGPTには読み取りだけをさせる方法、そしてChatGPTのスケジュールタスクを使わない方法です。

| 設計 | 対象 | 方法 | コストとリスク |
| --- | --- | --- | --- |
| A. MCPを直接利用 | Business、Enterpriseワークスペース | 管理者がアプリを作成・公開し、そのアプリに「すべてのアクションを許可」を設定したうえで、通常のチャットでタスクを一度手動で実行する | スケジュール実行が安全性チェックでブロックされる場合があるというフォーラムでの報告あり。ログイン経由の接続には個別のUSD上限を設定できないため、費用を固定したい場合はCodexで`/mcp`に接続する |
| B. 分離：外部で取得し、ChatGPTは読み取り | Plus、Pro、個人アカウント | 月曜08:00にcron、n8n、またはGitHub Actionsが当社のREST APIを呼び出し、Google DriveまたはGitHubにファイルを書き込みます。09:00にChatGPTのタスクがGoogle DriveまたはGitHubのコネクタを使ってそのファイルを読み取り、レポートを作成します | 追加のスケジューラーが1つ必要です。ChatGPTは読み取りだけを行うため、最も安定した構成です |
| C. ChatGPTのスケジュールタスクを使わない | 完全な自動化を求め、少しコードを書ける人 | 自前のcronからリモートMCPツールを指定してOpenAI Responses APIを呼び出します（`require_approval: "never"`、`allowed_tools`）。または、Claude Codeのクラウドスケジュールタスクを使います | トークン単位で課金されます。ChatGPTのインターフェース内では利用できません |

設計Bは、同じフォーラムスレッドに寄せられた、エンタープライズ向け自動化を構築している人からの返信に基づいています。AIが判断を下し、構造化されたレコードを書き出します。ユーザーが管理する決定論的なプログラムがアクションを実行します（[2026年10月7日の返信](https://community.openai.com/t/1403242)）。需要レーダーの場合、この「アクション」にあたるのがデータ取得です。スケジューラーは人の承認なしでEveryInfraを呼び出せます。ChatGPTのステップはファイルを読むだけなので、この境界では安定して動作する側にあります。

私たちは[EveryInfraを呼び出す毎日のn8nワークフロー](https://everyinfra.com/ja/build/n8n-everyinfra-daily-report)を検証しました。キーワードを変更して出力をGoogle Driveに書き込めば、設計Bの前半が完成します。設計Cについては、[Responses APIのMCPガイド](https://developers.openai.com/api/docs/guides/tools-connectors-mcp)で、`require_approval`で承認をスキップする方法と、`allowed_tools`で必要なツールだけを取り込む方法が説明されています。[Claude Codeのクラウドスケジュールタスク](https://code.claude.com/docs/en/routines)はPro、Max、Team、Enterpriseで利用でき、ノートPCを閉じていても実行されます。この記事ではどちらの方法も検証していません。選択肢として挙げているだけです。

## レーダーが取得すべきデータ：実際に行った3セットの呼び出し

レーダーが役に立つのは、データが今週のものであり、かつ対象カテゴリのものである場合だけです。2026年10月26日17:31〜17:41 SGTに、1つのテスト用APIキーで3つのセットを実行しました。セット1は、多くの英語圏の読者になじみがあるSaaSカテゴリのパスワードマネージャーです（キーワード「password manager」、コミュニティr/PasswordManagers）。セット2はハードウェアカテゴリのロボット掃除機です（キーワード「robot vacuum」「Roborock」「Dreame」「robot vacuum mop」、コミュニティr/RobotVacuums）。

セット3ではスペイン語のキーワードを使いました。合計32件のリクエストを送信し、そのうち22件が課金され、金額は$0.016397でした。残りの10件は、予定していたエラーテスト、レート制限を受けたリクエスト、結果が空だったリクエストで、いずれも料金はかかりませんでした。

1つ目の論点は対象期間です。カタログによると、`reddit/search`の対象期間は直近1か月に固定されており、`max_limit`は25です。`sort=newest`を指定すると、25件の投稿がカバーするのはわずか数時間です。「password manager」は7.1時間、「robot vacuum」は15.6時間、「Roborock」は24.4時間で、「Dreame」はわずか0.2時間でした（「dream」を含む新着投稿をすべてカウントしていたためです）。

月曜日の実行で「最新25件」だけを取得すると、見えるのは日曜日の午後から月曜日の朝までで、火曜日から土曜日までは見逃します。コミュニティ単位の取得は違います。`reddit/subreddit_posts`に`sort=top`、`date_range=past_week`、`limit=100`を指定すると、r/PasswordManagersの週間トップ投稿20件がすべて返されました。期間は10月19日から10月26日まで、160.1時間に及びます。r/RobotVacuumsでは、166.3時間にわたる100件が返されました。どちらもほぼ丸1週間です。

![呼び出しごとに、取得した投稿の最新から最古までの時間幅を示した横棒グラフ。sort newest、25件のreddit/search：robot vacuum 15.6時間、Roborock 24.4時間、Dreame 0.2時間、robot vacuum mop 6.5時間、password manager 7.1時間で、いずれも短いグレーの棒。subreddit_posts（週間トップ）：r/RobotVacuums 166.3時間、r/PasswordManagers 160.1時間で、1週間を示す168時間の破線に近い2本の長いティール色の棒。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/fig-04-newest-span.ja.light.svg)

そのため、この作業の中心は`reddit/subreddit_posts`です。キーワード検索は、まだ知らないコミュニティを見つけるためだけに使います。この点について事前に決めた基準では、25件の投稿のうち直近7日間に入る件数を数えました。結果は25件すべてでした。つまり当初の基準に従えば、週1回の実行で十分という結論でした。

しかし、この基準は測る対象を誤っていました。25件すべてが直近7日間に入っていたのは、すべてが直近数時間の投稿だったからです。テストログには、この基準を「設計の誤り」と記録しました。結論は時間幅から導き、当初の基準自体は変更していません。

キーワード検索にはノイズも混じります。「password manager」の結果25件は、25の異なるコミュニティからのものでした。両方の単語を含むのは12件だけです。それ以外には、カジノの宣伝、アプリの宣伝、無関係なアプリのログインに関する相談投稿などがありました。コミュニティ単位の取得ではこの問題は起きません。すべての投稿が、選んだコミュニティのものだからです。

2つ目の論点は、その週の投稿の中身です。分析スクリプトの冒頭に固定ルールを記述し、事前フィルタリングしています。タイトルが疑問符で終わる投稿は「質問」とみなします。タイトルがhow、what、which、best、anyoneで始まる場合も質問とします。

タイトルまたは本文の最初の400文字にstuck、broken、issue、avoid、returnなどの単語が含まれる投稿は「不満ワードあり」とします。r/PasswordManagersでは20件中8件が質問、1件が不満ワードありでした。r/RobotVacuumsでは100件中49件が質問、35件が不満ワードありでした。

![ルールベースの事前フィルタによる、4つの投稿セットにおける質問と不満ワードの件数を示したグループ化横棒グラフ。r/RobotVacuums週間トップ100件：質問100件中49件、不満ワード100件中35件。r/PasswordManagers週間トップ20件：質問20件中8件、不満ワード20件中1件。robot vacuumのキーワード検索25件：25件中11件と25件中6件。aspiradora robotのキーワード検索8件：8件中4件と8件中1件。グレーは質問、ティールは不満ワード。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/fig-09-question-complaint.ja.light.svg)

このルールは、精読する投稿を選ぶためのものにすぎず、調査結果ではありません。r/PasswordManagersで最もコメントが多かった投稿（76件）のタイトルは「Password manager scene is such a mess right now」で、不満ワードは含まれていません。単語リストでは不満として検出されませんが、その週で最も明確な不満です。「paranoid」なユーザーにおすすめを尋ねる投稿も2件ありました（コメントはそれぞれ35件と30件）。

あるユーザーは、3つのパスワードマネージャーが5つのサイトでのパスワード変更を検知できるかを検証していました（コメント37件、35ポイントで、その週の最高スコア）。新製品のセキュリティ監査はどこにあるのかを尋ねる投稿もありました（コメント20件）。本格的な分類はChatGPTのタスクに任せるべきです。モデルはスラングを読み取るのが得意で、ユーザーが何に不満を持っているかを判断できるからです。ルールの役割は1つだけで、質問や不満が先週と比べて今週増えたかどうかを示すことです。

3つ目の論点はコメントです。コメント数が最も多い2件の投稿に対して、`limit=100`を指定して`reddit/comments`を呼び出しました。返されたコメントは73件と49件でした。[Redditのコメントが不完全に見える理由](https://everyinfra.com/ja/blog/reddit-comments-morecomments-completeness)の記事で説明しているとおり、このAPIはトップレベルのコメントを返し、返信は展開しません。「mess」スレッドでは、13件のコメントに不満ワードが含まれていました。

最も票を集めた返信は、完璧な製品を探すのはやめるか、セルフホストするよう投稿者に勧めていました。「what do you actually use」スレッドでは、あるユーザーが、アプリとカスタマーサービスの両方で最近問題があったため、以前使っていたパスワードマネージャーから乗り換えたと書いていました。こうした詳細はアプリストアのレビューにはほとんど現れません。しかし、プロダクトチームに必要なのはまさにこの情報です。なぜユーザーが乗り換えるのか、何がベンダーへの不信を招くのかがわかります。

4つ目の論点はQuoraの件数です。`quora/questions`に`date_range=past_week_last_7_days`と`limit=30`を指定すると、「password manager」に関する新しい質問は1週間で4件、30日間で11件見つかりました。週間の4件のうち2件は、AIエージェント向けのパスワードマネージャーについての質問でした。同じ週、r/PasswordManagersには大手ベンダーの「AI agent integration」に関する投稿がありました。同じ週に2つのソースで同じ新しいトピックが現れることこそ、レーダーが捉えるべきシグナルです。

ロボット掃除機の場合、Quoraの質問は週に2件、月に30件（上限に達しているため、実際の数はさらに多い）でした。「Roborock」は週0件でした。この呼び出しには費用がかからず、レスポンスには`"reason": "empty_result_refunded"`と表示されました。Quoraからは「購入前に人々がどう考えているか」がわかります。件数は少ないため、月次の取得で十分です。

![上限30件で、quora/questionsが過去7日間と過去30日間に返した結果の件数を示したグループ化横棒グラフ。robot vacuum：7日間で2件、30日間で上限の30件。password manager：4件と11件。Roborock：7日間で0件、30日間は未実行。aspiradora robot：7日間は未実行、30日間で0件。ティールは7日間、グレーは30日間。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/fig-08-quora-window.ja.light.svg)

スペイン語のセットからは、言語の落とし穴が見えてきます。「aspiradora robot」で`reddit/search`（relevance）を実行すると8件が返され、7件が一致しました。一致した投稿は、r/AskChileやr/Colombiaといったラテンアメリカのコミュニティからのものです。一方、「robot aspirador」（newest）では11件が返されましたが、ほとんどがポルトガル語のコミュニティからでした。ポルトガル語ではロボット掃除機をaspirador robotと呼ぶためです。Quoraでは「aspiradora robot」は1か月で0件でした（返金済み）。

「robot aspirador」で過去1年間を検索すると、「aspiring」なロボティクス企業に関する英語の質問が5件返されました。結果が空ではなかったため、$0.001389が課金されました。複数言語を対象とするレーダーでは、各言語のキーワードを個別にテストしてください。英語のキーワードを翻訳するだけでうまくいくとは考えないでください。

| セット | 機能とパラメータ | 件数 | 一致 | 期間 | 課金額 |
| --- | --- | --- | --- | --- | --- |
| パスワードマネージャー | `reddit/subreddit_posts` r/PasswordManagers、top、past_week、100件 | 20 | すべて当該コミュニティから | 160.1 h | $0.000556 |
| パスワードマネージャー | `reddit/search`「password manager」、newest、25件 | 25 | 12 | 7.1 h | $0.000556 |
| パスワードマネージャー | `quora/questions`「password manager」、past_week、30件 | 4 | 4 | — | $0.001389 |
| パスワードマネージャー | `reddit/comments`、「mess」スレッド、上限100件 | 73 | — | — | $0.000556 |
| ロボット掃除機 | `reddit/subreddit_posts` r/RobotVacuums、top、past_week、100件 | 100 | すべて当該コミュニティから | 166.3 h | $0.000556 |
| ロボット掃除機 | `reddit/search`「Dreame」、newest、25件 | 25 | 3 | 0.2 h | $0.000556 |
| ロボット掃除機 | `quora/questions`「Roborock」、past_week、30件 | 0 | — | — | $0（返金済み） |
| スペイン語 | `quora/questions`「robot aspirador」、past_year、30件 | 5 | 0 | — | $0.001389 |

## タスクプロンプトの書き方

これらの数値があれば、タスクプロンプトに厳格な条件を設定できます。以下はChatGPTのスケジュールタスクの設定例です。ChatGPT上では実行していません。重要な点は三つあります。

各呼び出しでプラットフォーム、アクション、パラメータを明示するため、モデルが選ぶ余地はありません。レポートの末尾には、実行した呼び出しとそのコストを記載します。呼び出しが失敗またはブロックされた場合、モデルは別のパラメータで再試行しません。予定していた呼び出しを記載し、手動で実行できるようにします。

```text
Run every Monday at 09:00 (America/New_York). Use only everyinfra_call_api from the @EveryInfra plugin. Make the calls below in order. Do not change parameters. Do not add calls.
1. platform=reddit, action=subreddit_posts, params={subreddit: "PasswordManagers", sort: "top", date_range: "past_week", limit: 100}
2. platform=reddit, action=subreddit_posts, params={subreddit: "Bitwarden", sort: "top", date_range: "past_week", limit: 50}
3. platform=reddit, action=search, params={keyword: "password manager", sort: "relevance", limit: 25}
4. platform=quora, action=questions, params={keyword: "password manager", date_range: "past_week_last_7_days", limit: 30}
5. From step 1, take the 5 posts with the highest comment_count. For each, call platform=reddit, action=comments, params={url: <post url>, limit: 100}
Output:
- "Complaints this week": top 5. One-line summary, post link, comment count. Describe the issue; do not name individual users.
- "Questions this week": top 5, same format.
- Compare with last week's report. Deduplicate by post url. Mark new topics.
- Appendix: for each call, rows returned, rows that match the keyword, billing.charged_usd, and the total.
If a call fails or is blocked: do not retry with other parameters. List its platform, action, and params in the appendix, then continue.
```

ステップ5は、レーダーの中で最も時間がかかる部分です。17:33から17:41（SGT）に実行した6回の`reddit/comments`呼び出しは、中央値が22.2秒、最も遅いもので43.4秒でした。ChatGPTは、1回のツール呼び出しのタイムアウトを公開していません。参考になるのは[CodexのMCP設定](https://learn.chatgpt.com/docs/extend/mcp)で、`tool_timeout_sec`のデフォルトは60秒です。当社のカタログでは、同期呼び出しに85秒の上限を設けています。それを超える場合は、リクエストボディに`"mode": "async"`を追加し、`everyinfra_get_job`で結果を取得してください。

![機能ごとに、1回の呼び出しにかかる秒数を示した範囲ドットチャート。点は中央値、線は最速から最遅までの範囲。quora/questions：7回、中央値9.7秒。reddit/comments：6回、中央値22.2秒（ティール）、最遅43.4秒。reddit/search：8回、中央値6.4秒。reddit/subreddit_posts：2回、中央値30.2秒。85秒の破線は、カタログにおける同期呼び出しの上限を示します。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/fig-07-latency.ja.light.svg)

## 今週の実データによるサンプルレポート

良いレポートの形を示すため、サンプルを手作業で作成しました。17:31から17:41（SGT）の実行で取得した、r/PasswordManagersの週間トップ投稿20件、2つのコメントスレッド、Quoraから取得したデータを使っています。ChatGPTが書いたものではありません。週次レポートに含めるべき内容と、各行から実際の投稿をたどれる必要があることを示しています。

| セクション | 項目（コメント数） | 根拠 |
| --- | --- | --- |
| 不満1 | 「パスワードマネージャー界隈はめちゃくちゃ」：あるユーザーが主要製品すべてに問題を見つけている（76） | 投稿タイトルと73件のトップレベルコメント（うち13件が不満を示す語を含む） |
| 不満2 | 新製品に公開されたセキュリティ監査がない（20） | 投稿タイトル |
| 不満3 | あるユーザーが、アプリとサポートをめぐる最近の問題を理由に、それまで使っていたパスワードマネージャーから乗り換えた（コメント56件のスレッド内） | トップレベルコメント |
| 不満4 | フォームのすべてのフィールドを保存できる製品は少ないため、ユーザーは同じベンダーを使い続けている（7） | 投稿タイトル |
| 不満5 | 1つのアカウントに機能を集約しすぎて「Googleのようになりつつある」ベンダー（コメント56件のスレッド） | トップレベルコメント |
| 質問1 | 「実際に使っていておすすめのパスワードマネージャーは？」（56） | タイトルとトップレベルコメント49件 |
| 質問2 | 2つの新製品についての感想（41） | タイトル |
| 質問3 | ビジネス向けボールトへの率直なフィードバック（11） | タイトル |
| 質問4 | AIエージェント向けパスワードマネージャー：メリットと必須機能 | 1週間でQuoraの質問2件、さらにエージェント連携に関するReddit投稿1件 |
| 質問5 | 「パスワードマネージャーで本当にアカウントの安全性は高まるのか？」（回答5件） | Quora |

注目すべき点は2つあります。1つ目は、不満と質問が大きく重なっていることです。コメント数の多いスレッドは質問の形をとっていますが、コメント欄は不満であふれています。そのため、レポートはタイトルだけで分類できません。5回のコメント取得にかかった$0.00278は、このレーダーで最も有効に使われた費用です。

2つ目は、レポートの価値が週ごとの変化にあることです。1週間分だけでは、信頼性、監査、乗り換えといった従来からのテーマしか見えません。数週間続けると、「AIエージェント向けパスワードマネージャー」のように急に伸びるテーマが現れます。これこそがプロダクトチームやマーケティングチームの求めるシグナルです。そのため、後のセクションで「後で再実行：何が変わったか？」を検証しました。

## 1週間あたりのコスト

価格には、17:22 SGT時点で[機能カタログ](https://api.everyinfra.com/api/v1/social/catalog)から取得した`price_usd`を使用しています。週次のレシピは10回の呼び出しで構成され、内訳は`reddit/subreddit_posts`が1回、`reddit/search`が2回、`quora/questions`が2回、`reddit/comments`が5回です。

| 機能 | 単価 | 週あたりの呼び出し回数 | 小計 |
| --- | --- | --- | --- |
| `reddit/subreddit_posts` | $0.000556 | 1 | $0.000556 |
| `reddit/search` | $0.000556 | 2 | $0.001112 |
| `quora/questions` | $0.001389 | 2 | $0.002778 |
| `reddit/comments` | $0.000556 | 5 | $0.002780 |
| 合計 |  | 10 | $0.007226 |

52週間分の費用は$0.375752です。3つのカテゴリーでレーダーを1つずつ運用しても、年間約$1.13です。空の結果は返金されるため、実際の金額がこれを上回ることはありません。すぐに使えるペインポイントの要約が必要な場合は、カタログに`reddit/painpoint`もあり、料金は1回あたり$0.072です。キーワードで投稿を探してコメントを読み、ネガティブなコメントの数と頻出語を返します。1回の呼び出しに、このレシピの10週間分と同じ費用がかかります。毎週ではなく、新しいカテゴリーの下調べに1回だけ使ってください。

![横向きの積み上げ棒グラフ。上の棒は、10回の呼び出しからなる週次レシピを4つのセグメントで示しています。subreddit_postsは1回で$0.000556、searchは2回で$0.001112、questionsは2回で$0.002778、commentsは5回で$0.00278です。合計$0.007226はティール色の文字で表示されています。下のグレーの棒はreddit/painpointの1回の呼び出し（$0.072）を示し、レシピの約10倍です。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/fig-06-weekly-cost.ja.light.svg)

ChatGPT自体の費用は別にかかり、週単位の金額に換算することはできません。[ヘルプセンター](https://help.openai.com/en/articles/20001275)によると、WorkとCodexは同じ利用枠の仕組みを共有しています。[ChatGPTの料金ページ](https://learn.chatgpt.com/docs/pricing)には、Plusで5時間あたりに送れるメッセージ数の範囲（GPT-6.1 Solで15〜160件）が示されています。同じようなタスクでも使用量が大きく異なる場合があるとの注意書きもあります。

[2026年10月5日のr/codexの投稿](https://www.reddit.com/r/codex/comments/1wxxqwh/did_chatgpt_scheduled_tasks_recently_start/)で、あるPlusユーザーが毎日7:00に実行するタスクについて書いています。このタスクは13分間実行され、5時間の利用枠の29%を消費しました。OpenAIサポートからは、スケジュールタスクはWorkで実行され、利用枠をCodexと共有すると説明されたそうです。この規模になると、需要レーダーのChatGPT側の負担は、APIの利用料$0.007226よりはるかに大きくなります。

## レート制限、空の結果、失敗：料金はどうなるか

今回のテストでは、4種類の「データなし」に遭遇しました。4つとも費用は$0ですが、スケジュールタスクへの影響はそれぞれ異なります。

![当社APIから返された実際のレスポンス2件のスクリーンショット（ダークテーマのコードカード）。上段：POST /api/v1/social、reddit/comments、2026年10月26日17:37:57 SGT。ステータスは429で、retry-afterヘッダーの値は3です。エラーコードはrate_limitedです。メッセージには、このAPIキー固有の上限である毎分10リクエストを超えたため、3秒後に再試行するよう記載されています。reasonはkey_limitで、retry_after 3はティール色の枠で囲まれています。limitは10、retryableはtrue、max_retriesは3で、next_stepsには課金されていないことが記載されています。下段：過去1週間のRoborockに関するquora/questions、17:32:51 SGT。ステータスは200で、countは0です。billingではchargedがfalseで、charged_usd 0.0とreason empty_result_refundedがティール色の枠で囲まれています。返金ステータスはsucceededで、100クレジットが返還されました。](https://everyinfra.com/articles/chatgpt-scheduled-tasks-demand-radar/img-api-429-empty.webp)

*課金されない2つのケース：レート制限（429）と空の結果（自動返金）。2026年10月26日の実際の呼び出し* [EveryInfra APIのレスポンス](https://api.everyinfra.com/openapi.json)

| ケース | 実際のレスポンス | 課金額 | スケジュールタスクへの影響 |
| --- | --- | --- | --- |
| 認証情報なし | RESTは401 `missing_key`、MCPは認証情報の渡し方を説明する読みやすいエラーを返す（10月26日に確認。11月4日以降は、URLの`?key=`も401） | $0 | 認証の期限切れや、コネクタが古い`?key=`付きURLのままの場合でも、モデルは黙って失敗せず、明確なエラーを読み取れます。コネクタを削除してログインし直してください |
| 誤ったパラメータ | `sort: "latest"`は`allowed`リスト付きで422、`sort: "newst"`は`did_you_mean: ["newest"]`付きで422を返します | $0 | モデルは修正できますが、厳密なプロンプトならここに至ることはありません |
| レート制限 | SGT 17:37:57以降に429レスポンスが4回、`retry_after`は3秒、`limit`は1分あたり10 | $0 | 同じ時間帯に、他のセッションが同じテストキーを使用していました。モデルに数秒待機させてから再試行させる |
| 空の結果 | `quora/questions`でRoborockを検索、1週間で0件、`empty_result_refunded` | $0 | 「今週は新しい質問なし」と記載する。失敗ではない |

1点だけ補足があります。今回4回レート制限に達したのは、ほかのセッションが同じテスト用キーを同時に使っていたためです。1分あたり10回という制限はこのキー個別の設定で、本番用キーの制限ではありません。[OpenAPIドキュメント](https://api.everyinfra.com/openapi.json)では、アカウントのレート制限は1分あたり60リクエストとされています。

1回のレーダーで行う呼び出しは10回なので、通常は制限に達しません。ただし、複数のカテゴリのタスクが1つのキーを共有し、すべて月曜09:00に実行されると、制限に達する可能性があります。実行時刻を数分ずつずらしてください。

## ChatGPTのスケジュールタスクが動かない：よくある原因と対処法

検索候補では、「chatgpt scheduled tasks」の直後に最初に出てくる語が「not working」です。OpenAIのドキュメントに記載された制限とフォーラムの報告を突き合わせたところ、この用途で問題になる原因は9つありました。最初の5つは公式の記載があります。残りの4つはユーザーからの報告であり、あくまで報告として扱います。

| 症状 | 原因 | 出典 | 対処法 |
| --- | --- | --- | --- |
| タスクが実行されず、「一時停止」と表示される | 承認が必要なアクションがあるとタスクは一時停止します。誰も開かないタスクは一時停止することがあります。チャットを削除またはアーカイブすると、そのタスクも一時停止します | [ヘルプセンター：スケジュールタスク](https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt) | 「スケジュール」で保留中の承認を確認し、数週間に一度はレポートを開く |
| 新しいタスクを作成できない | アクティブなタスクの上限（3/5/10/15）に達している | 同上 | 使っていないタスクを一時停止または削除する |
| Freeプランのタスクが指定時刻に実行されない | Freeプランでは、朝や夜などの時間帯に1日最大1回実行される | 同上 | 正確な時刻で実行するには有料プランを使う |
| タスクがプロジェクトのファイルを読めない | プロジェクト内で作成したタスクは、そのプロジェクトにアップロードされたファイルを開けない | 同上 | 参照テキストをタスクのプロンプトに含めるか、コネクタが読める場所に置く |
| GPT、音声チャット、ロックされたチャットではタスクを作成できない | スケジュールタスクはこれらに対応していない | 同上 | 通常のチャットまたはWorkで作成する |
| 書き込みがブロックされる（ユーザー報告） | 通常のチャットでは同じプロンプトが動くのに「blocked by OpenAI's safety checks」と表示される。ドキュメントには書き込みに承認が必要とあるだけ | [2026年10月4日以降のフォーラム報告](https://community.openai.com/t/1403242) | データ取得を外部のスケジューラーに移し、ChatGPTは読み取りのみ行う（設計B） |
| 実行後にタスクが自動で無効になる | 書き込みに失敗してから約1分後に、タスクが無効に切り替わる | [2026年10月6日〜8日のフォーラム報告](https://community.openai.com/t/chatgpt-scheduled-tasks-fail-to-write-to-github-and-automatically-disable-themselves-after-failed-executions-without-user-authorization/1404882) | 同上。毎週タスクのステータスを確認する |
| カスタムMCPサーバーを追加すると「does not implement OAuth」と表示される | ファイアウォールが確認リクエストをブロックしたか、サーバーがOAuthに対応していない | [2026年9月3日のフォーラム報告](https://community.openai.com/t/feature-request-use-an-identifiable-user-agent-for-custom-mcp-server-validation/1394731) | EveryInfraの場合はOAuthを選び、URLに`/mcp/all`を入力する |
| タスクがWorkの利用枠を大量に消費する | スケジュールタスクはWorkで実行され、利用枠をCodexと共有する | [r/codex、2026年10月5日](https://www.reddit.com/r/codex/comments/1wxxqwh/did_chatgpt_scheduled_tasks_recently_start/)、[料金ページ](https://learn.chatgpt.com/docs/pricing) | タスクのプロンプトは短く厳密にし、要約のみを求める |

[r/ChatGPTの返信が30件を超えるヘルプスレッド](https://www.reddit.com/r/ChatGPT/comments/1x049yo/scheduled_tasks_failing/)では、これらの問題がいくつも混在しています。あるユーザーの毎時実行タスクは、GitHubへの書き込みが止まったあと自動で無効になりました。別のユーザーの月曜タスクは、3週間にわたってタスク一覧に表示されませんでした。3人目のユーザーがタスクが実行されなかった理由を尋ねると、モデルは処理を実行したと答えましたが、ユーザーにはその結果が見えませんでした。注意すべきなのはこの最後のケースです。

モデルが自分の行動について説明した内容は、証拠にはなりません。そのため、私たちのプロンプトではレポートの最後に呼び出し一覧と`charged_usd`を出力するよう求めています。呼び出しが行われたか、何回行われたか、いくらかかったかは、当社の呼び出し記録で確認すべきです。コンソール上の各呼び出しには`request_id`が付いています。[別のスレッド](https://www.reddit.com/r/ChatGPT/comments/1wvnhp8/isnt_this_kind_of_a_paradox_for_scheduled_tasks/)では、自動一時停止をパラドックスと呼んでいます。

うまく動いているタスクには手をかける必要がありません。しかし、誰も見ないタスクは未使用とみなされて一時停止することがあります。月曜のレーダーは、実際に目を通す場所に送ってください。これも対処法の一部です。

## 1時間以内に再実行：週をまたいで重複を排除する方法

週次レーダーでは、毎週月曜の結果を前週の月曜と比較します。2回の実行の間で何が変わるのか、どのフィールドで重複を除去すべきかを確かめるため、最初のセットから3つの呼び出しを同じパラメータで再実行しました。テストの前に「少なくとも45分空ける」という基準を記録しておきました。1回目の再実行はSGTで18:13:31から18:14:28に行われました。これは初回から38.7〜42.4分後で、基準を下回っています。

テストログにもそのとおり記録しました。そこで、SGTで18:21:02から18:21:52に2回目の再実行を行いました。初回から46.1〜50.0分後で、基準を満たしています。追加の6回の呼び出しにかかった費用は$0.003336です。これを含めると、テスト全体は38リクエスト、うち課金対象28件で、合計$0.019733でした。

| 呼び出し | 初回実行（SGT） | 件数：初回→再実行1→再実行2 | 再実行2で新規 | 再実行2で消失 |
| --- | --- | --- | --- | --- |
| `reddit/search`「robot vacuum」、newest、25件 | 17:31:05 | 25 → 25 → 25 | 1（セール情報の投稿） | 1（最も古いもの） |
| `reddit/subreddit_posts` r/RobotVacuums、top、past_week、100件 | 17:32:11 | 100 → 24 → 100 | 1（3.8 cmの敷居に関する新しい質問） | 1（その週のトップ投稿） |
| `reddit/subreddit_posts` r/PasswordManagers、top、past_week、100件 | 17:35:46 | 20 → 19 → 19 | 0 | 1 |

この表から3つの教訓が得られます。1つ目に、`past_week`は呼び出した時点で終わる168時間の移動ウィンドウです。暦上の週ではありません。r/RobotVacuumsのトップ投稿「Lol avoid Eureka」（32ポイント）は、2026年10月19日10:09 UTCに投稿されました。SGTの18:13、つまり10月26日10:13 UTCの時点で投稿から7日以上が経過し、ウィンドウから外れました。

r/PasswordManagersでも、10月19日10:06 UTCの投稿に同じことが起きました。タスクを毎回同じ時刻に実行すれば、各投稿はちょうど1つの週にだけ入ります。月曜の実行が遅れて始まると（たとえばタスクを一時停止してから再開した場合）、境界にある投稿を取りこぼします。早く始まると、一部の投稿が2回表示されます。

第2に、重複排除はタイトルではなく`id`または`url`で行ってください。実行と実行の間に残っている投稿は、ポイントやコメント数が変わっても同じ`id`と`url`を保ちます。先週の`id`の一覧を、レポートまたは設計Bのファイルに保存しておきます。`id`がその一覧にない投稿を「新規」とします。

私たちのスペイン語のデータセットでは、1人のユーザーが同じ質問をチリの2つのコミュニティに2分半の間隔で投稿していました。これは`id`が2つある、2件の投稿です。このようなケースに備え、正規化したタイトルは2次フィルターとしてのみ使ってください。

第3に、最も意外だったのは、返る件数が安定しないことです。同じパラメータで、r/RobotVacuumsは50分足らずの間に100件、24件、再び100件を返しました。1回目の再実行で返った24件は、ちょうど5ポイント以上の投稿でした。0ポイントの投稿76件が欠けており、それらのコメントは合計16件でした。24件のレスポンスはステータス200で返り、通常どおり課金されました。

レーダーにとってはあまり問題になりません。レポートに載る投稿はすべて含まれていたからです。ただし、週のボリュームを測るために投稿数を数える場合は問題になります。そのため、レポートの付録には呼び出しごとの行数を記載しています。コミュニティの取得結果が前週の半分未満の行数になることがあります。その場合は、コミュニティが静かになったと判断する前に、その呼び出しをもう一度実行してください。

もう1つ、票によるランキング全般に当てはまる影響があります。日曜の夜の投稿は、数時間分の票しか集まらないまま月曜の実行を迎えます。翌週にはウィンドウの外です。そのため、週の最後の数日の投稿は順位が低くなりがちです。これが気になる場合は、「過去24時間でコメントが最も多い投稿」の短いリストをレポートに追加してください。追加の呼び出しは不要で、同じコミュニティの取得結果を`posted_at`で絞り込むだけです。

## フィールドリファレンス：レポートで使用するすべてのフィールド

これらの機能のフィールドは、[カタログ](https://api.everyinfra.com/api/v1/social/catalog)の`response_fields`と1つずつ照合しました。この表には、レーダーで使用するフィールドのみを掲載しています。プラットフォームと機能のページ：[RedditデータAPI](https://everyinfra.com/ja/reddit-api)、[QuoraデータAPI](https://everyinfra.com/ja/quora-api)、[reddit/search](https://everyinfra.com/ja/reddit-api/search)、[reddit/subreddit_posts](https://everyinfra.com/ja/reddit-api/subreddit_posts)、[reddit/comments](https://everyinfra.com/ja/reddit-api/comments)、[quora/questions](https://everyinfra.com/ja/quora-api/questions)。

| 機能 | フィールド | 意味 | レーダーでの用途 |
| --- | --- | --- | --- |
| `reddit/search`, `reddit/subreddit_posts` | `id`、`url` | 投稿IDとリンク | 重複排除と前週との比較 |
| 同上 | `title`、`text` | タイトルと本文 | 分類用の入力 |
| 同上 | `community` | r/プレフィックスを除いたサブレディット名 | 話題が集中している場所を把握 |
| 同上 | `like_count`、`comment_count` | スコアとコメント数 | 投稿の順位付け、コメントを読む投稿の選定 |
| 同上 | `posted_at` | 投稿日時 | 今週の投稿かどうかを確認 |
| 同上 | `flair` | 投稿フレアのテキスト | 「おすすめ」や「ヘルプ」に使うコミュニティもある |
| `reddit/comments` | `text`、`like_count`、`posted_at` | コメント本文、スコア、日時 | 不満の詳細を読む |
| `quora/questions` | `title`、`url`、`posted_at` | 質問タイトル、リンク、日時 | 「今週の質問」 |
| 同上 | `answer_count`、`follower_count` | 回答数とフォロワー数 | 回答がつくかを確認 |

どちらのReddit機能も`author_username`を返しますが、レーダーでは不要です。[RedditのResponsible Builder Policy](https://support.reddithelp.com/hc/en-us/articles/42728983564564-Responsible-Builder-Policy)は、データから健康や政治などのセンシティブな属性を推測することを禁止しています。ユーザーを再特定しようとする行為も禁止しています。プロンプトにある「個々のユーザーの名前を出さない」という行は、そのためにあります。

## ChatGPTを使わずに同じデータを取得

設計Bのスケジューラーを使う場合や、まず手作業でデータを確認したい場合は、REST APIを呼び出します。パラメータはMCPと同じで、JSON-RPCのラッパーがないだけです。

```bash
curl -s https://api.everyinfra.com/api/v1/social \
  -H "Authorization: Bearer $EVERYINFRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"platform":"reddit","action":"subreddit_posts",
       "params":{"subreddit":"PasswordManagers","sort":"top","date_range":"past_week","limit":100}}'
```

SGTの17:35:46に、このリクエストはその週の投稿20件を30.2秒で返し、費用は$0.000556でした。カタログには機能ごとに`max_limit`が記載されています。`reddit/search`は25です（それより大きい値は25に切り詰められます）。`reddit/subreddit_posts`と`reddit/comments`は100です。

`quora/questions`は200です。課金は行単位ではなく呼び出し単位のため、上限内で`limit`を最大にすると、同じ費用で最も多くのデータを得られます。一連の初期手順については、[クイックスタート](https://everyinfra.com/ja/docs#quickstart)をご覧ください。MCP全般については、[リモートMCPの受け入れテスト](https://everyinfra.com/ja/build/remote-mcp-setup)と[MCPサーバーのページ](https://everyinfra.com/ja/mcp-server)をご覧ください。

## このアプローチが通用しなくなる場面

第1に、ChatGPTの動作は一切テストしていません。書けるのは、2026年10月26日時点でドキュメントに書かれていた内容だけです。OpenAIはプランや権限のルールを頻繁に変更するため（引用したヘルプセンターのページのほとんどに「数時間前に更新」と表示されていました）、利用する前に改めて確認してください。第2に、Proでデータツールを呼び出せるかについて、OpenAIの2つのページで記載が異なります。ヘルプセンターのFAQでは、ProのカスタムMCPはread/fetch権限のみとされています。開発者ガイドでは、読み取りと書き込みのツールに対応し、書き込みには承認が必要とされています。当社は開発者ガイドに従っており、Proはテストしていません。

3つ目に、質問と不満の判定ルールは事前フィルターです。20件や100件の投稿のサンプルからわかるのは「多い」ということであり、割合ではありません。割合を知るには、数週間にわたって実行し、毎週サンプルを手作業で確認してください。4つ目に、Redditは2026年11月13日にRSSを停止します。また、2027年3月までに公開APIを終了する予定です（[TechCrunch、2026年9月30日](https://techcrunch.com/2026/09/30/reddit-is-killing-rss-feeds-ending-public-api-access-because-of-ai-bots/)、原文は[r/modnewsの投稿](https://www.reddit.com/r/modnews/comments/1wubgvt/continuing_our_infrastructure_updates_whats/)）。RSSで構築したレーダーは変更が必要です。[Reddit RSS監視の置き換え方法](https://everyinfra.com/ja/blog/reddit-rss-shutdown)で解説しています。

5つ目に、レーダーからわかるのはユーザーが何について話しているかであり、市場の規模ではありません。RedditやQuoraのユーザーがすべて購入者というわけではありません。調査結果は営業やサポートのデータと照らし合わせてください。

## 最近の変更点

- **2026年8月25日：スケジュールタスクがアプリのイベントに反応できるようになり、共有も可能に**（[ChatGPTリリースノート](https://help.openai.com/en/articles/6825453-chatgpt-release-notes)）。PlusとProのユーザーは、GmailやSlackの新着メッセージ、GitHubのプルリクエストの変更をきっかけにWorkタスクを開始できます。Freeユーザーはアクティブなタスクを3件まで持てます。影響：時刻ベースのレーダーに変更はありませんが、完成したレーダーを、自分のアプリを接続したチームメンバーと共有できます。
- **2026年9月29日：対応プラグインがMCPイベントから自動化を開始可能に**（同じリリースノート。開発者向けページ：[MCP Events](https://developers.openai.com/plugins/build/mcp-events)）。MCP 2.0（プロトコルバージョン2026-07-28）が必要で、サーバーはWebhookでイベントをプッシュします。影響：EveryInfraはプル型のAPIで、イベントをプッシュしません。「新しい投稿があれば通知」を実現するには、引き続きスケジュールに沿ってプルする必要があります。
- **2026年10月1日：カスタムMCPに開発者モードが不要に**（[フォーラムでのOpenAIスタッフの回答](https://community.openai.com/t/1402157)）。サーバーはPluginsページから追加します。ワークスペースの権限は名称が変わりました。影響：「Developer mode」のスイッチを探すよう案内している古いガイドは、情報が古くなっています。
- **2026年10月4日以降：スケジュールタスクで書き込みがブロックされるというユーザー報告**（[公式フォーラム](https://community.openai.com/t/1403242)。OpenAIからの回答はまだありません）。読み取りには影響しません。影響：ドキュメントでは、書き込みアクションは承認を経て実行されるとされていますが、誰も見ていない状態でお金を使ったりデータを変更したりするツールは、ChatGPTの外に移すのが最も安定した選択です。
- **2026年9月30日：RedditがRSSの11月13日終了と、2027年1月12日までのアプリ登録の義務化を発表**（[r/modnews](https://www.reddit.com/r/modnews/comments/1wubgvt/continuing_our_infrastructure_updates_whats/)）。影響：RSSでコミュニティを追跡しているレーダーは、11月13日までに変更が必要です。
- **2026年10月26日：EveryInfraのキーにUSDの利用上限を設定可能に**（[OpenAPIドキュメント](https://api.everyinfra.com/openapi.json)の`spend_limit_usd`と`key_spend_limit_reached`）。影響：Codexなど、ヘッダーでキーを渡すクライアントで使うキーには、先に上限を設定しておけます。

## FAQ

**chatgpt tasksとchatgpt scheduled tasksは同じものですか？** はい、1つの機能の2つの名前です。旧インターフェースでは「Tasks」と表示されていました。現在のヘルプセンターのタイトルは「Scheduled tasks in ChatGPT」で、入口はサイドバーのScheduledページです。1回限り、繰り返し、モニタリングのタスクに対応しています。出典：検索候補で「chatgpt tasks」が「chatgpt scheduled tasks」と並んで表示（2026年10月26日確認）、[ヘルプセンター](https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt)。

**ChatGPTのスケジュールタスクが動かない（chatgpt scheduled tasks not working）：最初に何を確認すべきですか？** Scheduledページを開き、「paused」や承認待ちの表示がないか確認してください。ヘルプセンターによると、承認が必要なアクションがあるとタスクは一時停止し、誰も開かないタスクも一時停止することがあります。次に、そのタスクをプロジェクト、GPT、音声チャット、ロックされたチャットのいずれかで作成していないか確認します。タスクがデータを書き込む場合は、実行結果に「blocked by OpenAI's safety checks」と表示されていないか確認してください（フォーラムでユーザーが報告しているケースです）。出典：検索候補「chatgpt scheduled tasks not working」「chatgpt scheduled tasks not showing」、[ヘルプセンター](https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt)、[フォーラムの報告](https://community.openai.com/t/1403242)。

**ChatGPTのカスタムMCPコネクタ（chatgpt custom mcp connector）には今も開発者モードが必要ですか？** いいえ。2026年10月1日に変更されました。Pluginsを開いて「+」を選択し、メニューから「Add custom MCP server」を選択します。Businessワークスペースでは、作成できるのは管理者とオーナーのみです。EnterpriseとEduでは、管理者がロールごとに権限を付与します。出典：検索候補「chatgpt custom mcp connectors」、[OpenAIスタッフの回答](https://community.openai.com/t/1402157)、[ヘルプセンター](https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt)。

**ChatGPTのスケジュールタスクの上限（chatgpt scheduled tasks limit）は？** プランによって異なります。アクティブなタスク数は、FreeとGoが3件、Plusが5件、BusinessとEduが10件、ProとEnterpriseが15件です。有料プランでは1時間ごとの繰り返しが可能で、Freeは最大1日1回です。イベントトリガー型のタスクは、すべて合わせて1時間あたり最大30回、1日あたり最大720回実行できます。出典：検索候補「chatgpt scheduled tasks limit」、[ヘルプセンター](https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt)。

**読み取り専用と示されていないEveryInfraのデータツールを、ChatGPTから呼び出せますか？** はい。OpenAIの開発者ガイドによると、ChatGPTは読み取りツールと書き込みツールの両方に対応しています。`readOnlyHint`のないツールは書き込みアクションとして扱われ、書き込みアクションにはデフォルトで承認が必要です。各チャットで最初に呼び出すと確認が表示され、許可を選択すると、そのチャットではその選択を記憶させることができます。新しいチャットでは再度確認されます。スケジュールタスクでは承認する人がいないため、タスクは一時停止します。スケジュールを設定する前に、通常のチャットでタスクを一度手動で実行してください。出典：[Add custom MCP server](https://developers.openai.com/api/docs/guides/custom-mcp-server)、当社が2026年10月26日17:23および20:04（SGT）に取得した`tools/list`の結果。

**ChatGPTでEveryInfraを使う場合、どの認証方式を選べばよいですか？** OAuthを選び、接続URLに`https://api.everyinfra.com/mcp/all`を入力して、画面の案内に従いEveryInfraにログインしてください。キーの入力は不要です。以前の「No authentication」でキーをURLの`?key=`に含める方法は2026-11-04に停止し、401を返します。旧方式で追加したコネクタは削除してから追加し直してください。ログイン経由の接続は設定内容を確認しただけで、ChatGPT上ではまだ動作確認が取れていません。出典：本記事の2026年11月8日22:20（SGT）の検証、[認証に関する開発者ページ](https://developers.openai.com/plugins/build/auth)。

**週次の需要レーダーにかかる費用は？** 10回のAPI呼び出し料金は週あたり$0.007226で、年間では約$0.38です。結果が空の場合は返金されます。ChatGPTのプラン料金やWorkの利用量は、これとは別にかかります。出典：2026年10月26日時点の[機能カタログ](https://api.everyinfra.com/api/v1/social/catalog)の`price_usd`の値をもとにした当社の計算。実際の請求額もカタログと一致しました。

## 関連記事

- [EveryInfraを呼び出すn8nの日次レポート：2.45スケジューラーの検証](https://everyinfra.com/ja/build/n8n-everyinfra-daily-report)
- [リモートMCPの受け入れ確認：「ツールが見つかる」から「実際に使える」まで](https://everyinfra.com/ja/build/remote-mcp-setup)
- [RedditのRSSが11月13日に終了：モニタリングの変更方法](https://everyinfra.com/ja/blog/reddit-rss-shutdown)
- [Redditのコメントが不完全に見える理由](https://everyinfra.com/ja/blog/reddit-comments-morecomments-completeness)
- [複数プラットフォームにまたがる評判モニタリング](https://everyinfra.com/ja/build/multiplatform-reputation-monitoring)
- [Redditのペインポイント調査：reddit/painpoint](https://everyinfra.com/ja/reddit-api/painpoint)
- [料金](https://everyinfra.com/ja/pricing)

## 参考文献

1. 「Scheduled tasks in ChatGPT」、OpenAIヘルプセンター、2026年10月26日閲覧、<https://help.openai.com/en/articles/10291617-scheduled-tasks-in-chatgpt>
1. 「ChatGPT Work and Codex」、OpenAIヘルプセンター、2026年10月26日閲覧、<https://help.openai.com/en/articles/20001275>
1. 「ChatGPT release notes」（2026年8月25日、9月29日、10月20日の項目）、OpenAIヘルプセンター、<https://help.openai.com/en/articles/6825453-chatgpt-release-notes>
1. 「Connecting custom MCP servers in ChatGPT」、OpenAIヘルプセンター、2026年10月26日閲覧、<https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt>
1. 「Managing app permissions in ChatGPT」、OpenAIヘルプセンター、<https://help.openai.com/articles/20001495>
1. 「Connected apps in ChatGPT」、OpenAIヘルプセンター、<https://help.openai.com/en/articles/11487775-apps-in-chatgpt>
1. 「About ChatGPT Pro tiers」、OpenAIヘルプセンター、<https://help.openai.com/articles/9793128>
1. 「What is ChatGPT Plus?」、OpenAIヘルプセンター、<https://help.openai.com/en/articles/6950777-what-is-chatgpt-plus>
1. 「What is ChatGPT Go?」、OpenAIヘルプセンター、<https://help.openai.com/en/articles/11989085-what-is-chatgpt-go>
1. 「ChatGPT Business release notes」（2026年9月29日のTeam tasks）、OpenAIヘルプセンター、<https://help.openai.com/en/articles/11391654-chatgpt-business-release-notes>
1. 「ChatGPT Enterprise and Edu release notes」、OpenAIヘルプセンター、<https://help.openai.com/en/articles/10128477-chatgpt-enterprise-and-edu-release-notes>
1. 「リグレッション：スケジュールタスクの安全チェックが、許可済みのGmail送信とGoogle Sheetsへの書き込みをブロックする」、OpenAI Developer Community、2026年10月4日（ユーザー報告）、<https://community.openai.com/t/1403242>
1. 「ChatGPT Proの［セキュリティとログイン］に開発者モードが表示されない」（OpenAIスタッフの回答、2026年10月1日）、OpenAI Developer Community、<https://community.openai.com/t/1402157>
1. 「Feature Request: Use an Identifiable User-Agent for Custom MCP Server Validation」、OpenAI Developer Community、2026年9月3日（ユーザー報告）、<https://community.openai.com/t/feature-request-use-an-identifiable-user-agent-for-custom-mcp-server-validation/1394731>
1. 「ChatGPTのスケジュールタスクがGitHubへの書き込みに失敗し、自動的に無効化される」、OpenAI Developer Community、2026年10月8日（ユーザー報告）、<https://community.openai.com/t/chatgpt-scheduled-tasks-fail-to-write-to-github-and-automatically-disable-themselves-after-failed-executions-without-user-authorization/1404882>
1. 「プラグインを接続してテストする」、OpenAI Developers、<https://developers.openai.com/plugins/deploy/connect-chatgpt>
1. 「Authentication」、OpenAI Developers、<https://developers.openai.com/plugins/build/auth>
1. 「ツールを定義する」（安全性アノテーションの計画）、OpenAI Developers、<https://developers.openai.com/plugins/plan/tools>
1. 「MCPサーバーを構築する」、OpenAI Developers、<https://developers.openai.com/plugins/build/mcp-server>
1. 「Plugin UIの変更履歴」（アプリの権限コントロール、2026年6月12日）、OpenAI Developers、<https://developers.openai.com/plugins/changelog>
1. 「MCP Events」、OpenAI Developers、<https://developers.openai.com/plugins/build/mcp-events>
1. 「Scheduled tasks」（Codex）、OpenAI Developers、<https://developers.openai.com/codex/app/automations>
1. 「MCP and Connectors」（Responses API）、OpenAI Developers、<https://developers.openai.com/api/docs/guides/tools-connectors-mcp>
1. 「Model Context Protocol」、ChatGPT Learn、<https://learn.chatgpt.com/docs/extend/mcp>
1. 「Pricing」（ChatGPT Work and Codex）、ChatGPT Learn、<https://learn.chatgpt.com/docs/pricing>
1. 「Schema Reference: ToolAnnotations」、Model Context Protocol 2025-11-25、<https://modelcontextprotocol.io/specification/2025-11-25/schema#toolannotations>
1. 「Authorization」、Model Context Protocol 2025-11-25、<https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization>
1. 「RFC 6750: The OAuth 2.0 Authorization Framework: Bearer Token Usage」、IETF、2012年10月、<https://www.rfc-editor.org/rfc/rfc6750>
1. 「RFC 9728: OAuth 2.0 Protected Resource Metadata」、IETF、<https://datatracker.ietf.org/doc/html/rfc9728>
1. 「MCP Inspector」、Model Context Protocol、<https://modelcontextprotocol.io/docs/tools/inspector>
1. 「インフラ更新の続報：今後数か月の変更点」、r/modnews、2026年9月30日、<https://www.reddit.com/r/modnews/comments/1wubgvt/continuing_our_infrastructure_updates_whats/>
1. 「RedditがAIボットを理由にRSSフィードを廃止し、公開APIへのアクセスを終了」、TechCrunch、2026年9月30日（二次情報。原典は文献31）、<https://techcrunch.com/2026/09/30/reddit-is-killing-rss-feeds-ending-public-api-access-because-of-ai-bots/>
1. 「Responsible Builder Policy」、Reddit Help、<https://support.reddithelp.com/hc/en-us/articles/42728983564564-Responsible-Builder-Policy>
1. 「スケジュールタスクが失敗する」、r/ChatGPT（ユーザーのスレッド）、<https://www.reddit.com/r/ChatGPT/comments/1x049yo/scheduled_tasks_failing/>
1. 「これはスケジュールタスクにとってある種のパラドックスでは？」、r/ChatGPT（ユーザーのスレッド）、<https://www.reddit.com/r/ChatGPT/comments/1wvnhp8/isnt_this_kind_of_a_paradox_for_scheduled_tasks/>
1. 「ChatGPTのスケジュールタスクが最近、Codex/Workの利用枠を消費するようになりましたか？」、r/codex、2026年10月5日（ユーザー報告）、<https://www.reddit.com/r/codex/comments/1wxxqwh/did_chatgpt_scheduled_tasks_recently_start/>
1. 「スケジュールタスクで作業を自動化する」、Claude Code Docs、<https://code.claude.com/docs/en/routines>
1. 「ChatGPT Scheduled Tasks: What They Are and How to Use Them」、Zapier、2026年9月2日（二次情報）、<https://zapier.com/blog/chatgpt-scheduled-tasks/>
1. 「ChatGPT MCP: Connect MCP Servers to ChatGPT」、AnythingMCP、2026年9月25日（二次情報）、<https://anythingmcp.com/guides/clients/chatgpt>
1. EveryInfra機能カタログ、2026年10月26日17:22（SGT）閲覧、<https://api.everyinfra.com/api/v1/social/catalog>
1. EveryInfra OpenAPIドキュメント、2026年10月26日17:42（SGT）閲覧、<https://api.everyinfra.com/openapi.json>
1. r/PasswordManagersのコミュニティページ、<https://www.reddit.com/r/PasswordManagers/>
1. 「カスタムMCPサーバーを追加する」（セクション「ツール呼び出しの確認と承認」）、OpenAI Developers、2026年10月26日20:05（SGT）閲覧、<https://developers.openai.com/api/docs/guides/custom-mcp-server>

---

:::message
この記事は EveryInfra ブログの記事の転載です。原文：[ChatGPTのスケジュールタスクとMCPで週次の需要レーダーを作る](https://everyinfra.com/ja/blog/chatgpt-scheduled-tasks-demand-radar)

EveryInfra は AI エージェントと開発者向けの従量課金データ API です：[everyinfra.com/ja](https://everyinfra.com/ja)
:::
