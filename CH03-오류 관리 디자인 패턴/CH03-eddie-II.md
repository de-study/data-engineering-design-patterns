# 3.4 필터링

## 패턴 #14 필터 인터셉터

### 3.4.1 Problem — 필터 조건별 원인을 알 수 없다

#### 이 패턴이 속한 맥락

- 챕터: 3장 Error Management Design Patterns
- 섹션: Filtering
- 핵심 전제: 데이터 엔지니어링에서 에러는 기술적 장애만이 아니다.
  잘못 구현된 필터 조건으로 인한 **인간 실수(human error)** 도 에러다.

---

#### 필터링이란

- 데이터 파이프라인에서 비즈니스 조건에 맞는 레코드만 선택하는 가장 흔한 연산
- 문제는 "얼마나 필터됐는가"는 알 수 있어도,
  **"어떤 조건이 얼마나 필터했는가"는 알기 어렵다**는 것

---

#### 책의 시나리오

배치 잡 신버전을 운영에 배포한 직후, 필터된 데이터 비율이 **15% → 90%** 로 급증했다.

두 가지 가능성이 있다.
- 업스트림에서 들어오는 **데이터 자체가 이상해진 것**
- 우리 팀이 배포한 **Spark 잡 코드에 버그가 생긴 것**

Spark execution plan을 열었지만 원인을 알 수 없다.

```
Filter (((amount > 0) AND (status IS NOT NULL))
        AND ((merchant_id IS NOT NULL)
        AND (currency IN (KRW, USD, JPY))))

number of output rows: 1,800
```

Spark Catalyst Optimizer가 여러 filter 조건을 **하나로 합쳐버리기 때문**이다.
전체 통과 건수는 보이지만, 조건별로 몇 건이 걸렸는지는 알 수 없다.

---

#### 실무 시나리오 — 결제 트랜잭션 파이프라인

**파이프라인 구조**

- 매일 새벽 2시 배치 실행
- Kafka → Spark 배치 잡 → Delta Lake 적재
- Spark 잡에 필터 조건 4개 존재

**필터 조건 4개**

| 조건 | 목적 |
|---|---|
| `amount > 0` | 금액 0 이하 이상 데이터 제거 |
| `status IS NOT NULL` | 결제 상태값 누락 제거 |
| `merchant_id IS NOT NULL` | 가맹점 ID 누락 제거 |
| `currency IN ('KRW', 'USD', 'JPY')` | 지원 통화 외 제거 |

**알람 수신**

```
[ALERT] payments_pipeline
Daily valid record count: 12,000 → 1,800
Drop rate: 85%
```

**알람 해석**

- `Daily valid record count: 12,000 → 1,800`
  - 어제까지 하루 유효 결제 건수: 12,000건
  - 오늘 필터를 통과한 유효 결제 건수: 1,800건
  - "valid record" = 필터 조건을 모두 통과한 데이터. 전체 수신량이 아님
- `Drop rate: 85%`
  - 오늘 수신된 전체 데이터 중 85%가 필터에 걸려 날아갔다는 뜻
  - 반대로 말하면 15%만 Delta Lake에 적재됨

---

#### 엔지니어 독백 — 알람을 받는 순간부터

새벽 3시. 폰이 울렸어.

```
[ALERT] payments_pipeline
Daily valid record count: 12,000 → 1,800
Drop rate: 85%
```

알람을 받는 순간 머릿속에서 제일 먼저 하는 질문은 이거야.

**"이게 데이터 문제야, 아니면 내 코드 문제야?"**

이 두 가지를 구분하는 게 제일 중요해. 대응 방향이 완전히 달라지거든.

- 데이터 문제면 → Kafka로 결제 데이터를 보내주는 업스트림 팀에 연락해야 해
- 코드 문제면 → 우리 팀이 배포한 Spark 잡을 롤백하거나 핫픽스 해야 해

그 다음 질문은 이거야.

**"어제 뭐가 바뀌었지?"**

제일 먼저 Git 커밋 이력을 확인해.
"어제 우리 팀이 이 Spark 잡 코드를 수정해서 운영에 올린 게 있나?"

예를 들어 누군가 `currency IN ('KRW', 'USD', 'JPY')` 조건을 건드렸으면,
그게 의심 1순위야. 코드를 잘못 수정해서 정상 데이터까지 날리고 있을 수 있거든.

우리 팀 Git 이력에 아무것도 없으면 다음 질문은 이거야.

**"업스트림 팀이 자기네 결제 시스템을 어제 수정한 게 있나?"**

슬랙 채널에 물어보는 거야. "혹시 어제 배포한 거 있어요?" 하고.

근데 이걸 확인하는 데만 30분이 걸려.

나중에 알고 보니 업스트림 팀이 자기네 결제 시스템에서
통화코드를 `KRW` 대신 `krw` 소문자로 바꿔서 Kafka에 밀어넣도록 수정한 거였어.

우리 Spark 잡의 `currency IN ('KRW', 'USD', 'JPY')` 필터는 대문자만 통과시키니까,
소문자로 들어온 한국 결제 건 전부가 날아간 거지.

**조건별 카운트만 있었어도 5분 만에 잡을 수 있었던 거야.**

```
currency 조건 필터: 10,100건 제거  ← 이 숫자 하나만 있었어도
status 조건 필터: 80건 제거
merchant_id 조건 필터: 20건 제거
amount 조건 필터: 0건 제거
```







---

<br>


### 3.4.2 Solution — 필터에 카운터를 달아라

#### 핵심 아이디어

필터 조건을 그냥 쓰지 말고, 조건이 `False`가 될 때마다 카운터를 +1 하는 wrapper로 감싸라.
잡이 끝난 뒤 카운터를 읽으면 조건별로 몇 건이 걸렸는지 알 수 있다.

---

#### 구현 방식 2가지

| 방식 | 구현 방법 | 실무 선호 |
|---|---|---|
| Programmatic API | `mapInPandas` + Accumulator | 선호 |
| SQL | 서브쿼리 + `CASE WHEN` alias | 비선호 |

책은 SQL보다 Programmatic API가 더 낫다고 명시한다.
이유는 Consequences에서 다룬다.

---

#### 동작 원리

기존 필터:

```
row가 조건을 통과하면 → 살림
row가 조건을 실패하면 → 버림
```

Filter Interceptor 적용 후:

```
row가 조건을 통과하면 → 살림
row가 조건을 실패하면 → 해당 조건의 카운터 +1 → 버림
```

버리는 동작은 동일하다.
단지 버리기 전에 **"어느 조건에서 걸렸는지"** 기록을 남기는 것이다.

---

#### 실무 시나리오 적용 — 결제 트랜잭션 파이프라인

필터 4개에 각각 카운터를 붙인다.

```
amount > 0              → 위반 시 amount_counter   + 1
status IS NOT NULL      → 위반 시 status_counter   + 1
merchant_id IS NOT NULL → 위반 시 merchant_counter + 1
currency IN (...)       → 위반 시 currency_counter + 1
```

잡 종료 후 출력:

```
amount 조건 필터:      0건 제거
status 조건 필터:      80건 제거
merchant_id 조건 필터: 20건 제거
currency 조건 필터:    10,100건 제거  ← 범인
```

---

#### SQL 방식 구조

SQL은 함수를 직접 감쌀 수 없어서 다른 방식을 쓴다.

1. 필터 조건을 `CASE WHEN`으로 `status_flag` 컬럼으로 만든다
2. `status_flag IS NULL` 인 row만 유효 데이터로 적재한다
3. `status_flag IS NOT NULL` 인 row를 `GROUP BY` 로 집계해서 조건별 카운트를 뽑는다

임시 테이블을 한 번 더 만들어야 해서 Programmatic API보다 복잡하고 비용이 더 든다.

---

#### 엔지니어 독백

처음에 이 패턴 배웠을 때 "그냥 필터 전후 `count()` 찍으면 되는 거 아닌가?" 싶었어.

근데 그게 안 되는 이유가 있어.

`count()` 를 조건마다 따로 찍으려면 데이터를 조건 수만큼 여러 번 스캔해야 해.
필터 4개면 4번 스캔이야. 데이터가 수억 건이면 그게 바로 비용이고 시간이거든.

Filter Interceptor는 데이터를 **한 번만 스캔하면서** 동시에 4개 카운터를 다 올려. 그게 핵심이야.

그리고 실무에서 이 카운터 값을 그냥 로그로 출력하는 게 아니라,
Prometheus나 CloudWatch 메트릭으로 쏘는 게 맞아.
그래야 Grafana 대시보드에서 "오늘 currency 필터가 갑자기 튀었네"를 시각적으로 볼 수 있거든.
로그는 사고 난 다음에 보는 거고, 메트릭은 사고 나기 전에 보는 거야.

### 3.4.3 Consequences — 트레이드오프 3가지

책은 이 패턴을 쓸 때 대가가 따른다고 명시한다. 3가지다.

---

#### 1. Runtime Impact — 실행 시간 증가

wrapper를 씌우는 것 자체가 추가 연산이다.

- Programmatic API 방식: 카운터는 각 Executor 태스크 안에 로컬로 존재하는 단순한 정수값이다.
  태스크가 끝난 뒤 Driver로 네트워크를 통해 합산되는데, 이 비용은 작다. 실무적으로 무시할 수 있는 수준이다.
- SQL 방식: 임시 테이블을 별도로 생성해야 하기 때문에 추가 스캔이 발생한다. Programmatic API보다 비용이 크다.

실무 예시:

~~~
결제 데이터 1,000만 건 기준
Programmatic API 추가 비용: ~2~3초 수준
SQL 임시 테이블 방식 추가 비용: ~30초~1분 수준 (데이터 재스캔)
~~~

---

#### 2. Declarative Languages — SQL은 구현이 복잡하다

SQL은 함수를 직접 감쌀 수 없다.
그래서 CASE WHEN으로 status_flag 컬럼을 만들고, 그걸 기준으로 필터링과 집계를 분리해야 한다.

책은 명시적으로 이렇게 말한다.

> SQL만 쓰던 환경이더라도, Filter Interceptor만큼은 Programmatic API를 쓰는 게 낫다.

이유는 두 가지다.
- SQL 방식은 코드가 복잡하고 유지보수가 어렵다
- Programmatic API 방식은 조건 추가/삭제가 훨씬 간단하다

---

#### 3. Streaming — 스트리밍 잡에서는 더 복잡하다

배치 잡은 잡이 끝나면 카운터를 읽으면 된다. 스트리밍 잡은 끝이 없다.

두 가지 문제가 생긴다.

- **Stateless → Stateful 전환 필요**: 카운터를 누적하려면 상태를 저장해야 한다.
  원래 stateless였던 잡이 stateful 잡이 되면서 StateStore 관리 오버헤드가 생긴다.
- **시간 경계 정의 필요**: 스트리밍은 데이터가 계속 들어온다.
  카운터를 그냥 누적하면 "지금 현재 어느 조건이 문제인지"를 알 수 없다.
  10분 윈도우, 1시간 윈도우처럼 시간 단위로 집계해야 현재 상태를 파악할 수 있다.

실무 예시:

~~~
스트리밍 결제 파이프라인에서

잘못된 방식: currency 필터 카운터 누적값 = 50,000건
             → 이게 오늘 것인지, 지난 한 달 것인지 알 수 없음

올바른 방식: currency 필터 카운터 (최근 10분) = 8,500건
             currency 필터 카운터 (그 전 10분) = 120건
             → 10분 전부터 갑자기 튀었다는 걸 바로 알 수 있음
~~~

---

#### 정리 — 언제 어떤 방식을 써야 하나

| 상황 | 권장 방식 |
|---|---|
| 배치 잡 + PySpark | Programmatic API (mapInPandas + Accumulator) |
| 배치 잡 + SQL만 사용 가능 | SQL (CASE WHEN) — 단, 비용 감수 |
| 스트리밍 잡 | Programmatic API + 시간 윈도우 집계 필수 |

---

#### 엔지니어 독백

Consequences 읽으면서 "에이 그냥 SQL 쓰면 안 되나" 싶을 수 있어.

근데 실제로 SQL 방식으로 구현해보면 바로 느껴.
필터 조건 하나 추가할 때마다 CASE WHEN 블록 수정하고,
임시 테이블 쿼리 수정하고, GROUP BY 쿼리 수정하고... 3군데를 동시에 건드려야 해.
조건이 10개면 유지보수가 악몽이야.

Programmatic API는 FilterWithAccumulator 객체 하나만 딕셔너리에 추가하면 끝이야.
그게 실무에서 체감하는 차이야.

스트리밍에서 시간 경계 안 잡으면 진짜 큰일 나.
카운터가 100만이 됐는데 그게 오늘 1시간 동안 쌓인 건지, 지난 3개월 동안 쌓인 건지 모르면
알람 기준을 잡을 수가 없거든.
10분 윈도우로 끊어서 보는 습관을 처음부터 들여놔야 해.

---

### 3.4.4 Examples — 실제 구현

두 가지 구현 방식을 결제 파이프라인 시나리오에 맞춰 설명한다.

---

#### 방식 1. Programmatic API — mapInPandas + Accumulator

전체 흐름은 3단계다.

~~~
1단계: FilterWithAccumulator 클래스 정의 + 필터 조건 등록
2단계: mapInPandas 로 데이터 처리 (필터링 + 카운터 증가)
3단계: 잡 종료 후 카운터 값 수집
~~~

**1단계 — 클래스 정의 + 필터 조건 등록**

~~~
import dataclasses
from typing import Callable, Any
from pyspark import Accumulator

# FilterWithAccumulator: 필터 조건 하나 + 카운터 하나를 묶는 단위
@dataclasses.dataclass
class FilterWithAccumulator:
    name: str                         # 필터 이름 (로그용)
    filter: Callable[[Any], bool]     # 실제 필터 조건 함수
    accumulator: Accumulator[int]     # 조건 위반 시 +1 되는 카운터

# 컬럼별로 필터 조건 목록을 딕셔너리로 등록
# key   : 컬럼명 (내가 정하는 변수)
# value : 해당 컬럼에 적용할 FilterWithAccumulator 목록
filters_with_accumulators = {
    'amount': [
        FilterWithAccumulator(
            'amount is zero or negative',
            lambda row: row['amount'] > 0,       # True면 통과, False면 카운터 +1
            spark_context.accumulator(0)          # 초기값 0
        )
    ],
    'status': [
        FilterWithAccumulator(
            'status is null',
            lambda row: row['status'] is not None,
            spark_context.accumulator(0)
        )
    ],
    'merchant_id': [
        FilterWithAccumulator(
            'merchant_id is null',
            lambda row: row['merchant_id'] is not None,
            spark_context.accumulator(0)
        )
    ],
    'currency': [
        FilterWithAccumulator(
            'currency not supported',
            lambda row: row['currency'] in ('KRW', 'USD', 'JPY'),
            spark_context.accumulator(0)
        )
    ],
}
~~~

**2단계 — mapInPandas로 데이터 처리**
> mapInPandas란
> "Spark가 관리하는 분산 서버의 메모리 데이터를 파이썬의 Pandas DataFrame 형태로 통째로 변환해서 내 코드(함수)에 집어넣어 주는 연결 통로(엔진)"
> 
> 실무 시니어들은 분산 아키텍처를 유지하면서 파이썬/Pandas의 강력한 기능을 쓰고 싶을 때 무조건 이 `mapInPandas`를 선택하는 거야.
> 
> **JVM-Python 프로세스 간 이종(Heterogeneous) 데이터 전송 엔진**: 
> Spark의 핵심 연산이 일어나는 자바 가상 머신(JVM) 메모리 영역과, 
> 실제 네 파이썬 코드가 실행되는 파이썬 워커(Python Worker) 프로세스 간에 데이터를 
> 주고받는 구동체야.
> 
> "코드 추상화 레이어에서 '연결 통로'니 뭐니 가볍게 말하니까 그냥 파이썬 함수 호출하면 대충 데이터가 넘어가는 줄 아는데, 시스템 내부 뜯어보면 완전히 하드웨어 레벨 연산이야.

> 네가 짠 `valid_payments = input_dataset.mapInPandas(...)` 줄이 실행될 때, 분산 서버(Executor) 안에서는 두 개의 프로세스가 동시에 요동쳐. Spark 자체 엔진인 JVM 프로세스가 있고, 네가 작성한 `check_row` 파이썬 코드를 실행할 Python Worker 프로세스가 각각 독립되어 떠 있거든.

> 자바 메모리에 있는 데이터를 파이썬 프로세스로 넘겨야 네 파이썬 코드가 돌 거 아냐? 이때 데이터 가속 장치인 **Apache Arrow**가 개입하는 거야. 자바 쪽 힙(Heap) 메모리에 있는 데이터 물리 주소를 파이썬 파판다스가 바로 읽을 수 있게 메모리 매핑(Memory Mapping)을 새로 해버려. 복사 연산이 안 일어나니까 대용량 데이터도 순식간에 파이썬 프로세스로 넘어가는 거지.

> 그리고 그 데이터를 한 번에 다 던지면 파이썬 프로세스 메모리가 터지니까, 내부 인프라 장치가 '자, 1만 건짜리 Pandas DataFrame 1호기 간다', '처리 끝났어? 그럼 2호기 간다' 하면서 `Iterator` 형태로 쪼개서 집어넣어 주는 거야.

> 그러니까 `mapInPandas`를 정확히 지칭하자면, **'자바 분산 메모리 파티션을 파이썬 프로세스의 Pandas DataFrame 이터레이터로 물리 가속 변환해 주는 Spark 오퍼레이터 엔진'**이라고 명확하게 정의하는 게 맞아."

~~~
import pandas
from typing import Iterator

def filter_payments(payments_iterator: Iterator[pandas.DataFrame]):

    def check_row(payment_row):
        for column in payment_row.keys():
            if column not in filters_with_accumulators:
                continue
            for fwa in filters_with_accumulators[column]:
                if not fwa.filter(payment_row):   # 조건이 False면
                    fwa.accumulator.add(1)         # 카운터 +1  ← 핵심
                    return False                   # 이 row는 버림
        return True                               # 모든 조건 통과 → 살림

    for payments_df in payments_iterator:
        yield payments_df[
            payments_df.apply(lambda row: check_row(row), axis=1) == True
        ]

# parquet 파일에서 읽기
input_dataset = spark.read.parquet('s3://bucket/payments/raw/date=2026-05-23/')

# mapInPandas: 파티션 단위로 filter_payments 함수 실행
valid_payments = input_dataset.mapInPandas(filter_payments, input_dataset.schema)

# Delta Lake에 적재 → 이 시점에 실제 실행이 시작되고 카운터가 쌓임
valid_payments.write.mode('append').format('delta').save('s3://bucket/payments/valid/')
~~~

**3단계 — 카운터 값 수집**

~~~
# write 완전히 끝난 뒤에 읽어야 함
for column, fwa_list in filters_with_accumulators.items():
    for fwa in fwa_list:
        print(f'{column} // {fwa.name} // {fwa.accumulator.value}건 제거')
~~~

출력 결과:

~~~
amount      // amount is zero or negative // 0건 제거
status      // status is null             // 80건 제거
merchant_id // merchant_id is null        // 20건 제거
currency    // currency not supported     // 10100건 제거  ← 범인 즉시 특정
~~~

---

#### 방식 2. SQL — CASE WHEN + 임시 테이블

~~~
# 1단계: 각 row에 어느 필터에 걸렸는지 status_flag 컬럼을 붙인 임시 테이블 생성
spark.sql('''
    SELECT * FROM (
        SELECT
            CASE
                WHEN amount <= 0                         THEN 'invalid_amount'
                WHEN status IS NULL                      THEN 'null_status'
                WHEN merchant_id IS NULL                 THEN 'null_merchant_id'
                WHEN currency NOT IN ('KRW','USD','JPY') THEN 'invalid_currency'
                ELSE NULL
            END AS status_flag,
            amount, status, merchant_id, currency
        FROM payments_input
    )
''').createTempView('payments_with_flags')

# 2단계: 조건별 필터 카운트 집계
spark.sql('''
    SELECT status_flag, COUNT(*) as filtered_count
    FROM payments_with_flags
    WHERE status_flag IS NOT NULL
    GROUP BY status_flag
''').show()

# 3단계: 유효 데이터만 Delta Lake에 적재
(spark.sql('''
    SELECT amount, status, merchant_id, currency
    FROM payments_with_flags
    WHERE status_flag IS NULL
''')
.write.mode('append').format('delta').save('s3://bucket/payments/valid-sql/'))
~~~

2단계 출력 결과:

~~~
+------------------+--------------+
|status_flag       |filtered_count|
+------------------+--------------+
|null_status       |80            |
|null_merchant_id  |20            |
|invalid_currency  |10100         |
+------------------+--------------+
~~~

SQL 방식의 한계:
- CASE WHEN은 첫 번째로 매칭된 조건만 기록한다.
  amount도 0이고 currency도 잘못된 row는 invalid_amount 하나만 기록되고 invalid_currency는 누락된다.
- 임시 테이블 생성으로 데이터를 한 번 더 스캔한다

---

#### 엔지니어 독백

mapInPandas 처음 보면 복잡해 보여. 근데 구조는 단순해.
Spark가 파티션 하나를 Pandas DataFrame으로 만들어서 내 함수에 던져줘.
나는 그 안에서 row 하나씩 보면서 조건 검사하고 카운터 올리는 거야.

중요한 거 하나 짚어줄게.
accumulator.value는 반드시 valid_payments.write...가 완전히 끝난 다음에 읽어야 해.
Spark는 lazy evaluation이라서 mapInPandas를 정의하는 시점에는 실제 실행이 안 돼.
write를 호출하는 순간 실제 실행이 시작되고 카운터가 쌓여.
그 전에 accumulator.value 읽으면 전부 0이야.

SQL 방식은 CASE WHEN이 첫 번째 매칭에서 멈춘다는 것도 함정이야.
amount가 0이면서 currency도 잘못된 row가 있으면,
invalid_amount만 카운트되고 invalid_currency는 카운트 안 돼.
그래서 currency 문제를 과소평가할 수 있어.
Programmatic API는 조건을 모두 순회하니까 이 문제가 없어.




---


# 3.5 Fault Tolerance

## 패턴 #15 Checkpointer

### 3.5.1 Problem — 스트리밍 잡이 죽으면 어디서부터 다시 읽어야 하는가

#### 등장인물 정의

- Spark Structured Streaming 잡: Kafka에서 결제 이벤트를 읽어 Delta Lake에 적재하는 스트리밍 처리 잡
- Kafka: 결제 이벤트가 흘러들어오는 메시지 큐. offset으로 어디까지 읽었는지 추적
- Delta Lake: Spark Structured Streaming 잡이 처리 결과를 최종 적재하는 저장소

---

#### 배치 잡 vs 스트리밍 잡의 차이

배치 잡은 실패하면 그냥 다시 실행하면 된다.
처리할 데이터가 S3 파티션 같은 곳에 고정돼 있으니까 `date=2026-05-23` 파티션을 다시 읽으면 그만이다.

스트리밍 잡은 다르다.
Kafka 같은 append-only log에서 데이터가 계속 흘러들어온다.
Spark Structured Streaming 잡이 죽었다가 다시 살아나면 "어디서부터 다시 읽어야 하는가"를 알 방법이 없다.

---

#### 마이크로배치 1개의 동작 순서

~~~
1. Kafka에서 결제 이벤트 읽기  (예: offset 200~300)
2. 데이터 처리 (변환, 집계)
3. Delta Lake에 적재
4. 체크포인트에 "offset 300까지 완료" 기록
~~~

체크포인트는 마이크로배치가 완전히 끝난 후에 기록된다.

---

#### 잡이 죽는 시점별 케이스

**케이스 1 — 1~3번 사이에 Spark Structured Streaming 잡이 죽은 경우**

~~~
체크포인트: offset 200까지 완료 (이전 배치)
현재 배치 Delta Lake 적재: 안 됨 (잡이 죽었으니까)

→ 재시작 시 체크포인트 보고 offset 200부터 다시 읽음
→ 정상 재처리. 문제 없음
~~~

**케이스 2 — Delta Lake 적재는 완료됐는데 체크포인트 기록 전에 Spark Structured Streaming 잡이 죽은 경우**

~~~
체크포인트: offset 200까지 완료 (이전 배치)
현재 배치 Delta Lake 적재: 완료됨
체크포인트 기록: 안 됨 (죽어버림)

→ 재시작 시 체크포인트 보고 offset 200부터 다시 읽음
→ 200~300 결제 데이터가 Delta Lake에 중복 적재
~~~

---

#### 체크포인트가 없으면 어떻게 되나

체크포인트가 없으면 재시작 시 어디서부터 읽어야 할지 기준이 없다.

~~~
처음부터 읽으면  → 전체 데이터 중복 적재
최신부터 읽으면  → 장애 구간 데이터 유실
~~~

---

#### 엔지니어 독백

스트리밍 처음 짜면 다들 이거 간과해. "잡 죽으면 그냥 재시작하면 되지 않나?" 하고.

근데 재시작하는 순간 Kafka에서 어디서부터 읽을지가 문제야.
체크포인트가 없으면 시작점 자체를 모르니까 idempotency로도 못 막아.

케이스 1은 사실 문제가 아니야. 어차피 Delta Lake에 안 들어갔으니까 재처리하면 돼.
진짜 무서운 건 케이스 2야. 중복이 생기는데 이건 체크포인트만으로는 못 막아.
그래서 4장 idempotency 패턴이 반드시 필요한 거야.

---

### 3.5.2 Solution — 처리 진행상황을 외부 영구 저장소에 기록한다

#### 체크포인트의 정의

Spark Structured Streaming 잡이 "어디까지 처리했는지"를 잡 외부의 영구 저장소에 기록하는 메커니즘이다.
잡 내부 메모리에만 있으면 잡이 죽는 순간 사라지기 때문에 S3 같은 외부 저장소에 써둔다.

---

#### 뭘 저장하나

두 가지를 저장한다.

- Kafka offset: 어디까지 읽었는가
- 처리 상태(State): stateful 잡인 경우 지금까지 집계된 중간 결과

~~~
체크포인트 파일 내용 예시:
{"payments_topic": {"partition_0": 1500, "partition_1": 1320}}

→ Kafka payments_topic의
  partition 0은 offset 1500까지 처리 완료
  partition 1은 offset 1320까지 처리 완료
~~~

---

#### 구현 방식 2가지

**방식 1 — Data Processing Framework 기반 (실무 선호)**

Spark Structured Streaming, Apache Flink가 여기 해당한다.
엔지니어가 체크포인트 저장 위치만 설정하면 프레임워크가 마이크로배치마다 자동으로 기록한다.

~~~
.option('checkpointLocation', 's3://bucket/checkpoint/')
~~~

**방식 2 — Data Store 기반**

Kafka SDK나 Amazon Kinesis를 직접 쓸 때 해당한다.
엔지니어가 코드로 직접 커밋 시점을 제어해야 한다.

~~~
Kafka SDK  → __consumer_offsets 토픽에 offset 직접 커밋
Kinesis KCL → DynamoDB 테이블에 checkpoint 직접 기록
~~~

| 상황 | 권장 방식 |
|---|---|
| Spark Structured Streaming 잡 사용 | Framework 기반 (설정 한 줄로 끝) |
| Kafka SDK 직접 사용 | Data Store 기반 (직접 커밋 제어) |

---

### 3.5.3 Consequences — 트레이드오프 2가지

#### 1. Delivery Guarantee vs Latency 트레이드오프

체크포인트는 마이크로배치마다 S3에 파일을 쓰는 추가 작업이다.

| 체크포인트 주기 | 장점 | 단점 |
|---|---|---|
| 자주 (예: 10초) | 재시작 시 재처리할 데이터 적음 | S3 쓰기 횟수 많아서 잡이 느려짐 |
| 드물게 (예: 10분) | 잡이 빠름 | 재시작 시 재처리할 데이터 많음 |

stateful 잡은 비용이 더 크다.
offset 숫자 몇 개가 아니라 집계 중간 상태 전체를 S3에 써야 하기 때문이다.

---

#### 2. Exactly-Once는 환상이다

체크포인트는 exactly-once 느낌을 주지만 실제로는 아니다.

케이스 2처럼 Delta Lake 적재 완료 후 체크포인트 기록 전에 Spark Structured Streaming 잡이 죽으면
재시작 시 중복이 발생한다. 이를 해결하려면 4장 idempotency 패턴을 함께 써야 한다.

**Delivery Mode 3가지**

| 모드 | 체크포인트 시점 | 결과 |
|---|---|---|
| Exactly-once | 적재와 동시에 원자적으로 | 중복도 유실도 없음. 구현 어려움 |
| At-least-once | 적재 완료 후 체크포인트 | 재시작 시 중복 발생 가능. 실무에서 가장 흔함 |
| At-most-once | 체크포인트 먼저 후 적재 | 재시작 시 데이터 유실. 실무에서 쓰면 안 됨 |

Spark Structured Streaming은 기본적으로 at-least-once다.

---

#### 엔지니어 독백

"체크포인트 있으면 exactly-once 아닌가요?" 라고 묻는 신입이 많아.

아니야. 체크포인트는 "어디서부터 다시 시작할지"를 아는 것이고,
exactly-once는 "같은 데이터가 딱 한 번만 적재되는 것"이야. 이 둘은 달라.

Flink의 `EXACTLY_ONCE` 옵션도 마찬가지야.
Flink 내부 state 집계에서 같은 데이터가 두 번 반영되는 걸 막는 것이지,
Delta Lake 같은 외부 저장소 중복 적재를 막는 게 아니야.

~~~
Flink 내부 state  → Flink가 exactly-once 보장
         ↓
    Delta Lake     → Flink 책임 끝. 내가 직접 idempotency 구현해야 함
~~~

지 입장에서만 exactly-once인 거야. 경계를 넘어가는 순간 Flink의 책임이 끝나.

at-most-once는 절대 쓰면 안 돼. 결제 데이터 유실은 금전적 손실이야.
at-least-once로 중복이 생기는 게 훨씬 낫고, 그 중복을 idempotency로 처리하는 게 정석이야.

---

### 3.5.4 Examples — Spark Structured Streaming vs Flink 구현

#### 구현 1 — Spark Structured Streaming

마이크로배치 단위로 체크포인트를 자동 기록한다.
엔지니어가 할 일은 `checkpointLocation` 설정 한 줄이다.

~~~
write_query = (
    input_stream_data
    .writeStream
    .outputMode('update')
    .option('checkpointLocation', 's3://bucket/payments/checkpoint/')  # ← 핵심
    .foreachBatch(process_payments)
    .start()
)
~~~

실제 체크포인트 파일:

~~~
$ cat s3://bucket/payments/checkpoint/offsets/18

{"payments_topic": {"0": 1276, "1": 1224}}

→ 마이크로배치 18번 완료 시점에
  partition 0 → offset 1276까지 처리 완료
  partition 1 → offset 1224까지 처리 완료
~~~

재시작 시 Spark Structured Streaming 잡이 이 파일을 읽어서
partition 0은 1276부터, partition 1은 1224부터 Kafka를 다시 읽는다.

---

#### 구현 2 — Apache Flink

마이크로배치 단위가 아닌 시간 기반으로 체크포인트를 기록한다.
체크포인트 주기, 모드, 파일 유지 여부를 엔지니어가 직접 설정해야 한다.

~~~
checkpoint_interval_30_sec = 30000   # 단위: ms, 30초마다 체크포인트

env.enable_checkpointing(
    checkpoint_interval_30_sec,
    mode=EXACTLY_ONCE        # Flink 내부 state 중복 반영 방지 (외부 저장소 아님)
)

env.get_checkpoint_config().enable_externalized_checkpoints(
    RETAIN_ON_CANCELLATION   # 잡 취소/장애 시에도 체크포인트 파일 유지
)
~~~

옵션 설명:
- `EXACTLY_ONCE`: Flink 내부 state 집계에서 같은 데이터가 두 번 반영되는 걸 방지. Delta Lake 같은 외부 저장소 중복은 막지 못함
- `RETAIN_ON_CANCELLATION`: 기본적으로 Flink는 잡이 죽으면 체크포인트 파일을 삭제함. 이 옵션을 켜야 재시작 시 복구 가능

---

#### Spark Structured Streaming vs Flink 비교

| 항목 | Spark Structured Streaming | Apache Flink |
|---|---|---|
| 체크포인트 시점 | 마이크로배치 완료 후 자동 | 설정한 시간 주기마다 |
| 엔지니어 설정 | `checkpointLocation` 경로만 | 주기 + 모드 + 파일 유지 여부 |
| 기본 체크포인트 파일 유지 | 유지됨 | 삭제됨 (RETAIN_ON_CANCELLATION 필요) |

---

#### 엔지니어 독백

Spark Structured Streaming 쓰면 체크포인트는 진짜 쉬워.
`checkpointLocation` 하나만 S3로 잡으면 끝인데, 이걸 로컬로 잡는 실수를 많이 해.

~~~
.option('checkpointLocation', '/tmp/checkpoint/')    # 틀림 — Executor 재시작 시 소멸
.option('checkpointLocation', 's3://bucket/ckpt/')   # 맞음 — 영구 저장
~~~

Flink는 `RETAIN_ON_CANCELLATION` 빼먹으면 장애 났을 때 체크포인트 파일이 사라져버려.
복구하러 들어갔더니 체크포인트가 없는 거야. 반드시 켜놔야 해.

그리고 체크포인트 경로는 잡마다 반드시 다르게 잡아야 해.

~~~
s3://bucket/checkpoint/payments_pipeline/   # 결제 파이프라인
s3://bucket/checkpoint/user_pipeline/       # 유저 파이프라인
~~~

같은 경로를 공유하면 두 잡의 체크포인트가 섞여서
재시작할 때 잘못된 offset을 읽는 대참사가 나.
