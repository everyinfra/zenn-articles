---
title: "ChatGPT Spacesで競合のSNSトラッカーを作る：設定、MCP、実地テスト"
emoji: "📊"
type: "tech"
topics: ["mcp", "openai", "api", "ai", "webscraping"]
published: true
---

:::message
この記事は [EveryInfra ブログ](https://everyinfra.com/ja/blog/chatgpt-spaces-competitor-social-tracker) からの転載です（原文が正本です）。
:::

OpenAIはChatGPT Spaceのページを「常に最新に保たれる」ページとして打ち出しています。しかし自社のドキュメントには、Keep Updatedは「リリース時点では利用できない」とあり、毎週更新するにはスケジュールタスクかチームタスクが必要です（[Work with agents in Space](https://learn.chatgpt.com/codex/space/agents)）。2026-11-04のシンガポール時間00:16から00:33にかけて、EveryInfra経由でDuolingo、Babbel、BusuuのTikTok、X、Redditのデータを取得しました。週1回の取得は12回の呼び出しで、API料金は$0.006672、月あたり約$0.0289です。ページの内容が正しいかどうかを左右するのは、1回の呼び出しでどこまで過去にさかのぼれるかです。TikTokの動画50件では55〜272日前までさかのぼれますが、DuolingoのReddit検索1回では4.0時間前までしか届きません。

> **要点**
>
> - ChatGPT SpaceはPro、Business、Enterpriseで利用できます。Libraryを置き換えるもので、Projectsはそのまま残ります。モバイルではページの閲覧と共有はできますが、編集はできません。
> - Keep Updatedはリリース時点では利用できません。毎週の更新はChatGPT Workのスケジュールタスクで行い、このタスクではプラグインを使えます。BusinessまたはEnterpriseでは、チームタスクをチームのサービスアカウントで実行できます。
> - カスタムMCP：Businessでは、作成と公開ができるのは管理者とオーナーのみです。Proには読み取り（read）とフェッチ（fetch）の権限があります。`?key=`付きのEveryInfra URLは2026-11-04に使えなくなったため、ChatGPTでは`https://api.everyinfra.com/mcp/all`にサインインして接続します。
> - 3ブランドで週12回の呼び出し、費用は$0.006672です。DuolingoのReddit検索は4.0時間分しかカバーしないため、Duolingoには`reddit/subreddit_posts`を使います。BabbelとBusuuは投稿が少なく、`reddit/search`の1回の呼び出しで1週間以上をカバーできます。
> - Xでは、Duolingoの最新50件の投稿のうち29件がリプライです。Busuuの最新のX投稿は2025-02-12のものです。集計するのは直近7日間のオリジナル投稿だけにしてください。

このガイドは、英語圏の市場で活動するグロースチームやマーケティングチーム向けです。チームは、ChatGPT上に競合ページを1つ作り、複数のメンバーとChatGPTで最新の状態に保ちたいと考えています。毎週月曜日に、各競合のフォロワー数、投稿数、エンゲージメントが表示されるようにします。

ここでは6つの疑問に順に答えます。まず、ページをどこに置くか、データをどこから取得するか、ChatGPTをデータAPIにどう接続するか。続いて、何が毎週の更新を開始するか、毎月の費用はいくらか、どの数値が誤解を招くかを扱います。

このガイドの作成にあたって、ChatGPTにはサインインしていません。ChatGPTの手順はすべてヘルプセンター、learn.chatgpt.com、開発者向けドキュメントに基づくもので、設定例として記載しています。EveryInfraの数値はすべて実際の呼び出しによるものです。生のレスポンスはすべて保存してあるため、チームの誰でも数値を再確認できます。

![イラスト。左側では、大きさの異なる3つのじょうごが、上から落ちてくる白紙のカードを受け止めています。1つ目のじょうごはあふれ、2つ目は半分ほど、3つ目には数枚しか入っていません。各じょうごの下には定規のようなバーがあり、カードがカバーする期間を示しています。1つ目のバーが最も短く、ティール色です。右側には共有ページがあり、何も入っていない整った表が載っています。じょうごから3本の矢印が表の3つの行へ伸びています。ページの上にある時計の文字盤は、7つの区画のうち1つがティール色です。](https://everyinfra.com/articles/chatgpt-spaces-competitor-social-tracker/img-explain.webp)

*投稿の多いブランドでは1回の呼び出しで数時間分、投稿の少ないブランドでは数か月分をカバーします。週次ページの呼び出し回数は、これを基準に決めてください。*

## ChatGPT Spacesとは：Space、ページ、チーム

まず名称を正しく押さえましょう。製品名は単数形の「Space」です。ヘルプセンターでは、Spaceをページやファイルなど、ChatGPTでの作業を置く場所と定義しています（[Getting started with Space in ChatGPT](https://help.openai.com/en/articles/20001549-getting-started-with-space-in-chatgpt)）。ChatGPTでページを作成・編集し、関連する作業をスペースにまとめて共有します。複数形の「chatgpt spaces」で検索する人もいますが、指しているものは同じです。各アカウントには個人用スペースがあり、トピックやチームごとにスペースを追加できます。

ページは編集可能なドキュメントです。ヘルプセンターでは、プロジェクト概要、議事録、調査のまとめ、チームの進捗報告などを例に挙げています。複数人が同時に1つのページを編集でき、それぞれが自分のChatGPTに手伝ってもらえます。ヘルプセンターには、このガイドにとって重要な一文もあります。ページには、ChatGPTが作成したチャート、トラッカー、インタラクティブなツールを含められます。競合トラッカーは週次の表といくつかのメモ、チャートで構成されるため、1ページに収まります。

OpenAIは2026-09-29のDevDay 2026でSpaceをリリースしました（[DevDay 2026 recap](https://openai.com/index/devday-2026-recap/)）。製品ページには「Pro、Business、Enterpriseプランで利用可能」とあります（[Space product page](https://chatgpt.com/features/space/)）。ヘルプセンターにはさらに詳しい説明があります。Webとデスクトップでは、ページの作成と編集ができます。

モバイルでは、ページの検索、閲覧、共有、サブページ間の移動はできますが、編集はできません。データレジデンシーをカナダまたはUAEに設定しているBusinessおよびEnterpriseのワークスペースでは、Spaceは「近日」提供予定です。これらのプランではSpaceがLibraryを置き換え、Projectsは変わりません（ヘルプセンター 20001549、2026-11-04閲覧）。

| 項目 | ドキュメントの記載 | トラッカーへの影響 |
| --- | --- | --- |
| プラン | Pro、Business、Enterprise（Free、Go、Plusは対象外） | 1人で使う場合でも最低Pro 100（月$100）が必要 |
| 編集 | Webとデスクトップは編集可、モバイルは閲覧と共有のみ | 月曜日はスマートフォンでページを読み、編集はパソコンで行う |
| 置き換え対象 | Library（Projectsは存続） | 既存のProjectsを移行する必要はない |
| 共有の単位 | 1ページまたはスペース全体。サブページはアクセス権を継承 | トラッカーをチームスペースに置けば、新しく加わったメンバーにも表示される |
| ページの内容 | テキスト、表、チャート、トラッカー、ツール、ファイル | テーブル、週次サマリー、グラフを1ページにまとめられる |
| リージョン | カナダとUAEのデータレジデンシーは「近日対応」 | 該当するワークスペースは提供待ち |

チーム機能も同じ日に登場しました。Businessのリリースノートによると、チームを作成して同僚を追加し、ページやスペースをチームと共有できます（[ChatGPT Business release notes](https://help.openai.com/en/articles/11391654-chatgpt-business-release-notes)）。チームではプラグインを共有したり、スケジュールに従って実行されるチームタスクを管理したりすることもできます。チームに関するヘルプセンターのページでは、チームとスペースの違いを明確に説明しています。

チームは誰が一緒に作業するかを定め、スペースはメンバーが共有する作業とコンテキストを保持します（[Teams in ChatGPT](https://help.openai.com/en/articles/20001541-teams-in-chatgpt)）。1つのチームが複数のスペースを持つことも、1つのスペースを複数のチームと共有することもできます。チームはBusinessとEnterpriseのワークスペースにしかないため、個人のProアカウントにはチームもチームタスクもありません。

## ChatGPT SpacesとProjectsの違い：トラッカーをどこに置くか

検索候補で「chatgpt spaces vs projects」が上位に出ることから、両者を混同する人は多いようです。どちらも共有でき、ファイルを保存でき、ChatGPTがその内容を使って回答できます。ただし、整理する対象が異なります。Projectsは関連するチャット、ファイル、指示をまとめて管理します（[Projects in ChatGPT](https://help.openai.com/en/articles/10169521-projects-in-chatgpt)）。Spaceはページとファイルを整理するもので、ページそのものが成果物になります。競合トラッカーの場合、違いは次のとおりです。

| 観点 | Space（ページ） | Projects |
| --- | --- | --- |
| プラン | Pro、Business、Enterprise | 共有はFree、Go、Plus、Pro、Business、Enterprise、Eduで利用可能 |
| 主な対象 | 複数人がリアルタイムで編集するページ | チャット、ファイル、プロジェクトの指示 |
| メンバー | 上限の記載なし。BusinessとEnterpriseではチームで共有 | Freeは5、PlusとGoは10、Proは100。ワークスペースのプロジェクトは最大100 |
| ファイル | 記載なし | Freeは5、PlusとGoは25、Proとワークスペースは40 |
| メモリ | 各自が自分のChatGPTとメモリを使用。ページに書かれた内容はアクセス権を持つ全員が閲覧可能 | 共有プロジェクトはプロジェクト専用のメモリを使い、個人のメモリは参照しない |
| アクセスレベル | 閲覧、コメント、編集（チームへの招待では閲覧と編集のみの場合あり） | チャット、編集 |
| モバイル | 閲覧と共有のみ | Webとモバイルで利用可能 |
| トラッカーに向いているか | 向いている。テーブルがそのまま成果物になり、全員が1つのページを編集する | 毎週の「なぜ下がったのか」を話し合うチャットや、生データのCSVエクスポートに向いている |

Projectsの数値はヘルプセンターの記事10169521に基づいています。Spaceのアクセスレベルは[Collaborate in Space](https://learn.chatgpt.com/codex/space/collaboration)に基づいています。私たちの結論はシンプルです。トラッカーはSpaceに置きましょう。チームが必要としているのはチャットの一覧ではなく、更新されていくテーブルだからです。

毎週の議論は、ページのそばでチャットかページのコメントを使って行います。チームにPlusプランしか持っていないメンバーがいる場合、そのメンバーはSpaceを開けません。その場合は、毎週の数値を共有プロジェクト内のCSVにエクスポートし、PlusのユーザーにはそこでChatGPTに質問してもらいます。

メモリについては、もう少し詳しく見ておく価値があります。競合に関するメモは機密性が高い場合があるからです。メモリがオンの場合、ChatGPTはページの作成を手伝う際に個人のコンテキストを使うことがあります（[ChatGPT Space: sharing, data, and controls](https://help.openai.com/en/articles/20001544-chatgpt-space-sharing-data-and-controls)）。そのテキストがページに載ると、ページを閲覧できる全員が読めるようになります。また、自分がメモリをオフにしていても、メモリをオンにしている共同作業者のChatGPTがページの内容を記憶することがあります。

学習に関する設定もユーザーごとに適用されます。個人プランでは、ページを読み書きしたエージェントを利用するユーザーの設定が適用されます。BusinessとEnterpriseは、デフォルトではデータを学習に使用しません。そのため、個人のProアカウントから同僚をページに招待する場合は、先に各メンバーの「すべての人のためにモデルを改善する」設定を確認するか、Businessワークスペースを使ってください。

もう1つの落とし穴は移行です。個人のdots、Pages、Spacesは、個人ワークスペースをBusinessやEnterpriseに移行しても引き継がれません（[Migrating your personal workspace](https://help.openai.com/en/articles/8801890-migrating-your-personal-workspace-to-chatgpt-business-or-enterprise)）。事前に自分で保存しておく必要があります。たとえば個人のProで1か月試してからBusinessに移る場合は、ページのレイアウトをプロンプトかCSVテンプレートとして残しておきましょう。そして移行後にページを作り直してください。ページが一緒に移行されることは期待できません。

## ChatGPT Spaceのページ：テーブル、Visualizeブロック、Promptブロック

learn.chatgpt.comのガイド[Work with pages](https://learn.chatgpt.com/docs/space/pages)では、ページをブロック単位で説明しています。編集中に`/`を入力すると挿入メニューが開きます。見出し、リスト、チェックリスト、テーブル、コード、ハイライト、画像、ファイル、サブページ、他のページへのリンクを追加できます。ファイルやチャットへの参照や、インタラクティブなグラフを表示する「Visualize」ブロックも追加できます。

エージェント向けのブロックは、Prompt、Task、Agent Instructionsの3つです。Promptブロックにはリクエストを保存でき、読み手がクリックするとチャットが始まります。テーブルブロックの幅は標準、フレキシブル、全幅から選べ、列幅は内容に合わせて調整できます。

これらのブロックはトラッカーにうまく当てはまります。一番上のテーブルブロックに今週の数値を入れます。その下のVisualizeブロックで4週間の推移を表示します。さらにその下のPromptブロックに、「このページのルールに従って今週のデータを取得し、テーブルを更新して」というリクエストを保存します。

一番下には週ごとに1つずつサブページを置き、生データのサマリーを残します。結論はテーブルに、根拠はサブページにまとめます。チームメンバーが数値に疑問を持ったら、その週のサブページを開けば、データ取得時に返された投稿を確認できます。

![learn.chatgpt.comにあるOpenAIのドキュメントページ「Work with agents in Space」の「Verify recurring work separately」セクションのスクリーンショット。青緑色の枠で2つの箇所を囲んでいます。1つ目の箇所には、Keep Updatedはリリース時点では利用できないこと、定期的な作業を設定するにはチャットでChatGPTにスケジュールの作成を依頼し、保存されたスケジュールを確認すること、ページや指示に書かれた実行間隔はタスクが存在する証拠にはならないことが書かれています。続く段落には、設定後にタスク、時刻、有効状態を確認し、その後、実際の実行1回分とページ上の結果を確認するよう書かれています。](https://everyinfra.com/articles/chatgpt-spaces-competitor-social-tracker/img-doc-keep-updated.webp)

*OpenAIドキュメントのスクリーンショット（シンガポール時間2026-11-04 00:30に取得）。青緑色の枠は当社が追加したものです* [learn.chatgpt.com、Work with agents in Space](https://learn.chatgpt.com/codex/space/agents)

「ページは常に最新の状態に保たれる」という説明は慎重に受け止めてください。製品ページには「接続したツールからページを常に最新の状態に保てる」とあります（[Space product page](https://chatgpt.com/features/space/)）。一方、同じドキュメント群にあるエージェントのガイドははっきりと書いています。Keep Updatedはリリース時点では利用できません。

定期的な作業を設定するには、チャットでChatGPTにスケジュールの作成を依頼し、保存されたスケジュールを確認します。ページに「毎週月曜日に更新」のような一文があっても、タスクが存在する証拠にはなりません（Work with agents in Space、2026-11-04閲覧）。そのため、このガイドでは更新はスケジュール済みのタスクが行い、ページは結果を置く場所としてのみ使います。

通常のChatでページを編集できるのでしょうか。OpenAIは明言していません。ただし、SlidesはWorkで作成すると説明しています（[Creating and collaborating on slides in ChatGPT](https://help.openai.com/en/articles/20001605-creating-and-collaborating-on-slides-in-chatgpt)）。また2026-10-01には、通常のChatでは添付したページを読むことしかできず、Workでは編集できたとコミュニティのユーザーが報告しています（[Pages in Spaces for normal ChatGPT chats](https://community.openai.com/t/pages-in-spaces-for-normal-chatgpt-chats/1402509)）。そのため、以下の設定ではWorkでページを編集します。

## トラッカーで集計する項目と各数値の取得元

何よりも先に列を決めてください。列によって、毎週どの呼び出しを行うかが決まるからです。ここでは、ブランドとプラットフォームの組み合わせごとに1行を使います。列には、1回の呼び出しで正確に得られる数値だけを入れます。下の表は、ページ上の表ブロックのレイアウトです。右の列には、各数値がEveryInfraのどの機能とフィールドから得られるかを示しています。すべてのフィールドは、[TikTokデータAPI](https://everyinfra.com/ja/tiktok-api)、[XデータAPI](https://everyinfra.com/ja/twitter-api)、[RedditデータAPI](https://everyinfra.com/ja/reddit-api)のカタログで確認できます。

| 列 | 意味 | 取得元フィールド |
| --- | --- | --- |
| フォロワー数 | 月曜日の取得時点のフォロワー数 | `tiktok/profile` → `follower_count`。Xでは各`twitter/user_posts`行の`author_follower_count` |
| 直近7日間の新規投稿数 | 取得前の7日間に公開された投稿。TikTokでは固定表示の動画を除外 | `posted_at`、`is_pinned` |
| 直近7日間のオリジナル投稿数（X） | 返信とリポストを除外 | `is_reply`、`is_repost` |
| 再生数の中央値 | 直近7日間の新規投稿の再生数の中央値。新規投稿がない場合は「なし」 | `view_count` |
| エンゲージメント率の中央値 | (いいね + コメント + シェア) / 再生数 | `like_count`、`comment_count`、`share_count` |
| 今週のトップ投稿 | リンクと再生数 | `url`、`view_count` |
| Redditでの話題 | 話題の多いブランド：今週のブランドコミュニティの上位投稿。話題の少ないブランド：7日間の言及数 | Duolingo：`reddit/subreddit_posts`。Babbel、Busuu：`reddit/search` |
| 備考 | 人が執筆、または@ChatGPTが下書きして人が編集 | ページのコメント |

なぜ平均値ではなく中央値なのでしょうか。Duolingoは直近7日間にTikTok動画を5本投稿しました。最も多い動画の再生回数は2.4Mで、中央値は803,300でした。大きくヒットした動画が1本あると平均値が引き上げられ、週ごとの数値が大きく上下してしまいます。

TikTokで固定動画を除外する理由は何でしょうか。私たちのテストでは、どのブランドでも「newest」の先頭の動画は固定動画でした。Duolingoの固定動画は2025-06-10のものです。これを残すと、「最新の投稿」が1年以上ずれてしまいます。

## データ：4つの機能、パラメータ、料金

トラッカーは4つの機能に加え、Redditのコミュニティ用にもう1つの機能を使います。シンガポール時間の2026-11-04 00:16にカタログを確認したところ、5つすべてが`available: true`で、料金は1回の呼び出しあたり$0.000556でした（[カタログ](https://api.everyinfra.com/api/v1/social/catalog)は無料で閲覧できます）。課金は呼び出しごとで、1回の呼び出しで最大`max_limit`行が返ります。行数によって料金は変わりません。失敗した呼び出しや結果が空の呼び出しには料金がかかりません。

| 機能 | 必須 | 便利なオプション | 1回あたりの上限 | 料金 | 用途 |
| --- | --- | --- | --- | --- | --- |
| [`tiktok/profile`](https://everyinfra.com/ja/tiktok-api/profile) | `username` | — | 1アカウント | $0.000556 | フォロワー数、総いいね数、動画数 |
| [`tiktok/user_videos`](https://everyinfra.com/ja/tiktok-api/user_videos) | `username` | `sort`（newest/oldest/popular）、`until`、`min_likes` | 50件 | $0.000556 | 新着動画、再生数、エンゲージメント |
| [`twitter/user_posts`](https://everyinfra.com/ja/twitter-api/user_posts) | `username` | — | 200件 | $0.000556 | オリジナル投稿、表示回数、いいね数 |
| [`reddit/search`](https://everyinfra.com/ja/reddit-api/search) | `keyword` | `sort`（newestを含む） | 25件 | $0.000556 | ブランドに関する最新の言及 |
| [`reddit/subreddit_posts`](https://everyinfra.com/ja/reddit-api/subreddit_posts) | `subreddit` | `sort`、`date_range` | 100件 | $0.000556 | コミュニティ内の今週の人気投稿 |

ChatGPTを使わずにテストするには、curlでAPIを呼び出します。APIキーはリクエストヘッダーに入れ、コードリポジトリには絶対にコミットしないでください。

```bash
# Example: Babbel's newest 50 TikTok videos (real call at 00:18 Singapore time, 2026-11-04, charged_usd 0.000556)
curl -s https://api.everyinfra.com/api/v1/social \
  -H "Authorization: Bearer $EVERYINFRA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"platform":"tiktok","action":"user_videos","params":{"username":"babbel","sort":"newest","limit":50}}'
```

カタログにある各機能の`field_note`も読んでください。トラッカーの動作に関わる注記が2つあります。`twitter/user_posts`の注記によると、他のソースが失敗した場合、結果はフォールバックソースから返されます。このソースには返信が含まれないため、`is_reply`と`reply_to`は空になります。それも失敗した場合は、プロフィール文やフォロワーのフィールドが欠ける可能性があり、最後のフォールバックが返すのは最大60件です。

`reddit/search`の注記によると、期間は直近1か月に固定されており、並び順を変えてもこの期間は変わりません。キーワードに一致する投稿だけを返すため、取得できる行数が`limit`より少なくなることがあります。一致する投稿がない場合、その呼び出しは空の結果として扱われ、返金されます。つまり、Xの「オリジナル投稿」が正確なのは返信フィールドに値がある場合だけで、Redditの検索は1か月より前にはさかのぼりません。

## ChatGPTでEveryInfraのリモートMCPに接続する：個人アカウントとBusinessの違い

要点をまとめると、Businessでは管理者に依頼してEveryInfraをワークスペースのプラグインとして追加してもらいます。個人のProアカウントでは、まず手動で1回呼び出し、ChatGPTが毎回承認を求めるかどうかを確認してから、タスクをスケジュールします。ChatGPTでは現在、外部ツールを「プラグイン」と呼んでいます（[プラグインの製品ページ](https://chatgpt.com/features/plugins/)）。ワークスペースには、プラグインの使用、MCPによるプラグインの作成（Enterpriseではデフォルトでオフ）、ワークスペースへの公開といった設定があります（[Plugins in ChatGPT](https://help.openai.com/en/articles/20001256-plugins-in-chatgpt)）。

カスタムMCPサーバーの手順は[Connecting custom MCP servers in ChatGPT](https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt)に記載されています。2026-11-04に確認した時点で、このページには「Updated 2 days ago」と表示されていました。「Plugins」を開き、プラス記号をクリックして「Add custom MCP server」を選択します。名前と説明を入力します。

「Connection」で公開されているHTTPSのMCP URLを入力するか、Secure MCP Tunnelを使う場合は「Tunnel」を選択します。認証を設定し、「I understand and want to continue」にチェックを入れます。次に「Create as a plugin」を選択し、検出されたツールとメタデータを確認します。

開発者向けガイド[Connect and test your plugin](https://developers.openai.com/plugins/deploy/connect-chatgpt)にも同じ手順が記載されています。このガイドでは、事前にMCP Inspectorですべてのツールをテストすることも勧めています。非公開のMCPサーバーでは[Secure MCP Tunnels](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)を利用できます。EveryInfraのサーバーは公開URLを持つため、トンネルは不要です。

誰が設定できるかはプランによって異なります。ヘルプセンターに詳しく記載されているため、表にまとめました。

| アカウント | カスタムMCP対応 | 作成・公開できる人 | トラッカーへの影響 |
| --- | --- | --- | --- |
| Pro（個人） | 対応（「read/fetch権限付き」）。フルMCPはBusinessおよびEnterprise/Edu向け | 本人 | データツールは読み取り専用としてマークされていません。Proで呼び出しのたびに承認を求められるかテストしてください |
| Business | 対応（フルMCP） | 作成・公開できるのは管理者とオーナーのみ | メンバーは管理者に依頼し、EveryInfraをワークスペースのプラグインとして追加してもらいます |
| Enterprise / Edu | 対応（フルMCP） | 管理者がロールベースのアクセス制御でテスト権限を付与します。公開できるのは管理者とオーナーのみです | 必要なツールだけを公開できます |
| モバイル | 不可。Webで設定 | — | 設定はパソコンで行う |

Proに関する記述は、ヘルプセンターの記事12584461にあるFAQ「Are apps and full MCP beta available to Pro users?」に基づいています。ProユーザーはApps SDKでアプリを構築できます。フルMCPは現時点でBusinessとEnterprise/Eduのみで利用できます。Proユーザーは、読み取り（read）と取得（fetch）の権限でカスタムMCPサーバーに接続できます。これは、データツールに設定されたアノテーションと矛盾します。

シンガポール時間00:16に、`https://api.everyinfra.com/mcp`へ無料の`tools/list`リクエストを1回送信しました。12個のツールのうち、データツール`everyinfra_call_api`は`readOnlyHint: false`、`destructiveHint: true`、`openWorldHint: true`となっています。MCP仕様では、`readOnlyHint`はツールが環境を変更しないことを意味します（[MCP spec, ToolAnnotations](https://modelcontextprotocol.io/specification/2025-11-25/schema#toolannotations)）。データを取得しても、TikTok上では何も変わりません。

ただし、呼び出すたびに残高から料金が差し引かれるため、当社のサーバーはこのツールを読み取り専用として指定していません。OpenAIの記述から、Proではデータを呼び出すたびに承認を求められる可能性が高いと推測しています。スケジュールされたタスクがこの確認で止まるかどうかはドキュメントに記載がなく、ChatGPT上でも検証していません。

このセクションで最も大きな変更点は認証です。開発者ガイド[ChatGPT Developer mode](https://developers.openai.com/api/docs/guides/developer-mode)には、MCPの認証方式としてOAuth、認証なし、混合の3つが挙げられています。混合とは、サインイン不要のツールとOAuthを使うツールが併存する方式です。詳細はApps SDKの[Authentication](https://developers.openai.com/apps-sdk/build/auth)ガイドに記載されています。

OAuthはバージョン2.1を使用します。ChatGPTはClient ID Metadata Document（CIMD）で自身を識別でき、サーバーは所定のURLでOAuth設定を公開します。そのため、ChatGPTには「APIキーをヘッダーに貼り付ける」ための欄がありません。従来の回避策は、「認証なし」を選び、キーを`?key=`付きのURLに含める方法でした。

この方法は現在使えません。2026-11-04のシンガポール時間00:28に、URLにキーを含めて`tools/call`を送信したところ、サーバーは401を返しました。メッセージは「The EveryInfra API key in this connector's URL (?key=) stopped working on 2026-11-04」でした。

メッセージには新しいURLが2つ示されていました。キーをヘッダーで渡す`https://api.everyinfra.com/mcp`と、サインインを使う`https://api.everyinfra.com/mcp/all`です。この401では料金は発生しませんでした。ChatGPTはキーをヘッダーに入れてくれないため、ChatGPTで使えるのは2つ目のURLだけです。

シンガポール時間00:28から00:29にかけて、`/mcp/all`のサインイン設定を確認しました。3回のリクエストはいずれも無料でした。まず、認証情報なしで送った`initialize`は401を返しました。`WWW-Authenticate`ヘッダーには`resource_metadata`のURLが含まれていました。これは、MCPの認可仕様が求める検出方法です（[MCP authorization](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)）。

次に、そのURLは[RFC 9728](https://datatracker.ietf.org/doc/html/rfc9728)で定義された保護リソースメタデータを返します。このメタデータは、認可サーバーとして`https://auth.everyinfra.com/oidc`を指定し、`data:call`、`search:call`、`account:read`、`offline_access`などのスコープを列挙しています（[メタデータ](https://api.everyinfra.com/.well-known/oauth-protected-resource/mcp/all)）。最後に、OpenIDの構成情報には`client_id_metadata_document_supported: true`とあり、`offline_access`も記載されています。PKCEでは`S256`のみを受け付けます。

`offline_access`スコープは、ChatGPTの要件に対応しています。ヘルプセンターによると、OAuthを使う場合、IDプロバイダーはリフレッシュトークンを発行する必要があります。OpenID Connectでは、`offline_access`をリクエストし、`scopes_supported`に記載します。リフレッシュトークンがないと、最初の認可の有効期限が切れた時点でChatGPTの接続が切れ、ユーザーが再度サインインしなければならない場合があります。

CIMDはIETFのドラフトに由来し、2026-11-02にバージョン-03が公開されました（[IETF draft](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/)）。以下は、これらのドキュメントに基づく設定例です。ChatGPT上で実際に操作して確認したものではありません。

```text
# Configuration example: ChatGPT web → Plugins → + → Add custom MCP server
Name:           EveryInfra social data
Description:    Public TikTok, X and Reddit data; each call billed at catalog price, failures free
Connection:     https://api.everyinfra.com/mcp/all
Authentication: OAuth (ChatGPT identifies itself with CIMD; sign in to EveryInfra and approve)
[x] I understand and want to continue
→ Create as a plugin → check the tools it finds (everyinfra_call_api, everyinfra_list_capabilities, everyinfra_get_job ...)

# Business workspace: only an admin or owner can create it. Then publish in
# Admin Console → Plugins → Manage Legacy Apps → Drafts → Publish, and share it with the tracker's team.
```

サインインにはもう1つ利点があります。キーをChatGPTに渡す必要がありません。当社のサーバーは、ユーザー、クライアント、エントリーポイントの組み合わせごとに内部キーを1つ作成します。課金、返金、レート制限、呼び出しログはそのアカウントに紐づき、接続を解除するとキーは無効になります。詳しくは[MCPサーバーのページ](https://everyinfra.com/ja/mcp-server)と[リモートMCPセットアップガイド](https://everyinfra.com/ja/build/remote-mcp-setup)をご覧ください。Businessワークスペースでは、サインインした人のEveryInfra残高から支払われます。同僚の個人アカウントではなく、チームで共有するEveryInfraアカウントでサインインしてください。

最後に、長期的な安定性に関わるワークスペースのルールがもう1つあります。ヘルプセンターのFAQによると、管理者がMCPサーバーを初めて承認すると、ChatGPTはその時点のツール一覧を固定したスナップショットを使います。その後のサーバー側の変更は、管理者が再度レビューして公開するまで反映されません。実際のツールとスナップショットが異なると、呼び出しが失敗することがあります。新しい任意パラメータの追加など、互換性のある変更であれば、引き続き動作する場合もあります。

互換性のない変更の場合は、管理者がワークスペース設定でプラグインを更新する必要があります。このとき、ChatGPTは更新を促すことも、管理者に通知することもありません。当社のツール名は増えることはあっても削除されることはないため、古い名前は引き続き使えます。ただし、フィールドは追加されます。壊れた実行をデバッグするより、Admin Consoleで毎月更新するほうが手間がかかりません。

## 実際の1往復：ChatGPTが送るリクエストと返ってくるレスポンス

ChatGPTがプラグイン経由で当社のサーバーを呼び出すと、標準的なMCPの`tools/call`リクエストが送信されます。2026-11-04のシンガポール時間00:29に、同じリクエストを`https://api.everyinfra.com/mcp/all`に送信しました。今回は当社側のキーを使いましたが、OAuthで接続した場合もツールとレスポンスは同じです。リクエストの内容は、DuolingoのTikTok最新動画10件の取得です。レスポンスは16.78秒で返り、`charged_usd`は0.000556でした。

```text
{"jsonrpc":"2.0","id":7,"method":"tools/call",
 "params":{"name":"everyinfra_call_api",
           "arguments":{"platform":"tiktok","action":"user_videos",
                        "params":{"username":"duolingo","sort":"newest","limit":10}}}}
```

![EveryInfraのAPIから返された実際のレスポンスのスクリーンショットで、ダークテーマのコードカードとして表示されています。最上段には、POST https://api.everyinfra.com/mcp/all、tools/call、00:29:28 SGT、16.78秒、24,746バイトと表示されています。リクエスト部分には、everyinfra_call_apiがplatform tiktok、action user_videos、username duolingo、sort newest、limit 10で呼び出されている様子が示されています。レスポンスの抜粋にはcount 10と表示されています。1件目は2025-06-10に投稿された固定表示の動画で、is_pinnedがtrue、再生回数は390万回です。2件目は2026-11-02の投稿で、再生回数は116,800、いいね数は8,274です。注記には残り8件（各35フィールド）とあり、課金欄にはcharged_usd 0.000556と表示されています。](https://everyinfra.com/articles/chatgpt-spaces-competitor-social-tracker/img-api-response.webp)

*実際のMCPレスポンスのスクリーンショット：「newest」の1件目は1年以上前のピン留め動画* [EveryInfra MCPレスポンス](https://api.everyinfra.com/mcp/all)

`content[0].text`フィールドには24,746バイトのJSON文字列が入っています。10本の動画それぞれに35のフィールドがあります。再生数、いいね数、コメント数、シェア数、保存数のほかに、カバー画像のURL、音楽、ハッシュタグ、広告フラグ、商品フラグ、投稿者のフォロワー数があります。スクリーンショットは、このテストで最初に見つかった意外な点を示しています。`sort: newest`を指定しても、1件目は2025-06-10のピン留め動画で、2件目が2026-11-02の新しい動画です。トラッカーが1件目を最新の投稿とみなすと、1年以上ずれます。

この1往復の後、ChatGPTの手元には構造化されたJSONがあります。次に、ページ上のルールに従って集計する必要があります。`is_pinned`がtrueの行を除外し、`posted_at`が直近7日以内の行を残して、中央値を取ります。これらのルールは、Promptブロックまたはタスクの指示に記述してください。毎週ChatGPTに推測させるより安定します。

## 週次更新の仕組み：スケジュールタスクとチームタスク

ページは自動では更新されないため、タスクで更新する必要があります。アカウントの種類に応じて、方法は2つあります。ドキュメントの内容を次の表にまとめました。

| 観点 | 個人：ChatGPT Workのスケジュールタスク | Business / Enterprise：チームタスク |
| --- | --- | --- |
| 作成場所 | チャットでChatGPTに依頼。Scheduledで確認可能 | チームページのScheduledエリア |
| プラグイン | 対応。WebのWork、またはデスクトップのWorkやCodexで作成したタスクはプラグインを使用可能 | 対応。ワークスペースがチーム向けに設定した接続を通じて、チームで共有するプラグインを使用可能 |
| 実行主体 | 自分のアカウントと接続 | チームのサービスアカウント。タスクは作成者のメモリ、カスタム指示、チャット履歴を引き継がない |
| 実行タイミング | 毎日、毎週、またはカスタム。上級ユーザーはRRULEも記述可能 | 繰り返しは「Schedule」、1回のみは「Date and time」。タスクごとにタイムゾーンを設定 |
| オンラインである必要があるか | Webでは不要。デスクトップでは、ローカルファイルを使うタスクはパソコンの電源が入っている必要あり | 不要。クラウドで実行 |
| 初回サインイン | 先にチャットでプラグインにサインイン | 無人実行では新規のサインインを完了できないため、事前にサインイン |
| 支出上限 | 個人プランの利用枠 | 管理者による支出管理。個人のタスク上限は適用されない |

左列はlearn.chatgpt.comの[Scheduled tasks](https://learn.chatgpt.com/codex/automations)、右列はヘルプセンターの記事[Creating and managing team tasks in ChatGPT](https://help.openai.com/en/articles/20001540-creating-and-managing-team-tasks-in-chatgpt)に基づいています。この記事には、トラッカーにとって重要な記述が4つあります。スペースをチームと共有しても、スペース内のすべてが各実行に含まれるわけではありません。指示の中で参照元を明記してください。

スケジュールの「Completed」は、残りの実行がないことを示すだけで、実行が成功したことを意味しません。一時的なエラーの中には自動で再試行されるものもありますが、実行中に失敗したアプリのアクションは再試行されない場合があります。ファイルを編集するタスクを再実行する前に、前回の実行ですでに何が行われたかを確認してください。

![フローのアニメーション（8秒ごとに1ループ）。上部の破線の枠は初回のみの設定を示します。管理者がEveryInfraをプラグインとして追加し（/mcp/all、サインイン）、スペースとトラッカーページを作成して、スケジュールタスクまたはチームタスクを作成します。6つの枠が順に点灯します。1 トリガー：毎週月曜09:00、タイムゾーンはタスクで指定。2 プラグインを呼び出す：@EveryInfra、everyinfra_call_api。3 データを取得：3ブランド×4機能＝12回の呼び出し。4 週次の集計：ピン留めを除外し、直近7日間のみ。5 ページを更新：表と週次サマリーのみ。6 チームでレビュー：コメントで@ChatGPTに質問。中央には「API費用 1ラウンドあたり$0.006672、12回の呼び出し、ChatGPTプランは別途課金」と表示されています。](https://everyinfra.com/articles/chatgpt-spaces-competitor-social-tracker/fig-02-weekly-flow.ja.light.svg)

毎週ページに正確な数値が入るかどうかは、タスクの指示で決まります。私たちは以下のように書きました。これをページ上部のPromptブロックに入れ、同じ文章をスケジュールタスクにも貼り付けます。2つのコピーは常に同じ内容にしてください。各ルールは、この後のテストで実際に直面した問題への対処です。

```text
# Task instructions example (every Monday 09:00, America/New_York)
Use @EveryInfra social data. Call once per brand for Duolingo, Babbel and Busuu, in this order, one call at a time:
1. tiktok/profile {username}; tiktok/user_videos {username, sort: newest, limit: 50}
2. twitter/user_posts {username, limit: 50}
3. Reddit. Duolingo: reddit/subreddit_posts {subreddit: duolingo, sort: top, date_range: past_week, limit: 25}
   Babbel and Busuu: reddit/search {keyword: brand name, sort: newest, limit: 25}
Counting rules:
- TikTok: drop videos where is_pinned is true; keep only posted_at within 7 days before the run
- X: drop posts where is_reply or is_repost is true, then count original posts in the last 7 days
- Reddit search: count only posts with posted_at in the last 7 days; count posts with the brand in the title or community name separately
- Use the median, not the mean. Views and likes over 10,000 are rounded by the platform; call a change under 1% "flat"
- On 429, wait retry_after seconds and try once more. On 422, fix the parameter from did_you_mean
Output: update only the "This week" table and the three-line weekly summary; put the raw rows in a subpage named after this week's date
```

なぜ「1回ずつ呼び出す」のでしょうか。このテストで使ったキーは1分あたり10回まで呼び出せます。1ラウンド目では、シンガポール時間の00:17:48と00:17:55に429が返されました。どちらのレスポンスにも`retry_after: 12`と「Nothing was charged」が含まれていました。ChatGPTは1つのタスク内で複数のツール呼び出しを同時に送ることがあります。同じ1分間に12回呼び出すと、レート制限に達します。

## テスト1：1回の呼び出しでどこまで遡れるか

テストの前に基準を決めました。週次トラッカーでは、1回の呼び出しで少なくとも7日間をカバーできないと、「直近7日間の新規投稿数」が実際より少なくなります。3つのプラットフォームの結果は大きく異なりました。

| ブランド・プラットフォーム | 返却行数 | 最古の行（取得時点での経過時間） | 直近7日間の行数 | 週次に十分か |
| --- | --- | --- | --- | --- |
| Duolingo · TikTok | 49（ほかにピン留め1件） | 55日 | 5 | 可能 |
| Babbel · TikTok | 49（ほかにピン留め1件） | 85日 | 1 | 可能 |
| Busuu · TikTok | 47（ほかにピン留め3件） | 272日 | 1 | 可能 |
| Duolingo · X | 50 | 12.8日 | 28（うちオリジナル投稿10） | はい（余裕は少ない） |
| Babbel · X | 50 | 131日 | 2 | 可能 |
| Busuu · X | 50 | 1,129日 | 0 | はい（ただしアカウントはほぼ休眠状態） |
| Duolingo · Reddit検索 | 23 | 4.0時間 | 23 | 不要 |
| Babbel · Reddit検索 | 23 | 9.1日 | 17 | ぎりぎり十分 |
| Busuu · Reddit検索 | 23 | 15.6日 | 9 | 可能 |

![対数目盛（1目盛ごとに10倍）の横棒グラフ。1回の呼び出しでどこまで遡れるかを9本の棒で示しています。Duolingo：TikTokは動画50本で55日、Xは投稿50件で13日、Reddit検索は23件でわずか4.0時間で、これが唯一のティール色の棒です。Babbel：TikTok 85日、X 131日、Reddit 9.1日。Busuu：TikTok 272日、X 1,129日、Reddit 16日。投稿の多いブランドではReddit検索が数時間分しかカバーしないため、週1回の取得ではデータを取りこぼします。](https://everyinfra.com/articles/chatgpt-spaces-competitor-social-tracker/fig-03-coverage-bar.ja.light.svg)

最も意外だったのはDuolingoのReddit検索です。新しい順に25件をリクエストしたところ、返ってきたのは23件でした。最も古い投稿でもわずか4.0時間前のもので、2026-11-03の12:23から16:05（UTC）の範囲に収まっていました。つまり、4時間ごとに20件以上のReddit投稿がDuolingoに言及していることになります。週1回の取得では、月曜朝のこの数時間分しか見えません。

このペースだと、1週間分をすべて取得するには約40回の呼び出しを連続で行う必要があります。また、`reddit/search`は直近1か月に固定されており、1回の呼び出しで返すのは最大25件です。1週間分の言及数を数えるのは得意ではありません。BabbelとBusuuは投稿がずっと少なく、1回の呼び出しでそれぞれ9.1日分と15.6日分をカバーできます。

この結果には限界があります。サンプルは11月3日から4日にかけての1回分の取得で、週末やローンチの週は異なるはずです。23件は、キーワードに一致しない投稿を除いた残りで、カタログにも件数は`limit`を下回ることがあると書かれています。それでも傾向は変わりません。同じ機能でも、小さなブランドでは大きなブランドの50倍の期間をカバーできます。ブランドごとに呼び出し回数を設定し、すべてのブランドに同じ手順を当てはめないでください。

Duolingoについては、ブランドのコミュニティに切り替えました。シンガポール時間の00:28に、`reddit/subreddit_posts`でr/duolingoの今週のトップ25投稿を取得しました（`sort: top`、`date_range: past_week`）。1回の呼び出しで、2026-10-27から2026-11-02（UTC）までの6.1日間をカバーしました。25件の投稿のコメント数は合計1,528件で、トップ投稿は2026-10-29のもので、1,615件のアップボートを獲得していました。ここから、ブランド自身のコミュニティで今週何が議論されているかがわかります。ブランド名を挙げた人の数よりも有用で、呼び出しも1回で済みます。

BabbelとBusuuではこの変更は不要です。1回の検索で、Babbelは9.1日、Busuuは15.6日をカバーします。最も古い行はそれぞれ2026-10-25と2026-10-19（UTC）のものです。1週間を超えているため、`reddit/search`を1回呼び出すだけで直近7日間の言及をすべて数えられます。Babbelは17件、Busuuは9件でした。

そのため、週次のレシピではブランドごとにRedditの機能を使い分けます。Duolingoにはコミュニティのトップ一覧、BabbelとBusuuには検索を使います。各ブランドのReddit呼び出しは1回のままなので、合計は12回に収まります。2つの機能は数えている対象が異なります。これらのRedditの行を直接比較しないでください。各行がどちらの種類かをメモに記載してください。

英語圏のチームであれば、1回の呼び出しを割く価値のあるコミュニティがもう1つあります。シンガポール時間の00:33に、このカテゴリの総合フォーラムであるr/languagelearningで今週のトップ25投稿を取得しました。1回の呼び出しで2026-10-27から2026-11-03（UTC）までの6.6日間をカバーし、コメント数は合計732件、トップ投稿のアップボート数は481件でした。カテゴリのスレッドからは、特定のブランドに直接向けた話題でないときに、ユーザーがどのアプリを比較しているかがわかります。トラッカーにこのコミュニティ用の行を1つ追加し、トップ投稿のタイトルに登場するブランドをChatGPTに列挙させてください。

## テスト2：TikTokのピン留め動画、丸められた数値、is_ad

`sort: newest`でブランドごとに50本の動画を取得しました。3ブランドとも、ピン留め動画が先頭に来ました。Duolingoは1本（2025-06-10）、Babbelは1本（2025-07-11）、Busuuは3本で、最も古いものは2022-11-17でした。ピン留めの行の後は、残りがすべて厳密に新しい順に並んでおり、全行を確認しました。ルールはシンプルです。まず`is_pinned`を確認し、次に`posted_at`で絞り込みます。

| ブランド | ピン留めの行数 | ピン留め以外の行の遡及期間 | 7日間の投稿数 | 再生数の中央値（最新50本） | エンゲージメントの中央値 | `is_ad`が付いた行 |
| --- | --- | --- | --- | --- | --- | --- |
| Duolingo | 1 (2025-06-10) | 55日 | 5 | 660,600 | 14.1% | 50件中42件 |
| Babbel | 1 (2025-07-11) | 85日 | 1 | 35,700 | 3.26% | 50件中0件 |
| Busuu | 3件（最古は2022-11-17） | 272日 | 1 | 2,769 | 1.57% | 50件中1件 |

2つ目の発見は数値の丸めです。Duolingoの動画50本はすべて再生数が10,000を超えており、50本すべての再生数が3,900,000や116,800のように丸められていました。有効数字が4桁を超えるものは1本もありませんでした。10,000回を超えるBabbelの動画43本も、すべて丸められていました。

私たちは何も丸めていません。TikTokはWeb上で「3.9M」と表示しており、再生数もその形で届きます。その結果、数百万回再生の動画は100,000単位でしか変化しません。

シンガポール時間の00:16と00:29に、同じ動画のデータを取得しました。13分間で再生回数は2.4Mから2.5Mに増えました。これでは50,000回増えたのか140,000回増えたのか判別できません。別の動画は116,100から116,800に増えており、こちらは明確に700回増えています。そのため、タスクの指示では「1%未満の変化は横ばいとみなす」としています。

3つ目の発見は`is_ad`です。Duolingoは50本中42本が`is_ad: true`で、Babbelは0本、Busuuは1本でした。カタログではこのフィールドを「動画が広告かどうか」と説明しています。ブランド自身の動画にこのフラグが付いている場合、よくある理由は、同じ動画が有料広告としても配信されたことです。動画ごとの理由は確認していません。そのため、トラッカーではこれを備考列として表示し、再生数の調整には使っていません。

エンゲージメントも大きく異なります。固定されていない動画のエンゲージメント率の中央値は、Duolingoが14.1%、Babbelが3.26%、Busuuが1.57%でした。再生数の中央値は660,600、35,700、2,769です。直近7日間では、BabbelとBusuuがそれぞれ1回ずつ投稿しており、再生数は9,377と696でした。「直近7日間」の列しかないと、小規模なブランドは1件のサンプルで数値が大きく振れてしまいます。そこで「最新50本の中央値」を参考列として残しています。

## テスト3：XでのDuolingoの投稿50件のうち、過半数がリプライ

`twitter/user_posts`を1回呼び出すと、50件の投稿が返りました。Duolingoはリプライが多いため、この50件はわずか12.8日分でした。内訳はリプライが29件、リポストが4件です。オリジナル投稿17件のうち10件は引用投稿でした。オリジナル投稿だけを数えると、Duolingoは直近7日間に10件投稿しています。その中央値は表示回数56,364、いいね1,077でした。

Babbelの50件は、リプライが12件、オリジナル投稿が38件でした。50件に達するまでに131日分さかのぼり、直近7日間の投稿は2件だけでした。オリジナル投稿のいいね数の中央値は1です。Busuuの最新投稿は2025-02-12のもので、直近7日間の投稿は0件でした。

| アカウント | フォロワー数 | リプライ | リポスト | オリジナル（引用を含む） | 50件でさかのぼれる期間 | 7日間のオリジナル投稿 |
| --- | --- | --- | --- | --- | --- | --- |
| @duolingo | 1,049,014 | 29 | 4 | 17 | 12.8日 | 10 |
| @babbel | 34,286 | 12 | 0 | 38 | 131日 | 2 |
| @busuu | 35,699 | 3 | 1 | 46 | 1,129日 | 0 |

![積み上げ棒グラフ。50件ずつの投稿を示す3本の棒。Duolingo：引用を含むオリジナル投稿17件（ティール）、リプライ29件、リポスト4件。Babbel：オリジナル38件、リプライ12件。Busuu：オリジナル46件、リプライ3件、リポスト1件。サブタイトルには、Busuuの最新投稿が2025年2月のもので、アカウントはほぼ投稿を停止していると記されています。](https://everyinfra.com/articles/chatgpt-spaces-competitor-social-tracker/fig-04-x-mix-bar.ja.light.svg)

これらの数字からわかることは2つあります。1つ目に、投稿数にリプライを含めると、DuolingoはBabbelの14倍活発に見えます。7日間で28件対2件です。リプライを除くと10件対2件になります。リプライの多さから、DuolingoはXを主にユーザーとの対話に使っていることがわかります。

これはメモに1行書いておく価値がありますが、「どれだけコンテンツを投稿しているか」には含めるべきではありません。第2に、アカウントが数か月更新されていない場合、トラッカーは「新しい投稿なし」と表示すべきです。2025年の投稿から中央値を計算してはいけません。当社のタスク指示では、直近7日間にオリジナル投稿がない場合は「none」とするよう指定しています。

フォールバックソースにはもう1つ制約があります。カタログによると、`twitter/user_posts`がフォールバックを使う場合は返信が含まれず、`is_reply`は空になります。その週は、Duolingoの50件がすべてオリジナル投稿になります。カバー範囲は広がり、「7日間のオリジナル投稿数」はより正確になります。一方で、返信の割合は計算できません。そのため、返信の割合はメモに記載し、週ごとに比較するメインの表には入れません。

## テスト4：Redditのブランド検索のうち、ノイズはどれくらいか

Reddit検索では、タイトルまたは本文にキーワードが含まれている投稿がヒットします。投稿者の自己紹介にブランド名が出てくるだけでも、検索結果に含まれます。23件の結果を大まかなルールで分けました。タイトルかコミュニティ名にブランドが含まれていれば「ブランドについての投稿」、本文にだけ含まれていれば「単なる言及」です。Duolingoは15件対8件、Babbelは5件対18件、Busuuは6件対17件でした。

| キーワード | 件数 | 最新〜最古の期間 | タイトルかコミュニティにブランド名 | 本文のみ | コミュニティ | コメント |
| --- | --- | --- | --- | --- | --- | --- |
| duolingo | 23 | 4.0時間 | 15 | 8 | 16 | 96 |
| babbel | 23 | 9.1日 | 5 | 18 | 19 | 462 |
| busuu | 23 | 15.6日 | 6 | 17 | 20 | 660 |

![積み上げ棒グラフ。ブランドごとのReddit検索結果23件。Duolingo：タイトルかコミュニティ名にブランドを含むもの15件（ティール）、本文でついでに言及しているだけのもの8件。Babbel：5件対18件。Busuu：6件対17件。タイトルには、babbelの検索で23件の投稿が返り、そのうち18件はついでに言及しているだけだと記されています。](https://everyinfra.com/articles/chatgpt-spaces-competitor-social-tracker/fig-05-reddit-noise-bar.ja.light.svg)

具体例を挙げます。Duolingoの23件のうち最新の結果は、r/MakeFriendsOver30の友達募集の投稿でした。投稿者は趣味をたくさん挙げており、その1つがDuolingoの利用でした。同じ結果には、r/jaxfanclubやr/CultofDaisyのような、語学学習とはまったく関係のないコミュニティの投稿も含まれていました。

Babbelの23件は19のコミュニティからのもので、ブランド自身のr/babbelにあったのは3件だけでした。これらが質の悪いデータというわけではありません。誤りは、それらを「ブランドの話題」として数えることです。

そのため、Redditの行はブランドによって扱いを変えています。Duolingoについては、`reddit/subreddit_posts`で取得したブランドコミュニティの週間トップ投稿をメインの表に使います。これは1回の呼び出しで済みます。BabbelとBusuuについては、直近7日間の言及をメインの表に表示します。別の数値として、タイトルかコミュニティ名にブランドを含む投稿の件数も数えます。ChatGPTはその少数の投稿だけを読んで製品を比較している投稿を選び、備考に記載します。このルールは指示の中で固定してください。そうしないと、ChatGPTは毎週異なる基準で選んでしまいます。

## テスト5：レスポンスのサイズと、ページに収まるかどうか

データはJSONとしてChatGPTに届くため、そのサイズによって各タスクが読み込む量が決まります。各レスポンスの生のバイト数を測定しました。`tiktok/profile`は約1.5KBでした。50件の`tiktok/user_videos`は平均119KBでした。

50件の`twitter/user_posts`は約54KB、23件の`reddit/search`は約50KBでした。MCP経由のTikTok動画10本は、テキストで24,746バイトでした。

1ラウンド目の12回の呼び出しは、3ブランドすべてのReddit検索を含めて合計675,752バイトでした。1トークンあたり約4バイトとすると、約169,000トークンになります。上記のレシピでは、DuolingoのRedditの呼び出しは代わりにコミュニティのトップ一覧を返します。そのレスポンスは27,508バイトではなく18,916バイトなので、合計は約667,000バイトに下がります。規模は変わりません。

![レスポンスあたりの平均サイズを示す横棒グラフ。tiktok/profile：1件、1.5KB。tiktok/user_videos：50件、119KB（ティール色で最大）。twitter/user_posts：50件、54KB。reddit/search：23件、50KB。MCP経由で取得したTikTok動画10本：25KB。タイトルには、週12回の呼び出しの合計が約676KB、およそ169kトークンになると書かれています。](https://everyinfra.com/articles/chatgpt-spaces-competitor-social-tracker/fig-06-bytes-bar.ja.light.svg)

169,000トークンという数字は測定値ではなく推定値ですが、規模感はわかります。TikTok動画1本には35個のフィールドがあります。カバー画像や音楽のリンクは長い署名付きURLで、トラッカーではどれも使いません。節約する方法は2つあります。1つ目は、`limit`を必要な数に設定することです。

各ブランドについてTikTok動画を20本取得すれば、3ブランドすべてで7日間をカバーできます。Duolingoは7日間に5本を投稿しており、さらに固定動画が1本あります。これでサイズは約5分の2に減ります。2つ目は、各呼び出しの後、次の呼び出しに進む前に、6つのフィールドからなる小さな表を作るようChatGPTに指示することです。12回分の未加工のレスポンスを最後まで保持させないでください。

## 1週目のトラッカー

以下は、上記のルールに基づいて2026-11-04に実施したラウンドのメイン表です。すべての数値は、このラウンドで実際に行った呼び出しから得たものです。「直近7日間」は、各呼び出し時点のシンガポール時間を基準にさかのぼって数えています。

| ブランド | プラットフォーム | フォロワー数 | 新規投稿（7日間） | 表示回数の中央値（7日間） | 直近50件の中央値 | 備考 |
| --- | --- | --- | --- | --- | --- | --- |
| Duolingo | TikTok | 18.4M | 5 | 803,300 | 660,600、エンゲージメント率14.1% | 50件中42件にis_adフラグ |
| Duolingo | X | 1,049,014 | オリジナル10件（リプライとリポスト18件） | — | オリジナル投稿の表示回数56,364 | 50件で12.8日分 |
| Duolingo | Reddit r/duolingo | — | 今週のトップ投稿25件 | — | コメント1,528件 | コミュニティのトップリスト、最上位投稿は1,615アップボート |
| カテゴリ | Reddit r/languagelearning | — | 今週のトップ投稿25件 | — | コメント732件 | 最上位投稿は481アップボート |
| Babbel | TikTok | 587,800 | 1 | 9,377 | 35,700、エンゲージメント率3.26% | 最新動画は10月28日 |
| Babbel | X | 34,286 | オリジナル2件 | — | オリジナル投稿の表示回数636 | いいね数の中央値1 |
| Babbel | Reddit検索「babbel」 | — | 言及17件 | — | 23件の投稿にコメント462件 | 検索、タイトルまたはコミュニティ名にブランド名を含むものが5件、うちr/babbelが3件 |
| Busuu | TikTok | 844,200 | 1 | 696 | 2,769、エンゲージメント率1.57% | 固定動画3本 |
| Busuu | X | 35,699 | 0 | — | — | 最新投稿は2025-02-12 |
| Busuu | Redditで「busuu」を検索 | — | 9件の言及 | — | 23件の投稿に対する660件のコメント | 検索。タイトルまたはコミュニティ名にブランド名を含むものが6件、うちr/Busuuが2件 |

Redditの行同士は比較しないでください。Duolingoの行はブランドコミュニティのトップリストです。BabbelとBusuuの行は、過去7日間のサイト全体の検索で見つかった言及です。各行がどちらの種類かは備考欄に記載しています。また、コメント数の合計は1回の取得で得た23件すべてを対象としています。

Xの7日間の中央値はあえて空欄にしています。7日間のオリジナル投稿はBabbelの2件のように少なすぎるため、中央値にはあまり意味がありません。参考列には直近50件のオリジナル投稿を使っています。今回は第1週なので、「変化」列はまだありません。第2週からは、ChatGPTが先週のサブページを読み込んで追加します。

## 週次・月次のAPI利用料：月約$0.0289

API利用料は、カタログ価格×呼び出し回数です。基本レシピは3ブランド×4回=12回で、週$0.006672です。4回の呼び出しは`tiktok/profile`、`tiktok/user_videos`、`twitter/user_posts`と、Redditの呼び出し1回（Duolingoは`reddit/subreddit_posts`、BabbelとBusuuは`reddit/search`）です。52週では年$0.346944になります。1か月を52/12週とすると、月約$0.0289です。Redditの2つの機能は同じ価格なので、ブランドごとにどちらを選んでも合計は変わりません。

3ブランドすべてで検索とトップリストの両方を呼び出すと15回になり、週$0.00834、月約$0.0361です。r/languagelearningの行を加えると1回増え、週$0.000556かかります。今回は17回分、合計$0.009452を支払いました。初回分を超えた5回は追加テストで、内訳はコミュニティのトップリスト2回、スペイン語での検索1回、旧アカウントのテスト1回、MCPのラウンドトリップ1回です。

![対数スケールの横棒グラフ（1目盛りごとに10倍）。API利用料：12回の呼び出しで週$0.0067、月$0.0289（青緑）、年$0.3469。Standardシート2席のBusinessは月$50。Pro 100は月$100。タイトルは、API利用料は月$0.0289で、主なコストはChatGPTのシート代であることを示しています。](https://everyinfra.com/articles/chatgpt-spaces-competitor-social-tracker/fig-07-cost-bar.ja.light.svg)

| 項目 | 週あたり | 月あたり | 出典 |
| --- | --- | --- | --- |
| EveryInfra API利用料（12回） | $0.006672 | 約$0.0289 | カタログ価格は1回あたり$0.000556（2026-11-04時点） |
| EveryInfra API利用料（15回、3ブランドすべてで検索とトップリスト） | $0.00834 | 約$0.0361 | 同上 |
| ChatGPT Pro 100（個人向け、Spaceを含む） | — | $100 | [ChatGPT Proのティアについて](https://help.openai.com/en/articles/9793128-about-chatgpt-pro-tiers) |
| ChatGPT Business Standardシート | — | 1ユーザーあたり$25（年払いは$20）、最低2シート | [ChatGPT Businessの概要](https://help.openai.com/en/articles/8792828-chatgpt-business-overview)、[請求とシートの管理](https://help.openai.com/en/articles/8792536-managing-billing-and-seats-in-chatgpt-business) |
| ChatGPT Business Premiumシート | — | 1ユーザーあたり$125（年払いは$100） | 同上、[Premiumシートの発表](https://openai.com/index/premium-seats-chatgpt-business) |

ChatGPTの料金は2026-11-04時点で確認しました。Proには$100、$200、$500のティアがあります。Business Standardシートは2026-04-02に$5値下げされており（ヘルプセンター記事8792536）、購入は最低2シートからです（[ChatGPT Businessに登録する](https://help.openai.com/en/articles/8980713-sign-up-for-chatgpt-business)）。現在、ProプランとBusinessプランには、OpenAI自身のAPIで使えるAPIクレジットが含まれています（[ヘルプセンター記事20001626](https://help.openai.com/en/articles/20001626-api-credits-benefit-for-eligible-chatgpt-subscriptions)）。

Proにはティアに応じて月$100〜$500が付与されます。BusinessにはStandardシート1席あたり$20、1ワークスペースあたり最大$500が付与されます。これらのクレジットは、当社のAPI利用料とは関係ありません。

結論は明快です。トラッカーのAPI利用料は月3セント未満で、ChatGPTのシート1つ分よりおよそ3桁安くなります。構築するかどうかを左右するのはデータのコストではなく、チームがすでにProやBusinessを契約しているかどうかです。1回の実行で消費するChatGPTの利用枠はモデルやワークロードによって異なり、ドキュメントにも引用できる数値は記載されていません。

## エラーと課金：429、422、401は課金されません

今回の21リクエストのうち、4件は課金されませんでした。いずれもシンガポール時間の00:17〜00:29に発生したもので、生のレスポンスはすべて保存してあります。00:17:48と00:17:55の429の2件は、テスト用キーに設定された1分あたり10回のレート制限によるものです。どちらも`retry_after: 12`を返しました。約4分後の00:21:57と00:22:11に再試行したところ、2件とも成功し、通常どおり課金されました。

00:28:29の422の1件は意図的なもので、`sort`に`latest`を指定しました。レスポンスには許可された値`newest`、`oldest`、`popular`が示され、`did_you_mean`として`newest`が提示され、「Nothing was charged」と記載されていました。00:28の401の1件は、前述したURLにキーを含めたMCP呼び出しです。

| ケース | HTTP | レスポンスの内容 | 課金額 | ChatGPTタスクで行う対応 |
| --- | --- | --- | --- | --- |
| 1分あたりの呼び出しが多すぎる | 429 | `retry_after`の秒数、`retryable: true` | 不要 | 待ってから1回だけ再試行。並列呼び出しはしない |
| パラメータ値の誤り | 422 | `allowed`、`did_you_mean`、`catalog_url` | 不要 | 該当パラメータを`did_you_mean`の値に変更して再送信 |
| URLにキーを含めた場合 | 401 | 2つの新しいURL | 不要 | 古いコネクタを削除し、`/mcp/all`に接続 |
| 空の結果 | 200 | `count: 0` | 返金済み | ページに「今週はデータなし」と記載 |

422のメッセージはAIが読むことを前提に書かれています。ChatGPTは`did_you_mean`を見れば、自力でパラメータを修正できます。そのため、タスクの指示には、422が返ったら`did_you_mean`に従ってパラメータを修正するよう記載しています。エラーフィールドの全容は[OpenAPI仕様](https://api.everyinfra.com/openapi.json)に記載されています。

## ChatGPT Spaceのトラッカーでできないこと

公開後に初めて気づくことがないよう、制限をここにまとめます。第1に、提供開始時点ではKeep Updatedは使えません。ページは自動更新されず、スケジュールされたタスクがない週は、表は前週のままです。第2に、モバイルではページの閲覧と共有はできますが、編集はできません。第3に、個人向けProのカスタムMCPには読み取りと取得の権限があります。スケジュールを設定する前に、データツールが呼び出しのたびに承認を求めるかどうかを手動でテストしてください。

第4に、Businessではツールのスナップショットが固定されるため、新しいフィールドやパラメータは管理者が更新して初めてワークスペースに反映されます。第5に、`reddit/search`がさかのぼれるのは最大1か月で、1回の呼び出しで返すのは最大25件です。大きなブランドの週間言及数は数えられません。第6に、TikTokの10,000を超える数値は丸められるため、再生回数が数百万の動画の週ごとの変化は100,000単位で動きます。第7に、ページを閲覧できる人は全員、そこにある競合メモを読めます。メモリをオンにしている共同作業者の場合、その人のChatGPTが内容を記憶する可能性があります。

データそのものにも制限があります。取得するのは公開プロフィールと公開投稿です。非公開アカウント、ダイレクトメッセージ、広告費は取得できません。`is_ad`が示すのは、プラットフォームがその動画を広告としてフラグ付けしたことだけで、広告費の額はわかりません。ページに「競合他社が今週広告にいくら使ったか」といったメモを書く場合は、推測であることを明記してください。

| 制約 | 出典 | 回避策 |
| --- | --- | --- |
| リリース時点ではKeep Updatedなし | learn.chatgpt.comのエージェントガイド | スケジュールタスクまたはチームタスク |
| モバイルでは編集不可 | ヘルプセンター記事20001549 | Webまたはデスクトップで編集 |
| ProのMCPは読み取りと取得のみ | ヘルプセンター記事12584461 | 承認が求められるか確認、またはBusinessを利用 |
| ツールのスナップショットが固定 | ヘルプセンター記事12584461 | 管理者が月1回更新 |
| Reddit検索：1か月、25件 | EveryInfraカタログ | コミュニティのトップリスト |
| TikTokの数値の丸め | 当社のテスト：50件中50件で丸め処理 | 1%未満の変化は横ばいとみなす |

## 最近の変更点

2026-09-29：DevDayでSpace、チーム、チームタスクが公開されました。BusinessとEnterpriseでは、ページやSpaceをチームと共有できます（[Businessリリースノート](https://help.openai.com/en/articles/11391654-chatgpt-business-release-notes)）。影響：メンバーを1人ずつ追加しなくても、トラッカーをチーム単位で共有できます。

2026-10-01：OpenAIは、Space Pagesでエラーが増加していたインシデントを解決しました（[コミュニティでの回答](https://community.openai.com/t/chatgpt-work-spaces-creates-download-links-that-do-not-download/1402511/3)）。影響：10月上旬のページエラーは既知の問題だったため、ページ機能は現在の動作をもとに判断してください。

2026-10-15：Businessの管理者は、Admin Consoleで最大25個のプラグインを一括で管理できるようになりました。プラグインを利用可能にしたり、ワークスペースにインストールしたり、無効にしたりできます（[Businessリリースノート](https://help.openai.com/en/articles/11391654-chatgpt-business-release-notes)）。影響：データプラグインをチームに展開する手順が少なくなります。

2026-10-26：対象となるEnterpriseワークスペースでは、カスタムGPTを新規作成できなくなりました。新しい再利用可能なワークフローにはPlugin Creatorを使用します（[Enterprise・Eduリリースノート](https://help.openai.com/en/articles/10128477-chatgpt-enterprise-and-edu-release-notes)）。影響：トラッカーはGPTではなく、プラグインとタスクの組み合わせで構築してください。

2026-11-02：IETFのClient ID Metadata Documentドラフトがバージョン-03になりました（[IETFドラフト](https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/)）。影響：ChatGPTがMCPサーバーにサインインする方式はまだ標準化の途中で、詳細が変わる可能性があります。

2026-11-04：EveryInfraは、`?key=`でキーを渡すMCP URLの受け付けを終了しました。影響：ChatGPTで`https://api.everyinfra.com/mcp/all`に接続し、OAuthを選んでサインインしてください。古いコネクタは削除してから追加し直してください。

## ChatGPT Spacesのよくある質問

### ChatGPT Spaces vs Notion：競合比較表を移行すべきか？

この質問は、検索候補「chatgpt spaces vs notion」に基づいています。ドキュメントによると、Spaceの強みは、共同編集者がそれぞれ自分のChatGPTやdotに依頼して同じページを編集できる点です。ページには、ChatGPTが作成したビジュアライゼーションも配置できます。弱点は、リリース時点でKeep Updatedが使えないこと、モバイルでは編集できないこと、Pro以上のプランが必要なことです（[ヘルプセンター記事20001549](https://help.openai.com/en/articles/20001549-getting-started-with-space-in-chatgpt)）。コミュニティの機能リクエストでも、ピン留めやより深いサイドバー階層といった基本機能がSpaceには不足していると指摘されています（[Space needs the rest of its UX](https://community.openai.com/t/space-needs-the-rest-of-its-ux/1403608)）。データの原本はエクスポートできる場所に置き、Spaceのページは毎週の閲覧と議論に使ってください。

### ChatGPTのSpaceページでエラーが出る場合はどうすればよいですか？

検索候補「elevated errors in chatgpt space pages」は、OpenAIのステータスページに掲載されたインシデント「Elevated errors in ChatGPT Space Pages」を指しています（[ステータスページ](https://status.openai.com/incidents/01M3R1T4FCR1337FBBYY9K2RPB)）。OpenAIのサポートは、2026-10-01に解決したとコミュニティで回答しています。その後に問題が起きた場合は、まず次の2点を確認してください。通常のチャットではなくWorkで操作していますか。モバイルではなくWeb版またはデスクトップ版を使っていますか。コミュニティのスレッドでは、通常のチャットで操作していることと、モバイルを使っていることが、原因として最も多く挙がっています（[コミュニティのスレッド](https://community.openai.com/t/pages-in-spaces-for-normal-chatgpt-chats/1402509)）。

### ChatGPTのSpaceからEveryInfra経由でTikTokのデータを取得できますか？

はい、EveryInfraをプラグインとして追加すれば可能です。`https://api.everyinfra.com/mcp/all`を指定し、OAuthを選んでサインインします。Businessワークスペースでは、追加できるのは管理者とオーナーのみです。個人のProプランでは、カスタムMCPに読み取り（read）と取得（fetch）の権限が付与されるとドキュメントに記載されています。データツールを一度手動で呼び出し、毎回承認を求められるかどうかを確認してください（[ヘルプセンター記事12584461](https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt)）。

### ChatGPT SpacesについてRedditではどう言われていますか？

検索候補「chatgpt spaces reddit」からは、r/ChatGPTProのスレッドなどが見つかります。あるユーザーは、スマートフォンからChatGPTでページを作成できず、代わりにGoogleドキュメントを使ったと投稿しています。返信では、Chatではページを編集できないもののWorkでは編集できると説明されています（[r/ChatGPTPro](https://www.reddit.com/r/ChatGPTPro/comments/1x9pt47/spaces_pages_so_half_baked/)）。これは1人のユーザーの体験で、OpenAIの公式見解ではありません。ただし、SlidesはWorkで作成するというOpenAIの説明とは一致しています。

## 始め方

1. [EveryInfraコンソール](https://everyinfra.com/ja/docs#quickstart)で新規登録し、アカウントにチャージします。無料のカタログで、必要な各機能が`available: true`になっていることを確認してください。
1. 4つの機能をそれぞれcurlまたは[プレイグラウンド](https://everyinfra.com/ja/market)で一度ずつ呼び出します。フィールドと行数を確認してから、ブランドごとに`limit`を設定してください。
1. Web版のChatGPTで、PluginsからカスタムMCPとして`https://api.everyinfra.com/mcp/all`をOAuthで追加します。チームで共有しているアカウントでサインインしてください。
1. チーム用のSpaceとトラッカーページを作成してください。テーブルブロック、Visualizeブロック、Promptブロックを追加し、週ごとにサブページを1つ作成してください。
1. Workで、毎週月曜09:00に実行するタスク（Businessではチームタスク）の作成をChatGPTに依頼してください。「Run now」を一度クリックし、テーブルとサブページを確認してください。

## 関連記事

- [ChatGPT dotsで作るプロダクトレーダー：データ取得を割り当てタスクに任せる理由](https://everyinfra.com/ja/blog/chatgpt-dots-product-research-radar)：常時稼働するdotが持つ2種類の権限セットを解説しています。本ガイドでは、チームで管理する1つのテーブルを扱います。
- [ChatGPTのスケジュールタスクで需要レーダーを作る](https://everyinfra.com/ja/blog/chatgpt-scheduled-tasks-demand-radar)：1人のユーザーが毎日メッセージを受け取る構成です。本ガイドの出力はチームのページで、Businessではチームタスクとサービスアカウントを使います。
- [Claude CodeのスケジュールタスクでRedditの不満点を収集する](https://everyinfra.com/ja/blog/claude-code-scheduled-tasks-reddit-pain-points)：Claudeで作る同様の週次レポートで、データはローカルファイルに保存します。
- [Claude Coworkのスケジュールタスクで競合の価格とレビューを追跡する](https://everyinfra.com/ja/blog/claude-cowork-scheduled-tasks-competitor-price-reviews)：競合の価格とレビューに関する週次レポートです。
- [MCPサーバーの設定](https://everyinfra.com/ja/mcp-server)

## 参考文献

1. Getting started with Space in ChatGPT、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/20001549-getting-started-with-space-in-chatgpt
1. ChatGPT Space：共有、データ、管理機能、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/20001544-chatgpt-space-sharing-data-and-controls
1. Teams in ChatGPT、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/20001541-teams-in-chatgpt
1. ChatGPTでのチームタスクの作成と管理、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/20001540-creating-and-managing-team-tasks-in-chatgpt
1. Connecting custom MCP servers in ChatGPT、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/12584461-connecting-custom-mcp-servers-in-chatgpt
1. Plugins in ChatGPT、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/20001256-plugins-in-chatgpt
1. Projects in ChatGPT、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/10169521-projects-in-chatgpt
1. About ChatGPT Pro tiers、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/9793128-about-chatgpt-pro-tiers
1. ChatGPT Business overview、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/8792828-chatgpt-business-overview
1. ChatGPT Businessでの請求とシートの管理、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/8792536-managing-billing-and-seats-in-chatgpt-business
1. Sign up for ChatGPT Business、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/8980713-sign-up-for-chatgpt-business
1. ChatGPT Business release notes、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/11391654-chatgpt-business-release-notes
1. ChatGPT Enterprise and Edu release notes、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/10128477-chatgpt-enterprise-and-edu-release-notes
1. 個人ワークスペースをChatGPT BusinessまたはEnterpriseに移行する、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/8801890-migrating-your-personal-workspace-to-chatgpt-business-or-enterprise
1. ChatGPTでのスライドの作成と共同編集、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/20001605-creating-and-collaborating-on-slides-in-chatgpt
1. 対象となるChatGPTサブスクリプションのAPIクレジット特典、OpenAIヘルプセンター、2026-11-04閲覧、https://help.openai.com/en/articles/20001626-api-credits-benefit-for-eligible-chatgpt-subscriptions
1. Work with pages、learn.chatgpt.com、2026-11-04閲覧、https://learn.chatgpt.com/docs/space/pages
1. Work with agents in Space、learn.chatgpt.com、2026-11-04閲覧、https://learn.chatgpt.com/codex/space/agents
1. Collaborate in Space、learn.chatgpt.com、2026-11-04閲覧、https://learn.chatgpt.com/codex/space/collaboration
1. Scheduled tasks、learn.chatgpt.com、2026-11-04閲覧、https://learn.chatgpt.com/codex/automations
1. プラグインの接続とテスト、OpenAI開発者ドキュメント、2026-11-04閲覧、https://developers.openai.com/plugins/deploy/connect-chatgpt
1. ChatGPT Developer mode、OpenAI開発者ドキュメント、2026-11-04閲覧、https://developers.openai.com/api/docs/guides/developer-mode
1. Authentication (Apps SDK)、OpenAI開発者ドキュメント、2026-11-04閲覧、https://developers.openai.com/apps-sdk/build/auth
1. Secure MCP Tunnels、OpenAI開発者ドキュメント、2026-11-04閲覧、https://developers.openai.com/api/docs/guides/secure-mcp-tunnels
1. MCP仕様、Schema Reference (ToolAnnotations)、バージョン2025-11-25、https://modelcontextprotocol.io/specification/2025-11-25/schema#toolannotations
1. MCP仕様、Authorization、バージョン2025-11-25、https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization
1. RFC 9728、OAuth 2.0 Protected Resource Metadata、IETF、2025-04、https://datatracker.ietf.org/doc/html/rfc9728
1. OAuth Client ID Metadata Document (draft-03)、IETF、2026-11-02、https://datatracker.ietf.org/doc/draft-ietf-oauth-client-id-metadata-document/
1. EveryInfraデータカタログ、シンガポール時間2026-11-04 00:16閲覧、https://api.everyinfra.com/api/v1/social/catalog
1. EveryInfra /mcp/allの保護リソースメタデータ、シンガポール時間2026-11-04 00:28閲覧、https://api.everyinfra.com/.well-known/oauth-protected-resource/mcp/all
1. EveryInfra OpenAPI仕様、2026-11-04閲覧、https://api.everyinfra.com/openapi.json
1. DevDay 2026 Recap、OpenAI、2026-09-29、https://openai.com/index/devday-2026-recap/
1. ChatGPT Space製品ページ、OpenAI、2026-11-04閲覧、https://chatgpt.com/features/space/
1. 「Plugins in ChatGPT」製品ページ、OpenAI、2026-11-04閲覧、https://chatgpt.com/features/plugins/
1. ChatGPT Businessにプレミアムシートが登場、OpenAI、2026-11-04閲覧、https://openai.com/index/premium-seats-chatgpt-business
1. ChatGPTが最も野心的な仕事のパートナーに、OpenAI、2026-07-09、https://openai.com/index/chatgpt-for-your-most-ambitious-work/
1. Elevated errors in ChatGPT Space Pages、OpenAIステータスページ、2026-11-04閲覧、https://status.openai.com/incidents/01M3R1T4FCR1337FBBYY9K2RPB
1. Pages in Spaces for normal ChatGPT chats、OpenAI Developer Community、2026-10-01、https://community.openai.com/t/pages-in-spaces-for-normal-chatgpt-chats/1402509
1. ChatGPTのWork「Spaces」で作成されるダウンロードリンクからダウンロードできない（サポートの回答）、OpenAI Developer Community、2026-10-16、https://community.openai.com/t/chatgpt-work-spaces-creates-download-links-that-do-not-download/1402511/3
1. SpaceのUXにはさらなる整備が必要、OpenAI Developer Community、2026-10-06、https://community.openai.com/t/space-needs-the-rest-of-its-ux/1403608
1. Spaces / Pages... so half baked、Reddit r/ChatGPTPro、2026-11-04閲覧、https://www.reddit.com/r/ChatGPTPro/comments/1x9pt47/spaces_pages_so_half_baked/

---

:::message
この記事は EveryInfra ブログの記事の転載です。原文：[ChatGPT Spacesで競合のSNSトラッカーを作る：設定、MCP、実地テスト](https://everyinfra.com/ja/blog/chatgpt-spaces-competitor-social-tracker)

EveryInfra は AI エージェントと開発者向けの従量課金データ API です：[everyinfra.com/ja](https://everyinfra.com/ja)
:::
