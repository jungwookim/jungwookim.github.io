Title: Clojure 사용 6개월 후기
Date: 2022-05-22
Tags: Clojure, Retrospectives

지금 회사로 11월 15일에 옮겼으니 만으로 딱 6개월이 지났다. 시간이 꽤 빨리 흘렀다. 그 사이에 여러가지 많이 배운 것 같다.

우리 회사는 지금 다이나믹 타입을 가진 Lisp계열의 함수형언어인 `Clojure`를 사용 중인데 이를 6개월가량 써보면서 드는 생각들을 조금 정리해보도록 하자. 덤으로 사용하고 있는 스택들도 조금 정리해보도록 하자.

# 인상
부족하거나 막힘이 없는 언어라고 느꼈다. 비즈니스 로직에 필요한 구현을 해야할 때 코드가 장황해지지 않았다. 간결했고 가독성이 높았다. 적은 코드량으로 깨끗한 코드를 짤 수 있었고 입/출력만 있고 side effects가 없는 함수들로 파이프라인만 잘 구성하면 되었다. 또한 매크로를 사용해서 함수를 만드는 함수나 새로운 함수를 쉽게 정의할 수도 있었다. 없으면 만들면 되었다. 메타프로그래밍이 쉽고 재밌었다.

처음 약 3개월동안은 타입이 없다는 불편함이 발목을 잡지 않을까 했는데 딱히 없었다. 언어의 설계가 `hashmap`을 중심으로 하고, nil에 대한 처리를 잘 하고 있어서 보통의 개발자라면 의심하지 않으면 잘 사용할 것 같다.

클래스에 얽매이지 않다보니 코드 읽기도 수월하고 이만저만 좋은게 많다.

# Server Tech Stack
대충 정리하니 아래와 같다. 다른 3rd 이용 API나 사소한 것들은 제외시켰다.
- RESTful API: [Reitit](https://github.com/metosin/reitit). Data-driven router일 뿐이다. (파이썬의 장고나 자바의 스프링 같은게 아니다)
- Data-driven schemas: [malli](https://github.com/metosin/malli)
- GraphQL API: [Lacinia](https://lacinia.readthedocs.io/en/latest/)
- Dataloader: [superlifter](https://github.com/oliyh/superlifter)
- Date and Time: [java-time](https://github.com/dm3/clojure.java-time)
- HTTP
    - [ring](https://github.com/ring-clojure/ring): HTTP server abstraction
    - [clj-http](https://github.com/dakrone/clj-http): Apache HttpComponents client wrapper
- Database: [next.jdbc](https://github.com/seancorfield/next-jdbc)
- Connection pools: [hikari-cp](https://github.com/tomekw/hikari-cp)
- Structural Migrations: [Migratus](https://github.com/yogthos/migratus)
- DSL for SQL generation: [honeysql](https://github.com/jkk/honeysql)
- Security: [Buddy](https://github.com/funcool/buddy)
- Pattern Matching: [core.match](https://github.com/clojure/core.match)
- Testing: [kaocha](https://github.com/lambdaisland/kaocha)
- Code Analysis and Linter: [clj-kondo](https://github.com/borkdude/clj-kondo)
- Error utility: [failjure](https://github.com/adambard/failjure)
- Dependency injection(*Managed lifecycle of stateful objects*): [integrant](https://github.com/weavejester/integrant)

# 더 알아나가야할 것들
더 알아나가야할 것들을 안다는 것 자체가 이미 잘 알고 있다는 건데... 실제 코드에서는 core.async를 잘 사용하지 않게 되는데 지금과 같은 규모와 서비스 특성상 비동기적 처리를 할 일이 자주 있을까 싶다. 그리고 protocol 관련해서도 코드 작성을 꺼려하게 되는데 (의견이 분분) 이것도 조금 더 살펴보도록 해야겠다.

# 데이터로서 프로그래밍 한다는 것(DSL관련)
Terraform을 비롯하여 데이터로서 행동을 정의하고 규약한다는 것은 이미 프로그래밍 세계에서 많이 이루어졌다. Graphql Schema도 이 중 하나일테고. 회사 내부에서 지금 GraphQL을 적극 사용 중인데 DSL로 만들어서 사용할 수 있을 각을 보고 있다. 조금 더 디벨롭할 수 있는 시간을 쓰도록 해야겠다.
