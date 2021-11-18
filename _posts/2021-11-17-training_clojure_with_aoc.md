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

# Reference
- [my github repo][github]
- [clojure library documents][clojure_doc]
- [advent of code][advent_of_code]


[github]:https://github.com/jungwookim/aoc-exercise/tree/master/src
[clojure_doc]:https://clojuredocs.org/clojure.core
[advent_of_code]:https://adventofcode.com/2018/day/3