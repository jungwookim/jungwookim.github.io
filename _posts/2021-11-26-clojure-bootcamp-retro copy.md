---
layout: post
title:  "Clojure 다형성"
categories: clojure polymorphism
---

clojure의 다형성에 대해서 알아보자

다형성이라고 불러도 될 지 모르겠는데 약간 디자인 패턴 중 하나인 팩토리 패턴과 유사하다고도 볼 수 있겠다.

```clojure
(defmulti 이름 docString? attr-map? 디스패치함수 & 옵션)

(defmethod 멀티펑션 디스패치값 & fn-tail) ; fn-tail이라고 하면 파라미터와 함수 내용을 말하는 것으로 확인
```

예제들을 여러개 봤는데 처음엔 이해하기 힘들었는데 천천히 이해해보도록 하자.

```clojure
(ns were-creatures)
➊ (defmulti full-moon-behavior (fn [were-creature] (:were-type were-creature))) ; multifn 함수 이름과 함수 내용을 정의했다
➋ (defmethod full-moon-behavior :wolf
  [were-creature]
  (str (:name were-creature) " will howl and murder")) ; multifn이름을 그대로 쓰고 두번째 인자는 파라미터로 받은 후 디스패치 함수의 실행 결과값이 :wolf와 대응되는지 확인하는 인자, 그리고 그게 대응된다면 마지막에 fn-tail이 호출된다
➌ (defmethod full-moon-behavior :simmons
  [were-creature]
  (str (:name were-creature) " will encourage people and sweat to the oldies"))

(full-moon-behavior {:were-type :wolf
➍                      :name "Rachel from next door"})
; => "Rachel from next door will howl and murder"

(full-moon-behavior {:name "Andy the baker"
➎                      :were-type :simmons})
; => "Andy the baker will encourage people and sweat to the oldies"
```

# Reference
- [brave clojure multimethods]:(https://www.braveclojure.com/multimethods-records-protocols/)
- [defmulti]:(https://clojuredocs.org/clojure.core/defmulti)
- [defmethod]:(https://clojuredocs.org/clojure.core/defmethod)
