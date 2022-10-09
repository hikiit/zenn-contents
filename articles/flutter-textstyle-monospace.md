---
title: "Flutter 等幅フォントをText Widget単位で指定する方法"
emoji: "🔤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Flutter"]
published: true
---

## fontFeatures を指定する

`TextStyle`の`fontFeatures`に`FontFeature.tabularFigures()`を指定します。

```dart
Text("foo",
    style: TextStyle(
        fontFeatures: [FontFeature.tabularFigures()],
    ))
```

その他 FontFeature の機能については下記を参照してください。
https://api.flutter.dev/flutter/dart-ui/FontFeature-class.html
