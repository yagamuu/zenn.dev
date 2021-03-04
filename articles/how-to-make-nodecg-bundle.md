---
title: "NodeCGを活用した配信レイアウトができるまで - 不思議RTAフェス2編"
emoji: "📹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["javascript","nodejs","nodecg"]
published: false
---

# はじめに
ここ数年でRTAコミュニティ並びに各種ゲームコミュニティで注目を浴びるようになったストリーミング配信レイアウト開発フレームワークであるNodeCGですが、実用的な配信レイアウトが具体的にどのような流れで開発が行われているかを記した記事は現在までほとんど存在していないように思われます。  
そこで今回は私が先日開発を行った、[第2回不思議のダンジョンRTAフェス](https://oengus.io/marathon/mysrtafes2)というRTA系イベントの配信レイアウトの開発をどのように進めてきたかを簡潔にですが当記事でお話したいと思います。

まずNodeCGとは何ぞや？と思った方は、以下の記事の参照を推奨します。

- [ライブ配信レイアウトを作るNode.jsのフレームワーク](https://qiita.com/Hoishin/items/36dcea6818b0aa9bf1cd)
- [【2020年版】1から学ぶNodeCG#1：NodeCG導入編](https://zenn.dev/cma2819/articles/start-nodecg-01)

## 当記事の想定読者
- 最低限のWeb/JavaScript関連の知識がある方
- 実際にNodeCGを活用したレイアウト開発を行ったことが無い、または始めたばかりの方
- RTA系イベントの配信レイアウトに必要な要件、仕様を知りたい方

# その1: 配信レイアウトのデザイン作成、各種仕様の策定の依頼
開発を進めるに当たってまず最初は具体的にどのような生産物を作るのかを考える必要があります。今回はイベント主催者の[ポンズさん](https://twitter.com/ponzu24)から依頼を受けて配信レイアウトを開発しており、レイアウトのデザインも彼が担当することになっていたので、まずは配信に必要な各種レイアウトのデザイン作成と、どのような機能が欲しいかを考えてもらう作業をお願いしました。  
最終的に今回は次のゲームへの待機画面とゲーム機種,プレイヤーの人数に対応した7つのゲームプレイ画面のデザインとそのサンプル画像、各種コンポーネント間のマージン情報や使用するフォントの仕様提供、最後にレイアウトに必要な機能^[タイマーやプレイヤーのSNSアカウントの表示、Twitterのツイート表示など]の洗い出しと仕様策定を行っていただきました。
デザイン作成と仕様策定が完了したら、次にレイアウト構築に必要な各種画像素材(レイアウトに表示される各種ロゴ、アイコンや背景画像など)を提供して頂き次はソースコード実装の段階に進みます。

ちなみにポンズさんはデザインに挑戦するのは初めてらしく、Web関連の知識に長けているわけでも無いので開発を行うのに当たってお互いに連携を行うのは難しい状況であったと思います。  
実務でデザイナーの方と連携を行う際も基本的にWebデザインに強い方との連携しか経験が無かったこともあり、自身のヒアリングなどが上手くやれていたかと言うと怪しく、結果的にポンズさんには多くの負担をお掛けしてしまったと思います。  
次回があればこの点を反省し、デザイナー担当者への各種ヒアリングとHTML/CSSコーディングの実務経験に豊富な同コミュニティ某氏に協力をお願いしようと思います……。  
他力本願かい！という感じですが、やはり餅は餅屋と言いますし。~~そもそもぼくの実務における守備範囲ってサーバーサイドのコーディングが中心なので……😇（言い訳）~~

# その2: 実装に使用する技術の選定
## 1. 言語、フレームワーク(Vue.js/TypeScript)
いよいよ本格的に実装を進めていきますが、まずは実装に使用するフレームワークや言語を選定します。
NodeCGを活用したシステム(以下bundleと呼びます)は各種データを操作するための土台である`Dashboard`と`Extension`、そして実際に配信に表示する画面を構築する`Graphics`の3つで構築されていますが、これら3つ、特にGraphicsにおいては開発手法をほぼ自由に選べ、プレーンなJavaScriptとHTML/CSSのみを使用したものから、ReactやVue等の本格的なフロントエンドフレームワークを使用して実装をすることも可能です。
私はフロントエンドフレームワークに関してはVue(2系)の浅い経験しか無い事や、参考にしているbundleがVueを採用している事からGraphicsにおいてはVueで実装を行うことにしました。
次にGraphicsのロジック部分やDashboardとExtensionの実装は`TypeScript`を採用しました。理由としてはやはり現在のJavaScriptコミュニティではTypeScriptを採用したプロジェクトが中心であることや、型が付くことによる予期せぬ不具合の回避、EcmaScriptの最新機能を自由に使える点、各種補完が効く事等メリットが大きいためです。多分気付いてない恩恵も色々ありそう……。
個人的にTypeScriptは経験が浅く型定義などで躓く機会が多く苦手意識があったので、経験を積み多少なりとも克服するという目的もありました。

## 2. NodeCGのテンプレート(nodecg-vue-ts-template)
というわけで採用フレームワーク/言語はTypeScriptとVueになりましたが、この2つを使用するにはコンパイルを行う各種モジュールバンドラーの用意や設定等事前準備が面倒です。ですがNodeCGコミュニティには非常にありがたい事に既にこの2つを採用したbundle開発のテンプレート、[nodecg-vue-ts-template](https://github.com/zoton2/nodecg-vue-ts-template)が公開されています。
こちらのテンプレートには以下の技術的特徴があります。

- TypeScriptとVue(2系)を各範囲で実装するためのモジュールバンドラー、Linterの設定同梱
- `vue-class-component`、`vue-property-decorator`、`vuex-module-decorators`、`vuex-class`等のVue用デコレーターの採用
- NodeCGのReplicant^[NodeCGのDB的な機能であり、Dashboard,Extension,Graphics間をリアルタイムに通信できる。詳しくはこちらの記事を参照 https://zenn.dev/cma2819/articles/nodecg-implements-replicant]をVuexを使用して管理^[個人的にReplicantをVuexで管理するメリットがよくわからないのですが(Replicantにはグローバル変数からアクセスでき実質Storeと同じ役割なので)、Replicantにデータを投げないコンポーネント内で完結してるデータなども含めてStoreで一括管理できる点などが良いのでしょうか？]
- DashboardのUI実装用にVuetifyを同梱

サンプルコードも上記テンプレート内に含まれているので、ありがたく参考にさせて頂きながらこちらのテンプレートを使用し実装を行っていきました。

## 3. RTAイベント向けレイアウト用各種機能セットbundle(nodecg-speedcontrol)
RTAイベントの配信レイアウトに必要な機能はある程度共通しています。特に必要とされるのは実プレイ時間のタイマー表示とプレイヤー/解説者とゲームの情報表示そして切り替えでしょう。
これらの機能を一通り提供してくれるbundleも既に存在しています、それが[nodecg-speedcontrol](https://github.com/speedcontrol/nodecg-speedcontrol)、通称`Speedcontrol`です。
Speedcontrolには多くの機能が含まれており、以下がその一部です。

- 各種ゲームとプレイヤー/解説情報の操作と管理
- [Horaro](https://horaro.org/)、[Oengus](https://oengus.io/)^[イベントのスケジュールを簡潔に作成できるWebサービス、OengusはRTAイベントに特化しておりイベント参加者の募集から応募まで可能]から各種ゲームとプレイヤー/解説情報のインポート
- Twitchの配信タイトルとゲームカテゴリーの自動更新
- RTAプレイヤー愛用のタイマー、[LiveSplit](https://livesplit.org/)のコアシステムを採用したタイマーシステム
- 英語/日本語対応のUI(Configで切り替え)

このようにRTAイベントに必要な機能が一通り揃っており、日本のRTAコミュニティでも多数の採用実績があります。
今回のレイアウト実装でもありがたく活用させていただきました。

一つ注意が必要なのはSpeedcontrolにはGraphicsが用意されておらず、レイアウトとして使用するにはSpeedcontrolから出力される各種データのReplicantを参照し出力するGraphicsを自分で実装する必要があります。
こちらについては開発者の方が[speedcontrol-simpletext](https://github.com/speedcontrol/speedcontrol-simpletext)というサンプルbundleを公開しているので、初めてNodeCGのbundle実装に挑戦し、Speedcontrolを使ってみたいという方はこちらを参考にすると良いかと思います。
上記サンプルbundleをベースに開発されたRTAイベントのレイアウトも存在するので、詳細について興味のある方は以下の記事をご参照下さい。

- [レイアウトをNodeCG化した話](https://hackmd.io/@HiST/SkwhnBKjv)

## 4. 解説者、プレイヤーのSNSアカウント情報追加(speedcontrol-additions)
Speedcontrolを使用することで必要な機能の多くを用意することができました。しかしSpeedcontrolだけでは今回のレイアウト要件の全てを満たすことができません。その一つがプレイヤーと解説者のSNSアカウントの表示です。
SpeedcontrolにはプレイヤーのTwitch IDを管理するデータ列は存在しますが、Twitchを除く他のSNSアカウントと、解説者の名前及びSNSアカウントを管理することはSpeedcontrol単体ではできません。
Speedcontrolにはゲーム情報にデータ列を増やす`Custom Data`という機能が存在し、こちらを使えば解説者の名前を追加する程度は簡単にできますが、プレイヤーと解説が複数人いるケースや表示するSNSアカウントが増えていくケースを考えるとCustom Dataで実装するとなるとデータ構造が複雑になり、仕様変更などが難しくなるかと思います。
これを解決するのが[speedcontrol-additions](https://github.com/cma2819/speedcontrol-additions)というbundleです。
Speedcontrolをインストールした状態でこのbundleをインストールすると、Speedcontrolで用意した各ゲーム情報にプレイヤーのSNSアカウント、解説者の名前とSNSアカウントをDashboard上から追加することができるようになります。
追加したデータはspeedcontrol-additions側のReplicantに出力されるので、Speedcontrol側のReplicantと適合するデータを照らし合わせながらレイアウトに表示する流れとなります。

ただしこのbundleを使用するには現状Speedcontrolのカスタマイズが必要であり、具体的にはゲーム/プレイヤー情報(`runDataArray`)にユニークID(`externalID`)のデータ列を追加する必要があります。([コード例1](https://github.com/cma2819/nodecg-speedcontrol/commit/4f026e884e9e7988a63be794f1a6ac3d4e16f4ce),[2](https://github.com/yagamuu/nodecg-speedcontrol/commit/7ea8b25be80e2d30a399cde274fcf1b441c0689a))
理由としては、プレイヤーのSNSアカウントを追加するには各ゲームのプレイヤーに紐づけを行うためユニークなデータが必要ですが、現状のSpeedcontrolにはそれが存在しないからです。^[ゲーム情報そのものにはユニークIDがある(`RunData.externalID`)ものの、プレイヤーを管理する`RunData.teams.players`にユニークID列が存在しない]
そのためSpeedcontrolを自分でforkしたものに上記のカスタマイズを行うか、よくわからない方は既に上記カスタマイズが行われたSpeedcontrolを使用すると良いでしょう。自分が確認している限りでは上記カスタマイズに対応しているのは以下2つのforkです。
- [yagamuu/nodecg-speedcontrol(branch:mysrtafes2021)](https://github.com/yagamuu/nodecg-speedcontrol/tree/mysrtafes2021)
- [cma2819/nodecg-speedcontrol](https://github.com/cma2819/nodecg-speedcontrol)

余談ですが1つ目の方はおまけでイベント中のチェックリスト機能が含まれています。[Speedcontrol本家にもプルリクを出しているので](https://github.com/speedcontrol/nodecg-speedcontrol/pull/94)そのうち取り入れられるかも？

## 5. 外部Webサービスとの連携
今回のレイアウト実装においてはさらに以下2つのbundleを使用しています。
- [cma2819/nodecg-twitter-widget](https://github.com/cma2819/nodecg-twitter-widget)
- [cma2819/nodecg-spotify-widget](https://github.com/cma2819/nodecg-spotify-widget)

nodecg-twitter-widgetはTwitterから任意のワードを検索しヒットしたツイート情報をReplicantに出力するbundleで、nodecg-spotify-widgetはダッシュボードからログインを行ったSpotifyアカウントの現在再生している音楽の情報をReplicantに出力するbundleです。
ツイート表示もBGMの情報表示も1から実装しようとする場合、各種APIへのアクセス周りなどから実装する必要があり面倒なのでこれらbundleの存在は非常にありがたかったです。

今回のレイアウト開発では上記2つのbundle並びにspeedcontrol-additionsの開発者である[Cmaさん](https://twitter.com/cma2819)に、こちらの質問に快く回答を頂いたり各種bundleのアップデートなど多くのサポートを頂きました。この場をお借りし改めてお礼を申し上げます。

# その3: 実装
使用する技術の選定も終わりようやくコードを書き始めるフェーズまで来ました。
とはいえ、筆者はNodeCG bundleの開発を行った経験は過去に一度しかなく経験不足なので自分のも含め既存bundleの実装を参考にすることにしました。
Speedcontrol+αを活用した配信レイアウトはCmaさんが[Online Marathon Eventers](https://portal.rtamarathon.online/)という団体を通してたくさんリリースを行っており、そしてそれらのソースコードは基本全て公開されていたので、その中から直近で開催されていた[Puzzle Game RTA Festivalのソースコード](https://github.com/cma2819/nodecg-pgrf)を~~パクリ~~めちゃくちゃ参考にさせて頂きました。
ディレクトリ構造からファイル名の命名、各種コンポーネントの実装など多分見比べるとかなり酷似してると思います、すみません……

あとは既存bundleを参考にひたすら要件通りにコードを書くだけです。
今回実装を行ったコードの内容を簡潔にまとめていきます。

## 1.Dashboard、Extension
必要な機能のほとんどはSpeedcontrolを始めとした多くのbundleを使用することで用意できましたが要件に含まれた機能の内1つがまだ足りません。
それがゲームの待機画面下部にある運営のお知らせ表示機能です。

![](https://storage.googleapis.com/zenn-user-upload/32yklihsg6gno9renfp6mopaphqk)
*下の青い文字が運営からのお知らせ*

別bundleに分けるほど汎用的な機能でも無いかなと思ったので今回のレイアウトbundleに含めることにしました。
この機能はダッシュボードから入力した各お知らせを順番に1つずつ表示するだけのかなりシンプルな機能で、特に複雑なデータ構造もしていないのでテンプレートに含まれたVuetifyを使いつつサクッとDashboard、Extensionのコードを実装しました。
![](https://storage.googleapis.com/zenn-user-upload/uhegp1mcowg7z7ziewrkewnepu21)
*ただのCRUDだけ*

Replicantへの反映はDashboardから`message`を飛ばしてExtensionが感知したら更新という普通の実装です。今回に限らずReplicantの更新はExtensionに集約させて他からは参照のみにするとデータフローが簡略になるのでオススメです。
```js:src/extension/information.ts
// 元のコードからある程度簡略化しています
import { v4 as uuid } from 'uuid';
import { get as nodecg } from '@/util/nodecg';
import { SetupInformation, SetupInformationArray } from '@/types/schemas/setupInformationArray';

const informationArrayRep = nodecg().Replicant<SetupInformationArray>('setupInformationArray', {
  defaultValue: [],
});

const createInformation = (text: string): void => {
  if (!informationArrayRep.value) { return; }

  informationArrayRep.value.push({
    id: uuid(),
    text,
  });
};

const updateInformation = (information: SetupInformation): void => {
  if (!informationArrayRep.value) { return; }

  const informationIndex = informationArrayRep.value.findIndex(
    (informationRep) => informationRep.id === information.id,
  );

  if (informationIndex < -1) { return; }

  informationArrayRep.value[informationIndex] = information;
};

const deleteInformation = (id: string): void => {
  if (!informationArrayRep.value) { return; }

  informationArrayRep.value = informationArrayRep.value.filter(
    (information) => information.id !== id,
  );
};

nodecg().listenFor('createInformation', (data, ack) => {
  createInformation(data.text);
});

nodecg().listenFor('updateInformation', (data, ack) => {
  updateInformation(data);
});

nodecg().listenFor('deleteInformation', (data, ack) => {
  deleteInformation(data.id);
});
```

## 2. Graphics
### 2-1.Store
各bundleのDashboard、Extensionから出力されたReplicantを参照するStoreをVuexを通じて実装します。
使用したテンプレートに沿ってReplicantの参照は`src/browser_shared/replicant_store.ts`へ、実際に表示するデータ構造への加工は`src/graphics/store/*.ts`で行っています。

### 2-2.各種レイアウトと必要なコンポーネント
今回のモジュールバンドラーの設定を使用すると各ディレクトリ内の`main.ts`を通して最終的なHTMLファイル(とJS,CSSファイル)が出力されるので必要な画面の数だけディレクトリを分け`main.ts`を用意し、`compornents`ディレクトリに用意した各画面で使われる共通のコンポーネントを取得して描画します。
NodeCGで実装した配信レイアウトは基本的にOBS内に同梱されたChromeで動作するブラウザソース上から読み込むので、特にブラウザや機種差を考慮する必要はありません。
そのため各ページの実装はフルHD解像度のページを用意しその中に表示する各種コンポーネントを`position: absolute`で座標指定し出力するという簡潔な方法で実装しています。

さらに各種ゲーム用レイアウトではCanvasのfillRectメソッドを使用し、OBSで読み込んだ際にゲーム画面を合成する部分が透明になるようにしています。
こうした上でOBS側で配信レイアウトを最前面に置くことで、ゲーム画面が配信レイアウトの無関係な部分を侵食しないようにできます。

```js:src/graphics/components/ClipCanvas.vue
<template>
  <canvas
    width="1920"
    height="1080"
  />
</template>

<script lang="ts">
import { Vue, Component, Prop } from 'vue-property-decorator';
import { ClipPath } from '@/types/ClipPath';
import background from '../images/gameLayout/background.png';

@Component
export default class ClipCanvas extends Vue {
  @Prop(Array)
  readonly clipPaths!: ClipPath[];// ゲーム画面となる部分の座標とサイズ

  ctx?: CanvasRenderingContext2D;
  backgroundImage = background;

  mounted(): void {
    const element = this.$el as HTMLCanvasElement;
    const ctx = element.getContext('2d');
    if (ctx) {
      this.ctx = ctx;
    }
    const bg = new Image();
    bg.src = this.backgroundImage;
    bg.addEventListener('load', () => {
      this.draw(bg);
    });
  }
  draw(bg: HTMLImageElement): void {
    if (!this.ctx) {
      return;
    }
    // Canvas上に背景画像を描画
    this.ctx.drawImage(bg, 0, 0);
    // 合成種別をxorにすることで、ゲーム画面となる四角形と重なった箇所は透明に
    this.ctx.globalCompositeOperation = 'xor';
    // ゲーム画面と同じ座標、サイズの四角形を描画
    this.clipPaths?.forEach((clipPath) => {
      this.ctx?.fillRect(
        clipPath.x,
        clipPath.y,
        clipPath.width,
        clipPath.height,
      );
    });
  }
}
</script>
```

![](https://storage.googleapis.com/zenn-user-upload/qu1qgd29was4k3krnm622j7frvdm)
*このようにゲーム部分は透明となりレイアウトの下にゲーム画面を置ける*

以上が今回実装を行ったコードのざっとした紹介です。
他にもし記事を読んだ方で疑問や実装について詳しく知りたい部分などありましたら後で追記しようかと思います。

これだけだと内容として薄いかなと思い以下おまけで実装にあたって苦労した話を思い出せる範囲で書きます。

## 1. プレイヤー解説の名前+SNSアカウント表示
不思議RTAフェス2のゲーム画面レイアウトでは、4:3比率のゲームをプレイヤー3名が行うレイアウトを除き、プレイヤーと解説者の名前とSNSアカウントを横に並べて表示するデザインを採用しています。
![](https://storage.googleapis.com/zenn-user-upload/sbxbswhk5ekes8r32cdg5k8dkm0o)
*16:9比率のゲームのプレイヤー3名レイアウト*
一見なんとも無い構成に見えますが、名前あるいはSNSアカウント名の文字数が多い場合、特に今回のようにその2つを横に並べるケースでは容易にレイアウト崩れが発生しやすいです。
適当に実装をすると文字数が一定以上を超えると文字がコンポーネントをはみ出てしまうか、改行が発生してしまいデザインが崩れてしまいます。

![](https://storage.googleapis.com/zenn-user-upload/3s2m2u18quh7r8wvkajs78l8slni)
*文字列が長すぎてはみ出てしまう例*

最大文字数がデザインの要件定義で決まってはいたので運用でカバーすることもできますが、イベント中に想定外の事情で最大文字数を超えるデータを入力するケースも考えられますし、実用性を考慮するとある程度多くの文字を入力されてもデザインが壊れない実装を目指すべきだと思いました。

試行錯誤した結果、名前とSNSアカウントの表示幅を個別に固定幅にした上で、各文字列が指定幅を越えそうになったらフォントサイズを動的に小さくするという実装にしました。
上記のフォントサイズを可変で変動するというロジックは、[RTA in Japanのレイアウトに実装されていた](https://github.com/RTAinJapan/rtainjapan-layouts/tree/master/src/browser/graphics/components/lib/nameplate)ので、こちらを参考に[実装しました](https://github.com/yagamuu/mysrtafes2021-layouts/blob/master/src/graphics/components/OneLineTextBlock.vue)。

![](https://storage.googleapis.com/zenn-user-upload/njtacjmym7rflkkmd9i0orhzcpyu)
*名前とSNSアカウントのフォントサイズが可変となり、はみ出なくなりました*

名前とSNSアカウントの表示幅を一括ではなく個別に固定幅にしている理由ですが、これはSNSアカウント表示の仕様の都合です。
今回のレイアウトは各SNSアカウントを一定時間ごとに切り替えて表示するという仕様でしたが、これをフォントサイズ可変のロジックを入れつつそのまま実装すると、SNSアカウント切替時に各文字列のフォントサイズや表示位置が変動し、名前文字列がガクッと移動してしまい見た目が悪くなってしまいます。そのため名前とSNSアカウントを個別に表示幅を決めることでSNSアカウントが切り替わっても名前文字列の表示位置、フォントサイズは固定のままとなるようにし違和感を無くしています。

実を言うとあまりにも長い文字列を入力するとこれでもなお崩れますが、運用上起こり得ないだろう……ということで無視しています。
![](https://storage.googleapis.com/zenn-user-upload/yw4il5y4otp8fc800x6ut4au7xkp)
*長すぎてはみ出てしまった例、打ち込んだテキストは最長ゲームタイトルとして有名な[アレ](https://ja.wikipedia.org/wiki/%E5%A4%8F%E8%89%B2%E3%83%8F%E3%82%A4%E3%82%B9%E3%82%AF%E3%83%AB%E2%98%85%E9%9D%92%E6%98%A5%E7%99%BD%E6%9B%B8)です*

この可変サイズの文字列表示問題、どの配信レイアウトでも発生しうる話だと思うのですが、他の皆さんはどう対応しているのか気になります。もっと良い解決策がありましたら教えてほしいです……。

## 2. Twitterのツイート表示
不思議RTAフェス2の配信レイアウトでは、指定のハッシュタグを付けた特定のツイートをレイアウト上に表示する機能が存在します。(先程触れたnodecg-twitter-widgetを活用しています)
![](https://storage.googleapis.com/zenn-user-upload/cxqzgvwl8n5ku7gdcmujg4y6vzaz)
*ツイート表示、待機画面を含めた全ての画面に存在します*

ダッシュボードから表示するツイートを選択すると、画面外から表示ブロックが降りてくるアニメーションとともに選択したツイートを表示するというものなのですが、このアニメーションを付けるのに無駄に苦労してしまいました……。

Vueには要素の追加/削除に反応して自動的に指定のCSS transitionを適用してくれる機能が存在するので、こちらを使用しつつ目的のアニメーションを実現するCSSを適用すれば基本的な動作は簡単に実装できます。

```html:参考例(Vue)
<template>
  <transition name="twitter">
    <twitter-text-box v-if="activeTweetReplicant"/>
  </transition>
</template>
```

```css:参考例(CSS)
.twitter-enter-active {
  transition: all 1s;
}

.twitter-leave-active {
  transition: all 1s;
}

.twitter-enter, .twitter-leave-to {
  transform: translateY(-100%);
}
```

トランジションの開始、終了部分に`transform: translateY(-100%);`を適用すればツイート表示コンポーネントの表示位置上部から初期位置までトランジションします。
しかし、ゲーム画面レイアウトでは表示部分の上にイベントで使用するTwitterハッシュタグを表す画像が存在し、この画像は背景が透過されているので、先程のCSSをそのまま適用してしまうとアニメーション開始時にハッシュタグ画像の背景下からいきなりツイート表示コンポーネントが現れてしまい見栄えが悪いです。

![](https://storage.googleapis.com/zenn-user-upload/y59kopz52y09q80qymtbmn6ipcti)
*なんかちょっと残念*

ところでゲーム用レイアウトの背景はイベントが始まってから2日目の途中辺りまではただの単色でした。
そのため、最初はハッシュタグ画像の背景にレイアウト背景と同じ背景色を適用することで画像背景下に現れるツイート表示コンポーネントを見えないようにしてごまかしていました。
![](https://storage.googleapis.com/zenn-user-upload/z9l026pwu4s7i1k49mk7jq8l4wxk)
*初期レイアウトのツイート表示、再現するのが面倒だったのでイベントのアーカイブ動画から拝借*

しかし、イベント開始直前辺りにポンズさんから「背景単色だと殺風景すぎるので背景画像を別途付けたい」という至極当然な依頼が飛んできたので、いよいよもっとこのやっつけな対策法は取れなくなりました😇

そのため今までのハッシュタグ画像下からではなくレイアウトの画面外から表示位置までトランジションさせるようにしたいと思いましたが、ここで一つ問題が発生します。
画面外から初期位置までトランジションするにはツイート表示コンポーネントが画面外に見切れる座標をCSSで指定すれば実現できそうですが、このツイート表示コンポーネントの要素幅は各レイアウト毎にバラバラな為、画面外に見切れる座標はレイアウト毎に変動してしまい同じCSSを使い回すことでは解決できません。

![](https://storage.googleapis.com/zenn-user-upload/90spy48jr0dukqo0ch4jljr6e7sn)
*各レイアウトのツイート表示コンポーネント一覧、要素幅が見事にバラバラです*

トランジションのCSSはVueの単一ファイルコンポーネント内の`<style>`ブロックに書いてましたが、このブロック内のCSSプロパティを外部から変化させる方法は恐らく存在しないので(あったらごめんなさい)、トランジションのCSSはVue側で与えるようにしよう、と考え試行錯誤しましたがどうにも上手く行かず……詰んでしまいました。(各レイアウト別に専用のツイート表示コンポーネントを用意するというゴリ押しは……やりたくなかったです)

悩んだ挙げ句最終的には、トランジションの開始/終了時は透明にして見えなくし(`opacity: 0;`)、大半のレイアウトでツイート表示コンポーネントが画面外に行くであろう位置を適当に指定し(`transform: translateY(-130%);`)ごまかすという妥協策を取りました。正直自分で書いてて恥ずかしくなる残念な対応なので何か良い感じの解決策あったら教えて下さい（他力本願）

改めて振り返って見るとほぼCSS周りでつまづいてますね、実務でめったに触ってないとは言えもっと精進すべきですね……😇

# その4: サーバー構築
色々つまづきながらもなんとかレイアウトが完成したので、最後に外部からNodeCGの操作を行えるようにするためにサーバーへの公開作業を行います。

最初に利用するサーバーサービスを選ぶ必要があります。以前は無難にAWSのEC2を選択していましたが今回は[Vultr](https://www.vultr.com/)というクラウド型VPSサービスを選択しました。
Vultrは少し前から話題になっていた海外発のサービスなのですが、以下の理由から今回採用しました。

1. 利用価格が安い
    最低プランは価格が月2.5ドル(1時間0.005ドル)、特にNVMe SSDを採用した高速プランは最低価格月6ドル(1時間0.009ドル)であり魅力的
2. 日本リージョンがある
    海外発のVPSとしては珍しく日本リージョンがある、他リージョンとの価格差もなし
3. 時間単位の課金プランがある
    RTAイベントでは特定の期間しか動かす必要がないので利用価格を安価に抑えることができる
4. スナップショット機能が無料
    サーバーの状態を丸々保存できる機能が無料で使え、気軽にサーバーの構築/破棄ができる

特に3と4はRTAイベントでの用途としては魅力的に感じます。必要な時に用意したスナップショットからサーバーを立ち上げ、イベントが終了したらスナップショットを取って破棄すれば次回以降にそのまま使い回せる上に利用価格をぐっと抑えられます。
さらにVultrはLinux系OSだけではなくWindowsサーバーも利用可能なので、Vultr上でOBS立ち上げて配信……みたいなことも可能かもしれませんね（未検証）

立ち上げたサーバーの初期状態はセキュリティ的には丸裸な状態なので、ある程度きちんとした設定を行う事を推奨しますが、インフラ周りは疎いので以下の記事を参考に設定しました。

- [海外VPS「VULTR」でしておくべき「初期設定」の内容を紹介](https://vector-ium.com/vultr-setting/)

あとはローカル環境の構築を行ったようにNode.jsをインストールし、nodecg-cliをインストールして……と作業を進めていきますが、新規でサーバーを借りるたびにこの手順を行うのも面倒だし手順も忘れてしまいかねないので、次回以降はDockerをサーバー内に導入して起動した方が楽かな……と思いました。
ちなみにDockerを使用せずにNodeCGを起動する場合は[pm2](https://pm2.keymetrics.io/)というNode.jsアプリのプロセス管理ツールがオススメです。NodeCGの公式サイト内でも[推奨されています](https://www.nodecg.dev/docs/creating-bundles)。

サーバーへのNodeCG構築が完了したらあとはサーバーを公開するだけです。NodeCGを起動したらそのままサーバーのIPアドレス(+初期設定なら9090ポート)を叩けば外部からアクセスできますが、今回は一応多少なりのセキュリティ面も考慮しつつURLもわかりやすくなるのでドメインを取得しつつSSL接続できるようにしました。

ドメインは今回初めて取得したのですが、Google Domainで取得しました。多分どこで取得しても良いとは思うのですが、UIの扱いやすさやGoogleのブランド力に惹かれて選びました。

次にSSL接続を実現するために証明書を発行し、サーバーに設置する必要がありますがこちらは無料でSSL証明書を発行できることでおなじみ`Let's Encrypt`で発行しました。
手順を簡潔に説明すると、以下のサイトを参考に`certbot`をインストール、`--standalone`オプションを指定しつつコマンド実行をして証明書発行を行い最後にNodeCGのconfigファイルを編集して作業完了です。
- [Let's Encrypt 総合ポータル](https://free-ssl.jp/)

```json:cfg/nodecg.json(設定例)
{
  "host": "example.com",
  "port": 443,
  "ssl": {
    "enabled": true,
    "keyPath": "/etc/letsencrypt/live/example.com/privkey.pem",
    "certificatePath": "/etc/letsencrypt/live/example.com/fullchain.pem"
  }
}
```

最初はおまけ程度の対応でしたが後々SpeedcontrolのTwitch連携を行うのに必要な、Twitch APIアプリケーション設定の「OAuthのリダイレクトURL」がいつの間にhttpsの指定が必須となっていたので結果としては対応して正解でした。

# まとめ
以上、簡潔にと言いながらもなかなかの長文となってしまいましたが、配信レイアウトの開発工程を一通り記してみたつもりなのですがいかがだったでしょうか。
わかる人には蛇足な話が多かったようにも思えますし、初心者の方からすると情報不足にも思えますしなんだか中途半端な立ち位置の記事になってしまった気がしてならないです……😇

実装の段落でも書きましたが、もしもっとこういう所の話をして欲しいなどありましたら別記事で公開するか、この記事に追記致しますので気軽に声を掛けて頂ければと思います！

もし実際にNodeCGのbundle実装に挑戦していて困ったことなどありましたら、気軽に公式Discordサーバー等で質問して頂ければと思います。有識者の方々から丁寧に回答して頂けるはずです(自分自身とてもお世話になってます)。日本語のチャンネルもありますので英語が苦手な方も安心！

以上で今回の記事は終わりです。少しでも当記事が誰かの参考になれれば幸いです。

## 参考リンク
- [NodeCG 公式サイト](https://www.nodecg.dev/)
- [NodeCG 公式Discordサーバー](https://discord.com/invite/GJ4r8a8)
- [第2回不思議のダンジョンRTAフェスのレイアウトリポジトリ](https://github.com/yagamuu/mysrtafes2021-layouts)