---
title: "個人開発者向け、RevenueCatの紹介。レシート検証と課金のまとめ方"
emoji: "💳"
type: "tech"
topics: ["revenuecat", "ios", "reactnative", "expo", "swift"]
published: true
---


## はじめに

ZEN大学で知能情報を学んでいる2年生の松延です。個人で iOS アプリを作っています。

アプリにサブスクリプションや買い切りの課金を入れると、レシートの検証、更新や解約の状態管理、iOS と Android で違う商品 ID の扱い、Paywall の表示まで、自分で持つ範囲が急に広がります。RevenueCat は、その課金まわりの基盤を SDK とバックエンドとして提供するサービスです。

この記事は、個人開発者向けの紹介です。料金、無料枠、SDK 名、コマンド、API は、いずれも RevenueCat の公式サイトとドキュメントで確認できた範囲だけを書いています。確認できなかったことは書いていません。コード例の API キーは、すべてプレースホルダです。特定のアプリでの利用体験や、収益の数字は扱っていません。

## RevenueCatとは何か

RevenueCat の公式ドキュメントでは、StoreKit、Google Play Billing、RevenueCat Billing を包んで、アプリ内課金とサブスクリプションの実装を簡単にするバックエンドと SDK だと説明されています。SDK を使うと、自分でアプリ内課金のインフラを維持しなくても、複数のプラットフォームで課金を扱えます。

公式の「Implementation Responsibilities」では、RevenueCat を iOS、Android、Web をまたいだサブスクリプション状態の単一の情報源（single source of truth）として位置づけています。レシートの検証や、Apple の StoreKit、Google の BillingClient といったネイティブの課金 API とのやりとりは、SDK 経由で自動的に処理されます。

一方で、RevenueCat は決済代行業者ではありません。料金ページの FAQ にも、「Does RevenueCat pay me?」への答えとして、支払いはこれまでどおり Apple、Google、Stripe など、連携した決済事業者から受け取ると書かれています。商品の作成や、ユーザーへの課金画面の提示、有料機能の解放は、開発者側の責任として残ります。

## 個人開発者にとって何が楽になるか

個人開発では、課金のサーバーを自分で立てて保守するコストが、本体の機能開発を圧迫しがちです。公式が挙げている範囲で、特に負担が軽くなりやすい点を整理します。

### レシート検証を自前のサーバーで持たなくてよい

購入後、SDK が購入トークンやレシートを RevenueCat のバックエンドへ送り、バックエンドがストア側で検証し、顧客の状態（`CustomerInfo`）を返します。公式の Implementation Responsibilities では、レシートの同期・解析・検証、Entitlement の状態追跡、購入履歴の追跡は RevenueCat の責任だと整理されています。

つまり、Apple や Google のサーバーサイド検証を自分で実装し、更新や失効のたびに状態を計算するサーバーを、最初から自分で持たなくてよくなります。ただし、有料コンテンツを自分のサーバーから配る場合は、クライアントの結果だけを信じず、後述する Webhook や REST API でサーバー側でも確認する必要があります。

### iOSとAndroidの課金をまとめて扱える

同じプロジェクトに複数のストア（Apple App Store、Google Play など）を接続できます。ユーザーの Entitlement は、同じプロジェクトに含まれるすべてのアプリで共有されます。アプリ側では、ストアごとの商品 ID を直接追い続ける代わりに、「pro が有効か」といった Entitlement を見て、機能の解放を判断できます。

ハイブリッド SDK（React Native や Flutter など）では、iOS と Android で別々の公開 API キーを使って SDK を初期化します。キーはダッシュボードの Project Settings > API keys > App specific keys にあります。

### Entitlement・Offering・Product の考え方

公式ドキュメントで繰り返し出てくる中心的な概念は、次の3つです。

| 概念 | 役割 |
| --- | --- |
| **Product** | ユーザーが実際に購入する個別の SKU。本番では App Store Connect や Google Play Console など、各ストアで価格や期間を設定し、RevenueCat へ取り込む |
| **Entitlement** | ユーザーが「利用する権利を持っている」機能やコンテンツの段階。購入した Product に紐づけて解除する。アプリコードでは商品 ID の一覧ではなく、この識別子を見る |
| **Offering** | Paywall に出す Product の集まり。中の **Package** が、iOS・Android・Web で同等の商品をまとめる |

多くのアプリでは Entitlement は1つ（たとえば `pro`）で足りると、公式は説明しています。Gold と Platinum のように段階がある場合は、Entitlement を分けます。サブスクリプションの Product は契約期間中だけ Entitlement を有効にし、非消耗型（および Entitlement に付けた消耗型）は、そのコンテンツを永続的に解除します。公式は、ゲームの追加ライフのような消耗型は、Entitlement に付けないことが多い、とも書いています。

Offerings の利用自体は任意ですが、Paywalls、Experiments、Targeting を使うために必要です。ダッシュボードで Default Offering を決め、SDK の `getOfferings` で返る `current` を参照して Paywall を組むと、アプリを更新しなくても、ダッシュボード側で出す商品の組み合わせを切り替えられます。公式は、Identifier をハードコードするより `current` を使うことを強く推奨しています。

開発中は、ストアに接続しなくても、RevenueCat 上の Test Store で Product を作って購入フローを試せます。本番用の商品は各ストアで作成し、ダッシュボードの Import Products で取り込むのが推奨手順です。ストア側で購入されたのに RevenueCat に未登録の Product は、取引の追跡とともに自動作成される、と公式には書かれています。

### Paywallをダッシュボードから差し替えられる

RevenueCat Paywalls は、コード変更やアプリ更新なしに、課金画面の見た目と出す商品を遠隔で設定する機能です。Paywall は Offering に紐づきます。ダッシュボード上でテンプレートから作る、ゼロから組む、AI Editor で生成する、Figma から取り込む、といった方法が公式に案内されています。単一画面だけでなく、オンボーディング向けの複数ステップの流れにもできます。

アプリ側では RevenueCat の UI SDK を入れ、対象の Offering に Paywall を作成したうえで表示します。iOS なら `RevenueCatUI`、React Native / Expo なら `react-native-purchases-ui` が、その UI コンポーネント用のパッケージです。デフォルトの Paywall には「購入の復元」ボタンが含まれるため、自前の Paywall を組む場合を除き、復元用のコードを自分で書く必要はありません。

制約もあります。公式の Paywalls ドキュメントによると、対応は iOS 15.0 以降、Android 7.0（API 24）以降、Mac Catalyst 15.0 以降、macOS 12.0 以降、および Web Purchase Links 経由の Web です。watchOS、tvOS、visionOS は対象外です。各 SDK には推奨バージョンがあり、複数ステップの Paywall など機能によっては、さらに新しいバージョンが必要です。掲載されているバージョンは変わることがあるので、導入時は公式の表を確認してください。

### ダッシュボードと指標

ダッシュボードの Overview では、次のカードが確認できます。

- Active Trials（追跡中の有効な無料トライアル数）
- Active Subscriptions（追跡中の有効な有料サブスクリプション数）
- MRR（月次経常収益）
- Revenue（直近 28 日間のグロス売上。ストア手数料や税を引く前）
- New Customers（直近 28 日間に作られた App User ID 数）
- Active Customers（直近 28 日間に RevenueCat と通信した App User ID 数）

公式によると、多くのアカウントではサブスクリプション系の指標はほぼリアルタイムに更新され、New Customers と Active Customers は 1〜2 時間キャッシュされることがあります。Sandbox と本番の切り替えもできます。ただし Charts に出るのは本番の取引データだけで、サンドボックス環境の制約のため、Charts には出ません。

Charts はクライアントのイベント送信に依存せず、RevenueCat に保存された購入データから作られます。Apple や Google、Stripe の数字と完全一致するとは限らない、と公式も注記しています。税務の申告には、RevenueCat ではなく決済事業者側のデータを使うよう案内されています。ダッシュボードの日時は、特に指定がなければ UTC です。

### Webhookで自分のサーバーへイベントを送れる

自分のバックエンドに、購入やサブスクリプションの状態変化を同期したいときは、Webhook を使います。ダッシュボードの Integrations から HTTPS のエンドポイントを登録し、RevenueCat が JSON を `POST` します。サーバーはステータス 200 を返す必要があり、それ以外は失敗とみなされます。失敗時は最大 5 回、指数バックオフで再送され、再送の累計はおよそ 80 分です。応答が 60 秒を超えると切断されるため、公式は先に 200 を返し、処理は後回しにすることを推奨しています。

用途の例として、公式は次を挙げています。

- 自前のバックエンドに、サブスクリプションと購入の最新状態を持つ
- サブスクリプションのライフサイクルに合わせた自動化
- 解約時のリマインドや、課金トラブルの通知

認証用の Authorization ヘッダを設定すること、必要なら HMAC 署名（`X-RevenueCat-Webhook-Signature`）を有効にすること、同じイベントが複数回届く前提で冪等に処理することが推奨されています。状態の同期では、各イベントの差分を独自に解釈するより、Webhook を受け取ったあとに REST API の `GET /subscribers` で顧客情報を取り直す方法が勧められています。

Webhook は Pro プランの機能です。現在のセルフサーブの標準プランは Pro で、後述するとおり一定の売上までは無料で使えます。レガシープランで Webhook が使えない場合は、新しい Pro への移行が案内されています。

## 料金体系

料金は [Pricing](https://www.revenuecat.com/pricing) と、アカウント管理のドキュメントに基づきます。確認時点の内容です。変更されている可能性があるので、導入前に公式の料金ページを見てください。

現在のセルフサーブは、機能を段階で分ける複数プランではなく、Pro が標準です。料金ページでは、機能一式が標準で含まれ、始めること自体は無料だと案内されています。

課金の基準は **Monthly Tracked Revenue（MTR）** です。MTR は、1 つの請求期間に RevenueCat が追跡した売上（USD）で、ストア手数料を引く前の金額です。顧客に請求された額であり、手元に残る額ではありません。有料サブスクリプション、更新、買い切り（ワンタイム購入）が含まれます。MRR（月次経常収益）とは別の指標です。請求期間はカレンダー月ではなく、ある月の同じ日から翌月の同じ日まで（例: 7月15日から8月15日）です。

Pro プランでの扱いです。

- MTR が **2,500 ドル未満**の月は無料
- MTR が **2,500 ドルを超えた**月は、その期間の MTR 全体の **1%** が請求される（しきい値を超えた差分だけではない）
- 公式の例: MTR が 2,500 ドルなら 25 ドル、2,600 ドルなら 26 ドル
- 翌月以降に 2,500 ドルを下回れば、再び無料に戻る

サインアップ時のクレジットカード登録は任意です。ただし、MTR が 2,500 ドルに達したときにカード情報がないと、一部機能が使えなくなります。利用開始直後の最初の月にすぐ上限を超えた場合は、達した時点から 30 日の猶予があります。それ以降の月に上限を超えた場合は、支払いが完了するまで機能が制限されます。制限される例として、Charts の閲覧とフィルタ、新しい Customer List の作成やエクスポート、Customer History の閲覧（Customer Details の閲覧は残る）、個別イベントの閲覧、新しい Paywall の作成、既存 Paywall の編集（既存 Paywall の利用は残る）などが挙げられています。

Enterprise の料金は、コミットメントや MTR、使う機能、導入支援などに応じて個別に決まります。この記事ではその金額には触れません。

## 導入の流れ

ここでは全体の流れと、公式で確認できた注意点を紹介します。画面のすべてを追う手順書ではなく、公式ドキュメントへの案内を兼ねています。

大まかな順番は次のとおりです。

1. RevenueCat のアカウントとプロジェクトを作る
2. App Store Connect（および Android なら Google Play Console）側を整える
3. RevenueCat にストアを接続し、Product・Entitlement・Offering を設定する
4. SDK を入れて、公開 API キーで初期化する
5. 必要なら Paywall をダッシュボードで作り、アプリから表示する
6. テストする。本番提出前に、Test Store 用の API キーから各プラットフォーム用のキーへ切り替える

### App Store Connect側の準備

RevenueCat を使っても、Apple 側の準備は必要です。公式の App Store Connect セットアップガイドに沿うと、次が前提になります。

- Apple Developer Program への登録
- App Store Connect の Agreements, Tax, and Banking で Paid Applications Agreement に同意する
- 銀行口座と税務情報を登録する
- Bundle ID を登録し、In-App Purchase の Capability を有効にする
- App Store Connect にアプリを作成する（Xcode の Bundle ID と一致させる）
- サブスクリプションを出す場合は、Subscription Group を作り、商品（Product ID、価格、期間、ローカライズ）を追加する

Sandbox での確認には、App Store Connect の Users and Access > Sandbox Testers でテスターを作ります。初回にサブスクリプションを出すときは、アプリのバージョン提出にアプリ内課金を含める必要があります。含めないと審査されない、と公式ガイドは強調しています。消耗型、非消耗型、自動更新サブスクリプション、非更新サブスクリプションは、それぞれ最初の1つをアプリバージョンと一緒に提出する必要があります。

RevenueCat のプロジェクトに Apple App Store を接続するときは、アプリ名、Bundle ID、Shared Secret、In-App Purchase Key を設定します。あわせて App Store Connect API キーを置くと、商品と価格を App Store Connect から取り込みやすくなります。

資格情報の役割は、公式では次のように整理されています。

- **App-Specific Shared Secret**: StoreKit 1 でレシート検証するときに必須。StoreKit 1 は Apple により非推奨で、新規アプリには StoreKit 2 が推奨されています
- **In-App Purchase Key**: StoreKit 2 と RevenueCat の Purchases v5.x 以降（iOS SDK 5.0.0 以降、React Native SDK 8.0.0 以降など）で必須。未設定だと取引が記録されず、ユーザーが購入した内容にアクセスできないことがあります。App Store Connect の Users and Access → Integrations → In-App Purchase で生成し、ダウンロードした `.p8` を RevenueCat にアップロードして Issuer ID を入力します。同じ App Store Connect アカウントのアプリで、同じキーを使えます
- **App Store Connect API Key**: 商品と価格のインポート用。Users and Access → Integrations → App Store Connect API で、少なくとも App Manager 権限のキーを作り、`AuthKey_XXXXXXXXXX.p8` という元のファイル名のままアップロードします。Issuer ID と、Payments and Financial Reports 左上の Vendor number も必要です。In-App Purchase Key（`SubscriptionKey_XXXXXXXXXX.p8`）とは別物です

Xcode では、Signing & Capabilities に In-App Purchase の Capability を追加します。Test Store だけで試す段階なら、この Capability は App Store に接続するときに戻って設定できます。

ストアを接続したあとは、Platform Server Notifications の設定が推奨されています。購入やサブスクリプションのイベントを早く RevenueCat に届け、Webhook や連携の遅延を減らすためです。

### SwiftでのSDK導入

iOS / Apple プラットフォーム向けの公式の導入方法は、Swift Package Manager が推奨です。CocoaPods と Carthage も、既存プロジェクト向けに案内されています。

Xcode の File > Add Package Dependencies で、次のリポジトリ URL を指定します。

```text
https://github.com/RevenueCat/purchases-ios-spm.git
```

Dependency Rule は Up to Next Major Version、起点は `5.0.0` です。パッケージ製品の RevenueCat と、Paywall などを使う場合は RevenueCatUI を、アプリのターゲットに追加します。導入の確認は、ソースに `import RevenueCat` を書いてビルドが通ることです。

初期化はアプリ起動の早い段階で、一度だけ行います。使うのは公開 API キーだけです。秘密 API キー（`sk_` で始まるもの）はアプリやウェブサイトに埋め込んではいけません。

```swift
import RevenueCat

// application(_:didFinishLaunchingWithOptions:) など、起動時の早い段階で一度だけ
Purchases.logLevel = .debug
Purchases.configure(
    withAPIKey: "<revenuecat_project_apple_api_key>",
    appUserID: "<app_user_id>"
)
```

`<revenuecat_project_apple_api_key>` と `<app_user_id>` はプレースホルダです。公開キーはダッシュボードの API keys から取得します。`appUserID` の扱い（匿名ユーザーと、自分たちのユーザー ID の紐づけ）は、公式の Identifying Users のガイドを見てください。

購読状態の確認は `getCustomerInfo` で、`entitlements` が有効かどうかを見ます。Offerings は `getOfferings` で取得します。

### React Native / ExpoでのSDK導入

React Native と Expo では、中核が `react-native-purchases`、Paywall や Customer Center などの UI が `react-native-purchases-ui` です。このパッケージには iOS 向けの RevenueCat SDK がすでに含まれるため、Xcode プロジェクトへ `purchases-ios` や `purchases-ios-spm` を別途追加してはいけません。重複やバージョン衝突の原因になります。

**React Native（Expo ではないプロジェクト）** の公式前提は、React Native 0.80 以降です。

```shell
npm install --save react-native-purchases react-native-purchases-ui
```

yarn の場合は `yarn add react-native-purchases react-native-purchases-ui` です。初回起動時に、React Native CLI が iOS の CocoaPods 依存関係を入れます。確認は、`import Purchases from 'react-native-purchases'` を書いた状態で `npx react-native run-ios` または `npx react-native run-android` が通ることです。App Store 向けに配布する段階で、Xcode の In-App Purchase Capability を有効にします。Test Store だけの確認なら、その設定は後回しにできます。

**Expo** の公式ガイドは、Expo SDK 54 以降を前提にしています。RevenueCat を使ってテストするには、Expo development build が必要です。

```sh
npx expo install expo-dev-client
npx expo install react-native-purchases react-native-purchases-ui
```

インストール後は、ホットリロードだけでは不十分で、公式が案内するフルビルドが必要です。ビルドしないと、`Invariant Violation: new NativeEventEmitter() requires a non-null argument.` のようなエラーになる、と書かれています。

初期化の例です。ハイブリッド SDK なので、プラットフォームごとに公開 API キーを分けます。値はすべてプレースホルダです。

```tsx
import { Platform } from 'react-native';
import { useEffect } from 'react';
import Purchases, { LOG_LEVEL } from 'react-native-purchases';

export default function App() {
  useEffect(() => {
    Purchases.setLogLevel(LOG_LEVEL.VERBOSE);

    if (Platform.OS === 'ios') {
      Purchases.configure({ apiKey: "<revenuecat_project_apple_api_key>" });
    } else if (Platform.OS === 'android') {
      Purchases.configure({ apiKey: "<revenuecat_project_google_api_key>" });
    }
  }, []);

  return null;
}
```

顧客情報の取得と、Entitlement の確認例です。

```tsx
try {
  const customerInfo = await Purchases.getCustomerInfo();

  if (typeof customerInfo.entitlements.active["<my_entitlement_identifier>"] !== "undefined") {
    // 対象の Entitlement が有効
  }
} catch (e) {
  // 取得に失敗
}
```

#### Expo Goでは本番相当の課金は動かない

公式の Expo ガイドに、Expo Go 向けの説明があります。Expo Go はカスタムネイティブコードを実行できないサンドボックスで、アプリ内課金に必要なネイティブモジュールはその対象です。一方、`react-native-purchases` には Expo Go 用の Preview API Mode があります。Expo Go 内ではネイティブ呼び出しが JavaScript のモックに置き換わり、アプリは落ちずにサブスクリプション関連のロジックを実行できます。ただし、このモードでは実際の購入は機能しません。

課金 UI のプレビューや、結合の流れを先に見る用途には使えます。アプリ内課金を実際に試し、RevenueCat の本番相当の機能を確認するには、development build が必要です。iOS シミュレータ向けには、公式ガイドどおり EAS で `ios-simulator` プロファイルを使ったビルドが案内されています。

補足として、React Native SDK のバージョン 12 では、iOS 依存関係に Swift Package Manager が必要になる予定です。公式によると、React Native / Expo での SPM 対応は現時点では実験的で、CocoaPods の手順は SDK 11 向けです。アップグレード前に、公式の移行案内を確認してください。

## 合わないケースや注意点

向いているかどうかは、課金の持ち方次第です。公式の記述から、次には注意した方がよいです。

- **決済そのものはストア側のままです。** RevenueCat は決済代行業者ではなく、Paid Applications Agreement や銀行・税務情報、商品作成、審査提出は、これまでどおり App Store Connect（や Google Play）で行います。RevenueCat だけ入れればストア側の準備が不要になる、というものではありません。
- **有料機能の解放はアプリ（または自前サーバー）の責任です。** RevenueCat は Entitlement の状態を返しますが、コンテンツのロック解除や、購入済みコンテンツのダウンロードは開発者側の仕事です。とくに有料コンテンツをサーバーから配る場合、公式はクライアントの結果だけを信じず、Webhook か REST API でサーバー側でも確認するよう案内しています。REST API の認証には秘密 API キーを使い、サーバーにだけ置きます。
- **Expo Go では実際の購入はできません。** Preview API Mode でロジックは動かせますが、確認には development build が必要です。
- **Paywalls はすべての Apple プラットフォーム向けではありません。** watchOS、tvOS、visionOS は公式に非対応です。それらの上に課金画面を出すなら、自前 UI など別の手段を検討することになります。
- **指標はストアのレポートと一致するとは限りません。** Charts は本番取引のみです。定義の違いや通貨換算の見積もりなどで差が出ることがあります。税務には決済事業者のデータを使います。
- **有料月の 1% は、ストア手数料を引く前の MTR に対する割合です。** 手元の手取りに対する 1% ではありません。2,500 ドル未満の月は無料で、超えた月だけ MTR 全体の 1% です。
- **あとから外すときは、自前の検証と状態管理が必要になります。** 公式のアカウント削除案内では、SDK を外し、必要なら自分で購入検証とサブスクリプション管理を用意したうえで、プロジェクトを削除します。プロジェクト削除は即時で取り消せません。古いアプリバージョンは RevenueCat のサービスにアクセスできなくなります。ただし、顧客の有効なサブスクリプション自体は、アカウント削除ではキャンセルされません。
- **提出前に API キーを間違えないでください。** Test Store で開発している場合、審査提出前に Test Store の API キーから、各プラットフォーム用の公開 API キーへ切り替える必要があります。
- **Product を Entitlement に付け忘れると、購入しても機能が解除されません。** 新しい商品を追加したときは、対応する Entitlement への紐づけを忘れないでください。消耗型を Entitlement に付けるかどうかも、用途に合わせて公式の考え方を確認した方がよいです。

自分で StoreKit や Google Play Billing を直接書き、検証サーバーも自前で持つ選択もあります。すでに安定した自前基盤がある場合や、RevenueCat に依存したくない場合は、無理に入れる必要はありません。個人開発で、レシート検証やクロスプラットフォームの状態管理、ダッシュボード、Paywall の遠隔更新までを最初から自分で持ちたくない、というときに検討するサービス、というのが紹介としての位置づけです。

## まとめ

RevenueCat は、アプリ内課金とサブスクリプションの検証・状態管理・分析を、SDK とバックエンドとして提供するサービスです。個人開発者にとっての要点は、次の通りです。

- レシート検証やサブスクリプション状態の追跡を、自前サーバーなしで始めやすい
- Entitlement を軸に、iOS と Android の商品をまとめて扱える
- Paywall や Charts、Webhook など、運用で必要になりやすい機能がダッシュボードに揃っている
- 確認時点では、MTR が月 2,500 ドル未満なら Pro の機能を無料で使え、超えた月は MTR（ストア手数料控除前）の 1%

一方で、ストア側の契約・商品登録・審査は残り、有料機能の解放も自分たちのアプリとサーバーの責任です。Expo Go では実際の購入はできず、development build が必要です。API キーは公開キーだけをアプリに入れ、秘密キーはサーバーに置いてください。

導入の詳細と、この記事のあとに変わりうる料金・バージョンは、次の公式ドキュメントを正とします。

## 参考リンク

- [RevenueCat Pricing](https://www.revenuecat.com/pricing)
- [Billing and account settings](https://www.revenuecat.com/docs/welcome/set-up-revenuecat/account-management)
- [What is RevenueCat / Expo ガイド内の説明](https://www.revenuecat.com/docs/getting-started/installation/expo)
- [SDK Quickstart](https://www.revenuecat.com/docs/getting-started/quickstart)
- [Configuring the SDK](https://www.revenuecat.com/docs/getting-started/configuring-sdk)
- [iOS & Apple Platforms（SDK 導入）](https://www.revenuecat.com/docs/getting-started/installation/ios)
- [React Native（SDK 導入）](https://www.revenuecat.com/docs/getting-started/installation/reactnative)
- [Expo（SDK 導入と Expo Go の注意）](https://www.revenuecat.com/docs/getting-started/installation/expo)
- [Implementation Responsibilities](https://www.revenuecat.com/docs/platform-resources/implementation-responsibilities)
- [Products Overview](https://www.revenuecat.com/docs/offerings/products-overview)
- [Entitlements](https://www.revenuecat.com/docs/getting-started/entitlements)
- [Offerings](https://www.revenuecat.com/docs/offerings/overview)
- [Paywalls](https://www.revenuecat.com/docs/tools/paywalls)
- [Creating Paywalls](https://www.revenuecat.com/docs/tools/paywalls/creating-paywalls)
- [Overview Metrics](https://www.revenuecat.com/docs/dashboard-and-metrics/overview)
- [Charts Overview](https://www.revenuecat.com/docs/dashboard-and-metrics/charts)
- [Webhooks](https://www.revenuecat.com/docs/integrations/webhooks)
- [API Keys / Authentication](https://www.revenuecat.com/docs/projects/authentication)
- [Connect Apps & Web Providers](https://www.revenuecat.com/docs/projects/connect-a-store)
- [App Store Connect Setup Guide](https://www.revenuecat.com/docs/platform-resources/apple-platform-resources/app-store-connect-setup-guide)
- [App Store Service Credentials](https://www.revenuecat.com/docs/store-configuration/app-store/service-credentials-index)
- [In-App Purchase Key Configuration](https://www.revenuecat.com/docs/service-credentials/itunesconnect-app-specific-shared-secret/in-app-purchase-key-configuration)
- [App-Specific Shared Secret](https://www.revenuecat.com/docs/service-credentials/itunesconnect-app-specific-shared-secret)
- [App Store Connect API Key Configuration](https://www.revenuecat.com/docs/service-credentials/itunesconnect-app-specific-shared-secret/app-store-connect-api-key-configuration)
- [Infrastructure（サーバーサイドのレシート検証の説明）](https://www.revenuecat.com/feature/infrastructure)
