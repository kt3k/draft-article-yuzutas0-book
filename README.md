yuzutas0 さんの本に載せる予定の記事の原稿

---

# TL;DR

- [🌱buttons][buttons] という趣味の web サービスを作った
- [GitHub][] でソースコードを全部公開
- ランニングコストが ¥0 (ユーザーが増えたら課金すればスケール可能)
- JavaScript only
- 自分が考えうる限りモダンなスタックで作ったつもり

# 概要

[🌱buttons][buttons] (くさ・ボタンズ) という趣味の web サービスを作りました。サービスの内容としては *毎日やるタスク* のボタンを作って、そのタスクが実行出来たら、そのボタンを押して、その記録をカレンダー形式で表示できる (github のような草🌱が生える) というシンプルなサービスです。(他のユーザーの 🌱の様子を眺めたりも出来ます)

<img width="380" alt="スクリーンショット 2018-10-21 19.08.35.png" src="https://qiita-image-store.s3.amazonaws.com/0/13825/4c6c930f-9042-474b-ed1e-933e159862c0.png">

ここ最近、仕事でも趣味でもスクラッチから web サービスを作っていなかったため、今 0 からサービスを作る場合ってどうやるのが良いんだっけを自分で確認する意味もありつつ、また、誰かの参考になれば良いなという思いも込め [GitHub][] 上で全て公開しつつ作ってみました。

# 全体アーキテクチャ

全体的なアーキテクチャとしては DDD と JAMstack を採用しています。

## [DDD][ドメイン駆動設計]

DDD ([ドメイン駆動設計][]) は大雑把に言えば、ビジネスロジックとそれ以外をきちんと区別しながら綺麗に (複雑性を抑えながら) アプリケーションを作るための設計手法です。🌱buttons では User, Button, Check, など[5つのドメインモデル](https://buttons.kt3k.org/domaindoc/)を定義し [src/domain](https://github.com/kt3k/buttons/tree/master/src/domain) 以下にビジネスロジックをまとめています。また [domaindoc][] というツールを使ってドメインモデルの[ドキュメント](https://buttons.kt3k.org/domaindoc/)を書き出して、アプリケーションが扱っているモデルが何かがズレないように注意しながら作っています。

個人的に DDD を採用するメリットで感じていることは、データ (モデル) の扱いに関する混乱が避けられること、テストすべきことに迷いが少なくなること、などが大きいと感じています。

## [JAMstack][]

JAMstack は [Netlify][] が提唱する新しい web の作り方で、サーバーサイドでのダイナミックな HTML 生成を一切やめて、静的サイトジェネレータでビルドした HTML を静的配信して、そこから API 通信で動的要素を取得することで web サイトを構築すべきとする設計手法です。

詳しくは Biilmann の[スライド](https://speakerdeck.com/biilmann/the-jam-stack) を参照してください。

個人的には JAMstack を採用することで、サーバーサイドとクライアントサイドの役割の違いが明確化されると感じていて、特にサーバーサイドは 100% API の作成のみに注力すればよくなるため、開発効率の向上に繋がると感じています。

🌱buttons では、後述の [bulbo][] という静的サイトジェネレータでビルドした HTML から、[Now][] 上でホストした API に動的要素をリクエストするという形でアプリケーションを構築しています。

# インフラ

サービスを載せるインフラとして [Auth0][], [Now][], [mlab][], [Netlify][] の4サービスを使っています。通常のサーバーは一切利用していません。全てのサービスを無料枠だけで構築しているため、ランニングコストは ¥0 です。(どのサービスも有料プランへアップグレード可能なため、ユーザーが増えた場合は課金すればスケールアップ/スケールアウトすることが出来ます。)

## [Auth0][]

Auth0 は認証専門の BaaS (Backend as a Service) です。月のアクティブユーザーが7,000 ユーザーまでは無料で使うことが出来ます。無料プランの場合、ログの保存期限に制限があってログインに関する調査可能範囲に限界があるようですが、とりあえずログインできればいいレベルのカジュアルな目的であれば無料で問題なく使えます。サービスが成長して、ログインの監査ログをきちんと残したくなった場合は課金すればアップグレードすることができます。

🌱buttons の場合は、ソーシャルログインのみでログインを実装したかったのですが、自前で OAuth2 認証を複数サービスに対して実装するのは相当手間がかかるため、Auth0 に認証機構を全て任せてしまいました。Auth0 を使うと自動的に JWT トークンで認証するようになります。session を自前で実装する必要もなくなるため、非常に工数が削減できたと感じました。

## Zeit [Now][]

Zeit [Now][] は無料から使える immutable なバックエンドサービスです。node.js アプリケーションないし docker コンテナを Now に対してデプロイすることができます。[Heroku][] と同様で、課金することでインスタンス数を増やしてスケーリングしていくことが出来ます。Heroku と異なる点は、Now はデプロイ時に勝手にサービスを新しいインスタンスに向けない点が少し違います。`now deploy` すると、ハッシュ値付きの新しいインスタンスが立ちあがって、そのインスタンスがステージング環境として機能します。ステージング環境で、十分に動作を確認してから、`now alias` で向き先を新しいインスタンスに向き変えるということで、リリースが出来ます (つまり自然に[ブルーグリーンデプロイ](https://www.publickey1.jp/blog/14/blue-green_deployment.html)のようになります)。

🌱buttons では、[node.js][] と [express][] で構築したAPI サーバーを Now にデプロイして動かしています。

## [mlab][]

(注、この記事を書いている時点で mlab は MongoDB, Inc. に買収されてしまいました。今後は [MongoDB Atlas][] という mlab と類似のサービスに統合されていくようです。)

mlab は MongoDB のマネージドサービスです。平たくいえば、MongoDB がインストールされたサーバーを MongoDB ごと貸してくれて、DB としての面倒な管理なども全てやってくれるというサービスです。500MB までの MongoDB インスタンスであれば、無料で貸してくれます。🌱buttons はこの 500MB の無料枠を使って実装しています。(mlab は買収されてしまいましたが、買収元の MongoDB, Inc. がやっている [Atlas][mongodb atlas] という同様のサービスでも同じように無料枠 500MB があります。)

## [Netlify][]

Netlify は無料の静的サイトホスティングサービスです。GitHub のレポジトリと連携することが出来て、特定ブランチの内容と同期して、静的サイトを自動的に更新してくれます。また、ブランチの更新にフックして、静的サイトのビルドコマンドを指定することも出来て、ブランチ更新 -> Netlify 上で自動ビルド -> サイト更新という簡単なワークフローを組む機能も持っています。

🌱buttons では、GitHub 上にはビルド前のソースコードだけを commit して、後述の [bulbo][] というビルドツールでビルドした静的サイトを Netlify 上にホスティングしています。

# Database

## [MongoDB][]

MongoDB はドキュメントデータベースと呼ばれるタイプのデータベースです。RDBMS よりもお手軽に開発を開始できます。

node.js では [mongoose][] という ODM ライブラリが昔から活発にメンテナンスされ続けていて、非常に安定感があり、また、MongoDB Atlas / mlab の様な無料のマネージドサービスが存在しているため、手っ取り早くサービスをプロトタイピングしたい目的にはかなり合致していると個人的には思っています。

🌱buttons では、ユーザーの情報、ボタンの情報、ボタンを押した記録の情報などを MongoDB に格納して、それを [node.js][] で書いた API サーバー経由で取り出して利用しています。

# Build ツール

## [bulbo][]

bulbo は [gulp][] 互換なパイプラインでビルドを記述できる静的サイトジェネレータです。gulp と互換性があるため、非常に豊富な gulp エコシステムからプラグインを選んで使えるため、カスタマイズ性が高く、ニーズに合わせて多様なビルドを組むことが出来る点が特徴です。

🌱buttons では、全てのフロントエンドリソースを bulbo でビルドして、Netlify 上にデプロイしています。

(disclosure: 筆者は bulbo の作者です)

# [docker][]

docker はコンテナと呼ばれる軽量な仮想環境の中でシステムを動かすためのツールです。2013年に最初のバージョンが公開されたかなり新しいツールですが、現在では開発環境・CI環境・本番環境などあらゆる場面で docker の活用が進んできています。

🌱buttons では開発環境での MongoDB のセットアップや、CI 環境で docker を利用しています。後述のように Kubernetes 利用を見送った理由もあり、本番環境では docker を使っていません。

# Backend

## [node.js][]

node.js はサーバーサイドで動く JavaScript 実行環境です。2009年に発表されて以来、非常に活発に開発が続けられており、また、node.js のパッケージレジストリである npm には[非常に多くのパッケージ](http://www.modulecounts.com/)が登録されており、巨大なエコシステムを形成しており、非常に安定感のあるプラットフォームであると言えます。

🌱buttons では node.js で REST API を作成して動的なデータのやりとりに利用しています。

## [express][]

express は node.js 用のサーバーサイドフレームワークです。 node.js 用のサーバーサイドフレームワークとしては他に、[hapi][], [koa][], [fastify][], [sails][], など様々なものがありますが、ユーザー数では express が圧倒的に多く、middleware のエコシステムも豊富なため、非常に安定感があります。

🌱buttons では express を使って、[12本](https://github.com/kt3k/buttons/blob/master/src/routes/index.js)の REST API エンドポイントを実装しています。

# Frontend

## [capsid][]

capsid はコンポーネントベースの DOM プログラミングをするための、軽量なフレームワークです。JavaScript のクラスと DOM をペアリングしてコンポーネントを構成するという点で、[Backbone][] と似ていますが、jQuery に依存していない点、JS クラスと DOM との対応は必ず 1対1である点などが、capsid の特徴です。また、イベントのハンドリングに対して、豊富なユーティリティデコレータが提供されていて、親から子/子から親への情報伝達を DOM イベントを使ってうまく実装できる点が特徴的です。

🌱buttons では、全てのフロントエンドロジックを capsid のコンポーネントの積み重ねで表現しています。

(disclosure: 筆者は capsid の作者です)

## [cal-heatmap][]

cal-heatmap は github の草カレンダー的なカレンダーを出すためのライブラリです。日別の表示以外にも月単位/時間単位/分単位など、多様な出力方法を持っています。この手のライブラリは他にもかなりの数が OSS として公開されていますが、中でも cal-heatmap は API ドキュメントがかなり充実しており、安定感があります。

🌱buttons の草カレンダーは cal-heatmap で生成しています。

## [bulma][]

bulma はいわゆる CSS フレームワークです。CSS フレームワークの有名どころとしては、他に[Bootstrap][] や [Foundation][] などがありますが、bulma はそれらより後発で軽量なフレームワークです。bootstrap や foundation は一部 JS を含むコンポーネントもありますが、bulma は純粋に CSS のみである点が特徴です。

🌱buttons では基本的な UI パーツは bulma で実装しています。

## [date-fns][]

date-fns は比較的新しい時刻を扱うライブラリです。🌱buttons では、日付毎の button の扱い周辺で、date-fns を多用しています。

少し前までは、JavaScript のライブラリの中では、[moment][] が圧倒的に安定感のあるライブラリとして、特出していましたが、最近では date-fns, dayjs, luxon などの軽量な代替ツールが出てきています。中でも date-fns は native の Date オブジェクトをそのまま日付の表現として扱っていて、ライブラリとしては関数だけを提供するという点が特徴的です。

# Testing

## [CircleCI][]

CircleCI は CI (継続的インテグレーション = サーバー上で自動的にテストを実行すること) を提供してくれるサービスです。CI を提供するサービスは現在では非常に数が多いですが、中でも CircleCI は 1) [コンテナベース](https://circleci.com/docs/2.0/containers/)で環境を構築できる点 2) ジョブ / ワークフロー定義の柔軟性 3) ビルドキャッシュの扱い 4) 万一の場合のための ssh アクセスの提供、などの点で他のサービスより優れていると個人的には思っています。

🌱buttuns ではバックエンドのユニットテストを [CircleCI 上で](https://circleci.com/gh/kt3k/buttons)回しています。

## [Codecov][]

Codecov はテストカバレッジレポートをコミットごとに全て記録してくれる web サービスです。単にコミットごとに記録するだけではなく、PR に対しては、その PR をマージするとどの程度カバレッジが上がる/下がるという情報を独自の [Codecov Δ](https://docs.codecov.io/docs/codecov-delta) という記法でレポートしてくれたり、ディレクトリツリーの各ノード毎のカバレッジの具合をサンバーストチャートで表示してくれたりと、テストに対するモチベーションを高めてくれる様々な工夫がなされた、非常に優れたサービスです。

Codecov のダッシュボード
<img width="485" alt="スクリーンショット 2018-10-22 14.32.12.png" src="https://qiita-image-store.s3.amazonaws.com/0/13825/ada82fd8-fc68-0c89-8816-f37b1ef396ba.png">

🌱buttons ではサーバーサイドのテストカバレッジを [codecov にアップロード]()して、テストのカバー率をモニタリングしつつ開発を進めました。

## [kocha][]

kocha は [mocha][] 互換のテストランナーライブラリです。基本的には mocha と同じ挙動ですが、global にキーワードを露出しないため、lint ツールなどの相性が良く、mocha 用の特殊な設定をしなくても lint を通すことが出来ます。

🌱buttons では kocha を使って、domain のユニットテストと API のユニットテストを実行しています。

(disclosure: 筆者は kocha の作者です)

## [power-assert][]

power-assert は node.js の [assert][] モジュール互換の、リッチな出力を持ったアサーションライブラリです。通常のアサーションライブラリと違い、アサーションした式の各部分がどの様な値になっていたかが詳細にテスト結果に表示されるため、特に複雑なオブジェクトのテストをする際に開発効率アップに繋がります。

🌱buttons では基本的に全てのアサーションに power-assert を利用しています。

## [nyc][]

nyc は node.js のテストカバレッジを計算してくれるツールです。以前は istanbul, blanket, JSCoverage, isprata, etc など、雑多な JavaScript 用カバレッジツールが乱立していましたが、最近はかなり nyc に収束してきた感があります。nyc が他のツールより優れている点はとにかくセットアップが簡単という点にあります。テストを実行するコマンドの前に nyc をつけて実行する。これだけで魔法のようにカバレッジレポートが出力されるようになります。

🌱buttons でも nyc を利用していて、セットアップにはほぼ全く時間がかかりませんでした。ありがとう [@bcoe](https://github.com/bcoe)! 🙌

## [mock-express-response][], [mock-express-request][]

[mock-express-response][], [mock-express-request][] は express の request, response オブジェクトのテスト用モックライブラリです。

🌱buttons では各ルートのテストは API を HTTP 経由で直接叩くのではなく、ルートのハンドラーに対して、上記のモックリクエスト/レスポンスオブジェクトを入力することで、ユニットテストを行なっています。 [例](https://github.com/kt3k/buttons/blob/master/src/routes/__tests__/users-id-checks.js)

このライブラリは今回初めて使いましたが、セットアップがそこまで面倒ではなく、使い勝手が良いと感じました。

# 採用しなかったもの

## [Kubernetes][]

[Kubernetes][] は [Cloud Native Computing Foundation](https://www.cncf.io/) が開発する次世代のコンテナオーケストレーションプラットフォームです。当初は(元)Google のエンジニアを中心に始まったプロジェクトでしたが、今や AWS や Azure でもマネージドな Kubernetes クラスターが提供されるようになり、業界標準になった感が強いです。🌱buttons も出来れば Kubernetes 上にデプロイ出来ればと考えましたが、コスト 0 で Kubernetes クラスターを立てる手段が無いように思われたため、今回は利用を見送りました。

## [Firebase realtime database][]

Firebase realtime database はクライアン側とサーバー側でデータが自動的に同期される新しいタイプの database です。Google が提供するマネージドサービスがあり、1GB のストレージが利用できる無料プランもあるため、プロトタイピングに非常に適していると思われます。

ただ、個人的には、Firebase realtime database 自体が OSS でない点、あまりにも Google に依存しすぎている点などが懸念となって採用を見送りました。

# あとがき

筆者が趣味の Web Programming を始めた 2000年代始めごろは、動的言語の Web フレームワークはまだリリースされておらず、Web プログラミング (CGI) といえば必要な知識はほとんど perl だけという感じだったように思います。その後 Catalyst, CakePHP, Django, Rails, などのいわゆる (動的言語向けの) Web フレームワークが次々登場し、Web 2.0 になってフロントエンド JavaScript が複雑化する中で、これが正解と言える Web プログラミングのスタイルがなかなか分かりづらい時期が続いていたように思います。また、上から下までの動的な Web サイトを作るのに必要な知識が相当に増えてしまったように感じました。そのせいか、(統計がある訳ではありませんが)自分の観測範囲では、誰かが趣味で作った変な Web サービスを見かける機会が減ってしまったように感じました。

🌱buttons では、perl しか無かった頃によく見かけた誰かが勢いで作った変な Web サービスの雰囲気を出しつつ、しかし中身はモダンでスケールできるサービスのコアとなり得るような作りを目指して作ってみたつもりです。この記事を読んで、変な思いつきの/独創的な Web サービスを作る人が出てくれたら良いなと思います。

# 追記

## 工数について (2018-10-24 追加)

<blockquote class="twitter-tweet" data-lang="en"><p lang="ja" dir="ltr">これはいいですね〜<br>構成もモダン😎<br>どれくらい工数かかってるのか気になりますね！<br><br>Netlify+Node.js+MonogoDBでJAMstackな草を生やすサービスをOSSで公開しました! <a href="https://t.co/RwJycWKPKz">https://t.co/RwJycWKPKz</a> <a href="https://twitter.com/hashtag/Qiita?src=hash&amp;ref_src=twsrc%5Etfw">#Qiita</a></p>&mdash; なべっつ🍲 / ためしがき (@nabettu) <a href="https://twitter.com/nabettu/status/1054922990855479296?ref_src=twsrc%5Etfw">October 24, 2018</a></blockquote>
<script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

工数についてですが、フルタイムの仕事の余り時間を使って1.5ヶ月程度かかっています。 ([initial commit は 9/3](https://github.com/kt3k/buttons/commit/fa40386e25c52401d80dab11edd91e9f5e153a0b)、[大まかに大体の機能が揃ったのが 10/12](https://github.com/kt3k/buttons/commit/eecb1569e93f4385869a917d3430fdee4d2aeafa)あたりです。その後ユーザーからの Feedback 等で気がついた XSS の修正などを継続的に入れています。)

(詳しくは筆者の[週報ページ](https://shuho.kt3k.org/)にも記載しています。)


[buttons]: https://buttons.kt3k.org/
[GitHub]: https://github.com/kt3k/buttons
[ドメイン駆動設計]: https://ja.wikipedia.org/wiki/%E3%83%89%E3%83%A1%E3%82%A4%E3%83%B3%E9%A7%86%E5%8B%95%E8%A8%AD%E8%A8%88
[domaindoc]: https://www.npmjs.com/package/domaindoc
[JAMstack]: https://jamstack.org/
[Netlify]: https://www.netlify.com/
[Kubernetes]: https://kubernetes.io/
[Auth0]: https://auth0.com/
[Now]: https://zeit.co/now
[Heroku]: https://www.heroku.com/
[mlab]: https://mlab.com/
[mongoose]: https://www.npmjs.com/package/mongoose
[bulbo]: https://www.npmjs.com/package/bulbo
[docker]: https://www.docker.com/
[node.js]: https://nodejs.org/en/
[express]: https://expressjs.com/
[capsid]: https://github.com/capsidjs/capsid
[cal-heatmap]: https://cal-heatmap.com/
[bulma]: https://bulma.io/
[date-fns]: https://date-fns.org/
[circleci]: https://circleci.com/
[codecov]: https://codecov.io/
[kocha]: https://github.com/kt3k/kocha
[power-assert]: https://github.com/power-assert-js/power-assert
[nyc]: https://www.npmjs.com/package/nyc
[mock-express-response]: https://www.npmjs.com/package/mock-express-response
[mock-express-request]: https://www.npmjs.com/package/mock-express-request
[assert]: https://nodejs.org/api/assert.html
[mocha]: https://mochajs.org/
[gulp]: https://gulpjs.com/
[Firebase realtime database]: https://firebase.google.com/docs/database/
[Ruby on rails]: https://rubyonrails.org/
[moment]: https://momentjs.com/
[bootstrap]: https://getbootstrap.com/
[foundation]: https://foundation.zurb.com/
[backbone]: http://backbonejs.org/
[V8]: https://github.com/v8/v8
[hapi]: https://www.npmjs.com/package/hapi
[koa]: https://www.npmjs.com/package/koa
[fastify]: https://www.npmjs.com/package/fastify
[sails]: https://www.npmjs.com/package/sails
[mongodb]: https://www.mongodb.com/
[mongodb atlas]: https://www.mongodb.com/cloud/atlas

