---
layout: post
title:  "clojure와 친해지기"
categories: clojure adventofcode
---


`clojure`와 친해져보도록 하자. `clojure`가 `JVM` 위에서 동작한다는 성질이나 함수형 언어로서의 특성을 글로 배우기 보다는 기본기를 [`advent of code`][advent_of_code]에서 닦아보도록 하자. 전체 코드는 [Github][github]에 다 올려놓았다.

# Day 1
AOC 2018년 Day1으로 시작. 아주 간단한 로직을 다루고 숫자 관련한 조작을 연습해보는 섹션이 아닌가 싶다.

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

# Day 2

어제 삽질한 파일 읽는 것 때문에 시간은 잘 벌었다. `string`을 잘 다루어보는 섹션인 것 같다. 근데 `laySeq`의 결과를 `contains?`와 함께 사용할 수 없고 map 데이터구조를 핸들링하는 것에 익숙치 않아서 시간이 오래 걸린 것 같다. 그리고 원래 string 관련 문제는 좀 귀찮은 면은 있는 것 같긴 하다. AOC 2018년 Day 2를 보자.

## part 1
결과부터 말하자면 `frequencies + char-array`를 사용하면 아주 쉽게 풀리는 문제였는데 저 2가지를 직접 구현하면서 겪은 각종 삽질들 때문에 시간을 많이 잡아 먹었다. 그리고 아래와 같은 코드를 빨리 구현하지 못해서 엄청 헤맸다. 어떻게 삽질 했는지 같이 살펴보자.

```typescript
// 조건을 만족할 때만 do sth을 하고 그 조건이 여러개일 때
if (isValid()) {
    // do sth...
}

if (isValid2()) {
    // do sth...
}

```

이라는 간단한 걸 구현하고 싶었는데 클로저에서는 이렇게 안된다.
```clojure
(when (isValid) (...)
when (isValid2) (...)) ; 난감했다
```

사실 expression자체도 지금 생각하면 안좋다는게 보이지만 아직은 적응이 덜 됐는지 위에 처럼 구현하고 싶은 마음이 처음엔 컸다. 여튼, 그래서 처음에 답은 구했는데 아주 아주 돌아갔다.

```clojure
; 처음 제출한 답의 메인 로직
(defn calc-freq
  "input: [[1 2 3 4] [2 2 0 0] ...]"
  [li]
  (apply * (reduce (fn [[res1 res2] val]
                     (cond
                       (and (some #(== % 3) val) (some #(== % 2) val)) [(inc res1) (inc res2)]
                       (some #(== % 3) val) [res1 (inc res2)]
                       (some #(== % 2) val) [(inc res1) res2]
                       :else [res1 res2])) [0 0] li)))
```

## 리뷰
- reduce는 almighty해서 가급적 안쓰는게 좋다. 쓰더라도 뚱뚱하게 쓰면 안된다.
- 고차함수인 map, filter 같은 것만 사용해서 다 구현할 수 있다 (대부분): 이 사실을 명심하자
- contains?는 set에서만 쓰고 map, vector, list에서는 다른 방식으로 contains? 유무를 확인하자
- `frequencies`, `char-array`를 알자

## 리뷰 반영 후 다시 구현해본 코드
이게 베스트인지는 모르겠지만 일단 조금 더 가독성이 높아졌다.
```clojure
(defn logic-part1 [n str-li]
  (->> str-li
       (map frequencies)
       (map vals)
       (filter #(some (fn [x] (== x n)) %))
       count))

(defn solve-part1 [path]
  (let [res1 (->> path
                  read-input
                  (logic-part1 2))
        res2 (->> path
                  read-input
                  (logic-part1 3))]
    (* res1 res2)))
```

## part 2

# Day 3

`matrix`를 다루는 내용이 목적인 것 같다. 2d 리스트나 벡터를 사용하는 것을 다루어보자는 느낌.

## Intro
part 1과 part 2가 거의 한번에 풀릴 수 있을 것처럼 보였다. 전체 색칠한 `matrix`를 만들어내고 이 결과를 가지고 두번 색칠해졌는지 아니면 해당 id가 한번도 덮어씌어진 적 없는지 확인하면 됐었다.

## parsing
input이 좀 괴상하게 들어와서 이것도 연습하는 건가? 싶었다. `#1 1,3: 4x4` 이런 식으로 인풋이 들어오는데 이걸 잘 파싱해보도록 하자. 위에서 사용한 `clojure.string/split`과 `subs`를 요리조리 사용했다.

```clojure
(defn read-input [path]
  (-> path
      slurp
      (clojure.string/split #"\n")))

(defn parse-id
  "input: \"#44\", output: 44"
  [string]
  (Integer/parseInt (subs string 1 (count string))))

(defn parse-pos
  "input: \"1,3:\", output: (1 3)"
  [string]
  (->> (clojure.string/split (subs string 0 (dec (count string))) #",")
       (mapv (fn [x] (Integer/parseInt x)))))

(defn parse-size
  "input: \"4x4\", output: (4 4)"
  [string]
  (->> (clojure.string/split string #"x")
       (mapv (fn [x] (Integer/parseInt x)))))

(defn prepare-data [path]
  (->> path
       read-input
       (mapv (fn [x] (clojure.string/split x #" ")))
       (mapv (fn [[id _ pos size]]
               (list (parse-id id) (parse-pos pos) (parse-size size))))))
```
id, position, size를 위와 같은 방식으로 파씽했었다.

## part 1
주요 아이디어는 `#99 1,3: 2x2`라는 인풋이 있을 때 `(1 3 99)`, `(1 4 99)`, `(2 3 99)`, `(2 4 99)` 의 collections을 가지고 있으면 x, y index에 원하는 id를 갱신하면 된다고 생각했다. 초기값을 다 -1 로 matrix를 만들어 놓았으므로 id가 양수임을 가정했을 때 갱신하려고 할 때 -1이 아니라면 (갱신된 적 있다면) 0으로 갱신해서 (문제에서 X랑 같음) 중복 칠한게 있는지 확인했다. 그리고 마지막으로 0의 갯수를 세어주면서 마무리하였다.

코드는 아래와 같다.
```clojure
(defn vec2d
  "2d vectors"
  [sx sy f]
  (mapv (fn [x] (mapv (fn [y] (f x y)) (range sx))) (range sy)))

(defn matrix
  "init matrix"
  []
  (vec2d 2000 2000 (constantly -1)))

(defn update-cell
  "update cell to id at position (x, y) on matrix"
  [matrix id x y]
  (if (neg? (get-in matrix [x y]))
    (update-in matrix [x y] (constantly id))
    (update-in matrix [x y] (constantly 0))))

(defn gen-modified-vals
  "output: sequence of (target-x target-y id)"
  [id ix iy sx sy]
  (map (fn [x] (map (fn [y] (list (+ iy x) (+ ix y) id)) (range sx))) (range sy)))

(defn flatten-vals
  "flatten and partition 3"
  [values]
  (->> values
       flatten
       (partition 3)))

(defn logic-part1
  "update each one"
  [data]
  (reduce (fn [acc [x y id]]
            (update-cell acc id x y))
          (matrix)                                          ; 초기값
          (flatten-vals (map (fn [[id [ix iy] [sx sy]]]
                               (gen-modified-vals id ix iy sx sy))
                             data))))

(defn solve-part1 [path]
  (->> path
       prepare-data
       logic-part1
       flatten
       (filter #(zero? %))
       count))
```

get-in, update-in 같은 nested structure에 사용하는 core 함수를 알게 되었다. 이렇게 푸는게 맞나? 그리고 flatten 왠만해서 안쓰려고 했는데 쓰면 편해서 써버렸다.

## part 2
part 1을 풀어서 쉽게 풀었다. 필터만 잘 하면 된다.

```clojure
(defn get-total-count-by-id [path]
  (->> path
       prepare-data
       (map (fn [[id _ [sx sy]]] (list id (* sx sy))))))

(defn logic-part2
  [path data]
  (let [current-id-count (->> data
                              logic-part1
                              flatten
                              (filter #(pos? %))
                              frequencies)

        total-count-by-id (get-total-count-by-id path)]
    (-> (filter (fn [[x y]] (== y (get current-id-count x -1))) total-count-by-id)
        first
        first)))

(defn solve-part2 [path]
  (->> path
       prepare-data
       (logic-part2 path)))
```

## 회고
언제나 그렇듯 2차 배열을 다루는 문제들은 약간 까다로운 면은 있다. 자주 풀어야 좀 익숙해지는 느낌. 그리고 immutable하게 짠다는게 문제 풀이에서 꽤 까다롭다고 느껴졌다.

## 리뷰
- 전반적으로 naming을 더 신경쓰면 좋을 것 같다는 의견
- naming을 할 때 함수 이름만 보고 뭐하는 건지 알 수 있도록, 함수 이름이 짓기 어렵다고 느꼈다면? 함수로 꼭 만들지 않아도 되는게 아닐까? 라는 생각도 해보자
- matrix 만들 때 2중 map도 좋지만 2중 for를 쓰면 좀 더 나을 수도
- 복잡한 parsing을 할 때는 정규식을 쓰는 것도 방법 (항상 좋은 것은 아니지만 경우에 따라)
- `vector`는 random access가 필요할 때 쓰긴 하지만 `list`보다는 `vector`를 주로 사용해도 좋음
- 그리고 `vector`나 `list`를 사용하더라도 순서에 따른 context가 있는 경우에는 `hash-map`을 쓰는게 훨씬 좋음
- matrix 초기활 때 사용한 -1, 0 같은 값들은 keyword로 관리하는 것이 좋음
- `==`와 `=` 차이: `==`는 타입은 신경 쓰지 않고 값만 비교, `=`는 타입과 값 모두 비교
- ex. `(= 0.2 1/5) ; false (== 0.2 1/5) ; true`
- 관용적으로... `->`의 경우 hash-map이나 string에 쓰고 `->>` seq를 다룰 때 많이 씀 (생각해보기)
- `first + first` 는 `ffirst`로

## 리뷰 반영 후 전체적인 코드
길이가 길어서 링크로 대체한다. 2중 for 문이 2d 다루는데 꽤나 편리한 것 같아서 유용했고 javascript의 object처럼 hash-map을 주요하게 사용하니 편했다. 이게 ps에서는 혼자 빨리 푸는게 중요해서 신경 안썼는데 부트캠프이니만큼 가독성도 신경 써야겠다.

---
# 쉬어가기
3일정도 리뷰해보니 코드를 대충 어떻게 작성해야하는지 약간 감은 왔다. 물론 뒤에도 더 많은 리뷰가 있겠지만 뒷 부분에는 코드는 [github 링크][github]로 대체하고 내용 위주로 살펴보자.

---
# Day 4
AOC 2018 Day 4. datetime을 parsing해서 사용하고 싶지만 그렇게 풀지 않아도 되는 문제. 문제의 요구사항을 잘 읽어보면 `00:00~00:59` 사이의 데이터만 있는 것을 알 수 있음. input이 까다로워서 데이터 전처리하는데 드는 시간이 더 많았던 것 같다. 앞선 리뷰들을 잘 반영해서 `hash-map` 적절히 잘 사용했음.

- [내가 푼 코드](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2018_day4.clj)
- [리뷰 받은 코드](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2018_day4_reviewed.clj)

## 주요 내용
- `map + flatten` 같은 건 `mapcat`을 사용해도 좋다.
- `assoc-in + get-in` 대신 `update-in`이라는 걸 쓰자
- `map`을 부분적으로 바꿀 때는 `update` 혹은 `update-in`을 쓰자
- `hash-map desctructing`은 같은 이름을 사용할거라면 `{:keys […]}`를 사용해도 좋다.
- `max-key`라는 좋은게 있더라

# Day 5
AOC 2018 Day 5. 대소문자 잘 구분하는 문제. 어렵진 않았음.

- [내가 푼 코드](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2018_day5.clj)
- [리뷰 받은 코드](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2018_day5_reviewed.clj)

특별한 내용은 없었던 것 같다.

# Day 6
AOC 2018 Day 6. 문제 접근법 자체가 약간 어려웠던 문제. 실제 코드로 구현하는데도 시간이 많이 걸렸음.

- [내가 푼 코드](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2018_day6.clj)
- [리뷰 받은 코드](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2018_day6_reviewed.clj)

## 주요 내용
- return type의 일관성을 가지면 좋다.
- `group-by`라는 것이 있다.
- aggregation할 때 `frequencies`를 잘 사용해도 좋다.

# Day 7
AOC 2018 Day 7. workers들의 일을 잘 할당하는 문제.

- [내가 푼 코드](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2018_day7.clj)
- [리뷰 받은 코드](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2018_day7_reviewed.clj)

## 주요 내용
- `iterate` + `drop-while`을 잘 쓰면 좋다
- ^ 이거 쓸 때 주의해야하는 점들이 몇가지 있는데 일단 `LazySeq`를 반환하기 때문에 이게 가능한 것을 꼭 인지하고 써야한다. 그렇기 때문에 `take`, `first` 등 을 하지 않으면 drop-while의 결과가 무한하기 때문에 `first`, `take` 등을 같이 꼭 사용하도록 하자.
- 사용법이라기 보다는 이 개념을 인지하고 있자.
- 상태라는 걸 잘 관리한다는 측면에서 생각하자
- 이 문제는 클로저랑 상관 없이 ps적으로 봐도 좋은 문제인 것 같다
- `threading-macro`는 함수의 조합으로 절차지향적으로 생각할 수 있게 하는 좋은 도구인 것 같다
- 그럼에도 불구하고 디버깅에 많은 시간을 썼는데 반성하도록 하자

# Day 8
AOC 2020 Day 1, 4. 간단한 두 문제였다. 생각해보면 첫 날에 풀었다면 조금 헤맸을 수도 있지만 부트캠프 8일차에 풀기엔 쉬웠던 문제.

- [내가 푼 코드](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2018_day8.clj)

## 주요 내용
- Day 7에서 얻은 지식들 덕분에 쉽게 푼 것 같다.
- `map-indexed`는 처음 써봤다.
- `drop-while` 멈추는 조건이 할 때마다 약간 헷갈린다.

## 코드
- [2020-day1](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2020_day1.clj)
- [2020-day4](https://github.com/jungwookim/aoc-exercise/blob/master/src/p2020_day4.clj)

## 회고
- `for`를 사용하면 combination처럼 2중 루프를 돌게 되는데 이 때 break-point로 `:when` 같은 걸 사용할 수 있더라.
- 두번째 문제는 파싱만 잘하면 되는 문제였다. 다만 정규식에 익숙하지 않아 항상 불편한 점은 있다.
- `clojure.spec` 이라는 걸 활용해보자

# Reference
- [my github repo][github]
- [clojure library documents][clojure_doc]
- [advent of code][advent_of_code]


[github]:https://github.com/jungwookim/aoc-exercise/tree/master/src
[clojure_doc]:https://clojuredocs.org/clojure.core
[advent_of_code]:https://adventofcode.com/2018/day/3