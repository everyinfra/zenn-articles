---
title: "Claude Coworkのスケジュールタスクで、毎週月曜に競合の価格とレビューの一覧表を自動作成する"
emoji: "📊"
type: "tech"
topics: ["datascience", "claude", "api", "ai", "webscraping"]
published: true
---

:::message
この記事は [EveryInfra ブログ](https://everyinfra.com/ja/blog/claude-cowork-scheduled-tasks-competitor-price-reviews) からの転載です（原文が正本です）。
:::

AmazonとWalmartで小型家電の製品ラインを担当しているなら、競合価格を手作業で確認しているかもしれません。6つの商品ページを開き、数値をシートに写して、先週と比べる作業です。この記事では、その1時間をClaude Coworkのスケジュールタスクに置き換えます。タスクは毎週月曜にクラウドで実行され、カスタムコネクタ経由でデータAPIを呼び出し、変化をまとめた表をGoogle Sheetsに書き込みます。データ側は2026年11月3日に実際の呼び出しでテストしました。Cowork側は実行していないため、ここで紹介するCoworkの手順はすべて設定例であり、Anthropicの情報源へのリンクを添えています。

最も役立った発見はWalmartで得られました。同じWalmartの商品（Ninja Crispiのガラス製エアフライヤー）で、2つの検索キーワードに対して販売者と価格がそれぞれ異なっていました。「ninja air fryer 4 quart」では販売者がPatent Paidで価格は$135、「ninja crispi glass air fryer」では販売者がMercshopで価格は$134.50でした。約1時間後、シンガポール時間の01:13:54と01:14:03に、9秒間隔で両方の検索を再実行しました。それでも、キーワードごとに前回と同じ販売者と価格が表示されました。

つまり、違いは時刻ではなくキーワードによるものです。週次タスクでキーワードが変わると、単に販売者が違うだけなのに、表に$0.50の値下げとして表示される可能性があります。そのため、各商品には固定のキーワードを1つ割り当て、各行には価格と並べて販売者も保存します。

> **要点**
>
> - 2ストアで競合3社を対象にした週次実行は10回の呼び出しで、カタログ価格では$0.014737です。
> - 1つのWalmart商品IDで、2つのキーワードに対して2つの販売者が表示されました。各行のキーはストア、商品ID、販売者の組み合わせにします。
> - 1つの`walmart/reviews`レスポンスに、AF100WMの総レビュー数として12,562、12,571、12,573の3つの値が含まれていました。
> - Walmartの星1〜2のレビューのうち、12件中5件、20件中6件は製品ではなく配送に関するものでした。
> - 新しいCoworkのスケジュールタスクは、実行ごとに削除されるクラウドサンドボックスで動作します。先週のデータはGoogle Sheetsに保存しておきます。
> - Cowork側はテストしていませんが、EveryInfraの数値はすべて実際の呼び出しによるものです。

![イラスト。浅いトレイが2つ並んでいます。左側の明るいトレイは先週、右側のトレイは今週を表します。各トレイには小さな箱が5つずつ入っており、ふたには円、三角形、四角形、六角形、星のいずれかが描かれています。同じ形の箱同士は破線でつながっています。三角形のペアはティール色のリボンで結ばれ、今週の箱には追加のタグが付いているので、変化がすぐにわかります。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/img-explain.webp)

*2週間分を比較する方法：まず同じキーで行を照合し、次に変化した行を探します。* [本記事のために作成したイラストであり、実データではありません](https://everyinfra.com/ja/blog/claude-cowork-scheduled-tasks-competitor-price-reviews)

## タスク：毎週月曜の競合価格・レビュー表

以下の数値はすべてタスクの定義に左右されるため、まずタスクを定義します。入力は少数の競合をまとめたリストです。各競合について、Amazonの商品を1つ（ASINで識別）、Walmartの商品を1つ（アイテムIDで識別）指定します。例として、4クォートのエアフライヤー3機種を使います。Ninjaのベーシックモデル、Ninja Crispi 4-in-1のガラス製モデル、Instant Vortex miniです。エアフライヤーを選んだのは、どちらのストアでも数多く販売され、価格が頻繁に変動するためです。

タイミングも重要です。Walmartは、2026年11月12日から12月2日まで開催する3つのホリデーセールイベントを発表しました。イベント期間は計14日間で、2025年の10日間より長くなっています（[Walmartプレスリリース](https://corporate.walmart.com/news/2026/10/walmart-unwraps-three-major-holiday-shopping-events)、[Retail Dive](https://www.retaildive.com/news/walmart-longer-black-friday-cyber-monday-sales-events/872747/)）。これからの5週間は、ブランドマネージャーにとって週次の価格チェックが最も必要な時期です。

出力は1つの表です。ストアごと、競合ごとに1行あります。最初の列は今週の価格、定価、割引、販売者、評価、評価数またはレビュー数です。次の列には、先週からの価格の変化とレビュー数の変化を示します。さらに2列で、新しい星1〜2のレビューの数と、その短いメモを示します。最後の列は1行の判断で、「すぐ対応」「様子見」「無視」のいずれかです。

このタスクがスケジュールタスクに向いている理由は3つあります。1つ目は、決まった曜日に繰り返すことです。2つ目は、データを読み取るだけであることです。価格の変更、注文、メッセージ送信は行わないため、エラーが起きてもセルの値が間違うだけで、金銭的な損失にはなりません。3つ目は、前回の実行結果と比較する必要があることです。Coworkで最も注意が必要なのはこの3つ目の点で、後のセクションで詳しく扱います。

セラー向けツールを使わない理由は、表示されるのが競合ではなく自社の出品だからです。Walmart Seller Centerの評価ページには、自社の出品を購入した顧客からのフィードバックしか表示されません。他の販売者の出品に対するレビューは表示されません（[Walmart Marketplace Learn](https://marketplacelearn.walmart.com/guides/Growth%20opportunities/Reviews/Marketplace-reviews:-overview)）。

Walmartの価格インサイトダッシュボードは自社の価格をBuy Box獲得者の価格と比較しますが、これも対象は自社の商品だけです（[Walmart Buy Boxガイド](https://marketplacelearn.walmart.com/guides/Item%20setup/Item%20content,%20imagery,%20and%20media/Product-detail-page:-the-buy-box)）。競合の情報を見るには、公開されている商品ページを読み取ります。それを行うのが[AmazonデータAPI](https://everyinfra.com/ja/amazon-api)と[WalmartデータAPI](https://everyinfra.com/ja/walmart-api)です。

競合リストはモデルではなく、人が決めるべきです。設定シートに一度だけ書いておきます。各競合について、AmazonのASIN（商品URLの`/dp/`の後に続く10文字）を記録します。WalmartのアイテムID（商品URLの`/ip/`の後にある最後の数字）も記録します。たとえば、ピンクのCrispiは`https://www.walmart.com/ip/20370717802`です。さらに商品ごとに検索キーワードを1つ追加し、タスクが検索結果の上位20件からそのIDを見つけられるようにします。

毎週モデルに「同じ商品」を選び直させてはいけません。検索順位は毎週変わります。今週6位の商品が、翌週には25位に落ちることもあります。モデルが新しい「一致商品」を選ぶと、表は異なる2つの型番を比較することになり、変化の列は意味をなさなくなります。固定のIDリストを使えば、これを防げます。

![フローチャート。ステップ1：月曜09:10に、Coworkのスケジュールタスクが新しいクラウドセッションで起動します。ステップ2：EveryInfraのリモートMCPサーバーへのカスタムコネクタを使います。ステップ3：amazon/searchを1回、walmart/searchを3回、product_detailを3回、reviewsを3回、計10回の呼び出しを行います。費用は週$0.014737です。ステップ4：Google Sheetsのスナップショット用シートに未加工の行データを追記します。ステップ5：ストア、アイテムID、販売者をキーに先週の行と照合します。ステップ6：月曜の朝に開く変更点シートを書き出します。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/fig-02-weekly-flow.ja.light.svg)

## Claude Coworkのスケジュールタスクで現在できること

Anthropicは2026年2月25日、Coworkにスケジュールタスクを追加しました。リリースノートには、Coworkで定期タスクやオンデマンドタスクを作成できると書かれていました（[Claudeリリースノート](https://support.claude.com/en/articles/12138966-release-notes)）。当初、タスクは自分のコンピューター上で実行されていました。デスクトップアプリを閉じていたり、コンピューターがスリープしていたりすると、何も実行されませんでした。

これが最も多かった不満です。あるRedditユーザーは、マシンが起動している間しか動かないスケジュールでは意味がないと書いています（[Redditスレッド](https://www.reddit.com/r/ClaudeAI/comments/1u6ab7d/scheduled_tasks_routines_whats_actually_holding/)）。この状況は10月に変わりました。

現在ヘルプセンターには、スケジュールタスクはすべての有料プラン（Pro、Max、Ultra、Team、Enterprise）で利用できると記載されています。タスクはリモートで実行されるため、コンピューターがスリープ中でも、デスクトップアプリを閉じていても、スケジュールどおりに実行されます。各実行はそれぞれ独立したCoworkセッションです。実行時には、Coworkで設定したコネクタ、スキル、プラグインを使えます（[Schedule recurring tasks in Claude Cowork](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-claude-cowork)）。

別のヘルプ記事には、変更日が記載されています。2026年10月6日以降、Pro、Max、Ultraの新しいCoworkタスクはクラウドで実行されます。「Only on your computer」設定は削除され、元に戻すことはできません。10月6日より前から自分のコンピューターで実行されていたスケジュールタスクは、引き続きそこで実行されます。新しいスケジュールタスクはクラウドで実行されます（[Use Claude Cowork on web, desktop, and mobile](https://support.claude.com/en/articles/15520349-use-claude-cowork-on-web-desktop-and-mobile)）。

![Claudeヘルプセンターのページ「Schedule recurring tasks in Claude Cowork」のスクリーンショット。ティール色の枠で、スケジュールタスクがリモートで実行されることを示す箇所が囲まれています。コンピューターがスリープ中でも、デスクトップアプリを閉じていても、スケジュールどおりに実行されます。今後の実行と過去の実行は、左サイドバーの「Scheduled」から確認できます。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/img-doc-sched.webp)

*ヘルプセンターの記載：スケジュールタスクはリモートで実行され、コンピューターのスリープ中も動作します（2026年11月3日確認）。* [Claudeヘルプセンターのスクリーンショット](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-claude-cowork)

手動でタスクを設定する場合、フォームには6つのフィールドがあります。名前、プロンプト、承認モード、頻度、任意のモデル、任意の作業フォルダーです。頻度の選択肢は、毎時、毎日、毎週、平日、手動のみです。同じ記事には、見落としやすい短い注記があります。

スケジュールタスクがローカルのファイルやアプリを必要とする場合、そのタスクは自分のコンピューター上でしか実行されません（[同じヘルプ記事](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-claude-cowork)）。そのため、価格表はローカルファイルを必要としない設計にします。データの読み込みも、結果の書き込みもコネクタ経由で行います。そうすれば、ノートPCがスリープしていても、実際にタスクが実行されます。

操作の入口も、人によって表示が異なる場合があります。2026年9月16日、AnthropicはPro、Max、Ultra向けにCoworkとチャットの統合を開始しました。メッセージボックスに「Chat」と「Cowork」の選択肢が表示されなくなっていれば、新しい統合版に切り替わっています。その場合は、どの会話でも「毎週月曜の午前9時にXをして」のような依頼を入力できます。Claudeが名前、スケジュール、指示を表示するので、「Schedule」をクリックして確定します（[Claude Cowork and chat are one Claude](https://support.claude.com/en/articles/16761823-claude-cowork-and-chat-are-one-claude)）。

| 項目 | ヘルプセンターの記載（2026年11月3日確認） | 価格表への影響 |
| --- | --- | --- |
| プラン | Pro、Max、Ultra、Team、Enterprise | Freeプランにはスケジュールタスクはありません。月額$20のProで十分です（[料金](https://claude.com/pricing)） |
| 実行場所 | リモート。10月6日以降の新しいタスクはクラウドで実行 | 月曜にコンピューターを起動する必要はありません |
| 頻度 | 毎時、毎日、毎週、平日、手動 | 毎週・月曜の朝を選択 |
| ツール | コネクタ、スキル、プラグイン | カスタムコネクタからデータAPIにアクセスできます |
| ローカルファイル | ローカルのファイルやアプリが必要なタスクは、ローカルでのみ実行 | スナップショットをローカルフォルダーに保存しない |
| 操作 | 「Scheduled」サイドバー：表示、一時停止、再開、削除、今すぐ実行 | 一度「今すぐ実行」を使い、承認プロンプトに応答しておく |

## CoworkのスケジュールタスクとClaude Codeのスケジュールタスクの違い

11月1日に、[毎晩Redditのペインポイントを読み取るClaude Codeのスケジュールタスク](https://everyinfra.com/ja/blog/claude-code-scheduled-tasks-reddit-pain-points)についてのガイドを公開しました。その記事では、Anthropicが以前「routines」と呼んでいた、Claude Codeのクラウドスケジュールタスクを使いました。どちらの製品も「スケジュールタスク」という名前を使っています。どちらもクラウドで実行され、どちらもサブスクリプションの利用枠を消費します。ただし、向いている作業は異なります。

Claude Codeのクラウドスケジュールタスクは、保存されたClaude Codeの構成です。プロンプト、1つ以上のリポジトリ、コネクタ一式で構成されます。スケジュール、API呼び出し、GitHubイベントのいずれかで起動できます。各実行は完全なClaude Codeセッションで、シェルコマンドの実行、コードの編集、プルリクエストの作成ができます（[Automate work with scheduled tasks](https://code.claude.com/docs/en/routines)）。

Coworkのスケジュールタスクにはリポジトリがありません。レポート、ブリーフィング資料、ファイル整理などのナレッジワーク向けです。出力はCoworkセッションとその中のファイルです。

|  | Coworkのスケジュールタスク | Claude Codeのクラウドスケジュールタスク |
| --- | --- | --- |
| トリガー | 頻度（毎時、毎日、毎週、平日）または手動 | スケジュール、API呼び出し、GitHubイベント（自由に組み合わせ可能） |
| 紐づくもの | コネクタ、スキル、プラグイン | リポジトリ、クラウド環境、コネクタ |
| カスタムスケジュール | ヘルプセンターには5つのプリセットのみ。cronに関する記載は見つかりませんでした | cron式（最短で1時間に1回） |
| 実行回数の上限 | 公開された数値は見つかりませんでした | アカウントあたり1時間に100回のスケジュール実行 |
| 先週のデータの保存先 | コネクタ経由のシートまたはドキュメント | リポジトリにコミットしたファイル |
| 向いている用途 | 週次の競合比較表、朝のブリーフィング、受信トレイの要約 | 毎晩のコードレビュー、バックログのトリアージ、プルリクエスト |

Claude Codeの列は、同じドキュメントページの使用量とトリガーのセクションに基づいています（[Claude Code cloud scheduled tasks](https://code.claude.com/docs/en/routines)）。Coworkの列については、スケジュールタスク、TeamとEnterprise、安全性、アーキテクチャの4つのヘルプ記事を読みました。いずれにも、スケジュールタスクの最大数や1時間あたりの実行回数の上限は記載されていません。あるRedditユーザーも同じ質問をしましたが、Anthropicから数値は示されませんでした（[Redditスレッド](https://www.reddit.com/r/ClaudeAI/comments/1u6ab7d/scheduled_tasks_routines_whats_actually_holding/)）。そのため、推測はせず「公開された数値は見つかりませんでした」と記載しています。

ブランドマネージャーにとって、選択は明快です。価格表にはリポジトリもコード変更も不要で、マーチャンダイジングチーム向けのシートです。Coworkのほうが適しています。

Claude Codeを選ぶのは、チームがすでにデータとスクリプトをGitで管理している場合に限りましょう。その場合、週ごとの履歴をコミットログに残せます。どちらの構成でも同じリモートMCPアドレスを使えるため、コネクタの設定は一方からもう一方へそのままコピーできます。

## EveryInfraをカスタムコネクタとして追加する

Coworkはカスタムコネクタを通じて外部データにアクセスします。ヘルプセンターによると、リモートMCPを使うカスタムコネクタはClaude、Cowork、Claude Desktopで動作します。Free、Pro、Max、Ultra、Team、Enterpriseの各プランで利用でき、Freeユーザーが追加できるのは1つだけです（[Get started with custom connectors using remote MCP](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)）。

個人プランの場合は、Customize > Connectorsを開きます。「+ Add」をクリックし、続けて「Add custom connector」をクリックします。名前とリモートMCPサーバーのURLを入力します。

設定で最初につまずきやすいのは認証です。EveryInfraのリモートMCPサーバーは、OAuthではなくAPIキーを使います。`initialize`と`tools/list`メソッドはキーなしで動作します。ツール呼び出しでは、`Authorization: Bearer`の後にAPIキーを続けたヘッダーが必要です（[EveryInfra MCPセットアップページ](https://everyinfra.com/ja/mcp-server)）。MCP仕様では認可は任意とされており、そのOAuthフローはサーバーが対応している場合にのみ適用されます（[MCP仕様：Authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)）。`Authorization: Bearer`ヘッダーの形式は、OAuthのベアラートークン標準に由来します（[RFC 6750](https://www.rfc-editor.org/rfc/rfc6750)）。

シンガポール時間11月3日00:47に、キーなしで`tools/call`を1回送信しました。サーバーは0.09秒で`unauthorized: no API key provided`を返し、続くテキストブロックには「Nothing was charged.」（課金は発生していません）と表示されました。

Anthropicのドキュメントには、APIキー方式のサーバーでの手順が記載されています。Authenticationで「No sign-in」を選択し、Request headersにキーを追加します。Claudeは値を暗号化し、再表示はせず、リクエストごとに送信します。Claudeは入力された値をそのまま送信し、プレフィックスは付けません。そのため、`Authorization`の値は`Bearer`という語、半角スペース、キーの順にする必要があります（[Add a connector that isn't in the directory](https://claude.com/docs/connectors/custom/add-unlisted)）。

```text
Configuration example (Customize > Connectors > + Add > Add custom connector).
Field names follow the Anthropic documentation. We did not run this on our machine.
Name:            EveryInfra
MCP server URL:  https://api.everyinfra.com/mcp
Authentication:  No sign-in
Request headers: Authorization = Bearer <your EveryInfra API key>
Advanced > Transport: keep the default (the URL does not end in /sse, so it uses Streamable HTTP)
```

2つ目の問題はネットワークです。ヘルプセンターには「Network requirements」というセクションがあります。カスタムコネクタを追加すると、ClaudeはユーザーのコンピューターではなくAnthropicのクラウドからMCPサーバーに接続します。これはclaude.ai、Claude Desktop、Cowork、モバイルアプリのいずれでも同じです。サーバーは、AnthropicのIPレンジから公開インターネット経由で到達できる必要があります。

社内ネットワーク上、VPNの内側、ファイアウォールの内側にあるサーバーには接続できません。自分のコンピューターからは到達できる場合でも同様です（[ヘルプセンター](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)）。Anthropicは、外部MCPツールへの呼び出しに使う送信元アドレス範囲として`160.79.104.0/21`を公開しています（[IP addresses](https://platform.claude.com/docs/en/api/ip-addresses)）。EveryInfraのアドレスは公開されているため、このルールの影響は受けません。ただし、IT部門が前段に社内ゲートウェイを置く場合は、この範囲を許可する必要があります。

![カスタムコネクタに関するClaudeヘルプセンターのページにある「Network requirements」セクションのスクリーンショット。ティール色の枠で、MCPサーバーはAnthropicのIPレンジから公開インターネット経由で到達できる必要がある、という記述が示されています。社内ネットワーク上、VPNの内側、ファイアウォールの内側にあるサーバーは、自分のコンピューターから到達できる場合でも接続できません。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/img-doc-connector.webp)

*コネクタの接続元はユーザーのコンピューターではなく、Anthropicのクラウドです（2026年11月3日閲覧）。* [Claudeヘルプセンターのスクリーンショット](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)

3つ目の問題は、TeamプランとEnterpriseプランでの認証情報の共有です。Teamプランでは、組織にカスタムコネクタを追加できるのはOwnerとPrimary Ownerのみです。Enterpriseでは、ライブラリ管理の権限を持つカスタムロールも追加できます。Ownerがコネクタを追加した後、各メンバーがCustomize > Connectorsを開いて「Connect」をクリックします。

ヘルプセンターでは、この設計について説明しています。コネクタがAPIキーなどの共有認証情報を使わない限り、Claudeがアクセスできるのは各メンバーがアクセスできる範囲だけです。TeamとEnterpriseでは、そのコネクタを使うすべてのメンバーが同じ認証情報を使います。そのため、その認証情報にはメンバーに必要なアクセス権だけを与えてください（[ヘルプセンター](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)）。

EveryInfraの場合、キーを共有すると、全メンバーのスケジュールタスクが同じアカウントに課金されます。Cowork専用のキーを別に1つ用意し、1分あたりの上限を低めに設定することをおすすめします。そうすれば、不適切なプロンプトがAPI呼び出しをループしても、そのキーの上限に達した時点で停止します。また、コネクタを追加した後は認証設定を編集できません。キーを変更するにはコネクタを削除して再度追加する必要があり、メンバーも接続し直す必要があります（[Anthropicのドキュメント](https://claude.com/docs/connectors/custom/add-unlisted)）。

Anthropicはリクエストヘッダー名も審査しています。承認されていないヘッダー名を使うと、保存時にエラーになります。EveryInfraが使うのは標準の`Authorization`ヘッダーだけです。以前はキーを`?key=` URLパラメータとして渡すこともできましたが、この方法は2026-11-04に停止し、`?key=`付きのURLは401を返します。Coworkではリクエストヘッダーだけを使ってください。

## ツールの承認：タスクを手動で1回実行

スケジュールタスクは誰も見ていないときに実行されます。そのため、結果が空になる原因として最も多いのが承認プロンプトです。安全に関する記事では、3つの承認モードが紹介されています。「Automatically approve」は各アクションの前に安全チェックを実行してから続行します。「Skip all approvals」はチェックを一切行いません。

「Manually approve」は各ステップで確認を求めます（[Use Claude Cowork safely](https://support.claude.com/en/articles/13364135-use-claude-cowork-safely)）。同じ記事では、要約のような低リスクのタスクから始めるよう勧めています。また、メッセージの送信や購入を行うタスクはスケジュールしないよう求めています。読み取り専用の競合比較表は、まさに推奨されている種類のタスクです。

EveryInfraの12個のMCPツールにはすべて、MCP仕様のツールアノテーションが付いています。11月3日に`tools/list`を確認しました。カタログツール`everyinfra_list_capabilities`とジョブ取得ツール`everyinfra_get_job`は`readOnlyHint: true`です。データツール`everyinfra_call_api`は費用が発生するため、`readOnlyHint: false`と`destructiveHint: true`が設定されています。

MCP仕様では、これらの値をヒントと呼んでいます。クライアントはこれを使って、先に人に確認するかどうかを判断できます。アノテーションのないツールは、破壊的な操作を行う可能性があるものとして扱われます（[MCP仕様：Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)、[ツールアノテーションに関するMCPブログ](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/)）。

これはTeamとEnterpriseで特に重要です。Teamプランのヘルプ記事では、「Allow 'Always allow' for connector tools」という組織設定が説明されています。この設定はデフォルトでオフです。オフの間、メンバーはCoworkの承認プロンプトで「Allow for all tasks」を選択できません。

書き込みを行うツールについては、以前に選んだ「always allow」も無効になります。プロンプトを省略できるのは、コネクタが読み取り専用としてマークしたツールだけです（[Use Claude Cowork on Team and Enterprise plans](https://support.claude.com/en/articles/13455879-use-claude-cowork-on-team-and-enterprise-plans)）。このルールにより、デフォルト設定のままでは`everyinfra_call_api`はタスクごとに承認を求めます。

当社にはTeamプランの組織がないため、スケジュール実行がこのステップで待機するのか、スキップするのかは検証できませんでした。安全に進めるには、Ownerにこの設定をオンにしてもらうか、タスクの承認モードを「Automatically approve」に設定してください。タスクを作成したら、「Run now」を1回クリックします。承認プロンプトが表示されたら、「Allow for all tasks」を選択します。

最近のアップデートにも承認に関わるものがあります。10月22日のデスクトップアプリv3.6556.1で、サーバー側で人による確認が必要とマークされたMCPツールの挙動が変わりました。これらのツールはアプリ内のカードでのみ確認を求めるようになり、通知にはAllowボタンが表示されません。スケジュールタスクは、これらの呼び出しを行わずに続行します（[Claude Desktop changelog](https://claude.com/docs/cowork/changelog)）。EveryInfraのツールはこのマーカーを使用しておらず、使っているのは4つの標準アノテーションだけです。

スケジュール実行時に承認プロンプトがどのように表示されるか、また実行が停止するかどうかは、Cowork上では検証していません。そのため、「Run now」を1回クリックして、このステップを自分の目で確認してください。

このルールが対象とするのはCoworkの組み込みWebフェッチで、MCPツールではありません。ただし、Anthropicがスケジュールタスクでできることをいまも週単位で調整していることがわかります。ツール名とパラメータを正確に指定したプロンプトのほうが、モデルにやり方を自分で探させるプロンプトよりも安定します。

## 実際の1往復：モデルが送るものと返ってくるもの

Coworkでは実行していません。ただし、モデルがコネクタ経由で送るのは、標準のMCP `tools/call`リクエストです。実際の形式を示すため、2026年11月3日00:25:46（シンガポール時間）に、同じリクエストをcurlで`https://api.everyinfra.com/mcp`に送信しました。テストキーを使い、「ninja crispi glass air fryer」で`walmart/search`を呼び出しました。

43.6秒後、サーバーはHTTP 200を返しました。結果には13,115文字のテキストブロックが1つあり、20件の商品が含まれていました。料金は$0.001389でした。

```text
// Request: the tools/call that the model sends through the connector
// (November 3, 2026, 00:25:46 Singapore time; real request; the key is in the header and not shown)
{"jsonrpc":"2.0","id":8,"method":"tools/call",
 "params":{"name":"everyinfra_call_api",
           "arguments":{"platform":"walmart","action":"search",
                        "params":{"keyword":"ninja crispi glass air fryer"}}}}

// Response: JSON inside result.content[0].text (result 2, unchanged)
{"item_id": "20370717802",
 "name": "Ninja Crispi 4-in-1 Portable Glass Air Fryer Cooking System FN101PK1, ...",
 "price": 134.5, "was_price": 179.99, "currency": "USD",
 "rating": 4.5, "rating_scale": 5, "review_count": 1877,
 "in_stock": true, "seller_name": "Mercshop", ...}
// Billing in the same response: {"charged": true, "charged_usd": 0.001389, ...}
```

同じセッションでは失敗したやり取りも1回あったため、起きたとおりに報告します。シンガポール時間00:24:06に、MCP経由でNinja AF101（ASIN B07FDJMC9Q）を対象に`amazon/reviews`を呼び出しました。88.8秒後、`isError: true`が返されました。理由は、この商品のレビューページにはログインが必要か、アクセスが制限されているというものでした。この呼び出しの料金は返金され、`charged_usd`は0で、レスポンスにはすぐに再試行しても改善する見込みは低いとも書かれていました。カタログにはもともと、`amazon/reviews`は商品ページに表示される注目のレビューのみを返し、通常は1商品あたり12件程度であると記載されています（[EveryInfraカタログ](https://api.everyinfra.com/api/v1/social/catalog)）。

そのため、Amazonの行ではレビュー本文を読み取りません。`amazon/search`の評価と評価数をレビューのシグナルとして使います。レビュー本文はWalmartから取得します。ご自身のカテゴリでAmazonのレビュー本文が重要な場合は、まず自社のASINで`amazon/reviews`を試してください。

![EveryInfra APIの実際のレスポンス2件のスクリーンショット。上半分は、00:16:52にキーワードninja air fryer 4 quartで実行したwalmart/searchへのREST呼び出しです。6番目の結果は商品20370717802で、価格135、販売者Patent Paidです。価格と販売者をティール色の枠で囲んでいます。下半分は、00:25:46にキーワードninja crispi glass air fryerで実行したwalmart/searchへのMCP呼び出しです。同じ商品IDで価格134.5、販売者Mercshopと表示され、同じく枠で囲んでいます。どちらの呼び出しも0.001389ドルでした。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/img-api-card.webp)

*同じWalmart商品IDに対する2つのレスポンス、2つの価格、2つの販売者。* [実際のEveryInfra APIレスポンスのスクリーンショット（2026年11月3日、シンガポール時間）](https://api.everyinfra.com/api/v1/social/catalog)

## テスト1：2つのストア間の価格差

最初のスナップショットは、シンガポール時間00:15:54から00:26:55まで実行しました。「ninja air fryer 4 quart」で`amazon/search`を1回呼び出したところ、36.8秒で20件の結果が返され、料金は$0.002236でした。`walmart/search`はNinja 4クォート、Ninja Crispi、Instant Vortex miniについて1回ずつ、計3回呼び出し、料金は各$0.001389でした。下の表の数値はすべて、これらのレスポンスからそのまま取ったものです（[Amazon検索API](https://everyinfra.com/ja/amazon-api/search)、[Walmart検索API](https://everyinfra.com/ja/walmart-api/search)）。

| 競合製品 | Amazonのモデル、価格 | Amazonの参考価格、割引率 | Amazonの評価数 | Walmartのモデル、価格 | Walmartの旧価格、割引率 | Walmartのレビュー数、販売者 | 差額（Amazon − Walmart） |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Ninja基本モデル | AF101、$119.99 | none | 59,532 | AF100WM、$59.00 | $89.00, 33.7% | 12,571、Walmart.com | $60.99（モデルが異なる） |
| Ninja Crispi | FN101ST、$149.00 | $179.99, 17.2% | 8,426（4バリエーションで共有） | FN101PK1、$134.50 | $179.99, 25.3% | 1,877、Mercshop | $14.50 |
| Instant Vortex mini | 4 Qt、$69.95 | $89.99, 22.3% | 36,519（3バリエーションで共有） | 4 Qt、$69.99 | $89.99, 22.2% | 108、Instant Pot | −$0.04 |

![グループ化された棒グラフ。3つの競合製品それぞれについて、Amazonの価格をグレーの棒、Walmartの価格をティール色の棒で示しています。Ninja基本モデル：Amazon 119.99ドル、Walmart 59.00ドル（モデルが異なる）。Ninja Crispi：Amazon 149.00ドル、Walmart 134.50ドル。Instant Vortex mini：Amazon 69.95ドル、Walmart 69.99ドル。2026年11月3日、シンガポール時間00:15〜00:27に実施した呼び出しで取得したデータです。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/fig-03-price-gap-bar.ja.light.svg)

1行目は注意して見てください。$60.99の差は大きく見えますが、AF101とAF100WMは別のモデルです。AF100WMのWalmartでの商品名には「3 Functions」とあり、AF101は「4-in-1」です。小型家電では、小売店専用モデルが一般的です。

ブランドとサイズで商品を照合すると、表には毎週、意味のない大きな差額が表示されます。競合製品はモデル番号で照合してください。モデルが一致しない場合は、その行に印を付け、差額は計算しないでください。

割引率にも同じ注意が必要です。同じAmazon検索では、20件中15件に参考価格が表示されており、割引率は8.0%から44.4%まででした。参考価格は出品ごとに独自に設定されるため、ある商品の40%割引と別の商品の20%割引を比べてもほとんど意味がありません。最終価格を追跡し、割引率は同じ出品を週ごとに比較する場合にのみ使ってください。

![Amazon米国の検索結果20件における割引率の分布を示す棒グラフ。割引率0〜10%が1件、10〜20%が5件（ティール色で表示、17.2%のCrispi FN101STを含む）、20〜30%が3件、30〜40%が3件、40〜50%が3件です。5件は参考価格の表示がありませんでした。2026年11月3日、シンガポール時間に実施したamazon/searchの呼び出しで取得したデータです。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/fig-10-discount-dist-bar.ja.light.svg)

Walmartの検索結果には`upc`フィールドがあります。カタログによると、UPCはストアをまたいで同じ商品を照合する最も正確な方法です。同じ注記には、メインのソースが利用できない場合、このフィールドが欠落することがあるとも書かれています。今回の3件の`walmart/search`レスポンスには、いずれも`upc`の値がありませんでした。そのため、照合リストは人が一度作成し、タスクは毎週同じIDを読み込むようにしてください。毎週モデルに照合を推測させないでください。

2つ目の発見は、Amazonの評価数がバリエーション間で共有されていることです。同じ検索で、Instant Pot VortexのASIN 3件（B08R6KMBQT、B0F9B4YCZF、B0F9B48Q1B）はいずれも評価数が36,519でした。Ninja CrispiのASIN 4件（FN101の3色とFN103の1件）はいずれも8,426でした。色や型番の末尾は違っても数が同じなので、1つのバリエーションファミリーとして評価数を共有しています。4件のCrispiのうち1件は5-in-1のFN103で、4-in-1のFN101とは機能も異なります。

「今週Amazonの評価数が200増えた」を特定の1色の販売シグナルとして読むと、ファミリー全体のレビューをその色の実績として扱ってしまいます。表ではこの数値を「ファミリー評価数」と表記し、それを共有するASINを列挙しています。

## テスト2：1つの商品、2つのキーワード、2つの販売者

00:16:52に「ninja air fryer 4 quart」で実行した`walmart/search`の呼び出しでは、Crispi FN101PK1（商品20370717802）は6番目の結果でした。価格は$135、旧価格は$179.99で、販売者はPatent Paidでした。00:25:46に「ninja crispi glass air fryer」で実行したMCPの`walmart/search`の呼び出しでは、同じ商品が2番目の結果でした。価格は$134.50、旧価格は変わらず、販売者はMercshopでした。2回の呼び出しは約9分間隔だったため、当初はその間に販売者が変わったと解釈しました。

約1時間後の2回目のスナップショットで、この解釈が誤りだとわかりました。シンガポール時間01:13:54、1つ目のキーワードでは、Crispiが再び6番目の結果に表示され、販売者はPatent Paid、価格は$135でした。その9秒後の01:14:03、2つ目のキーワードでは再び2番目の結果に表示され、販売者はMercshop、価格は$134.50でした。この違いは時間ではなく、キーワードによるものです。

これはWalmart自身のルールで説明できます。1つの商品ページに、複数の販売者からのオファーが存在することがあります。「Add to cart」ボタンの上に表示される販売者がBuy Boxを獲得しています。Walmartは価格、送料、配送スピード、販売者のパフォーマンス、顧客の所在地をもとに獲得者を決めます（[Walmart Buy Boxガイド](https://marketplacelearn.walmart.com/guides/Item%20setup/Item%20content,%20imagery,%20and%20media/Product-detail-page:-the-buy-box)）。同じガイドでは、Walmart Repricerを有効にして価格を自動で変更するよう販売者に勧めています。

価格ルールのページには、もう1つの仕組みが記載されています。Walmartは、Walmart.comや他サイトでの最近の価格を大きく上回るオファーを自動的に非公開にします。それほど極端でない場合は、商品ページから「Add to cart」ボタンと価格が削除されます（[Walmart価格ルール](https://marketplacelearn.walmart.com/guides/Catalog%20management/Price%20management/Pricing-rules)）。つまり、サードパーティの販売者が多い商品では、1つの商品IDに複数のオファーがひもづいています。ただし、これらのルールでは、同じ時点で2つのキーワードによる検索結果に異なる販売者が表示された理由は説明できません。原因はわからないため、観察した事実のみを記録しています。

そのため、各行では価格と併せて販売者を記録します。ルールは3つです。各商品には設定タブで固定キーワードを1つ割り当て、タスクは毎週同じキーワードを使います。行のキーはストア＋商品ID＋販売者名です。販売者が変わった場合は行に「販売者変更」フラグを付け、タスクでは価格変更として報告しません。

レビューデータからは、1つの商品に販売者がどれだけいるかがわかります。Crispiのレビュー100件のうち30件に販売者名があり、その30件は7社の販売者によるものでした。Walmart.comが11件、Shark Ninja Storeが10件、SDG Distributionが4件で、残り4社がそれぞれ1〜2件でした。Ninja基本モデルのAF100WMでは、100件中98件がWalmart.comでした。販売者が多いほど、表で混同しうるオファーも増えます。

## テスト3：レビュー（低評価の一部は配送に関するもの）

シンガポール時間00:19:07に、同期の`walmart/reviews`呼び出しでNinja AF100WMのレビュー100件を取得しました。結果は66.1秒で返されました。00:22:20には、Crispi FN101PK1を対象に同じ呼び出しを非同期モードで送信し、00:23:49に結果を受け取りました。料金はいずれも$0.001389でした（[WalmartレビューAPI](https://everyinfra.com/ja/walmart-api/reviews)）。

| 指標 | Ninja AF100WM | Ninja Crispi FN101PK1 |
| --- | --- | --- |
| 取得したレビュー数 | 100 | 100 |
| 星5／4／3／2／1 | 70 / 12 / 6 / 3 / 9 | 63 / 10 / 7 / 6 / 14 |
| 星1〜2の割合 | 12.0% | 20.0% |
| 配送に関する星1〜2のレビュー | 12件中5件 | 20件中6件 |
| 購入確認済みフラグ | 99件にあり、1件はフィールドなし | 55件にあり、45件はフィールドなし（未確認としてのフラグはなし） |
| 最も古いレビューと最も新しいレビュー | 2025-07-17と2026-10-30 | 2025-08-20と2026-10-30 |
| 商品評価とレビュー総数（レビューAPIの呼び出しで取得） | 4.7。レビュー総数は1つのレスポンス内で12,562、12,571、12,573の3つの値 | 4.5と1,877（100件すべてで同じ） |

![積み上げ棒グラフ。各棒はWalmartのレビュー100件を表しています。Ninja AF100WM：星5が70件、星4が12件、星3が6件、星2が3件、星1が9件。Ninja Crispi FN101PK1：星5が63件、星4が10件、星3が7件、星2が6件、星1が14件。ティール色の部分は配送に関する星1〜2のレビューを示し、AF100WMは5件、Crispiは6件です。2026年11月3日のシンガポール時間00:19と00:23に実施した呼び出しで取得したデータです。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/fig-04-stars-bar.ja.light.svg)

ここで最も役立つ数値は、低評価のうち商品とは無関係なものがどれだけあるかです。判定にはシンプルなキーワードルールを使いました。receive、deliver、ship、arrive、package、address、DHL、「never got」などの語を含むレビューを「配送」に分類します。このルールでは、AF100WMの低評価12件中5件、Crispiの20件中6件が配送に関するものでした。短い例としては「Never got it」「Delivered to wrong address」のほか、DHLの注文が届かないという苦情があります。

Walmartのレビューガイドラインでは、商品レビューは商品とその使用体験に焦点を当てるべきだとしています（[Walmartのレビュー規約](https://www.walmart.com/help/article/ratings-reviews-question-and-answer-terms-of-use/0048b13a848c47279cf37023d6ef536d)）。しかし実際には、配送に関する苦情も投稿され、評価を下げています。ブランドマネージャーにとって、ここからわかることは2つあります。低評価の割合が高い競合商品でも、商品自体に問題があるとは限りません。サードパーティの販売者による配送に問題がある可能性があるためです。また、自社の低評価の大半が配送に関するものであれば、商品を改良しても解消しません。そのため、表では低評価を「商品」と「配送・販売者」の2列に分けています。

もう1つ注意すべき点は、100件のレビューの対象期間です。AF100WMのレビューは2025年7月から2026年10月にわたります。Crispiは100件中93件が2026年8月から10月に集中しています。並び順は厳密な新着順ではありません。

AF100WMのレビューのうち最初の30件には、6月、9月、10月の日付が混在しています。そのため、このサンプルから「今週の新規レビュー」を数えることはできません。サンプル内では、AF100WMのレビュー5件が10月27日〜11月2日、4件がその前の週に該当します。これはサンプル内の件数であり、総数ではありません。

今週のレビュー総数から先週の総数を単純に引かないでください。テスト4を見ると、その理由が分かります。新たな低評価については、先週の`review_id`の値を保存しておき、まだ見ていないIDだけを数えます。

![2025年7月から2026年10月までの16か月間の棒グラフ。グレーの棒はAF100WMのレビュー100件の月別件数、ティールの棒はCrispiのレビュー100件の月別件数を示します。AF100WMのレビューは16か月全体に分散しており、件数が特に多いのは2026年10月（14件）と2025年11月（12件）です。Crispiのレビューは2026年8月（24件）、2026年9月（45件）、2026年10月（24件）に集中しています。データは、2026年11月3日（シンガポール時間）に行った呼び出しから取得しました。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/fig-07-review-months-timeline.ja.light.svg)

考慮すべき遅延もあります。Walmartによると、新しいレビューがサイトに表示されるまでに5〜7営業日かかります（[Walmartのレビュー規約](https://www.walmart.com/help/article/ratings-reviews-question-and-answer-terms-of-use/0048b13a848c47279cf37023d6ef536d)）。WalmartはMarketplaceの顧客に、配送または受け取りの7日後にレビュー依頼メールを送ります。顧客がそれを開封しない場合は、28日後に2通目のメールを送ります（[Walmart Marketplace Learn](https://marketplacelearn.walmart.com/guides/Growth%20opportunities/Reviews/Marketplace-reviews:-overview)）。

つまり、月曜日の「新たな低評価」の多くは2〜3週間前の注文によるものです。競合が先週、販売数を伸ばすために値下げした場合、そのレビューへの影響が表に現れるのは2〜3週間後です。この遅延は判断列に反映しています。価格が変化した場合は、同じ週に注記を出します。レビューが変化した場合は、少なくとも2週間待ってから対応します。

## テスト4：1つのレスポンス内で1つの数値項目が取る3つの値

数値そのものが安定していなければ、週次比較は成り立ちません。1つのセッションで、AF100WMのレビュー数を3つのツールから読み取りました。00:16:52の`walmart/search`の呼び出しでは12,571でした。00:21:10に送信し00:22:03に読み取った`walmart/product_detail`のジョブも12,571でした。

00:19:07の`walmart/reviews`の呼び出しでは、100件のレビューが返されました。各レビューには総数を示す`product_review_count`フィールドがあります。その1つのレスポンス内で、このフィールドには3つの値がありました。12,562が40件、12,571が40件、12,573が20件です。約1時間後の2回目の呼び出しでは、同じ100件の`review_id`が返されました。このときの内訳は、12,562が20件、12,571が40件、12,573が40件でした。

1つのレスポンスの中でさえ、値が11件分ずれることがあります。そのため、最初のレビューだけを読まないでください。また、週ごとの総数を引き算して新規レビュー数を求めないでください。

週次の表では3つのルールを使います。1つ目は、各指標を毎週同じツールから読み取ることです。2つ目は、100件のレビューの中で最も大きい`product_review_count`を採用することです。最頻値は安定しません。最初のレスポンスでは、12,562と12,571がそれぞれ40件で並んでいます。

両方の生レスポンスに対してPythonを実行しました。`statistics.multimode()`は最初のレスポンスに対して2つの値を返します。`statistics.mode()`は同数の場合に最初に見つかった値を返すため、1回目は12,562、2回目は12,571になります。すると表には、存在しない新規レビューが9件と表示されます（[statisticsのドキュメント](https://docs.python.org/3/library/statistics.html)）。最大値はどちらも12,573でした。

3つ目は、総数の先週との差が15件以下なら「変化なし」と記録することです。15件という基準は、2つのサンプルで見られた11件のばらつきをもとに設定しました。プラットフォームのルールではありません。数週間運用したら、自社のデータに合わせて調整してください。

2つ目の意外な点は、`walmart/product_detail`に価格がないことです。2つの詳細レスポンスに含まれていたのは、`id`、`url`、`name`、`text`、`brand`、`rating`、`rating_scale`、`review_count`、`image_url`、`platform`のフィールドだけでした。カタログではこのアクションに`price`、`original_price`、`in_stock`、`seller_name`が記載されていますが、いずれも含まれていませんでした（[Walmart商品詳細API](https://everyinfra.com/ja/walmart-api/product_detail)）。表はこれを前提に設計しています。

価格、値下げ前価格、出品者は`walmart/search`から取得します。商品が検索結果の上位20件から外れた場合は、詳細の呼び出しで評価とレビュー数を補います。モデル名の表記を確認する際にも役立ちます。

最初に同期モードで詳細呼び出しを行ったところ、88.0秒後にHTTP 503が返され、料金はかかりませんでした。`"mode":"async"`を指定すると、ジョブIDは2.0秒で返り、結果は約53秒後に取得可能になりました。

| 機能 | このテストで最も遅い呼び出し | 結果 | 課金額 |
| --- | --- | --- | --- |
| `amazon/search`（20件） | 36.8 s | 200 | $0.002236 |
| `walmart/search`（20件） | 43.6秒（MCP経由） | 200 | $0.001389 |
| `walmart/reviews`（100件、同期） | 66.1 s | 200 | $0.001389 |
| `walmart/product_detail`（同期） | 88.0 s | 503タイムアウト | $0 |
| `walmart/product_detail`（非同期） | 送信に2.0秒、結果は約53秒後に取得可能 | 200 | $0.001389 |
| `amazon/reviews`（MCP経由） | 88.8 s | 開けず、返金済み | $0 |

![リクエストからレスポンスまでの秒数を示す横棒グラフ。Amazonメキシコの検索は19.3秒、WalmartでのVortexの検索は35.8秒、Amazonの検索は36.8秒、Walmartの検索は39.0秒、MCP経由のWalmartの検索は43.6秒、Walmartのレビューは66.1秒。ティールの2本の棒に対応する呼び出しでは、約88秒待ってもデータが返されませんでした。同期モードのWalmart詳細は88.0秒でタイムアウトしました。MCP経由のAmazonレビューは88.8秒待っても開けず、返金されました。データは、2026年11月3日（シンガポール時間）に行った呼び出しから取得しました。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/fig-05-latency-bar.ja.light.svg)

これらの時間は、スケジュールされたタスクにとって何を意味するでしょうか。週1回の実行で10回の呼び出しを行います。順番に実行すると7〜8分かかり、最も遅いのはreviewsとdetailの呼び出しです。プロンプトでは、detailとreviewsの呼び出しを非同期モードで送信するようタスクに指示します。タスクはその後、各結果を`everyinfra_get_job`で読み取ります。ツールの説明によると、ジョブの読み取りは無料で、1分あたりの制限にもカウントされません。

タイムアウトした同期呼び出しは再送しないでください。503レスポンスには「Not charged」と記載され、20秒後の再試行（最大2回）と非同期モードの利用が推奨されていました。あるRedditユーザーは、無人実行の主なリスクを次のように説明しています。

429エラーやネットワークの一時的な障害が起きると、エージェントは素直に再試行せず、次々と独創的な回避策を試し始めます。対策は、タスクに「Xが失敗したら停止して報告し、自分で修正しようとしないこと」と書いておくことです（[Redditのスレッド](https://www.reddit.com/r/ClaudeAI/comments/1u6ab7d/scheduled_tasks_routines_whats_actually_holding/)）。

## 今週と先週の比較：データの保存場所

これが、Coworkのスケジュールされたタスクと通常のチャットとの最大の違いです。アーキテクチャの記事によると、クラウドセッションでは、Claudeのループとコードは、Anthropicがホストする隔離された一時的なサンドボックスで実行されます。セッションごとに専用のサンドボックスが割り当てられます。サンドボックスはセッション開始時に作成されて終了時に破棄され、サンドボックス間で状態は共有されません（[Claude Coworkアーキテクチャの概要](https://support.claude.com/en/articles/14479288-claude-cowork-architecture-overview)）。そのため、今週月曜日の実行で作成したCSVは、翌週月曜日の実行からは参照できません。

クラウドセッションでは、お使いのコンピューターで接続したフォルダーを読み書きできます。ただし、それができるのはデスクトップアプリが開いている間だけです（[Use Claude Cowork on web, desktop, and mobile](https://support.claude.com/en/articles/15520349-use-claude-cowork-on-web-desktop-and-mobile)）。これでは、コンピューターを起動したままにしておく必要があるという従来の問題が再び生じます。

| 保存先 | スリープ中も動作するか | 長所 | 注意点 |
| --- | --- | --- | --- |
| Googleスプレッドシート（Googleコネクタ） | 可能（コネクタ経由） | シートを作成・編集でき、チームメンバーも閲覧できる | プロンプトで「Googleスプレッドシート」と明記しないと、Claudeはローカルファイルを作成する |
| OneDriveまたはSharePoint（Microsoft 365コネクタ） | 可能（コネクタ経由） | 書き込みツールを有効にすれば、ファイルの作成・更新が可能 | 管理者が書き込みツールを有効にする必要がある |
| コンピューター上の接続済みフォルダー | 不可（デスクトップアプリを開いておく必要がある） | 設定が最も簡単 | ローカルファイルが必要なタスクはローカルでのみ実行される |
| セッション内のファイル | 次のセッションからは読み取れない（これと異なる記述は見つからなかった） | 週ごとに1つのアーカイブコピーを残せる | 人が読むには向くが、比較には向かない |

おすすめはGoogleスプレッドシートです。ヘルプセンターによると、Googleスプレッドシートコネクタでは新しいスプレッドシートの作成と編集ができます。検索、アップロード、共有はGoogleドライブコネクタが担います。同じ記事では、「Google」という語を付けず、単に「doc」と依頼すると、代わりにローカルファイルが作成されると注意しています（[Use Google Workspace connectors](https://support.claude.com/en/articles/10166901-use-google-workspace-connectors)）。

Microsoft 365を使っているチームは、コネクタの書き込みツールを有効にできます。これらのツールで、OneDriveやSharePointにファイルを作成・更新できます（[Set up the Microsoft 365 connector](https://support.claude.com/en/articles/12542951-set-up-the-microsoft-365-connector)）。タブは2つで十分です。`snapshots`タブには毎週新しい生データの行を、`changes`タブには毎週の比較結果を追加します。

比較が正しくなるかどうかはキーで決まります。週には、Pythonの`date.isocalendar()`が返す年と週番号を使います。そうすれば、年や月の境目でもエラーが起きません（[datetime documentation](https://docs.python.org/3/library/datetime.html)）。2026年11月2日（月）は2026-W45、11月9日（月）は2026-W46に含まれます。

行のキーは、ストア、商品ID、販売者の組み合わせです。レビューは`review_id`で重複を除去します。

```text
# Comparison logic example. It runs in the Cowork session's code tool.
# We ran it on our machine with two real snapshots.
def key(r):  # store + item ID + seller
    return (r["store"], r["item_id"], r.get("seller_name") or "-")

def diff(last_week, this_week):
    prev = {key(r): r for r in last_week}
    prev_item = {(r["store"], r["item_id"]): r for r in last_week}
    out = []
    for r in this_week:
        p = prev.get(key(r))
        if p is None and (r["store"], r["item_id"]) in prev_item:
            out.append({**r, "flag": "seller change", "seller_before": prev_item[(r["store"], r["item_id"])].get("seller_name")})
        elif p is None:
            out.append({**r, "flag": "new"})
        else:
            out.append({**r, "price_change": round((r["price"] or 0) - (p["price"] or 0), 2),
                        "count_change": (r.get("review_count") or 0) - (p.get("review_count") or 0),
                        "flag": "price change" if r["price"] != p["price"] else ""})
    return out
```

## 2つのスナップショットによる実際の比較

比較コードを実データでテストするため、1回目の約1時間後に2回目のスナップショットを取得しました。実行時間は2026年11月3日のシンガポール時間01:12:58から01:15:52で、パラメータは同じです。呼び出しは5回で、内訳は`amazon/search`が1回、`walmart/search`が3回、AF100WMに対する非同期の`walmart/reviews`が1回です。課金額は$0.007792でした。

続いて、両方のスナップショットを前述の`diff()`関数に渡し、ストア、商品ID、販売者をキーとして比較しました。実際のレポートでは2週間分を比較します。今回は間隔が1時間なので、このテストで分かるのは手法と、自然に変動するフィールドです。

| ストア、商品ID | 販売者 | 1回目の価格 | 2回目の価格 | 価格の変化 | 件数の変化 | フラグ |
| --- | --- | --- | --- | --- | --- | --- |
| Amazon、B07FDJMC9Q（AF101） | none | $119.99 | $119.99 | 0 | 0 (59,532) | none |
| Walmart、844320666（AF100WM） | Walmart.com | $59.00 | $59.00 | 0 | 0 (12,571) | none |
| Amazon、B0DJFBDFZK（Crispi） | none | $149.00 | $149.00 | 0 | 0 (8,426) | none |
| Walmart、20370717802（Crispi） | Mercshop | $134.50 | $134.50 | 0 | 0 (1,877) | none |
| Amazon、B0F9B4YCZF（Vortex） | none | $69.95 | $69.95 | 0 | 0 (36,519) | none |
| Walmart、17316906990（Vortex） | Instant Pot | $69.99 | $69.99 | 0 | 0 (108) | none |

1時間のあいだに、追跡中の6行はいずれも価格、販売者、評価数、レビュー数が変わりませんでした。AF100WMのレビューを2回目に呼び出したところ、同じ100個の`review_id`が同じ順序で返りました。そのため新規レビュー数は0です。`product_review_count`フィールドも同じ3つの値のままでした。変わったのは内訳だけで、40/40/20から20/40/40になりました。最大値はどちらも12,573だったため、ルールは「変化なし」と記録します。

一方、Amazonの検索順位は変化しました。同じキーワードの上位20件のうち、17件のASINは2回とも現れ、3件は新たに現れました。AF101は2位から14位に下がりました。Crispiは9位から1位に上がり、Vortexは3位から4位になりました。

タスクが「上位の結果」を選ぶのではなく、固定のASINを読み取るのはこのためです。価格が変わらなくても、順位は1時間で大きく動くことがあります。順位で商品を突き合わせる表では、実際にはない変化が多数表示されます。

![Amazon検索結果の上位20件における競合3製品の順位を示すグループ化棒グラフ。グレーは00:15の1回目、青緑は01:12の2回目の実行です。Ninja basic AF101は2位から14位、Ninja Crispiは9位から1位、Instant Vortex miniは3位から4位になりました。価格、販売者、評価数、レビュー数はどちらの回も同じでした。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/fig-08-diff-bar.ja.light.svg)

同じコードで、キーワードを変えた場合のテストもしました。Crispiの2行目を「ninja air fryer 4 quart」の結果に置き換えました。この結果ではPatent Paidが$135で表示されます。`diff()`の出力は「price up $0.50」ではなく「seller change, before: Mercshop」でした。これがキーに販売者を含める理由です。この行に対する`diff()`の実際の出力は次のとおりです。

```text
{"store": "walmart.com", "item_id": "20370717802", "seller_name": "Patent Paid", "price": 135, "review_count": 1877, "flag": "seller change", "seller_before": "Mercshop"}
```

2回目のスナップショットでは、`walmart/search`の3回の呼び出しにかかった時間は5.9〜8.7秒で、1回目は35.8〜43.6秒でした。レスポンスには、結果がキャッシュから返されたかどうかを示すフィールドはありません。そのため、変化は呼び出しにかかった時間ではなく、スナップショットのフィールドで判断してください。

## プロンプトと出力テーブル

Coworkのスケジュール済みタスクは、毎回同じように実行される保存済みのプロンプトです。ヘルプセンターでは「Create with Claude」について説明しています。Claudeがいくつかの選択式の質問をした後、名前、スケジュール、手順を表示して確認を求めます（[Schedule recurring tasks in Claude Cowork](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-claude-cowork)）。おすすめは「Set up manually」と以下のプロンプトです。ツール、パラメータ、失敗時のルールを固定したリストのほうが、モデルが毎週考え直すプロンプトよりも安定します。

「Weekly」を選び、月曜日の朝の時刻を指定します。手動設定のフォームでは、モデルのフィールドは任意です。Redditユーザーの報告によると、タスクの編集ページにもモデル設定があります（[Reddit thread](https://www.reddit.com/r/Anthropic/comments/1ukail1/cowork_mode_any_way_to_set_which_model_scheduled/)）。このタスクは主にツールを呼び出して表を埋めるだけなので、低コストのモデルで十分です。

```text
Configuration example: scheduled task prompt
(Set up manually. Frequency: weekly, Monday 09:10. Approval mode: Automatically approve.)

You are a competitor report assistant. Read only. Do not order, change prices or send messages.
Use only the EveryInfra connector and the Google Sheets connector.
1. Open the Google Sheets file "Competitor weekly" (create it if missing).
   Read all rows in the snapshots tab for the previous ISO week (for example 2026-W45).
2. Read the config tab. One row per competitor: key, amazon_asin, walmart_item_id, walmart_keyword, model.
3. Call everyinfra_call_api: platform=amazon, action=search,
   params={keyword: "ninja air fryer 4 quart", limit: 20}.
   Keep only ASINs from config. Record price, list_price, rating, rating_count (label it family rating count).
4. For each competitor:
   a. platform=walmart, action=search, params={keyword: walmart_keyword}.
      Find walmart_item_id. Record price, was_price, seller_name, rating, review_count.
   b. If step a did not find the ID: platform=walmart, action=product_detail,
      params={url: "https://www.walmart.com/ip/<ID>"}, mode=async. Record rating and review_count only.
   c. platform=walmart, action=reviews, params={url: same, limit: 100}, mode=async.
      Read the result with everyinfra_get_job. Keep only review_id values not seen last week.
      Split new 1-2 star reviews into "product" and "delivery or seller". Write one sentence for each.
5. If a call fails: record the error and request_id, skip that item, retry at most once, do not try other methods.
6. Append this week's rows to snapshots. Key: week + store + item ID + seller.
7. Compare with last week on that key. Append to changes: price change, seller change,
   review count change, new low ratings and summary, action.
8. At the end, show this week's changes rows in the session, sorted by action.
   State how many API calls ran and how many failed.
```

| 列 | 出典 | 備考 |
| --- | --- | --- |
| week | 実行日 | ISO週（例：2026-W45） |
| store, item_id, seller | 検索結果 | 照合キー。新しい販売者には「seller change」フラグを付けます |
| price, list_price, discount_pct | `amazon/search`, `walmart/search` | 割引率 = 1 − 価格 ÷ 定価 |
| rating, count | 検索。上位20件から外れた場合は詳細を取得 | Amazon：バリエーション全体の評価数 |
| price_change, count_change | 同じキーで今週の値から先週の値を引く | 常に同じツールから取得 |
| new_low_star, themes | `walmart/reviews` | review_idで重複を除外し、商品と配送に分類 |
| action | ルール | 価格の変化：今週中に記録。レビューの変化：2週間様子を見る |

## 週間・月間のAPIコスト

カタログ価格でのコストは次のとおりです。`amazon/search`は1回あたり$0.002236（1,000回あたり$2.236）です。`walmart/search`、`walmart/product_detail`、`walmart/reviews`は1回あたり$0.001389（1,000回あたり$1.389）です。これらの`price_usd`の値は、2026年11月3日のシンガポール時間00:15に[無料のカタログ](https://api.everyinfra.com/api/v1/social/catalog)から取得しました。失敗した呼び出しと空の結果には課金されず、非同期ジョブの結果の取得は無料です。以下の数値はすべて、これらの価格からスクリプトで算出しました（[料金](https://everyinfra.com/ja/pricing)）。

| 方式 | 週あたりの呼び出し回数 | 週あたり | 月あたり（52週 ÷ 12） | 年あたり |
| --- | --- | --- | --- | --- |
| 競合3社、毎週 | 1 + 3 + 3 + 3 = 10 | $0.014737 | $0.0639 | $0.77 |
| 競合10社、毎週 | 2 + 10 + 10 + 10 = 32 | $0.046142 | $0.1999 | $2.40 |
| 競合3社、毎日 | 70 | $0.103159 | $0.447 | $5.36 |

![3つのプランの月間APIコストを示す横棒グラフ。競合3社・毎週：0.0639ドル（ティール色）。競合10社・毎週：0.1999ドル。競合3社・毎日：0.447ドル。サブタイトルには、Claude Proの料金は月20ドルで、毎週プランのAPIコストはその0.32%であると記されています。価格は2026年11月3日に取得したEveryInfraのカタログによるものです。](https://everyinfra.com/articles/claude-cowork-scheduled-tasks-competitor-price-reviews/fig-06-cost-bar.ja.light.svg)

サブスクリプションと比べると、APIコストはわずかです。競合3社を週1回追跡する場合の費用は月$0.0639で、月額$20のClaude Proの0.32%です（[Claudeの料金](https://claude.com/pricing)）。実質的な負担となるのはClaudeの使用量です。ヘルプセンターによると、Coworkは複雑な複数ステップのタスクを処理します（[Get started with Claude Cowork](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork)）。claude.ai、Claude Code、Claude Desktopでの使用量は、共通の1つの上限にカウントされます。有料プランには、5時間ごとにリセットされるセッション上限と週間上限があります（[How do usage and length limits work](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work)）。

スケジュール実行のたびに、この上限枠を消費します。20件の結果を含むAmazon検索1回分は71,614バイト、レビュー100件は約60,000バイトです。モデルがこれをすべて読み込むと、1回の実行で上限枠を大きく消費します。そのため、プロンプトでは設定タブにIDだけを保持し、新しいレビューだけを読むよう指示しています。

## 制限事項と未解決の点

| 項目 | 今回のテストでの状況 | 対処法 |
| --- | --- | --- |
| Amazonのレビュー本文 | `amazon/reviews`でAF101を開けず、返金済み | 検索結果の評価数を使用 |
| `walmart/product_detail`の価格 | 両商品とも欠落 | 価格は`walmart/search`から取得 |
| Temu | `temu/search`は一時停止中（`available: false`） | 今回のバージョンでは対象外 |
| Cowork側 | 当社では未実行 | 手順を例として参照し、「Run now」でテスト |
| キーワードによって販売者が変わる原因 | 不明 | 商品ごとにキーワードを1つに固定 |

1つ目に、今回のテストではAmazonのレビュー本文を取得できませんでした。AF101に対する`amazon/reviews`は「needs a login or is restricted」を返し、返金されました。正常に動作した場合でも、カタログによると、返されるのは商品ページに掲載された12件前後の注目レビューのみです。そのため、表のAmazon側には価格、定価、評価、バリエーション全体の評価数しかありません。

第2に、今回のテストでは`walmart/product_detail`はどちらの商品についても価格、在庫、販売者を返しませんでした。価格は`walmart/search`から取得する必要があります。検索は20件の結果を1ページで返します。競合が上位20件から外れたら、より具体的なキーワードを使ってください。

3つ目に、多くのブランドマネージャーはTemuも追跡しています。しかし、カタログでは`temu/search`が一時停止中です（`available: false`）。呼び出すと503が返り、課金はされないため、今回のバージョンの表にはTemuを含めていません。

4つ目に、Cowork側はテストしていません。承認、コネクタ、Google Sheetsへの書き込み、モデルの選択は、すべてAnthropicのヘルプセンターとドキュメントに基づいています。Teamプランで「Always allow」がオフの場合にスケジュール実行がどう動作するかは、テストできませんでした。

5つ目に、この表は自社の価格設定と製品に関する判断にのみ使用してください。2024年8月のFTC規則は、偽レビューの販売と購入を禁止しています。また、肯定的または否定的なレビューへの対価の支払いや、否定的なレビューの抑制も禁止しています（[FTCプレスリリース](https://www.ftc.gov/news-events/news/press-releases/2024/08/federal-trade-commission-announces-final-rule-banning-fake-reviews-testimonials)、[16 CFR Part 465](https://www.ecfr.gov/current/title-16/chapter-I/subchapter-D/part-465)）。競合商品に低評価が多く付いた場合は、自社の製品と配送を改善してください。競合のレビューに影響を与えようとしないでください。本記事はデータ活用の方法を説明するもので、法的助言ではありません。

## 最近の変更点

- 2026年10月6日：Pro、Max、Ultraプランの新しいCoworkタスクとスケジュール済みタスクはクラウドで実行されるようになり、「Only on your computer」は廃止されました（[ヘルプセンター](https://support.claude.com/en/articles/15520349-use-claude-cowork-on-web-desktop-and-mobile)）。つまり、表の更新にコンピューターを起動しておく必要がなくなった一方、スナップショットの保存にローカルフォルダを使えなくなりました。
- 2026年10月22日：デスクトップアプリv3.6556.1では、人の操作が必要なMCPツールを使わずにスケジュール済みタスクを続行できるようになりました。また、スケジュール済みタスクがプラグインやスキルを見つけられない問題も修正されています（[変更履歴](https://claude.com/docs/cowork/changelog)）。つまり、一部が欠けた状態で完了した実行は、ツールをスキップした可能性があります。
- 2026年10月26日：Walmartは、11月12日から12月2日にかけて3つのホリデーイベントを開催すると発表しました。開催期間は合計14日間で、2025年の10日間から増えています（[Walmartのプレスリリース](https://corporate.walmart.com/news/2026/10/walmart-unwraps-three-major-holiday-shopping-events)、[Retail Dive](https://www.retaildive.com/news/walmart-longer-black-friday-cyber-monday-sales-events/872747/)）。つまり、今後5週間が価格変動の最も大きい時期となります。毎日実行する設定に切り替えることができ、競合3社の場合は週$0.10かかります。
- 2026年10月27日：v4.29.0では、2つの自動モード以外のスケジュールタスクでWebフェッチが制限されます。URLは指示内に記載されているか、検索結果から得たものである必要があります（[変更履歴](https://claude.com/docs/cowork/changelog)）。つまり、Cowork自体のWebフェッチで商品ページを読み込むのは難しくなり、コネクタを使う方が安定します。

## FAQ

### Claude Coworkのスケジュールタスクが動作しない場合はどうすればよいですか？

この質問は、検索候補の「claude cowork scheduled tasks not working」と「claude cowork scheduled tasks connector not working」に基づいています。まず、どのレイヤーで障害が発生したかを特定してください。Cowork自体では、あるmacOSユーザーがタスクを作成できず、「Invalid file path: path traversal detected」というエラーが表示されたと報告しています（[Redditスレッド](https://www.reddit.com/r/ClaudeAI/comments/1wwq0p9/anyone_else_unable_to_create_or_edit_scheduled/)）。関連するGitHub Issueには、リモートセッションではローカルのスケジュールタスクを管理するツールを利用できないとあります（[GitHub Issue #76968](https://github.com/anthropics/claude-code/issues/76968)）。

10月22日のアップデートでは、スケジュールタスクがプラグインやスキルを見つけられない問題も修正されたため、まずデスクトップアプリを更新してください（[変更履歴](https://claude.com/docs/cowork/changelog)）。コネクタについては、ヘッダーの値が`Bearer`とキーの組み合わせになっているか、サーバーが公開されているか、承認が許可されているかを確認してください。EveryInfraについては、request_idを指定して`everyinfra_list_requests`を呼び出すと、ステータスと課金額を確認できます。この呼び出しは無料です。

この表は、実際の呼び出しと上記のスレッドで確認された症状を、問題が通常発生するレイヤーとともにまとめたものです。

| 症状 | 考えられるレイヤー | 確認すること |
| --- | --- | --- |
| 「Failed to create scheduled task: Invalid file path」 | Coworkアプリ | アプリを更新してください。Redditのスレッドと Issue #76968も参照してください |
| 実行は完了するが、一部が欠けている | 承認 | 一度「Run now」をクリックし、「Allow for all tasks」を選択してください |
| `unauthorized: no API key provided` | コネクタのヘッダー | 値は`Bearer`、スペース、キーの順にする必要があります。変更するには、コネクタを削除して再度追加してください |
| コネクタが接続できない | ネットワーク | サーバーが公開されており、`160.79.104.0/21`からの接続を許可している必要があります |
| 詳細取得で503タイムアウト | 同期呼び出し | `"mode":"async"`を使い、`everyinfra_get_job`で結果を取得してください |
| `amazon/reviews`：「needs a login or is restricted」 | 制限付きの商品 | この呼び出しは返金されます。検索結果の評価数を使ってください |
| 実際には新しい出品者によるものだった値下がり | テーブルのキー | キーに出品者を含める必要があります |

### Claude Coworkのスケジュールタスクの上限はいくつですか？

この質問は、検索候補「claude cowork scheduled tasks limit」に基づいています。あるRedditユーザーが作成できるスケジュールタスクの数を尋ねましたが、Anthropicが公表した数値を挙げた回答はありませんでした（[Redditのスレッド](https://www.reddit.com/r/ClaudeAI/comments/1u6ab7d/scheduled_tasks_routines_whats_actually_holding/)）。関連するヘルプ記事を4本確認しましたが、Coworkのスケジュールタスク数や実行回数の上限は見つかりませんでした。確かなのは、実行によってサブスクリプションの利用枠が消費され、5時間ごとおよび週ごとの上限が適用されることです（[使用量の制限](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work)）。比較すると、Claude Codeのクラウドスケジュールタスクでは、アカウントあたり1時間に100回のスケジュール実行が可能です（[ドキュメント](https://code.claude.com/docs/en/routines)）。EveryInfra側のレート制限は、APIキーごとの1分あたりのリクエスト数で決まり、コンソールで引き下げることができます。

### Claude CoworkのスケジュールタスクとClaude Codeのルーティン、どちらを使うべきですか？

この質問は、検索候補「claude cowork scheduled tasks vs claude code routines」に基づいています。データと結果がどこにあるかで決めましょう。結果はビジネスユーザー向けのシートやドキュメントですか？それなら、Google SheetsまたはMicrosoft 365のコネクタと組み合わせてCoworkを使ってください。データやスクリプトはGitリポジトリにありますか？

それなら、Claude Codeのクラウドスケジュールタスク（旧称ルーティン）を使い、履歴はコミットに残しましょう。こちらはAPI呼び出しやGitHubイベントをきっかけに開始することもできます（[Claude Codeのドキュメント](https://code.claude.com/docs/en/routines)）。どちらも同じEveryInfraのMCPアドレスを使えます。もう一方のセットアップは[Claude Codeスケジュールタスクのガイド](https://everyinfra.com/ja/blog/claude-code-scheduled-tasks-reddit-pain-points)で紹介しています。

### パソコンの電源が切れていても、Claude Coworkのスケジュールタスクはクラウドで実行されますか？

この質問は、検索候補「claude cowork scheduled tasks cloud」に基づいています。はい。ヘルプセンターによると、スケジュールタスクはリモートで実行されるため、パソコンがスリープ中でも、アプリを閉じていても実行されます。10月6日より後に作成された新しいタスクはクラウドで実行されます（[ヘルプセンター](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-claude-cowork)）。

例外は、ローカルのファイルやアプリを必要とするタスクと、10月6日より前からすでにパソコン上で実行されていた既存のタスクです。あるRedditユーザーはクラウドのオプションを見つけられませんでした（[Redditのスレッド](https://www.reddit.com/r/ClaudeCowork/comments/1v8tchd/i_cant_create_scheduled_tasks_on_remote_cloud/)）。そのスレッドの返信では、クラウドのオプションはプランごとに順次提供されていると書かれていました。現在のヘルプセンターでは、Pro、Max、Ultraの新しいタスクは10月6日からクラウドで実行されると説明されています。

### スケジュールタスクのモデルは選べますか？

この質問は、スケジュールタスクで使用するモデルの設定方法を尋ねるReddit投稿に基づいています（[Redditのスレッド](https://www.reddit.com/r/Anthropic/comments/1ukail1/cowork_mode_any_way_to_set_which_model_scheduled/)）。はい。手動設定フォームでは、モデルは任意のフィールドです（[ヘルプセンター](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-claude-cowork)）。投稿への返信では、タスクの編集ページにあるモデル設定も紹介されていました。価格表のタスクは主にツールを呼び出してシートを埋めるだけなので、低コストのモデルで十分です。そこで節約できた利用枠は、判断が必要なタスクに回しましょう。

### スケジュールタスクが時間どおりに始まらないのはなぜですか？

この質問は、リモートスケジューラーの実行時刻が、ときには約10分ずれると述べたReddit上の返信に基づいています（[Redditのスレッド](https://www.reddit.com/r/ClaudeAI/comments/1x7199x/s_using_claudeai_scheduled_tasks_to_trigger_the/)）。Coworkのヘルプセンターには、実行時刻の精度についての記載がありません。Claude Codeのクラウドのドキュメントによると、正時ちょうどに設定した実行は数分遅れて始まることがあります。そのため、9:07のように正時から数分ずらすことが推奨されています（[ドキュメント](https://code.claude.com/docs/en/routines)）。週次の価格表では、実行時刻が厳密である必要はありません。私たちの例でも、同じ理由で09:10を使っています。

## 始め方

1. 無料のEveryInfraアカウントを作成し、Cowork専用のAPIキーを発行します。[クイックスタート](https://everyinfra.com/ja/docs#quickstart)に沿って、curlで`walmart/search`を一度呼び出します。
1. Claudeで「Customize」>「Connectors」を開き、カスタムコネクタを追加します。URLに`https://api.everyinfra.com/mcp`を指定し、「No sign-in」を選んで、ヘッダー`Authorization = Bearer <key>`を設定します。詳しい手順は[MCPセットアップページ](https://everyinfra.com/ja/mcp-server)をご覧ください。
1. Google Sheetsコネクタをオンにします。「Competitor weekly」というスプレッドシートを作成し、設定タブに競合のIDを入力します。
1. 「Scheduled」で「Set up manually」を選び、上記のプロンプトを貼り付けます。「Weekly」を選択します。「Run now」を一度クリックし、表示される承認リクエストに回答します。
1. 次の月曜日に、変更タブを開きます。

関連する方法については、[サブスクリプション商品としての価格モニタリング](https://everyinfra.com/ja/blog/price-monitoring-subscription-product)をご覧ください。あわせて、[比較可能なオファーを使ったAmazonの価格追跡](https://everyinfra.com/ja/blog/amazon-price-tracking-comparable-offers)と[エージェントスキルを使った競合レビュー分析](https://everyinfra.com/ja/blog/agent-skills-competitor-review-analysis)もご覧ください。

## 参考文献

1. [Claude Coworkで定期タスクを設定する](https://support.claude.com/en/articles/13854387-schedule-recurring-tasks-in-claude-cowork)、Claudeヘルプセンター、2026-11-03閲覧。
1. [リモートMCPを使ったカスタムコネクタの始め方](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)、Claudeヘルプセンター、2026-11-03閲覧。
1. [ディレクトリにないコネクタを追加する](https://claude.com/docs/connectors/custom/add-unlisted)、Anthropicドキュメント、2026-11-03閲覧。
1. [Claude Coworkとチャットは同じClaudeです](https://support.claude.com/en/articles/16761823-claude-cowork-and-chat-are-one-claude)、Claudeヘルプセンター、2026-11-03閲覧。
1. [Web、デスクトップ、モバイルでClaude Coworkを使う](https://support.claude.com/en/articles/15520349-use-claude-cowork-on-web-desktop-and-mobile)、Claudeヘルプセンター、2026-11-03閲覧。
1. [Claude Coworkを安全に使用する](https://support.claude.com/en/articles/13364135-use-claude-cowork-safely)、Claudeヘルプセンター、2026-11-03閲覧。
1. [TeamプランとEnterpriseプランでClaude Coworkを使用する](https://support.claude.com/en/articles/13455879-use-claude-cowork-on-team-and-enterprise-plans)、Claudeヘルプセンター、2026-11-03閲覧。
1. [Claude Coworkを使い始める](https://support.claude.com/en/articles/13345190-get-started-with-claude-cowork)、Claudeヘルプセンター、2026-11-03閲覧。
1. [使用量と長さの制限の仕組み](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work)、Claudeヘルプセンター、2026-11-03閲覧。
1. [リリースノート](https://support.claude.com/en/articles/12138966-release-notes)、Claudeヘルプセンター、2026-02-25から2026-10-21までの項目。
1. [Claude Desktopの変更履歴](https://claude.com/docs/cowork/changelog)、Anthropic、v3.6556.1（2026-10-22）からv4.4212.0（2026-10-29）まで。
1. [料金プラン](https://claude.com/pricing)、Anthropic、2026-11-03閲覧。
1. [IPアドレス](https://platform.claude.com/docs/en/api/ip-addresses)、Anthropicドキュメント、2026-11-03閲覧。
1. [Claude Coworkのアーキテクチャ概要](https://support.claude.com/en/articles/14479288-claude-cowork-architecture-overview)、Claudeヘルプセンター、2026-11-03閲覧。
1. [スケジュールタスクで作業を自動化する](https://code.claude.com/docs/en/routines)、Claude Codeドキュメント、2026-11-03閲覧。
1. [プロンプトをスケジュール実行する](https://code.claude.com/docs/en/scheduled-tasks)、Claude Codeドキュメント、2026-11-03閲覧。
1. [Google Workspaceコネクタを使用する](https://support.claude.com/en/articles/10166901-use-google-workspace-connectors)、Claudeヘルプセンター、2026-11-03閲覧。
1. [Microsoft 365コネクタを設定する](https://support.claude.com/en/articles/12542951-set-up-the-microsoft-365-connector)、Claudeヘルプセンター、2026-11-03閲覧。
1. [ツール](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)、Model Context Protocol仕様（2025-11-25版）。
1. [トランスポート](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports)、Model Context Protocol仕様（2025-11-25版）。
1. [リスクを表す語彙としてのツールアノテーション](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/)、MCPブログ、2026-03-16。
1. [Issue #76968](https://github.com/anthropics/claude-code/issues/76968)、GitHub anthropics/claude-code、2026-09-14のフォローアップ。
1. [Claude Coworkでスケジュールタスクを作成・編集できない人はほかにもいますか](https://www.reddit.com/r/ClaudeAI/comments/1wwq0p9/anyone_else_unable_to_create_or_edit_scheduled/)、Reddit r/ClaudeAI。
1. [スケジュールタスク／ルーティン：実際にうまく機能しているのはどれですか？](https://www.reddit.com/r/ClaudeAI/comments/1u6ab7d/scheduled_tasks_routines_whats_actually_holding/)、Reddit r/ClaudeAI。
1. [claude.aiのスケジュールタスクで5時間枠を開始させる](https://www.reddit.com/r/ClaudeAI/comments/1x7199x/s_using_claudeai_scheduled_tasks_to_trigger_the/)、Reddit r/ClaudeAI。
1. [リモート（クラウド）でスケジュールタスクを作成できない](https://www.reddit.com/r/ClaudeCowork/comments/1v8tchd/i_cant_create_scheduled_tasks_on_remote_cloud/)、Reddit r/ClaudeCowork。
1. [Coworkモード：スケジュールタスクを実行するモデルを指定する方法はありますか？](https://www.reddit.com/r/Anthropic/comments/1ukail1/cowork_mode_any_way_to_set_which_model_scheduled/)、Reddit r/Anthropic。
1. [EveryInfraデータ機能カタログ](https://api.everyinfra.com/api/v1/social/catalog)、EveryInfra、2026-11-03 00:15（シンガポール時間）閲覧。
1. [商品詳細ページ：Buy Box](https://marketplacelearn.walmart.com/guides/Item%20setup/Item%20content,%20imagery,%20and%20media/Product-detail-page:-the-buy-box)、Walmart Marketplace Learn。
1. [価格設定ルール](https://marketplacelearn.walmart.com/guides/Catalog%20management/Price%20management/Pricing-rules)、Walmart Marketplace Learn、2025-12-12更新。
1. [マーケットプレイスレビューの概要](https://marketplacelearn.walmart.com/guides/Growth%20opportunities/Reviews/Marketplace-reviews:-overview)、Walmart Marketplace Learn。
1. [評価、レビュー、質問と回答の利用規約](https://www.walmart.com/help/article/ratings-reviews-question-and-answer-terms-of-use/0048b13a848c47279cf37023d6ef536d)、Walmartヘルプ。
1. [Walmart、3つの大型ホリデーセールイベントを発表](https://corporate.walmart.com/news/2026/10/walmart-unwraps-three-major-holiday-shopping-events)、Walmart、2026-10-26。
1. [Walmart、ブラックフライデーとサイバーマンデーのセール期間を延長](https://www.retaildive.com/news/walmart-longer-black-friday-cyber-monday-sales-events/872747/)、Retail Dive、2026-10-26。
1. [FTC、偽のレビューと体験談を禁止する最終規則を発表](https://www.ftc.gov/news-events/news/press-releases/2024/08/federal-trade-commission-announces-final-rule-banning-fake-reviews-testimonials)、米連邦取引委員会（FTC）、2024-08-14。
1. [16 CFR Part 465](https://www.ecfr.gov/current/title-16/chapter-I/subchapter-D/part-465)、eCFR。
1. [datetime：基本的な日付型および時間型](https://docs.python.org/3/library/datetime.html)、Pythonドキュメント。
1. [認可](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)、Model Context Protocol仕様（2025-11-25版）。
1. [RFC 6750: The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750)、IETF、2012-10。
1. [statistics：数理統計関数](https://docs.python.org/3/library/statistics.html)、Pythonドキュメント。

---

:::message
この記事は EveryInfra ブログの記事の転載です。原文：[Claude Coworkのスケジュールタスクで、毎週月曜に競合の価格とレビューの一覧表を自動作成する](https://everyinfra.com/ja/blog/claude-cowork-scheduled-tasks-competitor-price-reviews)

EveryInfra は AI エージェントと開発者向けの従量課金データ API です：[everyinfra.com/ja](https://everyinfra.com/ja)
:::
