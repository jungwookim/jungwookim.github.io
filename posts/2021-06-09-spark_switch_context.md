Title: Switch spark context
Date: 2021-06-09
Tags: Spark, Pyspark

# Intro
`Azure Databricks` 원격에서 바로 작업하던 환경에서 아래와 같은 몇가지 이유로 로컬 환경에서도 작업할 수 있도록 해야했다.

- 버전 관리가 되었으면 한다
- 코드 재사용이 더 활발했으면 한다
- 인터페이스 사용이 더 빨랐으면 한다
- 로컬 컴퓨터에서 원격으로 `Job sumit`을 할 수 있으면 좋겠다
- `local spark` 환경에서 `Unit test` 할 수 있으면 좋겠다

작업자가 혼자일 때는 저런 것들이 필요 없을 수 있었다면 2명이 된 지금, 앞으로를 생각했을 때 필요한 일이었다.

# 버전 관리과 코드 재사용
`.ipynb`과 `.py`을 관리 해야했는데 크게 어려운 건 없었다. policy만 일단 잘 정하고 빨리 시작하는게 중요했다. 모듈화 할만한 건 `.py`으로 다 관리하고 `notebook`에서는 `data manipulation`만 하기로 하자. 그럼 코드 재사용도 가능해지게 되었고 추후에 `serving api` 만들 때도 도움이 될 것 같다.

# 로컬 IDE 사용과 원격 연결
원격에서 작업하면, 특히 나처럼 spark 입문자에게 `api reference`를 매번 찾아보면서 하기엔 속도가 안나는 경우가 종종 있다. 실수도 잦고. 그래서 `pycharm`에서 조금만 설정하면 연결할 수 있게 된다. 그럼 `azure databricks notebook`에서 작업하는 것과 거의 똑같다. 다만 azure databricks에서 제공하는 몇가지 서비스들(display해서 plotting하거나 등등)을 못쓰는데 그건 `pandas`로 몇몇 모듈만 만들어놓으면 되니까 문제는 아니다.

# Spark Context 변경 및 단위 테스트
로컬에서 원격에 `job sumit`을 하면서 작업을 주로 하겠지만, 때론 이게 번거로울 때도 많다. 예를 들어, 쉬고 있던 클러스터를 `run` 해야하거나 `unit test`를 해야하는데 비싼 원격 클러스터를 쓰기엔 돈이 아까웠다. 그래서 로컬 클러스터랑 원격 클러스터랑 쉽게 context switching을 하고 싶었다. `master` 연결만 잘 하면 되겠거니 했는데 아무리 해도 `session`이 원격에 붙길래, 그게 보니까 databricks-connect 패키지에 pyspark 설정을 건드려줘야할 것 같은데 뭔가 영 이상하다. pyspark sub-dep로 관리하는게 아니라 지들 pyspark를 wrapping한 것처럼 사용해서 context switching이 잘 안됐다.

결론만 말하면, 가상환경을 새로 만들어서 `pyspark`를 pypi로 새로 깔고 그 환경을 interpreter에 연결해서 실행하니 로컬에서 잘 실행되었다. 후... 로컬 테스트할 때는 환경도 바꾸고 interpreter도 바꾸고 해야해서 약간 번거롭지만 일단 목표한 바는 이루었으니 쓰도록 하자.

```python
from pyspark.sql import SparkSession

# 기본 원격 클러스터 연결 방식 (databricks-connect configure 에 세팅된대로 연결)
spark = SparkSession.builder.getOrCreate()

# 로컬 클러스터 연결 방식
spark = SparkSession.builder.master('local').getOrCreate()
```

# SPARK_HOME 충돌 문제
원격에서 로컬로 잘 연결을 바꿨었다. 문제는 다시 원격에 붙어서 작업하고 싶어서 python interpreter를 바꾸고 작업하는데 여전히 로컬에 붙고 있었다. 이상해서 계속 삽질하다가 `databricks-connect test`의 결과를 보고 단서를 찾았다. 내용은 `SPARK_HOME` 환경 변수가 local이랑 가상환경에서 사용하는 거랑 충돌이 나고 있었던 것. 그래서 환경 변수를 잘 수정해서 해결했다. 근데 이거 환경 바꿀 때마다 일일이 신경써주기 귀찮은데 좀 더 편하게 하는 방향을 잡긴 해야겠다.
## Reference

- [databricks-connect][databricks-connect]
- [github-version-control][github-version-control]

[databricks-connect]: https://docs.microsoft.com/en-us/azure/databricks/dev-tools/databricks-connect
[github-version-control]: https://docs.microsoft.com/en-us/azure/databricks/notebooks/github-version-control
