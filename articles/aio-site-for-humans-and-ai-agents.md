---
title: "人間にもAIエージェントにも読まれるサイトを作る：AIO入門（セマンティックHTML・JSON-LD・robots.txt・llms.txt）"
emoji: "🧭"
type: "tech"
topics: ["aiagent", "seo", "html", "webdev", "llm"]
published: true
---

## はじめに

ZEN大学で知能情報を学んでいる2年生の松延です。AIエージェントやアプリ（iOSアプリ「カミメグ」や、Expo / React Native でのアプリ）を個人開発しています。

先日、LinkedIn にこんなことを書きました。

> これからのWebエンジニアに必須のスキルは「人間が見やすいサイト作り」だけではありません。「AIエージェント（GrokBotやMuseなど）に正しく認識されるサイト作り」が不可欠です。

AIエージェントを作る側にいると、エージェントは「Webを読む側」でもあると実感します。人がブラウザでリンクをクリックする時代から、エージェントがWeb上の情報を集めて解釈し、回答やアクションとして提示する時代へ。そうなると、サイトの作り手に問われるのは「見た目」だけではなく、**機械が構造を取り違えずに読めるか**（AIにとってのDX＝Data Experience）です。

この記事では、その視点を **AIO（AI Optimization / AI最適化）** と呼び、投稿で挙げた4つを、最小限のコードで具体化します。

1. セマンティックHTML
2. 構造化データ（JSON-LD）
3. AIクローラーを意識したアクセス制御（robots.txt）と、llms.txt
4. APIやデータの取得しやすさ・パフォーマンス

先にお断りしておくと、この記事は「こうすればAIに必ず拾われる」という話ではありません。特定のAIエージェントが内部でどうサイトを解釈しているかは、私は一次情報を確認できていないので書きません。ここで扱うのは、公式ドキュメントや仕様に書かれている「機械が読みやすい作り方」と「クローラーへの意思表示の方法」です。

## 1. セマンティックHTML：タグで意味を語る

### 何が問題か

見た目が同じでも、`div` だらけのHTMLは「どこが見出しで、どこが本文で、どこがナビゲーションか」をタグが語っていません。人間は見た目で判断できますが、HTMLを読む機械は、タグ以外に手がかりが少なくなります。

### Before

```html
<div class="header">
  <div class="logo">My Blog</div>
  <div class="menu">
    <div><a href="/">ホーム</a></div>
    <div><a href="/posts">記事</a></div>
  </div>
</div>
<div class="content">
  <div class="title">AIOとは</div>
  <div class="date">2026-11-18</div>
  <div class="text">本文……</div>
</div>
<div class="footer">© 2026</div>
```

### After

```html
<header>
  <p class="logo">My Blog</p>
  <nav aria-label="メイン">
    <ul>
      <li><a href="/">ホーム</a></li>
      <li><a href="/posts">記事</a></li>
    </ul>
  </nav>
</header>
<main>
  <article>
    <h1>AIOとは</h1>
    <p><time datetime="2026-11-18">2026年11月18日</time></p>
    <p>本文……</p>
  </article>
</main>
<footer>© 2026</footer>
```

ポイントは次のとおりです。

- ページの主要な内容は `<main>` に入れる（1ページに1つ）。
- 独立して読める記事やカードは `<article>`、ナビゲーションは `<nav>`。
- 見出しは `h1` → `h2` → `h3` と階層を飛ばさずに使う。見た目の大きさ調整のために見出しタグを選ばない（大きさはCSSで）。
- 日付は `<time datetime="...">`、画像には意味のある `alt` を付ける。

これはアクセシビリティ（スクリーンリーダー対応）の定番でもあります。「人間にとっての使いやすさ」と「機械にとっての読みやすさ」が、ここでは同じ作業になります。

参考：[MDN: Semantics](https://developer.mozilla.org/en-US/docs/Glossary/Semantics) / [MDN: `<main>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/main)

> 補足：内容がJavaScriptで描画された後にしかHTMLに現れない作りだと、JavaScriptを実行しないクライアントには何も見えません。実行するかどうかは読む側次第なので、重要な本文は最初のHTMLレスポンスに含めておくのが無難です（SSR / SSG）。Expo や React Native のコードをWeb向けに出す場合も、紹介ページなど「読まれたいページ」はこの点を意識したいところです。

## 2. 構造化データ（JSON-LD）：意味を機械可読な形で足す

セマンティックHTMLが「文書の骨格」を語るのに対し、構造化データは「このページは何についてのものか」（記事なのか、アプリなのか、誰が書いたのか）を [Schema.org](https://schema.org/) の語彙で明示します。Googleは構造化データの形式として JSON-LD を推奨しています（[Google: 構造化データの概要](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data)）。

### 記事ページ（Article）

`<head>` か `<body>` 内に、次のような `script` を置きます。

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "AIOとは",
  "datePublished": "2026-11-18T09:00:00+09:00",
  "dateModified": "2026-11-18T09:00:00+09:00",
  "author": [
    {
      "@type": "Person",
      "name": "松延弘平",
      "url": "https://zenn.dev/koheip"
    }
  ],
  "image": ["https://example.com/images/aio-cover.png"]
}
</script>
```

GoogleのArticleのドキュメントでは、必須プロパティはなく、当てはまる推奨プロパティをできるだけ入れる、という方針になっています（[Google: Article 構造化データ](https://developers.google.com/search/docs/appearance/structured-data/article)）。

### アプリの紹介ページ（SoftwareApplication）

アプリのLPなら `SoftwareApplication` が使えます。下はダミー値の例です。

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "SoftwareApplication",
  "name": "サンプルアプリ",
  "operatingSystem": "iOS",
  "applicationCategory": "LifestyleApplication",
  "offers": {
    "@type": "Offer",
    "price": "0",
    "priceCurrency": "JPY"
  }
}
</script>
```

注意点が2つあります。

- 値は**ページに実際に書いてある内容と一致させる**こと。ページに無い評価や価格を構造化データだけに書くのは避けます。
- Googleの検索結果でソフトウェアアプリのリッチリザルトを狙う場合は、`name`、`offers.price`、`aggregateRating` または `review` といった独自の必須プロパティがあります（[Google: ソフトウェアアプリ](https://developers.google.com/search/docs/appearance/structured-data/software-app)）。評価やレビューが実在しないなら、無理に足さないでください。Schema.org の語彙としては、上の最小形でも有効です。

使える型とプロパティの一覧は [Schema.org/Article](https://schema.org/Article) や [Schema.org/SoftwareApplication](https://schema.org/SoftwareApplication) で確認できます。なお、構造化データを入れたからといって、特定のAIエージェントがそれを使うと保証されるわけではありません。Googleの検索機能については、上記ドキュメントが一次情報です。

## 3. AIクローラーを意識したアクセス制御：robots.txt と llms.txt

### robots.txt でクローラーに意思表示する

ここで大事なのは、**クローラーごとに目的が違う**ことです。たとえば各社のドキュメントには、次のような区別が書かれています。

| ユーザーエージェント / トークン | 提供元 | ドキュメントに書かれている用途 |
| --- | --- | --- |
| `GPTBot` | OpenAI | 生成AI基盤モデルの学習に使われる可能性のあるコンテンツのクロール |
| `OAI-SearchBot` | OpenAI | ChatGPTの検索機能で、サイトを検索結果に表示するため |
| `ChatGPT-User` | OpenAI | ユーザーの操作に応じたアクセス。自動クロールではなく、robots.txtのルールが適用されない場合がある、と明記されている |
| `ClaudeBot` | Anthropic | 生成AIモデルの学習に寄与しうるWebコンテンツの収集 |
| `Claude-SearchBot` | Anthropic | 検索結果の品質向上のためのクロール |
| `Claude-User` | Anthropic | Claudeのユーザーが質問したときのアクセス |
| `Google-Extended` | Google | 将来のGeminiモデルの学習等にクロール済みコンテンツを使うかを制御するトークン。Google検索への掲載には影響しない。専用のHTTPユーザーエージェント文字列はない |
| `PerplexityBot` | Perplexity | Perplexityの検索結果にサイトを表示・リンクするため。基盤モデルの学習用ではない、と記載 |
| `Perplexity-User` | Perplexity | ユーザーの操作に応じたアクセス |

（出典は記事末尾。名称や用途は変わりうるので、設定前に各社の最新ドキュメントを確認してください。）

これを踏まえると、たとえば「学習には使わせたくないが、検索結果には載りたい」という方針なら、こう書けます。

```txt
# 生成AIの学習への利用は断る
User-agent: GPTBot
Disallow: /

User-agent: ClaudeBot
Disallow: /

User-agent: Google-Extended
Disallow: /

# 検索結果に表示されるためのクローラーは許可する
User-agent: OAI-SearchBot
Allow: /

User-agent: Claude-SearchBot
Allow: /

User-agent: PerplexityBot
Allow: /

# それ以外は通常どおり
User-agent: *
Allow: /

Sitemap: https://example.com/sitemap.xml
```

逆に「全部許可」なら、何も書かなければ（あるいは `User-agent: *` に `Allow: /`）それで足ります。どちらが正解かはサイトの方針しだいです。

押さえておきたい点：

- **robots.txt は強制ではなく、お願い（規約）です。** プロトコルの仕様である [RFC 9309](https://www.rfc-editor.org/rfc/rfc9309) にも、これらのルールは「アクセス認可の一形態ではない」と書かれています。守るかどうかはクローラー側に委ねられています。本当に見せたくない情報は、認証をかけるか、そもそも公開しないのが原則です。
- ユーザーの操作で動くエージェント（`ChatGPT-User` など）は、上の表のとおり robots.txt の扱いがクローラーと異なることがあります。
- OpenAI や Perplexity はクローラーのIPアドレス範囲を公開しているので、WAFなどで「UAを名乗るだけの偽物」を見分ける、といった運用もありえます（各社ドキュメント参照）。

### llms.txt：AI向けの「入口ページ」の提案

`llms.txt` は、Jeremy Howard 氏が提案している、AIエージェント向けにサイトの要点をMarkdownでまとめたファイルです。サイトのルート（`/llms.txt`）などに置きます。

**重要：llms.txt は標準規格ではなく「提案（proposal）」です。** 仕様のページ自体も "We propose..." という書き方です。また、すべてのAIサービスが読むとは限りません。私は、特定のAIが必ず読むと確認できる一次情報を確認していません。「置けば順位や引用が上がる」といった話は、根拠が示されていない限り鵜呑みにしないほうがよいと考えています。

それでも試す価値があると思うのは、コストが小さいからです。仕様（[llmstxt.org](https://llmstxt.org/)）では、次の順で書くことになっています。

- H1：サイトやプロジェクトの名前（**唯一の必須要素**）
- 引用ブロック：短い要約
- 任意：補足のMarkdown（見出しを除く）
- 任意：H2 見出しごとのリンク集（`[名前](URL): 補足`）。`## Optional` は、短くしたいときに読み飛ばせる二次的な情報、という慣習

例（架空のブログ）：

```md
# My Blog

> AIエージェントとアプリ開発について、個人開発の経験をもとに書いている技術ブログです。

記事は日本語です。コード例は各記事内に掲載しています。

## 記事

- [AIOとは](https://example.com/posts/aio.md): セマンティックHTML、JSON-LD、robots.txt、llms.txtの入門
- [Expoでアプリを作る](https://example.com/posts/expo.md): Expo / React Native の個人開発メモ

## プロフィール

- [自己紹介](https://example.com/about.md): 筆者のプロフィール

## Optional

- [過去記事の一覧](https://example.com/archive.md)
```

リンク先には、HTMLではなく広告やナビゲーションを除いたMarkdown版を用意する（`page.html.md` や `page.md`）ことも、同じ仕様ページで提案されています。

「アクセス制御」という点では、`llms.txt` は制御ではなく**案内**です。許可・拒否の意思表示は robots.txt、案内は llms.txt、と役割を分けて考えると整理しやすいです。

## 4. APIやデータの取得しやすさ・パフォーマンス

最後の観点は、地味ですが一番「エンジニアの仕事」らしい部分だと思います。これは私の意見です。

- **安定したURLとURL設計**：同じ内容が複数のURLに散らばっていると、読む側は迷います。正規URL（`<link rel="canonical" href="...">`）を指定する。
- **機械向けの出口を用意する**：サイトマップ（`sitemap.xml`）、RSS/Atom、公開してよいデータならJSONのAPIなど。HTMLをスクレイピングしなくても取れる経路があると、読み手も作り手も楽になります。
- **ステータスコードを正しく返す**：存在しないページに `200` を返す（いわゆるソフト404）などは避ける。
- **軽く、速く**：重いページはクロールされる量やタイムアウトに影響しえますし、人間にとってもユーザー体験が悪化します。何をもって「速い」とするかは測って決める（ここでは数字は挙げません）。
- **アクセシビリティ**：キーボード操作、`alt`、ラベル付きフォームなど。セマンティックHTMLと同じく、人にも機械にも効きます。

## 自分のサイトに当てはめるチェックリスト

- [ ] 主要コンテンツが `main` / `article` / `h1`〜 で構造化されている
- [ ] 重要な本文が、JS実行前のHTMLに含まれている
- [ ] ページ内容と一致したJSON-LD（Article / SoftwareApplication など）を入れた
- [ ] robots.txt で、学習用・検索用・ユーザー操作用を区別して方針を決めた
- [ ] （任意）llms.txt を置いた。ただし効果は保証されない前提で
- [ ] 正規URL、サイトマップ、正しいステータスコード
- [ ] 計測して遅い箇所を直した

私自身も、これから自分のアプリの紹介ページや技術ブログを、このリストで見直していきたいと考えています。

## おわりに

「AIに見つけてもらえない、構造を理解してもらえないサイトは、存在しないのと同じになりつつある」というのが、私の問題意識です。ただし、どう読まれるかはエージェントごとに違い、今後も変わるでしょう。だからこそ、特定のAIに合わせた小手先の対策よりも、**標準に沿って意味をきちんと書く**ことが、人間にもAIにも長持ちする対策だと考えています。

皆さんは、開発現場でAIエージェント向けの最適化を意識し始めていますか？ぜひコメントで教えてください。

## 参考リンク

- [llms.txt 仕様（llmstxt.org）](https://llmstxt.org/)
- [Schema.org](https://schema.org/) / [Article](https://schema.org/Article) / [SoftwareApplication](https://schema.org/SoftwareApplication)
- [Google: 構造化データの概要](https://developers.google.com/search/docs/appearance/structured-data/intro-structured-data)
- [Google: Article 構造化データ](https://developers.google.com/search/docs/appearance/structured-data/article)
- [Google: ソフトウェアアプリ構造化データ](https://developers.google.com/search/docs/appearance/structured-data/software-app)
- [Google: クローラーの概要（Google-Extended を含む）](https://developers.google.com/crawling/docs/crawlers-fetchers/overview-google-crawlers) / [Google の一般的なクローラー](https://developers.google.com/crawling/docs/crawlers-fetchers/google-common-crawlers)
- [OpenAI: Overview of OpenAI Crawlers](https://developers.openai.com/api/docs/bots)
- [Anthropic: Does Anthropic crawl data from the web...](https://privacy.claude.com/en/articles/8896518-does-anthropic-crawl-data-from-the-web-and-how-can-site-owners-block-the-crawler)
- [Perplexity: Perplexity Crawlers](https://docs.perplexity.ai/docs/resources/perplexity-crawlers)
- [RFC 9309: Robots Exclusion Protocol](https://www.rfc-editor.org/rfc/rfc9309)
- [MDN: Semantics](https://developer.mozilla.org/en-US/docs/Glossary/Semantics) / [MDN: `<main>`](https://developer.mozilla.org/en-US/docs/Web/HTML/Reference/Elements/main)

