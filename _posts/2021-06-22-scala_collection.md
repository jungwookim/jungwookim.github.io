---
layout: post
title:  "Scala Collections"
categories: scala
---

# Intro

Scala를 사용하면서 Collection에 대해서 많이 헷갈려서 정리를 좀 하고자 한다. 정적 언어로 개발을 안했다보니 익숙치 않는게 문제.

# Mutable and Immutable collections

Scala collections은 mutable/immutable한 컬렉션으로 나뉜다. mutable하다는 건 값을 바로 변경하거나 확장할 수 있다. 이 말은 값을 수정할 수 있다는 말이고, 이는 side effect가 발생할 수 있다는 말과 같다. 반대로, immutable한 것은 여전히 변경 가능하지만, return값으로 새로운 컬렉션을 준다. 즉, 이전 collection은 변하지 않는다.

모든 collection은 `scala.collection` 패키지 혹은 이것의 서브 패키지 중 하나인 `scala.collection.mutable` or `scala.collection.immutable` 패키지에서 찾을 수 있다. `mutable` 패키지는 다 mutable하고 `immutable` 패키지는 다 immutable하다.

`scala.collection`에 있는 컬렉션은 둘 중 하나가 될 수 있는데, 예를 들어 `collection.IndexedSeq[T]`는 `collection.immutable.IndexedSeq[T]`와 `collection.mutable.IndexedSeq[T]`의 슈퍼 클래스이다.

기본적으로 scala는 immutable 컬렉션을 선택한다.

아래는 `scala.collection` 패키지에 있는 컬렉션들이다. 이것들은 전부 high-level abstract classes or traits이며 mutable할수도, immutable 할 수도 있다.

![scala.collection](https://docs.scala-lang.org/resources/images/tour/collections-diagram-213.svg)
`scala.collection.immutable 패키지`
![scala.collection.immutable](https://docs.scala-lang.org/resources/images/tour/collections-immutable-diagram-213.svg)
`scala.collection.mutable 패키지`
![scala.collection.mutable](https://docs.scala-lang.org/resources/images/tour/collections-mutable-diagram-213.svg)
그래프 읽는 법
![legend]https://docs.scala-lang.org/resources/images/tour/collections-legend-diagram.svg

collection api들은 대부분 위 그림에서 보이는 이름으로 바로 선언해서 쓸 수 있다.
```scala
Iterable("x", "y", "z")
Map("x" -> 24, "y" -> 25, "z" -> 26)
Set(Color.red, Color.green, Color.blue)
SortedSet("hello", "world")
Buffer(x, y, z)
IndexedSeq(1.0, 2.0)
LinearSeq(a, b, c)
```

그리고 api들의 결과 타입은 콜렉션 그 타입을 그대로 가진다. 이를 `uniform return type principle`이라고 한다. 대부분의 컬렉션 클래스는 root, mutable, immutable 중 하나인데, 단 하나 `Buffer`의 경우에만 mutable collection을 가진다.

## Reference

- [Scala Collection][scala_collection]

[scala_collection]: https://docs.scala-lang.org/overviews/collections-2.13/overview.html
