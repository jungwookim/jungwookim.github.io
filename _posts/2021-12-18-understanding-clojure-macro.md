---
layout: post
title:  "클로저 매크로 초급 딱지 떼기 (1)"
categories: clojure macro clojure-macro
---

# 목표
클로저 매크로를 할 일은 별로 없을 것 같고 보고 읽을 줄 아는 정도의 지식을 쌓자

# 알거라 생각하거나 알면 좋은 내용
- clojure는 lisp 계열의 언어이다.
- 클로저의 기본 문법을 알고 있다.

# 사전 지식
사전 지식이라기 보다는 이 글 내내 반복해서 나오는 것들에 대해서 미리 요약해둔 것이라고 보면 된다. 이 글을 다 읽고도 아래 나열된 것들을 이해 못했다면 글쓴이의 잘못이다.
- `symbol`: `심볼`, `기호`라고 부른다. 값을 가지지 않는 심볼이다. 평가된 심볼인지 평가되지 않는 심볼인지 구분할 수 있다.
- `'`: `single quoting`이라고 부른다. **평가(evaluation)을 꺼놓는 것(turn off). 평가되지 않은 데이터 구조를 반환한다. 심볼을 그대로 반환하기 위함이다.**
- `` ` ``: `syntax quoting`이라고 부른다. single quote와 비슷하지만 중요한 2가지 다른 점이 있다. 하나는 namespace를 포함한 심볼을 반환한다.(the fully qualified symbols) 두번째는 `~`를 사용해서 `unquote`하게 만들 수 있게 한다. 즉, 평가되지 않게 한 것을 평가를 시켜버린다. 다르게 말하면 `syntax quoting` 능력을 없애버린다.
- `~`: `syntax quoting`에서 설명했지만 `quoting`된 것을 `unquote`해버린다. 반복하자면 평가되지 않게 한 것을 평가를 시켜버린다.
- `~@`: `unquote slicing`이라고 부른다. 망치 모양이랑 비슷하다. `seqable data structure`를 unwrap한다고 보면 된다. 간단하게 생각해서 괄호를 벗겨내고 그 발가벗은 순서대로 둔다고 생각하자.
- `~'`: 일종의 꼼수인데, `syntax quote`으로 평가를 미뤘는데 let binding 같은 것 할 때 네임스페이스가 붙어 있는데, 네임스페이스를 없애줘야할 때 `'`를 통해 평가 되지 않게 다시 하고 `~`를 통해 다시 평가하여 심볼을 얻을 때 필요하다. 즉, 매크로 안에서 심볼을 만들어써야될 때 필요한데 아니면 `symbol#`을 직접 사용해도 된다. 사실 `symbol#`은 사실 auto-gensym'd 심볼이다.
- `symbol#`: auto-gensym'd symbol인데 `~'`는 지양하고 얘를 쓰도록 하자.
- `gensym`: variable capture에 쓰인다. 고유한 symbol을 만들어준다.

# 예제를 사전 지식을 가다듬자

예제는 [Brave Clojure][braveclojure_macro]를 일부 참고했다.

평가된 `+` 심볼.
```clojure
+
; => #function[clojure.core/+]
```

평가되지 않은 `+` 심볼.
```clojure
'+
; +
(quote +)
; +
```

보통 클로저에서 사용하는 (함수 인자 인자) 표현식이다. 괄호 안의 표현이 평가되었다.
```clojure
(+ 1 2)
; => 3
```

`single quote`를 사용해보자. 괄호 안의 표현이 평가되지 않았다.
```clojure
'(+ 1 2)
; => (+ 1 2)
```

`syntax quote`를 사용해보자. `+` 함수의 네임스페이스를 포함한 심볼을 반환했다.
```clojure
`(+ 1 2)
; => (clojure.core/+ 1 2)
```

조금 어색하지만 `syntax quote`를 하고도 `~`를 이용해서 평가를 해보자.
```clojure
`~(+ 1 2)
; => 3
```

생각해보자. 우리가 평소에 사용하는 방식이다. 쉽다.
```clojure
(+ 1 (inc 1))
; => 3
```

이거는 어떻게 될까? 위에서 생각한대로 평가를 미룬다고 생각해보자. 괄호 안의 평가들이 다 미루어졌다.
```clojure
`(+ 1 (inc 1))
; => (clojure.core/+ 1 (clojure.core/inc 1))
```

`(inc 1)`는 평가가 되었으면 한다. 그럼 이렇게 해보면 된다.
```clojure
`(+ 1 ~(inc 1))
; => (clojure.core/+ 1 2)
```

이렇게 하면 어떻게 될까?
```clojure
`(+ 1 (~inc 1))
; => (clojure.core/+ 1 (#function[clojure.core/inc] 1))
; inc만 평가되어 나왔다.
```

`unquote slicing`을 알아보자. 먼저 우리가 알던 것을 보자. 평가를 꺼뒀지만 두번째 괄호 안에서는 평가를 바로 하도록 `~`를 썼다. list가 잘 반환되었다.
```clojure
`(+ ~(list 1 2 3))
; => (clojure.core/+ (1 2 3))
```

망치로 바꿔보자. 이렇게 하면 unwrap된 상태이다.
```clojure
`(+ ~@(list 1 2 3))
; => (clojure.core/+ 1 2 3)
```

매크로는 함정들이 많으니 주의하도록 하자.

# Reference
- [그린랩스 기술블로그 매크로][greenlabs_macro]
- [eunmin-gitbooks-macro][eunmin_macro]
- [Writing macros in Brave Clojure ][braveclojure_macro]

[greenlabs_macro]:https://green-labs.github.io/the-macro
[eunmin_macro]:https://eunmin.gitbooks.io/clojure-for-beginners/content/9_macros.html
[braveclojure_macro]:https://www.braveclojure.com/writing-macros/
