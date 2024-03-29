---
title: "Flutter + FirebaseでiOSとAndroidの定期購入(サブスク)を実装する"
emoji: "🪙"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["flutter", "firebase", "cloudfunctions"]
published: true
---

>本記事は、以前(2021年8月)に書いた記事の再投稿になります。
>情報が古いことがありますので、お気をつけください。

## はじめに

先日公開したアプリでは定期購入を実装しています。いわゆるサブスクです。

Flutter(iOS/Android)でサブスクの実装例は少なく、苦労した点もあるので知見を公開します。定期購入の仕様について丁寧に解説するというよりは、実際に私がどのような実装をしているかについて書いています。

### 注意事項

- 本記事は公式ドキュメント、技術記事投稿サイト、個人ブログなどの情報を自分なりに解釈して実装した内容です。
- 課金周りはアップデートが多いため、常に最新情報の確認を推奨します。特に英語版公式ドキュメントは最も信頼できると思います。
- 本番環境で動作していることを確認済みですが、サブスクユーザーは現状数名です。サブスクユーザーが多数の場合による影響は未検証です。
- その他、十分に検証できていない内容があります。何かお気づきの点があればご指摘いただけると助かります。

### 環境

本記事執筆時の環境です。

Flutter 環境は下記です。

```sh
[✓] Flutter (Channel stable, 2.2.3, on macOS 11.4 20F71 darwin-x64, locale en-JP)
[✓] Android toolchain - develop for Android devices (Android SDK version 29.0.3)
[✓] Xcode - develop for iOS and macOS
[✓] Chrome - develop for the web
[✓] Android Studio (version 4.0)
[✓] VS Code (version 1.59.1)
[✓] Connected device (1 available)

• No issues found!
```

利用したライブラリ一覧 (関係のあるものだけ抜粋)

```yaml
dependencies:
  flutter:
    sdk: flutter

  # firebase
  firebase_core: ^1.5.0
  cloud_firestore: ^2.5.0
  cloud_functions: ^3.0.1
  firebase_auth: ^3.0.2
  firebase_remote_config: ^0.10.0+4
  firebase_analytics: ^8.3.0

  # billing
  # (注意 Androidについて) 今後 Billing Library バージョン 3以降が必須になります。
  # in_app_purchaseはバージョン0.5.0からBilling Library バージョン 3に対応しています。
  in_app_purchase: ^1.0.8
```

## 実装しなければいけないこと

サブスク機能を実現するためには、クライアント(Flutter アプリ) / サーバー両方の実装が必要です。

それぞれざっくりとした役割は以下になります。

### クライアントサイドの実装

- ストア情報の取得  
  GooglePlayStore / AppStore からストア情報を取得します。  
  主にサブスクの情報取得(金額など)を行います。

- サブスクの購入  
  アプリ内でサブスクの購入を検知します。  
  検知した購入情報はサーバーに渡し、正規のものであるか検証してもらいます。

- サブスク状況のチェック  
  ユーザーがサブスクユーザーであるか否かでアプリ内で処理を分けます。サブスクユーザーであるか否かはサーバーサイドで判断します。

- 購入情報のリストア  
  アプリの再インストールなどによりローカルデータが消えた状態からでも復帰可能にするボタンを用意します。  
  iOS では必須とされている機能です。

### サーバーサイドの実装

- 購入情報の検証  
  アプリから渡された購入情報が正規のものかを検証します。

- 購入情報の保存  
  検証済みの購入情報をデータベースに保存します。

- 購入情報の更新  
  サブスクの更新確認を行い、データベースを更新します。

## 事前準備

詳細は割愛しますが、予め各プラットフォームのストアからサブスクアイテムを作成しておいてください。

サブスクアイテム作成時に軽くハマったところだけ記載しておきます。

### アプリ内アイテム(課金アイテム)は削除できない

#### Android

基本的に一度アプリ内アイテムを作成したら削除することはできません。また、一度アプリ内アイテムを有効化したら無効化することはできません。

私の知る唯一の方法として「アプリを削除する」と課金アイテムもろとも削除することができますが、一度でも「Play Store を利用する配信」を行った場合はアプリを削除できません。  
なお「Play Store を利用する配信」には、 **内部テストやクローズドテストが含まれます。**

私はこれを知らず「後で一旦アプリを削除すればいいや」と考えていたので、結果的に現在は使っていない課金アイテムが 2 つほど存在しています。

#### iOS

削除することはできますが、同じ ID で再度作成することはできません。

### アプリ内アイテム ID の決め方

Android: アイテム ID  
iOS: 製品 ID

これらはアプリ内アイテムを一意に定める役割があります。管理上支障がなければなんでもいいと思いますが、私は下記のように決めました。

アプリケーション ID.アイテムのタイプ(サブスク / 消費型など).期間.アイテム名  
例) com.sample.subscription.monthly.premium

## フローと実装

具体的なケースごと、図とコードを交えて説明していきます。

### 1. ユーザーがサブスク購入ページを開いた

Flutter アプリが直接ストアに問い合わせてストア情報を取得します。

ストア情報はアプリ内で表示する価格などに利用します。こうすることでアプリ側で価格情報を持たなくて済むようになります。

例えば、グローバルに展開することになっても容易に現地の価格で表示することができます。

![](/images/b3dbc2824800456743d5/billing_flow1.png)

#### Flutter 側の実装

私は課金関係の処理を `ChangeNotifierProvider` として定義し、Widget ツリーの Root 近くに置いています。

なぜ ChangeNotifier なのかは後述します。

```dart
class BillingService extends StateNotifier<bool> {

  // ...

  final InAppPurchase _connection = InAppPurchase.instance;

  List<String> _productIds = <String>[
    Platform.isIOS
        ? 'com.example.iod.appname.pro' // それぞれのproductID
        : 'com.example.android.appname.pro'
  ];

  ProductDetails? _product; // 課金アイテムは1種類と仮定する

  InAppPurchase get connection => _connection;
  ProductDetails? get product => _product;

  /// ストア情報の取得を行う
  /// アプリ起動時や課金ページを開いた時に実行する
  Future _initStoreStatus() async {

    // ストア情報が有効か判断
    final bool isAvailable = await _connection.isAvailable();

    if (!isAvailable) {
      return;
    }

    // サブスクの商品情報を取得する
    ProductDetailsResponse productDetailResponse =
        await _connection.queryProductDetails(_productIds.toSet());

    if (productDetailResponse.error != null) {
      return;
    }

    if (productDetailResponse.productDetails.isEmpty) {
      return;
    }

    // 課金プランは1つと仮定した実装です
    if (productDetailResponse.productDetails.length == 1 &&
        productDetailResponse.productDetails.first.id == _productIds.first) {
      _product = productDetailResponse.productDetails.first;
    }
  }
}
```

`ProductDetails` からは価格など取得できるので、それを利用してアプリ画面上に反映します。

### 2. ユーザーがサブスク購入ボタンをタップした

「サブスク購入ボタンのタップ」 から 「課金の検証」 までを行います。

この処理からはサーバーサイドの実装が必要になります。

今回は表題の通り、サーバーサイドの実装に Firebase (Firestore / Cloud Functions for Firebase)を利用しています。

![](/images/b3dbc2824800456743d5/billing_flow2.png)

#### Flutter 側の実装

```dart
class BillingService extends StateNotifier<bool> {
  BillingService() : super(false) {
    // 課金処理を監視する
    Stream purchaseUpdated = _connection.purchaseStream;

    _subscription = purchaseUpdated.listen((purchaseDetailsList) {
      _listenToPurchaseUpdated(purchaseDetailsList);
    }, onDone: () {
      _subscription.cancel();
    }, onError: (error) {
    }) as StreamSubscription<List<PurchaseDetails>>;
  }

  late StreamSubscription<List<PurchaseDetails>> _subscription;
  final InAppPurchase _connection = InAppPurchase.instance;

  bool _isBillingUser = true;
  List<String> _productIds = <String>[
    Platform.isIOS
        ? 'com.sample.subscription.monthly'
        : 'com.sample.subscription.monthly'
  ];

  ProductDetails? _product; // 現在課金アイテムは1種類

  bool get isBillingUser => _isBillingUser;
  InAppPurchase get connection => _connection;
  ProductDetails? get product => _product;

  /// ストア情報の初期化を行う
  Future _initStoreStatus() async {
    // ...
  }

  /// サブスクの購入を実行する
  Future<void> buyNonConsumable() async {
    print("BUY NON CONSUMABLE");
    try {
      PurchaseParam purchaseParam = PurchaseParam(
        productDetails: product!,
        applicationUserName: null,
      );
      await _connection.buyNonConsumable(purchaseParam: purchaseParam);
    } catch (err) {
      // TODO
    }
  }

  /// CloudFunctions経由でレシート検証, 期限検証, (検証成功であれば)Firestoreへレシート登録を行う
  Future<int> _verifyPurchase(PurchaseDetails purchaseDetails) async {
    print("VERIFY PURCHASE");

    try {
      // iOSのレシート検証
      if (Platform.isIOS) {
        HttpsCallable verifyReceipt =
            FirebaseFunctions.instanceFor(region: 'asia-northeast1')
                .httpsCallable('verifyIos');
        final HttpsCallableResult result = await verifyReceipt.call(
            {'data': purchaseDetails.verificationData.localVerificationData});

        print("Verify Purchase RESULT: " + result.data.toString());
        return result.data[BillingStatus.result];
      }
      // Androidのレシート検証
      else if (Platform.isAndroid) {
        if (purchaseDetails is GooglePlayPurchaseDetails) {
          HttpsCallable verifyReceipt =
              FirebaseFunctions.instanceFor(region: 'asia-northeast1')
                  .httpsCallable('verifyAndroid');

          final HttpsCallableResult result = await verifyReceipt.call({
            'data': purchaseDetails.verificationData.localVerificationData,
            'signature': purchaseDetails.billingClientPurchase.signature
          });

          print("Verify Purchase RESULT: " + result.data.toString());
          return result.data[BillingStatus.result];
        }
      }
      return BillingStatus.UNEXPECTED_ERROR;
    } catch (_) {
      return BillingStatus.UNEXPECTED_ERROR;
    }
  }

  /// 購入処理のリスナー
  void _listenToPurchaseUpdated(
      List<PurchaseDetails> purchaseDetailsList) async {
    print("LISTEN TO PURCHASE UPDATED");

    if (purchaseDetailsList.isEmpty) {
      return;
    }

    purchaseDetailsList.forEach((PurchaseDetails purchaseDetails) async {

      // PurchaseStatus.pending
      if (purchaseDetails.status == PurchaseStatus.pending) {
        showPendingUI(true);
      } else {
        // PurchaseStatus.error
        if (purchaseDetails.status == PurchaseStatus.error) {
        }

        // PurchaseStatus.purchased
        else if (purchaseDetails.status == PurchaseStatus.purchased) {
          final result = await _verifyPurchase(purchaseDetails);
          if (result == BillingStatus.SUCCESS) {
            _isBillingUser = true;
          }
        }

        // PurchaseStatus.restored
        else if (purchaseDetails.status == PurchaseStatus.restored) {
          final result = await _verifyPurchase(purchaseDetails);
          if (result == BillingStatus.SUCCESS) {
            _isBillingUser = true;
          }
        }

        if (purchaseDetails.pendingCompletePurchase) {
          await _connection.completePurchase(purchaseDetails);
        }
        showPendingUI(false);
      }
    });
  }

  /// 画面をロックする
  void showPendingUI(bool pending) {
    state = pending;
  }
}

class BillingStatus {
  static const String result = 'result';
  static const SUCCESS = 0; // 成功 (期限内)
  static const EXPIRED = 1; // 期限切れ
  static const DOCUMENT_NOT_FOUND = 2; // Firestoreにドキュメントなし
  static const NO_AUTH = 3; // 認証情報なし
  static const INVALID_RECEIPT = 4; // レシート情報が不正です
  static const ALREADY_EXIST = 5; // 同じトランザクションが存在している
  static const UNEXPECTED_ERROR = 99; // 不明なエラー
}
```

ユーザーがサブスク購入のボタンをタップした際には下記処理を走らせます。

```dart
  /// サブスクの購入を実行する
  Future<void> buyNonConsumable() async {
    print("BUY NON CONSUMABLE");
    try {
      PurchaseParam purchaseParam = PurchaseParam(
        productDetails: product!,
        applicationUserName: null,
      );
      await _connection.buyNonConsumable(purchaseParam: purchaseParam);
    } catch (err) {
      // TODO
    }
  }
```

`_connection.buyNonConsumable` でストアと通信を行い、購入処理の状況は Stream で監視します。

```dart
    Stream purchaseUpdated = _connection.purchaseStream;
    _subscription = purchaseUpdated.listen((purchaseDetailsList) {
      _listenToPurchaseUpdated(purchaseDetailsList);
    }, onDone: () {
      _subscription.cancel();
    }, onError: (error) {
    }) as StreamSubscription<List<PurchaseDetails>>;
```

購入ステータスが完了になれば購入情報を取得できます。

```dart
  /// 購入処理のリスナー
  void _listenToPurchaseUpdated(
      List<PurchaseDetails> purchaseDetailsList) async {
    print("LISTEN TO PURCHASE UPDATED");

    if (purchaseDetailsList.isEmpty) {
      return;
    }

    purchaseDetailsList.forEach((PurchaseDetails purchaseDetails) async {

      // PurchaseStatus.pending
      if (purchaseDetails.status == PurchaseStatus.pending) {
        showPendingUI(true);
      } else {
        // PurchaseStatus.error
        if (purchaseDetails.status == PurchaseStatus.error) {
        }

        // PurchaseStatus.purchased
        else if (purchaseDetails.status == PurchaseStatus.purchased) {
          final result = await _verifyPurchase(purchaseDetails);
          if (result == BillingStatus.SUCCESS) {
            _isBillingUser = true;
          }
        }

        // PurchaseStatus.restored
        else if (purchaseDetails.status == PurchaseStatus.restored) {
          final result = await _verifyPurchase(purchaseDetails);
          if (result == BillingStatus.SUCCESS) {
            _isBillingUser = true;
          }
        }

        if (purchaseDetails.pendingCompletePurchase) {
          await _connection.completePurchase(purchaseDetails);
        }
        showPendingUI(false);
      }
    });
  }
```

購入情報を取得したら Cloud Functions で検証を行います。

検証結果の受け渡しには値を独自に定義して利用しています。(ここはもっとスマートな方法ありそう、、)

```dart
  /// CloudFunctions経由でレシート検証, 期限検証, (検証成功であれば)Firestoreへレシート登録を行う
  Future<int> _verifyPurchase(PurchaseDetails purchaseDetails) async {
    print("VERIFY PURCHASE");

    try {
      // iOSのレシート検証
      if (Platform.isIOS) {
        HttpsCallable verifyReceipt =
            FirebaseFunctions.instanceFor(region: 'asia-northeast1')
                .httpsCallable('verifyIos');
        final HttpsCallableResult result = await verifyReceipt.call(
            {'data': purchaseDetails.verificationData.localVerificationData});

        print("Verify Purchase RESULT: " + result.data.toString());
        return result.data[BillingStatus.result];
      }
      // Androidのレシート検証
      else if (Platform.isAndroid) {
        if (purchaseDetails is GooglePlayPurchaseDetails) {
          HttpsCallable verifyReceipt =
              FirebaseFunctions.instanceFor(region: 'asia-northeast1')
                  .httpsCallable('verifyAndroid');

          final HttpsCallableResult result = await verifyReceipt.call({
            'data': purchaseDetails.verificationData.localVerificationData,
            'signature': purchaseDetails.billingClientPurchase.signature
          });

          print("Verify Purchase RESULT: " + result.data.toString());
          return result.data[BillingStatus.result];
        }
      }
      return BillingStatus.UNEXPECTED_ERROR;
    } catch (_) {
      return BillingStatus.UNEXPECTED_ERROR;
    }
  }
```

参考ですが、私はステータスの更新を検知した際に UI も更新しています。

先に書いたように私は課金処理用クラスを root 付近に置いていますが、課金処理用クラスを StateNotifier にして課金処理中の画面ロックを行なっています。

```dart
  /// 画面をロックする
  void showPendingUI(bool pending) {
    state = pending;
  }
```

#### Firebase (Cloud Functions) 側の実装

購入情報の検証には Cloud Functions を利用しています。

コードは必要な箇所だけ抜粋しています。できるだけ支障がないようにしていますが、整合性の取れていないところがあるかもしれません。

```typescript
import * as functions from "firebase-functions";
import * as admin from "firebase-admin";

const firestore = admin.firestore();

const SUCCESS = 0; // 成功
const EXPIRED = 1; // 期限切れ
const DOCUMENT_NOT_FOUND = 2; // Firestoreにドキュメントなし
const NO_AUTH = 3; // 認証情報なし
const INVALID_RECEIPT = 4; // レシート情報が不正です
const ALREADY_EXIST = 5; // 同じトランザクションが存在している
const UNEXPECTED_ERROR = 99; // 不明なエラー

// iOS 購入を検証してFirestoreに保存する
export const verifyIos = functions
  .region("asia-northeast1")
  .https.onCall(async (data, context) => {
    if (context.auth === null) {
      return { result: NO_AUTH };
    }

    const uid: string = context.auth!.uid;
    const verificationData: string = data["data"];

    const latestReceipt = await verifyReceiptIos(
      verificationData,
      context.auth
    );
    if (latestReceipt === null || latestReceipt === undefined) {
      return { result: INVALID_RECEIPT };
    }

    // 同じTransactionIdが存在するか確認し、購入情報が既に登録済みでないか確認する
    const queryId: admin.firestore.QuerySnapshot = await firestore
      .collectionGroup("receipts")
      .where("transactionId", "==", latestReceipt["transaction_id"])
      .get();

    if (!queryId.empty) {
      return { result: ALREADY_EXIST };
    }

    // TODO Firestoreに保存する
    // 最低限最新レシートを保存しておけばいいと思います

    // 期限ないであることを確認する
    const now: number = Date.now();
    const expireDate: number = Number(latestReceipt["expires_date_ms"]);
    if (now < expireDate) {
      return { result: SUCCESS };
    } else {
      return { result: EXPIRED };
    }
  });

// Android 購入を検証してFirestoreに保存する
export const verifyAndroid = functions
  .region("asia-northeast1")
  .https.onCall(async (data, context) => {
    if (context.auth === null) {
      return { result: NO_AUTH };
    }

    const uid: string = context.auth!.uid;
    const verificationData: string = data["data"];
    const signature: string = data["signature"];

    const latestReceipt = await verifyReceiptAndroid(
      verificationData,
      signature,
      context.auth
    );
    if (latestReceipt === null || latestReceipt === undefined) {
      return { result: INVALID_RECEIPT };
    }

    // 同じorderIdが存在しないか確認し、購入情報が既に登録済みでないか確認する
    const queryId: admin.firestore.QuerySnapshot = await firestore
      .collectionGroup("receipts")
      .where("orderId", "==", latestReceipt["orderId"])
      .get();

    // 例として、ここでは購入情報が他のアカウントに登録済みの時は失敗としていますが、
    // アカウント作り直しのケースなども考えると、既存のものから購入情報を移行したり、重複を許容したりすることを考える必要があります。
    if (!queryId.empty) {
      return { result: ALREADY_EXIST };
    }

    /**
     * TODO
     * ここでFirestoreに保存する処理を書いてください
     * 最低限最新レシートを保存しておけばいいと思います
     */

    const now: number = Date.now();
    const expireDate: number = Number(latestReceipt["expiryTimeMillis"]);
    if (now < expireDate) {
      return { result: SUCCESS };
    } else {
      return { result: EXPIRED };
    }
  });
```

実際にストアの API を叩いて検証する関数です。

後に説明しますが、スケジューリング実行するために検証部は別に切り出しています。

```typescript
import * as functions from "firebase-functions";
import axios, { AxiosResponse } from "axios";
import * as crypto from "crypto";
import { google } from "googleapis";

const PACKAGE_NAME_IOS = "com.example";
const PACKAGE_NAME_ANDROID = "com.example";

const RECEIPT_VERIFICATION_ENDPOINT_FOR_IOS_SANDBOX =
  "https://sandbox.itunes.apple.com/verifyReceipt";
const RECEIPT_VERIFICATION_ENDPOINT_FOR_IOS_PROD =
  "https://buy.itunes.apple.com/verifyReceipt";
const RECEIPT_VERIFICATION_ENDPOINT_FRO_ANDROID =
  "https://androidpublisher.googleapis.com/androidpublisher/v3/applications/";

// Android検証用のAuthClient
const authClient = new google.auth.JWT({
  email: SERVICE_CLIENT_EMAIL_FOR_ANDROID,
  key: SERVICE_PRIVATE_KEY_FOR_ANDROID,
  scopes: ["https://www.googleapis.com/auth/androidpublisher"],
});

// レシートデータを検証してlatest_receiptを返す
// latest_receipt以外には自動購読の有効/無効などの情報もあるが、現状はlatest_receiptだけで十分な情報量と考える
export async function verifyReceiptIos(verificationData: string, auth: any) {
  let response: AxiosResponse;
  try {
    // 本番用APIにデータを送信する
    response = await axios.post(RECEIPT_VERIFICATION_ENDPOINT_FOR_IOS_PROD, {
      "receipt-data": verificationData,
      password: "TODO",
      "exclude-old-transactions": true,
    });

    // 本番用APIから返却された`status`が`21007`の場合、送信されたレシートがサンドボックス環境用と判断する
    if (response.data && response.data["status"] === 21007) {
      response = await axios.post(
        RECEIPT_VERIFICATION_ENDPOINT_FOR_IOS_SANDBOX,
        {
          "receipt-data": verificationData,
          password: "TODO",
          "exclude-old-transactions": true,
        }
      );
    }

    // レシート検証用APIから返却されたレスポンス内の`status`が`0`であれば検証は成功 https://developer.apple.com/documentation/appstorereceipts/status#possible-values
    const result = response.data;
    if (result["status"] !== 0) {
      return null;
    }

    // レスポンスデータ内の`bundle_id`が自身のパッケージ名と一致しているか確認する
    if (
      !result["receipt"] ||
      result["receipt"]["bundle_id"] !== PACKAGE_NAME_IOS
    ) {
      return null;
    }

    // `latest_receipt_info`は定期購読タイプのアイテムを購入したことがある場合のみ存在する
    return result["latest_receipt_info"][0];
  } catch (err) {
    return null;
  }
}

export async function verifyReceiptAndroid(
  verificationData: string,
  signature: string,
  auth: any
) {
  try {
    // 鍵があるなら検証を行う (renewの時には不要)
    if (signature !== "") {
      const validator = crypto.createVerify("SHA1");
      validator.update(verificationData);
      let validity = false;
      validity = validator.verify(
        STORE_PUBLIC_KEY_FOR_ANDROID,
        signature,
        "base64"
      );
      if (!validity) {
        return null;
      }
    }

    const decodedReceipt = JSON.parse(verificationData);

    // サブスク商品であることを確認する
    if (!decodedReceipt["autoRenewing"]) {
      return null;
    }

    const credential = await authClient.authorize();

    const url =
      RECEIPT_VERIFICATION_ENDPOINT_FRO_ANDROID +
      PACKAGE_NAME_ANDROID +
      "/purchases/subscriptions/" +
      decodedReceipt["productId"] +
      "/tokens/" +
      decodedReceipt["purchaseToken"] +
      "?access_token=" +
      credential.access_token;

    const response = await axios.get(url);

    const verifiedData = response.data;
    if (!verifiedData) {
      return null;
    }
    return verifiedData;
  } catch (err) {
    return null;
  }
}
```

検証に使う秘密情報は下記です。それぞれ各プラットフォームのコンソール経由で取得できます。

```typescript
RECEIPT_VERIFICATION_PASSWORD_FOR_IOS = "App用共有シークレット";
SERVICE_CLIENT_EMAIL_FOR_ANDROID =
  "Play Consoleからサービスアカウントを作成し、ダウンロードできるJSONファイルから取得します";
SERVICE_PRIVATE_KEY_FOR_ANDROID =
  "-----BEGIN PUBLIC KEY-----\nPUBLICキーに64文字ごと改行コードを挟んでいく\n-----END PUBLIC KEY-----\n"; // こちらもJSONファイルから取得します
STORE_PUBLIC_KEY_FOR_ANDROID =
  "-----BEGIN PUBLIC KEY-----\nPUBLICキーに64文字ごと改行コードを挟んでいく\n-----END PUBLIC KEY-----\n"; // Play Consoleにあるアプリのライセンス用鍵を利用します
```

Cloud Functions で秘密情報を使う場合、環境変数を利用することは推奨されていません。簡単な方法としては「ソースコードに直接埋め込む」が紹介されています。

https://cloud.google.com/functions/docs/env-var?hl=ja

Google のサービスアカウントには下記の権限を付与しています。定期購入がなくてもいけました。

![](/images/b3dbc2824800456743d5/google_permission.png)

### 3. サブスク購入済みのユーザーがアプリを起動した

ここではストアと通信を行いません。

Firestore からユーザー情報を確認し、サブスクが有効であるかを確認します。

![](/images/b3dbc2824800456743d5/billing_flow3.png)

:::message
アプリ内から直接 Firestore のユーザー情報を確認してもいいですが、サブスク期限内か判断するロジックがアプリ内にある場合、端末の時間がズレていたりあるいは悪意を持ってずらしていたりすると意図しない判断をしてしまう可能性があります。

そのため「Cloud Functions 経由で課金情報の確認を行い、Cloud Functions 内で期限内か判断する」や「最初からサーバーのユーザー情報内に有効無効のフラグを立てておき、期限の過ぎたものがあれば事前にサーバー内でフラグを更新しておく」などの対策が必要になります。
:::

### 4. サブスクが自動更新した

サブスクの期限が来ると各ストアが更新処理を行い、無事決済が完了するとユーザーのサブスク期限が延長されます。

サブスクがストア内で更新されたならば Firestore 側のユーザー情報も更新する必要があります。

ユーザー情報の更新方法として 2 つ紹介します。

#### 1. ストアのサーバー通知機能を利用する

課金情報が更新されるたびにストアから通知を受け取ることができます。  
通知を受け取ったらリアルタイムにサブスク期限を更新します。  
実装するためにはストアからの通知を受け取る環境が必要です。(PubSub とか)

https://help.apple.com/app-store-connect/#/dev0067a330b

https://developer.android.com/google/play/billing/getting-ready?hl=ja

![](/images/b3dbc2824800456743d5/billing_flow4_1.png)

#### 2. 定期的にユーザーのサブスク期限を確認する

Cloud Functions のスケジューリングなどで定期的にユーザーの課金情報問い合わせを行い、サブスク期限の更新があれば Firestore のユーザー情報を更新します。

![](/images/b3dbc2824800456743d5/billing_flow4_2.png)

#### 私が採用している更新確認方法

「定期的にユーザーのサブスク期限を確認する」で更新確認を行なっています。環境構築も実装も簡単であると考えたためです。

下記フローを 1 時間に 1 回走らせています。

1. Firestore のユーザーデータから「サブスク期限まで 2 時間切っているユーザー から サブスク期限から 2 時間経過しているユーザー まで」を抽出する。
2. Firestore に保存済みの購入情報データを利用し、ストアに購入情報の検証を依頼する。
3. 2 で受け取った検証結果(購入情報)を確認する。
4. サブスク期限が更新されていれば Firestore のユーザー情報も更新する。

> なぜ期限前後 2 時間を抽出しているのか？

フローが走るタイミングにより検証漏れが発生しないようにするためです。実際のところサブスク前後 1 時間でも十分と思いますが、二重に検証してもコスト的にも大きな問題とはならないため若干の余裕を持たせています。

:::message
購入情報検証用の API の呼び出し回数には制限があるため、大規模アプリにおいてはストアからのリアルタイム通知を利用することが推奨されています。
:::

> 注: 割り当て制限があるため、リアルタイム デベロッパー通知を活用せずに Google Play Developer API を定期的にポーリングしてステータスを確認することは推奨されません。

https://developer.android.com/google/play/billing/subscriptions?hl=ja

https://developers.google.com/android-publisher/quotas?hl=ja

iOS についてのドキュメントを見つけることはできませんでしたが、制限がある前提で考えた方が無難だと思います。

### 5. 購入情報の復元ボタンをタップした

各ストアの API を叩き、過去の購入情報を入手して再検証します。購入情報は purchaseStream で受け取ります。

![](/images/b3dbc2824800456743d5/billing_flow5.png)

```dart
class BillingService extends StateNotifier<bool> {
  BillingService() : super(false) {
    // ...
  }

  // ...

  /// ストア情報の初期化を行う
  Future _initStoreStatus() async {
    // ...
  }

  /// サブスクの購入を実行する
  Future<void> buyNonConsumable() async {
    // ...
  }

  /// CloudFunctions経由でレシート検証, 期限検証, (検証成功であれば)Firestoreへレシート登録を行う
  Future<int> _verifyPurchase(PurchaseDetails purchaseDetails) async {
    // ...
  }

  /// restorePurchaseを実行するとbuyNonConsumableと同じようにpurchaseStreamが走る
  Future<void> restorePurchase() async {
    print("RESTORE BILLING");
    try {
      await _connection.restorePurchases();
    } catch (_) {}
  }

  /// 購入処理のリスナー
  void _listenToPurchaseUpdated(
    List<PurchaseDetails> purchaseDetailsList) async {
    print("LISTEN TO PURCHASE UPDATED");

    if (purchaseDetailsList.isEmpty) {
      return;
    }

    purchaseDetailsList.forEach((PurchaseDetails purchaseDetails) async {

      // PurchaseStatus.pending
      if (purchaseDetails.status == PurchaseStatus.pending) {
        showPendingUI(true);
      } else {
        // PurchaseStatus.error
        if (purchaseDetails.status == PurchaseStatus.error) {
        }

        // PurchaseStatus.purchased
        else if (purchaseDetails.status == PurchaseStatus.purchased) {
          final result = await _verifyPurchase(purchaseDetails);
          if (result == BillingStatus.SUCCESS) {
            _isBillingUser = true;
          }
        }

        // PurchaseStatus.restored
        else if (purchaseDetails.status == PurchaseStatus.restored) {
          final result = await _verifyPurchase(purchaseDetails);
          if (result == BillingStatus.SUCCESS) {
            _isBillingUser = true;
          }
        }

        if (purchaseDetails.pendingCompletePurchase) {
          await _connection.completePurchase(purchaseDetails);
        }
        showPendingUI(false);
      }
    });
  }

  /// 画面をロックする
  void showPendingUI(bool pending) {
    state = pending;
  }
}
```

## その他補足など

### 購入情報の検証をモバイル端末で行う

小規模アプリならわざわざサーバー側で購入情報を検証せずとも、モバイル端末側で検証すれば十分ではないかと考える方もいるかもしれません。

しかしながら、それは「技術的には可能かもしれないが、推奨は一切できない」と考えています。

一番の理由は Apple 公式ドキュメントには明示的に禁じられているためです。(Android もどこかに書いてあった気がしますが、記載場所を忘れてしまった)

https://developer.apple.com/jp/documentation/storekit/in-app_purchase/validating_receipts_with_the_app_store/

また、先に述べた API 呼び出し回数の問題もありますので推奨はできません。

### RevenueCat

ここまで Firestore や Cloud Functions を駆使して購入情報の検証を説明してきましたが、最近は購入情報の検証を代わりに行ってくれる RevenueCat というサービスがあるようです。

自分でバックエンドを用意しなくてもいいのは楽そうですね。もし私が本アプリ作成前に知っていたら利用していたかもしれません。

https://www.revenuecat.com/

## 参考にさせていただいたサイト

https://tkzo.jp/blog/flutter-iap-implementation/

https://ameblo.jp/principia-ca/entry-12071725733.html

https://qiita.com/ckm/items/b8cf23ba4bd0ae5bbf34

https://devpixiv.hatenablog.com/entry/2014/12/09/111310
