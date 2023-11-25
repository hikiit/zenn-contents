---
title: "Kotlin Callback形式のコードはCoroutine形式にラップしよう"
emoji: "🔁"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["kotlin"]
published: false
---

## 本記事の目的

前提として、本記事の内容は Kotlin を利用しているプロジェクトを対象に書いています。

最近では数が減ってきてはいますが、古いライブラリを利用していたり、歴史の長いプロジェクトでは Callback が今も活躍していると思います。

もちろんそれは悪いことではないのですが、Callback 形式のコードは Kotlin Coroutine でいい感じに書き直せるので今更ながら紹介します。(Coroutine の基礎的な非同期処理については割愛)

## よくある Callback 形式のコード

まずはよくある Callback 形式を利用したコードを紹介します。

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

class TestModel() {
    interface ResultListener {
        fun onSuccess()
        fun onFailure()
    }

    fun api1(listener: ResultListener) {
        Thread.sleep(5000)

        listener.onSuccess()
    }
}

fun main() {
    val testModel = TestModel()

    val callback = object: TestModel.ResultListener {
        override fun onSuccess() {
            print("success")
        }

        override fun onFailure() {
            print("failure")
        }
    }

    testModel.api1(callback)
}
```

動作を試す場合はこちら "https://pl.kotl.in/_zwwlvv-e"

今回 `Test Model` は自分で作成していますが、外部ライブラリと思っていただいてもいいと思います。

なんてことはない、普通のコードだと思いますが、「複数の Callback が連続する場合はネストが深くなり、メンテナンス性も低下する」という懸念を抱えています。  
これは Kotlin に限らない話ですね。

そこで、Kotlin のとある機能を利用します。

## SuspendCoroutine

suspendCoroutine を利用します。

これにより、非同期処理が終わるまでコルーチン内で待つことができます。

https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.coroutines/suspend-coroutine.html

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

class TestModel() {
    interface ResultListener {
        fun onSuccess()
        fun onFailure()
    }

    fun api1(listener: ResultListener) {
        Thread.sleep(5000)

        listener.onSuccess()
    }
}

enum class Result {
    Success,
    Failure,
}

suspend fun TestModel.api1() = suspendCoroutine<Result> { continuation ->
    val callback = object: TestModel.ResultListener {
        override fun onSuccess() {
            continuation.resume(Result.Success)
        }

        override fun onFailure() {
            continuation.resume(Result.Failure)
        }
    }

    api1(callback)
}

fun main() {
    runBlocking {
        val testModel = TestModel()
	    val result = testModel.api1()

        when (result) {
            Result.Success -> print("success")
            Result.Failure -> print("failure")
        }
    }
}
```

動作を試す場合はこちら https://pl.kotl.in/Nx1AhCpbL

合わせて拡張関数も利用することで、かなりコードの見通しが良くなったと思います。

### その他参考

他にも、 `suspendCancellableCoroutine` を利用することで途中のキャンセル(サンプルではタイムアウトを設定)を可能にしたりすることができます。

```kotlin
import kotlin.coroutines.*
import kotlinx.coroutines.*

class TestModel() {
    interface ResultListener {
        fun onSuccess()
        fun onFailure()
    }

    fun api1(listener: ResultListener) {
        Thread.sleep(5000)

        listener.onSuccess()
    }
}

enum class Result {
    Success,
    Failure,
}

suspend fun TestModel.api1() = suspendCancellableCoroutine<Result> { continuation ->
    val callback = object: TestModel.ResultListener {
        override fun onSuccess() {
            continuation.resume(Result.Success)
        }

        override fun onFailure() {
            continuation.resume(Result.Failure)
        }
    }

    api1(callback)
}

fun main() {
    runBlocking {
        val testModel = TestModel()
        val result = withTimeoutOrNull(10000) {
            testModel.api1()
        }

        when (result) {
            Result.Success -> print("success")
            Result.Failure -> print("failure")
            else -> print("timeout")
        }
    }
}
```

動作を試す場合はこちら https://pl.kotl.in/uUUfvOMK-

複数回コールバックがよばれるようなケースでは`Flow`を利用することで、同様に Coroutine で扱いやすいようにすることができます。
