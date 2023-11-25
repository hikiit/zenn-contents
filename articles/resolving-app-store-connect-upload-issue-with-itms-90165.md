---
title: "ERROR ITMS-90165 でApp Store Connectへアップロードできなくなった時の解決方法"
emoji: "🍎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ios", "appstoreconnect"]
published: false
---

## 現象

- App Store Connect へアップロードしようとしたら `ERROR ITMS-90165: Invalid Provisioning Profile Signature.` とエラーが出てアップロードできない。
- Profile の期限は十分に余裕がある。
- 同じ環境(Codemagic) で昨日アップロードした時はうまくいっていた。

## 解決策

1. Apple Developer サイトの`Certificates, Identifiers & Profiles`を開く
2. 期限が切れているはずなのに`EXPIRATION`が`Expired`になっていない Profile があれば開く (開くと Expired になる)
3. 期限内の Profile を開く -> Edit を選択する -> 何も変更しないで Save する

以上

![](/images/50061e6288d2ae/certificate_apple.png)

なぜこれで直ったのか不思議ですが、、うまくいきました。  
完全な切り分けはできていないので、不要な手順もあるかもしれません。

## 参考にさせていただいたサイト

どうやら世界的に起こっているようです。

https://stackoverflow.com/questions/71850186/invalid-provisioning-profile-signature-state-error-validation-error-90165

https://developer.apple.com/forums/thread/703995
