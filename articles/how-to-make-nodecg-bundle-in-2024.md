---
title: "NodeCGを活用した配信レイアウトができるまで2 - 2024年編(不思議RTAフェス5)"
emoji: "📹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript","nodejs","nodecg"]
published: false
---

# はじめに
この記事は以前私が投稿した[NodeCGを活用した配信レイアウトができるまで - 不思議RTAフェス2編](https://zenn.dev/yagamuu/articles/how-to-make-nodecg-bundle)という記事のアップデート版に当たる記事です。  
あの記事を投稿してからおよそ3年が経過しスタッフのチーム構成や開発環境などに様々な変化が生じたことや、今でもありがたいことに前回の記事を参考にする方がいらっしゃるので、改めて現状の開発工程を記しておこうと思いました。  
そのため必須ではありませんが、文脈や変更点などがより理解しやすくなると思われるため以前の記事を事前に読む事を推奨します。

そもそもNodeCGをよく知らないという方は、以下の記事の参照を推奨します。

- [ライブ配信レイアウトを作るNode.jsのフレームワーク](https://qiita.com/Hoishin/items/36dcea6818b0aa9bf1cd)
- [【2020年版】1から学ぶNodeCG#1：NodeCG導入編](https://zenn.dev/cma2819/articles/start-nodecg-01)

## 当記事の想定読者
- [前回の記事](https://zenn.dev/yagamuu/articles/how-to-make-nodecg-bundle)を読んだ方
- 最低限のWeb/JavaScript関連の知識がある方
- NodeCGを活用した具体的なレイアウト開発の流れを知りたい方

# TL;DR
- 3年前は主催が仕様策定とレイアウトデザインを一任するワンオペだったが、スタッフが増え且つ適切な役割分担がされたことでイベントの円滑な準備が可能になった
- 3年前はVue2を採用していたが時代の流れと共にVue3に移行した
- Dockerを用いることで環境構築が本番/開発共にかなり楽になった
- レイアウトのデプロイ先はCloud Run辺りがオススメかも

# その1: 配信レイアウトのデザイン作成、各種仕様の策定
3年前イベント運営に関わるメンバーは自分を含めても6～7名しかおらず、レイアウトのコード開発ができるのは自分一人かつデザイナーも不在という状況でした。
そのため当時はデザイン作成、仕様策定などを作業に慣れない主催が一任することになり、非常に大きな負担を抱えさせてしまいました。  
しかし2024年現在はイベントとしての規模も大きくなり、それに伴い大規模なスタッフ募集を行ったことでWebデザインを得意とするスタッフが加入し開発がスムーズになりました。
またファシリテーターやストリーミング技術全般担当、広報担当など各分野における専門スタッフも多く加入したことで、各自の作業負担も減った上にイベント全体で実現出来る物が広がったことで参加者の満足度を向上させることができました。  
人員はむやみに増やせば良いというわけではありませんが、各分野に適切なスタッフを配置することの重要性を再確認しました。  
ちなみに配信レイアウト周りを担当するスタッフは現在以下のようになっています。苦手としていたデザイン作成やCSS実装を担当してくれる方が増え自身の作業進行も3年前と比較し非常に楽になりました。

- レイアウトのコード開発全般(筆者)
- レイアウトのデザイン担当1名(Webデザインも可能な為部分的にコードも担当)
- デザインを元としたHTML/CSSコーディング担当1名

# その2: 実装に使用する技術の選定(2024年版)
## 1. 言語、フレームワーク(Vue.js/TypeScript)
2024年現在も引き続きVue/TypeScriptを採用しています。
Vue2+TypeScriptの開発体験があまり良くなかったことや他のNodeCGレイアウトを実装している方の多くが採用していたことからReactに移行することも視野に入れてましたが、以前も紹介したレイアウト開発用テンプレートの大規模アップデートが行われ不満点が大きく改善された為、引き続きの採用に至りました。

## 2. NodeCGのテンプレート(nodecg-vue-ts-template)、関連ライブラリ
ではそんな大規模アップデートが行われたテンプレート、[nodecg-vue-ts-template](https://github.com/zoton2/nodecg-vue-ts-template)ですが、3年前から以下の変更が加わりました。

- Vue3系に対応
- Vue3移行に合わせ各種デコレータの廃止
- [nodecg-vue-composable](https://github.com/Dan-Shields/nodecg-vue-composable)というReplicantの使用を支援するコンポーザブルヘルパーを採用^[上記の採用に合わせVuexの採用は廃止されました]
- ビルドツールをWebpackからViteに移行
- UIライブラリをVuetifyからQuasarに移行^[これはテンプレートのアップデート当時まだVuetifyのVue3対応がされてなかった為です。]
- NodeCGの型定義参照元に[nodecg-types](https://github.com/codeoverflow-org/nodecg-types)を使用^[ただしNodeCGのv2アップデートにより使用するメリットが無くなったため、v2対応に合わせて廃止される予定です]

Vue3でComposition APIなどの新機能が使えることになったことや、開発体験を悪くする原因の一つであったデコレーターの廃止、そしてビルドツールのVite移行などにより大きく開発体験が向上しました。
ただし現在のテンプレートは(対応に向けた開発用ブランチが切られていますが)NodeCGの最新バージョンに対応していないので、筆者は[Fork先でカスタマイズした物](https://github.com/yagamuu/nodecg-vue-ts-template)を使用しています。

## 3. RTAイベント向けレイアウト用各種機能セットbundle(nodecg-speedcontrol)
こちらも3年が経ったことで様々なアップデートが行われました。主な変更点としては以下の通りです。

- 各プレイヤーに個別のカスタムデータ列を追加できるようになった
- 各プレイヤーに代名詞(pronouns)列が追加
- Oengusからのインポート機能が強化(カスタムデータ、出身国、代名詞のインポート対応)

特に嬉しいのは各プレイヤー個別のカスタムデータ列追加機能だと思います。これによりRTAイベント配信によくある各SNSのアカウント情報表示などの実装を行いやすくなりました。

## 4. 解説者、プレイヤーのSNSアカウント情報追加(speedcontrol-additions)
以前使用していたこのbundleですが、現在の不思議RTAフェスではプレイヤー/解説者のSNS情報はレイアウトに表示せずに公式X(旧Twitter)アカウントから告知する方針となった為現在は使用していません。
ただしbundleのアップデートがつい先日行われ、speedcontrolやNodeCGの最新バージョンへの対応などにより運用しやすくなったので、気軽にSNSアカウント情報表示の実装を行いたい方は使ってみると良いかもしれません。

## 5. 外部Webサービスとの連携
以前はX(旧Twitter)のツイート表示機能を実装していましたが、例のAPI仕様変更等に合わせ廃止しています。
BGMの情報表示は引き続き[nodecg-spotify-widget](https://github.com/cma2819/nodecg-spotify-widget)を使用していますが、現在NodeCGの最新バージョンへの対応がされておらずそのまま使うことはできない為注意が必要です。

# その3: 実装
以前の記事では実装にまつわる話をいくつか書きましたが、あの記事からテンプレートのアップデート以外で大きく手法や構造を変えたことをした覚えが無いのであまり書くことがありません。精々Replicant依存のデータ加工をコンポーザブル化したくらいでしょうか……？
少し味気ないので以前の記事で苦戦していた可変幅文字列表示についての話について記してみます。

## プレイヤー解説の名前+SNSアカウント表示(+ゲームタイトル)
名前やアカウント文字列は可変幅であるため、デザインが崩れないように考慮したCSSの実装が難しい……という話でしたが、こちらについてはデザイナーが各参加者の最長文字数を元にカスタマイズしたデザインを作成するという力技で対応しています。
ちなみに以前紹介した[RTA in Japanのレイアウトにおけるフォントサイズを改変するロジック](https://github.com/RTAinJapan/rtainjapan-layouts/blob/master/src/browser/graphics/components/lib/fit-text.tsx)は、現在改行コードへの対応などが行われパワーアップしてますので、気になる方は見てみると良いかもしれません。

# その4: サーバー構築
以前はサーバー内で直接各パッケージのインストールや証明書発行などの環境構築を行っていましたが、依存パッケージの変更に対応したり環境を共有するなどの行為が面倒なので現在はDockerを導入し、本番環境でもそのままDockerからイメージを起動することで公開しています。
Dockerを併用したNodeCGサービスの外部公開には[https-portal](https://github.com/SteveLTN/https-portal)というイメージを使用しています。
このイメージは80/443ポートで公開されているWebアプリケーションをNginxを動作させることで自動でSSL対応付きで公開し、SSL証明書の発行や自動更新も行う便利なイメージです。
これによりNodeCG本体の設定を外部公開に合わせ変更する必要がほぼ無くなったのも嬉しい点です^[NodeCGだけで完結するにはホスト名指定、公開ポートの変更や証明書のパス指定などが必要です]。
参考として使用しているcomposeファイルのサンプルを掲載しておきます。
```yaml
services:
  https-portal:
    image: steveltn/https-portal:1
    ports:
      - '80:80'
      - '443:443'
    links:
      - nodecg
    restart: always
    environment:
      DOMAINS: 'example.com -> http://nodecg:9090'
      STAGE: 'production'
      CLIENT_MAX_BODY_SIZE: 100M # 指定しないとassetsファイルのアップロードなどで躓きます
      WEBSOCKET: 'true' # NodeCGは基本的にデータ通信にWebSocketを使用するため指定がほぼ必須です
    volumes:
      - https-portal-data:/var/lib/https-portal
  nodecg:
    build:
      context: .
      dockerfile: Dockerfile
    user: root
    command: sh -c "npm i --omit=dev && node --enable-source-maps ../.."
    working_dir: /opt/nodecg/bundles/mysrtafes2024-layouts
    init: true
    ports:
      - 9090:9090
    volumes:
      - nodecg_db:/opt/nodecg/db
      - nodecg_assets:/opt/nodecg/assets
      - node_modules:/opt/nodecg/bundles/mysrtafes2024-layouts/node_modules
      - ./cfg:/opt/nodecg/cfg:ro
      - ./package.json:/opt/nodecg/bundles/mysrtafes2024-layouts/package.json:ro
      - ./package-lock.json:/opt/nodecg/bundles/mysrtafes2024-layouts/package-lock.json
      - ./dashboard:/opt/nodecg/bundles/mysrtafes2024-layouts/dashboard:ro
      - ./extension:/opt/nodecg/bundles/mysrtafes2024-layouts/extension:ro
      - ./graphics:/opt/nodecg/bundles/mysrtafes2024-layouts/graphics:ro
      - ./schemas:/opt/nodecg/bundles/mysrtafes2024-layouts/schemas:ro
      - ./src:/opt/nodecg/bundles/mysrtafes2024-layouts/src:ro
      - ./shared:/opt/nodecg/bundles/mysrtafes2024-layouts/shared:ro

volumes:
  nodecg_db:
  nodecg_assets:
  node_modules:
  https-portal-data:
```

また、以前の記事にてサーバーサービスの選択にVultrを推奨していました。その理由の中にスナップショット機能の無料を挙げていましたが残念ながら有料となってしまいました。
月額0.05$/GBとそこまで高価でも無いので現在もそのまま使用していますが、Dockerイメージがそこそこ容量を食うこともあり以前ほど安価で済ますことができなくなってしまいました。
現在はCloud Runというサービスが気になっており、こちらは任意のDockerイメージをデプロイするだけで自動でアプリの実行/公開が可能なサービスとなっています。
実際にCloud RunでNodeCGレイアウトをデプロイした他の方から以下の利点を共有して頂いており、開発体験の向上に繋がりそうと考えています。

- VSCodeの拡張機能を使用すればGUIでデプロイ可能
- `*.run.app`というドメインを提供してくれる為ドメイン名にこだわらない場合はそのまま使用可能
- Cloud Storageと合わせることで問題なくReplicantの永続化も可能
- WebSocketに対応

利用価格についても無料枠があるため数日間だけ起動すれば良い今回の要件では安価に済ませれそうなので、もしVultrから移行する強い動機が発生したら移行しようかと考えています。

# まとめ
以前の記事と比較すると変更点だけを述べたため非常にコンパクトな記事になり、内容が薄い記事となってしまったかもしれませんが以前の記事では古くなってしまった点の補足という目的は達成できたのではないかと思います。
もしNodeCGレイアウトの実装における疑問点などがありましたら当記事のコメント欄や、NodeCGのDiscordサーバー(日本語チャンネルもあります)にて気軽に質問して頂ければと思います。
改めましてこの記事がNodeCGレイアウト開発の一助となれれば幸いです。

## 参考リンク
- [NodeCG 公式サイト](https://www.nodecg.dev/)
- [NodeCG 公式Discordサーバー](https://discord.com/invite/GJ4r8a8)
- [第5回不思議のダンジョンRTAフェスのレイアウトリポジトリ](https://github.com/yagamuu/mysrtafes2024-layouts)