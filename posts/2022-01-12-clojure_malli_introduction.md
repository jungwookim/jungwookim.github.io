Title: Clojure Malli
Date: 2022-01-12
Tags: Clojure

# 동기
처음에는 type system을 일부 빌려오고 싶었다. 클로저가 동적 타입의 언어이다 보니 개발하는 사람은 데이터 스키마를 어느정도 머릿 속에 가지고 있고 디버깅도 하다보니 오히려 속도를 낼 수 있는 장점이 있지만 해당 코드를 개발하지 않은 사람이 봤을 때에는 추가 개발이 힘든 점이 있다고 느꼈다.

그래서 리서치를 하다가 Data-driven development를 알게 되었고 찾다보니 Malli를 썼을 때 장점이 있어 보였다.

# 소개
Malli는 Data-driven 개발을 위해 만들어진 clojure library이다. 새로운 validation과 specification을 제시한다. schema definition, validation, error, value and schema transformation, generation and registries 같은 것들을 포함하고 있다.

간단하게 어떻게 동작하는지 한번 살펴보자.

# Syntax와 simple validation

Vector 방식으로 정의
```clojure
(require '[malli.core :as m])

(def non-empty-string
  (m/schema [:string {:min 1}]))

(m/schema? non-empty-string)
; => true

(m/validate non-empty-string "")
; => false

(m/validate non-empty-string "kikka")
; => true

(m/form non-empty-string)
; => [:string {:min 1}]
```

Map 방식으로 정의
```clojure
(def non-empty-string
  (m/from-ast {:type :string
               :properties {:min 1}}))

(m/schema? non-empty-string)
; => true

(m/validate non-empty-string "")
; => false

(m/validate non-empty-string "kikka")
; => true

(m/ast non-empty-string)
; => {:type :string,
;     :properties {:min 1}}
```

Schema AST를 쓰는 방식이 훨씬 빠르다고 한다.

# 간단한 schema definition
```clojure
(m/validate [:sequential any?] (list "this" 'is :number 42))
;; => true

(m/validate [:vector int?] [1 2 3])
;; => true

(m/validate [:vector int?] (list 1 2 3))
;; => false

; fixed length vector
(m/validate [:tuple keyword? string? number?] [:bing "bang" 42])
;; => true


(m/validate [:repeat {:min 2, :max 4} int?] [1]) ; => false
(m/validate [:repeat {:min 2, :max 4} int?] [1 2]) ; => true
(m/validate [:repeat {:min 2, :max 4} int?] [1 2 3 4]) ; => true ; :max is inclusive
(m/validate [:repeat {:min 2, :max 4} int?] [1 2 3 4 5]) ; => false


; string schemas
(m/validate string? "kikka") ; using a predicate

(m/validate :string "kikka") ; using :string schema
;; => true

(m/validate [:string {:min 1, :max 4}] "")
;; => false

; maybe schemas
(m/validate [:maybe string?] "bingo")
;; => true

(m/validate [:maybe string?] nil)
;; => true

(m/validate [:maybe string?] :bingo)
;; => false

; fn schemas
(def my-schema
  [:and
   [:map
    [:x int?]
    [:y int?]]
   [:fn (fn [{:keys [x y]}] (> x y))]])
   
(m/validate my-schema {:x 1, :y 0})
; => true

(m/validate my-schema {:x 1, :y 2})
; => false

(def pants-schema
  [:and
   [:map
    [:id int?]
    [:size {:optional true} [:maybe :int]]
    [:size-alphabet {:optional true} [:maybe [:enum "S" "M" "L"]]]]
   [:fn {:error/message "size and size alphabet should be nil-matched"}
    '(fn [{:keys [size size-alphabet]}]
       (or (and (nil? size) (nil? size-alphabet))
           (and (some? size) (some? size-alphabet))))]])

(m/validate pants-schema {:id   1
                          :size nil
                          :size-alphabet "S"})
; => false
```

# 함수에서 어떻게 사용하고 있는지
```clojure
(defn foo-meta
  "schema via var metadataz"
  {:malli/schema [:=> [:cat :int] :int]}
  [x]
  (inc x))

(m/=> foo-declare [:=> [:cat :int] :int])
(defn foo-declare
  "schema via separate declaration"
  [x]
  (inc x))

(foo-meta 1) ; ok
(foo-meta "1") ; clj-kondo가 빨간 줄 그어줌

(foo-declare 1) ; ok
(foo-declare 1) ; 역시 clj-kondo가 빨간 줄 그어줌
```

# Errors
```clojure
; 바로 알아보기 힘들다
(m/validate pants-schema {:id   1
                          :size nil
                          :size-alphabet "S"})

(-> pants-schema
    (m/explain {:id 1})
    (me/humanize))
```

# Value transformation
Default Transformers: `string-transformer`, `json-transformer`, `strip-extra-keys-transformer`, `default-value-transformer` and `key-transformer`.

```clojure
(m/decode int? "42" mt/string-transformer)
; 42

(m/encode int? 42 mt/string-transformer)
; "42"

(m/decode
  Address
  {:id "Lillan",
   :tags ["coffee" "artesan" "garden"],
   :address {:street "Ahlmanintie 29"
             :city "Tampere"
             :zip 33100
             :lonlat [61.4858322 23.7854658]}}
  mt/json-transformer)
;{:id "Lillan",
; :tags #{:coffee :artesan :garden},
; :address {:street "Ahlmanintie 29"
;           :city "Tampere"
;           :zip 33100
;           :lonlat [61.4858322 23.7854658]}}

(m/encode
  Address
  {:id "Lillan",
   :tags ["coffee" "artesan" "garden"],
   :address {:street "Ahlmanintie 29"
             :city "Tampere"
             :zip 33100
             :lonlat [61.4858322 23.7854658]}}
  (mt/key-transformer {:encode name}))
;{"id" "Lillan",
; "tags" ["coffee" "artesan" "garden"],
; "address" {"street" "Ahlmanintie 29"
;            "city" "Tampere"
;            "zip" 33100
;            "lonlat" [61.4858322 23.7854658]}}

(def strict-json-transformer
  (mt/transformer
    mt/strip-extra-keys-transformer
    mt/json-transformer))

(m/decode
  Address
  {:id "Lillan",
   :EVIL "LYN"
   :tags ["coffee" "artesan" "garden"],
   :address {:street "Ahlmanintie 29"
             :DARK "ORKO"
             :city "Tampere"
             :zip 33100
             :lonlat [61.4858322 23.7854658]}}
  strict-json-transformer)
;{:id "Lillan",
; :tags #{:coffee :artesan :garden},
; :address {:street "Ahlmanintie 29"
;           :city "Tampere"
;           :zip 33100
;           :lonlat [61.4858322 23.7854658]}}

; 종합 예제
(m/encode
  [:map {:default {}}
   [:a [int? {:default 1}]]
   [:b [:vector {:default [1 2 3]} int?]]
   [:c [:map {:default {}}
        [:x [int? {:default 42}]]
        [:y int?]]]
   [:d [:map
        [:x [int? {:default 42}]]
        [:y int?]]]
   [:e int?]]
  nil
  (mt/transformer
    mt/default-value-transformer
    mt/string-transformer))
;{:a "1"
; :b ["1" "2" "3"]
; :c {:x "42"}}
```

# Schema generation
Schema 추론
```clojure
(require '[malli.provider :as mp])

(def samples
  [{:id "Lillan"
    :tags #{:artesan :coffee :hotel}
    :address {:street "Ahlmanintie 29"
              :city "Tampere"
              :zip 33100
              :lonlat [61.4858322, 23.7854658]}}
   {:id "Huber",
    :description "Beefy place"
    :tags #{:beef :wine :beer}
    :address {:street "Aleksis Kiven katu 13"
              :city "Tampere"
              :zip 33200
              :lonlat [61.4963599 23.7604916]}}])

(mp/provide samples)
;[:map
; [:id string?]
; [:tags [:set keyword?]]
; [:address
;  [:map
;   [:street string?]
;   [:city string?]
;   [:zip number?]
;   [:lonlat [:vector double?]]]]
; [:description {:optional true} string?]]
```
# Value generation
```clojure
(mg/generate pants-schema {:seed 2})
; => {:id -28, :size 8083038, :size-alphabet "M"}
```

# Performance
ideomatic clojure보다 훨씬 빠르다고 한다. 직접 테스트는 안하고 결과만 공유한다.
```clojure
(require '[criterium.core :as cc])

(def valid {:x true, :y 1, :z "zorro"})

;; idomatic clojure 54ns
(let [valid? (fn [{:keys [x y z]}]
               (and (boolean? x)
                    (if y (int? y) true)
                    (string? z)))]
  (assert (valid? valid))
  (cc/quick-bench (valid? valid)))

(require '[malli.core :as m])

;; malli 39ns
(let [valid? (m/validator
               [:map
                [:x :boolean]
                [:y {:optional true} :int]
                [:z :string]])]
  (assert (valid? valid))
  (cc/quick-bench (valid? valid)))
```

# 우리 프로젝트에서의 활용 - 데이터베이스 입력/업데이트시 값 validation을 해보자
```clojure
(def subsidy-schema
  [:and
   [:map
    [:id int?]
    [:area int?]
    [:area-ineqaulity [:enum ">=" ">" "<" "<="]]]
   [:fn {:error/message "area and area-inequality는 nil match가 되어야함"}
    (fn [{:keys [area area-ineqaulity]}]
      (or (and (nil? area) (nil? area-ineqaulity))
          (and (some? area) (some? area-ineqaulity))))]])

(defn insert-subsidy-info
  {:malli/schema [:=> [:cat subsidy-schema] :nil]}
  [subsidy]
  (prn subsidy))
```

# Spec, Schema and Malli

`Schema`라는게 있는데 다 좋은데, serializing이랑 de-serializing할 때 non-trivial하다. `Spec`도 개쩌는데, `runtime transformation`을 지원하지 않는게 가장 큰 흠이다. `malli`라는 건 다이나믹 시스템 개발에서 모든 것을 다 지원하기 위해 등장했다. Spec은 runtime transformation이 없다. 그리고 Spec은 global registry 하나를 통해서 다 spec을 관리하는데 Malli는 그럴 필요는 없다. schema와 registry를 값처럼 프로그래밍을 할 수 있다. null이나 optional한 것들도 더 쉽게 처리할 수 있다. 또한 Spec은 schema를 표현하기 위해 macro를 사용한다. 하지만 malli는 vector와 map을 사용한 pure data 그 자체이다. Data가 곧 system이라고 했을 때 더 쉽게 받아들일 수 있다. Spec, Schema 그리고 Malli는 하나만 써야하는 것은 아니고 같이 사용될 수 있다는 것이 중요하다.

# Reference
- [malli github](https://github.com/metosin/malli)
- [malli blog 1](https://www.metosin.fi/blog/malli/)
- [malli blog 2](https://www.karimarttila.fi/clojure/2020/12/07/malli-clojure-library.html)
