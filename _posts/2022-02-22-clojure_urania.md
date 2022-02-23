---
layout: post
title:  "Urania 정리"
categories: clojure urania
---

# 소개
종종 비즈니스 로직이 DB, Cache, web services 등 여러 소스로부터 받아보고 싶은 원격 데이터에 의존한다. 이를 다루는게 보통 쉬운 일이 아니다. Urania는 비즈니스 로직을 쉽게 관리하게 해주는데 아래 3가지를 효율적으로 다룬다.

- batch multiple requests to the same data source
- request data from multiple data sources concurrently
- cache previous requests

# 유저 가이드
## 원리
모든 프로그램은 성능과 표현력에 대해서 항상 고민이다. 쉽게 표현하면 성능이 떨어지는 경우가 있기 때문. 아래의 예시를 보자.

```clojure
(require '[clojure.set :refer [intersection]])

(defn friends-of
  [id]
  ;; ...
  )

(defn count-common
  [a b]
  (count (intersection a b)))

(defn count-common-friends
  [x y]
  (count-common (friends-of x) (friends-of y)))

(count-common-friends 1 2)
```
`(friends-of x)`와 `(friends-of y)`는 서로 독립적이다. 동시에 가져오면 좋겠다. 게다가, x, y가 심지어 같은 사람이면 두번할 필요도 없을 것이다. 최적화 하면 어떻게 될까? caching과 batching을 혼합적으로 생각해야한다.

`Urania`는 코드를 약간 바꾸고도 **데이터**를 동시에 가지고 올 수 있게 해준다. 대략 아래처럼.
```clojure
(require '[urania.core :as u])

(defn count-common-friends [x y]
  (u/map count-common
         (friends-of x)
         (friends-of y)))

(u/run! (count-common-friends 1 2))
```
보면 알겠지만 데이터 가져오는 것과 실행을 분리했다.
run할 때 `urania`는 아래와 같은 일을 한다.
- request data from multiple data sources concurrently
- batch multiple requests to the same data source
- cache repeated requests

## 원격에서 데이터 가져오기
보통 원격에서 데이터 가져오는 건 비동기적이고(asynchronous) 혹은 이거나 에러가 날 수도 있다. 그래서 `urania`가 [Promesa][promesa]를 사용한다.

```clojure
(require '[promesa.core :as prom])

(defn remote-req [id result]
  (prom/promise
    (fn [resolve reject]
      (let [wait (rand 1000)]
        (println (str "-->[ " id " ] waiting " wait))
        (Thread/sleep wait)
        (println (str "<--[ " id " ] finished, result: " result))
        (resolve result)))))
```

## Remote data sources
`Urania`의 `DataSource` protocol을 구현한 타입인 data sources를 정의해보자.
- **`-identity`**, which returns an identifier for the resource (used for caching and deduplication).
- **`-fetch`**, which fetches the result from the remote data source returning a promise.

대략 예시
```clojure
(require '[urania.core :as u])

(defrecord FriendsOf [id]
  u/DataSource
  (-identity [_] id)
  (-fetch [_ _]
    (remote-req id (set (range id)))))

(defn friends-of [id]
  (FriendsOf. id))
```

- `(u/run! (friends-of 10))` : return promise
- `(deref (u/run! (friends-of 10)))`: deref를 이용한 block
- `(u/run!! (friends-of 10))`: block하는 두번째 방법. clojure에서는 이게 더 나음

### 가져온 데이터 변환하기
`u/map`을 이용해서 결과를 변환할 수 있다
```clojure
(u/run!!
  (u/map dec (u/map count (friends-of 10))))
```

### 결과들의 의존성
fetch 먼저하고 계산하는 로직이 있다고 하자. `u/mapcat`을 사용하면 된다.

```clojure
(defrecord ActivityScore [id]
  u/DataSource
  (-identity [_] id)
  (-fetch [_ _]
    (remote-req id (inc id))))

(defn activity
  [id]
  (ActivityScore. id))
```

첫번째 친구의 활동성만 찾아보자.
```clojure
(defn first-friends-activity
  [id]
  (u/mapcat (fn [friends]
              (activity (first friends)))
            (friends-of id)))

(u/run!! (first-friends-activity 10))
;; -->[ 10 ] waiting 575.5289747556875
;; <--[ 10 ] finished, result: #{0 7 1 4 6 3 2 9 5 8}
;; -->[ 0 ] waiting 63.24540090623976
;; <--[ 0 ] finished, result: 1
;; => 1
```

모든 친구에 대해서 한다면?
```clojure
(defn friends-activity
  [id]
  (u/mapcat (fn [friends]
              (u/collect (map activity friends)))
            (friends-of id)))

(u/run!! (friends-activity 5))
;; -->[ 5 ] waiting 480.8846764476696
;; <--[ 5 ] finished, result: #{0 1 4 3 2}
;; -->[ 0 ] waiting 488.58045819535687
;; -->[ 1 ] waiting 87.96784013662884
;; -->[ 4 ] waiting 868.2747930486679
;; <--[ 1 ] finished, result: 2
;; -->[ 3 ] waiting 293.59429652774116
;; <--[ 3 ] finished, result: 4
;; -->[ 2 ] waiting 280.68098217346835
;; <--[ 0 ] finished, result: 1
;; <--[ 2 ] finished, result: 3
;; <--[ 4 ] finished, result: 5
;; => [1 2 5 4 3]
```

동시에 실행되는 것을 위에서 확인할 수 있다. 게다가 알아서 중복 요청은 하지 않는다.
```clojure
(u/run!! (u/collect [(friends-of 1) (friends-of 2) (friends-of 2)]))
;; -->[ 2 ] waiting 634.8383950264134
;; -->[ 1 ] waiting 924.8381446535985
;; <--[ 2 ] finished, result: #{0 1}
;; <--[ 1 ] finished, result: #{0}
;; => [#{0} #{0 1} #{0 1}]
```

`collect` + `mapcat`은 `traverse`로 대체될 수 있긴 하다.
```clojure
(defn friends-activity
  [id]
  (u/traverse activity (friends-of id)))
```

### 배치 요청

위의 예시들에서 `urania`가 중복요청들을 잘 없앴는데 여전히 개선 여지가 있다. 위 예시 `u/collect`에서 동일한 데이터 소스에 대한 요청이 동시에(concurrently) 실행되는 방법을 봤다. 이걸 batch로 만들어보자.

```clojure
(extend-type ActivityScore
  u/BatchedSource
  (-fetch-multi [score scores _]
    (let [ids (cons (:id score) (map :id scores))]
      (remote-req ids (zipmap ids (map inc ids))))))
```

여기 보면 한번에 요청했다. n+1(6)번의 요청이 2번으로 줄었다.
```clojure
(u/run!! (friends-activity 5))
;; -->[ 5 ] waiting 123.11807342157954
;; <--[ 5 ] finished, result: #{0 1 4 3 2}
;; -->[ (0 1 4 3 2) ] waiting 97.95578032830765
;; <--[ (0 1 4 3 2) ] finished, result: {0 1, 1 2, 4 5, 3 4, 2 3}
;; [1 2 5 4 3]
```

# 응용
Cache, Executor, Environment 관련된 건데 공식 문서보고 확인하자.

# Reference
- [Document][urania_doc]
- [Github][urania_github]
- [Promesa][promesa]


[urania_doc]:http://funcool.github.io/urania/latest/
[urania_github]:https://github.com/funcool/urania
[promesa]:https://github.com/funcool/promesa
