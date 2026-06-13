## 5.2 데이터 데코레이션(Data Decoration)

- 패턴#25 래퍼(Wrapper)
- 패턴#26 메타데이터 데코레이터(Metadata Decorator)

> 5.1은 외부데이터셋 조인
> 5.2는 실행되는 잡에서 파생컬럼 생성



## 패턴#26 메타데이터 데코레이터(Metadata Decorator)
### (1)문제상황

`visits` 데이터를 Kafka에 쓰는 Spark Structured Streaming 잡이 있다. 이 잡은 매주 버전이 올라간다.

```
v1.0.3 배포 → v1.0.4 배포 → v1.0.5 배포 ...
```

각 버전마다 데이터 가공 로직(예: `page_referral_key` 생성 방식)이 바뀐다. 어느 날 분석팀이 "집계값이 이상하다"고 한다. 이게 실제 데이터 패턴 변화인지, 아니면 특정 버전 배포 이후 로직이 바뀌어서 생긴 차이인지 구분해야 한다. 그러려면 각 레코드가 어떤 `job_version`에서 만들어졌는지 알아야 한다.

요구사항(이미 정해진 것): `job_version`, `batch_version`을 레코드에 남겨야 한다.

진짜 문제(여기서 풀어야 하는 것): 이 정보를 **어디에** 남기느냐다.

Kafka 레코드는 3칸으로 구성된다.

```
Kafka 레코드 = [ key 칸 | value 칸 | headers 칸 ]
```

- `value 칸` - 컨슈머(분석팀 코드)가 `record.value()`로 읽는 부분
- `headers 칸` - `record.headers()`로 따로 접근해야 하는 부분, 분석팀 코드는 보통 안 읽음

패턴#25 래퍼로 `job_version`을 넣으면:

```
Kafka 레코드:
  key 칸    = "v1001"
  value 칸  = {
                "raw": {"visit_id": "v1001", "page": "/home", ...},
                "decorated": {"job_version": "v1.0.3", "batch_version": 42}
              }
  headers 칸 = (비어있음)
```

`job_version`이 `decorated`라는 이름으로 포장되어 있을 뿐, 여전히 분석팀이 읽는 `value 칸` 안에 있다. 분석팀이 `record.value()`를 읽으면 `decorated.job_version`이 그대로 보이고, "이건 또 뭐지?"라는 불필요한 질문이 생긴다.

즉 풀어야 할 문제는: `job_version`, `batch_version`을 `value 칸`에서 완전히 빼서, 분석팀이 읽는 데이터에는 존재하지 않게 하면서도 디버깅이 필요할 때는 꺼내볼 수 있게 하는 것이다.




### (2)솔루션

주요컨셉: 기술 정보(`job_version` 등)를 `value 칸`이 아니라 레코드의 메타데이터 영역(Kafka의 `headers 칸`, 또는 그에 대응하는 영역)에 따로 둔다.

```
Kafka 레코드:
  key 칸    = "v1001"
  value 칸  = {"visit_id": "v1001", "page": "/home", "event_time": "...", ...}
              (job_version, batch_version 없음)
  headers 칸 = {"job_version": "v1.0.3", "batch_version": "42"}
```

분석팀이 `record.value()`를 읽으면 `job_version`은 전혀 보이지 않는다. 디버깅이 필요한 엔지니어만 `record.headers()`로 따로 꺼낸다.

데이터 스토어별로 "메타데이터 영역"의 정체가 다르다.

- Kafka - `headers` (key-value 목록, 네이티브 지원)
- 오브젝트 스토어(S3) - 파일 단위 태그(tag)

```
s3://bucket/visits/2025-01-10/part-001.parquet
  파일 내용(value에 해당): visit_id, page, event_time, ... (job_version 없음)
  태그(headers에 해당): { job_version: "v1.0.3", batch_version: "42" }
```

- DB/DW (headers/tag 미지원) - 두 가지 방법
    - 방법A: 같은 테이블에 `processing_context` 컬럼 추가 + 일반 사용자는 이 컬럼을 뺀 VIEW로 조회
    - 방법B: 메타데이터를 완전히 별도 테이블로 분리


방법B (별도 테이블 분리) 예시를 표로 다시 작성한다.
**visits 테이블 (분석팀 조회)**

|visit_id|event_time|page|context_id|
|---|---|---|---|
|v1001|2025-01-10T10:00:00Z|/home|42|

**visits_context 테이블 (기술팀 전용 스키마, 분석팀 접근 차단)**

|context_id|job_version|loading_time|
|---|---|---|
|42|v1.0.3|2025-01-10T10:02:00Z|

- 분석팀: `visits` 테이블만 조회 권한 보유
- 기술팀: `visits_context` 테이블 접근 권한 보유 (분석팀은 접근 불가)

패턴#25 래퍼와 차이
- 래퍼 : **`value 칸`** 안에 `decorated`로 포장해서 넣고(노출됨), 
- 메타데이터 데코레이터 : **`value 칸` 밖**(`headers`/태그/별도테이블)에 넣는다(비노출).

### (3)결과

장점:

- `value 칸`(본문 스키마)을 그대로 유지하면서 디버깅용 정보 확보
- Kafka처럼 네이티브 지원되는 경우 구현이 매우 단순

단점 및 주의사항:

- 스토어가 메타데이터 영역을 지원 안 하면 구현 비용 증가. 예: Amazon Kinesis Data Streams는 `headers` 자체가 없음
- DB 방식(방법A/B)은 추가 컬럼/테이블 + 권한/VIEW 관리가 필요
- 메타데이터에 비즈니스 정보(예: 결제금액, 배송지)를 절대 넣으면 안 된다. 분석팀은 `value`만 조회하므로, `headers`/태그에 숨긴 비즈니스 정보는 영원히 안 쓰이는 죽은 데이터가 됨

### (4)예시

엔지니어 독백:

> 매주 금요일 배포하는데, 어느 주에 집계값이 이상하다는 얘기가 들어왔어. `value`엔 버전 정보가 없으니 "v1.0.4 배포 이후 데이터만 봐줘" 같은 요청에 답할 수가 없었지. 그래서 Kafka에 쓸 때 `headers`에 `job_version`을 박기 시작했어. 핵심은 분석팀이 읽는 `value`는 절대 안 건드린다는 거야.

`visits` 데이터를 Kafka에 쓸 때 `job_version`, `batch_version`을 `headers`에 붙이는 코드:

```python
from pyspark.sql import functions as F

visits_with_metadata = visits_to_save.withColumn(
    "headers",
    F.array(
        F.struct(F.lit("job_version").alias("key"), 
        F.lit(job_version).alias("value")),
        F.struct(F.lit("batch_version").alias("key"), 
        F.lit(str(batch_number).encode("utf-8")).alias("value"))
    )
)

(visits_with_metadata.write.format("kafka")
    .option("kafka.bootstrap.servers", "localhost:9094")
    .option("includeHeaders", True)
    .option("topic", "visits-decorated")
    .save())
    
# ★핵심: includeHeaders=True를 줘야 위에서 만든 headers 컬럼이
#        출력 레코드의 value가 아니라 Kafka의 headers 영역으로 분리되어 들어간다.
#        즉 headers 컬럼 자체는 visits-decorated 토픽의 value 스키마에 포함되지 않음.
```


DB 방식(방법B)에서 기술 테이블을 만드는 DDL:
```sql
CREATE TABLE dedp.visits_context (
    execution_date_time TIMESTAMPTZ NOT NULL,
    loading_time TIMESTAMPTZ NOT NULL,
    code_version VARCHAR(15) NOT NULL,
    loading_attempt SMALLINT NOT NULL,
    PRIMARY KEY (execution_date_time)
);
-- ★핵심: visits_context는 기술팀 전용 스키마(예: dedp_internal)에 두고,
--        분석팀이 조회하는 visits 테이블에는 context_id 컬럼만 추가해서 연결한다.
--        분석팀에게는 visits_context 테이블 접근 권한을 차단.
```

### (5)최신트렌드

요즘은 `job_version` 같은 정보를 잡 코드에서 `headers`/태그로 직접 박기보다, 데이터 카탈로그·옵저버빌리티 도구로 분리해서 관리한다.

- OpenLineage - 잡 실행 메타데이터(버전, 실행시각, 입출력 데이터셋)를 별도 이벤트로 발행. `value`/레코드 자체와 완전히 분리됨
- Delta Lake / Iceberg 테이블 속성(table properties) - 커밋(commit) 단위로 작성 엔진/버전 정보가 자동 기록됨 (snapshot metadata)
- Datadog, Monte Carlo 같은 옵저버빌리티 툴 - `job_version`을 레코드 메타데이터가 아니라 모니터링 시스템의 태그/라벨로 관리

- 예전 방식: 잡 코드 안에서 직접 `headers`/태그/별도컬럼을 만들어서 "버전 추적용 메타데이터"를 수동 구현
- 요즘 방식: OpenLineage, Delta/Iceberg table properties, Datadog/Monte Carlo 같은 플랫폼/툴이 이 추적을 기본 기능으로 제공 → 직접 구현 안 해도 됨


<br><br><br>






## 5.3 데이터 집계(Data Aggregation)

- 패턴#27 분산 집계기(Distributed Aggregator)
- 패턴#28 로컬 집계기(Local Aggregator)

5.2(데이터 데코레이션)는 레코드 1개에 속성을 추가하는 작업이었다. 
5.3(데이터 집계)는 여러 레코드를 grouping key 기준으로 묶고, 
reduce 연산(count, avg 등)으로 줄이는 작업이다.
즉 입력 레코드 수보다 출력 레코드 수가 줄어든다는 점에서 5.2와 다르다.


## 패턴#27 분산 집계기(Distributed Aggregator)

### (1)문제상황

Bronze 레이어의 raw `visits` 이벤트를 Silver로 정제했고, 이제 대시보드용 OLAP 큐브를 만들어야 한다. 요구사항은 `device_type`별 평균 방문 시간(`avg duration`)을 구하는 것이다.

문제는 이 `visits` 데이터가 클러스터의 여러 노드(서버)에 물리적으로 나뉘어 저장돼 있다는 점이다.

Node1에 있는 데이터:

|visit_id|device_type|duration_sec|
|---|---|---|
|v1|mobile|30|
|v2|desktop|120|

Node2에 있는 데이터:

|visit_id|device_type|duration_sec|
|---|---|---|
|v3|mobile|50|
|v4|desktop|80|

`device_type=mobile`의 평균을 구하려면 `v1`(Node1)과 `v3`(Node2)을 같은 곳에서 봐야 한다. 근데 둘은 다른 노드에 있다. 한 노드의 메모리/디스크만 봐서는 `mobile`의 평균을 계산할 수 없다.

### (2)솔루션

주요컨셉: Shuffle
- `device_type`이 같은 레코드들을 네트워크로 한 노드에 모은 뒤(shuffle), 
- 그 노드에서 reduce(평균 계산)를 수행한다.

이 "모으는 동작"을 shuffle이라고 부른다. 위 예시에서는 `v1`을 Node2로 보내거나 `v3`을 Node1로 보내서, `mobile` 레코드 2개를 한 노드에 모은다.

#### 실제 구조
```
EMR/Databricks 클러스터
├─ Driver (1대) - 실행 계획 짜고 작업 분배
├─ Worker Node1 (EC2 인스턴스 1대) - Executor 프로세스 실행
└─ Worker Node2 (EC2 인스턴스 1대) - Executor 프로세스 실행
```

`visits` 데이터는 S3에 여러 파일(파티션)로 쪼개져 저장돼 있다.
```
s3://bucket/visits/part-0000.parquet  (v1, v2 들어있음)
s3://bucket/visits/part-0001.parquet  (v3, v4 들어있음)
```

Spark 잡을 실행하면:
- Driver가 "이 파일은 Node1 Executor가, 저 파일은 Node2 Executor가 읽어라"고 작업을 분배
- Node1 Executor → `part-0000.parquet`를 읽어서 자기 메모리에 올림 (v1, v2)
- Node2 Executor → `part-0001.parquet`를 읽어서 자기 메모리에 올림 (v3, v4)


```
Shuffle 후 (mobile을 Node1에 모은 경우)

Node1: v1(mobile, 30), v3(mobile, 50) → reduce → avg = 40
Node2: v2(desktop, 120), v4(desktop, 80) → reduce → avg = 100
```

코드 자체는 작은 로컬 데이터셋을 다룰 때와 똑같다 - `groupBy('device_type').avg('duration_sec')`. 
다만 내부적으로 shuffle이라는 네트워크 교환 단계가 추가로 들어간다.

raw 레코드를 그대로 shuffle하는 대신, 
가능하면 부분 집계(partial aggregation)를 먼저 하고 그 결과만 shuffle하면 더 효율적이다. 
예를 들어 count는 각 노드에서 부분 count를 먼저 구하고, 그 숫자들만 모아서 합산하면 된다.

이 패턴은 2004년에 등장한 MapReduce 프로그래밍 모델의 대표 예시다. 
Hadoop MapReduce(디스크 기반) → Apache Spark(메모리 기반)으로 구현이 진화해왔다.


### (3)결과
- 네트워크 교환 2단계 발생 - 
	1) 입력 데이터를 각 노드로 가져오는 단계, 
	2) 같은 키를 한 노드로 모으는 shuffle 단계. 
	 - 둘 다 레이턴시 이슈의 원인이 될 수 있음
- 데이터 스큐(skew) 
	- 특정 키(예: `device_type=mobile`)의 레코드 수가 압도적으로 많으면, 그 키를 처리하는 노드 1개가 병목이 됨
    - 대응: salting - grouping key에 임의값(salt)을 붙여서 1차 그룹핑 후, salt를 제거하고 2차 그룹핑으로 재집계
    - 또는 Spark의 AQE(Adaptive Query Execution)가 자동으로 skew를 감지하고 분할
- 스케일링
	- reduce가 끝난 노드도 fault tolerance 때문에 작업이 끝날 때까지 자원이 반납되지 않을 수 있음
    - 대응: shuffle service(예: Spark External Shuffle Service, GCP Dataflow Shuffle)를 별도 컴포넌트로 분리해서, 연산 노드는 끝나면 즉시 반납 가능하게 함

### (4)예시

엔지니어 독백:

> visits와 devices가 PostgreSQL, 파일시스템(JSON)으로 물리적으로 분리돼 있었는데, Spark는 이걸 그냥 join 코드 한 줄로 합쳐줘. 근데 운영 중에 특정 join이 느려서 `explain()`을 찍어봤더니 `Exchange hashpartitioning` 노드가 두 군데나 있더라고. 그게 shuffle이 두 번 일어난다는 신호야. 그때부터 실행계획에서 Exchange 개수를 꼭 확인하는 버릇이 생겼어.

PostgreSQL(`devices`)과 JSON 파일(`visits`)을 join해서 분산 집계하는 코드:

```python
visits = spark_session.read.json(f"{base_dir}/input-visits")

devices = spark_session.read.jdbc(
    url="jdbc:postgresql:dedp",
    table="dedp.devices",
    properties={"user": "dedp_test", "password": "dedp_test", "driver": "org.postgresql.Driver"}
)

visits_with_devices = visits.join(
    devices,
    [devices.type == visits.context.technical.dev_type,
     devices.version == visits.context.technical.dev_version],
    "inner"
)
# ★핵심: visits(JSON)와 devices(PostgreSQL)는 물리적으로 분리된 데이터셋.
#        join 키(dev_type, dev_version)가 같은 레코드를 한 노드로 모으기 위해
#        양쪽 모두 shuffle(Exchange)이 발생한다.
```

실행계획으로 shuffle 확인:
```python
visits_with_devices.explain()
```

```
== Physical Plan ==
AdaptiveSparkPlan isFinalPlan=false
+- SortMergeJoin [ctx.technical.dev_type, ctx.technical.dev_version], ...
   :- Sort [...]
   :  +- Exchange hashpartitioning(ctx.technical.dev_type, ...)   <- shuffle #1
   :     +- Filter (...)
   :        +- FileScan json [...]
   +- Sort [...]
      +- Exchange hashpartitioning(type, version, 200), ...        <- shuffle #2
         +- Scan JDBCRelation(...)
```

엔지니어 독백 (로그 해석):

> `Exchange hashpartitioning`이 2번 보이면, "양쪽 데이터셋 모두 join 키 기준으로 네트워크 재배치가 일어난다"는 뜻이야. 데이터 양이 한쪽이 훨씩 작으면(예: devices가 작은 차원 테이블), 이걸 broadcast로 바꿔서 Exchange 1개를 없애는 게 첫 번째 최적화 포인트야.

데이터 스큐 대응 salting 예시 (`device_type=mobile`이 압도적으로 많은 경우):

```python
from pyspark.sql.functions import rand

salted = visits.withColumn("salt", (rand() * 3).cast("int"))

partial = salted.groupBy("device_type", "salt").avg("duration_sec")
# ★핵심: salt 컬럼을 추가해서 mobile 키를 3개(salt=0,1,2)로 인위적으로 분산.
#        같은 device_type이라도 여러 노드에서 병렬로 1차 집계됨.

final = partial.groupBy("device_type").avg("avg(duration_sec)")
# 2차 집계로 salt를 제거하고 최종 device_type별 결과로 합산
```

### (5)최신트렌드

- Spark AQE(Adaptive Query Execution) - 런타임에 실제 파티션 크기를 보고 skew join을 자동 분할. 수동 salting을 안 해도 되는 경우가 많아짐
- Remote Shuffle Service(Apache Celeborn, Apache Uniffle 등) - shuffle 데이터를 연산 노드가 아닌 별도 storage로 분리. Kubernetes처럼 노드가 자주 뜨고 죽는 환경에서 shuffle 데이터 손실/재계산 문제를 줄임
- 클라우드 DW(BigQuery, Snowflake, Redshift) - 사용자는 `GROUP BY`만 작성하고, shuffle/분산 실행은 엔진이 완전히 숨김. 데이터 엔지니어가 Exchange를 직접 신경 쓸 일이 줄어드는 방향



<br><br><br>






## 패턴#28 로컬 집계기(Local Aggregator)

패턴#27 분산 집계기는 shuffle(네트워크로 같은 키를 한 노드에 모으는 작업)이 필수였다. 패턴#28은 이 shuffle을 없애는 게 핵심이다.

### (1)문제상황

스트리밍 잡이 Kafka `visits` 토픽에서 이벤트를 읽어서, `visit_id` 기준으로 세션(방문한 페이지 목록)을 집계해야 한다.

여기서 두 개념을 구분해야 한다.

- `visit_id` - 사용자의 한 번의 방문(세션)을 식별하는 엔티티 ID. 한 번의 방문 동안 본 여러 페이지뷰 이벤트가 같은 `visit_id`를 공유한다. 실제로는 `v1001`, `v1002`, ... 처럼 수천~수만 개가 존재한다
- 파티션 - `visits` 토픽을 물리적으로 나눈 저장 공간 개수. 이 토픽은 파티션 4개로 고정 생성되어 있다. `visit_id` 개수(수만 개)와 파티션 개수(4개)는 전혀 다른 숫자다

Producer는 `visit_id`를 partition key로 사용해서 쓴다. 즉 `hash(visit_id) % 4`로 어느 파티션에 쓸지 정해지고, 같은 `visit_id`는 항상 같은 파티션에 들어간다.

Partition0:

|visit_id|event_time|page|
|---|---|---|
|v1001|10:00:00|/home|
|v1001|10:00:05|/products|
|v1005|10:01:00|/home|

Partition1:

|visit_id|event_time|page|
|---|---|---|
|v1002|10:00:10|/home|
|v1002|10:00:20|/cart|
|v1002|10:00:30|/checkout|

목표는 이런 결과를 만드는 것이다.

|visit_id|pages|
|---|---|
|v1001|[/home, /products]|
|v1002|[/home, /cart, /checkout]|
|v1005|[/home]|

`v1001`의 이벤트(`/home`, `/products`)는 항상 Partition0에만 들어오고, 
`v1002`의 이벤트는 항상 Partition1에만 들어온다. 
즉 `visit_id`로 그룹핑할 때 필요한 데이터가 이미 같은 파티션 안에 다 모여있다.

패턴#27 분산 집계기 방식대로 `groupBy('visit_id')`를 쓰면, 
Spark는 이 사실을 모른 채로 모든 레코드를 다시 shuffle해서 재배치한다. 
파티션 수가 고정이고 데이터 분포도 안정적인 상황에서, 
`v1001` 데이터를 Partition0에서 다른 파티션으로 옮기는 이 shuffle은 불필요한 작업이다.

질문: Producer가 이미 `visit_id` 기준으로 파티셔닝해서 써준 데이터를, Consumer가 shuffle 없이 그대로 활용할 수 있는 방법은 없을까?



### (2)솔루션

주요컨셉: shuffle 없이, 각 노드가 자기 파티션 데이터만으로 `visit_id`별 집계를 끝내고, 결과를 합치지 않고 그대로 출력한다.

- 전제 (1)에서 확인한 사실
    - 같은 `visit_id`의 모든 이벤트는 항상 같은 파티션에 들어옴
    - Node1이 Partition0에서 만든 `v1001 → [/home, /products]` 결과는 그 자체로 완결됨
    - Node2가 `v1001` 관련 데이터를 가질 가능성 없음 → 결과 합치는 단계 불필요
- 구현 주체 (프레임워크별 차이)
    - Kafka Streams: 자동
        - `groupByKey()`가 "파티션에 해당 key 데이터가 다 모여있음"을 인식
        - shuffle 없이 로컬 집계 → 바로 output topic 전송
    - Spark: 직접 구현 필요
        - 1단계: `sortWithinPartitions('visit_id', 'event_time')` - 파티션 내부 정렬 (노드 간 이동 없음)
        - 2단계: `foreachPartition(누적_함수)` - `visit_id` 바뀌는 시점마다 이전 세션 완성 → output topic 전송


```
sorted_visits = visits_to_save.sortWithinPartitions(["visit_id", "event_time"])
# ★핵심: Partition0 내부에서만 정렬 (v1001, v1001, v1005 순). 노드 간 이동 없음

def write_records_from_spark_partition_to_kafka_topic(visits):
    kafka_writer = KafkaWriter(...)
    for visit in visits:
        kafka_writer.process(visit)
    kafka_writer.close()

sorted_visits.foreachPartition(write_records_from_spark_partition_to_kafka_topic)
# ★핵심: Node1은 Partition0에 대해, Node2는 Partition1에 대해 각각 독립 실행
#        서로 결과를 주고받지 않음 (merge 단계 없음)
```


```
class KafkaWriter:
    def __init__(self, bootstrap_server, output_topic):
        self.in_flight_visit = {"visit_id": None}

    def process(self, row):
        if row.visit_id != self.in_flight_visit["visit_id"]:
            send_visit_to_kafka(self.in_flight_visit)  # 이전 visit_id 완성 → 전송
            self.in_flight_visit = {"visit_id": row.visit_id, "pages": []}
        self.in_flight_visit["pages"].append(row.page)
        # ★핵심: visit_id가 바뀌는 시점(v1001 → v1005) = v1001 세션 종료 신호
        #        sortWithinPartitions로 정렬돼 있어야 이 판단이 성립
```

### (3)결과

- 파티션 수 변경 시 깨짐
    - 같은 `visit_id`가 항상 같은 파티션에 들어간다는 보장이 전제
    - 파티션 4개 → 8개로 변경 시 `hash(visit_id) % 8` 재계산되어 `v1001`이 다른 파티션으로 이동 가능
    - 한 `visit_id`의 데이터가 여러 파티션에 흩어지면 패턴#28 전제 깨짐
    - 대응: 전체 데이터 재배치 필요. 스트리밍은 기존 파티션 처리 완료 후 전환해야 해서(stop-the-world) 운영 부담 큼
- Grouping key가 컨슈머마다 다를 때
    - Producer는 `visit_id` 하나의 기준으로만 파티셔닝
    - 다른 컨슈머 B가 `user_id` 기준 집계 필요 시, B 입장에서는 같은 `user_id`의 데이터가 여러 파티션에 흩어져있음
    - 패턴#28 전제 충족 불가
    - 대응: B는 패턴#27(분산 집계기)로 shuffle 받아들여야 함

### (4)예시

엔지니어 독백:

> visits 토픽을 처음부터 visit_id로 파티셔닝해서 쓰게 만든 게 신의 한 수였어. 세션화 잡 짤 때 `groupBy('visit_id')` 쓰면 매번 shuffle이 떴는데, `sortWithinPartitions` + `foreachPartition`으로 바꾸니까 실행계획에서 Exchange가 사라졌어. 다만 나중에 파티션 수를 늘리는 이슈가 생겼을 때, 이 전제가 깨진다는 걸 미리 문서화해두지 않아서 한참 헤맸어.

DW에서의 로컬 집계 예시 - AWS Redshift 분산 방식 설정:
```sql
CREATE TABLE visits (
    visit_id INT,
    user_id INT
)
DISTSTYLE KEY,
DISTKEY(visit_id);
-- ★핵심: DISTKEY(visit_id) → 같은 visit_id는 항상 같은 노드에 저장됨.
--        visit_id 기준 집계/조인은 노드 간 이동(shuffle) 없이 로컬 처리됨

CREATE TABLE users (
    user_id INT
)
DISTSTYLE ALL;
-- ★핵심: DISTSTYLE ALL → 작은 테이블을 모든 노드에 복제.
--        users와의 join도 로컬에서 처리 가능 (broadcast와 동일 효과)
```

### (5)최신트렌드

- Delta Lake / Iceberg bucketing
    - 테이블 생성 시점에 `CLUSTER BY visit_id` (또는 `bucketBy`)를 설정해야 함 - 이건 엔지니어가 신경 써야 함
    - 한 번 설정되면, 이후 `visit_id` 기준 집계/조인 쿼리들은 작성자가 shuffle 회피 코드를 따로 안 짜도 엔진이 자동으로 회피함
    - 실무 포인트: 신규 테이블 설계할 때 "이 테이블이 어떤 컬럼으로 자주 집계/조인되는지"를 미리 파악해서 클러스터링 키로 박아두는 게 일이다. 나중에 바꾸려면 테이블 전체 재작성(`OPTIMIZE` + 재정렬) 비용이 든다
- Snowflake clustering key
    - 마찬가지로 테이블 생성/운영 중 클러스터링 키 지정은 엔지니어 작업
    - 다만 Snowflake는 자동 재클러스터링(auto-clustering)을 백그라운드로 돌려주므로, Spark의 `bucketBy`처럼 "파티션 수 고정" 같은 운영 제약은 덜함
- Kafka Streams co-partitioning
    - 양쪽 토픽의 파티션 수 + partition key를 일치시키는 건 토픽 설계 시점에 엔지니어가 결정해야 함
    - 한 번 맞춰두면, 이후 join 코드는 `groupByKey` + `join`만 쓰면 되고 repartition 토픽이 안 생김
    - 실무 포인트: 두 토픽을 나중에 다른 팀이 운영하다가 한쪽 파티션 수만 늘리면 co-partitioning이 깨지고, join 쿼리가 갑자기 repartition 토픽을 만들면서 비용/지연이 튄다 - 이게 신입이 자주 놓치는 부분

요약: 패턴#28(로컬 집계기)의 "shuffle 없이 집계"라는 효과 자체는 최신 도구에서도 동일하게 적용되지만, 그 효과를 얻으려면 "테이블/토픽을 어떤 키로 사전 분배할지"를 설계 단계에서 한 번 정해놓는 작업은 여전히 엔지니어 몫이다.



<br><br><br>



## 5.4 세션화(Sessionization)

- 패턴#29 증분 세션화기(Incremental Sessionizer)
- 패턴#30 상태 저장 세션화기(Stateful Sessionizer)

5.3(데이터 집계)
- grouping key별로 여러 레코드를 하나의 스칼라 값(평균, 카운트 등)으로 줄이는 작업
5.4(세션화)
- grouping key(여기서는 `visit_id`)별로 레코드를 묶는다는 점은 같지만, 
- 결과물이 스칼라가 아니라 "시작 시점 ~ 종료 시점 + 그 사이의 이벤트 목록"으로 구성된 세션
- "언제 세션이 끝났다고 판단할 것인가"라는 경계(boundary) 정의 문제 발생



## 패턴#29 증분 세션화기(Incremental Sessionizer)

### (1)문제상황

- `visits` 이벤트가 시간 단위 파티션으로 저장됨 (`hour=09`, `hour=10`, ...)
- 세션 정의: 첫 방문 시작, 2시간 동안 추가 방문 없으면 종료
- 세션 길이 몇 분~3시간 → 세션 1개가 최대 3개 시간 파티션에 걸침

데이터 예시:

|visit_id|user_id|event_time|page|
|---|---|---|---|
|v2001|u123|09:50|/home|
|v2001|u123|10:05|/products|
|v2001|u123|10:40|/cart|

- `hour=09` 파티션: v2001의 09:50만 있음
- `hour=10` 파티션: v2001의 10:05, 10:40만 있음
- `hour=10`만 처리하면 세션 시작점(09:50, /home)을 모름
- 정확한 세션을 만들려면 `hour=09`도 같이 봐야 함
- 매번 몇 시간 전 파티션까지 다시 읽어야 하는지 알 방법이 없어서, 분석팀이 매번 여러 연속 파티션을 재처리해야 했음

질문: 과거 파티션을 매번 다 읽지 않고, "이전 실행에서 안 끝난 세션"만 이어받을 수 없을까?

### (2)솔루션

주요컨셉: "입력/완료/대기"는 실제 테이블 3개고, 매 실행마다 이 3개 사이에서 데이터를 옮긴다.

- 입력 데이터 - S3의 시간별 raw `visits` 파일 (기존 데이터, 새로 안 만듦)
- 완료 테이블(`dedp.sessions`) - 종료된 세션만, 분석팀 조회용
- 대기 테이블(`dedp.pending_sessions`) - 비활성 기간(2시간) 안 지난 세션, 기술팀 전용 (패턴#26과 동일한 접근제어 방식)

매 실행 흐름:

- 1. 이번 시간 파티션의 raw `visits` 읽기
- 2. `pending_sessions`에서 같은 `visit_id`의 미완료 세션 조회
- 3. 둘을 합쳐서 세션 갱신 (시작시간/마지막방문시간/페이지목록)
- 4. 만료 판단 (`last_visit_time` + 2시간 경과 여부)
    
    - 만료 → `sessions`(완료)에 저장
    - 미만료 → `pending_sessions`(대기)에 다시 저장

핵심:

- `hour=09` 실행 → `pending_sessions`에 v2001 row 1개 저장: `{start: 09:50, last: 09:50, pages: [/home]}`
- `hour=10` 실행 → 그 row를 다시 읽어서 갱신 → 만료 안 됐으면 `pending_sessions`에 재저장
- 즉 "이전 실행 상태"는 메모리 캐시가 아니라 DB 테이블의 row로 전달됨

재실행(idempotency):

- 이번 실행 시점 이후로 만들어진 row를 먼저 삭제 후 재생성 (패턴#16과 동일한 사고방식)
- 재실행해도 중복 row 안 생기게 보장

### (3)결과

- 비활성 기간 길이의 트레이드오프
    - 길게: late data 포함 가능 → 정확도 ↑
    - 길게: `pending_sessions`에 더 오래 머무름 → 비용 ↑
- 부분 세션 노출 시 일관성 위험
    - 완료 전 미완성 세션을 먼저 노출할 경우 `is_completed: false` 플래그 필수
    - 예: 사기 탐지에서 1차 파티션은 "안전" → 2차 파티션에서 "위험"으로 바뀜. 플래그 없으면 컨슈머가 1차 결과를 최종으로 오판
- late data + forward dependency
    - `hour=09` 결과가 `hour=10`에 영향, `hour=10`이 `hour=11`에 영향 (순차 의존)
    - `hour=09`에 late data 들어와서 재처리하면 이후 모든 파티션도 재처리 필요 → 비용 급증
    - 대응: 전체 재처리(단순/비쌈) vs 영향받는 entity만 탐지 후 재처리(복잡/저비용) - 트레이드오프

### (4)예시

엔지니어 독백:

> 처음엔 매일 지난 3시간 파티션을 통째로 다시 읽어서 세션을 만들었는데, 데이터량 늘면서 느려졌어. `pending_sessions` 테이블 두고 "안 끝난 세션만 이어받는" 방식으로 바꾸니 매 실행 읽는 양이 확 줄었어. 다만 09시 파티션에 1시간 늦은 이벤트가 들어왔을 때, 그게 10시·11시 세션까지 영향 준다는 걸 처음엔 몰라서 한참 헤맸어.

DAG 흐름 (이번 실행 정리 → 입력 로드 → pending과 결합 → 완료/대기 분기 저장):

sql

```sql
-- 1) 재실행 대비: 이번 실행 이후 생성된 row 정리
DELETE FROM dedp.sessions WHERE execution_time_id >= '{{ ds }}';
DELETE FROM dedp.pending_sessions WHERE execution_time_id >= '{{ ds }}';

-- 2) 이번 시간 파티션 입력 로드
CREATE TEMPORARY TABLE visits_now AS
SELECT * FROM raw_visits WHERE hour = '{{ ds_nodash }}';

-- 3) 입력 + pending 결합 (신규/갱신/대기유지 케이스를 FULL OUTER JOIN으로 한번에)
CREATE TEMPORARY TABLE sessions_to_classify AS
SELECT
    COALESCE(p.session_id, n.session_id) AS session_id,
    LEAST(p.start_time, n.event_time) AS start_time,
    GREATEST(p.last_visit_time, n.event_time) AS last_visit_time,
    ARRAY_CAT(p.pages, n.pages) AS pages,
    CASE WHEN n.session_id IS NULL THEN p.expiration_batch_id
         ELSE '{{ macros.ds_add(ds, 2) }}' END AS expiration_batch_id
FROM visits_now n
FULL OUTER JOIN dedp.pending_sessions p ON p.session_id = n.visit_id;

-- 4) 만료 여부로 완료/대기 분기 저장
INSERT INTO dedp.sessions
    SELECT * FROM sessions_to_classify WHERE expiration_batch_id = '{{ ds }}';

INSERT INTO dedp.pending_sessions
    SELECT * FROM sessions_to_classify WHERE expiration_batch_id != '{{ ds }}';
```

### (5)최신트렌드

- dbt incremental + `MERGE` - pending/completed 분리를 dbt 모델로 표현, `MERGE INTO`(패턴#18)로 `pending_sessions` 갱신
- Delta Lake `MERGE INTO` - 신규/갱신/만료를 한 문장으로 처리, INSERT/DELETE 분리 관리 부담 감소
- Lakehouse 배치-스트림 통합 - 시간 단위 배치 세션화 대신 패턴#30(상태 저장 세션화기)처럼 스트리밍+state store로 전환하는 추세


