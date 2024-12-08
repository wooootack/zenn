---
title: "ECSのTaskProtectionを利用して安全なタスクの入れ替えを実現する"
emoji: "🌀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - aws
  - ecs
  - kotlin
  - springboot
published: false
# published_at: 2024-12-08 09:00
publication_name: "loglass"
---

![アドカレのOGP画像](https://storage.googleapis.com/zenn-user-upload/639989a2fc77-20241202.png)

:::message
この記事は **[株式会社ログラス Productチーム Advent Calendar 2024](https://qiita.com/advent-calendar/2024/loglass)** のシリーズ 1、**8日目** の記事です。
:::

## はじめに

こんにちは、ログラスでエンジニアをしています粟田（[@wooootack](https://twitter.com/wooootack)）です。
今日は久しぶりに純粋な技術記事を書こうと思います！ぜひ最後までお付き合いください。

唐突ですが、皆さんのバックエンドアプリケーションには非同期処理が実装されているでしょうか？
恐らく、ほとんどの方が **「はい」** と答えるのではないでしょうか。

現代のアプリケーションにおいて、非同期処理は非常に重要な役割を担っています。特に、ユーザーリクエストに対する迅速な応答が求められる場面では、バックグラウンドでのデータ処理を非同期で実行することで、システムの効率性を高め、UIの応答性を保つことができます。

たとえば、大量のデータを扱う処理や外部APIとのやり取りなど、長時間かかるタスクを非同期で実行することで、ユーザーには速やかにレスポンスを返しつつ、バックエンドでの処理を行うことができます。しかし、特に登録系の非同期処理は、排他制御を入れたり正しくロールバックされるようにしておくなどの工夫が必要で、正しくハンドリングができなければデータの不整合や重複登録などの問題が発生する可能性があります。

特に、デプロイやスケールインによるコンテナ入れ替えの際には気を付ける必要があり、うまく工夫をしないと非同期処理が中断される といった問題が発生してしまします。今回は、この課題を解決した方法について、サンプルコードも混じえて詳しくご紹介します。

ちなみに、記事内で紹介するサンプルコードは、全て以下のリポジトリで公開しています。実際にローカルで動かせる状態にしているので、参考にしていただければと思います。

https://github.com/wooootack/ecs_task_protection_example

## 我々の非同期処理の実行環境

私たちのバックエンドアプリケーションは Kotlin × Spring Boot で構築されており、AWSのElastic Container Service（以下 ECS）上で運用されています。

:::message
より詳細な技術スタックについては、以下の記事を参照してください
:::
https://zenn.dev/loglass/articles/open-loglass-tech-stack-2023

また、弊社の非同期処理の実行環境としては以下の2種類が混在している状況ですが、どちらの環境でも同様の問題が発生します。

1. バックエンドAPIが実行されているタスクで、別スレッドを使って実行
2. Worker と呼ばれる非同期処理専用の別タスクで実行（SQS を使ったポーリング型）

:::message
より詳細な非同期処理基盤については、以下の記事を参照してください
:::
https://zenn.dev/loglass/articles/51447768d35958

発生していた問題は、前述した通り **「デプロイやスケールインによるコンテナ入れ替えの際に、非同期処理が中断される」** なのですが、もう少し深堀りして解説します。

私たちのアプリケーションでは以下のような非同期処理が実装されています。

:::message

- 簡略化しているため実際のコードとは異なります
  - 細かい例外ハンドリングやバリデーションは省略しています
- サンプルコードは全て API と同じタスクで実行される処理の想定です
  - ただしWorkerの実装もそこまで大きくは変わりません

:::

```kotlin
package com.example.ecs_task_protection_example

import org.slf4j.LoggerFactory
import org.springframework.scheduling.annotation.Async
import org.springframework.stereotype.Service

@Service
class HeavyAsyncProcessWithoutTaskProtectionService(
    private val asyncProcessEventRepository: AsyncProcessEventRepository,
    // 引数で受け取った関数をトランザクションを切って実行するサービスです
    private val transactionalExecutor: TransactionalExecutor,
) {
    private val logger = LoggerFactory.getLogger(HeavyAsyncProcessWithoutTaskProtectionService::class.java)

    fun execute() {
        logger.info("""
            Start heavy process without task protection
        """.trimIndent())

        val event = asyncProcessEventRegister()
        runAsyncProcess(event)

        logger.info("""
            End heavy process without task protection
        """.trimIndent())
    }

    /**
     * 非同期処理が始まる前に別トランザクションでイベントを登録する
     */
    fun asyncProcessEventRegister(): HeavyAsyncProcessEvent {
        return transactionalExecutor.execute {
            val asyncProcessEvent = asyncProcessEventRepository.findRunningEvent()
            if (asyncProcessEvent != null) {
                throw Exception("既に処理中のイベントが存在します")
            }

            val runningEvent = HeavyAsyncProcessEvent.createRunningEvent()
            asyncProcessEventRepository.insert(runningEvent)

            runningEvent
        }
    }

    /**
     * イベント登録後に別トランザクションを切って実行する
     * これは非同期処理なのですぐにレスポンスを返す
     *
     * このメソッドが中断されたりそもそも呼び出されなかった場合、イベントのステータスが更新されない
     */
    @Async
    fun runAsyncProcess(event: HeavyAsyncProcessEvent) {
        try {
            transactionalExecutor.execute {
                Thread.sleep(30000) // 重い処理を模擬
                event.success()
                asyncProcessEventRepository.update(event)
            }
        } catch (e: Throwable) {
            transactionalExecutor.execute {
                // この後に再スローするためトランザクションを切ってステータスを変更する
                event.failure()
                asyncProcessEventRepository.update(event)
            }
            throw e
        }
    }
}
```

このように、複数のトランザクションに分けて、非同期処理イベントの登録と実行とイベントのステータス更新を行っています。そして、`asyncProcessEventRepository.update` が呼び出される前に、コンテナが入れ替わって非同期処理が中断されてしまうと、イベントのステータスが更新されないままになってしまうのです。これを解決するために、ECS の TaskProtection を利用して、処理が完了するまではコンテナの入れ替えを行わないようにすることにしました。

## ECS の TaskProtection について

TaskProtection は、その名の通り「タスクを保護」する機能で、タスクは終了させずに動かし続けるために使用します。私たちのアプリケーションは経営管理のためのデータを取り扱うため、どうしても時間がかかってしまう処理があり、タスクを終了させずに動かし続ける必要があったため、まさに求めていた機能と言えます。

ざっくり説明すると、この機能を使って保護されたタスクに対して、SIGTERMやSIGKILLのシグナルがECSのコントロールプレーンから送信されることがなくなるため、タスクが終了せずに処理を実行できるといった仕組みになります。

TaskProtection のより詳しい説明は以下の公式ドキュメントも参照してください。
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task-scale-in-protection.html

TaskProtection は、以下のようにエンドポイントを叩くことで有効化と無効化を切り替えることができます。

```bash
curl --request PUT --header 'Content-Type: application/json' \
  ${ECS_AGENT_URI}/task-protection/v1/state \
  --data '{"ProtectionEnabled":true,"ExpiresInMinutes":60}'
```

詳細なパラメータなどは、以下の公式ドキュメントを参照してください。
https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task-scale-in-protection-endpoint.html

次のセクションからは具体的なサンプルコードを混じえて、より詳細に解説していきます。

## TaskProtection を使った非同期処理のサンプルコード

まずは公式ドキュメントに記載されているサンプルコードを読んでイメージを掴んでみましょう。

https://aws.amazon.com/jp/blogs/news/announcing-amazon-ecs-task-scale-in-protection/

こちらが公式サイトから持ってきたサンプルコードです。

```javascript
async function pollForWork() {
  console.log('Acquiring task protection');
  await TaskProtection.acquire();

  var message = await receiveMessage();

  if (message) {
    await processMessage(message);
  }

  console.log('Releasing task protection');

  await TaskProtection.release();
  return maybeContinuePolling();
}
```

これは SQS からメッセージを取ってきて使うパターンですが、以下のような手順で処理を行っています。

1. TaskProtection を有効化
2. SQS からのメッセージを受け取り処理を実行する
3. TaskProtection を無効化する

これによって、メッセージを受け取って処理をする部分は確実に終了することが保証できる、という流れになっています。

続いて、先ほどのサンプルコードに TaskProtection を取り入れたサンプルを以下に用意しました。

```kotlin
package com.example.ecs_task_protection_example

import org.slf4j.LoggerFactory
import org.springframework.scheduling.annotation.Async
import org.springframework.stereotype.Service

@Service
class HeavyAsyncProcessWithTaskProtectionService(
    private val asyncProcessEventRepository: AsyncProcessEventRepository,
    private val transactionalExecutor: TransactionalExecutor,
    private val taskProtectionExecutor: TaskProtectionExecutor,
) {
    private val logger = LoggerFactory.getLogger(HeavyAsyncProcessWithTaskProtectionService::class.java)

    fun execute() {
        logger.info("""
            Start heavy process with task protection
        """.trimIndent())

        val event = asyncProcessEventRegister()
        taskProtectionExecutor.execute("HeavyAsyncProcess") { runAsyncProcess(event) }

        logger.info("""
            End heavy process with task protection
        """.trimIndent())
    }

    /**
     * 非同期処理が始まる前に別トランザクションでイベントを登録する
     */
    fun asyncProcessEventRegister(): HeavyAsyncProcessEvent {
        return transactionalExecutor.execute {
            val asyncProcessEvent = asyncProcessEventRepository.findRunningEvent()
            if (asyncProcessEvent != null) {
                throw Exception("既に処理中のイベントが存在します")
            }

            val runningEvent = HeavyAsyncProcessEvent.createRunningEvent()
            asyncProcessEventRepository.insert(runningEvent)

            runningEvent
        }
    }

    /**
     * イベント登録後に別トランザクションを切って実行する
     * これは非同期処理なのですぐにレスポンスを返す
     *
     * このメソッドが中断されたりそもそも呼び出されなかった場合、イベントのステータスが更新されない
     */
    @Async
    fun runAsyncProcess(event: HeavyAsyncProcessEvent) {
        try {
            transactionalExecutor.execute {
                Thread.sleep(30000) // 重い処理を模擬
                event.success()
                asyncProcessEventRepository.update(event)
            }
        } catch (e: Throwable) {
            transactionalExecutor.execute {
                // この後に再スローするためトランザクションを切ってステータスを変更する
                event.failure()
                asyncProcessEventRepository.update(event)
            }
            throw e
        }
    }
}
```

パッと見、どこが変わったか分からないかもしれません。実際、たった1行しか変わらない形で実装されています。それが以下の部分です。

```kotlin
// before
runAsyncProcess(event)

↓

// after
taskProtectionExecutor.execute("HeavyAsyncProcess") { runAsyncProcess(event) }
```

この `TaskProtectionExecutor` というのが、TaskProtection の有効化と無効化を切り替えながら処理をいい感じに実行してくれるクラスです。そちらは以下のような実装になっています。

```kotlin
package com.example.ecs_task_protection_example

import org.slf4j.LoggerFactory
import org.springframework.http.MediaType
import org.springframework.stereotype.Service
import org.springframework.web.client.RestClient
import java.util.concurrent.atomic.AtomicInteger

@Service
class TaskProtectionExecutor(
    private val transactionalExecutor: TransactionalExecutor,
    private val asyncProcessHistoryRepository: AsyncProcessHistoryRepository,
) {
    companion object {
        private val coroutineCount = AtomicInteger(0)
    }

    private val logger = LoggerFactory.getLogger(TaskProtectionExecutor::class.java)

    fun execute(
        name: String,
        action: () -> Unit,
    ) {
        val asyncProcessHistory = AsyncProcessHistory.createStartedJob(name, getContainerId())

        try {
            logger.info(
                """
                Start job!
                id: ${asyncProcessHistory.id}, container_id: ${asyncProcessHistory.containerId}, name: ${asyncProcessHistory.name}
                """.trimIndent()
            )

            protectTask()
            transactionalExecutor.execute {
                asyncProcessHistoryRepository.insert(asyncProcessHistory)
            }

            return action()
        } catch (e: Throwable) {
            logger.error(
                """
                Error occurred in job!
                id: ${asyncProcessHistory.id}, container_id: ${asyncProcessHistory.containerId}, name: ${asyncProcessHistory.name}
                """.trimIndent(),
                e,
            )
            throw e
        } finally {
            logger.info(
                """
                Complete job!
                id: ${asyncProcessHistory.id}, container_id: ${asyncProcessHistory.containerId}, name: ${asyncProcessHistory.name}
                """.trimIndent(),
            )

            unprotectTask()
            transactionalExecutor.execute {
                asyncProcessHistoryRepository.update(asyncProcessHistory.complete())
            }
        }
    }

    private fun protectTask() {
        val count = coroutineCount.incrementAndGet()
        logger.info("Current Coroutine count: $count")
        setProtectionEnabled(true)
        logger.info("Protection enabled")
    }

    private fun unprotectTask() {
        val count = coroutineCount.decrementAndGet()
        logger.info("Current Coroutine count: $count")
        if (count <= 0) {
            setProtectionEnabled(false)
            logger.info("Protection disabled")
        }
    }

    private fun setProtectionEnabled(enabled: Boolean) {
        val ecsAgentUri = System.getenv("ECS_AGENT_URI")
        if (ecsAgentUri.isNullOrBlank()) {
            // NOTE: 環境変数がセットされていない場合はローカルで実行しているとみなし、何もしない
            return
        }

        val response = RestClient
            .create()
            .post()
            .uri("$ecsAgentUri/task-protection/v1/state")
            .contentType(MediaType("application/json"))
            .body("{\"ProtectionEnabled\":$enabled,\"ExpiresInMinutes\":1440}")
            .retrieve()
            .body(String::class.java)

        logger.info(
            """
            Set task protection
            Response: $response
            """.trimIndent(),
        )
    }

    private fun getContainerId(): String {
        // NOTE: ECS_AGENT_URIは、以下のような形式で取得できる
        // http://169.254.170.2/api/b194c578e01b40db82f0a5c7b9b717c7-1622016141

        val ecsAgentUri = System.getenv("ECS_AGENT_URI")
        if (ecsAgentUri.isNullOrBlank()) {
            return ""
        }

        // NOTE: ECS_AGENT_URIの最後のパスがコンテナIDになる
        return ecsAgentUri.split("/").last()
    }
}
```

こちらのクラスは、Kotlin の AtomicInteger を使って実行中の非同期処理をカウントしながら、実行中の非同期処理が1つでもある場合は TaskProtection を有効化し、全ての非同期処理が完了したら TaskProtection を無効化するように実装されています。

APIサーバーを起動して、以下のコマンドを使って連続してリクエストを送ってみてください。

```bash
curl localhost:8080/with_task_protection -X POST
```

そうすると、以下のようにログが出力されるはずです。横長で少し見づらいですが、カウントが増減することと、タスク保護の有効化・無効化が正しく切り替わっていることが確認できます。（実際には ECS で動作させないと動作確認はできませんが、今回は省略します）

![ログのキャプチャ](https://storage.googleapis.com/zenn-user-upload/c7b81bbcf00a-20241202.png)
*ターミナルに表示されたログのキャプチャ*

また、お気づきの方もいるかもしれませんが、`AsyncProcessHistory` には `container_id` を持たせているため、どのコンテナで動いている処理なのかも確認できるようになっています。万が一、OOM などが発生してコンテナが強制終了してしまった場合も、起動中のコンテナとデータベースのレコードを比較することで、巻き込まれてしまった処理なのかどうかもすぐに判断できるようになります。

## アプリケーションコード以外の変更点

さて、これまではアプリケーションコードの変更について紹介しましたが、それだけではうまく動きません。細かい設定は各社ごとに異なるかと思いますが、以下に代表的な変更点を挙げてみます。

1. TaskProtection を設定するための権限付与
2. トラフィック・ポーリングの切り替え

### TaskProtection を設定するための権限付与

「アプリケーションを修正してデプロイしたら、権限が不足していてうまく動きませんでした」なんてことがないように、必ず以下の公式ドキュメントを参考に権限を付与しておきましょう！

https://docs.aws.amazon.com/ja_jp/AmazonECS/latest/developerguide/task-scale-in-protection.html#task-scale-in-protection-iam

### トラフィック・ポーリングの切り替え

TaskProtection を使うことで新旧のタスクが混在する期間がこれまでよりも長くなります。古いアプリケーションで処理が実行されることを防ぐために、B/Gデプロイを使ったトラフィックの切り替えや、ポーリングを実行するタスクの切り替えも必要になります。（今回 AWS 周りは本題ではないのでこれ以上は割愛します）

## まとめ

今回は、ECS の TaskProtection を活用して、非同期処理が中断しないようタスクを安全に入れ替えるための方法をご紹介しました。
こちらの方法を使うことで、ほとんどの場合においては非同期処理は中断されることはなくなりますが、OOM などでコンテナが強制終了してしまった場合など防ぎきれないケースもまだ残っており、引き続き取り組んでいきます。将来を見据えた大きなアーキテクチャの変更も検討していく必要があるフェーズに来たのかなと最近はよく考えるようになりました。

最後に少しだけ宣伝ですが、ログラスではエンジニアを大募集していますので、ご興味がある方はぜひ以下もご覧いただければと思います。

https://www.loglass.co.jp/recruit/for-engineers

いきなり申し込むのはちょっとハードル高いなーという場合は、真面目な話も不真面目な話にも対応した私の Piita にご応募お待ちしています。

https://pitta.me/matches/CMegIkhpGqgD

https://pitta.me/matches/sRRdWLqHjuQD
