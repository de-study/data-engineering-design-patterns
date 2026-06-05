### CH05 Data Value Design Patterns
**"raw 데이터를 비즈니스에서 실제로 쓸 수 있는 데이터로 만드는 방법"** 을 다루는 챕터.

패턴 목록:
- 패턴#22 정적조인기(Static Joiner)
- 패턴#23 동적조인기(Dynamic Joiner)
- 패턴#24 래퍼(Wrapper)
- 패턴#25 메타데이터 데코레이터(Metadata Decorator)
이 패턴들은 전부 **데이터에 가치를 더하는 작업**이야.
CH03은 에러 관리, CH04는 멱등성 — 둘 다 "잘못되지 않게" 하는 방어적인 패턴들이었어.
CH05부터는 방향이 바뀌어. "어떻게 하면 데이터가 더 유용해지냐"는 거야.

패턴 흐름은 **데이터를 풍부하게 만드는 작업의 복잡도가 낮은 것부터 높은 것 순서**야.
- 패턴#22, #23 (Data Enrichment) — 다른 데이터셋과 합쳐서 컨텍스트를 추가
    - #22 Static Joiner: 잘 안 바뀌는 정적 데이터와 조인(단순함)
    - #23 Dynamic Joiner: 실시간으로 변하는 스트리밍 데이터와 조인. 시간 경계 관리가 추가
    - 정적 → 동적 순서로 난이도가 올라가
- 패턴#24, #25 (Data Decoration) — 조인 없이 개별 레코드에 속성을 추가
    - #24 Wrapper: 데이터를 새로운 구조로 감싸기
    - #25 Metadata Decorator: 처리 시간, 출처 같은 메타정보 추가
    - Enrichment보다 가벼운 작업


### 5.1 데이터 보강 (Data Enrichment)
- 패턴#22 정적조인기(Static Joiner)
- 패턴#23 동적조인기(Dynamic Joiner)





## 패턴#22 정적조이너(Static Joiner)

### (1) 문제상황
raw 데이터는 이벤트 발생 시점의 기술적 정보만 담고 있어서, 비즈니스 분석에 필요한 컨텍스트가 부족하다.

예시 상황:
- Kafka → Spark 배치 잡 → Delta Lake로 매일 결제 트랜잭션을 적재
- `payments` 테이블에는 `user_id`, `amount`, `event_time`만 있음
- 비즈니스팀이 "유저 가입일과 결제 패턴의 관계를 분석하고 싶다"고 요청

근데 `payments` 테이블엔 가입일이 없어. 가입일은 `users` 테이블에 있어.
두 테이블을 합쳐야 하는데, `users` 테이블은 자주 바뀌지 않는 정적인 데이터야.


### (2) 솔루션

주요 컨셉: 잘 안 바뀌는 정적 데이터셋을 JOIN 키로 결합해서 컨텍스트를 추가한다.

구현 방법은 두 가지야.
- SQL JOIN — `user_id` 같은 공통 키로 두 테이블을 결합. 가장 일반적인 방법
- API 호출 — 외부 서비스에서 데이터를 가져와 결합. IP → 지역 정보 변환 같은 경우

근데 여기서 주의할 점이 있어.
`users` 테이블은 "지금 이 순간"의 최신 데이터야. 백필(과거 데이터 재처리)을 하면 1월 데이터를 처리할 때도 지금 기준의 유저 정보로 조인하게 돼. 1월에 이미 탈퇴한 유저 정보가 달라질 수 있어.

이걸 해결하는 게 SCD(Slowly Changing Dimensions, 천천히 변하는 차원)야. 
유저 정보가 바뀔 때마다 이력을 쌓아두는 방식이야.

SCD는 두 가지 타입을 주로 써.
- SCD Type 2 — 하나의 테이블에 유효 시작일/종료일 컬럼을 추가해서 이력 관리. 종료일이 비어있으면 현재 값
- SCD Type 4 — 테이블 두 개로 분리. 하나는 현재 값만, 하나는 전체 이력
멱등성이 필요하면 SCD를 써서 특정 시점의 유저 정보를 항상 동일하게 가져올 수 있게 해야 해.

SCD Type 2 예시:
`users` 테이블에 `valid_from`, `valid_to` 컬럼이 추가된 형태야.

|user_id|email|grade|valid_from|valid_to|
|---|---|---|---|---|
|1|[a@gmail.com](mailto:a@gmail.com)|BRONZE|2024-01-01|2024-06-01|
|1|[a@gmail.com](mailto:a@gmail.com)|SILVER|2024-06-01|NULL|
|2|[b@gmail.com](mailto:b@gmail.com)|GOLD|2024-03-01|NULL|

user_id=1인 유저는 원래 BRONZE였다가 2024-06-01에 SILVER로 바뀐 거야. `valid_to`가 NULL이면 현재 유효한 값이야.

2024-03-01 기준으로 백필하면 user_id=1의 grade는 BRONZE가 나와. 하나의 테이블 안에서 시점별로 조회 가능해.

---

SCD Type 4 예시:
테이블을 두 개로 분리해.
`users_current` (현재 값만):

|user_id|email|grade|
|---|---|---|
|1|[a@gmail.com](mailto:a@gmail.com)|SILVER|
|2|[b@gmail.com](mailto:b@gmail.com)|GOLD|

`users_history` (전체 이력):

|user_id|email|grade|valid_from|valid_to|
|---|---|---|---|---|
|1|[a@gmail.com](mailto:a@gmail.com)|BRONZE|2024-01-01|2024-06-01|
|1|[a@gmail.com](mailto:a@gmail.com)|SILVER|2024-06-01|NULL|
|2|[b@gmail.com](mailto:b@gmail.com)|GOLD|2024-03-01|NULL|

최신 데이터 조회는 `users_current`에서 빠르게, 이력이 필요하면 `users_history`에서 조회하는 방식이야.

실무에서는 Type 2가 더 많이 쓰여. 테이블 하나로 현재값과 이력을 모두 관리할 수 있고, dbt snapshot이 자동으로 관리해줘서 운영이 편하거든.

Type 4는 현재값 조회 성능이 중요한 경우에 써. 이력 테이블이 커져도 current 테이블 조회 속도는 유지되니까.


### (3) 결과

장점:
- 구현이 단순함. SQL JOIN 한 줄로 끝남
- 배치, 스트리밍 파이프라인 모두에서 사용 가능

단점 및 주의사항:
- 늦게 도착한 데이터(late data) 문제 — 유저 정보가 이벤트보다 늦게 들어오면 조인 결과가 비거나 부정확해짐
- 멱등성 문제 — SCD 없이 백필하면 재실행할 때마다 다른 결과가 나옴
- API 호출 방식은 네트워크가 병목. 반드시 bulk 요청으로 묶어서 호출해야 함
- 보강 데이터셋이 클수록 조인 비용이 커짐. Spark의 `broadcast` 옵션으로 작은 테이블을 모든 노드에 뿌려서 최적화 가능



### (4) 예시

엔지니어 독백:
> 결제 데이터 분석 요청이 왔는데, payments 테이블엔 user_id밖에 없고 유저 등급이나 가입일은 다 users 테이블에 있었어. 처음엔 그냥 JOIN 붙여서 끝냈는데, 나중에 백필 요청이 왔을 때 문제가 생겼어. 6개월 전 데이터를 재처리했더니 그 사이에 등급이 바뀐 유저들 결과가 달라진 거야. 그때 SCD를 도입했지.

Spark 배치 잡이 `users` 테이블(정적 보강 데이터)과 `payments` 테이블을 `user_id`로 조인한다.
```
payments = spark.read.parquet("s3://bucket/payments/")
users = spark.read.parquet("s3://bucket/users/")

# ★ 핵심: user_id 기준으로 정적 보강 데이터를 결합
enriched = payments.join(users, on="user_id", how="left_outer")
```

SCD Type 2를 적용하면 특정 시점의 유저 정보를 정확히 가져올 수 있다.
```
# ★ 핵심: event_time이 유저 정보의 유효 기간 안에 있는 행만 조인
enriched = payments.join(
    users,
    on=[
        payments.user_id == users.user_id,
        payments.event_time >= users.valid_from,
        (users.valid_to.isNull()) | (payments.event_time < users.valid_to)
    ],
    how="left_outer"
)
```

`valid_from`, `valid_to` 사이에 해당하는 유저 정보만 조인하기 때문에 백필해도 항상 같은 결과가 나온다.

엔지니어 팁:
> SCD 없이 운영하다가 백필 요청 받으면 그때 가서 SCD 도입하려면 과거 이력이 없어서 못 해. 처음 설계할 때 이력 관리가 필요한 테이블인지 미리 판단해두는 게 중요해.



### (5) 최신트렌드

요즘 실무에서 Static Joiner를 구현하는 방식은 두 가지야.
- dbt(data build tool) + SCD 매크로 (가장 선호)
    - dbt의 `snapshot` 기능이 SCD Type 2를 자동으로 관리해줌
    - `valid_from`, `valid_to` 컬럼을 dbt가 알아서 추가하고 갱신
    - SQL만 작성하면 되기 때문에 코드가 단순하고 이력 관리가 쉬워서 현업 1순위
- Delta Lake / Iceberg Time Travel
    - `TIMESTAMP AS OF`로 특정 시점의 테이블 스냅샷을 바로 조인 가능
    - SCD 테이블 따로 관리할 필요 없이 테이블 자체가 이력을 보관
    - Delta/Iceberg 환경이면 가장 간단한 방법
