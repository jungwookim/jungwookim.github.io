---
layout: post
title:  "clojure와 친해지기"
categories: db lock go transaction
---


`clojure`와 친해져보도록 하자. `clojure`가 `JVM` 위에서 동작한다는 성질이나 함수형 언어로서의 특성을 글로 배우기 보다는 기본기를 `advent of code`에서 닦아보도록 하자. 전체 코드는 [Github][github]에 다 올려놓았다.

# Day 1
AOC 2018년 Day1으로 시작.

## 파일 읽고 파싱하기
text file을 읽어서 문제를 풀다보니 파일 입출력을 해야했다. line by line으로 읽고 싶었지만 문제의 의도와 거리가 있어서 `clojure.core/slurp`를 사용했다.

```clojure
(slurp "resources/input.txt") ; 간단하다
```

아주 간단하다. 읽은 결과는 string이므로 splitting을 잘 해준다.

```clojure
(clojure.string/split #"\n") ; delimiter는 줄바꿈으로 했다
```

아래와 같은 방식으로 string을 Integer로 파싱할 수 있다. java 라이브러리를 사용하는 것을 확인하면 된다.
```clojure
(Integer/parseInt "+3")
```

## part 1
문제는 간단하다. Interger collecitons를 받아서 합을 구하면 된다. `apply`나 `reduce`를 이용하면 쉽게 구현할 수 있다.

## part 2
주어진 Integer collections가 무한히 반복된다고 할 때, 같은 누적합이 발견될 때의 누적합을 반환하면 된다.

### 풀이
먼저 `repeat`을 알고 있었고 collections를 `repeat`하면 collections of collections이기 때문에 `flatten`을 사용했었다.

```clojure
(defn solve-part2 [li]
  (let [inf-seq (flatten (repeat li))]
    (loop [temp-set #{}
           cur-seq inf-seq
           acc 0]
      (if (temp-set (+ acc (first cur-seq)))
        (+ acc (first cur-seq))
        (recur (conj temp-set (+ acc (first cur-seq)))
               (rest cur-seq)
               (+ acc (first cur-seq)))))))
```

### 리뷰
- flatten + repeat -> cycle로 바꾸면 좋다. (lazy한 특성 때문에 둘 다 사용 가능한 점을 인지하자)
- let을 사용하는게 오히려 안좋지 않을까? 라고 생각했었는데 쓰는게 더 나음
- loop-recur가 clojure에서 좀 쓸만한 녀석인 줄 알고 남발했었는데 anti-pattern이라고 한다. 앞으로는 사용을 자제해보자
- 문제의 특성상 같은 누적합이 2번 나오면 그게 정답이다. 누적합은 `clojure.core/reductions`를 사용하면 쉽게 구할 수 있다.
- docString으로 input, output의 형태나 함수 설명을 해주도록 노력하자.

### reductions를 이용한 더 나은 코드
```clojure
(defn solve-part2-advanced [li]
  (let [inf-partial-sum li]
    (loop [temp-set #{}
           cur-seq inf-partial-sum]
      (let [sum (first cur-seq)]
        (if (temp-set sum)
          sum
          (recur (conj temp-set sum)
                 (rest cur-seq)))))))

(->> "resources/input.txt"
      parse-input
      cycle
      (reductions +) ; reductions의 결과인 partial-sum을 파라미터로 넘겼다.
      solve-part2-advanced)
```


## Reference
- [my github repo][github]
- [clojure library documents][clojure_doc]


[github]:https://github.com/jungwookim/aoc-exercise/tree/master/src
[clojure_doc]:https://clojuredocs.org/clojure.core