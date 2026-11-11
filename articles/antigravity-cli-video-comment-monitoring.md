---
title: "Antigravity CLIでコメントを監視する：セットアップと実データ"
emoji: "📊"
type: "tech"
topics: ["api", "ai", "webscraping", "datascience"]
published: true
---

:::message
この記事は [EveryInfra ブログ](https://everyinfra.com/ja/blog/antigravity-cli-video-comment-monitoring) からの転載です（原文が正本です）。
:::

動画の公開後24時間で、コメント欄の雰囲気が決まります。早めに人が対応すべきコメントは3種類あります。苦情、回答を求める質問、そして返信しないと冷淡な印象を与えかねないコメントです。このガイドでは、Gemini CLIの後継となるGoogleのターミナルエージェント、Antigravity CLIでコメント巡回を構築します。コマンドは`agy`です。数時間ごとにシステムのタイマーが`agy -p`を起動し、エージェントが新着コメントを取得して短いレポートを書きます。

コメントデータはEveryInfraのリモートMCPサーバーから取得します。YouTubeでは`youtube/comments`と`youtube/video`、Bilibiliでは`bilibili/comments`、`bilibili/danmaku`、`bilibili/video`を使います。データ取得部分は、2026年11月3日00:17〜01:22（シンガポール時間、SGT）に実際に実行しました。

コメントが活発な動画2本で巡回を3回行い、有料呼び出しは20回、API費用は合計$0.01137でした。テスト環境にはAntigravity CLIをインストールしていません。そのため、以下の`mcp_config.json`、スキル、タイマースクリプトはすべて公式ドキュメントに基づく設定例です。`agy`は私たち自身では実行していません。

このガイドには、どのドキュメントにも載っていない数字があります。あるYouTube動画には、公開から23.6時間で1,546件のコメントがありました。1回の呼び出しで取得できる上限である最新100件のコメントは、直近5時間分しかカバーしていませんでした。27分後の2回目の巡回では、その100件のうち新着は9件だけでした。Bilibiliでは、公開から3時間足らずで4,075件のコメントが付いた動画がありました。

コメントのエンドポイントは、3回の巡回で人気コメントをそれぞれ3件、10件、3件返しました。キーワードによる事前フィルターはYouTubeのコメント100件中34件にフラグを立てましたが、クリエイターが時間を割くべきものは13件だけでした。2時間ごとに1回巡回すると、丸1日のAPI費用は$0.023です。モデル利用料は、Gemini 3.8 Flash APIの公開料金に基づくと約$0.103で、その大半を固定プロンプトのオーバーヘッドが占めます。

> **要点**
>
> - テストしたのはデータ取得部分のみです。Antigravity CLIの設定、スキル、タイマースクリプトは11月3日時点のドキュメントに基づく例です。
> - 2026年6月18日、個人のGoogleアカウントではGemini CLIがAntigravity CLIに置き換わりました。現在、MCPサーバーの設定は`mcp_config.json`に記述します。
> - `/schedule`は会話の終了とともに止まるため、24時間の巡回ではlaunchd、cron、systemdで`agy -p`を起動します。
> - ヘッドレスモードでは、承認されていないツールは終了コード0のままソフト拒否されるため、stderrと`finished_reason`を確認してください。
> - 実測値：YouTubeの最新100件のコメントは5時間分で、Bilibiliの人気コメントは最大10件でした。
> - 2時間ごとに1回巡回すると、API費用は1日$0.023、モデル利用料はAPI価格で約$0.103です。

![リング状の線路のイラスト。駅には8台の小さなカートが停まっています。各駅には2つの画面とコメントカードの小さな山があり、1つの駅が1回の巡回に対応します。中央の大きなカートがカードを集めて箱に入れ、ティール色の引き出しには選ばれたカードが入っています。ティール色の破線が棒グラフ付きのレポートカードへ続いています。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/img-explain.webp)

*各巡回では新着コメントだけを取得し、重要なものを残して1つのレポートに追加します。* [本記事用に生成したイラスト（rolldek gpt-image-2.5）。流れはAntigravity CLIのヘッドレスモードのドキュメントに沿っています](https://antigravity.google/docs/cli/headless)

## 目的：24時間のコメント巡回で拾うべきもの

新しい動画のコメントの大半は対応不要です。絵文字、ジョーク、「一番乗り」、ファン同士の雑談がほとんどを占めます。巡回の目的は、対応が必要な一部を見つけて1画面にまとめることです。SNS担当者がスマートフォンで2分あれば読めるレポートにすべきです。実際に対応が必要な3件を見つけるのに、YouTube Studioで1,500件ものコメントをスクロールするようではいけません。

状況の異なる動画を2本選びました。YouTubeの動画はLinus Tech Tipsの「GTA 6 on PC May be Coming Sooner than We Thought」（[動画ページ](https://www.youtube.com/watch?v=g9IOdTisQyM)）です。11月2日00:45（SGT）に公開され、最初の取得時点で公開から23.6時間、再生回数794,104回、コメント1,546件でした。

Bilibiliの動画はDeepSeek Harnessアカウントの初投稿で、11月2日21:28（SGT）に公開されました（[動画ページ](https://www.bilibili.com/video/BV1i24P6rEmt)）。最初の取得時点で公開から2.9時間、再生回数225,698回、コメント4,075件、弾幕342件でした。1本は24時間の対象期間の終盤、もう1本は序盤にあたります。

| レポートの1行 | 取得元フィールド | 備考 |
| --- | --- | --- |
| コメントID、プラットフォーム、動画 | `youtube/comments`または`bilibili/comments`の`id`と`video_id` | IDは残し、ユーザー名は削除 |
| 本文（200文字に切り詰め） | `text` | 弾幕は別の列に格納 |
| 時間枠 | YouTubeは「2 hours ago」形式、BilibiliはISO形式の時刻 | 取得時にYouTubeのラベルを変換 |
| エンゲージメント | `like_count`、`reply_count` | 新しいコメントは通常0 |
| カテゴリ | スクリプトによる事前フィルターとモデルによるチェック | 苦情、質問、要返信、その他 |
| 重大度と返信案 | JSONスキーマに沿ったモデル出力 | 高リスクの項目は返信案を作らず人に回す |

![フローのアニメーション。上部の破線の枠には初回のみのセットアップが並びます。agyに対話モードでログイン、.agents/mcp_config.jsonにEveryInfraを追加、3つのツールを許可。6枚の角丸カードが順に点灯します。1 タイマー（launchd、cron、systemd）。2 agy -pがcomment-patrolスキルを読み込む。3 MCPツールを3回呼び出して取得。4 スクリプトで重複排除とフィルタリング（モデルのトークンは使わない）。5 JSONスキーマ出力でモデルがチェック。6 レポートを出力し、stderrと終了コードを確認。中央には1回の巡回あたりのAPI費用として3回の呼び出しで$0.001668と表示され、モデルの利用料は別途かかります。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/fig-02-patrol-flow.ja.light.svg)

## Antigravity CLIの概要とこのジョブに必要な4つの機能

「antigravity cli」で検索する人の多くは、いまだにGemini CLIの新しい名前だと考えています。実際には別のツールです。2026年5月19日、Google Developers Blogは個人ユーザー向けにGemini CLIからAntigravity CLIへ移行すると発表しました（[Google Developers Blog](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/)）。新しいツールは同日からすべてのユーザーが利用できるようになりました。

6月18日以降、Gemini CLIとGemini Code AssistのIDE拡張機能は、Google AI Pro、Ultra、無料ユーザーへの提供を終了しました。Gemini Code Assistのエンタープライズライセンスまたは有料APIキーを持つ組織は引き続き利用できます。同じスケジュールは[GitHub discussion #27274](https://github.com/google-gemini/gemini-cli/discussions/27274)にも記載されています。

ブログ記事によると、チームはツールをGoで書き直し、コマンド名を`agy`にしました。Agent Skills、フック、サブエージェント、拡張機能は引き続き使えますが、拡張機能はプラグインという名前に変わりました。また、CLIはAntigravity 2.0デスクトップアプリと同じエージェントコアを共有しています。このガイドで重要な事実は2つあります。

1つ目は、Antigravity CLIがクローズドソースであることです。[google-antigravity/antigravity-cli](https://github.com/google-antigravity/antigravity-cli)リポジトリにあるのはインストール手順、リリースノート、ディスカッションで、コードは含まれていません。そのため、このガイドに記載した動作はすべて公式ドキュメントとリリースノートに基づいています。

2つ目は、CLIの変化が速いことです。GitHub releases APIには、2026-10-06の1.3.0から2026-10-30（UTC）の1.4.12まで、20件のリリースが並んでいます（[1.4.12のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.12)）。その間には、2026-10-16の1.4.0、2026-10-21の1.4.4、2026-10-27の1.4.8、2026-10-29の1.4.10などがあります。1〜2日に1回のペースでリリースされている計算です。

| バージョン | 日付（UTC） | 定期巡回に関わる変更 |
| --- | --- | --- |
| 1.2.9 | 2026-09-23 | ヘッドレスモードは`--print-timeout`まで（最大30分）バックグラウンドタスクを待機 |
| 1.2.10 | 2026-09-24 | ストリーミング中の回答が途中で失敗すると終了コード3で終了 |
| 1.3.4 | 2026-10-13 | ヘッドレスモードで不明な`--agent`名を指定すると終了コード1で終了 |
| 1.3.6 | 2026-10-15 | ヘッドレスモードではサブエージェントからの質問をスキップ |
| 1.4.7 | 2026-10-24 | `-p`実行時のCPU使用量を削減。10 KBまでのMCP結果はエージェントに直接渡される |
| 1.4.11 | 2026-10-30 | JSON出力に`finished_reason`を追加 |

以下の例をコピーする前に、ローカルで`agy --version`を実行し、表示されたバージョンをドキュメントの記載と照らし合わせてください。

| このジョブで使う機能 | ドキュメントの記載 | 出典（2026-11-03に確認） |
| --- | --- | --- |
| ヘッドレスモード`agy -p` | プロンプトを1つ実行し、結果を出力して終了。`--output-format json`は`status`と`usage`を含むエンベロープを返す。`--json-schema`で構造化出力を強制 | [ヘッドレスモード](https://antigravity.google/docs/cli/headless) |
| リモートMCP | サーバーは`mcp_config.json`に定義。リモートサーバーは`serverUrl`と`headers`を使用。`enabledTools`で一部のツールだけを公開 | [MCPドキュメント](https://antigravity.google/docs/mcp?tab=cli)、[移行ガイド](https://antigravity.google/docs/cli/gcli-migration/) |
| エージェントスキル | ワークスペースのスキルは`.agents/skills/`に、グローバルのスキルは`~/.gemini/skills/`または`~/.gemini/antigravity-cli/skills/`に配置 | [エージェントスキル](https://antigravity.google/docs/skills?tab=cli) |
| スケジュール実行 | `/schedule`は現在の会話内でのみ有効。Sidecarsはセッションをまたいで動作するが、デスクトップアプリ向けとされている。CLIはOSのタイマーから実行可能 | [/schedule](https://antigravity.google/docs/commands/schedule)、[Sidecars](https://antigravity.google/docs/sidecars) |

## Antigravity CLIのMCP設定：mcp_config.jsonの場所と記述内容

Antigravity CLIでは、MCPサーバーの設定がメインの設定ファイルから`mcp_config.json`に移りました。グローバルファイルは`~/.gemini/config/mcp_config.json`です。プロジェクト用のファイルはプロジェクトルートの`.agents/mcp_config.json`に置きます（[移行ガイド「MCP config formatting changes」](https://antigravity.google/docs/cli/gcli-migration/)）。リモートサーバーのアドレスを指定するキーは、Gemini CLIの`url`または`httpUrl`から`serverUrl`に変わりました。

MCPドキュメントによると、`url`も引き続き使えます。認証情報はカスタムの`headers`に記述します。`enabledTools`キーは許可リストで、エージェントはそのサーバーの他のツールを一切参照できません。`timeoutSeconds`キーのデフォルトは、接続が300秒、ツール呼び出し1回あたり180秒です（[MCPドキュメント「MCP configuration properties」](https://antigravity.google/docs/mcp?tab=cli)）。

パトロール用プロジェクトの設定例です。`.agents/mcp_config.json`に記述します。

```text
{
  "mcpServers": {
    "everyinfra": {
      "serverUrl": "https://api.everyinfra.com/mcp",
      "headers": { "Authorization": "Bearer <your EveryInfra API key>" },
      "enabledTools": ["everyinfra_call_api", "everyinfra_get_job", "everyinfra_list_capabilities"],
      "timeoutSeconds": 120
    }
  }
}
```

各行にはそれぞれ意図があります。`enabledTools`には3つのツールだけを残しています。11月3日00:22（SGT）にAPIキーなしで`tools/list`を呼び出したところ、EveryInfra MCPサーバーには12個のツールがありました。そのうちアカウント照会やサポートチケットなど、いくつかはパトロールとは無関係です。余分なツールはコンテキストを消費し、モデルが誤ったツールを選ぶ余地を増やします。

`timeoutSeconds`は180ではなく120に設定しています。最も遅かった同期呼び出しは70.48秒で、人気順に並べたYouTubeコメント50件の取得でした。85秒を超える可能性がある呼び出しでは、リクエストに`"mode":"async"`を追加し、`everyinfra_get_job`をポーリングしてください。ポーリングは無料です。タイムアウトを延ばし続けるのは避けてください。

最も注意が必要なのは、APIキーを記述する行です。公式のリモート設定例では、`Bearer YOUR_API_TOKEN`を`headers`に直接書いています。ドキュメントには、`headers`で`${ENV_VAR}`が展開されるとは書かれていません。旧Gemini CLIのドキュメントでは、ヘッダー内の`$VAR_NAME`と`${VAR_NAME}`がサポートされていました（[Gemini CLI MCPドキュメント](https://github.com/google-gemini/gemini-cli/blob/main/docs/tools/mcp-server.md)）。

あるRedditユーザーは、Antigravityが`${MY_VAR}`をそのままの文字列として送信すると報告しています（[r/google_antigravityのスレッド](https://www.reddit.com/r/google_antigravity/comments/1s8i261/i_finally_looked_inside_my_antigravity_mcp_config/)）。ドキュメントで明記されるまでは、展開されないものと考えてください。APIキーはグローバルファイルに書いて自分だけが読めるようにするか、プロジェクトファイルを使う場合は`.agents/mcp_config.json`を`.gitignore`に追加してください。決してコミットしないでください。

2つ目のファイルは権限設定です。MCPドキュメントによると、ルールのないMCPツールはデフォルトで「Ask」になり、呼び出しのたびに人の承認を待ちます。許可ルールは`mcp(server/tool)`または`mcp(server/*)`の形式で書きます（[MCPドキュメントの権限の項](https://antigravity.google/docs/mcp?tab=cli)、[権限のドキュメント](https://antigravity.google/docs/permissions?tab=cli)）。ヘッドレスモードでは誰も承認しません。そのため、`~/.gemini/antigravity-cli/settings.json`にあらかじめルールを書いておきます。

```text
{
  "permissions": {
    "allow": [
      "mcp(everyinfra/everyinfra_call_api)",
      "mcp(everyinfra/everyinfra_get_job)",
      "mcp(everyinfra/everyinfra_list_capabilities)",
      "write_file(reports/)"
    ]
  }
}
```

`mcp(everyinfra/*)`も`--dangerously-skip-permissions`も使いません。ワイルドカードより、名前を指定した許可リストのほうが安全です。後からサーバーにツールが追加されても、名前指定のルールはそれを対象にしません。別のRedditユーザーもまさにこの手順でつまずきました。MCPサーバーは接続済みと表示されているのに、エージェントは接続できないと答えたのです。

解決策は`/permissions`で明示的に許可することでした。このユーザーは、設定が`~/.gemini/config/`ではなく`~/.gemini/antigravity-cli/`に書き込まれていたことにも気づいています（[r/google_antigravity「Antigravity CLI MCP Servers Pain」](https://www.reddit.com/r/google_antigravity/comments/1uhgm44/antigravity_cli_mcp_servers_pain/)）。セットアップが済んだら、一度対話モードでCLIを開いてください。`/mcp`でサーバーの状態を確認し、続けて`/permissions`でルールを確認します。両方の確認が通ってから、ジョブをタイマーに任せてください。

## 実際の1往復：エージェントが送る内容と返ってくる内容

`agy`は実行できませんでしたが、これがリモートMCPサーバーに送るリクエストは標準的なMCPメッセージです。クライアントはまず`tools/list`でツールカタログを取得し、その後必要に応じて`tools/call`を送ります。MCP仕様ではこのトランスポートをStreamable HTTPと呼びます。クライアントはJSON-RPCメッセージを1つPOSTし、サーバーはJSONまたはSSEストリームで応答します（[MCP仕様 2026-07-28「Transports」](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports)）。そこでクライアントを介さず、同じリクエストをEveryInfra MCPエンドポイントに直接送ります。

ここで確認できるのは、サーバーがエージェントに返す生の応答です。エージェントが実際に読む内容は、CLI側で変わる可能性があります。たとえば1.4.7のリリースノートには、10 KBを超えるツール結果はファイルに書き出されるとあります。この層はテストしていません。

11月3日00:45:27（SGT）に、APIキーを付けて次の`tools/call`を送信しました。YouTube動画の最新コメント100件を要求するものです。

```text
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/call",
  "params": {
    "name": "everyinfra_call_api",
    "arguments": {
      "platform": "youtube",
      "action": "comments",
      "params": {
        "url": "https://www.youtube.com/watch?v=g9IOdTisQyM",
        "sort": "newest",
        "limit": 100
      }
    }
  }
}
```

サーバーは30.99秒後にHTTP 200を返しました。MCPは処理結果を`result.content[0].text`に33,915バイトのJSON文字列として格納します。中には`platform`、`action`、`count`、`results`、`billing`が含まれます。`count`は100、`billing.charged_usd`は0.000556でした。最初のコメントは3分前に投稿されたものでした。

ある視聴者は自分のPCの構成を挙げ、番組の「平均的なゲーマーのPC」は少し高性能すぎると書いていました。2件目のコメントは9分前の投稿で、ゲームのリメイク版をエミュレーターでクリアしたという内容でした。下のスクリーンショットは、2件を含む実際のレスポンスです。ユーザー名は伏せ、本文は短く切っています。

![EveryInfraのMCPサーバーがtools/callの呼び出しに返した実際のレスポンスのスクリーンショットです。ヘッダー行にはPOST https://api.everyinfra.com/mcp、tools/call、00:45:27 SGT、30.99sと表示されています。上段はリクエストで、ツール名everyinfra_call_api、platform youtube、action comments、sort newest、limit 100が指定されています。下段はレスポンスの抜粋で、count 100、3分前と9分前にそれぞれ投稿されたコメント、マスクされた投稿者名、残り98件の表記、課金情報のcharged_usd 0.000556が表示されています。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/img-api-response.webp)

*ラウンド2のMCP経由のYouTubeコメント。100件を1回の呼び出し（$0.000556）として課金。* [EveryInfra MCPの実際のレスポンスのスクリーンショット（2026-11-03 00:45 SGT）。機能の詳細：](https://api.everyinfra.com/api/v1/social/catalog?platform=youtube)

設定ファイルを書く際は、このレスポンスサイズを考慮してください。Antigravity CLI 1.4.7のリリースノートによると、オンデマンドのMCP結果は10 KBまでならそのままエージェントに渡されます。それより大きい結果はファイルに書き出され、エージェントは追加のステップでそのファイルを読む必要があります（[1.4.7リリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.7)）。コメント100件の1ページは約33 KBなので、必ずファイル経由になります。

パトロール用途ではこれで問題ありません。生のレスポンスはどのみちディスクに保存しておきたいからです。スキルでは、33 KBのJSONをコンテキストに読み込む代わりに、そのファイルを`dedupe.py`に渡すようエージェントに指示しています。

MCPを使わずにRESTエンドポイントを呼び出すこともできます。タイマースクリプトが`agy`を起動する前にデータを取得する場合は、こちらを使ってください。1回目と3回目ではこの方法を使いました。`results`配列の形は、MCPレスポンス内のものと同じです。

```bash
curl -s https://api.everyinfra.com/api/v1/social \
  -H "Authorization: Bearer $EVERYINFRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"platform":"youtube","action":"comments","params":{"url":"https://www.youtube.com/watch?v=g9IOdTisQyM","sort":"newest","limit":100}}' \
  > youtube-$(date +%Y%m%d-%H%M).json
```

おすすめは、まずMCPで始めて、良いレポートがどのようなものかを確認することです。安定したら取得と重複排除をスクリプトに移し、判断だけをモデルに任せます。その理由は事前フィルタのセクションで説明します。

エラーも一連のやり取りに含まれます。`bilibili/comments`に、対応していない`sort`パラメータを送りました。00:21:33 SGTに、`unknown_param`を含むHTTP 422が返ってきました。メッセージには許可されたパラメータ`url`と`limit`が列挙され、最後に「Nothing was charged.」と書かれていました。モデルはこのメッセージを読み、対応していないパラメータを外して再試行できます。

サーバー側の障害の場合、エラーの内容は異なります。3回目の01:22:15 SGTに、弾幕の呼び出しがHTTP 503を返し、`fault`は`service`でした。つまり、リクエスト自体には問題がありませんでした。ボディでは`retryable`がtrue、`retry_after`が20秒に設定されていました。`next_steps`の行には、この呼び出しは課金されていないこと、そして3回中1回目の失敗であることが書かれていました。23秒後に同じリクエストを送るとHTTP 200が返り、100件の弾幕を取得でき、$0.000556が課金されました。タイマースクリプトでは、これらのフィールドに従ってください。`retry_after`で指定された時間だけ待ち、`max_retries`に達したら停止し、レポートに「今回はBilibiliの弾幕を取得できませんでした」と書きます。

リモートMCPサーバーのその他の接続方法については、[MCPサーバーガイド](https://everyinfra.com/ja/mcp-server)と[ドキュメント](https://everyinfra.com/ja/docs)をご覧ください。

| 今回のやり取り | 実測値 |
| --- | --- |
| リクエスト時刻 | 2026-11-03 00:45:27 SGT |
| ツール | `everyinfra_call_api`, platform=youtube, action=comments |
| 所要時間 | 30.99秒（同期。85秒未満ならそのまま返却） |
| レスポンス | HTTP 200、`count` 100、ボディ33,915バイト |
| 課金額 | $0.000556（`billing.charged_usd`） |
| 1回目との比較 | 100件中91件は既出、9件が新規 |

## YouTubeコメントの監視：1回100件で何時間分をカバーできるか

巡回間隔は、実用上の1つの指標で決まります。2回のラウンドのあいだに増える新規コメント数が、1回の呼び出しで返せる件数を超えるかどうかです。カタログでは、`youtube/comments`のデフォルトは20件、最大は100件です。任意パラメータは`sort`（newestまたはpopular）と`since`（日付フィルター）です。

フィールドは`id`、`text`、`author_name`、`like_count`、`reply_count`、`posted_at`、`is_reply`、`reply_to_id`です。ページング用のパラメータはありません。そのため、1回のラウンドで取得できるのは100件までです。

ラウンド1は00:18（SGT）に実行しました。動画は公開から23.6時間が経過しており、コメントは1,546件でした。新しい順に100件を要求し、呼び出しには22.2秒かかりました。最新のコメントは「1分前」、最も古いコメントは「5時間前」でした。公開1日目の終わりの時点で、100件がカバーするのは直近5時間だけでした。

3件は返信（`is_reply`がtrue）で、4件には「(edited)」が付いていました。ここから逆算すると、この動画は公開18時間後から23時間後のあいだも、1時間に約20件のコメントが増えていたことになります。6時間に1回のラウンドでは、2回の取得のあいだに埋もれて一度も表示されないコメントが出ます。

初日以降も、コメントは1時間あたり約20〜45件の安定したペースで増え続けました。

前回から27分後、00:45（SGT）のラウンド2では、動画のコメントは11件増えて1,557件でした。最新100件のうち9件が新規、91件が既出でした。100件がカバーするのは、やはり直近5時間だけでした。

前回から35分後、01:20（SGT）のラウンド3では、コメントは26件増えて1,583件でした。最新100件のうち、それまでの2回のどちらにも含まれない新規が18件、既出が82件でした。ラウンド2だけと比べると、新規が19件、重複が81件でした。100件がカバーするのは、やはり直近5時間だけでした。

新規行の数は、総数の増加分より少なくなっています。差の一部は返信で説明できます。最新100件中の返信は、1回目が3件、2回目が0件、3回目が1件でした。残りは非表示や折りたたまれたコメントによるものかもしれません。毎回両方の数値を記録し、差があればレポートで指摘してください。

![巡回ごとに、動画の公開経過時間と最新100件のコメントがカバーする時間を並べた集合棒グラフです。1回目（00:18）：公開経過23.56時間、最新100件のカバー範囲は最大5時間。2回目（00:45）：公開経過24.0時間、最新100件のカバー範囲は最大5時間。3回目（01:20）：公開経過24.59時間、最新100件のカバー範囲は最大5時間。ティール色の棒がカバーされた時間を示します。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/fig-03-yt-window-bar.ja.light.svg)

![巡回ごとに、最新100件のコメントに含まれる新規行と既出行を示した積み上げ棒グラフです。1回目（00:18）：100件すべてが新規。2回目（00:45）：既出91件、新規9件（ティール）、コメント数は11件増加。3回目（01:20）：既出82件、新規18件（ティール）、コメント数は26件増加。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/fig-04-new-per-round-bar.ja.light.svg)

`sort`パラメータは見落としがちです。同じタイミングで、人気順に並べたコメント50件も取得しました。この呼び出しには70.48秒かかり、今回の同期呼び出しの中で最も遅いものでした。人気順の50件と最新の100件には、共通する行が1つもありませんでした。

人気順のセットでは30件が返信でした。表示ラベルは「9分前」から「23時間前」までで、トップのコメントには3,500件のいいねがありました。巡回の主な目的は新規コメントを1件も見逃さないことなので、各ラウンドでは新しい順を使います。人気順は1日1回、別の問いに答えるために使います。いま議論の方向を決めているのはどのコメントか、という問いです。

`posted_at`フィールドはタイムスタンプではなく、「N時間前」のような相対ラベルです。スクリプトで取得時刻を記録し、各ラベルを時間幅に変換する必要があります。重複排除はコメントの`id`だけで行ってください。時刻は決して使わないでください。

| フィールド | 1回目の計測値 | パトロールでの用途 |
| --- | --- | --- |
| `id` | 一意な値が100個、`Ugz…AaABAg`のような形式 | 唯一の重複排除キー。`state/seen-<video ID>.json`に保存 |
| `text` | 全文。最長の行は1,876文字 | レポートでは200文字に切り詰め、全文はローカルに保持 |
| `posted_at` | 「1分前」〜「5時間前」、4件に「(edited)」 | 取得時に変換。編集されたコメントは内容が変わっている可能性あり |
| `like_count`、`reply_count` | 最新100件のうち、いいねが付いていたのは3件だけ | 新しいコメントにはまだ反応がないため、反応を見るなら人気順を使う |
| `is_reply`、`reply_to_id` | 返信3件 | 返信は親コメントと併せて読む。そうしないと雰囲気を読み違える |
| `author_name` | あり | ローカルでのみ使用し、レポートには含めない |

YouTubeについての推奨は、最初の6時間は1時間に1回、その後は2時間に1回ラウンドを実行することです。あるラウンドで新規がちょうど100件見つかったら、レポートの1行目に「このラウンドではコメントを取りこぼした可能性があります」と書きます。そして次のラウンドを前倒しします。このルールは、以下のスキルの判断ツリーにすでに組み込んであります。すべてのフィールドとパラメータについては[YouTubeコメントAPI](https://everyinfra.com/ja/youtube-api/comments)をご覧ください。コメント数と再生回数の合計は[YouTube動画API](https://everyinfra.com/ja/youtube-api/video)から取得します。

## Bilibili：人気コメントのみ、弾幕は第2のシグナル

Bilibiliのサンプルは投稿されたばかりの動画です。DeepSeek Harnessのアカウントは、11月2日21:28 SGTに最初の動画を投稿しました。長さは200秒です。00:20 SGTに最初の`bilibili/video`の取得を行った時点で、動画は公開からわずか2.9時間でした。それでもすでに再生数225,698、いいね22,070、コメント4,075、弾幕342に達していました。動画が公開直後の数時間で急激に伸びているときこそ、パトロールが最も重要です。

しかし、00:21 SGTに`bilibili/comments`が返したコメントは3件だけで、所要時間は1.26秒でした。3件とも人気コメントで、いいね数は48〜212でした。投稿されたのは、動画公開の3〜131分後です。00:46 SGTのラウンド2では、同じリクエストで最大件数の10件が返りました。

そのうち7件は初めて見るコメントで、いいね数は23〜1,100でした。同じ25分間に、動画のコメント数は4,075から4,566に増えています。491件の増加で、1時間あたり1,000件を超えるペースです。01:22 SGTのラウンド3では、再び`limit`を10に設定したにもかかわらず返ったのは3件だけで、3件とも既出のコメントでした。

このエンドポイントについて、カタログの記載は明確です。デフォルトは10、最大も10で、それより大きい値を指定しても10に制限されます。パラメータは`url`と`limit`のみです。`sort`パラメータを試したところ、前述の422エラーが返り、課金はありませんでした。つまりこのエンドポイントでは、最新のコメントを時系列順に取得することはできません。

得られるのは、その時点で最も人気のあるコメント最大10件で、件数も一定しません。それでも価値はあります。人気コメントが第一印象を決めるからです。ネガティブな人気コメントは最優先の項目です。ただし、新しいコメントをすべて読むことの代わりにはなりません。読者が全件を網羅していると思い込まないよう、レポートには「Bilibili：全4,566件中、人気コメント10件を確認」と明記する必要があります。

弾幕は2つ目のシグナルです。`bilibili/danmaku`のデフォルトは50件、最大は200件です。200件を要求したところ、返ってきたのは100件でした。弾幕は動画内の時刻順に0秒から98秒まで並んでおり、200秒の動画の前半しかカバーしていませんでした。送信時刻は21:35から00:19（SGT）まででした。

キーワードルールでは、そのうち29件が質問と判定されました。しかし29件中15件は、「クジラ！？本当に生きているの！？」のような、疑問符付きの歓声でした。実質的な内容は残りのいくつかにありました。4件の弾幕が「サバイバルモード」について尋ね、1件がデスクトップアプリを求め、1件がアカウントをフォローすると無料トークンがもらえるのかを尋ねていました。疑問符があっても質問とは限らず、弾幕ではなおさらです。

| 弾幕（意訳） | 動画内の時間（秒） | ルールの判定 | 私たちの判断 |
| --- | --- | --- | --- |
| 「クジラ！？本当に生きているの！？」 | 0〜10 | 質問 | 歓声。対応不要 |
| 「サバイバルモードはありますか？」 | 76 | 質問 | 機能に関する質問。共通テーマの一部 |
| 「ちょうどデスクトップアプリを探してたところ」 | 8 | 質問 | 機能に関する質問 |
| 「フォロワーは無料トークンをもらえる？」 | 53 | 質問 | 半分冗談、半分質問 |
| 「本当に定時で上がれるの？」 | 25 | 質問 | カウントダウンプラグインについての冗談 |

その後、サバイバルモードの話題はラウンド2の人気コメントにも現れました。352いいねを集めたコメントが、動画ではサバイバルモードに一切触れていないと指摘していたのです。散らばった4件の弾幕と1件の人気コメントが、同じ点を示しています。良いレポートなら、これらを「視聴者がサバイバルモードについて繰り返し質問している」という1行にまとめます。

ラウンド2の弾幕も100件で、98秒で終わっていました。そのうち21件が新規で、弾幕の総数は342件から388件に増えました。ラウンド3もやはり98秒で止まり、新規は16件、総数は430件でした。このエンドポイントは、動画前半の最初の100件の弾幕を返します。新しい弾幕は、タイムラインのその範囲に投稿された場合にしか表示されません。

![Bilibiliへの1回の呼び出しで見える範囲を示す横棒グラフ。ラウンド1のコメント：4,075件中3件（0.1%、青緑）。ラウンド2のコメント：4,566件中10件（0.2%、青緑）。ラウンド3のコメント：5,121件中3件（0.1%、青緑）。ラウンド3の弾幕：430件中100件。弾幕がカバーするのは、200秒の動画のうち98秒まで。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/fig-05-bili-coverage-bar.ja.light.svg)

| 機能 | カタログ | 実際の結果 | パトロールでの用途 |
| --- | --- | --- | --- |
| `bilibili/video` | 再生数、いいね数、コメント数、弾幕数を含む26フィールド | 1回の呼び出しに35.3秒、全フィールドあり | 4ラウンドごとに、コメントと弾幕の増加を追跡 |
| `bilibili/comments` | デフォルト10、最大10。パラメータは`url`と`limit`のみ | 3ラウンドで人気コメントは3件、10件、3件。`sort`は422を返し、課金なし | 人気コメントの監視：人気コメントが入れ替わったり、ネガティブに転じたりしたら報告 |
| `bilibili/danmaku` | デフォルト50、最大200 | 200を指定して100行を取得。動画内の時間順で、3ラウンドとも98秒で停止 | 動画のどの箇所で視聴者が声援を送ったり質問したりしているかを把握し、問題のある箇所を見つける |

自分のBilibiliアカウントを運用していて、すべてのコメントが必要な場合は、クリエイターセンターを使ってください。あるいは、Bilibili Open Platformに申請する方法もあります。こちらは開発者登録、資格審査、アプリ審査が必要です（[Bilibili Open Platform](https://open.bilibili.com/)）。YouTube Data APIと比べて、手続きはかなり重くなります。

競合動画の場合や、Bilibili Open Platformを利用しない場合は、取得できる範囲が「人気コメント＋弾幕」に限られることを受け入れ、その旨をレポートに明記してください。パラメータの詳細は[BilibiliコメントAPI](https://everyinfra.com/ja/bilibili-api/comments)と[Bilibili弾幕API](https://everyinfra.com/ja/bilibili-api/danmaku)のページにあります。プラットフォームの概要は[YouTube API](https://everyinfra.com/ja/youtube-api)と[Bilibili API](https://everyinfra.com/ja/bilibili-api)のページにあります。

## パトロールをスキル化する：.agents/skills/comment-patrol/の雛形

Antigravityでは、スキルは`SKILL.md`ファイルを含むフォルダです。YAMLヘッダーで必須なのは`description`だけです。`name`を省略すると、フォルダ名が使われます。各ターンでエージェントが参照するのは、すべてのスキルの名前と説明だけです。関連がありそうなスキルについてのみ、全文を読み込みます。

ドキュメントでは、これを「段階的開示」と呼んでいます（[Agent Skillsのドキュメント](https://antigravity.google/docs/skills?tab=cli)）。形式はオープンなAgent Skills標準に準拠しています（[agentskills.io](https://agentskills.io/home)）。この標準に対応する他のツールでも、パスを少し変更するだけで同じスキルを使えます。ドキュメントではさらに、パトロールに適したヒントを2つ紹介しています。スクリプトはブラックボックスとして扱い、まずエージェントに`--help`を実行させてください。複雑なフローは決定木として記述してください。

```text
.agents/
  mcp_config.json            # remote MCP config from above (keep out of Git)
  skills/
    comment-patrol/
      SKILL.md
      scripts/
        dedupe.py            # reads state/seen.json, keeps only new rows, turns "2 hours ago" into a window
        prefilter.py         # keyword pre-filter: question, negative, needs reply
      resources/
        report.schema.json   # output schema for --json-schema
state/
  seen-<video ID>.json       # comment IDs already seen, with first-seen time
reports/
  <video ID>-<round>.md
```

```text
---
name: comment-patrol
description: Patrol YouTube and Bilibili comments in the 24 hours after a video goes live. Fetch new comments, dedupe, pre-filter, and flag complaints, questions and comments that need a reply. Use when the user asks for a comment patrol, post-launch monitoring, or a check of the comment section.
---

# Comment patrol

## Each round
1. Read `state/seen-<video ID>.json`. Create it in round 1.
2. Call everyinfra_call_api. Use only these capabilities:
   - YouTube: platform=youtube, action=comments, params={url, sort: "newest", limit: 100}
   - Bilibili: platform=bilibili, action=comments, params={url}; then action=danmaku, params={url, limit: 200}
   - Every 4 rounds: youtube/video and bilibili/video, to record comment_count growth
3. Save each response as-is under `data/<round>/`. Run `python3 scripts/dedupe.py --help`, then use it to keep only new rows.
4. Run `scripts/prefilter.py`. Judge only rows it marks as question, negative or needs-reply, plus any theme you see repeated.
5. Output per `resources/report.schema.json`: category, severity, needs_reply, and a draft reply of at most two sentences. Never promise prices or dates.

## Decision tree
- New YouTube rows = 100 (the per-call cap): comments were missed between rounds. Write "this round may have missed comments" on line 1 and suggest a shorter interval.
- Zero new rows: do not call the model. Write "no new comments".
- Refunds, safety, legal or privacy: severity=high, no draft reply, hand to a human.
- Emoji only, "first", or similar: drop it.

## Never
- Never post a comment or reply from the account. This skill is read-only.
- Never put user names or profile links in the report. Use comment IDs only.
```

重複排除と事前フィルターを、モデルに数えさせるのではなくスクリプトに任せるのはなぜでしょうか。第1にコストです。コメント100件の生のレスポンスは約8,075トークンです。ラウンド2では新規は9件だけで、約505トークンでした。第2に、監査できることです。

スクリプトは固定のルールを適用するので、同じ入力からは2回実行しても同じ出力が得られます。モデルは言語の理解が必要なステップのために取っておきましょう。`report.schema.json`のアイデアは、ヘッドレスモードの`--json-schema`から来ています。このフラグはスキーマファイルのパスを受け取ります。パースされたオブジェクトは、出力エンベロープの`structured_output`フィールドに入ります（[ヘッドレスモードの「Structured output with a schema」](https://antigravity.google/docs/cli/headless)）。タイマースクリプトはこのフィールドだけを読み、自由記述のテキストから表をパースすることはありません。

## 事前フィルター：キーワードルールで拾えるもの、拾えないもの

モデルにコメントを渡す前に、ラウンド1のYouTubeコメント100件に固定ルールを適用しました。ルールは00:29 SGTに作成し、その後データが増えても変更していません。質問と判定する条件は、疑問符があること、中国語の疑問詞を含むこと、またはhow、why、what、is、canなどの語で始まることです。ネガティブ判定用の単語リスト（bad、worst、disappointed、clickbait、scam、brokenなど）に含まれる語があれば、ネガティブと判定します。

質問である場合、またはネガティブでいいねが10以上の場合は、返信が必要と判定します。ルールが検出したのは質問28件とネガティブ8件で、合計34件のコメントでした。弾幕100件では質問29件を検出し、ネガティブは0件でした。

| 規定 | 定義 | YouTubeコメント100件中のヒット数 | 弾幕100件中のヒット数 |
| --- | --- | --- | --- |
| 質問 | 疑問符、文頭の疑問詞、または中国語の疑問詞 | 28 | 29 |
| ネガティブ | ネガティブリストに含まれる語 | 8 | 0 |
| 要返信 | 質問、またはいいね10件以上のネガティブ | 28 | 29 |
| いずれかのフラグ | 質問またはネガティブ | 34 | 29 |

次に、フラグが付いた34件のコメントをすべて目視で確認しました。基準は「この動画に関するコメントで、クリエイターが把握するか回答する必要があるもの」です。これを満たしたのは13件だけでした。残り21件の大半は反語的な質問や議論でした。

「AIで作られていたって誰が気にするの？」には疑問符が付いていますが、誰も答える必要はありません。あるコメントはゲストの主張を「くだらない」と評したもので、「What」で始まっていたため、ルールが質問と判定しました。有用な13件は明確なタイプに分かれます。ある視聴者は、なぜ動画が30 fpsでアップロードされたのかを尋ねていました。

2人は9:43のぼかしバーについて質問していました。また、番組内でゲームのテストに戻ると約束していたのに、結局戻らなかったと指摘した人もいました。動画が海賊版を助長しているのではないかと尋ねる人もいました。これらこそパトロールレポートの上位に載せるべき内容です。

![集合棒グラフ：ラウンド1でYouTubeの最新コメント100件に事前フィルターを適用した結果。ルールによるフラグは合計34件で、うち有用なものは13件（青緑色）。テーマ「ぼかし漏れ」：4件中、ルールが検出したのは2件。テーマ「ハイエンドではない」：10件中2件を検出。テーマ「脱獄動画のリクエスト」：6件中1件を検出。キーワードルールは反語的な質問を誤検知し、疑問符のない繰り返しのフィードバックを見逃します。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/fig-06-prefilter-bar.ja.light.svg)

より重要なのは、ルールが見逃したものです。フラグが付かなかった66件のコメントには、繰り返し現れるテーマが少なくとも3つありました。1つ目はぼかし漏れです。4件のコメントが、ゲームフォルダにかけたぼかしが一瞬外れたと指摘していました。

そのうち1件は、チームに動画の見直しを求めていました。クリエイターがすぐに修正すべき制作上のミスですが、ルールが検出したのは疑問符付きの2件だけでした。2つ目は「これはハイエンドではない」です。ホストがテスト用PCを「それほどハイエンドではない」と言ったところ、10件のコメントが反論していました。

ルールが検出したのは2件です。3つ目は、PS5の脱獄動画を求める6件のコメントで、企画のヒントになります。ルールが検出したのは1件でした。

これらのテーマはコメントを読んだ後に見つけたものなので、ルールの評価には使いません。ここからわかることは1つです。キーワードルールは、明らかなノイズと明らかな質問を切り分けるのが得意です。疑問符のない同様の意見が繰り返し寄せられていることは、モデルがコメントをまとめて読み、テーマごとにクラスタリングして初めて浮かび上がります。

| コメント（言い換え、ユーザー名なし） | ルールの判定 | 私たちの判断 | レポートに記載すべき内容 |
| --- | --- | --- | --- |
| なぜ30 fpsでアップロードされたの？ | 質問 | 有用 | 制作上の問題。固定コメントで回答 |
| 9:43のぼかしバーは何？ | 質問 | 有用（「ぼかし漏れ」テーマ） | 優先度高：見せてはいけないものが映っている可能性 |
| フォルダのぼかしが外れています。動画を確認してください | フラグなし | 有用（同じテーマ） | 上の行に統合し、件数を+1 |
| これがハイエンドじゃないなら、自分のPCは何なの？ | フラグなし | 有用（「ハイエンドではない」テーマ） | 1行に要約：10件のコメントがこの発言に反論 |
| PS5の脱獄動画を作ってください | フラグなし | 企画のヒント | 「視聴者の要望」に入れる。返信は不要 |
| AIで作られていたって誰が気にするの？！ | 質問 | 有用ではない（反語） | レポートから除外 |

ラウンド2のレポートは、たとえば次のようになります。ラウンド1と2の実際のレスポンスをもとに手作業で作成したもので、モデルの実行結果ではなく、目標とする出力を示しています。

| 優先度 | プラットフォーム | 項目 | 根拠 | アクション |
| --- | --- | --- | --- | --- |
| 高 | YouTube | フォルダのぼかしが一瞬外れた | ラウンド1で4件。ルールが検出したのは2件 | 9:43付近のフレームを確認し、再編集するか注記を固定 |
| 中 | YouTube | 「ハイエンドではない」という説明に視聴者が反論 | ラウンド1で10件、ラウンド2でさらに2件のコメント | テスト用PCについて短い補足を固定表示 |
| 中 | Bilibili | 視聴者が「サバイバルモード」について質問 | 弾幕コメント4件と、352いいねの人気コメント1件 | 固定コメントで回答 |
| 低 | YouTube | 視聴者がPS5の脱獄動画を希望 | コメント6件 | 企画リストに追加 |
| 情報 | 両方 | 今回の取得範囲 | YouTube：新規9件、総数は+11。Bilibili：4,566件中、人気コメント10件 | 2時間間隔を維持 |

そこで、スキルは作業を次のように分担します。スクリプトは重複を除き、絵文字だけのコメントや「first」とだけ書かれたコメントを除外し、各件にルールラベルを付けます。モデルには「今回の新規コメントとルールラベル」を渡します。モデルの役割は、誤検知を除外し、同じテーマのコメントを件数付きの1行にまとめ、テーマごとに深刻度を設定することです。

モデルに送るデータはわずかに増えるだけですが、ルールでは拾えないテーマも抽出できます。その理由はラウンド2ですでに明らかになりました。YouTubeの新着コメント9件のうち2件が、「平均的」とされたテスト用PCは性能が高すぎると改めて指摘していました。どちらのコメントにも「ハイエンド」という語は含まれていませんでした。

## Antigravity CLIのcron：launchd、cron、systemdからagy -pを実行

Antigravityには、作業をスケジュールする方法が2つ組み込まれています。しかし、サーバーやノートPCで数時間ごとに巡回を実行したいCLIユーザーには、どちらも合いません。`/schedule`を使うと、エージェントは組み込みの`schedule`ツールでタイマーまたは5フィールドのcron式を設定します。指定した時刻になると、その指示が優先度の高い通知としてエージェントに戻ってきます。ドキュメントには、スケジュールは作成元の会話に属し、その会話とともに終了すると明記されています。セッションをまたぐ長期的なジョブについては、ドキュメントはサイドカーを案内しています（[/scheduleのドキュメント](https://antigravity.google/docs/commands/schedule)）。

サイドカーは、`~/.gemini/config/sidecars/<name>/sidecar.json`で定義するバックグラウンドプロセスです。`"builtin": "schedule"`を指定すると、cron式に従って`agentapi new-conversation`を呼び出せます。ただし、そのページの冒頭では、対応環境としてAntigravityデスクトップアプリだけが挙げられています。また、サイドカーはデフォルトで無効になっており、`~/.gemini/config/config.json`で1つずつ有効にする必要があります（[サイドカーのドキュメント](https://antigravity.google/docs/sidecars)）。残る選択肢はOSのタイマーで、これでヘッドレスモードを起動します。

| スケジューラー | 有効期間 | 実行ごとのコンテキスト | スリープ中の動作 | 今回の巡回への適性 |
| --- | --- | --- | --- | --- |
| `/schedule`（CLI内） | 現在の会話のみ | チャット履歴全体を保持したまま起動 | プロセスが終了していると実行されない | 最初の2時間の監視には十分だが、24時間には不向き |
| サイドカー + `agentapi` | セッションをまたいで持続、アプリが管理 | 毎回新しい会話 | デスクトップアプリの起動が必要 | ドキュメントに記載されているのはデスクトップアプリのみ |
| macOS launchd + `agy -p` | システムレベル、ログイン後に実行 | 新しいプロセス、状態なし | `StartCalendarInterval`はスリープ解除時に1回実行 | 良好。Macでは推奨 |
| cron + `agy -p` | システムレベル | 新しいプロセス、状態なし | スリープ中やシャットダウン中の実行はスキップされ、後から実行されない | サーバーでは問題ないが、ノートPCでは巡回を取りこぼす |
| systemdタイマー + `agy -p` | システムレベル | 新しいプロセス、状態なし | `Persistent=true`で起動後に未実行分を1回実行 | Linuxサーバーでは推奨 |

スリープ時の列は、OSのドキュメントに基づいています。Appleによると、cronジョブはマシンのスリープ中は実行されず、次の予定時刻まで待機します。`StartCalendarInterval`を設定したlaunchdジョブは、スリープ中に予定時刻を過ぎた場合、スリープ解除時に実行されます。マシンの電源が切れている間は、どちらも実行されません（[Apple「Scheduling Timed Jobs」](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/ScheduledJobs.html)）。

Linuxでこれに相当するのは、systemdタイマーの`Persistent=`オプションです（[systemd.timerマニュアル](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html)）。cronの構文と環境のルールについては、[crontab(5)](https://man7.org/linux/man-pages/man5/crontab.5.html)を参照してください。cronはデフォルトで`/bin/sh`を使い、シェルのプロファイルを読み込まないため、`agy`はフルパスで記述してください。

次のlaunchdの例は、macOSで2時間ごとに1ラウンドを実行します。ファイルは`~/Library/LaunchAgents/`に置きます。スクリプトとデータは、デスクトップフォルダや書類フォルダに置かないでください。置くとmacOSのプライバシー保護によってブロックされます。

```text
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0"><dict>
  <key>Label</key><string>com.example.comment-patrol</string>
  <key>ProgramArguments</key>
  <array><string>/Users/you/patrol/run.sh</string></array>
  <key>StartCalendarInterval</key>
  <array>
    <dict><key>Minute</key><integer>5</integer><key>Hour</key><integer>0</integer></dict>
    <dict><key>Minute</key><integer>5</integer><key>Hour</key><integer>2</integer></dict>
    <!-- the other 10 even hours follow the same pattern, 12 rounds in total -->
  </array>
  <key>StandardOutPath</key><string>/Users/you/patrol/logs/out.log</string>
  <key>StandardErrorPath</key><string>/Users/you/patrol/logs/err.log</string>
</dict></plist>
```

`run.sh`スクリプトはAntigravityを呼び出します。ここには3つの落とし穴があります。

```bash
#!/bin/bash
set -uo pipefail
cd /Users/you/patrol
out=$(/usr/local/bin/agy -p "Use the comment-patrol skill on the videos in videos.txt. This is round ${ROUND:-1}." \
  --output-format json \
  --json-schema .agents/skills/comment-patrol/resources/report.schema.json \
  --model gemini-3.8-flash-medium \
  --print-timeout 15m 2> logs/last-stderr.log)
status=$(echo "$out" | jq -r '.status')
reason=$(echo "$out" | jq -r '.finished_reason // "complete"')
if [ "$status" != "SUCCESS" ] || [ "$reason" != "complete" ]; then
  echo "$(date -u +%FT%TZ) patrol failed: $status $reason" >> logs/alerts.log; exit 1
fi
if grep -qi "permission\|denied" logs/last-stderr.log; then
  echo "$(date -u +%FT%TZ) a tool was soft-denied, check permissions.allow" >> logs/alerts.log; exit 1
fi
echo "$out" | jq '.structured_output' > "reports/$(date +%Y%m%d-%H%M).json"
```

![Antigravity公式ドキュメントのヘッドレスモードのページにある「Permissions in headless mode」セクションのスクリーンショット。青緑色の枠で囲まれた段落には、承認が必要なツールで承認を得られない場合はソフト拒否され、実行は継続して終了コード0で終了し、stderrにそのツール名と許可する方法が通知されると書かれています。その下には、~/.gemini/antigravity-cli/settings.jsonのpermissions.allowでツールを事前に許可する例と、すべてのツールを一度に許可するフラグについての警告があります。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/img-doc-headless.webp)

*ヘッドレスモードでは、ツールが拒否されてもプロセスは失敗しません。終了コードは0のままです。* [Antigravityドキュメント「Headless mode」、2026年11月3日撮影のスクリーンショット](https://antigravity.google/docs/cli/headless)

1つ目の落とし穴はソフト拒否です。ドキュメントによると、ヘッドレスモードには対話型のプロンプトがありません。承認が必要なのに承認を得られないツールは、ソフト拒否されます。実行はそのまま続き、プロセスは終了コード0で終了し、通知がstderrに出力されます（[Headless mode「Permissions in headless mode」](https://antigravity.google/docs/cli/headless)）。`permissions.allow`のルールが間違っていると、定期実行はすべて「成功」し、レポートはすべて空になります。

そのため、スクリプトはstderrも読む必要があります。ドキュメントには通知の正確な文言が記載されていません。このスクリプトはキーワードで照合しています。本番運用の前に、あえて許可ルールを1つ削除して実際の通知を一度確認し、照合条件を調整してください。

2つ目の落とし穴はログインです。ヘッドレスモードはキャッシュされた認証情報を使うため、一度`agy`を対話モードで実行してログインしておいてください。ターミナルもログインもない環境では、「authentication required」というエラーで失敗します。READMEによると、認証情報はシステムのキーチェーンに保存されます（[GitHub README「Authentication」](https://github.com/google-antigravity/antigravity-cli)）。そのため、launchdジョブはログイン中のユーザーのセッションで実行する必要があります。LaunchDaemonではなくLaunchAgentを使ってください。

3つ目の落とし穴はタイムアウトです。`--print-timeout`のデフォルトは5分です。2つのプラットフォームへの複数回の呼び出しとモデルによる処理を合わせると5分を超えることがあるため、この例では15分にしています。2026年10月30日（UTC）にリリースされたバージョン1.4.11では、JSON出力に`finished_reason`が追加されました。実行が途中で打ち切られた場合、値は`print_timeout`になります。

ターンが正常に終了した場合、このフィールドは含まれません（[1.4.11のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.11)）。これで、スクリプトは完全なレポートと途中までのレポートを区別できます。ここでは、以前の修正も重要です。2026年9月24日リリースのバージョン1.2.10以降、回答のストリーミング中に失敗した`-p`の実行は終了コード3で終了します。成功したように見えることはなくなりました（[1.2.10のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.2.10)）。

![GitHub上のAntigravity CLI 1.4.11のリリースノートのスクリーンショット。青緑色の枠で、output-format jsonおよびstream-jsonの結果にfinished_reasonフィールドを追加する項目が囲まれています。この項目によると、print_timeoutはprint-timeoutの期限切れによって回答が途中までになったこと、interruptedは実行がキャンセルされたこと、errorはそれ以外の失敗を意味し、ターンが正常に終了した場合はこのフィールドは含まれません。その上の項目には、PreInvocationフックがerrorMessage付きでterminateを返せること、これによっても-pの実行が0以外のコードで終了することが記載されています。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/img-release-1411.webp)

*1.4.11のリリースノート：finished_reasonにより、タイマースクリプトで完全なレポートと部分的なレポートを区別できます。* [GitHubのリリースノート、2026年11月3日撮影のスクリーンショット](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.11)

## 24時間分のコスト：APIは呼び出しごとの料金、モデルはクォータ消費量またはトークン使用量

まずデータ側から見ていきます。このガイドで使う5つの機能（`youtube/comments`、`youtube/video`、`bilibili/comments`、`bilibili/danmaku`、`bilibili/video`）は、2026年11月3日時点のカタログでいずれも1回の呼び出しあたり$0.000556です。カタログは`GET https://api.everyinfra.com/api/v1/social/catalog`で公開されており、APIキーは不要です。実行する当日に必ずカタログで料金を確認してください。巡回1回につき主要な呼び出しを3回行い、合計$0.001668かかります。内訳は、YouTubeの最新コメント100件、Bilibiliの人気コメント、Bilibiliの弾幕です。

4回ごとに、コメント数の増加を追跡するため、動画ごとに詳細取得の呼び出しを1回追加します。失敗した呼び出しは無料です。上記の422のリクエストにも費用はかかっていません。リクエストIDで請求を照合する方法は、[失敗時の返金の照合](https://everyinfra.com/ja/build/failure-refund-reconciliation)をご覧ください。

| 間隔 | 24時間の巡回回数 | API呼び出し数 | API料金（定価） | モデル側（Gemini 3.8 FlashのAPI料金に基づく概算） |
| --- | --- | --- | --- | --- |
| 1時間ごと | 24 | 84 | $0.047 | $0.205 |
| 2時間ごと | 12 | 42 | $0.023 | $0.103 |
| 3時間ごと | 8 | 28 | $0.016 | $0.069 |
| 4時間ごと | 6 | 22 | $0.012 | $0.051 |
| 6時間ごと | 4 | 14 | $0.008 | $0.034 |

モデルの列は概算です。ご自身の数値で計算できるよう、算出方法を示します。Antigravityのヘッドレスモードのドキュメントでは、1行のプロンプトでJSON出力の`input_tokens`が10,415になっています。そのうち`cache_read_tokens`が8,113、`output_tokens`が657です（[Headless mode, "JSON output"](https://antigravity.google/docs/cli/headless)）。つまり、`agy -p`を呼び出すたびに、システムプロンプトとツール定義で約1万トークンが送られます。

巡回1回ごとにこの固定入力があり、そのうち8,113トークンにはキャッシュ料金が適用されると仮定します。これに約805のデータトークンを加えます。内訳は、2回目の巡回で取得したYouTubeの新着コメント9件分の505トークンと、指示文とBilibiliの行データ分の約300トークンです。出力トークンは1,500とします。

料金は、2026年10月30日に更新されたGemini APIの料金ページに基づきます。Gemini 3.8 Flashは、入力100万トークンあたり$0.75、出力100万トークンあたり$3.75、キャッシュ済み100万トークンあたり$0.075です。このページによると、これらの料金は2026年12月31日まで有効で、2027年1月1日から2倍になります（[Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing)）。データトークンはバイト数を4で割って概算しています。Geminiのドキュメントでは、1トークンは約4文字とされています（[Gemini API, "Understand and count tokens"](https://ai.google.dev/gemini-api/docs/tokens)）。この概算は英語のコメントには当てはまりますが、中国語のテキストでは少なめに出ます。

![巡回間隔別の24時間あたりの巡回コストを示す集合棒グラフ。1時間ごと：API料金は$0.047、API料金換算のモデル利用料は約$0.205。2時間ごと：$0.023と$0.103。3時間ごと：$0.016と$0.069。4時間ごと：$0.012と$0.051。6時間ごと：$0.008と$0.034。API料金はティール色、モデル利用料の概算はグレーです。モデル利用料はAPI料金の4倍強です。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/fig-08-cost-24h-bar.ja.light.svg)

![ラウンドごとにモデルに送るトークン数の横棒グラフ。固定のヘッドレス入力は10,415（濃色）。100件の生のJSONは約8,075。5つのフィールドだけを残すと約6,153。フラグ付きの34件だけなら約2,648。新規コメント9件だけのラウンド2は約505（ティール色）。データのトークン数は、バイト数を4で割って推定しています。](https://everyinfra.com/articles/antigravity-cli-video-comment-monitoring/fig-07-tokens-bar.ja.light.svg)

2つのグラフを合わせて見てください。このジョブのコストはデータにはかからず、主な要因はコメントのトークンでもありません。コストの大半は、毎回`agy`を起動する固定コストです。そのため、次の順で節約してください。

まず、間隔を延ばします。2時間に1回の巡回なら、1時間に1回の半分のコストで済みます。次に、スキルの判断ツリーにあるとおり、新着コメントがない回ではモデルを呼び出さないようにします。そのうえで、コメントごとのフィールドを絞り込みます。

また、`--model`で巡回をFlash系のモデルに固定してください。モデルのドキュメントによると、無料ユーザーとPlusユーザーが使えるのはGemini 3.8 Flashのみで、Gemini 4 ArgonにはProまたはUltraが必要です（[Models docs](https://antigravity.google/docs/models)）。デフォルトのモデルがArgonの場合、同じ巡回でもコストが大幅に増えます。同じページによると、Gemini 4 Argonは2026年11月30日まで、入力100万トークンあたり$2、出力100万トークンあたり$10です。12月1日から料金は2倍になります。

個人ユーザーはこのトークン料金を直接支払うわけではなく、プランのクォータを消費します。Antigravityのプランのドキュメントには3つのティアが記載されています。無料ティアには週ごとのクォータがあります。

Proのクォータは5時間ごとにリセットされ、週ごとの上限もあります。Ultraのクォータはさらに多くなります。自前のキーは利用できません（[Plans docs](https://antigravity.google/docs/plans)）。2026年5月19日のプランに関するブログ記事によると、Proは月額$20です。

この記事では、$100のUltraティアが追加され、従来の$250のUltraが$200に値下げされています。FlashとProのモデルは1つのプールを共有し、API料金で消費されます。$100と$200のティアでは、Proの5倍と20倍のクォータが付与されます（[Antigravity blog, May 19, 2026](https://antigravity.google/blog/changes-to-antigravity-plans)）。モデルの列は「この巡回がドル換算でどれだけのクォータを消費するか」として読んでください。

YouTubeには無料の方法もあります。`commentThreads.list`は1回の呼び出しで1クォータユニットを消費し、デフォルトのクォータは1日あたり10,000ユニットです（[commentThreads.list](https://developers.google.com/youtube/v3/docs/commentThreads/list)、[クォータ計算ツール、2026-10-15更新](https://developers.google.com/youtube/v3/determine_quota_cost)）。公開動画のコメントはAPIキーで読み取れます。保留中のコメントやスパムの可能性があるコメントだけは、動画の所有者によるOAuth認証が必要です。

その代わり、Google Cloudプロジェクトが必要になり、クォータとページング処理を自分で管理しなければならず、Bilibiliには対応できません。2つの連携を作らずに両方のプラットフォームを1つのレポートにまとめたい場合は、呼び出しごとの従量課金のほうがシンプルです。一覧は[料金](https://everyinfra.com/ja/pricing)をご覧ください。

## Antigravity CLIとGemini CLIの比較：移行時に変更すべき点

Gemini CLIで同様のタイマースクリプトを動かしていた場合、変更点のほとんどは設定ファイルの場所とキー名です。ワークスペースのルールファイルはそのまま使えます。公式のインポートコマンド`agy plugin import gemini`は、既存の拡張機能をプラグインに変換し、そのMCP定義を`mcp_config.json`に書き込みます（[移行ガイド「Converting extensions to plugins」](https://antigravity.google/docs/cli/gcli-migration/)）。また、`agy`の初回実行時に古い設定が検出され、チェックリストが表示されます。

ただし、`~/.gemini/settings.json`に直接書いたMCPサーバーは、引き続き確認が必要です。信頼設定も確認してください。以下の表は、両方の公式ドキュメントを1行ずつ比較して作成しました。

| 項目 | Gemini CLI | Antigravity CLI | 対処法 |
| --- | --- | --- | --- |
| コマンド | `gemini -p` | `agy -p`（エイリアス：`--print`、`--prompt`） | スクリプト内のコマンドとフルパスを変更 |
| MCP設定の場所 | `~/.gemini/settings.json`の`mcpServers` | `~/.gemini/config/mcp_config.json`または`.agents/mcp_config.json` | 新しいファイルに移動 |
| リモートアドレスのキー | `url`（SSE）または`httpUrl`（Streamable HTTP） | `serverUrl`（`url`も可） | キー名を変更 |
| ヘッダー内の環境変数 | `$VAR`と`${VAR}`に対応 | ドキュメントに記載なし | 非対応とみなし、キーは保護されたファイルに保存 |
| 確認のスキップ | サーバー単位の`trust: true` | `permissions.allow`の`mcp(server/tool)`ルール | ツールごとに許可 |
| ツールの許可リスト | `includeTools` / `excludeTools` | `enabledTools` / `disabledTools` | キー名を変更 |
| タイムアウト | `timeout`（ms単位、デフォルトは600,000） | `timeoutSeconds`（デフォルトは呼び出しごとに180、接続時は300） | 単位を変更 |
| プロジェクトのスキルフォルダ | `.gemini/skills/` | `.agents/skills/` | フォルダを移動 |
| ルールファイル | `GEMINI.md`、`AGENTS.md` | 同上 | 変更なし |
| ログイン | 有料のGemini APIキーを使用可能 | 個人はキーチェーン内のGoogleアカウント、企業はGoogle Cloudプロジェクト。独自キーの持ち込みは不可 | サーバーで無人実行する前にログイン方法を確認 |

スケジュール実行するジョブでは、最後の行が最も重要です。プランのドキュメントには、クォータを増やすために独自のキーやエンドポイントを持ち込むことはできないと書かれています（[プランのドキュメント](https://antigravity.google/docs/plans)）。GitHubのディスカッションでは、新しいCLIがAI StudioのAPIキーを受け付けなくなったと開発者が不満を述べています（[ディスカッション#27274](https://github.com/google-gemini/gemini-cli/discussions/27274)）。5月の発表では、Gemini CLIは引き続き有料のAPIキーとエンタープライズライセンスで使えるとも説明されています。巡回をヘッドレスのクラウドサーバーで実行していて、APIキーしかない場合もあるでしょう。その場合は、Gemini CLIを使い続けるのが手軽かもしれません。サーバーでの無人ログインがAntigravityのドキュメントで説明されてから移行してください。

公式のクォータ比較はありません。あるRedditユーザーは、Antigravity CLIのほうがクォータの消費が速いと報告しています。末尾のFAQ「どちらがクォータの消費が少ないか」を参照してください。

ただし、これは公式ドキュメントの数値と合致します。ヘッドレスモードのJSONの例では、1行のプロンプトでもすでに入力トークンが10,415あり、そのうち8,113がキャッシュされています（[ヘッドレスモードのJSONの例](https://antigravity.google/docs/cli/headless)）。1日12回実行すると、固定のオーバーヘッドだけで約120,000トークンになります。上記のコストのセクションにはこの分も含めています。

3つ目の違いはモデルへのアクセスです。2026年10月13日、Gemini 4 ArgonがProおよびUltraユーザー向けにAntigravityアプリとCLIで利用可能になりました（[Antigravityブログ](https://antigravity.google/blog/gemini-4-argon-in-google-antigravity)）。2026年10月14日のフォーラムのスレッドでは、ビジネス向けUltraの一部ユーザーで問題が報告されています。アプリでもCLIでもArgonを選択できなかったというものです（[Google AI開発者フォーラム](https://discuss.ai.google.dev/t/google-ai-ultra-for-business-users-cannot-access-gemini-4-argon-in-antigravity-app-or-cli/189416)）。巡回にはFlashで十分なので、影響はほとんどありません。それでも、移行後に一度`/model`を確認し、スクリプトでモデルを固定してください。

## 制限事項と注意点

データ面では4つの制限があります。1つ目に、プラットフォームが審査のために保留したコメントは表示されません。YouTubeヘルプによると、「基本」または「厳格」の設定では、コメントがYouTube Studioで最大60日間保留されます。承認されるまでは非表示のままです（[YouTubeヘルプ、コメントの設定](https://support.google.com/youtube/answer/9483359?hl=en)）。当社を含む第三者が見られるのは公開コメントのみです。保留中のコメントを取得したいチャンネル所有者は、OAuthと`moderationStatus=heldForReview`を使ってYouTube Data APIを呼び出す必要があります（[commentThreads.listのドキュメント](https://developers.google.com/youtube/v3/docs/commentThreads/list)）。

第2に、YouTubeの時刻は「2時間前」のような相対表示で、ラウンド1では4件に「(edited)」も付いていました。変換した時刻の精度は1時間単位で、古いコメントほど誤差が大きくなります。第3に、Bilibiliのコメントは人気コメントのみです。取得できたのは3件、次に10件で、総数はそれぞれ4,075件と4,566件でした。Bilibiliでは「新規コメントをすべて拾う」巡回はできないため、人気コメントと弾幕の変化を追跡してください。

4つ目に、YouTubeは2026年9月30日、Data APIの改訂履歴でコメントリソースに`snippet.imageUrl`を追加しました。GIFや画像のコメントは11月に全ユーザーに提供されます（[YouTube Data APIの改訂履歴](https://developers.google.com/youtube/data/revision_history)）。`youtube/comments`のフィールド一覧には画像のフィールドがありません。画像のみのコメントは空のテキストとして表示されます。

空のテキストをすべてノイズとして扱わないでください。TechCrunchは2026年9月23日、YouTubeがクリエイターごとに調整できるコメントのモデレーション機能をテストしているとも報じています（[TechCrunch](https://techcrunch.com/2026/09/23/youtube-is-making-comments-more-fun-and-fandom-more-lucrative-for-creators/)）。このテストが拡大するにつれて、表示されるコメントの範囲が変わると考えてください。

Antigravity側では、CLIはクローズドソースで、変更が頻繁です。このガイドは、2026年11月3日時点のドキュメントと1.4.12のリリースノートに基づいています。サイドカーのドキュメントにはデスクトップアプリしか記載されていません。ヘッドレスモードはキャッシュされたログインに依存しており、クラウドサーバーでの無人ログインについてはドキュメントで説明されていません。

個人プランのクォータは処理量に応じて消費され、プランのドキュメントにはクォータが変更される場合があると記載されています。当社のコストの数値は目安としてお考えください。実際の数値は`/usage`パネルで確認できます（[/usageのドキュメント](https://antigravity.google/docs/cli/commands/usage/)）。

最後に、利用面での制限があります。このスキルは読み取り専用で、何も投稿しません。EveryInfraは公開データを読み取りますが、お客様のアカウントから投稿することはできません。返信案は、人がYouTube StudioまたはBilibiliのクリエイターセンターから送信します。レポートにはコメントIDとテキストのみを残し、ユーザー名やプロフィールリンクは決して含めないでください。各プラットフォームの利用規約と、事業を行う地域のプライバシー法に従ってください。

ラベルも完璧ではありません。キーワードルールでは34件のコメントにフラグが付きましたが、クリエイターが時間を割く価値があったのはそのうち13件だけでした。また、「ぼかしがずれた」というコメントの半分を見逃しました。

モデルによるラベルはテストしていません。本番運用の前に、同じ100件のコメントを手作業でラベル付けし、モデルの出力と比較してください。「high」の項目は必ず人が読みます。モデルは100件のコメントを十数件程度に絞り込むだけです。

## 最近の変化

- 2026年10月30日（UTC）、Antigravity CLI 1.4.11で`--output-format json`に`finished_reason`が追加されました。同日、1.4.12で長い回答のMarkdownの表が崩れる問題が修正されました（[1.4.11](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.11)、[1.4.12](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.12)）。実務への影響：タイマースクリプトで「レポート完了」と「タイムアウトによる中断」を区別できるようになりました。1.4.11以降を使用してください。
- 2026年10月29日、バージョン1.4.10でストリーミング時のターミナル書き込みが約70%削減され、インタラクティブUIでは起動後に約100 MBのメモリが解放されるようになりました。リリースノートによると、メモリの変更はprintモードとヘッドレスモードには影響しません（[1.4.10](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.10)）。実務への影響：多くのリリースはインタラクティブUIを対象としています。タイマージョブをアップグレードする前に、各リリースノートでヘッドレス関連の項目を確認してください。
- 2026年10月24日、バージョン1.4.7でヘッドレスの`-p`実行時のCPU使用量が削減されました。また、エージェントに直接返されるオンデマンドMCPの結果のサイズ上限が4 KBから10 KBに引き上げられました（[1.4.7](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.7)）。実務への影響：YouTubeのコメント100件は約33 KBで、依然として10 KBを超えます。先にスクリプトで新しい行を圧縮してください。
- 2026年10月13日、Gemini 4 ArgonがGoogle AI ProおよびUltraユーザー向けにAntigravityアプリとCLIで利用可能になりました（[Antigravityブログ](https://antigravity.google/blog/gemini-4-argon-in-google-antigravity)）。同日、バージョン1.3.4で、ヘッドレスモードで不明な`--agent`名を指定するとコード1で終了するようになりました（[1.3.4](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.3.4)）。実務への影響：コメントの分類にはGemini 3.8 Flashで十分です。また、スクリプトのタイプミスが明確なエラーになるようになりました。
- Gemini APIの料金ページは2026年10月30日に更新されました。Gemini 3.8 Flashは12月31日まで、100万トークンあたり入力$0.75、出力$3.75と記載されています。2027年1月1日から料金は2倍になります。Gemini 4 Argonは11月30日まで入力$2、出力$10で、その後2倍になります（[Gemini APIの料金](https://ai.google.dev/gemini-api/docs/pricing)）。実務への影響：プランのクォータはAPI料金に基づいて消費されます（[5月のプランに関する記事](https://antigravity.google/blog/changes-to-antigravity-plans)）。新しいAPI料金で消費ペースが変わるかどうか、Googleは明らかにしていません。運用開始後1週間は`/usage`を確認してください。

## FAQ

**Antigravity CLIのMCP設定はどこに置きますか？**

グローバルファイルは`~/.gemini/config/mcp_config.json`です。移行ガイドにあるとおり、プロジェクトファイルはプロジェクトルートの`.agents/mcp_config.json`です。あるRedditユーザーは、設定が`~/.gemini/antigravity-cli/`以下に書き込まれていたため、`~/.gemini/config/`にコピーする必要がありました（[r/google_antigravityのスレッド](https://www.reddit.com/r/google_antigravity/comments/1uhgm44/antigravity_cli_mcp_servers_pain/)）。セットアップ後に最も手早く確認する方法は`/mcp`です。（Googleのサジェスト「antigravity cli mcp config」からの質問です。）

**Antigravity CLIとGemini CLI：クォータの消費が少ないのはどちら？**

公式な比較はありません。Redditで35件のアップボートを得た投稿では、Antigravity CLIでWebのコーディングを2時間行ったところ、Proアカウント2つ分のクォータを使い切ったとされています。Gemini CLIでは4時間で約10%しか消費しなかったとのことです（1人の報告であり、実測値ではありません）。わかっているのは、個人向けクォータはAPI料金換算で消費されること、FlashとProが同じクォータ枠を共有すること、ヘッドレス呼び出しごとに約10,000トークン分の固定入力が含まれることです。パトロールのような短いジョブでは、先にスクリプトで重複除去と事前フィルタリングを行ってください。モデルには新しいコメントを数百トークン分だけ送り、`--model`をFlashに固定してください。（サジェスト「antigravity cli vs gemini cli quota」と[r/GeminiCLIのスレッド](https://www.reddit.com/r/GeminiCLI/comments/1tlqd44/why_does_antigravity_cli_burn_through_quotas_so/)から寄せられた質問です。）

**Antigravity CLIのcronジョブに/scheduleを使えますか？**

1つの会話を開いている間であれば、数分ごとにチェックできます。無人で24時間実行し続けるジョブには使えません。`/schedule`は5フィールドのcron式と`MaxIterations`を受け付けますが、そのスケジュールは現在の会話に紐づいています。ドキュメントでは長時間のジョブにサイドカーを勧めていますが、サイドカーの対応先として記載されているのはデスクトップアプリだけです。CLIユーザーは、launchd、cron、systemdから`agy -p`を呼び出すほうが安定して実行できます。（サジェスト「antigravity cli cron job」と「antigravity cli cron expression」から寄せられた質問です。）

**Antigravity CLIのyoloモードとは何ですか？タイマー実行で使うべきですか？**

CLIリファレンスにyoloという名前のフラグはありません。最も近いのは、すべてのツールを許可する`--dangerously-skip-permissions`と、設定の`toolPermission: "always-proceed"`です。パトロールに必要なのは3つのMCPツールと1つのレポートフォルダだけです。代わりに、これらを名前で`permissions.allow`に列挙してください。（サジェスト「antigravity cli yolo mode」から寄せられた質問です。フラグは[CLIリファレンス](https://antigravity.google/docs/cli/reference)に記載されています。）

**YouTubeにはすでにコメント管理ツールがあります。なぜパトロールを作るのですか？**

YouTube Studioのモデレーションは、疑わしいコメントを保留します。Studioが判断するのは「何を非表示のままにすべきか」です。パトロールが判断するのは「どの公開コメントが質問や苦情に当たるか、どのコメントに返信が必要か」です。さらに、YouTubeとBilibiliを1つのレポートにまとめられ、競合の動画にも使えます。

自分のチャンネルでは両方を使ってください。保留中のコメントにはStudio、公開コメントのフォローアップにはパトロールを使います。（サジェスト「youtube comment moderation tool」から寄せられた質問です。設定は[YouTubeヘルプ](https://support.google.com/youtube/answer/9483359?hl=en)にあります。）

**Bilibiliのコメントスクレイパーで、動画のコメントをすべて取得できますか？**

`bilibili/comments`ではできません。カタログによると1回の呼び出しで最大10件で、10件を超える上限値を指定しても10件に制限されます。私たちのテストでは、動画に4,000件を超えるコメントがあったにもかかわらず、最初に3件、その後10件の人気コメントが返されました。また、ソートパラメータも受け付けません。

すべてのコメントを閲覧できるのは動画の投稿者だけで、それもクリエイターセンターか、承認済みのオープンプラットフォームアプリを通じた場合に限られます。パトロールでは、Bilibiliを人気コメントと弾幕コメントの2つのシグナルとして扱ってください。（「Bilibiliのコメントスクレイパー」を意味する中国語の検索サジェストから寄せられた質問です。）

## 始め方

1. [クイックスタート](https://everyinfra.com/ja/docs#quickstart)でAPIキーを取得します。まず無料のカタログを確認してください：`GET https://api.everyinfra.com/api/v1/social/catalog?platform=youtube`。`?platform=bilibili`についても同様に確認します。
1. [YouTube API](https://everyinfra.com/ja/youtube-api)と[Bilibili API](https://everyinfra.com/ja/bilibili-api)のページで、自分の動画URLを使ってコメントのエンドポイントを一度試します。
1. 上記のとおりに`.agents/mcp_config.json`と`permissions.allow`を作成します。`agy`を一度対話モードで実行し、`/mcp`と`/permissions`を確認します。
1. スキルを`.agents/skills/comment-patrol/`に配置します。`agy -p`を一度手動で実行します。`structured_output`がスキーマと一致していること、stderrに拒否されたツールが表示されていないことを確認します。
1. launchd、cron、systemdのいずれかに実行を任せます。初日は2時間ごとに実行し、1行目に「may have missed comments」が出ていないかを確認してから、間隔を調整します。

同じ方法は他のAIツールでも使えます。Claude Codeでのスケジュール実行については、[Claude CodeのスケジュールタスクでRedditの不満を収集する](https://everyinfra.com/ja/blog/claude-code-scheduled-tasks-reddit-pain-points)をご覧ください。GeminiをリモートのMCPサーバーに接続する別の方法については、[Gemini Deep Research APIとリモートMCP](https://everyinfra.com/ja/blog/gemini-deep-research-api-mcp-social-data)をご覧ください。コメントのクリーニングとラベル付けについては、[DeepSeek V4とClaude Codeによるコメントのクリーニング](https://everyinfra.com/ja/blog/deepseek-v4-claude-code-comment-cleaning)をご覧ください。

本ガイドで参照した外部情報源は、すべて2026年11月3日（SGT）に確認しました。参考文献の日付は、各ページに表示されている公開日または更新日です。

## 参考文献

1. [Google Developers Blog：Gemini CLIからAntigravity CLIへの移行](https://developers.googleblog.com/an-important-update-transitioning-gemini-cli-to-antigravity-cli/)、2026年5月19日。
1. [gemini-cliのGitHubディスカッション#27274](https://github.com/google-gemini/gemini-cli/discussions/27274)。
1. [Antigravity CLI移行ガイド](https://antigravity.google/docs/cli/gcli-migration/)。
1. [Antigravity CLIリファレンス](https://antigravity.google/docs/cli/reference)。
1. [Antigravity CLIのヘッドレスモード](https://antigravity.google/docs/cli/headless)。
1. [Antigravityの/scheduleコマンド](https://antigravity.google/docs/commands/schedule)。
1. [Antigravityのサイドカー](https://antigravity.google/docs/sidecars)。
1. [Antigravity MCPドキュメント（CLIタブ）](https://antigravity.google/docs/mcp?tab=cli)。
1. [Antigravityのエージェントスキルのドキュメント（CLIタブ）](https://antigravity.google/docs/skills?tab=cli)。
1. [Antigravity権限ドキュメント（CLIタブ）](https://antigravity.google/docs/permissions?tab=cli)。
1. [Antigravityのプランのドキュメント](https://antigravity.google/docs/plans)。
1. [Antigravityのモデルのドキュメント](https://antigravity.google/docs/models)。
1. [Antigravity CLIのモード](https://antigravity.google/docs/cli/modes)。
1. [Antigravity CLIの/usageコマンド](https://antigravity.google/docs/cli/commands/usage/)。
1. [Antigravity変更履歴](https://antigravity.google/docs/changelog)。
1. [google-antigravity/antigravity-cliリポジトリ](https://github.com/google-antigravity/antigravity-cli)。
1. [Antigravity CLI 1.4.12のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.12)、2026年10月30日（UTC）。
1. [Antigravity CLI 1.4.11のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.11)、2026年10月30日（UTC）。
1. [Antigravity CLI 1.4.10のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.10)、2026年10月29日。
1. [Antigravity CLI 1.4.7のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.4.7)、2026年10月24日。
1. [Antigravity CLI 1.3.4のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.3.4)、2026年10月13日。
1. [Antigravity CLI 1.2.10のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.2.10)、2026年9月24日。
1. [Antigravity CLI 1.2.9のリリースノート](https://github.com/google-antigravity/antigravity-cli/releases/tag/1.2.9)、2026年9月23日。
1. [Gemini CLIのMCPドキュメント](https://github.com/google-gemini/gemini-cli/blob/main/docs/tools/mcp-server.md)。
1. [Antigravityブログ：プランの変更](https://antigravity.google/blog/changes-to-antigravity-plans)、2026年5月19日。
1. [Antigravityブログ：Gemini 4 Argon](https://antigravity.google/blog/gemini-4-argon-in-google-antigravity)、2026年10月13日。
1. [Antigravityブログ：Gemini 3.8 Flash](https://antigravity.google/blog/gemini-3-8-flash-in-google-antigravity)。
1. [Googleブログ：Gemini 4 Argonの発表](https://blog.google/innovation-and-ai/models-and-research/gemini-models/gemini-4-argon/)、2026年9月30日。
1. [Gemini APIの料金](https://ai.google.dev/gemini-api/docs/pricing)。
1. [Gemini API：トークンの理解とカウント](https://ai.google.dev/gemini-api/docs/tokens)。
1. [Google AI開発者フォーラム：ビジネス向けUltraユーザーがAntigravityでGemini 4 Argonを利用できない問題](https://discuss.ai.google.dev/t/google-ai-ultra-for-business-users-cannot-access-gemini-4-argon-in-antigravity-app-or-cli/189416)、2026年10月14日。
1. [MCP仕様2026-07-28：トランスポート](https://modelcontextprotocol.io/specification/2026-07-28/basic/transports)。
1. [Agent Skillsオープン標準](https://agentskills.io/home)。
1. [YouTube Data APIのcommentThreads.list](https://developers.google.com/youtube/v3/docs/commentThreads/list)。
1. [YouTube Data APIのクォータ計算ツール](https://developers.google.com/youtube/v3/determine_quota_cost)。
1. [YouTube Data APIの改訂履歴](https://developers.google.com/youtube/data/revision_history)。
1. [YouTubeヘルプ：コメントの設定](https://support.google.com/youtube/answer/9483359?hl=en)。
1. [TechCrunch：YouTubeのコメント機能](https://techcrunch.com/2026/09/23/youtube-is-making-comments-more-fun-and-fandom-more-lucrative-for-creators/)、2026年9月23日。
1. [Bilibiliオープンプラットフォーム](https://open.bilibili.com/)。
1. [Apple開発者ドキュメント：Scheduling Timed Jobs](https://developer.apple.com/library/archive/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/ScheduledJobs.html)。
1. [crontab(5)マニュアル](https://man7.org/linux/man-pages/man5/crontab.5.html)。
1. [systemd.timerマニュアル](https://www.freedesktop.org/software/systemd/man/latest/systemd.timer.html)。
1. [r/google_antigravityの投稿「Antigravity CLI MCP Servers Pain」](https://www.reddit.com/r/google_antigravity/comments/1uhgm44/antigravity_cli_mcp_servers_pain/)。
1. [r/google_antigravity：MCP設定での環境変数](https://www.reddit.com/r/google_antigravity/comments/1s8i261/i_finally_looked_inside_my_antigravity_mcp_config/)。
1. [r/GeminiCLI：Antigravity CLIがクォータを大量に消費する理由](https://www.reddit.com/r/GeminiCLI/comments/1tlqd44/why_does_antigravity_cli_burn_through_quotas_so/)。
1. [The New Stack：Gemini CLIの後継](https://thenewstack.io/gemini-cli-antigravity-replacement/)、2026年6月20日。
1. [YouTubeのサンプル動画](https://www.youtube.com/watch?v=g9IOdTisQyM)。
1. [Bilibiliのサンプル動画](https://www.bilibili.com/video/BV1i24P6rEmt)。
1. [EveryInfraカタログ：YouTube](https://api.everyinfra.com/api/v1/social/catalog?platform=youtube)。
1. [EveryInfraカタログ：Bilibili](https://api.everyinfra.com/api/v1/social/catalog?platform=bilibili)。

---

:::message
この記事は EveryInfra ブログの記事の転載です。原文：[Antigravity CLIでコメントを監視する：セットアップと実データ](https://everyinfra.com/ja/blog/antigravity-cli-video-comment-monitoring)

EveryInfra は AI エージェントと開発者向けの従量課金データ API です：[everyinfra.com/ja](https://everyinfra.com/ja)
:::
