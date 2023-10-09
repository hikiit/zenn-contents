---
title: "Flutter iOS17 「ATTのダイアログを表示していない」とリジェクトされた時の対策"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter", "ios", "appstoreconnect"]
published: true
---

## 環境

- Flutter 3.13.5
- Xcode 14.2
- ATTのダイアログ表示 app_tracking_transparency(2.0.1)  
(https://pub.dev/packages/app_tracking_transparency)

## リジェクト時の指摘内容

軽微な更新を実施したところ、下記指摘内容にてリジェクトを受けました。

> 2.1.0 Performance: App Completeness
> Guideline 2.1 - Information Needed
> We're looking forward to completing our review, but we need more information to continue. Your app uses the AppTrackingTransparency framework, but we are unable to locate the App Tracking Transparency permission request when reviewed on iOS 17.0.3.

## 原因

画面表示後、即座にATTダイアログの表示を試みると表示されないことがあるようです。

Issueに同じ現象が報告されていました。(https://github.com/deniza/app_tracking_transparency/issues/47)

## 対策

Issueに紹介されていた回避案を実装したところ、問題なくレビューを通過できました。

```dart
Future _trackingTransparencyRequest() async {
    final TrackingStatus trackingStatus = await AppTrackingTransparency.trackingAuthorizationStatus;
    if(trackingStatus == TrackingStatus.notDetermined) {
        await Future.delayed(const Duration(milliseconds: 1000)); // 1秒遅らせる
        final status = await AppTrackingTransparency.requestTrackingAuthorization(); // ATTダイアログを表示する
    }
}
```
