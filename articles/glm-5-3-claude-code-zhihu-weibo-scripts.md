---
title: "Claude CodeでGLM 5.3を使い、ZhihuとWeiboのトピック分析スクリプトを作る"
emoji: "📊"
type: "tech"
topics: ["claude", "api", "ai", "webscraping", "datascience"]
published: true
---

:::message
この記事は [EveryInfra ブログ](https://everyinfra.com/ja/blog/glm-5-3-claude-code-zhihu-weibo-scripts) からの転載です（原文が正本です）。
:::

中国のAIツールの動向を追うなら、開発者の考えは2つのサイトを見ればほぼわかります。ZhihuはQ&Aサイトで、ユーザーは長文の回答を書き、計算まで示します。Weiboはマイクロブログで、ユーザーは短い言葉で反応します。両方を合わせて読むと、全体の空気感とその背景にある理由の両方が見えてきます。

この記事では、その作業のためのセットアップを1つ試します。Claude CodeをZhipuのGLM-5.3またはGLM-5.3-Flashに接続します。次にGLMが、1つのトピックに関するZhihuの回答とWeiboのコメントをEveryInfra経由で取得するスクリプト`topic_pulse.py`を書きます。最後に、GLMが出力を読んで短い比較を書きます。

使用するトピックは「Coding Planの値上げ」です。Zhipuは2026-07-30にCoding Planをクレジット制に変更し、両サイトの開発者はいまもこの件で議論しています。範囲が狭く、技術的で、使用するツールにも近いトピックなので、テストに適しています。

何が実際の結果で、何がそうでないかを明確にしておきます。テスト用マシンにはZhipuのアカウントがありません。そのため、Claude CodeをGLMで動かす手順はすべて、2026-11-04に参照したZ.aiとBigModelのドキュメントに基づく設定例です。Claude Code内でGLMは実行していません。データ側は実際の結果です。

2026-11-04のSGT 00:16から02:08の間に、EveryInfra APIへ52件のリクエストを送信しました。そのうち34件がデータを返して課金され、合計は$0.020404でした。残りの18件は失敗または空の結果で、いずれも課金されていません。52件のうち1件は、MCPの`tools/call`メソッド経由です。

スクリプト全体は3回実行しました。2回目は回答の全文も取得し、12件のリクエストで$0.00581でした。3回目が最終版のスクリプトで、7件のリクエストがすべて成功し、$0.004142でした。

> **要点**
>
> - Claude CodeでGLM 5.3を使うには、`~/.claude/settings.json`に5つの環境変数を設定し、ベースURLを`https://api.z.ai/api/anthropic`または`https://open.bigmodel.cn/api/anthropic`にします。
> - `glm-5.3[1m]`のように`[1m]`サフィックスを付けないと、Claude Codeはコンテキストウィンドウを200Kとみなし、早すぎる段階でコンテキストを圧縮してしまいます。
> - Coding Planのクォータは対応するコーディングツール内でしか使えないため、データの取得はスクリプトが行い、要約はClaude Code上のGLMが書きます。
> - 同じトピックで、コメントの文字数の中央値はWeiboが14、Zhihuが28で、ノイズはWeiboがZhihuの2倍でした。
> - データ取得1回のコストは$0.00581でしたが、スクリプトを書くセッション1回のコストは、GLM-5.3-Flashで約$0.04、GLM-5.3で約$0.34と見積もられます。

## 1. Claude CodeでGLM 5.3を使う構成でできること

このタスクは前半と後半に分かれます。APIがデータを取得し、GLMがそれを読みます。前半に必要なのは構造化されたレコードで、回答やコメント1つにつき1件です。各レコードには、いいね数、日付、所属する質問や投稿が含まれます。こうしたレコードなら、モデルを一切使わずにスクリプトで数えたり並べ替えたりできます。

後半はデータの読み取りです。どのコメントが絵文字だけか、どれが重複しているか、どれが価格への不満か、どれがクォータの消費量を計算しているかを判断する必要があります。この部分にはモデルが必要です。

この2つをつなぐのがClaude Codeです。作業内容を1文で伝えると、モデルがスクリプトを書いて実行し、出力を読んで結果を報告します。GLM-5.3を選んだのは、ZhipuがClaude Code向けのガイドを公開しており、トークン単価も公開されているからです（関連ページはセクション2〜4でそれぞれ引用しています）。中国国外の読者は、Z.aiのサイトで同じモデルを米ドル建てで利用できます。

以下の表は、呼び出したすべての機能とその結果をまとめたものです。「行数」は各レスポンスに含まれる項目数です。価格はEveryInfraカタログに掲載されたリクエストごとの価格です。失敗したリクエストと空の結果は課金されません。

| 機能 | 2026-11-04の呼び出し回数 | カタログ価格 | 取得結果 |
| --- | --- | --- | --- |
| `weibo/search` | 6（うち1回はMCP経由） | $0.000806 | 毎回20件の投稿 |
| `weibo/comments` | データあり：9件、空：1件 | $0.000556 | 投稿あたり2〜43件のコメント |
| `zhihu/search` | 5 | $0.000556 | 20件の回答（抜粋は51〜86文字） |
| `zhihu/content_comments` | 7 | $0.000556 | 回答あたり3〜26件のコメント |
| `zhihu/content_detail`（任意） | データあり：7件、レート制限：1件 | $0.000556 | 回答の全文（最大756文字） |

最終版のスクリプトは、表の最初の4行だけを使います。Zhihuでは`zhihu/search`から始めます。これは質問タイトル、回答リンク、抜粋、いいね数をすでに返します。次に`zhihu/content_comments`で上位の回答に付いたコメントを読みます。1つの回答の全文が必要なら、`zhihu/content_detail`の呼び出しを1回追加します。GLM側は実行していないため、表にGLMの行はありません。

![2つの領域の間にあるロボットアームを描いたアイソメトリックイラスト。左側では、ガラス窓の向こうに本の背表紙が並んでいます（タイトルと抜粋だけの検索結果のように、タイトルしか見えません）。右側では、開いたアーカイブの引き出しに、糸で束ねた巻物や紙片が収められています（1つの引き出しはティール色で、データAPIが返す回答全文やコメントを表しています）。](https://everyinfra.com/articles/glm-5-3-claude-code-zhihu-weibo-scripts/img-search-vs-data.webp)

*Web検索で見えるのは本の背表紙だけ。データAPIなら引き出しを開けられます。* [この記事のために生成したイラスト](https://everyinfra.com/ja/mcp-server)

## 2. GLM-5.3とGLM-5.3-Flashの概要

GLM-5.3はZhipuのフラッグシップテキストモデルです。2026-08-19にBigModelプラットフォームで提供が始まりました。モデルページによると、入力はテキストのみで、コンテキストウィンドウは1Mトークン、出力は最大128Kトークンです（[GLM-5.3のモデルページ](https://docs.bigmodel.cn/cn/guide/models/text/glm-5.3)）。思考は常に有効です。`low`、`high`、`max`から選択でき、デフォルトは`max`です。思考を無効にすることはできません。

続いて2026-08-26にGLM-5.3-Flashがリリースされました。線形アテンションとスパースアテンションを組み合わせたマルチモーダルモデルです（[BigModelリリースノート](https://docs.bigmodel.cn/cn/update/new-releases)）。総パラメータ数は320B、アクティブパラメータ数は18Bです。Zhipuによると、GLM-5.3と比べて必要な計算量とKVキャッシュが大幅に少なくて済みます。

推奨設定は、`temperature`が1、`top_p`が0.95、`reasoning_effort`が`max`です（[GLM-5.3-Flashのモデルページ](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-5.3-flash)）。データ処理スクリプトで使う場合、主な違いはコストです。セクション11では、同じセッションにかかるFlashのコストはGLM-5.3の約9分の1です。

| 項目 | GLM-5.3 | GLM-5.3-Flash | 出典（2026-11-04確認） |
| --- | --- | --- | --- |
| リリース日 | 2026-08-19 | 2026-08-26 | BigModelリリースノート |
| 入力 | テキストのみ | テキストと画像 | モデルページ |
| コンテキストウィンドウ | 1Mトークン | `[1m]`設定時は1Mトークン | モデルページ、モデル切り替えガイド |
| 思考 | 常に有効（low、high、max） | 常に有効（max推奨） | モデルページ |
| Z.aiの料金（1Mトークンあたり） | 入力$1.4、キャッシュ入力$0.26、出力$4.4 | 入力$0.15、キャッシュ入力$0.03、出力$0.50 | Z.aiの料金ページ |
| BigModelの料金（1Mトークンあたり） | 入力CNY 8、キャッシュ入力CNY 2、出力CNY 28 | 入力CNY 0.8、キャッシュ入力CNY 0.23、出力CNY 2.8 | BigModelの料金ページ |
| Coding Planのクレジット係数 | 6.9, 1.7, 24 | 2.3, 0.56, 8 | Coding Plan概要 |

Z.aiの料金は[Z.aiの料金ページ](https://docs.z.ai/guides/overview/pricing)、CNYの料金は[BigModelの料金ページ](https://docs.bigmodel.cn/cn/guide/start/pricing)に基づいています。BigModelの料金ページには、キャッシュストレージが期間限定で無料であることも記載されています。また、1トークンは中国語の約1.6文字に相当するとも書かれています。

以下のトークン数の見積もりには、すべてこの比率を使っています。2026年11月3日の人民元中間レート（1 USDあたり6.7150 CNY）で換算すると、CNY 8は約$1.19です（[SAFEの為替レートページ](https://www.safe.gov.cn/AppStructured/hlw/RMBQuery.do)）。つまり、2つのサイトの料金はほぼ同じです。

2つのサイトではベースURLが異なります。Z.aiでは、Claude Codeは`https://api.z.ai/api/anthropic`に接続します（[Z.aiのClaude Codeガイド](https://docs.z.ai/devpack/tool/claude)）。BigModelでは、Anthropic互換のURLは`https://open.bigmodel.cn/api/anthropic`です。BigModelではさらに、Codex向けに`https://open.bigmodel.cn/api/v1`、その他のOpenAI互換コーディングツール向けに`https://open.bigmodel.cn/api/coding/paas/v4`が記載されています。通常の従量課金APIは`https://open.bigmodel.cn/api/paas/v4`です。これらを混同しないでください。プランのクォータを消費するのはコーディング用のURLだけです。

| ツール | ベースURL | プランのクォータを消費 | 確認日 |
| --- | --- | --- | --- |
| Z.aiのClaude Code | `https://api.z.ai/api/anthropic` | yes | 2026-11-04 |
| BigModelのClaude Code | `https://open.bigmodel.cn/api/anthropic` | yes | 2026-11-04 |
| BigModelのCodex | `https://open.bigmodel.cn/api/v1` | yes | 2026-11-04 |
| その他のOpenAI互換コーディングツール | `https://open.bigmodel.cn/api/coding/paas/v4` | yes | 2026-11-04 |
| 通常の従量課金API | `https://open.bigmodel.cn/api/paas/v4` | no | 2026-11-04 |

Zhipuの2つのページの間に食い違いも見つかりました。BigModelの[Claude Codeガイド](https://docs.bigmodel.cn/cn/guide/develop/claude)では、メインモデルに`glm-5.2[1m]`、スモールモデルに`glm-4.7`がまだ記載されています。新しい[モデル切り替えガイド](https://docs.bigmodel.cn/cn/coding-plan/latest-model)では、3つのモデルスロットすべてに`glm-5.3-flash[1m]`を使っています。このガイドには、Claude Code 2.1.140で動作確認済みとも記載されています。

古い例が今も動作する理由は、[Coding Plan概要](https://docs.bigmodel.cn/cn/coding-plan/overview)で説明されています。GLM-5.2またはGLM-5.1への呼び出しはGLM-5.3に、GLM-5-TurboまたはGLM-4.7への呼び出しはGLM-5.3-Flashに振り分けられます。そのため古い設定でも動作しますが、実際に使われているのは想定とは別のモデルです。

2つのページでは、エフォートレベルの対応付けも異なります。新しいページの記載を優先してください。デフォルトは`max`で、Claude Codeで`/effort`と入力して`high`を選ぶと、各ターンが短くなり、コストも下がります。

## 3. Coding Planのプラン区分、クレジット、ピーク時間帯

Coding Planは、コーディングツール向けの月額サブスクリプションです。2026年7月30日以降、使用量はクレジットで計算されます。Z.aiのドキュメントには、プランは「月額わずか18 USDから」と書かれています（[Z.aiのCoding Plan概要](https://docs.z.ai/devpack/overview)）。Z.aiのプランページは私たちのヘッドレスブラウザで表示されなかったため、各プランのUSD価格は引用できません。

中国版サイトの[GLM Coding Planページ](https://www.bigmodel.cn/glm-coding)には、2026年11月4日時点で月額料金が表示されていました。LiteはCNY 118、ProはCNY 538、MaxはCNY 1078です。1 USDあたり6.7150 CNYで換算すると、約$17.57、$80.12、$160.54です。四半期払いは20%割引、年払いは30%割引になります。

![Zhipu GLM Coding Planページのスクリーンショット。3つの料金カードが並んでいます。タブで月払い、四半期払い（20%割引）、年払い（30%割引）を選べ、四半期払いのタブが選択されています。カードには、Liteが月額94.4 CNY、Proが430.4、Maxが862.4と表示されています（取り消し線付きの定価は118、538、1078）。ティール色の枠で、タブ、価格欄、Liteのクレジット欄（週10,000クレジット。ProはLiteの6倍、MaxはLiteの14倍のクォータ）が強調されています。](https://everyinfra.com/articles/glm-5-3-claude-code-zhihu-weibo-scripts/img-glm-coding-price.webp)

*2026年11月4日に確認したCoding Planの料金（四半期払いのタブには月額の平均が表示され、取り消し線付きの価格は月払いプランの料金です）。* [GLM Coding Planページのスクリーンショット](https://www.bigmodel.cn/glm-coding)

各プランには、5時間あたりと週あたりの2つの上限があります。Liteは5時間あたり2,000クレジット、週10,000クレジットです。Proは12,000と60,000、Maxは28,000と140,000です。

5時間枠は、使用するたびにその5時間後にリセットされます。週の枠は注文日から7日間です。上限に達した場合は、リセットまで待つ必要があります。残高から料金が差し引かれることはありません。

クレジットの計算式は公開されています。モデルのクレジットは、入力トークン数×入力係数、キャッシュトークン数×キャッシュ係数、出力トークン数×出力係数を合計し、10,000で割った値です。Web検索、Webページの読み取り、リポジトリの読み取りのMCP呼び出しは、1回あたり1.2クレジットかかります。

オフピーク時のモデル呼び出しは、消費クレジットが半分になります。Z.aiのドキュメントによると、ピーク時間帯は月曜から金曜の14:00〜18:00（SGT）です。BigModelのドキュメントでは、同じ時間帯がUTC+8で記載されています。シンガポールから利用する場合は、午後の長時間セッションを避けてください。

| プラン | 2026-11-04時点の月額料金 | 概算（USD） | 5時間あたりのクレジット | 1週間あたりのクレジット | ピーク時の週あたりGLM-5.3セッション数 |
| --- | --- | --- | --- | --- | --- |
| Lite | CNY 118 | $17.57 | 2,000 | 10,000 | 約48 |
| Pro | CNY 538 | $80.12 | 12,000 | 60,000 | 約288 |
| Max | CNY 1078 | $160.54 | 28,000 | 140,000 | 約673 |

最後の列は、セクション11のサンプルセッションをもとにしています。このセッションのコストは、GLM-5.3で約207.8クレジット、Flashで68.7クレジットです。オフピーク時は回数が2倍になります。これらの数値は確約ではなく、おおよその規模を示す目安としてお考えください。

データ取得用のスクリプトを作る際に重要なルールが3つあります。1つ目は、プランは対応ツール内でしか使えないことです。それ以外の場所からのAPI呼び出しには、プランのクォータが適用されません。Z.aiの利用ポリシーでは、これを5語で「Use limited to supported tools」と述べています（[Z.aiの利用ポリシー](https://docs.z.ai/devpack/usage-policy)）。

BigModelのFAQでは、独自のアプリ、サイト、ボット、SaaS製品には標準APIが必要だと補足しています（[Coding Plan FAQ](https://docs.bigmodel.cn/cn/coding-plan/faq)）。2つ目は、2026-10-16以降、標準のAPIキーとプランのキーが分かれたことです。コーディングツールで標準キーを使っても、プランのクォータは使われません（[APIキーに関するお知らせ](https://docs.bigmodel.cn/cn/coding-plan/notice/api-key-management)）。3つ目は、プランは返金不可で、アカウントの共有もできないことです（[利用上の注意](https://docs.bigmodel.cn/cn/coding-plan/usage-notes)）。

1つ目のルールが設計全体を左右します。`topic_pulse.py`自体からGLMを呼び出してコメントにラベルを付ける、という案は魅力的です。そうすれば、スクリプトを人手を介さずタイマーで実行できます。しかしプランのルールでは、こうしたGLMの呼び出しはプランの利用としてカウントされず、標準APIの料金がかかります。

そのため、このスクリプトが行うのは2つだけです。データの取得と、モデルを必要としない単純な統計の計算です。読み取りと要約は、プランが適用されるClaude Code内で行います。さらに、このスクリプトはモデルにアクセスできないマシンを含め、どのマシンでも実行できます。

## 4. Claude CodeでGLM 5.3を使う方法：設定例

以下は、Z.ai向けの`~/.claude/settings.json`のサンプルです。Z.aiのClaude Codeガイドと、Claude Codeの環境変数リファレンスに沿っています。Z.aiのアカウントを持っていないため、実際には動作確認していません。キーはプレースホルダーです。

```text
{
  "env": {
    "ANTHROPIC_AUTH_TOKEN": "your-zai-coding-plan-key",
    "ANTHROPIC_BASE_URL": "https://api.z.ai/api/anthropic",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5.3[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-5.3[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-5.3-flash[1m]",
    "CLAUDE_CODE_AUTO_COMPACT_WINDOW": "1000000",
    "API_TIMEOUT_MS": "3000000",
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "EVERYINFRA_API_KEY": "your-everyinfra-key"
  }
}
```

Claude Codeは、`ANTHROPIC_AUTH_TOKEN`を`Authorization`ヘッダーのBearerトークンとして送信します（[Claude Codeの環境変数](https://code.claude.com/docs/en/env-vars)）。ここには標準のAPIキーではなく、Coding Planのキーを指定してください。3つの`ANTHROPIC_DEFAULT_*_MODEL`の値で、Claude Codeのopus、sonnet、haikuの各エイリアスが実際にどのモデルを指すかを指定します。Z.aiはopusとsonnetにGLM-5.3、haikuにFlashを割り当てています。より新しいBigModelのページでは、3つすべてにFlashを割り当てています。スクリプトの作成にはより高性能なモデルが必要なため、ここではZ.aiに従います。バックグラウンドタスクではFlashを使えば、クレジットを節約できます。

`[1m]`のサフィックスは残してください。Claude Codeは`glm-5.3`というIDを認識しません。モデル設定のドキュメントによると、未知のIDの場合は200Kトークンを想定し、起動時に警告を表示します（[Claude Codeのモデル設定](https://code.claude.com/docs/en/model-config)）。IDに`[1m]`を付けると1Mのウィンドウを想定するため、GLM-5.3と一致します。サフィックスがないと、Claude Codeは早すぎるタイミングで会話を圧縮してしまいます。これはモデルが長い結果を読み込むときに特に目立ちます。

`CLAUDE_CODE_AUTO_COMPACT_WINDOW`は、圧縮が始まるまでにコンテキストをどこまで使えるかを設定します。指定できる値は100,000〜1,000,000です。`API_TIMEOUT_MS`を3,000,000ミリ秒にするのはZhipuの推奨で、最大エフォートのGLMは長時間思考することがあるためです。`CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC`は、テレメトリなどの必須ではないリクエストを停止します。

最後の行の`EVERYINFRA_API_KEY`はスクリプト用です。Claude Codeが起動する子プロセスはこの値を読み取れるため、スクリプトは`os.environ`からキーを読み込みます。キーをコードに書くことはありません。

| 変数 | この例での値 | 用途 | 出典（2026-11-04確認） |
| --- | --- | --- | --- |
| `ANTHROPIC_BASE_URL` | `https://api.z.ai/api/anthropic` | リクエストをZ.aiに送信 | Z.aiのClaude Codeガイド |
| `ANTHROPIC_AUTH_TOKEN` | プランのキー | Bearerトークン | Claude Codeの環境変数リファレンス |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | `glm-5.3[1m]` | メインモデル | Z.aiのClaude Codeガイド |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | `glm-5.3-flash[1m]` | 小規模なバックグラウンドタスク | Z.aiのClaude Codeガイド |
| `CLAUDE_CODE_AUTO_COMPACT_WINDOW` | `1000000` | 圧縮のしきい値 | Claude Codeの環境変数リファレンス |
| `API_TIMEOUT_MS` | `3000000` | リクエストのタイムアウト | BigModelのClaude Codeガイド |

ここで、ある事実を明確にしておく必要があります。Claude CodeのLLMゲートウェイドキュメントでは、サポートされているAPI形式に対応したゲートウェイであれば動作すると説明されています。一方、Anthropicは「いかなるゲートウェイを介しても、Claude CodeをClaude以外のモデルにルーティングすることはサポートしていない」とも述べています（[Claude CodeのLLMゲートウェイ](https://code.claude.com/docs/en/llm-gateway)）。つまり、Claude CodeがGLMで動作するのは、Zhipuが互換性のあるエンドポイントを構築したためです。

問題が起きた場合は、Zhipuのドキュメントとサポートを頼ってください。2026-11-04時点で、Claude Codeの最新リリースは2026-11-03公開の2.1.317でした（[Claude Codeの変更履歴](https://code.claude.com/docs/en/changelog)）。Zhipuのページは2.1.140で動作確認されています。この2つのバージョンの間には多くのリリースがあるため、見慣れないエラーが出たときは両方の変更履歴を確認してください。

ガイド付きでセットアップしたい場合は、Zhipuが提供する対話型ツール`npx @z_ai/coding-helper`を使えます（[npmパッケージ](https://www.npmjs.com/package/@z_ai/coding-helper)）。セットアップ後、新しいターミナルを開いて`claude`を実行し、`/status`と入力します。設定ソースが`~/.claude/settings.json`を指し、モデルの行に`glm-5.3`または`glm-5.3[1m]`と表示されている必要があります。

## 5. 2つのMCPセットの比較

MCPはModel Context Protocolの略で、モデルが外部のツールを呼び出せるようにするオープンプロトコルです（[MCP仕様](https://modelcontextprotocol.io/specification/2026-07-28)）。Coding Planには独自のMCPツールが付属しています。2026-11-04時点のドキュメントには、Web検索（`webSearchPrime`）、Webページの読み取り（`webReader`）、zreadというリポジトリリーダー、ビジョンサーバーが掲載されていました。zreadサーバーには`search_doc`、`get_repo_structure`、`read_file`の3つのツールがあります。ビジョンサーバーは`@z_ai/mcp-server`というnpmパッケージとしてローカルで動作します。最新バージョンの0.1.6は2026-10-29にリリースされました（[npmパッケージ](https://www.npmjs.com/package/@z_ai/mcp-server)）。

Web検索とWebリーダーの両ページには、Claude CodeでCoding Planを使う場合、どちらのツールもモデルサービスに組み込まれていると記載されています（[Web検索用MCP](https://docs.bigmodel.cn/cn/coding-plan/mcp/search-mcp-server)、[Webページ読み取り用MCP](https://docs.bigmodel.cn/cn/coding-plan/mcp/reader-mcp-server)）。インストールは不要です。別のモデルで使いたい場合にのみ、手動で追加します。

| 組み込みツール | 返される内容 | 1回あたりのクレジット | バージョンまたは日付 |
| --- | --- | --- | --- |
| `webSearchPrime` | タイトル、URL、要約、サイト名、アイコン | 1.2 | 2026-11-04にドキュメントを確認 |
| `webReader` | タイトル、本文、メタデータ、リンク | 1.2 | 2026-11-04にドキュメントを確認 |
| zread | `search_doc`、`get_repo_structure`、`read_file` | 1.2 | 2026-11-04にドキュメントを確認 |
| ビジョンサーバー | 画像・動画の解析、ローカルで実行 | Flashとしてカウント | 0.1.6（2026-10-29リリース） |

ドキュメントには、各ツールが返す内容も記載されています。Web検索は、ページのタイトル、URL、要約、サイト名、サイトアイコンを返します。Webリーダーは1つのURLを取得し、タイトル、本文、メタデータ、リンクの一覧を返します。これはコーディング作業に適しています。ライブラリを調べたり、エラーに関するスレッドを読んだり、READMEを開いたりできます。

センチメント分析のスクリプトには、別の形のデータが必要です。回答やコメント1つにつき1件で、いいね数、返信数、日付、所属する質問や投稿を含むデータです。こうしたデータがあれば、スクリプトで並べ替え、ノイズ除去、中央値の計算ができます。下の表は、ツールの種類ごとに返されるフィールドを比較したものです。Zhipuのアカウントを持っていないため、組み込みツールの列はZhipuのドキュメントのみに基づいています。EveryInfraの列は、2026-11-04の実際の呼び出しで返ったフィールドを示しています。

| 必要なもの | 組み込みの検索・リーダー（Zhipuのドキュメント） | EveryInfraのフィールド（実際の呼び出し、2026-11-04） | 確認 |
| --- | --- | --- | --- |
| 特定トピックに関するZhihuの回答 | ページタイトル、URL、要約、サイト名 | `zhihu/search`：質問タイトル、回答リンク、抜粋、いいね数、コメント数、保存数、日付（1回あたり20件） | 2026-11-04 |
| 1件の回答の全文 | ページタイトル、本文、メタデータ、リンク | `zhihu/content_detail`：全文といいね数、コメント数、保存数 | 2026-11-04 |
| 回答へのコメント | 1ページ分の本文 | `zhihu/content_comments`：コメント1つにつき1件、text、like_count、reply_count付き（3〜26件） | 2026-11-04 |
| Weiboのキーワード検索 | ページタイトル、URL、要約 | `weibo/search`：id、url、text、posted_at、コメント数・リポスト数・いいね数（1回あたり20件） | 2026-11-04 |
| Weiboコメント | 同上 | `weibo/comments`：コメント1つにつき1件（2〜43件） | 2026-11-04 |
| ニュース、ドキュメント、GitHub | 適している | これらの機能の用途外 | 2026-11-04 |

どちらのサイトも、独自の開発者プラットフォームを運営しています。自分のアカウントのデータだけが必要な場合や、プラットフォーム独自の認可を通じて連携する場合に適しています。Zhihuデータプラットフォームの検索は、Bearerトークンとタイムスタンプヘッダーを使います。`Count`の最大値は10、`ContentText`は抜粋で、`HasMore`は常にfalseです（[Zhihu開発者ドキュメント：検索](https://developer.zhihu.com/docs?key=zhihu_search)）。ホットリストのスキルは、1から30までの`limit`を受け取ります（[Zhihu開発者ドキュメント：ホットリスト](https://developer.zhihu.com/docs?key=hot_list_skill)）。

Weiboでは、`comments/show`エンドポイントは認可されたユーザーのコメントを返します（[Weibo API：comments/show](https://open.weibo.com/wiki/2/comments/show)）。`search/topics`エンドポイントは高度なAPIで、最新200件を返します（[Weibo API：search/topics](https://open.weibo.com/wiki/2/search/topics)）。

つまり、2つのMCPツール群で役割を分担します。Zhipuの組み込みツールは、ドキュメント、ニュース、公開ページをカバーし、呼び出し1回あたり1.2クレジットかかります。EveryInfraのMCPサーバーまたはAPIはプラットフォーム内の構造化データを扱い、リクエストごとに米ドル建ての料金が設定されています。どちらも1つのClaude Codeセッションで接続したままにでき、モデルはツールの説明をもとに使い分けます。EveryInfraのリモートMCPサーバーは次のように追加します。実際に`initialize`と`tools/list`を実行したところ、12個のツールが表示されました。

```bash
claude mcp add --transport http everyinfra https://api.everyinfra.com/mcp \
  --header "Authorization: Bearer $EVERYINFRA_API_KEY"

# Check the connection: the list must show everyinfra and its tools
claude mcp list
```

トピック分析のスクリプトに必要なのは、12個のツールのうち2つだけです。`everyinfra_list_capabilities`は、プラットフォーム、機能、パラメータを一覧表示します。`everyinfra_call_api`は機能を実行します。SGT 00:23に、MCPの`tools/call`経由で、「Zhipu Coding Plan」を意味する中国語のキーワードを指定して`weibo/search`を呼び出しました。20件の投稿が返され、費用は$0.000806で、直接呼び出した場合と同じでした。テキスト結果は15,103文字でした。

このサイズが重要です。Claude Codeは、1つのMCP結果が10,000トークンを超えると警告を出します。デフォルトの上限は25,000トークンで、`MAX_MCP_OUTPUT_TOKENS`で変更できます。50,000文字を超えるテキスト結果は、コンテキストではなくファイルに書き出されます（[Claude Code MCPドキュメント](https://code.claude.com/docs/en/mcp)）。

20件の投稿なら問題なく収まります。しかし、モデルがMCP経由で1回ずつ呼び出して50件のコメントと20件の回答を取得すると、コンテキストはすぐに埋まります。そこで、GLMにスクリプトを書かせることにしました。スクリプトは数十回の呼び出しを1つの小さなJSONファイルにまとめ、モデルはそのファイルだけを読みます。

ゲートウェイに影響する点がもう1つあります。Claude Codeはデフォルトでツール検索を有効にするため、MCPツールは必要に応じて読み込まれます。その場合、リクエストには`defer_loading`フィールドと`tool_reference`ブロックが含まれます。Claude Codeは、`ANTHROPIC_BASE_URL`の背後にあるゲートウェイがこれらをそのまま転送することを想定しています。

バージョン2.1.305以降では、ゲートウェイが400または422を返すと、Claude Codeはツール検索なしで1回再試行し、すべてのMCPツールを最初に読み込みます。Zhipuのエンドポイントがこれらのフィールドを受け付けるかどうかは確認できませんでした。多数のMCPサーバーを使っていて、セッション開始時点でコンテキストが埋まっているように見える場合は、これが原因かもしれません。

## 6. 1回の比較の流れ

![ループするアニメーションのフロー図。ティールの点がプロンプト（あるトピックについてZhihuとWeiboを比較）から出発し、GLM-5.3を使うClaude Code（Z.aiのAnthropic互換エンドポイント）へ移動します。次にtopic_pulse.py（GLMが書くデータスクリプト）へ進み、EveryInfra APIとの間を往復します。結果はJSONファイル（テキスト79件、約2,207トークン）で、点はGLMが書く比較サマリーで止まります。](https://everyinfra.com/articles/glm-5-3-claude-code-zhihu-weibo-scripts/fig-02-task-flow.ja.light.svg)

作業全体は一直線に進みます。まず、Claude Codeに1文で指示します。例：「ZhihuとWeiboでCoding Planの値上げがどう語られているかを比較するスクリプトを書いて、その違いを教えて」。次に、GLMがリクエストを読み取ります。`everyinfra_list_capabilities`を呼び出すかAPIドキュメントを読んでパラメータを把握し、`topic_pulse.py`を書きます。

続いて、Claude Codeがスクリプトを実行します。スクリプトはWeibo検索、Weiboコメント、Zhihu検索、Zhihuコメントを順に呼び出し、結果と各呼び出しのコストを1つのJSONファイルに書き出します。最後に、GLMがそのファイルを読んでサマリーを書きます。

この流れで目立つ数字が2つあります。1つ目は圧縮率です。2回目の実行では、APIから返された生のJSONは50,859文字、約14,543トークンでした。スクリプトはテキストフィールドだけを残し、4,315文字、約2,207トークンになりました。入力の85%を削減したことになります。GLMは毎ターンコンテキストを読み直すため、この削減効果は1つのセッション内で何度も積み重なります。

2つ目は時間です。2回目の実行はSGT 00:33:55から00:37:01までで、3分強かかりました。その大半は、あらかじめ設けた待機時間でした。スクリプトは1分あたり10リクエストを超えないよう、呼び出しの間隔を空けています。また、HTTP 429レスポンスを1回受けた後にも待機しました。

## 7. GLMが書くべきスクリプト

Claude Codeに渡すプロンプトは次のとおりです。そのまま使えます。

> 標準ライブラリのみを使って、Pythonスクリプトtopic_pulse.pyを書いてください。スクリプトはトピックのキーワードを1つ受け取ります。環境変数EVERYINFRA_API_KEYのキーを使って、EveryInfra API（POST https://api.everyinfra.com/api/v1/social）を呼び出します。weibo/searchでWeiboの投稿を20件検索します。指定した語を含むコメントが3件以上ある投稿を残し、各投稿についてweibo/commentsで50件のコメントを読み込みます。

> zhihu/searchでZhihuの回答を20件検索し、質問ごとにグループ化します。タイトルが指定したパターンに一致する質問を残します。上位3件の質問について、最もいいねの多い回答に付いたコメントをzhihu/content_commentsで50件読み込みます。

> 呼び出しの間隔は6.5秒空けます。429または503の場合は、retry_afterの時間だけ待ってから再試行します。recent_failuresが3に達したら、その機能の呼び出しを停止します。絵文字のみのコメント、重複したコメント、極端に短いコメント、ボットのコメントは除外します。件数、長さの中央値、上位5件のコメント、そして各呼び出しのHTTPステータスとcharged_usdを1つのJSONファイルに書き出します。

各行には理由があります。標準ライブラリのみにすることで、何もインストールせずにどこでも実行できます。6.5秒の間隔は、デフォルトの上限である1分あたり10リクエストを超えないようにするためです。

再試行のルールと`recent_failures`のルールは、7分以上かかった最初の実行の後に追加しました。タイトルフィルターは2回目の実行の後に追加しました。どちらもセクション9で説明します。ボットのルールがあるのは、Weiboにはコメントスレッド内で返信する自動検索アシスタントがあるためです。その長い返信は統計を歪めてしまいます。

以下は3つ目のバージョンの中核部分です。上のプロンプトをもとに作成し、実際に実行しました。ファイル全体は約120行で、この抜粋では呼び出し、再試行、Zhihu関連のロジックのみを示しています。

```text
API = "https://api.everyinfra.com/api/v1/social"
KEY = os.environ["EVERYINFRA_API_KEY"]
GAP = 6.5            # seconds between calls: stay under 10 requests per minute
DOWN = set()         # capabilities that failed 3 times in 10 minutes

def call(platform, action, params, tries=3):
    if f"{platform}/{action}" in DOWN:
        return None
    body = json.dumps({"platform": platform, "action": action, "params": params}).encode()
    for attempt in range(tries):
        req = urllib.request.Request(API, data=body, method="POST", headers={
            "Authorization": f"Bearer {KEY}", "Content-Type": "application/json"})
        try:
            with urllib.request.urlopen(req, timeout=120) as r:
                code, d = r.status, json.load(r)
        except urllib.error.HTTPError as e:
            code, d = e.code, json.loads(e.read() or b"{}")
        log(platform, action, code, d)       # rows and billing.charged_usd
        time.sleep(GAP)
        err = d.get("error") or {}
        if code == 200:
            return d
        if (err.get("recent_failures") or 0) >= 3:   # the error says: stop retrying
            DOWN.add(f"{platform}/{action}")
            return None
        if code in (429, 503) and err.get("retryable") and attempt < tries - 1:
            time.sleep(min(err.get("retry_after", 30), 60))
            continue
        return None

# Zhihu: search, group by question, keep matching titles, read comments of the top answer
for q in top_questions:
    answers += by_q[q]                   # search rows: question title, link, excerpt, likes
    best = max(by_q[q], key=lambda x: x.get("like_count") or 0)
    zcomments += rows(call("zhihu", "content_comments", {"url": best["url"], "limit": 50}))
```

重要なのは、スクリプトがエラーをどう読むかです。EveryInfraのエラーは単なるメッセージではなく、フィールドを持つJSONオブジェクトです。`fault`は誰の問題かを示します。`retryable`と`retry_after`は、再試行すべきかどうかと、どれだけ待つべきかを示します。`recent_failures`は、この機能が最近何回失敗したかを示します。

`next_steps`は、次に何をすべきかを呼び出し元にわかりやすい言葉で伝えます。`example`は、動作が確認済みのリクエストを示します。これらのフィールドを読むスクリプトは、HTTPコードを数えるだけのスクリプトより的確に判断できます。GLMもClaude Code上で同じフィールドを読み、再試行を繰り返さずに計画を変更できます。2回目の実行では、SGT 00:35:34にHTTP 429が1回返されました。スクリプトはレスポンスの指示どおりに待機し、33秒後の再試行は課金なしで成功しました。

ノイズの判定ルールはシンプルです。まず、スクリプトは各コメントから「〜さんへの返信」という接頭辞を取り除きます。4文字以下のコメントは短文とみなします。絵文字タグ、スペース、数字、「はい」、笑いだけで構成されるコメントは、絵文字のみのコメントまたは反復コメントとみなします。投稿者名にWeibo検索アシスタントのタグが含まれるコメントはボットとみなします。

これらのルールは、事前にではなく、最初の実データを読んだ後に作りました。完璧ではありません。たとえば、2文字の「本当」も短文として除外されます。それでも「実のある意見を述べている人は何人いるか」という問いには、十分に機能します。

## 8. 両サイトで実際に1往復を試す

スクリプトを作る前に、各機能を手動で呼び出してパラメータとレスポンスの形式を確認しました。表は主な呼び出しを順に示しています。時刻はすべて2026-11-04のSGTです。

| 時刻（SGT） | 機能 | 対象 | HTTP | 件数 | 課金額 |
| --- | --- | --- | --- | --- | --- |
| 00:16:49 | `weibo/search` | キーワード「Coding Plan値上げ」 | 200 | 20 | $0.000806 |
| 00:17:03 | `zhihu/search` | 同じキーワード | 200 | 20 | $0.000556 |
| 00:21:42 | `zhihu/content_detail` | いいね47件の回答 | 200 | 1 | $0.000556 |
| 00:21:56 | `zhihu/content_comments` | 同じ回答 | 200 | 7 | $0.000556 |
| 00:22:20 | `weibo/comments` | 4つの価格戦略に関する投稿 | 200 | 11 | $0.000556 |
| 00:22:57 | `weibo/comments` | 数値IDではなく英字の短縮コード | 200 | 0 | $0（結果が空のため返金） |

Weiboは問題なく動作しました。`weibo/search`の各投稿には、`id`、`url`、`text`、`posted_at`と、コメント数・リポスト数・いいね数が含まれます。今回のキーワードで得た20件の投稿は、2026-08-17から2026-10-26のものでした。00:21 SGTに「GLM-5.3」で検索すると、2026-10-30から2026-11-03のより新しい投稿が返されました。`weibo/comments`には、数値IDか、数値IDを含むリンクを指定できます。

00:22:20 SGTに、4つの価格戦略を比較した投稿のコメント11件を読み取りました。ある読者は、「コストに利益を上乗せし、さらにピーク時とオフピーク時で分ける」方式のほうが一律料金よりわかりやすいと書いていました。別の読者は、Kimiは悪くないが「高くて遅い」ので、DeepSeekを使い続けると述べていました。唯一の落とし穴はリンクの形式でした。

英字の短縮コードは意図的に送信しました。APIは200を返しましたが、行数は0で、理由は`empty_result_refunded`となり、課金されませんでした。検索結果の`id`フィールドを使えば、この問題は起きません。

Zhihuの検索も問題なく動作しました。質問タイトル、回答リンク、抜粋、いいね数、コメント数、保存数、日付を含む20件の回答が返されました。これだけで2つの処理を行えます。スクリプトは、各リンクに含まれる質問IDで回答をグループ化します。次に、各グループで最もいいねの多い回答を選びます。`zhihu/content_comments`は回答リンクを受け取り、トップレベルのコメントを返します。各コメントには本文、いいね数、返信数が含まれます。

00:21:42 SGTに`zhihu/content_detail`で読み取った回答は、Maxプランを更新したユーザーによるものでした。前日には四半期プランが月額422 CNYだったのに、翌日には「800〜900」になっていたそうです。この回答にはいいね47件、コメント28件があり、00:21:56 SGTにそのうち7件のコメントを読み取りました。`zhihu/content_detail`が必要なのは抜粋では足りない場合だけで、回答1件につき1リクエストかかります。

![暗いターミナルに表示したAPIレスポンスのスクリーンショット（EveryInfraのソーシャルエンドポイントへのPOST、HTTP 200、11件、2026-11-04 02:07:56 SGT、topic_pulse.pyの第3版）。リクエストはZhihuの回答リンク1つとlimit 50を指定したzhihu/content_commentsで、11件のコメントのうち2件を名前を伏せて表示しています。ティールの枠はresults、text、like_count、charged_usd（0.000556）を示しています。](https://everyinfra.com/articles/glm-5-3-claude-code-zhihu-weibo-scripts/img-zhihu-comments-response.webp)

*最終版のスクリプトがZhihuのコメントについて受け取ったレスポンスの1つ（11件中2件を表示、名前は非表示）。コメント1つが1件で、いいね数と返信数が付いています。* [EveryInfra APIレスポンスのスクリーンショット](https://api.everyinfra.com/api/v1/social)

## 9. スクリプトの3回の実行：キーワードのずれ、レート制限、タイトルフィルター

1回目の実行では、中国語のキーワード「Zhipu Coding Plan」を使いました。実行時間は00:24:58から00:32:37 SGTまでの7分39秒です。15リクエストのうち6件が課金され、金額は$0.003586でした。残りの9件は失敗し、課金されませんでした。実行が遅かったのは、このバージョンが失敗のたびに60秒待機し、`recent_failures`を読んでいなかったためです。

キーワードのずれも起きました。Weiboでコメントが最も多かった投稿は、ライブコマースの販売投稿でした。トピックとは無関係でしたが、スクリプトはそのコメントを読み取りました。

| 実行 | キーワード | 時刻（SGT） | リクエスト数 | 課金額 | 費用 |
| --- | --- | --- | --- | --- | --- |
| 1回目 | Zhipu Coding Plan | 00:24:58〜00:32:37 | 15 | 6 | $0.003586 |
| 2回目 | Coding Plan値上げ | 00:33:55〜00:37:01 | 12 | 10 | $0.00581 |
| 3回目（最終版） | 同上、Zhihuのタイトルフィルター付き | 02:06:32〜02:08:22 | 7 | 7 | $0.004142 |

2回目の実行では3点を修正しました。まず、`recent_failures`のルールを追加しました。次に、`--must "coding plan"`を追加し、Weiboの投稿にそのフレーズが含まれることを条件にしました。さらに、キーワードをより絞り込んだ「Coding Plan値上げ」に変更しました。このバージョンでは、`zhihu/content_detail`で回答の全文も読み取りました。

実行時間は00:33:55から00:37:01 SGTまでの3分6秒です。12リクエストのうち10件が課金され、費用は$0.00581でした。課金されなかった2件には、00:35:34 SGTの429も含まれます。セクション10の比較数値は、この実行と手動の呼び出しの結果に基づいています。

2回目の実行の結果はすっきりしていましたが、いいね数で質問を選ぶ方法の欠点も明らかになりました。Zhihuの20件の回答のうち、8件はタイトルにZhipuまたはGLMを含んでいました。ある質問は、Zhipuのプランが中国国内では海外より高い理由を尋ねていました。別の質問は、「Zhipu Coding Plan値上げ」をどう見るかを尋ねていました。しかしスクリプトは、いいねの合計が最も多い3つの質問を選びました。

1つ目は、OpenAIが$200のProプランを一時停止した理由を尋ねる質問で、いいねは523件でした。2つ目は、今コストパフォーマンスの高いコーディングプランをどこで購入できるかを尋ねる質問で、いいねは378件でした。3つ目はいいね235件で、Alibaba CloudがLiteプランの更新受付を停止したことに関する質問でした。3つともコーディングプランの料金を扱っていますが、Zhipuに関するものはありません。

「Zhipuの値上げについて人々がどう思っているか」を知りたい場合は、タイトルフィルターを追加します。たとえば、タイトルにZhipuまたはGLMを含む質問に絞ります。これはClaude Codeに向いている作業です。最初の要約を読んで「Zhipuに関する質問だけ」と指示すれば、GLMが1行を変更し、1セント未満の費用で再実行します。

Weiboでは同じ問題がより顕著に表れました。「Coding Plan値上げ」で得た20件の投稿のうち、本文に「coding plan」という語句が含まれているのは7件だけでした。そのうちコメントが3件以上あるのは2件だけです。1件は、別のプランを解約して代わりのサービスを試したユーザーの投稿です。

もう1件は、GLM 5.3は大きく改善したが、値上げ幅が大きすぎるという内容です。2件合わせてもコメントは5件だけでした。残りの13件には、ニュースの転載、日次ダイジェスト、ゲームニュースなどが含まれます。コメントが最も多い投稿（50件）は、DeepSeekの価格に関するトピック投稿でした。

つまり、ニッチな技術トピックはWeiboではあまり議論されず、意見の大半はZhihuに集まっています。公平なサンプルを得るため、手動で選んだWeiboのコメントセット4つ（73件）と2回目の実行で得た5件を統合し、Weiboのコメントを78件にしました。Zhihuでは、2回目の実行で得た71件と手動段階の7件を統合し、こちらも78件にしました。件数をそろえると、割合を比較しやすくなります。

3回目の実行ではタイトルフィルターを追加しました。これが本記事の最終版スクリプトです。全文取得の処理は外し、4つの機能に絞りました。`--zhihu-title`を追加し、Zhipu、GLM、Zhipuのよく使われる愛称に一致するパターンを指定しました。実行時間は2026-11-04の02:06:32から02:08:22 SGTまでの1分50秒です。7件のリクエストはすべて200を返して課金され、費用は$0.004142でした。

Weiboでは、2回目の実行と同じ2件の投稿と5件のコメントを読み取りました。Zhihuでは、3つの質問すべてがZhipuに関するものになりました。1つ目は、2026-07-30のクレジット制への移行をどう見るかという質問です（トップ回答：いいね47件、コメント28件）。2つ目は、GLM-5のコーディングプランをどこで契約できるか、なぜこのモデルがこれほど人気なのかという質問です（いいね36件）。3つ目は、Zhipu Coding Planがなぜこれほど高いのかという質問です（いいね7件）。

この3件の回答に付いたコメントを、スクリプトはそれぞれ7件、11件、3件、合計21件取得しました。ノイズルールで除外されたものはなく、文字数の中央値は26文字でした。19件のいいねが付いたトップコメントは、3か月後にどのAIが首位に立っているか誰にもわからないのに、なぜ1年分を払うのかと問いかけています。別のコメントには4件のいいねが付き、国際版プランは「国内価格より50%高い」と指摘しています。

![最終スクリプトを1回実行したときの7件のリクエストを示す横棒グラフ。棒は時系列順に並び、棒の長さはUSDでの課金額を表します。最も長い棒は02:06:32のweibo/search（$0.000806）です。02:06:53と02:07:08の2件のweibo/comments、02:07:26の1件のzhihu/search、02:07:44、02:07:56、02:08:09の3件のzhihu/content_commentsは、それぞれ$0.000556です。合計は$0.004142です。](https://everyinfra.com/articles/glm-5-3-claude-code-zhihu-weibo-scripts/fig-03-run-cost-bar.ja.light.svg)

## 10. ZhihuとWeiboの声

このセクションでは、GLMが結果を読んだ後に書くべき内容を示します。モデルが自己採点しないよう、`analysis.py`はモデルを介さず、すべての数値を生のレスポンスから計算します。本文は、それらの数値に基づいて私たちが作成した例です。

まずは長さです。ノイズ除去後、Weiboコメントの文字数の中央値は14文字です。Weiboコメントの9割は36文字以下で、最長は158文字です。Zhihuコメントの中央値は28文字、9割は70文字以下で、最長は229文字です。

Zhihuの回答4件は、中央値が310.5文字、最大が756文字です。つまりZhihuのコメントはWeiboのコメントの2倍の長さがあり、Zhihuの回答はさらにその10倍の長さです。Weiboだけを読んでわかるのは雰囲気です。理由を知るにはZhihuの回答が必要です。

![対数スケールの横向きストリップチャート。各点はノイズ除去後の1件のテキストを表し、軸は文字数を示します。Weiboコメントは63点で中央値は14（大半は5〜42、長いものが92と158の2件）。Zhihuコメントは70点で中央値は28（9〜229）。Zhihu回答は4点（87、276、345、756）で中央値は310.5。3本の縦線が各中央値を示します。](https://everyinfra.com/articles/glm-5-3-claude-code-zhihu-weibo-scripts/fig-04-length-dist.ja.light.svg)

次はノイズです。Weiboコメント78件のうち、ルールで15件を除外しました。6件は「はい」のような4文字以下のもので、4件は絵文字のみでした。

3件はWeibo検索アシスタントによる返信でした。1件は重複、1件はリンクのみでした。Zhihuコメント78件のうち、ルールで除外したのは8件で、内訳は短すぎるもの3件、リンクのみ2件、重複2件、絵文字のみ1件です。ノイズの割合はWeiboで19%、Zhihuで10%です。

ボットの返信については補足が必要です。Weiboのユーザーはコメント内で検索アシスタントをタグ付けし、たとえば「DeepSeekはどれくらい値上げしたのか」と質問できます。アシスタントは長い回答を投稿し、その回答はいいね数で上位に入ります。ボットのルールがなければ、アシスタントの発言を人々の声として報告してしまいます。

![積み上げ横棒グラフ。各棒は78件のコメントの内訳を示します。Weiboは63件が残ります（短すぎるもの6件、絵文字のみ4件、ボットの返信3件、重複1件、リンクのみ1件）。Zhihuは70件が残ります（短すぎるもの3件、リンクのみ2件、重複2件、絵文字のみ1件）。残った部分はティール色です。ノイズはWeiboが19%、Zhihuが10%です。](https://everyinfra.com/articles/glm-5-3-claude-code-zhihu-weibo-scripts/fig-05-noise-share.ja.light.svg)

続いてテーマです。6つのテーマそれぞれにキーワードリストを作成しました。「価格」は値上げ、高い、いくら、人民元、ドルを表す語を対象とします。「利用枠」は利用枠、使用量、使い切り、5時間、週、クレジット、リセットを対象とします。「品質」は「劣化した」と量子化を対象とします。

「競合」はDeepSeek、Kimi、Codex、Claude、Qwen、MiniMaxを対象とします。「ローカル」はローカル、VRAM、デプロイを対象とします。「BAN」はBAN、リスク管理、レート制限を対象とします。1件のコメントが複数のテーマに該当することもあります。

残ったWeiboコメント63件のうち、22.2%が価格に、15.9%が競合との比較に触れています。品質とローカル環境はそれぞれ4.8%、利用枠はわずか3.2%です。残ったZhihuコメント70件では、25.7%が利用枠に触れています。価格と競合はそれぞれ20%です。品質とBANはそれぞれ5.7%で、ローカル環境は0件です。

![残ったコメントに占める6つのテーマの割合を示すグループ化横棒グラフ。価格はWeiboで22.2%、Zhihuで20%。利用枠はWeiboで3.2%、Zhihuで25.7%（Zhihuの棒はティール色）。その他4つのテーマは小さい値です（品質4.8%と5.7%、競合15.9%と20%、ローカル環境4.8%と0、BAN 0と5.7%）。どちらのサイトも価格を話題にしていますが、利用枠が主要な話題になっているのはZhihuだけです。](https://everyinfra.com/articles/glm-5-3-claude-code-zhihu-weibo-scripts/fig-06-theme-share.ja.light.svg)

| 指標 | Weibo、コメント78件 | Zhihu、コメント78件と回答4件 | 読み方 | データ取得日 |
| --- | --- | --- | --- | --- |
| ノイズルール適用後の残数 | 63 | 70 | Weiboはノイズの割合が2倍 | 2026-11-04 |
| 文字数の中央値 | 14文字 | コメント28文字、回答310.5文字 | 理由はZhihuにある | 2026-11-04 |
| ノイズ除去後の投稿のいいね数 | 112 | コメント349、回答1,183 | Zhihuのいいねは少数の回答に集中 | 2026-11-04 |
| 最多テーマ | 価格、22.2% | 利用枠、25.7% | 「高すぎるか」対「足りるか」 | 2026-11-04 |
| 一方にのみ見られるテーマ | ローカル環境、4.8% | BAN、5.7% | サンプルが少なく、手がかりにとどまる | 2026-11-04 |

これらの数値をもとにすると、GLMの要約はたとえば次のようになります。これは私たちが作成した例文で、モデルの出力ではありません。

> どちらのサイトも価格を話題にしていますが、その語り方は異なります。Weiboでは「値段がどんどんおかしくなっている」のような一言の評価が多く、理由が添えられることはほとんどありません。コメントのおよそ6件に1件が他社の名前を挙げています。Zhihuでは、ユーザーが具体的に計算しています。最上位の回答は、API料金の割引倍率が15倍を超えるプランは、ユーザーが使い切るとどれも赤字になると述べています。

> 上位のコメントは利用枠に関するものです。74いいねのコメントは、「リセットカードの配布後」に利用可能枠がおよそ半分に減ったと述べています。Zhihuではコメントの5.7%がBANやリスク管理に触れていますが、Weiboには見られません。Weiboにはローカル環境がプランの代わりになるかを尋ねるユーザーもいますが、今回のZhihuのデータには見られません。サンプルは各サイト78件のコメントと少なく、Zhihuの回答は最も多くいいねされた質問から取得したものです。

引用するのは公開されているコメント本文のみで、ユーザー名は含めていません。いいね数は`like_count`フィールドから取得しています。改めて強調しておきたい制約が1つあります。これは1つのトピック、1晩、各サイト78件のコメントに基づく結果です。

テーマのルールは私たちが作成したキーワードリストです。GLMのほうがより正確にコメントを分類できる可能性もあれば、誤る可能性もあります。この結果が答えるのは「この2つのサイトでトピックの語られ方がどう違うか」です。「値上げを支持する人の割合はどれくらいか」には答えません。

## 11. コスト：データ、GLMトークン、プランのクレジット

まずデータから見ます。2回目の実行では12回のリクエストを送りました。`weibo/search`は$0.000806、ほかの成功した9回のリクエストはそれぞれ$0.000556でした。失敗した2回のリクエスト（うち1回は429）は無料でした。合計は$0.00581です。この実行ではテキスト79件（Weiboのコメント5件、Zhihuの回答3件、Zhihuのコメント71件）に加え、検索結果40件が返りました。テキスト1,000件あたり$0.0735です。

最終版のスクリプトでは全文取得のステップを省いています。1回の実行で7回のリクエストを送り、費用は$0.004142で、Weiboのコメント5件、Zhihuのコメント21件、検索結果40件を返します。2回目の実行と同じ規模で毎日実行すると月$0.1743、最終版のスクリプトでは月$0.12426です。

| シナリオ | リクエスト数 | USD | 算出根拠 |
| --- | --- | --- | --- |
| 2回目の実行（全文あり） | 送信12件、課金10件 | $0.00581 | charged_usd |
| 3回目の実行（最終スクリプト） | 送信7件、課金7件 | $0.004142 | charged_usd |
| テキスト1,000件あたり | n/a | $0.0735 | 79件 |
| 30日間毎日 | 360 | $0.1743 | 30 × $0.00581 |
| Z.aiのFlashによる要約処理1回 | 1 | $0.001081 | 3,707トークン |

次はモデルです。アカウントがないため、実際のセッションは計測できませんでした。そこで、[Claude Codeのコストに関するドキュメント](https://code.claude.com/docs/en/costs)にあるセッション例を借用します。入力トークン1.2k、出力トークン5.3k、キャッシュ読み取り940k、キャッシュ書き込み50kです。

この構成は「100行のスクリプトを書き、2回実行し、1回修正する」作業に近いものです。各ターンでコンテキストを読み直すため、キャッシュ読み取りが大部分を占めます。Zhipuにはキャッシュ書き込みの個別料金がないため、通常の入力として計算します。

| モデルとサイト | セッション例 | 概算（USD） | 要約処理1回 | ピーク時のプランクレジット |
| --- | --- | --- | --- | --- |
| Z.aiのGLM-5.3 | $0.3394 | $0.3394 | $0.00969 | 207.8 |
| BigModelのGLM-5.3 | CNY 2.438 | $0.3631 | $0.008884 | 207.8 |
| Z.aiのGLM-5.3-Flash | $0.0385 | $0.0385 | $0.001081 | 68.7 |
| BigModelのGLM-5.3-Flash | CNY 0.272 | $0.0405 | $0.000888 | 68.7 |
| EveryInfraでのデータ取得1回 | n/a | $0.00581 | n/a | n/a |

「要約処理1回」の列は、入力2,207トークン（2回目の実行で得たクリーニング済みテキスト）と出力1,500トークンを前提としています。これは、スクリプトの完成後にGLMに新しいデータを読ませるために毎日追加でかかるコストです。CNYの値は2026-11-03のレート6.7150で換算しています。

![スクリプト作成セッション1回とデータ取得1回のコストを米ドルで比較した、対数目盛の横棒グラフ。GLM-5.3はBigModelで$0.3631、Z.aiで$0.3394。GLM-5.3-FlashはBigModelで$0.0405、Z.aiで$0.0385。EveryInfraでのデータ取得1回は$0.0058（ティール色の棒）。スクリプト作成のコストはデータ取得1回の7〜63倍です。](https://everyinfra.com/articles/glm-5-3-claude-code-zhihu-weibo-scripts/fig-07-session-cost-bar.ja.light.svg)

このグラフが示すことは1つです。この種のタスクでは、費用の大半はデータの取得ではなく、コードの作成と修正にかかります。GLM-5.3でのスクリプト作成セッション1回は、データの約60倍のコストです。Flashなら約7倍です。スクリプトが完成した後の毎日のコストは、データ取得1回分と要約処理1回分だけです。Z.aiのFlashなら1日$0.007未満です。

Coding Planでは、モデルのコストはクレジットに置き換わります。Liteでは週10,000クレジットが付与されます。すべてピーク時の場合、今回の例のようなセッションをGLM-5.3で約48回、Flashで約145回実行できます。オフピーク時はどちらも2倍になります。組み込みのWeb検索は1回あたり1.2クレジットなので、ドキュメントを数回調べる程度なら利用枠はほとんど減りません。

見落としやすいコストがもう1つあります。APIの生のレスポンスをそのままモデルに渡すコストです。2回目の実行で得た生のJSONは約14,543トークンでした。Z.aiのGLM-5.3の入力料金（100万トークンあたり$1.4）で計算すると、1回の読み込みで約$0.02かかり、データのコストの3倍以上になります。ターンが増えるたびに、同じJSONがまた読み込まれます。先にスクリプトで圧縮すれば、モデルが読むのは約2,207トークンだけです。この構成で最も大きな節約になるのがこの点です。

## 12. 制限事項

1つ目に、質問に付いたすべての回答を取得する`zhihu/question_answers`はテスト期間中に利用できず、失敗したリクエストは課金されませんでした。そのためスクリプトは`zhihu/search`から始め、各質問について検索で見つかった回答とそのコメントだけを取得します。

第2に、Zhipu側は何もテストしていません。設定は2026-11-04に読んだドキュメントに従っていますが、そのドキュメントは2か所で食い違っています。GLM-5.3がどんなスクリプトを書くか、何回の修正が必要か、何トークン使うかについてのデータはありません。モデルのコストはClaude Codeのドキュメントにあるセッション例からの見積もりで、実測値ではありません。

3つ目は、サンプルが少ないことです。各サイトのコメントは78件です。Weiboでは、そのうち73件がスクリプトではなく手作業で選んだ投稿のものです。Zhihuの回答は4件だけです。テーマのルールはキーワードのリストで、「week」のような短い語は偶然一致することがあります。結果は意見の割合ではなく、手がかりとして使ってください。

4つ目に、プランの利用枠は対応ツールでのみ使えます。タイマーでスクリプトを実行し、人を介さずにGLMに要約させる仕組みはプランの対象外で、標準APIが必要になります。今回の設計では、タイマーで実行するのはスクリプトだけです。要約は、Claude Codeを開いてGLMに最新のファイルを読むよう依頼するまで行いません。

5つ目に、`weibo/comments`で取得するWeiboのコメントには、IP所在地のフィールドがありません。スレッド内の返信は`weibo/comment_replies`で取得でき、1回あたり$0.000806で、所在地が含まれます。地域別に見たい場合は、この呼び出しを追加してください。

## 13. GLM-5.3とCoding Planをめぐる最近の変更

モデル面では、QbitAIが2026-10-12に、ZhipuがZCode 3.15.1にGLM-5.3-Highspeedを追加したと報じました。あるユーザーの計測では毎秒約490トークンでした（[QbitAI](https://www.qbitai.com/2026/10/502649.html)）。このモデルは現時点ではZCodeでのみ使え、Coding PlanのドキュメントにはClaude Code向けとして記載されていません。ビジョンMCPパッケージは2026-10-29にバージョン0.1.6になりました。

安全面では、Anthropicが2026-09-29に研究を公開しました。GLM-5.3はエンドツーエンドのサイバーエクスプロイトを単独で構築でき、強固な安全対策がないままリリースされたとしています（[Anthropicの研究](https://www.anthropic.com/research/glm-5-3-and-the-spread-of-advanced-cyber-capabilities)）。2026-10-30には、Help Net SecurityがIrregularによる評価を報じました。

GLM-5.3は13のシナリオのいずれも拒否せず、評価者は、より大きな自律性を与える前に制限をテストするよう勧めています（[Help Net Security](https://www.helpnetsecurity.com/2026/10/30/irregular-glm-5-3-cyber-capabilities-evaluation/)）。この構成に当てはめると、最小権限を徹底するということです。Claude CodeでGLMを動かす場合、スクリプトに必要なのは1つのフォルダと1つのAPIだけで、それ以上は与えません。

料金面では、Zhipuが2026-10-16に標準キーとプランキーを分けました。2026-10-23には36Krが、Zhipuは独身の日に実質的な割引を行った数少ないベンダーの1つだと報じました。Liteの月額カードはCNY 118からCNY 115に下がりました（[36Kr](https://m.36kr.com/p/4038373928949385)）。10月下旬には、BAAIコミュニティが、品質低下の補償として一部のプランユーザーに利用枠リセットカードが配布されたと報じました（[BAAI Hub](https://hub.baai.ac.cn/view/58992)）。これは「リセットカード配布」後の利用についてのZhihuのコメントと一致します。Zhipuのドキュメントには、このカードに関する規定は見つかりませんでした。

| 日付 | 変更内容 | 出典 |
| --- | --- | --- |
| 2026-09-29 | GLM-5.3のサイバー能力に関するAnthropicの研究 | Anthropic |
| 2026-10-12 | ZCode 3.15.1にGLM-5.3-Highspeedを追加 | QbitAI |
| 2026-10-16 | 標準キーとプランキーを分離 | BigModelのお知らせ |
| 2026-10-23 | Liteの月額カードがCNY 118からCNY 115に | 36Kr |
| 2026-10-29 | `@z_ai/mcp-server` 0.1.6 | npm |
| 2026-10-30 | IrregularによるGLM-5.3の評価 | Help Net Security |
| 2026-11-03 | Claude Code 2.1.317 | Claude Codeの変更履歴 |

## 14. 始め方

手順1〜3は設定例です。手順4〜6は実際に実行しました。

1. Z.aiでCoding Planを契約し、プラン用のキーを作成します。
1. セクション4のとおりに`~/.claude/settings.json`を編集します。
1. 新しいターミナルを開いて`claude`を実行し、`/status`と入力してモデルを確認します。
1. [クイックスタート](https://everyinfra.com/ja/docs#quickstart)でEveryInfraのキーを取得し、`EVERYINFRA_API_KEY`として追加します。
1. セクション7のプロンプトをClaude Codeに渡します。まずは`--weibo-posts 2 --zhihu-questions 2`と`--zhihu-title`を指定して小規模に実行します。
1. GLMにJSONを読ませて要約を書かせたら、タイトルのフィルターを追加して再度実行します。

各機能のパラメータとフィールドは、[Zhihu APIのページ](https://everyinfra.com/ja/zhihu-api)と[Weibo APIのページ](https://everyinfra.com/ja/weibo-api)に掲載されています。[Zhihuの検索](https://everyinfra.com/ja/zhihu-api/search)、[Zhihuのコメント](https://everyinfra.com/ja/zhihu-api/content_comments)、任意で[Zhihuのコンテンツ詳細](https://everyinfra.com/ja/zhihu-api/content_detail)を参照してください。Weiboについては[Weiboの検索](https://everyinfra.com/ja/weibo-api/search)と[Weiboのコメント](https://everyinfra.com/ja/weibo-api/comments)を参照してください。リモートMCPのツール一覧は[MCPサーバーのページ](https://everyinfra.com/ja/mcp-server)に、カタログ価格は[料金ページ](https://everyinfra.com/ja/pricing)にあります。

関連記事：クリーニングの工程でのモデルの性能とコストについては、[Claude CodeでDeepSeek V4を使ったコメントのクリーニング](https://everyinfra.com/ja/blog/deepseek-v4-claude-code-comment-cleaning)をご覧ください。`topic_pulse.py`が出力するJSONは、そのフローにそのまま渡せます。Claude Codeを使った別のデータ処理パターンについては、[Xiaohongshuコメント向けのClaude Code動的ワークフロー](https://everyinfra.com/ja/blog/claude-code-dynamic-workflows-xiaohongshu-comments)をご覧ください。「いいね数で選ぶ」ことで生じるバイアスについては[ソーシャルメディア調査におけるサンプルの完全性](https://everyinfra.com/ja/blog/social-media-research-sample-integrity)を、失敗した呼び出しの返金については[失敗時の返金照合](https://everyinfra.com/ja/build/failure-refund-reconciliation)をご覧ください。

## FAQ

### Claude CodeでGLM 5.3を使うには？[1m]サフィックスが必要なのはなぜ？

セクション4のとおり、`~/.claude/settings.json`の`env`ブロックに値を設定します。見落としがちなのが`[1m]`サフィックスです。Claude Codeは`glm-5.3`というIDを認識しないため、サフィックスがないと200Kのコンテキストウィンドウを前提とし、早い段階でコンテキストを圧縮してしまいます。変更後は新しいターミナルを開き、`/status`でモデルの行を確認してください。質問の出典：「how to use glm 5.3 in claude code」と「glm 5.3 claude code」の検索候補（2026-11-04確認）。根拠：[Z.aiのClaude Codeガイド](https://docs.z.ai/devpack/tool/claude)、[Claude Codeのモデル設定](https://code.claude.com/docs/en/model-config)。

### 自作のスクリプトでCoding Planの利用枠を使えますか？

いいえ。プランは対応ツールでのみ使えます。自作のアプリ、サイト、ボット、SaaS製品からの呼び出しには標準APIが必要です。2026-10-16以降は、2種類のキーも分かれています。そのため、今回のスクリプトはデータの取得だけを行い、要約はClaude Code上のGLMが書きます。質問の出典：「glm coding plan limits」の検索候補（2026-11-04確認）。根拠：[Z.aiの利用ポリシー](https://docs.z.ai/devpack/usage-policy)、[Coding Planの概要](https://docs.bigmodel.cn/cn/coding-plan/overview)。

### Claude Code向けのGLM Coding PlanのベースURLとエンドポイントは？

Z.aiでは`https://api.z.ai/api/anthropic`、BigModelでは`https://open.bigmodel.cn/api/anthropic`を使います。CodexなどのOpenAI互換ツールには、それぞれ専用のコーディング用URLがあります。通常のAPIのURLでは、プランの利用枠は消費されません。質問の出典：「glm coding plan base url」と「glm coding plan endpoint」の検索候補（2026-11-04確認）。根拠：[BigModelのモデル切り替えガイド](https://docs.bigmodel.cn/cn/coding-plan/latest-model)、[Z.aiのクイックスタート](https://docs.z.ai/devpack/quick-start)。

### スクリプトの作成にはGLM-5.3-FlashとGLM-5.3のどちらを選ぶべきですか？

Z.aiでは、Flashの入力料金はGLM-5.3の約9分の1で、出力料金も約9分の1です。当社の試算では、スクリプト作成1セッションあたりの費用はFlashで約$0.04、GLM-5.3で約$0.34です。100行のデータスクリプトなら、まずFlashを使い、Flashでバグを修正できない場合にのみGLM-5.3に切り替えてください。質問の出典：「glm 5.3 flash vs glm 5.3」と「glm 5.3 flash cost」の検索候補（2026-11-04確認）。根拠：[Z.aiの料金](https://docs.z.ai/guides/overview/pricing)。

### Coding Planに組み込まれたWeb検索では何を取得できますか？

Zhipuのドキュメントによると、Web検索はタイトル、URL、要約、サイト名、アイコンを返し、Webリーダーはタイトル、本文、メタデータ、リンクを返します。ドキュメントやニュースにはこれで十分です。トピックスクリプトに必要なのは、回答やコメント1つにつき1件で、いいね数と日付を含むデータです。そのために`zhihu/search`、`zhihu/content_comments`、`weibo/search`、`weibo/comments`を使います。Zhipuのアカウントがないため、組み込みツールはテストしていません。質問の出典：「glm coding plan web search」の検索候補（2026-11-04に確認）。根拠：「[MCP for web search](https://docs.bigmodel.cn/cn/coding-plan/mcp/search-mcp-server)」、「[MCP for web reading](https://docs.bigmodel.cn/cn/coding-plan/mcp/reader-mcp-server)」。

### ZhihuにAPIはありますか？

はい。Zhihuのデータプラットフォームには、検索とホットリストのエンドポイントがあります。検索は1回の呼び出しで最大10件の抜粋を注目コメント付きで返し、認証にはBearerトークンとタイムスタンプヘッダーを使います。同プラットフォームの認可を経て行う用途に適しています。

当社のスクリプトでは、`zhihu/search`で1回あたり20件の回答を、`zhihu/content_comments`でコメントを取得します。2026-11-04の最終実行では21件のZhihuコメントを読み込み、Zhihu部分の費用は$0.002224でした。質問の出典：「zhihu com api」の検索候補（2026-11-04確認）。根拠：[Zhihu開発者ドキュメント：検索](https://developer.zhihu.com/docs?key=zhihu_search)、[Zhihu検索の機能ページ](https://everyinfra.com/ja/zhihu-api/search)。

## 参考文献

1. [Z.ai：Claude Codeガイド](https://docs.z.ai/devpack/tool/claude)
1. [Z.ai：Coding Planの概要](https://docs.z.ai/devpack/overview)
1. [Z.ai：最新モデルへの切り替え](https://docs.z.ai/devpack/latest-model)
1. [Z.ai：利用ポリシー](https://docs.z.ai/devpack/usage-policy)
1. [Z.ai：クイックスタート](https://docs.z.ai/devpack/quick-start)
1. [Z.ai：料金](https://docs.z.ai/guides/overview/pricing)
1. [Z.aiブログ：GLM-5.3](https://z.ai/blog/glm-5.3)
1. [BigModel：モデル切り替えガイド（2026-11-04閲覧）](https://docs.bigmodel.cn/cn/coding-plan/latest-model)
1. [BigModel：Claude Codeガイド](https://docs.bigmodel.cn/cn/guide/develop/claude)
1. [BigModel：Claude Codeの紹介](https://docs.bigmodel.cn/cn/guide/develop/claude/introduction)
1. [BigModel：Coding Planの概要](https://docs.bigmodel.cn/cn/coding-plan/overview)
1. [BigModel：Coding Planクイックスタート](https://docs.bigmodel.cn/cn/coding-plan/quick-start)
1. [BigModel：Coding Planの利用上の注意](https://docs.bigmodel.cn/cn/coding-plan/usage-notes)
1. [BigModel：Coding Planのよくある質問](https://docs.bigmodel.cn/cn/coding-plan/faq)
1. [BigModel：プラン改定のお知らせ](https://docs.bigmodel.cn/cn/coding-plan/notice/usage-revision)
1. [BigModel：APIキーに関するお知らせ（2026-10-16）](https://docs.bigmodel.cn/cn/coding-plan/notice/api-key-management)
1. [BigModel：Web検索用MCP](https://docs.bigmodel.cn/cn/coding-plan/mcp/search-mcp-server)
1. [BigModel：Webページ読み取り用MCP](https://docs.bigmodel.cn/cn/coding-plan/mcp/reader-mcp-server)
1. [BigModel：zread MCP](https://docs.bigmodel.cn/cn/coding-plan/mcp/zread-mcp-server)
1. [BigModel：ビジョンMCP](https://docs.bigmodel.cn/cn/coding-plan/mcp/vision-mcp-server)
1. [BigModel：対応ツール](https://docs.bigmodel.cn/cn/coding-plan/tool/others)
1. [BigModel：GLM-5.3モデルページ](https://docs.bigmodel.cn/cn/guide/models/text/glm-5.3)
1. [BigModel：GLM-5.3-Flashモデルページ](https://docs.bigmodel.cn/cn/guide/models/vlm/glm-5.3-flash)
1. [BigModel：料金](https://docs.bigmodel.cn/cn/guide/start/pricing)
1. [BigModel：リリースノート](https://docs.bigmodel.cn/cn/update/new-releases)
1. [BigModel：料金に関する質問](https://docs.bigmodel.cn/cn/faq/fee-issues)
1. [BigModel：APIに関する質問](https://docs.bigmodel.cn/cn/faq/api-issues)
1. [GLM Coding Planページ（価格は2026-11-04閲覧）](https://www.bigmodel.cn/glm-coding)
1. [Claude Code：LLMゲートウェイ](https://code.claude.com/docs/en/llm-gateway)
1. [Claude Code：モデル設定](https://code.claude.com/docs/en/model-config)
1. [Claude Code：環境変数](https://code.claude.com/docs/en/env-vars)
1. [Claude Code：MCP](https://code.claude.com/docs/en/mcp)
1. [Claude Code：設定](https://code.claude.com/docs/en/settings)
1. [Claude Code：コスト](https://code.claude.com/docs/en/costs)
1. [Claude Code：変更履歴（2.1.317、2026-11-03）](https://code.claude.com/docs/en/changelog)
1. [Claude Codeリポジトリの変更履歴](https://github.com/anthropics/claude-code/blob/main/CHANGELOG.md)
1. [MCP仕様（2026-07-28）](https://modelcontextprotocol.io/specification/2026-07-28)
1. [Zhipu glm-cookbook：Claude Codeの例](https://github.com/MetaGLM/glm-cookbook/blob/main/vibecoding/glm-4.5-claude-code-integration.md)
1. [npm：@z_ai/mcp-server](https://www.npmjs.com/package/@z_ai/mcp-server)
1. [npm：@z_ai/coding-helper](https://www.npmjs.com/package/@z_ai/coding-helper)
1. [Zhihuデータプラットフォーム](https://developer.zhihu.com/)
1. [Zhihuデータプラットフォーム：検索](https://developer.zhihu.com/docs?key=zhihu_search)
1. [Zhihuデータプラットフォーム：ホットリストのスキル](https://developer.zhihu.com/docs?key=hot_list_skill)
1. [WeiboオープンプラットフォームCLI](https://open.weibo.com/cli)
1. [Weibo CLIクイックスタート](https://open.weibo.com/cli/quickstart)
1. [Weibo API：comments/show](https://open.weibo.com/wiki/2/comments/show)
1. [Weibo API：search/topics](https://open.weibo.com/wiki/2/search/topics)
1. [Weibo API：レート制限](https://open.weibo.com/wiki/Rate-limiting)
1. [EveryInfra機能カタログ（2026-11-04閲覧）](https://api.everyinfra.com/api/v1/social/catalog)
1. [SAFE：人民元の基準為替レート](https://www.safe.gov.cn/AppStructured/hlw/RMBQuery.do)
1. [Anthropic：GLM-5.3と高度なサイバー能力の拡散（2026-09-29）](https://www.anthropic.com/research/glm-5-3-and-the-spread-of-advanced-cyber-capabilities)
1. [Help Net Security：IrregularによるGLM-5.3の評価（2026-10-30）](https://www.helpnetsecurity.com/2026/10/30/irregular-glm-5-3-cyber-capabilities-evaluation/)
1. [QbitAI：毎秒490トークンのGLM-5.3-Highspeed（2026-10-12）](https://www.qbitai.com/2026/10/502649.html)
1. [36Kr：独身の日のトークンプラン（2026-10-23）](https://m.36kr.com/p/4038373928949385)
1. [BAAI Hub：品質低下とリセットカード](https://hub.baai.ac.cn/view/58992)

---

:::message
この記事は EveryInfra ブログの記事の転載です。原文：[Claude CodeでGLM 5.3を使い、ZhihuとWeiboのトピック分析スクリプトを作る](https://everyinfra.com/ja/blog/glm-5-3-claude-code-zhihu-weibo-scripts)

EveryInfra は AI エージェントと開発者向けの従量課金データ API です：[everyinfra.com/ja](https://everyinfra.com/ja)
:::
