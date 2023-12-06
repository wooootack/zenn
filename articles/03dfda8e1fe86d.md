---
title: "エクセル読み込みをPOIからFastExcelに置き換えてパフォーマンスを改善する"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - Kotlin
  - Java
  - POI
  - エクセル
  - パフォーマンス
published: true
publication_name: "loglass"
---

![thumbnail](https://storage.googleapis.com/zenn-user-upload/378659d2156d-20231207.png)

:::message
この記事は **[株式会社ログラス Product チーム Advent Calendar 2023](https://qiita.com/advent-calendar/2023/loglass)** の **16 日目** の記事です。
:::

# はじめに

こんにちは！10 月から株式会社ログラスでエンジニアをやっています、Kyosuke です！

ログラスでは、エクセルファイルをプログラムから操作する処理が一部存在しており、[Apache POI](https://poi.apache.org/)というライブラリを使用しています。（以後`POI`と呼びます）
しかし、`POI` には処理方式によってはメモリを多量に使用してしまうという問題があります。

今回はその対応として、まずエクセルファイルの読み込みを[FastExcel](https://github.com/dhatim/fastexcel)というライブラリに置き換えた話を振り返っていきます。

# TL;DR

- `POI`の`XSSFWorkbook`は、ファイルをメモリ内に読み込んで操作するため、大きなエクセルファイルを処理する際にファイルサイズ以上のメモリを必要とすることがある
  ![poiのメモリ問題](https://storage.googleapis.com/zenn-user-upload/20fbe40f55f4-20231207.png)

  > [POI の公式ドキュメントより](https://poi.apache.org/apidocs/dev/org/apache/poi/xssf/usermodel/XSSFWorkbook.html#XSSFWorkbook-java.io.InputStream-)

- `FastExcel`は、`POI`より機能は劣る代わりに、読み書きともにパフォーマンスは大幅に上回る

  - 書き込み
    ![fastexcelの処理時間](https://storage.googleapis.com/zenn-user-upload/47febbb72bab-20231207.png)
    ![fastexcelのメモリ使用量](https://storage.googleapis.com/zenn-user-upload/c834ee4b06d5-20231207.png)

    > [FastExcel 公式のベンチマークより](https://github.com/dhatim/fastexcel#benchmark)

  - 読み込み
    ![fastexcelの読み込みベンチマーク](https://storage.googleapis.com/zenn-user-upload/fbcc43363a16-20231207.png)
    > [FastExcel 公式のベンチマークより](https://github.com/dhatim/fastexcel#benchmarks)

- 読み込みにおいては、よほど特殊な内容のセルなどがなければ十分に置き換え可能
  - 数値、文字列、日付、時刻、関数、エラーは検証済み
- 書き込みは未検証だが、単純にセルにデータを書き込んでいくのであれば十分に置き換え可能と予想している

# 置き換えを決断したきっかけ

置き換えを決断したきっかけを一言で表すと **限界を迎えたから** です。

元々、既知の不具合として **「巨大なエクセルファイルを読み込んだタイミングでメモリを使い切ってしまい FullGC が発生する」** という現象があったのですが、これまでは ECS タスクのメモリ増強などで延命していました。
しかし、発生頻度が上がったことや今後は更に大きなエクセルファイルを読み込む可能性があることなどから、これ以上は先送りにできないという決断を下しました。

# なぜ FastExcel を選んだのか

今回の大きな目的はメモリ使用量を削減することなので、`POI`が提供している [XSSF and Sax (Event API)](https://poi.apache.org/components/spreadsheet/how-to.html#xssf_sax_api) の別のやり方を選ぶこともできます。
しかし、公式ドキュメントにも記載があるように、コード量が多かったりエクセルファイルの構造を少し学ぶ必要があります。

そんな時、偶然（ではないけど）`FastExcel` を発見し、メモリの問題も解決できそうかつ インターフェースもシンプルで良いのではと候補に上がり、`POI`と同じことが実現できるかを調査して問題なければ置き換えようとなりました。

`POI`と`FastExcel`の違いについて、以下の３つの観点でそれぞれまとめてみました。

- 仕様
- コード
- パフォーマンス

# POI vs FastExcel - 仕様編

まずは`POI`と`FastExcel`の仕様の違い（特に今回関係するところ）を確認してみます。

## POI - 仕様

前述したように、`POI`の`XSSFWorkbook`では、ファイルの中身を一気にメモリ上に読み込んでから処理を行うため、メモリを多く使用します。
エクセルファイルは非常に複雑な構造であり、正確に操作するためには一度にメモリ上に読み込む必要があり、`POI`はこれによって非常に多くの機能を提供しています。
ただ、20万行の空行が存在しているエクセルファイルの場合なども全て読み込むため、不要なメモリを使用してしまうことになります。

※ `POI`にもストリーム形式でファイルを読み込む方法もありますが、そちらは今回の話の対象に入れていません。

## FastExcel - 仕様

POI と比較して、`FastExcel`は、マルチスレッドに対応していたり、ストリーム形式で逐次的にファイルを読み込んだりと、パフォーマンスの面で大きく優れています。
（`FastExcel`は`POI`よりも約 10 倍の速度で処理が可能と Github にも記載されています）
そのため、`POI`と比較して不足している機能は一部存在しています。

最終的に、私たちの用途においては、`POI`が提供している多くの機能は必要としておらず、パフォーマンスを重視した`FastExcel`を選択しました。

# POI vs FastExcel - コード編

続いて、`POI`と`FastExcel`でセルに入力されている内容を取得するコードの違いを確認してみます。

## POI - コード

まずは `POI` であるセルの値を取得する処理を書いてみました。
特段自前の実装は必要なく、セルの値を取得することができます。
エクセルでは日付は実際には数値として保存されているため、種別が `NUMERIC` の場合には日付セルかどうかを判別する必要がありますが、その辺りも `POI` がメソッドを準備してくれているため、簡潔に処理を書くことができます。
また、ゼロ除算の際に表示される `#DIV/0!` なども、種別がエラーと判断してくれるため、検知することが可能です。

```kotlin
fun sample(inputStream: InputStream) {
    val workbook = XSSFWorkbook(inputStream)
    val sheet = workbook.getSheetAt(0)
    val cell = sheet.getRow(0).getCell(0)
    val value = when (cell.cellType) {
        CellType.BLANK -> ""
        CellType.BOOLEAN -> cell.booleanCellValue.toString()
        CellType.STRING -> cell.stringCellValue
        CellType.NUMERIC -> {
            if (DateUtil.isCellDateFormatted(cell)) {
                cell.dateCellValue.toString()
            } else {
                cell.numericCellValue.toString()
            }
        }
        CellType.FORMULA -> {
            // Excelの計算式はxmlで計算結果を保持しているため、都度計算しないようにする
            when (cell.cachedFormulaResultType) {
                CellType.BOOLEAN -> cell.booleanCellValue.toString()
                CellType.STRING -> cell.stringCellValue
                CellType.NUMERIC -> {
                    if (DateUtil.isCellDateFormatted(cell)) {
                        cell.dateCellValue.toString()
                    } else {
                        cell.numericCellValue.toString()
                    }
                }
                CellType.ERROR -> throw BadRequestException("エラーのセル")
                else -> throw IllegalStateException()
            }
        }
        CellType.ERROR -> throw BadRequestException("エラーのセル")
        else -> throw BadRequestException("未対応のデータ型")
    }
    println(value)
}
```

## FastExcel - コード

続いて `FastExcel` でのコードも見てみましょう。
大きな違いとしては 2 つで、エラーセルのハンドリングの部分と日付の取得の部分で、自前の実装をする必要があります。

現状はこちらの実装で問題は発生していませんが、抜け道は存在しているかもしれないので、もし可能であれば日付を入れたい場合のフォーマットを固定するなどして対処したいところです。

```kotlin
fun sample(inputStream: InputStream) {
    ReadableWorkbook(inputStream, readingOptions).use { workBook ->
        val sheet = workBook.firstSheet
        val cell = sheet.openStream().iterator().next().getCell(0)
        val text = if (cell.text.isNullOrBlank()) return "" else cell.text

        // FastExcelはこの辺りも文字列として取得してくるため、文字列判定で弾く必要がある
        val excelErrorTexts =
            listOf("#VALUE!", "#REF!", "#NAME?", "#DIV/0!", "#NULL!", "#N/A", "#NUM!", "#GETTING_DATA")
        if (text in excelErrorTexts) {
            throw BadRequestException("エラーのセル")
        }

        if (isDateCell(cell)) {
            println(cell.asDate().format(DateTimeFormatter.ofPattern("yyyy/M/d")))
        } else {
            println(text)
        }
    }
}

// 日付判定は自前で作る必要がある
private fun isDateCell(cell: Cell): Boolean {
    val dataFormatId = cell.dataFormatId ?: return false
    // 14..22は日付形式のID
    // 参考: https://www.iso.org/standard/71691.html
    if (dataFormatId in 14..22) {
        return true
    }
    val dataFormatString = cell.dataFormatString ?: return false
    // POIさんの関数を使わせて頂いております
    return DateUtil.isADateFormat(dataFormatId, dataFormatString)
}
```

# POI vs FastExcel - パフォーマンス編

最後に簡易的にではありますが、実際にパフォーマンスの差を確認してみます。
今回は [IntelliJ のプロファイラー](https://pleiades.io/help/idea/profiler-intro.html)を使ってパフォーマンスを確認しました。
自分も最近知ったのですが、これめっちゃ便利なのでぜひ使ってみてください。

そして今回使用するエクセルは、 `50万行×7列=18.8MB` のサイズで作成しています。
また、各列は以下のような状態になっています。
![テストで使うエクセル](https://storage.googleapis.com/zenn-user-upload/a17ad5bceab9-20231210.png)

## POI - パフォーマンス

単純に全ての行、全ての列を文字列として取得するようなコードで試してみます。

```kotlin
private fun parseByPOI(inputStream: InputStream) {
    // 読み込むファイルのサイズが大きい場合は最大値を増やさないとエラーになるため
    IOUtils.setByteArrayMaxOverride(200000000)
    val workbook = XSSFWorkbook(inputStream)
    val sheet = workbook.getSheetAt(0)
    val iterator = sheet.rowIterator()
    val data = iterator.asSequence().map {
        listOf(
            cellToString(it.getCell(0)),
            cellToString(it.getCell(1)),
            cellToString(it.getCell(2)),
            cellToString(it.getCell(3)),
            cellToString(it.getCell(4)),
            cellToString(it.getCell(5)),
            cellToString(it.getCell(6)),
        )
    }.toList()

    logger.info(jsonConverter.objectToJsonString(data))
}

private fun cellToString(cell: org.apache.poi.ss.usermodel.Cell?): String {
    if (cell == null) return ""

    return when (cell.cellType) {
        CellType.BLANK -> ""
        CellType.BOOLEAN -> cell.booleanCellValue.toString()
        CellType.STRING -> cell.stringCellValue
        CellType.NUMERIC -> { numericToString(cell) }
        CellType.FORMULA -> {
            when (cell.cachedFormulaResultType) {
                CellType.BOOLEAN -> cell.booleanCellValue.toString()
                CellType.STRING -> cell.stringCellValue
                CellType.NUMERIC -> { numericToString(cell) }
                CellType.ERROR -> cell.errorCellValue.toString()
                else -> throw IllegalStateException()
            }
        }
        CellType.ERROR -> cell.errorCellValue.toString()
        else -> throw IllegalStateException()
    }
}

private fun numericToString(cell: org.apache.poi.ss.usermodel.Cell): String {
    return if (DateUtil.isCellDateFormatted(cell)) cell.dateCellValue.toString() else cell.numericCellValue.toString()
}
```

### POI - メモリ使用量

まずはメモリの使用量から見てみましょう。
![POIのメモリ使用量](https://storage.googleapis.com/zenn-user-upload/aafc35f595ab-20231209.png)

`Workbook`を作成するところでめちゃめちゃメモリを使ってしまっています。
エクセルファイルのサイズが大きくなればここで使用するメモリは更に増えていくため、簡単にメモリ不足になる未来が想像できます。

### POI - CPU 時間

続いて CPU の使用時間を見てみましょう。
![POIのCPU使用時間](https://storage.googleapis.com/zenn-user-upload/32c28eb54db7-20231209.png)

こちらも同様に、`Workbook`の作成で時間をかなり使っており、エクセルファイルのサイズが大きくなればここで使用する CPU 時間も更に増えていくことになります。

## FastExcel - パフォーマンス

同様に、単純に全ての行、全ての列を文字列として取得するようなコードで試してみます。

```kotlin
private fun parseByFastExcel(inputStream: InputStream) {
    val workbook = ReadableWorkbook(inputStream, ReadingOptions(true, false))
    val sheet = workbook.firstSheet
    val stream = sheet.openStream()
    val data = stream.map {
        listOf(
            cellToString(it.getCell(0)),
            cellToString(it.getCell(1)),
            cellToString(it.getCell(2)),
            cellToString(it.getCell(3)),
            cellToString(it.getCell(4)),
            cellToString(it.getCell(5)),
            cellToString(it.getCell(6)),
            )
    }.toList()

    File("./output.json").writeText(jsonConverter.objectToJsonString(data))
}

private fun cellToString(cell: Cell?): String {
    if (cell == null) return ""
    if (cell.text.isNullOrBlank()) return ""
    return if (isDateCell(cell)) cell.asDate().format(DateTimeFormatter.ofPattern("yyyy/M/d")) else cell.text
}

private fun isDateCell(cell: Cell): Boolean {
    val dataFormatId = cell.dataFormatId ?: return false
    // 14..22は日付形式のID
    // 参考: https://www.iso.org/standard/71691.html
    if (dataFormatId in 14..22) {
        return true
    }
    val dataFormatString = cell.dataFormatString ?: return false
    // POIさんの関数を使わせて頂いております
    return DateUtil.isADateFormat(dataFormatId, dataFormatString)
}
```

### FastExcel - メモリ使用量

こちらもメモリ使用量から見てみましょう。
![FastExcelのメモリ使用量](https://storage.googleapis.com/zenn-user-upload/8223d85d01ff-20231209.png)

`Workbook` を作成するところに関しては、とんでもなく減っていますね。
また、`FastExcel`ではエクセルファイルのサイズは`Workbook`サイズと関係ないため、どれだけ大きなエクセルファイルであっても`Workbook`の作成で大量のメモリが消費されることはありません。

### FastExcel - CPU 時間

続いて CPU 時間です。
![FastExcelのCPU時間](https://storage.googleapis.com/zenn-user-upload/51d5974daf5f-20231209.png)

こちらもセルから値を取得する処理のところにボトルネックが移動しています。
また、全体の時間で見ても大幅に減っていることが確認できます。

# 今後の展望

今回の修正によって、エクセルファイルの読み込みと処理に関してのパフォーマンス改善はある程度達成されました。
しかし、実際に大量のデータを含むエクセルファイルの処理においては、全てのデータを一気に変数に保持する作りになっているため、まだメモリ不足の懸念は残っています。
チャンク形式で処理するなど、この辺りは今後対応していきたいところです。
また、書き込みの方も対応していこうと思っております。

# 最後に

`POI`から`FastExcel`への移行に関する今回の振り返りが、同様の課題に直面している方々の一助となれば幸いです。
興味を持っていただいた方はぜひ以下のページもご覧ください！

https://hrmos.co/pages/loglass/jobs/B0001
