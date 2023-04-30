Title: Scala and Spark
Date: 2021-05-25
Tags: Scala, Spark
---

# Intro

스칼라는 객체 지향 언어와 함수형 언어의 특징을 합쳐 놓은 거라고 함. 자바처럼 JVM(자바가상머신) 위에서 동작함. 

# 쌩 기본

## `Hello world`를 띄워보자

```scala
// way 1 싱글톤 오브젝트의 main 함수 구현
object HelloWorldObject {
    def main(ars: Array[String]): Unit = {
        println("Hello World main")
    }
}

// way2: app 트레잇 상속
object HelloWorld extends App {
    println("Hello World")
}
```

## 객체, 자료형, 문자열, 변수, 함수, 클래스, 트레잇, 싱글톤 객체, 콜렉션

스칼라는 기본 자료형, 함수, 클래스 등 모든 것을 객체로 취급함. Any를 최상위로 해서 AnyVal(Double, Float, Long, Int, Short, Byte, Unit, Boolean, Char), AnyRef(List, Option, Class)가 있음.

객체 비교는 `==` 아니면 `!=`를 사용함.

자료형, 문자열은 문서 찾아보자.

변수는 재할당 가능한 `var`와 재할당 불가능한 `val`이 있음. 가급적 `val`을 써서 동시성 처리에 유용하도록 하자.

함수는 `def`로 선언. 리턴과 리턴 타입은 생략 가능. 파라미터 타입은 생략 불가. 리턴 값이 없으면 `Unit` 이용하거나 리턴 값이 없으면 마지막 값을 리턴. 함수의 매개변수는 불변 변수(`val`)이기 때문에 재할당 불가. 리턴 타입 생략하면 자동으로 추론 함.

```scala
// 함수 선언 
def add(x: Int, y: Int): Int = {
  return x + y
}

// x는 val 이기 때문에 변경 불가 
def add(x: Int): Int = {
  x = 10 
}

// 리턴 타입 생략 가능 
def add(x: Int, y: Double) = {
  x + y
}

// 리턴 타입이 Unit 타입도 생략 가능 
def add(x: Int, y: Int) = {
  println(x + y)
}

// 리턴 데이터가 없는 경우 Unit을 선언  
def add(x: Int, y: Int): Unit = {
  println(x + y)
}
```

가변 길이 파라미터는 파라미터 입력시 `*`를 이용하면 `Seq`형으로 변환되어 입력.
```scala
// 여러개의 Int 형을 입력받아서 합계 계산 
def sum(num:Int*) = num.reduce(_ + _)

scala> sum(1, 2, 3)
res22: Int = 6

scala> sum(1)
res23: Int = 1
```

함수
- 람다함수


```scala
// exec는 3개의 파라미터(함수 f, x, y)를 받음
def exec(f: (Int, Int) => Int, x: Int, y: Int) = f(x, y)

// 람다 함수를 전달하여 처리. x+y 작업을 하는 함수 전달 
scala> exec((x: Int, y: Int) => x + y, 2, 3)
res12: Int = 5

// 선언시에 타입을 입력해서 추가적인 설정 없이 처리 가능 
scala> exec((x, y) => x + y, 7, 3)
res13: Int = 10

// 함수에 따라 다른 처리 
scala> exec((x, y) => x - y, 7, 3)
res14: Int = 4

// 언더바를 이용하여 묵시적인 처리도 가능 
scala> exec(_ + _, 3, 1)
res15: Int = 4
```


- 커링
- 클로저
- 타입(Generic)


```scala
// for class
import scala.collection.mutable

trait TestStack[T] {
  def pop():T
  def push(value:T)
}

class StackSample[T] extends TestStack[T] {

  val stack = new scala.collection.mutable.Stack[T]

  override def pop(): T = {
    stack.pop()
  }

  override def push(value:T) = {
    stack.push(value)
  }
}

val s = new StackSample[String]

s.push("1")
s.push("2")
s.push("3")

scala> println(s.pop())
3
scala> println(s.pop())
2
scala> println(s.pop())
1

// for method
def sample[K](key:K) {
  println(key)
}

def sample2 = sample[String] _

scala> sample2("Hello")
Hello
```


트레잇(Trait)
- 자바나 nodejs의 interface와 유사함.


```scala
// Machine 트레잇 
trait Machine {
  val serialNumber: Int = 1
  def work(message: String)
}

// KrMachine 트레잇 
trait KrMachine {
  var conturyCode: String = "kr"
  def print() = println("한글 출력")    // krprint 기본 구현 
}

// Computer 클래스는 Machine, KrMachine를 둘다 구현합니다. 
class Computer(location: String) extends Machine with KrMachine {
  this.conturyCode = "us"   // code 값 변경 
  def work(message: String) = println(message)
}

// Car 클래스는 Machine, KrMachine를 둘다 구현합니다.
class Car(location: String) extends Machine with KrMachine {
  def work(message: String) = println(message)
  override def print() = println("운전중입니다.") // print 재정의
}

var machine = new Computer("노트북")
var car = new Car("포르쉐")

scala> machine.work("computing...")
computing...

scala> machine.print()
한글 출력

scala> println(machine.conturyCode)
us

scala> car.work("driving...")
driving...

scala> car.print() 
운전중입니다.

scala> println(car.conturyCode)
kr
```

클래스 - 그냥 클래스스럽다.

## Spark

인메모리 기반의 대용량 데이터 고속 처리 엔진, 범용 분산 클러스터 컴퓨팅 프레임워크. 스파크는 작업을 관리하는 드라이버와, 작업이 실행되는 노드를 관리하는 클러스터 매니저 - 크게 두가지로 구성되어있음.

스파크 app 구현 방법은 RDD를 이용하는 방법과, Dataset, DataFrame을 이용하는 방법이 있는데, 주로 DataFrame을 이용하는 방법을 알아보자.

### Spark Sessin init

Dataset, DataFrame은 스파크 세션을 이용하여 처리 함.

```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession
  .builder()
  .appName("Spark SQL basic example")
  .config("spark.some.config.option", "some-value")
  .getOrCreate()
```

### DataFrame init

DataFrame은 스파크세션의 `read` method로 생성할 수 있음. `read`는 json, parquet, orc, text 등 다양한 형식의 데이터를 읽을 수 있음. 자세한 건 [API 문서](read_dfreader_api)를 확인.

### DataFrame 연산

#### 스키마 확인
`printSchema`를 이용함.
```scala
val df = spark.read.json("/user/people.json")

// 스키마 출력 
scala> df.printSchema()
root
 |-- age: long (nullable = true)
 |-- name: string (nullable = true)

scala> df.show()
+----+-------+
| age|   name|
+----+-------+
|null|Michael|
|  30|   Andy|
|  19| Justin|
+----+-------+
```

#### 조회
`select`
```scala
// name 칼럼만 조회 
scala> df.select("name").show()
+-------+
|   name|
+-------+
|Michael|
|   Andy|
| Justin|
+-------+

// name, age 순으로 age에 값을 1더하여 조회 
scala> df.select($"name", $"age" + 1).show()
+-------+---------+
|   name|(age + 1)|
+-------+---------+
|Michael|     null|
|   Andy|       31|
| Justin|       20|
+-------+---------+
```

#### show() 함수 설정
데이터의 길이와 칼럼의 사이즈를 제한하여 출력할 수 있음
```scala
// show 함수 선언 
def show(numRows: Int, truncate: Boolean): Unit = println(showString(numRows, truncate))

// 사용방법 
scala> show(10, false)
scala> show(100, true)
```

#### 필터링
`filter`
```scala
// 필터링 처리 
scala> df.filter($"age" > 21).show()
+---+----+
|age|name|
+---+----+
| 30|Andy|
+---+----+

// select에 filter 조건 추가 
scala> df.select($"name", $"age").filter($"age" > 20).show()
scala> df.select($"name", $"age").filter("age > 20").show()
+----+---+
|name|age|
+----+---+
|Andy| 30|
+----+---+
```

#### 그룹핑
`groupBy`
```scala
// 그룹핑 처리 
scala> df.groupBy("age").count().show()
+----+-----+
| age|count|
+----+-----+
|  19|    1|
|null|    1|
|  30|    1|
+----+-----+
```

#### 칼럼 추가
새로운 칼럼을 추가할 때는 `withColumn`을 이용
```scala
// age가 NULL일 때는 KKK, 값이 있을 때는 TTT를 출력 
scala> df.withColumn("xx", when($"age".isNull, "KKK").otherwise("TTT")).show()
+----+-------+---+
| age|   name| xx|
+----+-------+---+
|null|Michael|KKK|
|  30|   Andy|TTT|
|  19| Justin|TTT|
+----+-------+---+
```

#### DataFrame의 SQL을 이용한 데이터 조회
데이터프레임은 SQL 쿼리를 이용해 데이터를 조회할 수 있음. 데이터프레임을 이용하여 뷰를 생성하고 SQL쿼리를 실행하면 됨.

```scala
// 뷰생성
val df = spark.read.json("/user/people.json")

// DataFrame으로 뷰를 생성 
df.createOrReplaceTempView("people")

// 스파크세션을 이용하여 SQL 쿼리 작성 
scala> spark.sql("SELECT * FROM people").show()
+----+-------+
| age|   name|
+----+-------+
|null|Michael|
|  30|   Andy|
|  19| Justin|
+----+-------+

// SQL 사용
// 조회 조건 추가 
scala> spark.sql("SELECT * FROM people WHERE age > 20").show()
+---+----+
|age|name|
+---+----+
| 30|Andy|
+---+----+

// 그룹핑 추가 
scala> spark.sql("SELECT age, count(1) FROM people GROUP BY age").show()
+----+--------+
| age|count(1)|
+----+--------+
|  19|       1|
|null|       1|
|  30|       1|
+----+--------+
```

### DataFrame 저장/불러오기
Dataset이랑 DataFrame이랑 같음.

#### 데이터 저장
`save` 이용
```scala
val peopleDF = spark.read.json("/user/people.json")
case class People(name: String, age: Long)
val peopleDS = peopleDF.as[People]


peopleDS.write.save("/user/ds")
peopleDF.write.save("/user/df")
```

#### 저장 포맷 지정
`format`을 이용해 json, csv 등 포맷을 설정할 수 있음
```scala
// 데이터 저장 
peopleDS.select("name").write.format("json").save("/user/ds_1")

// 저장 위치 확인 
$ hadoop fs -ls /user/ds_1/
Found 2 items
-rw-r--r--   2 hadoop hadoop          0 2019-01-24 07:19 /user/ds_1/_SUCCESS
-rw-r--r--   2 hadoop hadoop         53 2019-01-24 07:19 /user/ds_1/part-r-00000-88b715ad-1b5b-480c-8e17-7b0c0ea93e9f.json

// 저장 형식 확인 
$ hadoop fs -text /user/ds_1/part-r-00000-88b715ad-1b5b-480c-8e17-7b0c0ea93e9f.json
{"name":"Michael"}
{"name":"Andy"}
{"name":"Justin"}
```

#### 압축 포맷 지정
`option`을 이용하여 압축 포맷을 지정할 수 있음. gzip, snappy 등의 형식을 이용할 수 있음. 
```scala
// snappy 형식 압축 
peopleDS.select("name").write.format("json").option("compression", "snappy").save("/user/ds_1")

// 저장 모드, SaveMode.ErrorIfExists, SaveMode.Append, SaveMode.Overwrite, SaveMode.Ignore 등의 모드가 있음
// SaveMode 사용을 위해 import
import org.apache.spark.sql._

val peopleDF = spark.read.json("/user/people.json")
peopleDF.select("name", "age").write.mode(SaveMode.Overwrite).save("/user/people/")
```

#### 테이블 저장
`saveAsTable`을 이용합니다.
```scala
peopleDF.select("name", "age").write.saveAsTable("people")
```

#### 데이터 불러오기
```scala
val peopleDF = spark.read.json("/user/ds_1/")
scala> peopleDF.show()
+-------+
|   name|
+-------+
|Michael|
|   Andy|
| Justin|
+-------+
```

## Spark SQL and Hive
- Spark SQL
- UDF(User Defined Function)

## Reference

- [read_dfreader_api][read_dfreader_api]
- [book][book]

[read_dfreader_api]: https://spark.apache.org/docs/2.3.0/api/java/org/apache/spark/sql/DataFrameReader.html
[book]: https://wikidocs.net/book/2350
