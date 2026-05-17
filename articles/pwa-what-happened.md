---
title: "PWAは失敗したのか — 理想と現実、そして受け継がれたもの"
emoji: "🌐"
type: "tech"
topics: ["PWA", "WebAssembly", "フロントエンド", "モバイル"]
published: false
---

## はじめに

2018年頃、「PWAがアプリを置き換える」と言われていました。
App Storeは不要になる。URLだけでアプリが配布される。

では2026年、その未来は実現したのでしょうか。

PWA（Progressive Web App）という言葉を日常的に使っているエンジニアは、今や多くないかもしれません。しかし実は、PWAの思想そのものは現在のWebに深く残っています。本記事ではその現在地を振り返ります。

## PWAとは

一言で言えば「URLで配布できるネイティブライクなアプリ」です。2015年にGoogleのAlex RussellとデザイナーのFrances Berrimanが共同で提唱した概念で、以下の3要素が核心でした。

- Service Workerによるオフライン対応・バックグラウンド処理
- Web App Manifestによるホーム画面へのインストール
- プッシュ通知による再エンゲージメント

ストアの審査なし、インストール不要、単一コードベースでクロスプラットフォーム。理念として、これほど筋のいい提案はありませんでした。

## なぜ"主流"になれなかったのか

### Appleという障壁

iOSのPWAサポートは長年、不完全なまま放置されていました。プッシュ通知はiOS 16.4（2023年）まで未対応、バックグラウンド同期は2026年現在も非対応です。App Storeという収益モデルを守る戦略的な判断と見る向きもあります。

この姿勢が最も鮮明になったのが2024年2月の出来事です。AppleはEUのデジタル市場法（DMA）対応を名目に、iOS 17.4のベータからEUユーザー向けのPWAサポートをひっそりと取り除きました。開発者コミュニティと欧州委員会からの激しい反発を受け、[約3週間後に撤回](https://techcrunch.com/2024/03/01/apple-reverses-decision-about-blocking-web-apps-on-iphones-in-the-eu/)しましたが、このエピソードはAppleのPWAに対するスタンスを象徴するものとして広く語られることになりました。その後、2025年4月にEU委員会はAppleをDMA違反として[€500Mの罰金を科しています](https://www.techpolicy.press/understanding-the-apple-and-meta-noncompliance-decisions-under-the-digital-markets-act/)。

日本はiOSシェアが高い特異な市場です（測定方法によって差はありますが、[主要調査で概ね60%前後](https://www.accio.com/business/apple-iphone-market-share-japan-2025-trend)）。Appleの壁が最も効く市場で開発をしているため、PWAの限界を肌で感じやすい構造になっています。

### Google Project Fuguの限界

WebからネイティブAPIの領域へ拡張しようとしたProject Fuguは構想としては面白かったのですが、ChromiumエコシステムのみでSafari・Firefoxが実装しなかったAPIは多く、「Chromeでしか動かないWeb機能」という難しさを抱えることになりました。クロスブラウザで動作しない機能は、プロダクションで採用しにくい。

### ネイティブ側が想定外に進化した

PWAが停滞している間に、React Native・Flutter・Capacitorといった技術が成熟し「WebコードをネイティブにWrapする」第三の道が現実解になりました。「PWAかネイティブか」という二択が崩れたことで、PWAの相対的な優位性が薄れました。

### UXの構造的な天井

PWAはJavaScriptのシングルスレッド構造上、アニメーションの描画とネットワーク通信・ロジック処理が同じスレッドで競合しやすい構造です。ネイティブアプリがOSレベルのAPIを使い60fps/120fps描画を保証できるのに対し、PWAはブラウザのレンダリングエンジンに依存するため、処理が重なった瞬間にフレームが詰まって画面がカクつきやすく、特に低〜中スペック端末でのビジュアル品質がネイティブを下回りやすいことが指摘されています。

ブラウザベースの大規模アプリでは、描画と処理負荷が競合した際に一瞬の引っかかりが体感されることがあります。これは実装の問題というよりも構造的な課題です。

### ビジネス判断の壁

技術的に正しくても、プロダクト判断の場では別の話になります。エンジニアが「PWAで作れる」と判断しても、PdMやデザイナーが「ユーザーはApp Storeを期待している」と言えば通りません。特に日本のコンシューマー向けプロダクトでは、この構造がPWA採用を阻み続けました。

| ユーザーが求めるもの | PWAの現実 |
|---|---|
| iOSプッシュ通知 | 2023年まで不可（EUでは一時削除騒動も） |
| 決済の自然なUX | ネイティブが優位 |
| App Storeでの発見 | PWAには検索面がない |
| スムーズなアニメーション | ネイティブに及ばないケースがある |

**PWAは「ネイティブアプリの代替」にはなれませんでした。しかし「Webをアプリ化する基盤」としては定着しました。** そしてその思想は、後続技術へと受け継がれていきます。

## それでもPWAは生きている

### 新興国市場での存在感

端末スペックが低く通信費が高い市場では、PWAの本質的な強みが生きています。

- **[Flipkart（インド最大EC）](https://developers.google.com/web/showcase/2016/flipkart)**：PWA（Flipkart Lite）移行後、ホーム画面追加経由のユーザーのコンバージョン率が70%向上。ユーザーの63%が2Gネットワーク経由でアクセスしており、ネイティブアプリ比でデータ使用量を3分の1に削減しています。
- **[AliExpress](https://web.dev/case-studies/aliexpress)**：PWA移行後にセッションあたりの訪問ページ数が2倍、セッション時間が74%増加、新規ユーザーのコンバージョン率が104%増加しました。
- **[Starbucks](https://nearform.com/work/starbucks-progressive-web-app/)**：PWAはiOSアプリ（148MB）と比較してわずか233KB、99.84%小さいサイズを実現し、日次のWeb注文ユーザー数を2倍に増やすことに成功しました。オフラインでのメニュー閲覧・カート操作にも対応しています。

### B2B / SaaS領域での活用

コンシューマー向けとは異なり、B2Bでは「インストール不要でセキュアに使える」はむしろ長所です。URLを共有するだけで使い始められ、社内端末へのアプリ配布管理が不要になるのは、IT管理者にとって実質的なメリットです。

FigmaやNotionのような「ブラウザファースト」のSaaS群が大きく普及したことは、PWA的な思想がB2B領域で静かに定着していることを示しています。

### 日本のエンジニアが体感とズレる理由

「PWAは流行らなかった」という感覚は正しくもあり、バイアスでもあります。調査機関や測定方法によって数値は異なりますが、日本はグローバル平均（[約29%](https://www.digitalapplied.com/blog/mobile-os-market-share-2026-ios-vs-android)）を大きく上回るiOSシェアを持つ市場です。Appleの壁が最も効く市場で開発をしているため、新興国での活用事例が実感として入ってきにくい構造になっています。

## 受け継がれた理想 — WebAssemblyという次章

PWA自体は配布モデルの提案でした。しかし「Webでネイティブ級の体験を作る」という期待も同時に背負っていました。その期待に、性能の面から応えようとしているのがWebAssembly（Wasm）です。

位置付けを明確にしておくと、**WasmはPWAの後継ではなく補完技術**です。PWAが「どう配布するか」を変えようとしたのに対し、Wasmは「ブラウザ上でどこまで速く動かせるか」を変えます。この二つは解こうとしている問題が異なります。

| | PWA | WebAssembly |
|---|---|---|
| 解こうとした問題 | 配布とオフライン | 性能の天井 |
| 手法 | Service Worker / Manifest | バイナリ実行形式 |
| 採用例 | Flipkart, Starbucksなど | Figma, Google Sheets, AutoCAD |
| 残課題 | iOS / API非対応 | エコシステム成熟途上 |

### Wasmが実際に変えたもの

**Figma：WebGPUへのさらなる進化（2023年〜）**

FigmaはC++で書かれたレンダリングエンジンをWasmで動かすことでロード時間を3倍以上高速化した実績を持ちます（[2018年の最適化フェーズで確認](https://www.figma.com/blog/figma-faster/)）。そこで終わらず、2023年にはChromiumがWebGPUをリリースしたタイミングで、FigmaはWasmベースのレンダラーをさらにWebGPU対応へと移行しました。[公式ブログ（2025年）](https://www.figma.com/blog/figma-rendering-powered-by-webgpu/)によると、WebGPU移行によりGPUへの処理移譲が進み、一部デバイスクラスで明確なパフォーマンス向上が確認されています。コンピュートシェーダーの活用やRenderBundlesによるCPUオーバーヘッド削減など、WebGL時代には不可能だった最適化が実現しつつあります。

**Google Sheets：WasmGCで計算速度を2倍に（2024年）**

より現代的な事例が、[2024年6月にGoogleが発表したGoogle SheetsのWasmGC移行](https://workspace.google.com/blog/sheets/new-innovations-in-google-sheets)です。計算エンジンをJavaScriptからWasmGC（WebAssembly Garbage Collection）へ移行した結果、Chrome・Edgeブラウザ上での計算速度が約2倍に向上しました。もとはJavaで書かれたサーバーサイドの計算エンジンをそのままWasmGCへコンパイルしたアプローチで、JavaScriptへの書き直しなしにブラウザ上でネイティブに近い速度を実現した点が技術的に注目されています。

これらの事例はどれも「ブラウザで動くとは思えなかった」カテゴリのアプリケーションです。Wasm採用が活きるのは、グラフィクスレンダリング・スプレッドシート計算・CAD処理のような、JavaScriptのシングルスレッド・JIT限界が直接ボトルネックになる領域です。

### Wasmの射程はブラウザを超えている

さらにWASI（WebAssembly System Interface）やComponent Modelの整備が進み、WasmはブラウザだけでなくEdge・サーバー・組み込みへと展開しつつあります。「どこでも動く、速い、サンドボックスで安全」というWasmの特性は、ブラウザ外でも価値を持ちます。[2026年時点でChrome訪問サイトの約5.5%がWasmを使用](https://www.matrixinternet.ie/webassembly-in-2026/)しており（2024年比で+1ポイント）、採用は静かに広がっています。

ただし、まだ整備しきれていない部分も多い。toolchainの複雑性、Component Modelの学習コスト、デバッグ体験の未成熟など、プロダクション採用のハードルは下がり切っていません。「URLで配布できる、インストール不要の高性能アプリ」——PWAが夢見た世界を、WasmがJavaScriptと組み合わさる形で別のレイヤーから実現できるかどうか、その答えはまだ出ていません。

## おわりに

PWAは、ネイティブアプリを置き換える存在にはなれませんでした。

しかし「WebをOffline-firstにする」「ブラウザをプラットフォームにする」という思想は、SaaS・新興国市場・そしてWebAssemblyという形で確実に受け継がれています。

技術の評価は、登場時の期待値ではなく、残したアーキテクチャと思想の射程距離で測るべきでしょう。その意味で、PWAの遺産は現在進行形です。

---

## 参考URL

| 記事内の事実 | URL |
|---|---|
| PWA命名者（Alex Russell + Frances Berriman, 2015） | https://en.wikipedia.org/wiki/Progressive_web_app |
| iOS PWA制限の現状（2026） | https://www.magicbell.com/blog/pwa-ios-limitations-safari-support-complete-guide |
| AppleのEU PWA削除→撤回（2024年2月） | https://techcrunch.com/2024/03/01/apple-reverses-decision-about-blocking-web-apps-on-iphones-in-the-eu/ |
| Apple EU €500M罰金（2025年4月） | https://www.techpolicy.press/understanding-the-apple-and-meta-noncompliance-decisions-under-the-digital-markets-act/ |
| 日本iOSシェア（~61%, 2025） | https://www.accio.com/business/apple-iphone-market-share-japan-2025-trend |
| グローバルiOSシェア（~29%, 2026） | https://www.digitalapplied.com/blog/mobile-os-market-share-2026-ios-vs-android |
| Flipkart PWA（70%コンバージョン向上、63%が2G、3x省データ） | https://developers.google.com/web/showcase/2016/flipkart |
| AliExpress PWA（104%コンバージョン増、2x PV、74%セッション時間増） | https://web.dev/case-studies/aliexpress |
| Starbucks PWA（99.84%小、日次注文2倍） | https://nearform.com/work/starbucks-progressive-web-app/ |
| Figma Wasm 3x改善（2018年実測、最適化フェーズ） | https://www.figma.com/blog/figma-faster/ |
| Figma WebGPU移行（2023年〜、2025年公式ブログ） | https://www.figma.com/blog/figma-rendering-powered-by-webgpu/ |
| Google Sheets WasmGC 2x高速化（2024年6月） | https://workspace.google.com/blog/sheets/new-innovations-in-google-sheets |
| Chrome上のWasm利用率5.5%（2026年） | https://www.matrixinternet.ie/webassembly-in-2026/ |