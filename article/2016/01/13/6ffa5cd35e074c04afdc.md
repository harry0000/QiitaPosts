---
title: SQLのResultSetを特定のキーでグルーピングしたJSONで返す方法
tags: Scala:2.11.7 PlayFramework:2.4.6 Anorm:2.5.0
author: harry0000
slide: false
---
ドーモ、Scala歴3週間エンジニアです。
実際、初歩的なことすぎてググっても見つからず爆発四散。

# どういうこと？

下記のようにSQLで取得した日次の集計データ(レコード)を特定のキーでグルーピングしたようなJSONで返したいことないですか？[^1]

**SQLの結果**

```
 code | date       | value
------+------------+-------
 foo  | 2015-12-01 |  100
 foo  | 2015-12-02 |  105
 foo  | 2015-12-03 |   90
 bar  | 2015-12-01 |  200
 bar  | 2015-12-02 |  250
 bar  | 2015-12-03 |  230
 ・
 ・
 ・
```

**返したいJSON**

```json
[
  {
    "code": "foo",
    "list": [
      { "date": "2015-12-01", "value": 100 },
      { "date": "2015-12-02", "value": 105 },
      { "date": "2015-12-03", "value":  90 }
    ]
  },
  {
    "code": "bar",
    "list": [
      { "date": "2015-12-01", "value": 200 },
      { "date": "2015-12-02", "value": 250 },
      { "date": "2015-12-03", "value": 230 }
    ]
  },
  ...
]
```

**Model**

```Summary.scala
case class Summary(
  code: String,
  list: Seq[(LocalDate, Option[Int])]
)
```

日時はUNIX epoch(ミリ秒)で扱うことが多いと思いますが、わかりやすさのためISO 8601形式(`org.joda.time.LocalDate`)にしています。

# レコードからオブジェクトへの変換

## とりあえず処理を考える

`groupBy()`というその名もズバリなメソッドがあるので、これを使えばcode毎にデータをまとめられそうな気がしてきます。

```scala
  val rows: List[(String, LocalDate, Option[Int])]

  rows.groupBy { case (code, _, _) => code }
      .map { case (code, rows) =>
        Summary(
          code,
          rows.map { case (_, date, value) => (date, value) }
        )
      }
      .toSeq
```

やったか！？

ですが[`TraversableLike.groupBy()`](https://github.com/scala/scala/blob/v2.11.7/src/library/scala/collection/TraversableLike.scala#L329-L341)は`immutable.Map`を使用するため、キーの順序、つまり今回の例でいうcodeの並び順が保証されません。[^2]
そのため、SQLで`order by`した順でデータを返したい場合には使えません。

キーの順序は維持したいので`mutable.LinkedHashMap`を使えば良さそうです。

```scala
  val rows: List[(String, LocalDate, Option[Int])]

  val m = mutable.LinkedHashMap.empty[String, mutable.Buffer[(String, LocalDate, Option[Int])]]

  rows.foreach {
    case (code, _, _) =>
      m.getOrElseUpdate(code, mutable.Buffer.empty) += row
  }

  m.map {
    case (code, rows) =>
      Summary(
        code,
        rows.map { case (_, date_ value) => (date, value) }
      )
  }.toSeq
```

## 処理を汎用化してみる

レコードをグルーピングしてMapで返す処理は別の所でも使うかもしれませんので、汎用化について考えてみます。

### 方法1: Package objectにメソッドを作る

```package.scala
import scala.collection.mutable

package object models {

  def groupBy[A, K, V](rows: Traversable[A])(f: A => (K, V)): mutable.LinkedHashMap[K, mutable.Buffer[V]] = {
    val m = mutable.LinkedHashMap.empty[K, mutable.Buffer[V]]
    for (row <- rows) {
      val (k, v) = f(row)
      m.getOrElseUpdate(k, mutable.Buffer.empty) += v
    }
    m
  }

}
```

できあがっているものがこちらになります。（3分クッキング）

グルーピングしたい`rows: Traversable[A]`とグルーピングのKeyとValueを生成する関数`f: A => (K, V)`を渡すと、`LinkedHashMap[K, Buffer[V]]`を返します。

`rows`と`f`をカリー化しているのが気になるかもしれませんが、これは2つのParameter listに分けることで`row`の型`A`を確定させてから、`f: A => (K, V)`の型推論をさせるためです。

**参考URL**
[Scalaの型推論について - Togetterまとめ](http://togetter.com/li/219702)

**参考書籍**
[Scalaスケーラブルプログラミング第2版](http://book.impress.co.jp/books/3084)
16.10 Scalaの型推論アルゴリズムを理解する (P.314-317)

[Scalaパズル 36の罠から学ぶベストプラクティス](http://www.shoeisha.co.jp/book/detail/9784798145037)
PUZZLE 6　引数でもうドン引き!――Arg Arrgh! (P.47-52)

### 方法2: Extension method(拡張メソッド)を定義する

Scalaは[Implicit Class](http://docs.scala-lang.org/sips/completed/implicit-classes.html)で既存クラスなどのメソッドを拡張(追加)することができます。これを利用して`Traversable`に`LikedHashMap`にグルーピングする`groupByOrdered()`メソッドを追加します。

```scala
  implicit class GroupByOrderedImpl[A](val s: Traversable[A]) extends AnyVal {

    def groupByOrdered[K, V](f: A => (K, V)): mutable.Map[K, mutable.Buffer[V]] = {
      val m = mutable.LinkedHashMap.empty[K, mutable.Buffer[V]]
      for (i <- s) {
        val (k, v) = f(i)
        m.getOrElseUpdate(k, mutable.Buffer.empty) += v
      }
      m
    }

  }
```

```scala
  val rows: List[(String, LocalDate, Option[Int])]

  rows.groupByOrdered {
        case (code, date, value) => code -> (date, value)
      }.map {
        case (code, rows) => Summary(code, rows)
      }.toSeq
```

`Traversable`が拡張され`groupByOrdered()`が呼び出せるようになりました。
ですが、個人的に今回やりたいことに対して方法が大げさすぎる気がしたため、採用は見送りました。

**参考URL**
[collections - Scala GroupBy preserving insertion order? - Stack Overflow](http://stackoverflow.com/a/9608800/4366193)
[Value Classes and Universal Traits - Scala Documentation](http://docs.scala-lang.org/overviews/core/value-classes.html#extension-methods)

# オブジェクトからJSONへの変換

DBから取得したレコードをcode毎にまとめたオブジェクトで返す`Summary.find(from: LocalDate, to: LocalDate)`が実装できました。次は`SummaryController`を実装してクライアントにJSONを返す処理を実装します。

**URLのイメージ**

```
http://example.com/summary?from=1448895600000&to=1451487600000

from: 2015-12-01 00:00:00.000
to  : 2015-12-31 00:00:00.000
```

**routes**

```
GET     /summary                    controllers.SummaryController.findSummary(from: Long, to: Long)
```

**SummaryController.scala**

```scala
class SummaryController extends Controller {

  def findSummary(from: Long, to: Long) = Action {
    val list = Summary.find(new LocalDate(from), new LocalDate(to))

    if (list.isEmpty) NotFound
    else Ok(Json.toJson(list))
  }

}
```

[`Json.toJson()`](https://github.com/playframework/playframework/blob/2.4.6/framework/src/play-json/src/main/scala/play/api/libs/json/Json.scala#L113-L118)は引数の型に応じた暗黙の[Writes](https://github.com/playframework/playframework/blob/2.4.6/framework/src/play-json/src/main/scala/play/api/libs/json/Writes.scala#L32)を使用して`JsValue`を返します。基本的な型については[`DefaultWrites `](https://github.com/playframework/playframework/blob/2.4.6/framework/src/play-json/src/main/scala/play/api/libs/json/Writes.scala#L106)で定義されています。[^3]
`app/models/Summary.scala`に`Summary`のための`Writes`を定義しておきましょう。

```scala
  implicit val summaryWrites = new Writes[Summary] {
    override def writes(s: Summary): JsValue = Json.obj(
      "code" -> s.code,
      "list" -> s.list.map {
        case (date, value) => Json.obj(
          "date" -> date,
          "value" -> value
        )
      }
    )
  }
```

`value`は`Option[Int]`型なのでnullの可能性がありますが、nullの時に0を返したい場合は`value.getOrElse[Int](0)`と書きます。[^4]

これでできた（コマンドー）

# ソースコード一覧

import文は省略してますが、今回のソースコードを列挙しておきます。

**app/models/Summary.scala**

```scala
case class Summary(
    code: String,
    list: Seq[(LocalDate, Option[Int])]
    )

object Summary {

  private val simple = {
    get[String]("code") ~
    get[LocalDate]("date") ~
    get[Option[Int]]("value") map {
      case code ~ date ~ value => (code, date, value)
    }
  }

  implicit val summaryWrites = new Writes[Summary] {
    override def writes(s: Summary): JsValue = Json.obj(
      "code" -> s.code,
      "list" -> s.list.map {
        case (date, value) => Json.obj(
          "date" -> date,
          "value" -> value
        )
      }
    )
  }

  def find(from: LocalDate, to: LocalDate): Seq[Summary] = {
    val rows = DB.withTransaction { implicit conn =>
      SQL("""
        |SELECT code, date, SUM(value) AS value
        |FROM summary
        |WHERE date BETWEEN {from} AND {to}
        |GROUP BY code, date
        |ORDER BY code, date
        |""".stripMargin
      ).on(
        'from -> from.toDate,
        'to -> to.toDate
      ).as(simple.*)
    }

    models.groupBy(rows) {
            case (code, date, value) => code -> (date, value)
          }.map {
            case (code, rows) => Summary(code, rows)
          }.toSeq
  }

}
```

**app/models/package.scala**

```scala
import scala.collection.mutable

package object models {

  def groupBy[A, K, V](rows: Traversable[A])(f: A => (K, V)): mutable.LinkedHashMap[K, mutable.Buffer[V]] = {
    val m = mutable.LinkedHashMap.empty[K, mutable.Buffer[V]]
    for (row <- rows) {
      val (k, v) = f(row)
      m.getOrElseUpdate(k, mutable.Buffer.empty) += v
    }
    m
  }

}
```

**app/controllers/SummaryController.scla**

```scala
class SummaryController extends Controller {

  def findSummary(from: Long, to: Long) = Action {
    val list = Summary.find(new LocalDate(from), new LocalDate(to))

    if (list.isEmpty) NotFound
    else Ok(Json.toJson(list))
  }

}
```

**conf/routes**

```
GET     /summary                    controllers.SummaryController.findSummary(from: Long, to: Long)
```

-----
[^1]: あ、無い。それをちょっとやってもらうから。
[^2]: 値の順序はイテレーターでループし追加されるので保証(維持)されます。
[^3]: [`Traversable`](https://github.com/playframework/playframework/blob/2.4.6/framework/src/play-json/src/main/scala/play/api/libs/json/Writes.scala#L193-L198)も含まれています。`Seq`などに対する`Writes`を定義しなくてよいのはこのためです。
[^4]: 型を明示的に指定しないと、実行時に例外が発生して怒られます。
