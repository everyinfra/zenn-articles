---
title: "店舗オーナー向けClaude Duet：新着の低評価レビューを毎朝まとめて受け取る"
emoji: "📊"
type: "tech"
topics: ["datascience", "ecommerce", "claude", "api", "ai"]
published: true
---

:::message
この記事は [EveryInfra ブログ](https://everyinfra.com/ja/blog/claude-duet-shop-owner-google-maps-reviews) からの転載です（原文が正本です）。
:::

2026年11月5日、シンガポール時間00:16〜00:27に、米国の小規模店舗3店のレビューをリアルタイムで取得しました。対象はFlitch Coffee（East Austin）、Chun Yang Tea（Flushing、New York）、Taquería Chabelita（East End、Houston）です。最新のGoogleレビューは、それぞれ13.5日前、41.8日前、22.9日前のものでした。直近90日間で新しいレビューが投稿された日は、それぞれ7日、8日、4日しかありません。つまり、毎朝低評価レビューをチェックするプロアクティブなアシスタントは、ほとんどの朝は何も通知しないはずです。`since`に最近の日付を指定した場合、新しいレビューがない呼び出しは空の結果を返し、費用はかかりませんでした。

> **要点**
>
> - 各店舗の最新Googleレビュー50件のうち、直近90日間のものはそれぞれ8件、8件、5件だけでした。
> - 新しいレビューがないときの`since`呼び出しは0件で、`charged_usd`は0、理由は`empty_result_refunded`でした。
> - Flitch Coffeeは、Googleの低評価レビュー4件のうち0件にしか返信しておらず、最後の返信はMay 31でした。
> - 提案するプランのAPI費用は月額約$0.0243です。DuetはPro（月額$20）に含まれています。
> - Duet自体はまだテストしていません。以下のDuetの設定はヘルプページに基づく設定例で、ヘルプページにはカスタムコネクタについての明記がありません。

この記事では、開発者のいない店舗が抱える実務上の問いに答えます。オーナーが望むのは、毎朝スマートフォンに届く1通のメッセージです。昨日、Google MapsやYelpに新しい低評価レビューは投稿されたか。内容はどうで、返信すべきか。近所の店で何か変化はあったか。使うツールはClaude Duetで、Anthropicが2026年10月21日にリサーチプレビューとして公開したものです。

データはEveryInfraの4つの機能、`google_maps_reviews/reviews`、`google_maps_reviews/search`、`yelp/reviews`、`yelp/search`から取得しています。以下のEveryInfraの数値はすべて、実際の呼び出しで得た生のレスポンスに基づいています。私たちはDuetのアカウントを持っておらず、今回のテストではWeb版のClaudeにもログインしていません。そのため、以下のDuetの画面と動作はすべて設定例です。Duetに関する各事実にはclaude.comまたはsupport.claude.comへのリンクを付けており、それらのページはすべて2026年11月5日に確認しました。

![薄いグレーの床に、同じ形の白い郵便受けが6つ並んだイラスト。各郵便受けがそれぞれ1回の朝を表しています。5つは閉じていて空で、1つは開いていて中に折りたたまれたティール色のカードが入っています。各郵便受けからクラウドサーバーへ点線が伸びており、ティール色なのは開いた郵便受けからの線だけです。](https://everyinfra.com/articles/claude-duet-shop-owner-google-maps-reviews/img-explain.webp)

*小規模な店舗ではレビューの投稿はまばらです。新しい低評価レビューがない朝がほとんどなので、ブリーフは何かが起きた日にだけ届くようにします。* [本記事のために作成したイラストであり、実データではありません](https://claude.com/resources/articles/claude-duet)

## 目的：コードを書かないオーナーに毎朝1通のメッセージを届ける

本サイトには地図レビューを扱った記事がほかに2本あるため、違いを整理しておきます。[Googleマップのレビューをスクレイピングする方法](https://everyinfra.com/blog/how-to-scrape-google-maps-reviews)は、レビューを表形式でエクスポートしたい開発者向けの記事です。Google公式のAPI、自作のスクレイパー、呼び出しごとに課金されるデータAPIを比較しています。[Claude Managed Agentsによる夜間のレビュー監視](https://everyinfra.com/blog/claude-managed-agents-nightly-review-monitoring)は、エンジニアがいるレストラングループ向けです。Anthropicのクラウド上で無人のジョブ（スケジュールデプロイ、Vault、Webhook）を実行します。サンプルはニューヨークのKatz's Delicatessenで、1日に20件を超える新しいレビューが付きます。

本記事の読者はコードを書きません。手元にあるのはスマートフォンとClaudeのプラン、そしてずっと小さな店舗です。その店舗に付く新しいレビューは月に数件、低評価レビューはせいぜい年に数件です。アシスタントは、何も起きなかった日は黙っていなければなりません。重要な1件の低評価レビューを見逃してもいけません。さらに、空振りのチェックに費用をかけてもいけません。

Anthropicの発表記事では、Duetは把握すべきことが多すぎるプロフェッショナルや小規模事業者向けだとされています（[Introducing Claude Duet](https://claude.com/resources/articles/claude-duet)）。英語版の本記事では、East AustinのコーヒートレーラーFlitch Coffeeを主なサンプルとしています。API呼び出しを行う前に、3つのテスト基準を定めました。1つ目は、星1〜2の新しいレビューがあったときか、競合店に明確な変化があったときだけブリーフのメッセージが届くことです。2つ目は、同じ低評価レビューを2回報告しないことです。3つ目は、新しいレビューがない朝のAPI費用がゼロ、またはほぼゼロであることです。

## Claude Duetとは：10月21日に公開されたプロアクティブなアシスタント

Anthropicは2026年10月21日（2026-10-21）、Claude Duetをリサーチプレビューとして公開しました。発表記事ではプロアクティブなエージェントと位置づけられています。対応が必要なことを自ら知らせ、ユーザーが離れたあとも作業を続けます。ユーザーに代わって行動する前には確認を求めます（[Introducing Claude Duet](https://claude.com/resources/articles/claude-duet)）。X上のClaudeアカウントによると、ウェイトリストはまず米国のProおよびMaxユーザーに開放されます（[XでのClaudeの発表投稿](https://x.com/claudeai/status/2112984669943968099)）。CNBCはDuetを、OpenAIのdotsやMetaのMuseに対抗するものと報じています（[CNBC、2026-10-21](https://www.cnbc.com/2026/10/21/anthropic-preview-of-claude-duet-personal-ai-agent-take-on-muse-dots.html)）。

レビューのブリーフで特に重要な機能は4つあります。第1に、Duetは毎朝、連携したアプリから最も重要な項目をまとめた短いブリーフを送ります。日中はそれらのアプリを監視し、対応が必要なときだけメッセージを送ります。第2に、クラウド上に専用の仮想マシンとブラウザを持つので、スマートフォンを起動したままにしておく必要はありません。第3に、連携アカウントは数時間ごとにチェックします。投稿ではその理由として、常時チェックするとプランの上限を早く使い切ってしまうことを挙げています。

4つ目に、Duetは送信、支払い、署名、削除を行う前に、アクションの全内容を表示して承認を待ちます（[Introducing Claude Duet](https://claude.com/resources/articles/claude-duet)）。何について、どのくらいの頻度で知らせてほしいかを指示することもできます。ヘルプページによると、Duetは決まった間隔でチェックします。制限事項の一覧には、緊急の項目が2回のチェックの間に届く場合があるとも書かれています（[Get started with Claude Duet](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。レビューのブリーフであれば、この程度の遅れはたいてい問題ありません。

| 機能 | 情報源の記載内容 | レビューのブリーフにとっての意味 | 情報源（2026-11-05に確認） |
| --- | --- | --- | --- |
| 朝のブリーフ | 毎朝メールとカレンダーに目を通し、対応が必要なものを選び出し、返信の下書きも作成できる | 例はメールとカレンダーのみなので、レビューについては明示的に依頼する必要がある | [ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet) |
| チェック頻度 | 数時間ごと。ヘルプページは「決まった間隔」と記載するのみで、具体的な数値はない | チェックのたびにAPIを呼び出すとコストが増えるため、プロンプトで1日1回に制限する | [発表記事](https://claude.com/resources/articles/claude-duet) |
| モデル | Claude Opus 5.5で動作し、モデルは選べない | 読み取り、仕分け、返信の下書きはすべて同じモデルで行う | [ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet) |
| 承認 | メッセージの送信、支払い、署名、削除の前に確認を求める | 返信の投稿には承認が必要。データの読み取りに承認が必要かどうかは情報源に記載がない | [安全性に関する記事](https://claude.com/resources/articles/claude-duet-safety-privacy-and-security) |
| 使用量 | プランの利用上限をClaudeのほかの機能と共有し、バックグラウンドのチェックもカウントされる | 読み込むレビュー本文が多いほど、利用上限を多く消費する | [ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet) |
| ブラウザ | コネクタのないサイトでは専用のクラウドブラウザを使う。操作を見たり引き継いだりはできない | Googleマップのページを開けるが、何を読んだかは確認できない | [ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet) |

![2行に並んだ6つのボックスのフローアニメーションで、ティールのハイライトが各ステップを順に移動します。ステップ1は1日1回のバックグラウンドチェックで、ステップ2はgoogle_maps_reviews/reviewsとsince（前日）を指定してeveryinfra_call_apiを呼び出します。ステップ3は結果が空かどうかを判定し（count 0なら課金もメッセージもなし）、点線が2行目へ戻ります。ステップ4はレビューidで重複を除いて星1〜2のレビューを選び、ステップ5ではyelp/reviewsの1ページ目とgoogle_maps_reviews/searchの週1回の読み込みを加えます。ステップ6は本文と返信の下書きをスマートフォンに送ります（承認後にのみ送信）。フッターには月約0.0243米ドルとあります。](https://everyinfra.com/articles/claude-duet-shop-owner-google-maps-reviews/fig-02-duet-flow.ja.light.svg)

この図は私たちが提案するフローで、以下のセクションでは実データを使ってステップ1、3、5を説明します。最初に注意点が1つあります。発表記事には、何に対応が必要で何を後回しにできるかはDuetが判断すると書かれています。また、その判断はテスターからのフィードバックをもとにAnthropicが現在も調整しているとしています（[Introducing Claude Duet](https://claude.com/resources/articles/claude-duet)）。そのため、DuetがGoogleマップを自ら思いつくとは想定していません。タスクでは、毎日何を読み、いつメッセージを送るかを明確に指示します。

## Claude Duetのウェイトリスト：申し込める人と必要なプラン

ヘルプページの「Who can get access」セクションには、3つの条件が挙げられています。Pro、Max、Ultraのいずれかのプラン、米国のClaudeアカウント、そしてウェイトリストからの招待です。次のセクションには、条件を満たしていても待つ可能性がある理由が4つ書かれています。招待は段階的に送られ、プレビューは米国限定です。TeamプランとEnterpriseプランにはまだ対応していません。また、ウェイトリストに登録したメールアドレスが、Claudeアカウントやスマートフォンのアカウントと一致していない場合もあります。

2026年11月5日00:38（シンガポール時間）に、両セクションのスクリーンショットを撮影しました。

![Claudeヘルプセンターのスクリーンショット。アクセスに関する2つのセクションがティール色の枠で囲まれています。Claude Duetを使うには、Pro、Max、Ultraのいずれかのプラン、米国のClaudeアカウント、ウェイトリストからの招待が必要です。招待は日程未定のまま段階的に送られるため、利用開始まで待つ場合があります。リサーチプレビューは米国限定で、TeamとEnterpriseにはまだ対応していません。ウェイトリストのメールアドレスは、Claudeアカウント、またはスマートフォンのGoogleアカウントかAppleアカウントと一致している必要があります。](https://everyinfra.com/articles/claude-duet-shop-owner-google-maps-reviews/img-support-duet-access.webp)

*Claude Duetヘルプページにあるアクセスに関する2つのセクションのスクリーンショット（2026-11-05 00:38シンガポール時間に確認）* [Claudeヘルプセンター](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)

1点、混乱しやすい部分があります。10月21日のX投稿ではProとMaxとされていました。一方、11月5日時点のヘルプページと製品ページではPro、Max、Ultraとなっています（[製品ページ](https://claude.com/product/claude-duet)）。Duetの対象が広がったのではなく、プランの名称が変わっただけです。2026年10月30日、AnthropicはMax 5xをMaxに、Max 20xをUltraに改称しました。料金は月額$100と$200のまま変わっていません（[The New Stack、2026-10-30](https://thenewstack.io/claude-ultra-session-limit/)。[Ultraプランのヘルプページ](https://support.claude.com/en/articles/17728952-what-is-the-ultra-plan)も参照）。

店舗オーナーにとって最も手頃な選択肢はProです。料金ページでは、月払いで月額$20、年払いで月額$17と表示されています（[Claudeの料金](https://claude.com/pricing)）。Proのヘルプページによると、地域ごとの料金は現地通貨で表示され、税制は地域によって異なります。また、Proには5時間単位の利用上限と週単位の利用上限があります（[What is the Pro plan?](https://support.claude.com/en/articles/8325606-what-is-the-pro-plan)）。Maxは月額$100で、Proの5倍利用できます（[What is the Max plan?](https://support.claude.com/en/articles/11049741-what-is-the-max-plan)）。レビューの要約であればProで十分です。

ヘルプページには申し込み手順も記載されています。まず製品ページで「Join the waitlist」をクリックします。名前とClaudeアカウントのメールアドレスを入力し、プランとスマートフォンの種類を選びます。順番が来るとメールが届きます。そのメールアドレスに紐づいたAppleアカウントまたはGoogleアカウントでアプリをインストールします（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。

新規ユーザー向けにちょっとした特典があります。プレビューに参加すると、5時間単位と週単位の利用上限を1回だけ完全にリセットできます。このリセットは2026年11月30日まで利用できます（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)、[制限のリセットとは](https://support.claude.com/en/articles/17007452-what-is-a-limit-reset)）。あるRedditユーザーは2026-10-22に申し込みましたが、投稿した時点でもまだ返信がなく、参加できた人はいるかと尋ねていました（[Redditスレッド](https://www.reddit.com/r/ClaudeAI/comments/1xeiaqv/has_anyone_actually_gotten_into_claude_duet_yet_i/)）。招待の時期を明示していないヘルプページの記載とも一致します。

| 条件 | ヘルプページの記載 | 店舗オーナーが注意すべき点 |
| --- | --- | --- |
| プラン | Pro、MaxまたはUltra。プレビュー期間中は追加料金なし | 月額$20から（Pro、月払い） |
| 地域 | リサーチプレビュー期間中は米国のみ | アカウントが米国のものである必要あり。他の国での提供時期は未定 |
| チーム向けプラン | TeamとEnterpriseは未対応。今後対応予定 | 1つのワークスペースを共有している店舗はまだ利用できない |
| 招待 | 段階的に送信、時期は未定 | ウェイトリストに登録したメールアドレスがClaudeアカウントと一致している必要あり |
| 使用量 | Claudeの他の機能と制限を共有。バックグラウンドでのチェックもカウントされる | 参加時に制限リセットが1回付与され、11月30日まで有効 |

## Duetが引き継ぐコネクタ：カスタムコネクタへの言及はない

DuetがEveryInfraのデータを読み取れるのは、Claudeで追加したカスタムコネクタをDuetが使える場合に限られます。ヘルプページによると、DuetはユーザーのClaudeアカウント上で動作します。そのため、Claudeで設定したメモリ、スキル、コネクタは最初のメッセージから利用できます。例として挙げられているのはGmail、Google Calendar、Google Drive、Slack、QuickBooks、HubSpot、Docusign、Calendlyです（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。安全性に関する記事では、接続済みのアプリはDuetにも引き継がれ、一方でオフにするともう一方でもオフになると説明されています（[安全性に関する記事](https://claude.com/resources/articles/claude-duet-safety-privacy-and-security)）。発表記事では、Claude Marketplaceに6,000を超えるコネクタとプラグインがあると紹介されています（[Introducing Claude Duet](https://claude.com/resources/articles/claude-duet)、[Claude Marketplace](https://claude.com/marketplace/connectors-plugins)）。

これらのページには「カスタムコネクタ」という言葉は出てきません。かといって、Duetがディレクトリ掲載のコネクタに限定されるとも書かれていません。私たちが見つけた唯一の直接的な根拠は、Redditのあるユーザーの報告です。このユーザーは複数のGoogleアカウントを接続するために独自のMCPサーバーを構築し、Duetはすぐにそれを使えたそうです（[Redditスレッド](https://www.reddit.com/r/claude/comments/1xhct14/claude_duet_first_takeaways/)）。これは1人のユーザーの報告であり、Anthropicが保証しているものではありません。

私たちはヘルプページの記載を文字どおりに受け取っています。つまり、Claudeで追加したEveryInfraのカスタムコネクタはDuetにも表示されるはずです。実際に頼る前に、ご自身のDuetで確認してください。ヘルプページには既知の制限も2つ挙げられています。一部のコネクタはDuetでは機能が制限されること（例としてSlack）と、Microsoft 365はまだ利用できないことです（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。

カスタムコネクタ自体はすべてのClaudeプランで利用できます。リモートMCPのカスタムコネクタは、すべてのプランでClaude、Cowork、Claude Desktopに対応しており、Freeユーザーも1つ追加できます（[コネクタのヘルプページ](https://support.claude.com/en/articles/11176164-use-connectors-to-extend-claude-s-capabilities)）。ネットワーク経路にも注意が必要です。カスタムコネクタは、スマートフォンからではなく、AnthropicのクラウドからMCPサーバーに接続します（[カスタムコネクタのヘルプページ](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)）。EveryInfraのリモートMCPアドレス`https://api.everyinfra.com/mcp`は公開インターネット上にあるため、この点は問題になりません。

## Duet自身のブラウザでGoogleマップを開かせない理由

多くの人がまずこう考えます。Duetにはブラウザがあるのだから、毎朝Googleマップを開かせればよいのではないか。そもそも、なぜデータAPIを導入する必要があるのか。情報源からは3つの理由が分かり、私たちの実地テストでそれぞれを詳しく補足します。

1つ目に、ブラウザが何を読み取ったかを確認できません。ヘルプページによると、Duetがブラウザで作業する様子を見ることはできず、操作を引き継ぐこともできません（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。リリース記事でも、Duetはまだブラウザを表示できないとされています（[Introducing Claude Duet](https://claude.com/resources/articles/claude-duet)）。レビュー要約で最も深刻な失敗は、レビューを見落とすことです。APIなら、`id`、`posted_at`、`rating`、`owner_response`を含む構造化されたレコードが返り、1件ずつ追跡できます。

第2に、サイト側は拒否できます。セーフティに関する投稿によると、Duetのブラウザは公開済みのIPアドレスから独自のUser-Agentを送ります。公開前に、Anthropicはいくつかの大規模サイトにDuetをブロックする方法を提供し、一部のサイトはブロックを選びました（[セーフティに関する投稿](https://claude.com/resources/articles/claude-duet-safety-privacy-and-security)）。投稿ではそれらのサイト名は明かされていません。Google MapsやYelpが含まれるかはわかりません。それでも、「ブラウザに見に行かせる」方法が安定した手段ではないことはわかります。

3つ目に、ページの表示順は当てになりません。Yelpのデフォルトの並び順では、投稿時期、ユーザーの投票、レビューの質が組み合わされるため、古いレビューが新しいレビューより上に表示されることがあります。「Newest First」を選べるのは、ログインしたユーザーだけです（[Yelp Support](https://www.yelp-support.com/article/How-is-the-order-of-reviews-determined?)）。Flitch CoffeeのYelpページの1ページ目では、10件のレビューが日付順に並んでいませんでした。Googleでは、レビューはまずポリシーチェックを通過する必要があり、表示されるまで数日かかることがあります（[クチコミが表示されない、または表示が遅れる場合](https://support.google.com/business/answer/10313341)）。オーナーが返信した後に顧客がレビューを編集すると、日付は最後の編集日に変わります（[顧客のクチコミを管理する](https://support.google.com/business/answer/3474050)）。

実際のデータでもこの影響を確認しました。Taquería ChabelitaのGoogleレビューの最新50件では、2026年のレビューの中に2023年のレビューが6番目に入っていました。このデータでは日付順の乱れが5か所ありました。Chun Yang Teaでは、2024年のレビューの中に2022年と2019年の古いレビューが2件混ざっていました。「上から順に昨日の分まで」読むブラウザは、こうしたレビューでつまずきます。

Google自身のAPIでも解決できません。Places APIが返すレビューは1つの場所につき最大5件で、関連性順に並びます（[Places APIリファレンス](https://developers.google.com/maps/documentation/places/web-service/reference/rest/v1/places)）。Yelp Fusionのレビューエンドポイントが返すのは最大3件のレビュー抜粋です（[Yelp開発者ドキュメント](https://docs.developer.yelp.com/reference/v3_business_reviews)）。自分の店舗であれば、Google Business Profile APIですべてのレビューを一覧表示し、返信を投稿できます。ただし、事前にアプリを登録し、OAuth 2.0の認証情報を取得する必要があります（[Google Business Profile API](https://developers.google.com/my-business/content/review-data)）。コードを書かないオーナーにはハードルが高く、競合店のレビューも読めません。

Googleがすでに新着レビューをオーナーに通知しているのでは、と思う読者もいるでしょう。実際に通知はあります。ビジネスプロフィールの通知には、新しいレビューや写真を知らせる「顧客のアクティビティ」というタイプがあります。通知はメール、Google検索、またはマップアプリで届き、タイプごとにオン・オフを切り替えられます（[通知を管理する](https://support.google.com/business/answer/7198436?hl=en)）。レビュー要約は、この通知に3つの要素を加えます。Yelpのレビューを加え、未返信の星1〜2のレビューを先頭に表示します。さらに、返信の下書きと競合店に関するメモも加えます。

| レビューの取得方法 | 得られるもの | 競合店も見られるか | 主な制約 |
| --- | --- | --- | --- |
| Duet自身のブラウザ | ページに表示される内容 | 可能 | 読み取った内容を確認できない。サイト側がブロック可能 |
| Google Places API | 最大5件、関連性順 | 可能 | 最新のレビューではない |
| Yelp Fusionのレビュー | 最大3件の抜粋 | 可能 | 抜粋のみ |
| Business Profile API | 自店舗のすべてのレビュー（返信付き） | 不要 | アプリ登録とOAuthが必要 |
| ビジネスプロフィールの通知 | 新着レビューごとの通知 | 不要 | Googleのみ、下書きなし |
| EveryInfraのレビューデータ | ID、日付、返信を含む構造化データ | 可能 | 呼び出しごとに課金。`since`の結果が空なら無料 |

## 設定例：ClaudeへのEveryInfraコネクタの追加

以下の手順は、ClaudeのヘルプページとClaudeのドキュメントに基づいています。私たちが実際に画面を操作して確認したわけではないため、メニュー名は実際の画面表示を優先してください。Pro、Max、Ultraプランでは、「Customize」→「Connectors」の順に進み、「Add」をクリックします。「Custom」にカーソルを合わせて「Web」を選び、名前とリモートMCPのURLを入力して「Continue」をクリックします。「Authentication」で「No sign in」を選び、「Request headers」にAPIキーを追加します（[コネクタのヘルプページ](https://support.claude.com/en/articles/11176164-use-connectors-to-extend-claude-s-capabilities)）。

Claudeのドキュメントには、注意点が2つ記載されています。Claudeはヘッダーの値を記述どおりに送信し、「Bearer」を自動では付加しません。そのため、`Authorization`の値は`Bearer `、半角スペース1つ、キーの順に記述する必要があります。また、保存後に認証方法を変更することはできません。キーを変更するには、コネクタを削除してから追加し直します（[ディレクトリにないコネクタを追加する](https://claude.com/docs/connectors/custom/add-unlisted)）。

```text
# Configuration example (Claude on the web -> Customize -> Connectors -> Add -> Custom -> Web)
Name:            EveryInfra review data
MCP server URL:  https://api.everyinfra.com/mcp
Authentication:  No sign in
Request headers:
  Authorization: Bearer <a separate EveryInfra key just for Duet>

# Tool permissions (Customize -> Connectors -> EveryInfra review data -> Tool permissions)
everyinfra_list_capabilities   Always allow     # free, lists capabilities and parameters
everyinfra_call_api            Always allow     # the daily data call; it is billed, see next section
everyinfra_get_job             Always allow     # free, reads async job results
all other tools                Blocked          # the brief does not need them
```

Duetには専用のAPIキーを使ってください。EveryInfraのキーでは、使用できる機能を一部に限定し、米ドル建ての利用総額の上限を設定できます。キーが上限に達すると、リクエストは拒否され、課金されません。各フィールドは[OpenAPIドキュメント](https://api.everyinfra.com/openapi.json)に記載されています。設定方法は[リモートMCPセットアップガイド](https://everyinfra.com/build/remote-mcp-setup)と[MCPサーバーのページ](https://everyinfra.com/mcp-server)で説明しています。後述のコスト試算では1店舗あたりの費用は年間$0.30未満なので、$2の上限で数年分をまかなえます。

## ツールの権限：everyinfra_call_apiにdestructiveHintが付いている理由

これは今回の設定で最大の未解決事項です。シンガポール時間2026年11月5日0時15分に、キーなしでMCPのツール一覧（`tools/list`）を1回取得しました。この呼び出しは無料です。一覧には12個のツールがあり、各ツールにはMCP仕様で定義された動作ヒント（アノテーション）が付いています。以下は、レビュー要約で使用する3つのツールです。

| ツール | readOnlyHint | destructiveHint | 内容 | Claudeでの推奨設定 |
| --- | --- | --- | --- | --- |
| `everyinfra_list_capabilities` | true | false | 機能、パラメータ、料金を一覧表示（無料） | 常に許可（Always allow） |
| `everyinfra_call_api` | false | true | 任意のデータ機能を呼び出し（呼び出しごとに課金） | 常に許可（Always allow）、上限付きキーと併用 |
| `everyinfra_get_job` | true | false | 非同期ジョブの結果を取得（無料） | 常に許可（Always allow） |

`everyinfra_call_api`は、どのレビューサイト上のデータも変更しません。では、なぜ「may be destructive」（破壊的な可能性あり）と表示されるのでしょうか。このツールはウォレットから料金を差し引くため、MCPの定義では元に戻せない外部への変更にあたるからです。仕様では、サーバーが信頼できる場合を除き、クライアントはこれらのヒントを信頼できないものとして扱わなければならないとしています（[MCP仕様：Tools](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)）。3月に公開されたMCPのブログ記事では、クライアントが読み取り専用ツールを自動承認し、`destructiveHint: true`のツールには確認ステップを追加する例が挙げられています（[MCPブログ、2026-03-16](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/)）。

このヒントのせいで、Duetは毎朝承認を求めて止まるのでしょうか。情報源には記載がありません。確認できる事実は3つあります。1つ目に、Duetはメッセージの送信、支払い、書類への署名、削除の前に確認を求めます。さらに、別の自動チェックが各アクションがユーザーの意図に沿っているかを判断します（[安全性に関する記事](https://claude.com/resources/articles/claude-duet-safety-privacy-and-security)）。このチェックは、Claude Codeのauto modeを支える分類器をもとに構築されています（[Anthropic Engineering: auto mode](https://www.anthropic.com/engineering/claude-code-auto-mode)）。

第2に、Claudeのコネクタの権限は、ツールごとに「Always allow」「Needs approval」「Blocked」のいずれかに設定できます（[コネクタのヘルプページ](https://support.claude.com/en/articles/11176164-use-connectors-to-extend-claude-s-capabilities)）。第3に、不在中に動くルーティンにも同じ保護が適用されます（[セーフティに関する投稿](https://claude.com/resources/articles/claude-duet-safety-privacy-and-security)）。したがって最も安全な設定は、`everyinfra_call_api`を「Always allow」にし、利用上限を設定したAPIキーを使うことです。それでもDuetが毎回確認を求める場合は、初日の朝に一度承認してください。そのうえで、「このタスクでは許可」のような選択肢が出るかを確認してください。

ヘルプページには役立つ補足もあります。期限内に承認しなかった場合、Duetはタスクの残りを終えてから、後で改めて承認を求めます（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。最悪の場合でもブリーフが遅れるだけで、お金を失うことはありません。レビューが月に数件しか届かない店舗にとって、ブリーフの遅れは小さな代償です。誤った課金のほうが問題ですが、上限付きのキーを使えば大きな課金は防げます。

次に、ほかの2つの製品と比べてみましょう。OpenAIによると、dotsのプロアクティブなリサーチでは読み取り専用ツールのみを使用します（[Introducing dots](https://openai.com/index/introducing-dots/)）。メッセージの送信やアプリのコンテンツの変更は行いません。このルールに従うと、`readOnlyHint: false`のデータツールは、ユーザーが割り当てたタスクの中でしか実行できません。詳しくは記事「[ChatGPT dotsによる商品リサーチレーダー](https://everyinfra.com/blog/chatgpt-dots-product-research-radar)」で解説しています。MetaのMuseでは、「Ask for some actions」を選択すると、すべての書き込みと重要な読み取りの前に確認を求めます（[Muse guidance and approvals](https://www.meta.com/help/artificial-intelligence/1385290430137537/)）。

## 朝のブリーフのプロンプトとフォーマット

ヘルプページによると、Duetには人にメッセージを送る感覚でタスクを頼めます（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。どんな情報をどのくらいの頻度で知りたいかも指定できます（[Introducing Claude Duet](https://claude.com/resources/articles/claude-duet)）。以下は実際のテスト結果をもとに作成したタスクで、そのままDuetに送信できます。費用を抑え、見落としを防ぐための工夫が3つあります。Googleは`since`を使って1日1回だけ取得します。Yelpと競合店は毎週決まった曜日に実行します。「新着」かどうかはレビューIDだけで判定します。

```text
Starting tomorrow, check my shop's new reviews once each morning at 7 a.m. Austin time.
Use only the EveryInfra review data connector. Do not use the browser.

1. Call everyinfra_call_api: platform=google_maps_reviews, action=reviews,
   params={url: "<my shop's Google Maps link>", sort: "newest", since: "<yesterday, YYYY-MM-DD>", limit: 20}.
   If count is 0, stop this step for today and do not message me.
2. Every Monday and Thursday, also call platform=yelp, action=reviews, params={url: "<my shop's Yelp link>"}.
   Yelp is not sorted by date, so read all 10 reviews on the page.
3. Every Monday, call platform=google_maps_reviews, action=search,
   params={keyword: "coffee shop", location: "East Austin, TX", limit: 20}.
   Note each shop's rating and review_count and compare with last week.
4. Use the review id to decide what is new. Remember the ids you already told me about. Never report the same one twice.
5. Message me only in three cases: a new 1-2 star review; a new 3 star review that mentions service or cleanliness;
   a rival whose rating moved by 0.1 or more, or that gained 10 or more reviews in a week.
6. Message format:
   - First line: how many bad reviews to handle today.
   - For each bad review: site, stars, date, the text, whether I replied,
     and a suggested reply (at most 3 sentences; apologize first; never offer a discount or a gift).
     Do not promise or invent an action that I did not take. If you do not know what I changed, leave the slot [what you actually changed] for me to fill in.
   - Last line: one sentence on rivals this week.
7. Show me reply drafts only. Never post anything to any site for me.
```

| ケース | メッセージ送信 |
| --- | --- |
| 新着レビューなし（`count` 0） | 不要 |
| 星4〜5の新着レビュー | 送信しない（IDのみ保存） |
| 星1〜2の新着レビュー | 送信する（本文と返信案を添付） |
| 接客や清潔さに関する星3の新着レビュー | 可能 |
| 競合店の評価が0.1以上変動 | 送信する（月曜日に1行で） |

手順6の「割引やプレゼントは決して提案しない」というルールは、文面のトーンのためのものではありません。FTCは2024年8月に消費者レビューに関する規則を発表しました（[FTCの発表](https://www.ftc.gov/news-events/news/press-releases/2024/08/federal-trade-commission-announces-final-rule-banning-fake-reviews-testimonials)）。この規則は、レビューで特定の肯定的または否定的な意見を示すことを条件にインセンティブを提供することを禁止しています。また、悪いレビューを削除させるための脅迫や威圧も禁止しています（[16 CFR Part 465](https://www.ecfr.gov/current/title-16/chapter-I/subchapter-D/part-465)）。FTCスタッフによるQ&Aによると、この規則は2024年10月21日に施行され、裁判所は民事制裁金を科すことができます（[FTC規則のQ&A](https://www.ftc.gov/business-guidance/resources/consumer-reviews-testimonials-rule-questions-answers)）。事業者に代わってレビューを書いたり評判を管理したりする企業も、責任を問われる可能性があります（[連邦官報の本文](https://www.federalregister.gov/documents/2024/08/22/2024-18519/trade-regulation-rule-on-the-use-of-consumer-reviews-and-testimonials)）。

Googleのポリシーでは、レビューの投稿、編集、削除と引き換えに、無料または割引価格で商品を提供することを禁止しています。店舗側は、否定的なレビューの投稿を思いとどまらせたり、良いレビューを投稿してくれそうな顧客だけに依頼したりしてはいけません（[Googleマップの禁止および制限されているコンテンツ](https://support.google.com/contributionpolicy/answer/7400114)）。返信については、プライバシーを守ること、個人攻撃をしないこと、誠実であること、適時に返信することをGoogleは推奨しています（[クチコミを増やすためのヒント](https://support.google.com/business/answer/3474122)）。Yelpはさらに踏み込み、顧客にレビューを依頼すること自体を推奨していません（[Yelp: Don't Ask for Reviews](https://www.yelp-support.com/article/Don-t-Ask-for-Reviews?l=en_US)）。Duetが返信の下書きを作成する場合、自動化されたツールがあなたの代わりに発言することになるため、投稿前に必ず各下書きを確認してください。ここではこれらの情報源を引用しているだけであり、法的助言ではありません。

以下は、Flitch Coffeeの実データをもとにしたブリーフの例です。9月27日に投稿された星2のGoogleレビューを、昨日届いたものとして使っています。

> 今日対応すべき低評価レビューが1件あります。Googleマップ、星2、9月27日：お客様は、コールドブリューが価格の割においしくなかったと書き、写真を添付しています。まだ返信していません。返信案（角括弧内はご自身で記入してください）：「ご意見をお寄せいただきありがとうございます。コールドブリューがご期待に沿えず申し訳ございません。[変更した点があれば記入]」今週の競合店：East Austinでは、The Brew and Brew（レビュー1,186件）とDesnudo Coffee（938件）がレビュー数で上位です。

## 実際の1往復：モデルが送信する内容とAPIが返す内容

Duet側の動作は私たちには実行できません。ただし、Duetがコネクタ経由で送信するリクエストは、標準的なMCPの`tools/call`です。シンガポール時間2026年11月5日0時24分に、同じリクエストをリモートMCPエンドポイントへ私たち自身で送信しました。このリクエストは、10月1日以降のChun Yang TeaのGoogleレビューを取得します。リクエストボディの全文を以下に示します。任意のMCPクライアントに貼り付ければ、同じテストを再現できます。

```text
{"jsonrpc":"2.0","id":7,"method":"tools/call","params":{"name":"everyinfra_call_api",
 "arguments":{"platform":"google_maps_reviews","action":"reviews",
  "params":{"url":"https://www.google.com/maps/search/?api=1&query=Chun%20Yang%20Tea%20Flushing&query_place_id=ChIJnzbAWQ5hwokRCFXB8oE55cg",
            "sort":"newest","since":"2026-10-01","limit":20}}}}
```

レスポンスはわずか1,274バイトでした。`results`は空の配列で、`count`は0でした。`billing.charged`はfalse、`charged_usd`は0.0、`reason`は`empty_result_refunded`です。`refund.status`は`succeeded`で、保留されていた100クレジットはすぐに戻りました。シンガポール時間00:26には、RESTでFlitch Coffeeの2026-10-15以降のレビューを取得しました。返ってきたのは2件で、いずれも星5つ・オーナーの返信なし、費用は$0.001389でした。

以下の2枚のカードは、残高フィールドだけを除いた生のレスポンスです。

![ダークテーマのコードカード2枚を示すスクリーンショット。左のカードは、sinceを2026-10-01に設定したChun Yang TeaのMCPレスポンスです（resultsは空、countは0、has_moreはfalse、stop_reasonはend_of_list）。chargedはfalse、charged_usdは0.0、reasonはempty_result_refunded、refund statusはsucceeded、creditsは100と表示されています。右のカードは、sinceを2026-10-15に設定したFlitch CoffeeのRESTレスポンスです。countは2で、2件とも星5つ、投稿日は2026-10-22と2026-10-18、owner_responseはnull、chargedはtrue、charged_usdは0.001389と表示されています。](https://everyinfra.com/articles/claude-duet-shop-owner-google-maps-reviews/img-api-since-response.webp)

*同じ機能から得た実際のレスポンス2件：新着レビューがなければ課金なし、2件なら1回分の料金* [EveryInfra APIのレスポンス、シンガポール時間2026-11-05 00:24および00:26](https://api.everyinfra.com/api/v1/social/catalog)

この2回のやり取りから2つのことがわかります。第1に、「空の結果は課金なし」は、数日かかる返金という意味ではありません。同じレスポンスに返金成功と書かれているので、Duetは`count: 0`を読んだ時点で処理を止められます。第2に、結果がある呼び出しは件数ではなく呼び出し単位で課金されます。2件でも50件でも$0.001389なので、`limit: 20`にしても追加料金はかかりません。フィールドはカタログの`response_fields`一覧と一致しています。`reviewer_name`は個人のニックネームで、ブリーフには不要です。

| 時刻（シンガポール時間） | 呼び出し | 件数 | 費用 |
| --- | --- | --- | --- |
| 00:17 | `google_maps_reviews/reviews`、Chun Yang Tea、最新50件 | 50 | $0.001389 |
| 00:20 | `google_maps_reviews/reviews`、Flitch Coffee、最新50件 | 50 | $0.001389 |
| 00:24 | MCP `everyinfra_call_api`、Chun Yang Tea、`since=2026-10-01` | 0 | $0 |
| 00:26 | REST、Flitch Coffee、`since=2026-10-15` | 2 | $0.001389 |

テスト中には失敗もありました。00:22、00:24、00:25のリクエストは429を返しました。このテスト用キーは1分あたり10回までの呼び出しに制限されており、その時間帯は他のジョブも同じキーを共有していたためです。60秒後の再試行は成功し、429は課金されません。返金と照合の詳細は[失敗時の返金と照合](https://everyinfra.com/build/failure-refund-reconciliation)をご覧ください。

## テスト1：小規模店舗では新しいGoogleレビューがどれほど少ないか

各店舗につき1回、`sort=newest`と`limit=50`で`google_maps_reviews/reviews`を呼び出しました。そのうえで、各レビューの`posted_at`をさかのぼって集計しました。3回とも、2026年11月5日のシンガポール時間00:17〜00:22に実行しました。1回あたり$0.001389で、3回の合計は$0.004167です。

| 店舗 | Google評価／レビュー数 | 最新のレビュー | 直近30日 | 直近90日 | 90日間で新規レビューがあった日数 | 50件のレビューの期間 |
| --- | --- | --- | --- | --- | --- | --- |
| Flitch Coffee（East Austin） | 4.7 / 475 | 2026-10-22（13.5日前） | 2 | 8 | 7日 | 551日 |
| Chun Yang Tea（Flushing） | 4.4 / 229 | 9月23日（41.8日前） | 0 | 8 | 8日 | 約6.9年（順序が乱れた古いレビュー2件を含む） |
| Taquería Chabelita（East End, Houston） | 4.6 / 217 | 2026-10-12（22.9日前） | 2 | 5 | 4日 | 1,291日 |

![店舗ごとに、直近30日・90日・180日の新しいGoogleレビュー数を3本の棒で示したグループ棒グラフ。Chun Yang Teaは0、8、16件、Flitch Coffeeは2、8、16件、Taquería Chabelitaは2、5、9件です。90日の棒はティール色です。サブタイトルには、最新のレビューが13.5〜41.8日前のものだったと記載されています。データは2026年11月5日、シンガポール時間00:17〜00:27に実際に行った呼び出しから得たものです。](https://everyinfra.com/articles/claude-duet-shop-owner-google-maps-reviews/fig-03-new-reviews-bar.ja.light.svg)

これらの数字がブリーフの設計を左右します。直近90日間で、Flitch Coffeeに新しいレビューがついたのは約13日に1日、Taquería Chabelitaは約22日に1日でした。毎日「今日は新しい低評価レビューはありません」というメッセージを送ると、オーナーには月に20回以上届くことになり、すぐに読まれなくなるでしょう。Duetは対応が必要なときにだけ連絡するように作られている（[製品ページ](https://claude.com/product/claude-duet)）ため、プロンプトのステップ1では`count: 0`のときは何も通知しません。

もう1つ重要な点があります。50件のレビューの中には、星だけで本文のないものがあります。Flitch Coffeeで17件、Chun Yang Teaで6件、Taquería Chabelitaで20件です。本文のない星1〜2のレビューも、低評価レビューであることに変わりありません。具体的な指摘に答えることはできませんが、ブリーフでは必ずフラグを立てるべきです。長い返信の下書きは必要ありません。

これらの数字には限界があります。3店舗は公開の検索結果から選んだもので、いずれも当社とは一切関係がありません。レビュー数は季節によって変わり、今回の90日間は晩夏から秋にかけての期間です。Googleは最終編集日を表示するため、集計には表示された日付を使っています。当社の記事[Claude Managed Agentsによる夜間レビュー監視](https://everyinfra.com/blog/claude-managed-agents-nightly-review-monitoring)では、Katz's Delicatessenの最新100件のGoogleレビューが4日未満の期間に収まっていました。大型のデリと小さなコーヒートレーラーでは、ブリーフの設計を変える必要があります。

## テスト2：低評価レビューに誰かが返信しているか

同じ150件のレビューのうち、`owner_response`があるレビューを数えました。次に、星1〜2のレビューだけを取り出して確認しました。下の表は、店舗ごとの返信率、返信までの時間の中央値、最新の低評価レビューを示しています。低評価レビューの返信フィールドが空であることこそ、ブリーフで最上位に挙げるべきシグナルです。

| 店舗 | 50件中の返信数 | 星1〜2のレビューへの返信 | 返信までの時間（中央値） | 最新の返信 | 最新の低評価レビュー |
| --- | --- | --- | --- | --- | --- |
| Flitch Coffee | 50件中13件 | 4件中0件 | 約130時間 | 5月31日 | 9月27日、星2つ、返信なし |
| Chun Yang Tea | 50件中43件 | 5件中4件 | 約47時間 | 9月24日 | 8月31日、星1、返信なし |
| Taquería Chabelita | 50件中0件 | 1件中0件 | none | 50件中なし | 2025年3月11日、星2、返信なし |

![3店舗の返信率を示す横向きのグループ棒グラフ。Chun Yang Teaは50件中43件のレビューに返信し、星1〜2のレビューには5件中4件に返信しています。Flitch Coffeeは50件中13件のレビューに返信し、星1〜2のレビューには4件中0件に返信しており、最後の返信は5月31日です。Taquería Chabelitaの返信は50件中0件で、星1〜2のレビューでも1件中0件です。星1〜2の棒はティール色で、データは2026年11月5日（シンガポール時間）に実際に行ったAPI呼び出しによるものです。](https://everyinfra.com/articles/claude-duet-shop-owner-google-maps-reviews/fig-04-reply-rate-bar.ja.light.svg)

3店舗には、よくある3つのパターンが表れています。Flitch Coffeeは5月31日まではまとめてレビューに返信していましたが、その後は返信をやめています。9月13日と9月27日の星2のレビュー2件は、どちらもコーヒーがまずいという内容です。どちらにもまだ返信はありません。2025年5月の星1のレビューには飲み物にハエが入っていたと書かれていますが、これにも返信はありません。

Chun Yang Teaはほぼすべてのレビューに返信しており、その多くは短いお礼です。ただし、8月31日の星1のレビュー（トイレもWi-Fiもない）には返信がありませんでした。その前後の星5のレビューには返信しています。Googleのヒントでは、全員に同じお礼を返すより、適切なレビューに役立つ返信をするほうが効果的だとされています（[レビューを増やすためのヒント](https://support.google.com/business/answer/3474122)）。Taquería Chabelitaは家族経営の小さな店で、3年半にわたって返信がありません。このデータの中で悪いレビューは1件だけです。

ブリーフィングには、星の評価よりも`owner_response`列のほうが役立ちます。誰も対応していない悪いレビューを上位に押し上げてくれるからです。DuetはBusiness Profileのコネクタを持たないため、返信の下書きはできますが、Googleには投稿できません。Business Profileにログインし、自分で返信を投稿する必要があります（[クチコミの管理](https://support.google.com/business/answer/3474050)）。Googleは返信を公開前に審査します。通常は10分以内ですが、最大30日かかる場合もあります。顧客には通知が届き、その後、顧客がレビューを編集することもあります。

## テスト3：Googleに表示される内容とYelpに表示される内容

Googleだけで十分でしょうか。各店舗について`yelp/search`を1回呼び出してビジネスページを探し、次に`yelp/reviews`を1回呼び出して1ページ目の10件のレビューを読みました。6回の呼び出しはシンガポール時間の00:18から00:23までに実行され、費用は合計$0.008334でした。下の表は、3店舗についてYelpの1ページ目を比較したものです。

| 店舗 | Yelpの評価／レビュー数 | 1ページ目の最新レビュー | 1ページ目、直近90日 | 1ページ目、星1〜2 | 1ページ目は日付順？ |
| --- | --- | --- | --- | --- | --- |
| Flitch Coffee | 4.6 / 134 | 2026-10-29 | 4 | 1件（7月12日、星2） | 不要 |
| Chun Yang Tea | 4.4 / 250 | 9月29日 | 1 | 2件（9月29日、星2；2025年4月25日、星1） | 不要 |
| Taquería Chabelita | 4.6 / 13 | 2026年6月19日 | 0 | 0 | 不要 |

![Flitch Coffeeの2026年5月から11月までのタイムライン散布図（Google Mapsの行とYelpの1ページ目の行）。Googleでは5月以降に16件のレビューがあります。そのうち2件は星1〜2で、9月13日と9月27日のものです。これらは大きなティール色の点で示されています。Yelpの1ページ目では5月以降に5件のレビューがあり、7月12日の星2のレビュー1件がティール色で、最新のレビューは10月29日です。データは2026年11月5日（シンガポール時間）に実際に行ったAPI呼び出しによるものです。](https://everyinfra.com/articles/claude-duet-shop-owner-google-maps-reviews/fig-05-google-yelp-timeline.ja.light.svg)

Flitch Coffeeでは、2つのサイトが示す内容が異なります。Googleでは、最新のレビューは10月22日、最新の悪いレビューは9月のものです。Yelpでは最新のレビューが10月29日で、Googleのどのレビューよりも1週間新しくなっています。Yelpの悪いレビューは7月12日の星2で、コーヒーのミルクが多すぎると書かれています。Googleだけのブリーフィングでは、このレビューは見えません。

Chun Yang Teaでは、このリスクがよりはっきり表れています。Googleの最新レビューは9月23日の星5で、Googleの最新の悪いレビューは8月31日のものです。Yelpでは、9月29日の星2のレビューに、30分近く待たされたと書かれています。その間、できあがった飲み物がカウンターに置かれたままでした。このレビューは同店のGoogleのどのレビューよりも新しく、直近の悪いレビューはこれだけです。Googleだけのブリーフィングなら、9月30日の朝に「新しい悪いレビューはなし」と伝えていたはずです。

Yelp側には、設計時に考慮すべき制約が2つあります。第1に、`yelp/reviews`には`since`パラメータがないため、毎回1ページ分すべてが返り、$0.001389かかります。Yelpを毎日読むと、月額料金で最大の項目になります。第2に、3店舗のいずれでも1ページ目は日付順になっていません。最新のレビューが2番目や3番目、さらには2ページ目にあることもあります。Katz'sでは、その記事で1ページ目の最も古いレビューより新しいレビューが2ページ目に5件見つかりました。

Yelpのレビューが月に数件程度の店舗なら、取るべき方針は明確です。1ページ目を週2回読み、「新しい」かどうかはidで判断します。コストは低く、見落としもほとんどありません。レビューの多い店舗では、毎回2ページ読むべきです。下のコストのセクションでは週2回のプランを使っています。

## テスト4：1回の検索で近隣の競合20店舗を調べる

競合店の情報を最も低コストで取得できるのは`google_maps_reviews/search`です。1回の呼び出しで、`name`、`rating`、`review_count`、`permanently_closed`を含む20店舗の情報が$0.001389で返されます。シンガポール時間の00:19に、「East Austin, TX」で「coffee shop」を検索し、20店舗を取得しました。Flitch Coffeeの評価4.7は、評価順で20店舗中15位（ほかの3店舗と同率）、レビュー数475件は8位でした。20店舗の中央値は評価4.8、レビュー数207件です。レビュー数の上位はThe Brew and Brew（1,186件、評価4.4）、Desnudo Coffee（938件、評価4.8）、Figure 8 Coffee Purveyors（806件、評価4.7）でした。

![x軸にGoogleのレビュー数（対数スケール）、y軸に評価をとった散布図。20個のグレーの点は、East Austinのコーヒーショップ20店舗です。大きなティール色の点はFlitch Coffeeで、評価4.7（20店舗中15位）、レビュー数475件（20店舗中8位）です。データは2026年11月5日（シンガポール時間）に実際に行ったAPI呼び出しによるものです。](https://everyinfra.com/articles/claude-duet-shop-owner-google-maps-reviews/fig-06-competitor-scatter.ja.light.svg)

1回のスナップショットでは、順位はわかっても変化はわかりません。時間の経過による変化を見るため、シンガポール時間の01:08〜01:09に同じ3つの検索をもう一度実行しました。これは最初の実行から約50分後で、ニューヨークでは2026-11-04の正午ごろに当たります。3回の呼び出しの費用は$0.004167でした。

| エリア（キーワード） | 両方のリストにある店舗 | 評価または件数の変化 | 順位が変わった店舗 | 最大の順位変動 | サンプル店舗の検索結果での順位 |
| --- | --- | --- | --- | --- | --- |
| Flushing（バブルティー） | 20店舗中20店舗 | 0 | 11 | 8位分 | Chun Yang Tea、11位→12位 |
| East Austin（コーヒーショップ） | 20店舗中20店舗 | 0 | 5 | 2位分 | Flitch Coffee、16位→16位 |
| East End, Houston（タケリア） | 20店舗中16店舗 | 0 | 15 | 8位分 | Taquería Chabelita、11位→15位 |

50分の間に、両方のリストにある56店舗のうち、評価やレビュー数が変わった店舗は1つもありませんでした。これはテスト1の結果と一致します。小さな店舗には月に数件しか新しいレビューがつきません。変わったのは順番です。Flushingでは20店舗中11店舗の順位が動き、Houstonでは4店舗がリストから外れました。たとえば、Bodegas Taco Shop（レビュー2,526件、14位）がリストから外れ、レビュー3件の新しい店舗が入ってきました。

つまり、検索結果の順番は安定せず、リストそのものも安定しません。「検索での順位」を競合の指標に使わないでください。「今週リストにない」を「閉店した」と解釈しないでください。最初の週に、競合店の`place_id`の値を固定リストとして保存します。その後は毎週、`place_id`ごとに`rating`と`review_count`だけを比較します。店舗が見つからない場合は、別のキーワードで再検索するか、`permanently_closed`フィールドを確認します。

この設計なら、競合を監視するコストは競合の数が増えても増えません。20店舗で1回の検索あたり$0.001389なので、週1回の検索で月に約$0.006です。各競合のレビュー本文を読むとコストはずっと高くなり、20店舗を1回ずつ呼び出すと$0.0278かかります。そのため、ブリーフィングで競合のレビューを読むのは、評価が動いたときかレビュー数が急に増えたときだけにします。それ以外の週は件数だけを確認します。

## APIの日額・月額料金

料金は無料のカタログ`https://api.everyinfra.com/api/v1/social/catalog`から、2026年11月5日のシンガポール時間00:12に取得したものです。4つの機能はいずれも`available`で、1回の呼び出しあたり100クレジット、つまり$0.001389（1,000回の呼び出しあたり$1.39）です。以下の4つのプランはいずれも30日で計算しています。Googleの`since`の行には上限値を使っています。新しいレビューは、投稿日と翌日の2回の朝に表示される可能性があるため、「新しいレビューがあった日数×2」で課金額を計算しています。新しいレビューがない朝は費用がかかりません。

| 方式 | 月間呼び出し数 | 月額API料金 | 備考 |
| --- | --- | --- | --- |
| バックグラウンドチェックのたびに3つすべてを全件読み込み | 540 | $0.7501 | 1日6回のチェックを想定。Duetはチェック間隔を公表していないため、これは仮定です |
| 毎朝3つすべてを全件読み込み | 90 | $0.1250 | Googleのレビュー50件、Yelpの1ページ目、競合検索 |
| Googleはsinceを指定、Yelpは毎日、競合は毎週 | 約39.0 | $0.0541 | Yelpにはsinceがないため、毎日課金されます |
| 推奨：Googleはsinceを指定、Yelpは週2回、競合は毎週 | 約17.5 | $0.0243 | Flitch Coffeeに新しいレビューがあったのは、直近90日のうち7日です |

![4つのプランごとのFlitch Coffeeの月額API料金を示す横棒グラフ。バックグラウンドチェックのたびに全件読み込みする場合（1日6回を想定）は0.7501米ドルです。毎朝3つを全件読み込みする場合は0.1250ドル、sinceの指定と毎日のYelpと毎週の競合検索を組み合わせた場合は0.0541ドルです。推奨プラン（since、Yelpは週2回、競合は毎週）は0.0243ドルで、ティール色で表示しています。計算には1回の呼び出しあたり0.001389ドルのカタログ価格を使い、30日分で算出しています。](https://everyinfra.com/articles/claude-duet-shop-owner-google-maps-reviews/fig-07-monthly-cost-bar.ja.light.svg)

どのプランも月1ドル未満に収まり、それより重要なコストがほかに2つあります。1つ目はDuet自体です。リサーチプレビュー期間中は、Pro、Max、Ultraに追加料金なしで含まれています（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。Proは月払いで月額$20です（[料金ページ](https://claude.com/pricing)）。2つ目はプランの使用量です。API料金よりも、こちらを抑えるほうが重要です。

ヘルプページによると、Duetの利用は、Claudeのほかの機能と同じ割合で、5時間ごとおよび週ごとの利用上限にカウントされます。朝のブリーフィングとバックグラウンドチェックはどちらもカウントされます（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)、[使用量と長さの制限](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work)）。Flitch Coffeeの最新レビュー50件を含む生のレスポンスを保存したファイルは44,564バイトでした。空の`since`レスポンスは1,274バイトで、約35分の1のサイズです。毎朝全件を読み込むと、古いレビューの長いブロックが不必要にDuetのコンテキストへ送り込まれます。推奨プランなら、API料金と使用量の両方を節約できます。

| 費用項目 | 金額 | 出典 |
| --- | --- | --- |
| Claude Proプラン | 月払いで月額$20 | Claudeの料金ページ |
| プレビュー期間中のDuet | 追加料金なし | Duetのヘルプページ |
| EveryInfra API（推奨プラン） | 月額約$0.0243 | カタログ価格×呼び出し数 |
| プランの使用量 | Claudeのほかの機能と共有 | 使用量制限のヘルプページ |

複数の店舗を運営している場合は、店舗数を掛けてください。推奨プランで10店舗なら、API料金は月額約$0.24です。競合検索はエリア単位なので、同じエリアの店舗は1回の検索を共有できます。同じ通りにある3台のコーヒートレーラーなら、毎週の検索は1回で済みます。GoogleとYelpの読み込みは店舗ごとに行います。

## Claude Duet vs Muse：3つのプロアクティブアシスタントを比較

Claude Duet、Meta Muse、ChatGPT dotsは、いずれも2026年9月から10月にかけてリリースされました。3つとも、アシスタント側から先に連絡することをうたっています。表には各出典に記載されている内容のみを掲載しており、「非公開」は出典に記載がないことを意味します。すべての出典は2026年11月5日に確認しました。

| 項目 | Claude Duet | Meta Muse | ChatGPT dots |
| --- | --- | --- | --- |
| リリース | 2026-10-21にリサーチプレビューとして公開、ウェイトリスト制（[発表記事](https://claude.com/resources/articles/claude-duet)） | 2026-09-08にリリース（[Meta Newsroom](https://about.fb.com/news/2026/09/introducing-muse-personal-ai-agent/)）。2026-09-29に中小企業向けのスキルとコネクタを追加（[Muse for Small Business](https://about.fb.com/news/2026/09/introducing-muse-small-business/)） | 2026-09-29にDevDayで発表。2026-11-03にGPT-6.1 Terraモデルを追加（[Introducing dots](https://openai.com/index/introducing-dots/)） |
| 利用対象 | 米国のPro、Max、Ultra。TeamとEnterpriseは対象外 | 米国とカナダ。ほとんどの機能は無料で、サブスクリプションもあり | ProとBusiness Premium、18歳以上。個人向けProはEEA、スイス、英国では利用不可（[ヘルプセンター](https://help.openai.com/en/articles/20001530-getting-started-with-your-dot)） |
| 料金 | プランに含まれる。Proは月額$20から | 使用量の上限付きで無料。Powerは月額$20、Maximumは月額$100（[Museのサブスクリプション](https://www.meta.com/help/subscriptions/1021145227643680/)） | Proは月額$100から（[dotsの記事](https://everyinfra.com/blog/chatgpt-dots-product-research-radar)を参照） |
| 連絡の仕方 | 朝のブリーフィング。数時間ごとにチェックし、対応が必要なことがあればメッセージを送信 | 変化があったときや承認が必要なときに連絡。毎週または毎日のタスクを実行可能（[Meta Business Help](https://www.facebook.com/business/help/38565157129799046)） | アイドル時に自発的なリサーチを行い、アイデアや質問を持ち帰る。割り当てたタスクはスケジュール実行が可能（[開発者向けドキュメント](https://developers.openai.com/codex/dots/tasks-and-memory)） |
| カスタムコネクタ | 出典によるとClaudeのコネクタを引き継ぐ。カスタムコネクタへの言及はなし | Museがカスタムコネクタを作成できる。Metaによる審査はなし（[Museのコネクタ](https://www.meta.com/help/artificial-intelligence/1687253048996149/)） | カスタムMCPプラグインに対応（dotsの記事を参照） |
| バックグラウンド処理で呼び出せるもの | 非公開 | 「常に許可」を含め、設定した読み取り・書き込み権限に従う | 自発的なリサーチでは読み取り専用ツールのみを使用 |
| 承認 | メッセージ送信、支払い、署名、削除の前に確認。別途自動チェックあり | 書き込みと重要な読み取りは事前確認に設定可能。外部への連絡は別のSentinelエージェントが審査 | ドキュメントはCodex配下。Codexの自動レビューでは、アノテーションで指定されたMCP呼び出しを含め、承認が必要な呼び出しを別のレビュー用エージェントが確認（[自動レビュー](https://learn.chatgpt.com/docs/sandboxing/auto-review)） |
| メッセージの受信先 | Duetアプリ（iOS、Android）とclaude.ai | Museアプリ、WhatsApp、muse.ai | ChatGPT、Slack、Teams。米国のProユーザーはiMessageとRCSでもやり取り可能（ベータ、[9to5Mac、2026-10-28](https://9to5mac.com/2026/10/28/openai-expanding-chatgpt-dot-access-to-imessage-and-rcs/)） |
| 出典に挙がっている小規模事業者向けの例 | Stripeの未払い請求書、QuickBooks、Gustoの給与計算、Shopify | Instagramプロアカウント、Facebookページ、Meta広告アカウント | 請求書のリマインダー（初期テスターの例） |

低評価レビューの日次レポートという用途では、3つの製品の違いは2点です。1点目は、バックグラウンドの処理から課金対象のデータツールを呼び出せるかどうかです。最も明確なのはdotsで、自発的なリサーチは読み取り専用のため、データの呼び出しはユーザーが割り当てるタスクに含める必要があります。Museには「常に許可」の選択肢があります。Duetには公開されたルールがないため、実際に試す必要があります。

2点目は、組み込みコネクタの連携先です。MuseはMeta自身のFacebookとInstagramに最も深く連携しています。dotsはオフィスツールや開発者ツール寄りです。Duetの例は記帳、請求書、契約書が中心です。3つとも組み込みのGoogleビジネスプロフィールやYelpのコネクタはないため、レビューデータはカスタムコネクタ経由で取り込む必要があります。RedditのあるDuetヘビーユーザーは、受信トレイの仕分け、放置されたスレッドの拾い上げ、返信の下書き、デイリーブリーフィングが得意だと述べています（[Redditスレッド](https://www.reddit.com/r/claude/comments/1xhct14/claude_duet_first_takeaways/)）。

[Claude Managed Agentsによる夜間のレビュー監視](https://everyinfra.com/blog/claude-managed-agents-nightly-review-monitoring)と比べると、店舗の規模は2桁違います。そのため結論は正反対になります。大きな店舗は一晩で大量のレビューを読む必要があり、小さな店舗は毎日の空振りチェックを避ける必要があります。日次アラートではなくレビュー分析が目的なら、[Agent Skillsによる競合レビュー分析](https://everyinfra.com/blog/agent-skills-competitor-review-analysis)を参照してください。

## できないことと注意点

この用途に関係するDuetの制限は4つあります。米国限定かつ個人プラン限定で、ウェイトリスト制のため利用開始日は未定です。本記事のDuetの画面はすべて出典に基づく設定例で、実際のDuetでテストしたものではありません。出典には、Duetでカスタムコネクタが使えるかどうか、`destructiveHint`で承認が求められるかどうかは書かれていません。Duetは一定間隔でチェックするため、チェックの合間に緊急の項目が届くこともあります（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。また、Duetは電話をかけたり他人にテキストメッセージを送ったりしないため、GoogleとYelpへの返信は自分で投稿します。

データ側にも制限があります。`since`は時刻ではなく日付なので、重複はidで除外する必要があります。Googleのレビューにはポリシー審査で数日待たされるものがあり、表示される日付がレビューの公開日より前になることがあります。`since`を前日にすると1日分余分に取得します。2日前にすると取りこぼしは減りますが、重複が増えます。Yelpには`since`がなく、1ページ目は日付順ではありません。今回の3つのサンプルはいずれもレビューの少ない店舗なので、レビューの多い店舗は代わりに[Claude Managed Agentsによる夜間のレビュー監視](https://everyinfra.com/blog/claude-managed-agents-nightly-review-monitoring)を参考にしてください。

| 制約 | 対処法 |
| --- | --- |
| ウェイトリスト制、米国限定 | Claudeアカウントのメールアドレスで申し込む |
| カスタムコネクタへの対応は未記載 | 利用開始の初日に自分のDuetで試す |
| 一定間隔でチェック | 即時アラートには頼らない |
| `since`は時刻ではなく日付 | レビューIDで重複を除外 |
| Yelpは`since`がなく、並び順もばらばら | 週2回1ページ目を読み、IDで判定 |

レビュー投稿者は一般の個人であり、そのニックネームは個人情報です。レポートでレビューを引用するときは、名前を省いてください。本記事でもレビュー投稿者の名前は一切挙げていません。レポートに必要なのは星の数、日付、本文、返信状況だけで、投稿者に関する情報は不要です。このルールにより、小さなスマートフォンの画面でもメッセージを短く保てます。

## 最近の変更点

- **2026-10-21：Claude Duetがリサーチプレビューとして公開**（[発表記事](https://claude.com/resources/articles/claude-duet)）。店舗オーナー向けには、Claudeアプリの側からデフォルトで連絡してくるようになりました。
- **2026-10-30：ClaudeがMax 20xをUltraに名称変更**（[The New Stack](https://thenewstack.io/claude-ultra-session-limit/)、[Ultraのヘルプページ](https://support.claude.com/en/articles/17728952-what-is-the-ultra-plan)）。Duetの3つのプラン区分は以前と変わりません。
- **2026-11-30：新規Duetユーザー向けの利用上限リセットが終了**（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。このリセットは、プロンプトを調整する最初の1週間に活用してください。

## FAQ

### Claude Duetのウェイトリストに登録してから、どのくらいで利用できますか？

この質問は、検索候補の「claude duet waitlist」とRedditのスレッド（[Redditスレッド](https://www.reddit.com/r/ClaudeAI/comments/1xeiaqv/has_anyone_actually_gotten_into_claude_duet_yet_i/)）に基づいています。ヘルプページには、招待は段階的に送られ、日程は未定とだけ書かれています（[ヘルプページ](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)）。最もよくあるつまずきは、ウェイトリストに登録したメールアドレスがClaudeアカウントと一致していないことです。申し込みにはClaudeアカウントのメールアドレスを使ってください。

### Claude DuetとMuse、小規模店舗に合うのはどちら？

この質問は、検索候補の「claude duet vs muse」に基づいています。答えは、顧客と帳簿がどこにあるかで決まります。顧客の多くがInstagramやFacebookにいるなら、Museがそれらのアカウントに最も簡単に接続できます。Museの大部分は無料です（[Muse for Small Business](https://about.fb.com/news/2026/09/introducing-muse-small-business/)）。帳簿がQuickBooksやStripeにあり、すでにClaudeを使っているなら、Duetのほうがセットアップの手間が1つ減ります。どちらにもGoogleマップやYelpの組み込みコネクタはありません。

### Claude Duetの評判：実際のユーザーの声は？

この質問は、検索候補の「claude duet review」と「claude duet reddit」に基づいています。Redditでの意見は2つに分かれています。一方は、デイリーブリーフィング、対応漏れになったスレッドの拾い上げ、返信の下書きが得意だという意見です（[Redditスレッド](https://www.reddit.com/r/claude/comments/1xhct14/claude_duet_first_takeaways/)）。もう一方は、Claude CodeやCoworkからタスクを引き継げないので作り直す必要があるという意見です（[Redditスレッド](https://www.reddit.com/r/ClaudeAI/comments/1xgu97b/whats_the_point_of_claude_duet/)）。新着レビューのレポートについては、2つ目の点はほとんど影響しません。

### 新着レビューがない朝もAPI料金はかかりますか？

この質問は、私たち自身のライブテストから生まれたものです。`since`を指定したGoogleレビューの読み取りで0件が返った場合、課金されません。レスポンスには`empty_result_refunded`と返金の成功が示されます。Yelpには`since`がないため、読み取りのたびに$0.001389かかります。そのため提案した運用では、Yelpを週2回だけ読み取ります。

## 始め方

1. EveryInfraに新規登録してAPIキーを取得します。[クイックスタート](https://everyinfra.com/docs#quickstart)に沿って進め、[プレイグラウンド](https://everyinfra.com/market)で`google_maps_reviews/search`を一度実行します。自店舗と近隣の競合店のリンクを控え、Yelpのページには`yelp/search`を使います。
1. 自店舗に対して、`since`を1週間前に設定して`google_maps_reviews/reviews`を一度呼び出します（[MCPサーバーのページ](https://everyinfra.com/mcp-server)）。[Google Maps ReviewsスクレイピングAPI](https://everyinfra.com/google-maps-reviews-api)と[reviews機能](https://everyinfra.com/google-maps-reviews-api/reviews)のページを読みます。あわせて[search機能](https://everyinfra.com/google-maps-reviews-api/search)、[YelpスクレイピングAPI](https://everyinfra.com/yelp-api)、[Yelpのreviews機能](https://everyinfra.com/yelp-api/reviews)のページも読みます。
1. Duet専用のAPIキーを別に作成します。`google_maps_reviews/*`と`yelp/*`のみを許可し、合計上限を$2に設定します。
1. 上記の設定例をもとに、Claudeでカスタムコネクタを追加します。`everyinfra_call_api`を「常に許可」に、使わないツールを「ブロック」に設定します。
1. 招待が届いたら、上記のタスクをDuetに送ります。最初の1週間は、毎日EveryInfraのコンソールで呼び出しログを確認します。呼び出しが1日1回であること、空の結果に課金されていないことを確認してください（[料金ページ](https://everyinfra.com/pricing)）。

## 参考文献

1. Anthropic：[Claude Duetの紹介（発表、2026-10-21）](https://claude.com/resources/articles/claude-duet)、2026-11-05閲覧
1. XのClaude：[Claude Duetのローンチ投稿（2026-10-21）](https://x.com/claudeai/status/2112984669943968099)、2026-11-05閲覧
1. CNBC：[AnthropicがパーソナルAIエージェントClaude Duetをプレビュー公開（2026-10-21）](https://www.cnbc.com/2026/10/21/anthropic-preview-of-claude-duet-personal-ai-agent-take-on-muse-dots.html)、2026-11-05閲覧
1. Claudeヘルプセンター：[Claude Duetを使い始める](https://support.claude.com/en/articles/17631495-get-started-with-claude-duet)、2026-11-05閲覧
1. Anthropic：[Claude Duetの安全性、プライバシー、セキュリティ](https://claude.com/resources/articles/claude-duet-safety-privacy-and-security)、2026-11-05閲覧
1. Anthropic：[Claude Duetの製品ページとウェイトリスト](https://claude.com/product/claude-duet)、2026-11-05閲覧
1. The New Stack：[Claude Ultraプランとセッション上限に関するレポート（2026-10-30）](https://thenewstack.io/claude-ultra-session-limit/)、2026-11-05閲覧
1. Claudeヘルプセンター：[Ultraプランとは？](https://support.claude.com/en/articles/17728952-what-is-the-ultra-plan)、2026-11-05閲覧
1. Anthropic：[Claudeの料金ページ](https://claude.com/pricing)、2026-11-05閲覧
1. Claudeヘルプセンター：[Proプランとは？](https://support.claude.com/en/articles/8325606-what-is-the-pro-plan)、2026-11-05閲覧
1. Claudeヘルプセンター：[Maxプランとは？](https://support.claude.com/en/articles/11049741-what-is-the-max-plan)、2026-11-05閲覧
1. Claudeヘルプセンター：[制限のリセットとは？](https://support.claude.com/en/articles/17007452-what-is-a-limit-reset)、2026-11-05閲覧
1. Reddit r/ClaudeAI：[スレッド：Claude Duetを実際に使えた人はいる？](https://www.reddit.com/r/ClaudeAI/comments/1xeiaqv/has_anyone_actually_gotten_into_claude_duet_yet_i/)、2026-11-05閲覧
1. Anthropic：[Claude Marketplaceのコネクタとプラグイン](https://claude.com/marketplace/connectors-plugins)、2026-11-05閲覧
1. Reddit r/claude：[スレッド：Claude Duetの第一印象](https://www.reddit.com/r/claude/comments/1xhct14/claude_duet_first_takeaways/)、2026-11-05閲覧
1. Claudeヘルプセンター：[コネクタでClaudeの機能を拡張する](https://support.claude.com/en/articles/11176164-use-connectors-to-extend-claude-s-capabilities)、2026-11-05閲覧
1. Claudeヘルプセンター：[リモートMCPでカスタムコネクタを使い始める](https://support.claude.com/en/articles/11175166-get-started-with-custom-connectors-using-remote-mcp)、2026-11-05閲覧
1. Yelpサポート：[レビューの表示順はどのように決まりますか？](https://www.yelp-support.com/article/How-is-the-order-of-reviews-determined?)、2026-11-05閲覧
1. Googleビジネスプロフィールヘルプ：[クチコミが表示されない、または表示が遅れる](https://support.google.com/business/answer/10313341)、2026-11-05閲覧
1. Googleビジネスプロフィールヘルプ：[顧客のクチコミを管理する](https://support.google.com/business/answer/3474050)、2026-11-05閲覧
1. Google Maps Platform：[Places API (New) Placeリソースのリファレンス](https://developers.google.com/maps/documentation/places/web-service/reference/rest/v1/places)、2026-11-05閲覧
1. Yelp開発者ドキュメント：[Reviewsエンドポイントのリファレンス](https://docs.developer.yelp.com/reference/v3_business_reviews)、2026-11-05閲覧
1. Google Business Profile API：[クチコミデータを扱う](https://developers.google.com/my-business/content/review-data)、2026-11-05閲覧
1. Googleビジネスプロフィールヘルプ：[通知を管理する](https://support.google.com/business/answer/7198436?hl=en)、2026-11-05閲覧
1. Claudeドキュメント：[ディレクトリにないコネクタを追加する](https://claude.com/docs/connectors/custom/add-unlisted)、2026-11-05閲覧
1. EveryInfra：[OpenAPIドキュメント](https://api.everyinfra.com/openapi.json)、2026-11-05閲覧
1. Model Context Protocol：[2025-11-25版仕様：ツール](https://modelcontextprotocol.io/specification/2025-11-25/server/tools)、2026-11-05閲覧
1. MCPブログ：[ツールアノテーション（2026-03-16）](https://blog.modelcontextprotocol.io/posts/2026-03-16-tool-annotations/)、2026-11-05閲覧
1. Anthropic Engineering：[Claude Codeのオートモードをどう構築したか](https://www.anthropic.com/engineering/claude-code-auto-mode)、2026-11-05閲覧
1. OpenAI：[dotsの紹介（2026-11-03の更新を含む）](https://openai.com/index/introducing-dots/)、2026-11-05閲覧
1. Metaヘルプセンター：[Museのガイダンスと承認](https://www.meta.com/help/artificial-intelligence/1385290430137537/)、2026-11-05閲覧
1. 米国連邦取引委員会：[偽のレビューと体験談を禁止する最終規則（2024-08-14）](https://www.ftc.gov/news-events/news/press-releases/2024/08/federal-trade-commission-announces-final-rule-banning-fake-reviews-testimonials)、2026-11-05閲覧
1. eCFR：[16 CFR Part 465](https://www.ecfr.gov/current/title-16/chapter-I/subchapter-D/part-465)、2026-11-05閲覧
1. 米国連邦取引委員会：[消費者レビューおよび体験談に関する規則：質問と回答](https://www.ftc.gov/business-guidance/resources/consumer-reviews-testimonials-rule-questions-answers)、2026-11-05閲覧
1. 連邦官報：[消費者レビューおよび体験談の利用に関する取引規制規則（2024-08-22）](https://www.federalregister.gov/documents/2024/08/22/2024-18519/trade-regulation-rule-on-the-use-of-consumer-reviews-and-testimonials)、2026-11-05閲覧
1. Googleマップの投稿ポリシー：[禁止および制限されているコンテンツ](https://support.google.com/contributionpolicy/answer/7400114)、2026-11-05閲覧
1. Googleビジネスプロフィールヘルプ：[クチコミを増やすためのヒント](https://support.google.com/business/answer/3474122)、2026-11-05閲覧
1. Yelpサポート：[レビューを依頼しないでください](https://www.yelp-support.com/article/Don-t-Ask-for-Reviews?l=en_US)、2026-11-05閲覧
1. EveryInfra：[機能カタログ（料金とパラメータ）](https://api.everyinfra.com/api/v1/social/catalog)、2026-11-05閲覧
1. Claudeヘルプセンター：[使用量と長さの制限の仕組み](https://support.claude.com/en/articles/11647753-how-do-usage-and-length-limits-work)、2026-11-05閲覧
1. Metaニュースルーム：[Museの紹介（2026-09-08）](https://about.fb.com/news/2026/09/introducing-muse-personal-ai-agent/)、2026-11-05閲覧
1. Metaニュースルーム：[スモールビジネス向けMuse（2026-09-29）](https://about.fb.com/news/2026/09/introducing-muse-small-business/)、2026-11-05閲覧
1. OpenAIヘルプセンター：[dotの使い始め方](https://help.openai.com/en/articles/20001530-getting-started-with-your-dot)、2026-11-05閲覧
1. Metaヘルプセンター：[Museのサブスクリプション](https://www.meta.com/help/subscriptions/1021145227643680/)、2026-11-05閲覧
1. Metaビジネスヘルプセンター：[Museのスケジュール済みタスク](https://www.facebook.com/business/help/38565157129799046)、2026-11-05閲覧
1. OpenAI開発者ドキュメント：[dots：タスクとメモリ](https://developers.openai.com/codex/dots/tasks-and-memory)、2026-11-05閲覧
1. Metaヘルプセンター：[Museのコネクタ](https://www.meta.com/help/artificial-intelligence/1687253048996149/)、2026-11-05閲覧
1. ChatGPT Learn：[自動レビュー](https://learn.chatgpt.com/docs/sandboxing/auto-review)、2026-11-05閲覧
1. 9to5Mac：[dotsがiMessageとRCSに対応（2026-10-28）](https://9to5mac.com/2026/10/28/openai-expanding-chatgpt-dot-access-to-imessage-and-rcs/)、2026-11-05閲覧
1. Reddit r/ClaudeAI：[スレッド：Claude Duetって何のためにあるの？](https://www.reddit.com/r/ClaudeAI/comments/1xgu97b/whats_the_point_of_claude_duet/)、2026-11-05閲覧

---

:::message
この記事は EveryInfra ブログの記事の転載です。原文：[店舗オーナー向けClaude Duet：新着の低評価レビューを毎朝まとめて受け取る](https://everyinfra.com/ja/blog/claude-duet-shop-owner-google-maps-reviews)

EveryInfra は AI エージェントと開発者向けの従量課金データ API です：[everyinfra.com/ja](https://everyinfra.com/ja)
:::
