---
title: "Android Firestoreでdata class利用時、プロパティ名にisを付けるならJvmFieldも付ける"
emoji: "👌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Android", "Kotlin", "Firestore"]
published: false
---

プロパティ名に`isFoo`と名づけることは珍しくないと思いますが、Firestore を利用する際には注意が必要です。

## プロパティ名に is を付けるなら JvmField も付けよう

例えば、以下のケースでは`isDone`という名前をつけていますが、`@field:JvmField`を付与しないと Firestore の`toObject(Task::class.java)`で正常にパースできません。

```kotlin
data class Task(
    @DocumentId val documentId: String? = null,
    val title: String = "",
    @field:JvmField // use this annotation if your Boolean field is prefixed with 'is'
    val isDone: Boolean = false,
)
```

### なぜか

通常 Kotlin でプロパティを宣言すると、Java 向けに getter/setter が生成され、Java からは getter/setter を通してアクセスすることになります。

しかし`isFoo`で生成される getter/setter が Firestore とコンフリクトしてしまっているようです。

`@field:JvmField` を付与すると Java からも直接アクセスできるので、問題は回避できます。

Firebase のドキュメントでも同様のコードで書かれています。

https://firebase.google.com/docs/firestore/manage-data/add-data#kotlin+ktx_3

### その他参考にしたサイト

https://stackoverflow.com/questions/60645561/firestore-boolean-returns-false-when-its-set-up-to-true
