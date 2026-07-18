# Ch07.데이터보안 디자인 패턴

**챕터 개요**
- 데이터 엔지니어링은 파이프라인 구축에서 끝나지 않음 → 보안까지 책임져야 함
- GDPR(유럽), CCPA(캘리포니아) 같은 데이터 프라이버시 법규 등장 → 데이터 삭제 의무 발생
- 이 챕터에서 다루는 보안 영역 4가지:

1. 데이터 삭제 (Data Removal)
2. 접근 제어 (Access Control)
3. 데이터 보호 (Data Protection)
4. 연결 보안 (Connectivity)



**챕터 7 패턴 목록**

| 영역       | 패턴                                        | 한 줄 목적                |
| -------- | ----------------------------------------- | --------------------- |
| 1.데이터 삭제 | 패턴#39 Vertical Partitioner                | 데이터 레이아웃으로 삭제 비용 줄이기  |
|          | 패턴#40 In-Place Overwriter                 | 기존 데이터를 제자리에서 덮어써서 삭제 |
| 2.접근 제어  | 패턴#41 Fine-Grained Accessor for Tables    | 테이블 컬럼/행 단위 접근 제어     |
|          | 패턴#42 Fine-Grained Accessor for Resources | 클라우드 리소스 단위 접근 제어     |
| 3.데이터 보호 | 패턴#43 Encryptor                           | 저장/전송 데이터 암호화         |
|          | 패턴#44 Anonymizer                          | PII 완전 제거             |
|          | 패턴#45 Pseudo-Anonymizer                   | PII를 가짜 값으로 대체        |
| 4.연결 보안  | 패턴#46 Secrets Pointer                     | 코드 외부에 자격증명 보관        |
|          | 패턴#47 Secretless Connector                | 자격증명 없이 DB 연결         |

---

<엔지니어 독백>
> 신입 때 보안은 인프라팀 일이라고 생각했어. 
> 근데 GDPR 위반 과징금이 수백억 단위로 터지는 거 보고 나서 생각이 바뀌었지. 
> 데이터 엔지니어가 파이프라인 설계할 때부터 
> "이 데이터 나중에 삭제 요청 오면 어떻게 지우지?"를 같이 고민 안 하면, 
> 나중에 삭제 요청 왔을 때 테이블 전체 스캔해서 지우는 최악의 상황이 생겨. 
> 이 챕터 패턴들이 그 고민을 미리 해결해주는 거야.


## 패턴#39 Vertical Partitioner
### (1) 문제상황

**핵심 고통**: 하나의 테이블에 민감 데이터와 비민감 데이터가 섞여 있으면, 삭제 요청 시 전체 데이터를 스캔해야 해서 비용과 시간이 폭발한다.


<엔지니어 독백>
> "개인정보 삭제 요청" 파이프라인 설계 문서 처음 작성했을 때 
> 동료들한테 리뷰 받았는데, 
> "야 이거 immutable 컬럼들이 레코드마다 중복 저장되잖아, 
> 삭제할 때 전부 다 뒤져야 하는 거 알아?" 라는 피드백이 왔어. 
> 그때 처음 깨달은 거야. 데이터 레이아웃 자체가 삭제 비용을 결정한다는 걸.


**배경 — 먼저 알아야 할 것**
- GDPR / CCPA: 사용자가 "내 데이터 삭제해달라" 요청 시 의무적으로 삭제해야 하는 법규
- PII(Personally Identifiable Information): 개인 식별 가능 정보 — 이름, 생년월일, 개인 ID 등
- Immutable 컬럼: 레코드마다 값이 바뀌지 않는 컬럼 — 생년월일, 개인 ID 번호 등
- Mutable 컬럼: 레코드마다 값이 달라지는 컬럼 — 방문 시각, 방문 페이지 등


**실무 시나리오**
- 블로그 플랫폼 운영 중, `visits` 테이블에 사용자 방문 로그 적재
- 현재 테이블 구조:

|visit_id|event_time|page|user_id|birthday|personal_id|
|---|---|---|---|---|---|
|v001|2024-01-01 09:00|/home|u123|1990-05-01|ID-9900|
|v002|2024-01-01 09:30|/about|u123|1990-05-01|ID-9900|
|v003|2024-01-01 10:00|/blog|u123|1990-05-01|ID-9900|

- `birthday`, `personal_id`: Immutable 컬럼 → 사용자 u123의 모든 레코드에 **동일한 값이 반복**
- 사용자 u123이 방문할 때마다 같은 `birthday`, `personal_id`가 계속 쌓임
- 1년치 방문 로그 = 365개 레코드 × PII 컬럼 반복 저장



**구체적 실패 장면**
- 사용자 u123이 GDPR 삭제 요청 발송
- 삭제 파이프라인 실행:
	- [1]`visits` 테이블 전체 스캔 → u123 레코드 찾기
	- [2]1억 건 테이블에서 u123 레코드 365개 찾아서 삭제
	- [3]전체 테이블 스캔 비용 발생 → 시간 + 비용 폭발
- 더 큰 문제: `birthday`, `personal_id`가 365개 레코드에 **중복 저장**
	- 삭제해야 할 PII 데이터가 365곳에 흩어져 있음
	- 하나라도 누락되면 GDPR 위반


**개선 방향
- "PII 컬럼을 별도 테이블에 한 번만 저장하면 삭제가 훨씬 쉬워지지 않나?"
- 근데 하나의 테이블을 두 개로 쪼개면 읽을 때 JOIN이 필요해짐 → 읽기 성능 저하
- 쓸 때는 편해지지만 읽을 때는 복잡해지는 트레이드오프 → **Vertical Partitioner 패턴** 등장
- 이름 의미: 테이블을 행(Row) 단위가 아니라 **열(Column) 단위로 수직 분할**해서 저장

> 그냥 정규화하는거임







### (2) 솔루션

#### 주요 컨셉: 
- 데이터 처리 job이 하나의 레코드를 mutable 컬럼과 immutable 컬럼으로 분리해서 각각 다른 저장소에 쓴다.


#### **분리 기준 — 2가지 컬럼 그룹**
- Mutable 컬럼: 방문할 때마다 값이 달라지는 컬럼 → 방문 로그 테이블에 저장
- `visit_id`, `event_time`, `page`, `user_id`
- Immutable 컬럼: 사용자마다 고정된 값, PII 포함 → 별도 PII 테이블에 **한 번만** 저장
- `user_id`, `birthday`, `personal_id`


#### **분리 후 테이블 구조**
visits 테이블 (mutable):

|visit_id|event_time|page|user_id|
|---|---|---|---|
|v001|2024-01-01 09:00|/home|u123|
|v002|2024-01-01 09:30|/about|u123|
|v003|2024-01-01 10:00|/blog|u123|

visits_users 테이블 (immutable, PII):

|user_id|birthday|personal_id|
|---|---|---|
|u123|1990-05-01|ID-9900|

- `user_id`: 두 테이블을 나중에 합칠 때 쓰는 **결합 키(join key)**
- PII 컬럼이 365번 반복 → **단 1번** 저장으로 줄어듦



#### **삭제 요청 처리 흐름 비교**
분리 전:
```
visits 테이블 전체 스캔 (1억 건)
→ u123 레코드 365개 찾기
→ 각 레코드에서 birthday, personal_id 삭제
→ 365번 삭제 작업
```

분리 후:
```
visits_users 테이블에서 user_id = 'u123' 조회
→ 레코드 1건 삭제
→ 끝
```



#### **구현 방법 — SELECT로 컬럼 분리**
- 데이터 처리 job에서 `SELECT` 또는 컬럼 프로젝션으로 두 그룹을 각각 쿼리
- 각 쿼리 결과를 별도 저장소에 쓰는 방식
- 필요하면 각 그룹에 비즈니스 룰(중복 제거 등) 추가 적용 가능



#### **Horizontal Partitioning과 차이**
- Horizontal Partitioning: 행(Row) 전체를 기준으로 분할 → 날짜별 파티션 등
- Vertical Partitioning: 열(Column) 기준으로 행을 쪼개서 분할 → 이 패턴
- 두 방식은 함께 사용 가능 → mutable 테이블에 날짜 파티션 추가 적용 가능


원본 테이블:

|visit_id|event_time|page|user_id|birthday|
|---|---|---|---|---|
|v001|2024-01-01|/home|u123|1990-05-01|
|v002|2024-01-02|/about|u456|1985-03-15|
|v003|2024-01-02|/blog|u123|1990-05-01|



**Horizontal Partitioning — 행 전체를 날짜 기준으로 분리**
```
visits_20240101 테이블:
v001 | 2024-01-01 | /home | u123 | 1990-05-01

visits_20240102 테이블:
v002 | 2024-01-02 | /about | u456 | 1985-03-15
v003 | 2024-01-02 | /blog  | u123 | 1990-05-01
```
- 행 전체가 통째로 이동
- 컬럼 구조는 동일, 위치만 달라짐



**Vertical Partitioning — 행을 컬럼 기준으로 쪼개서 분리**
```
visits 테이블 (mutable):
v001 | 2024-01-01 | /home  | u123
v002 | 2024-01-02 | /about | u456
v003 | 2024-01-02 | /blog  | u123

visits_users 테이블 (immutable):
u123 | 1990-05-01
u456 | 1985-03-15
```
- 행이 쪼개져서 각각 다른 테이블에 저장
- `birthday` 컬럼이 분리됨, `user_id`로 나중에 합칠 수 있음


**Horizontal Partitioning — 분리 방식**
- 관계형 DB (PostgreSQL 등): 날짜별 **파티션 테이블** 생성
- `visits_20240101`, `visits_20240102` 테이블로 분리
- 파일 시스템 (S3, HDFS 등): 날짜별 **디렉토리(폴더)**로 분리
- `s3://bucket/visits/dt=2024-01-01/`, `s3://bucket/visits/dt=2024-01-02/`
- Spark, Parquet: `partitionBy('event_date')` → 날짜별 **폴더 + 파일** 분리


**Vertical Partitioning — 분리 방식**
- 관계형 DB: mutable/immutable **별도 테이블**로 분리
- `visits` 테이블, `visits_users` 테이블
- 파일 시스템: mutable/immutable **별도 디렉토리**로 분리
- `s3://bucket/visits/events/`, `s3://bucket/visits/users/`
- Delta Lake: mutable/immutable **별도 Delta 테이블**로 분리




### (3) 결과(Consequences)

**핵심**: 삭제 비용을 줄이는 대신 읽기 성능과 쿼리 복잡도를 희생한다.



**단점 1 — 읽기 성능 저하 (Query Performance)**
- 배경: 분리 전엔 `visits` 테이블 하나만 읽으면 전체 데이터 조회 가능
- 문제: 분리 후엔 `visits` + `visits_users` 두 테이블을 **JOIN**해야 전체 데이터 조회 가능
- JOIN = 네트워크 트래픽 발생 → 로컬 읽기보다 느림
- 테이블이 클수록 JOIN 비용 증가
- 대응: VIEW로 두 테이블을 미리 조인해서 단일 진입점 제공 → 소비자는 VIEW만 읽으면 됨



**단점 2 — 쿼리 복잡도 증가 (Querying Complexity)**
- 배경: 분리 전엔 소비자가 테이블 구조 하나만 알면 됐음
- 문제: 분리 후엔 소비자가 "어떤 컬럼이 어느 테이블에 있는지" 파악해야 함
- `birthday`가 `visits`에 없다는 걸 모르면 쿼리 실패
- 팀 규모가 클수록 혼란 가중
- 대응:
- VIEW로 단일 진입점 노출
- 데이터 카탈로그로 문서화
- 데이터 리니지(lineage) 명확히 표현



**단점 3 — 폴리글랏 환경 복잡도 (Complexity in a Polyglot World)**
- 배경: 실무에서 같은 데이터를 **여러 종류의 저장소**에 동시에 저장하는 경우가 많음
- 검색팀: 검색 최적화 DB인 OpenSearch에 저장
- 저지연 API 서버: 빠른 조회를 위해 Redis에 저장
- 분석팀: 대용량 분석을 위해 Delta Lake에 저장
- 같은 `visits` 데이터가 3군데에 동시에 존재하는 상황
- 문제: Vertical Partitioner를 적용하면 **저장소마다 분리 구현을 따로 해야 함**

분리 전 삭제 파이프라인:
```
삭제 요청 → visits 데이터 삭제
           ↓
     [OpenSearch] 삭제
     [Redis]      삭제
     [Delta Lake] 삭제
```

분리 후 삭제 파이프라인:
```
삭제 요청 → immutable 데이터 삭제
           ↓
     [OpenSearch] visits_users 삭제  ← OpenSearch용 분리 구현 필요
     [Redis]      visits_users 삭제  ← Redis용 분리 구현 필요
     [Delta Lake] visits_users 삭제  ← Delta Lake용 분리 구현 필요
```
- 저장소가 3개면 분리 job도 3개, 삭제 파이프라인도 3개
- 저장소 추가될 때마다 분리 구현 추가 필요 → 유지보수 복잡도 증가
- 대응: 분리 job이 mutable/immutable 결과를 Kafka 토픽에 발행 → 각 저장소 전용 consumer가 구독해서 각자 저장

```
[분리 job]
    ↓                    ↓
[Kafka: mutable]   [Kafka: immutable]
    ↓                    ↓
[OpenSearch consumer]  [Delta Lake consumer]
[Redis consumer]       [Redis consumer]
```
- 분리 로직은 job 하나에만 존재 → 저장소 추가돼도 consumer만 추가하면 됨



**단점 4 — 원본 데이터(Raw Data) 미적용**
- Vertical Partitioner는 **첫 번째 데이터 변환 단계부터** 적용 가능
- Raw 데이터(분리 전 원본)는 이 패턴으로 처리 불가 → 별도 솔루션 필요
- 단기 보존 기간 설정으로 원본 자동 삭제 처리
- 단, 보존 기간이 짧으면 백필 시 원본 데이터 없어서 재처리 불가 → 트레이드오프



<엔지니어 독백>
> VIEW로 단일 진입점 만들어두는 거 진짜 중요해. 
> 분리해놓고 소비자한테 "테이블 두 개 JOIN해서 써"라고 하면 
> 팀마다 JOIN 로직이 달라지고, 나중에 테이블 구조 바꿀 때 모든 소비자 코드 다 찾아서 고쳐야 해. VIEW 하나 만들어두면 구조 바뀌어도 VIEW만 수정하면 끝이야. 
> 그리고 Raw 데이터 보존 기간 설정할 때 법무팀이랑 반드시 협의해. 
> GDPR 삭제 요청 처리 기한이 30일인데 Raw 데이터 보존을 7일로 잡으면 문제없지만, 
> 백필 범위가 7일 이내로 제한되는 거니까 비즈니스 요구사항도 같이 봐야 해.



### (4)예시

**핵심**: Kafka producer가 레코드를 발행하면, Spark job이 mutable/immutable 컬럼을 분리해서 각각 Delta Lake 테이블에 쓴다.

<엔지니어 독백>
> Vertical Partitioner 실제로 구현할 때 핵심은 분리 job을 얼마나 단순하게 유지하냐야. SELECT로 컬럼 골라서 두 군데 쓰는 게 전부인데, 여기다 비즈니스 로직 덕지덕지 붙이면 나중에 유지보수가 힘들어져. 분리 job은 최대한 단순하게, 비즈니스 로직은 다운스트림에서 처리하는 게 실무 원칙이야.


#### **예시 1 — Spark로 mutable/immutable 컬럼 분리 후 Delta Lake에 쓰기**
입력 데이터를 읽어서 두 개의 Delta Lake 테이블로 분리해서 쓰는 job.
```python
input_data = spark.read.parquet('/data/visits/')

mutable_attributes = input_data.select(
    'visit_id', 'event_time', 'page', 'user_id'
)

immutable_attributes = input_data.select(
    'user_id', 'birthday', 'personal_id'
).dropDuplicates(['user_id'])

mutable_attributes.write.format('delta') \
    .mode('append') \
    .save('/delta/visits/')

immutable_attributes.write.format('delta') \
    .mode('ignore') \
    .save('/delta/visits_users/')
```
- `select()`: 컬럼 그룹별로 분리하는 핵심 연산
- **핵심 라인**: `dropDuplicates(['user_id'])`
- immutable 컬럼은 user_id당 한 번만 저장해야 함
- 같은 user_id가 여러 번 들어와도 최초 1건만 저장
- `mode('ignore')`: 이미 존재하는 user_id 레코드는 덮어쓰지 않음 → 중복 방지
- `mode('append')`: mutable은 방문할 때마다 계속 쌓임





#### **예시 2 — Kafka + Spark Structured Streaming으로 실시간 분리**
Kafka topic에서 실시간으로 들어오는 방문 로그를 스트리밍으로 분리해서 쓰는 job.
```python
visits_stream = spark.readStream \
    .format('kafka') \
    .option('kafka.bootstrap.servers', 'localhost:9092') \
    .option('subscribe', 'visits') \
    .load()

parsed = visits_stream.select(
    from_json(col('value').cast('string'), schema).alias('data')
).select('data.*')

mutable_stream = parsed.select(
    'visit_id', 'event_time', 'page', 'user_id'
)

immutable_stream = parsed.select(
    'user_id', 'birthday', 'personal_id'
).dropDuplicates(['user_id'])

mutable_stream.writeStream \
    .format('delta') \
    .option('checkpointLocation', '/checkpoint/visits/') \
    .start('/delta/visits/')

immutable_stream.writeStream \
    .format('delta') \
    .option('checkpointLocation', '/checkpoint/visits_users/') \
    .outputMode('update') \
    .start('/delta/visits_users/')
```
- Kafka topic `visits`: 방문 로그가 실시간으로 발행되는 스트림 소스
- `from_json()`: Kafka에서 문자열로 들어오는 값을 구조체로 파싱
- **핵심 라인**: `dropDuplicates(['user_id'])`
- 스트리밍에서도 동일하게 user_id 기준 중복 제거
- `outputMode('update')`: immutable 테이블은 기존 레코드 업데이트 모드로 씀
- checkpoint: 스트리밍 exactly-once 보장을 위한 체크포인트 경로 (내가 정하는 경로)

#### **분리 후 삭제 요청 처리**
user_id = 'u123' 삭제 요청 시:
```python
from delta.tables import DeltaTable

dt = DeltaTable.forPath(spark, '/delta/visits_users/')
dt.delete("user_id = 'u123'")
```
- `visits_users` 테이블에서 1건만 삭제하면 끝
- `visits` 테이블은 건드릴 필요 없음 → 삭제 비용 최소화

<엔지니어 독백>
> 스트리밍에서 `dropDuplicates(['user_id'])` 쓸 때 주의할 게 있어. 
> Spark Structured Streaming의 `dropDuplicates`는
> 워터마크 없이 쓰면 StateStore에 모든 user_id를 무한정 쌓아두거든. 
> 유저가 수백만 명이면 메모리 폭발해. 워터마크 설정해서 오래된 상태는 정리되도록 해야 해. 
> 아니면 아예 immutable 쓰기를 별도 배치 job으로 분리해서 처리하는 게 더 안전할 때도 있어.


<질문>
> ??독백 무슨말임?

**핵심 정리**
- `dropDuplicates` 단독: StateStore 무한 증가 → 메모리 폭발
- `withWatermark` + `dropDuplicates` 조합: StateStore 자동 정리 → 안전


**1단계 — dropDuplicates가 왜 StateStore를 쓰나**
- 스트리밍은 데이터가 **실시간으로 계속** 들어옴
- `dropDuplicates(['user_id'])`: 
	- 중복제거기능
	- "이미 본 user_id면 버려라"
- 근데 "이미 봤는지" 판단하려면 **지금까지 본 user_id 목록을 어딘가에 저장**해야 함
- 그 저장 공간이 **StateStore** (Spark가 내부적으로 관리하는 상태 저장소)


**2단계 — 워터마크 없으면 뭐가 문제냐**
- 워터마크 없으면 StateStore에 user_id를 **영원히** 쌓음
- 스트리밍 job이 1년 돌면 → 1년치 user_id 전부 메모리에 유지
- 유저 100만 명이면 100만 개 user_id가 메모리에 계속 쌓임 → 메모리 폭발
```
워터마크 없을 때 StateStore:

T+1일:  [u001, u002, u003 ...]          → 메모리 1GB
T+30일: [u001, u002, ... u100만 ...]    → 메모리 10GB
T+1년:  [u001, u002, ... 전체 유저 ...] → 메모리 폭발
```



**3단계 — 워터마크가 뭐냐**
- 워터마크: "이 시각 이전 데이터는 더 이상 안 온다고 간주하는 기준선"
- StateStore에서 기준선보다 오래된 user_id는 **자동으로 삭제**
- 메모리가 무한정 쌓이지 않음
```
워터마크 1일 설정 시:

현재 시각 = 1월 10일
워터마크 기준선 = 1월 9일

→ 1월 9일 이전 user_id는 StateStore에서 삭제
→ 메모리가 항상 "최근 1일치"만 유지
```


**워터마크 없이 dropDuplicates만 쓰면**
```
# 이렇게만 쓰면 문제
immutable_stream.dropDuplicates(['user_id'])
```
- Spark가 "지금까지 본 user_id 전부" 기억해야 함
- 기억을 지우는 기준이 없으니 → StateStore에 영원히 쌓임
- 유저 100만 명 → 메모리 폭발


**워터마크 + dropDuplicates 같이 쓰면**
```
# 이렇게 써야 함
immutable_stream \
    .withWatermark('event_time', '1 day') \  # "1일 지난 데이터는 잊어도 돼"
    .dropDuplicates(['user_id'])              # 그 기간 안에서만 중복 제거
```
- `withWatermark('event_time', '1 day')`: 1일 지난 user_id는 StateStore에서 삭제
- Spark가 "최근 1일치 user_id"만 기억 → 메모리 관리됨




**실무 선택지 2가지**
**선택지 1 — 워터마크 + dropDuplicates 조합**
- `withWatermark` + `dropDuplicates` 같이 설정
- StateStore가 워터마크 기간만큼만 user_id 기억 → 메모리 관리됨
- 단점: 워터마크 기간이 지난 후 같은 user_id가 다시 들어오면 → "처음 보는 user_id"로 판단 → 중복 저장

**선택지 2 — immutable 쓰기를 배치 job으로 분리**
- 스트리밍 job: mutable 컬럼만 처리
- immutable 컬럼(birthday, personal_id): 별도 배치 job이 주기적으로 처리
- StateStore 아예 안 씀 → 메모리 문제 없음, 중복 걱정 없음
- 단점: 실시간이 아니라 배치 주기만큼 지연 발생


### (5) 최신트렌드

**핵심**: 요즘은 데이터 레이아웃 설계 단계부터 삭제 비용을 고려하는 방향으로 가고 있고, 이를 지원하는 포맷과 플랫폼이 발전하고 있다.



**1. Delta Lake — 삭제 대상 파일 최소화**
- 정체: Parquet 파일 위에 트랜잭션 로그(_delta_log)를 추가한 레이크하우스 스토리지 포맷
- 이전 한계 (Parquet 단독):
- 어느 파일에 어떤 user_id가 있는지 추적하는 메타데이터 없음
- user_id 1건 삭제 → 전체 파일 1000개 스캔 → 1000개 전부 재작성
- 왜 요즘 쓰나:
- 트랜잭션 로그가 "u123은 part-003.parquet에 있다"고 추적
- `DELETE FROM visits_users WHERE user_id = 'u123'` 실행 시 → part-003.parquet **1개만** 재작성
- 파일 재생성 자체는 동일하지만 **재생성 대상 파일 수가 최소화**됨
- 체감 이점: 1000개 파일 중 1~2개만 재작성 → 삭제 비용, 시간 대폭 감소



**2. Apache Iceberg — Row-level Delete**
- 정체: 대규모 분석 테이블을 위한 오픈 테이블 포맷
- 이전 한계: Delta Lake처럼 행 단위 삭제가 특정 플랫폼(Databricks)에 종속됨
- 왜 요즘 쓰나: Iceberg는 Spark, Flink, Trino 등 다양한 엔진에서 행 단위 삭제 지원 → 멀티 엔진 환경에서 Vertical Partitioner 구현 가능
- 체감 이점: 플랫폼 종속 없이 GDPR 삭제 파이프라인 구축 가능



**3. Apache Kafka — 토픽 단위 데이터 분리**
- 정체: 분산 스트리밍 플랫폼
- 이전 한계: 하나의 토픽에 mutable/immutable 데이터가 섞여서 들어옴 → 소비 단계에서 분리해야 함
- 왜 요즘 쓰나: 토픽 설계 단계부터 mutable/immutable을 별도 토픽으로 분리 → consumer가 필요한 토픽만 구독
- `visits-mutable` 토픽 → mutable consumer
- `visits-immutable` 토픽 → immutable consumer
- 체감 이점: 분리 job 없이 토픽 설계만으로 Vertical Partitioner 구현 가능



**4. BigQuery / Snowflake — 컬럼 단위 마스킹**
- 정체: 클라우드 기반 데이터 웨어하우스 — SQL로 대용량 데이터 분석하는 플랫폼
- 이전 한계:
	- Vertical Partitioner로 PII 컬럼을 별도 테이블로 분리해도, 그 테이블에 접근 권한이 있으면 PII 전체가 노출됨
	- "삭제는 쉬워졌는데, 누가 볼 수 있는지 제어가 안 됨"
- 왜 요즘 쓰나:
	- BigQuery/Snowflake는 컬럼 단위 **Dynamic Data Masking** 지원
	- 예) 분석팀이 `visits_users` 테이블 조회 시 → `birthday` 컬럼은 `****`로 마스킹, `personal_id`는 `NULL`로 반환
	- Vertical Partitioner로 PII 테이블 분리 + 컬럼 마스킹 정책 적용 → **분리(삭제 비용 절감) + 열람 제한(보안)** 동시 해결
- 체감 이점:
- 삭제는 Vertical Partitioner로, 열람 제한은 마스킹으로 각각 역할 분리
- PII 테이블을 아예 숨기지 않아도 됨 → 접근 권한 관리 단순화



<엔지니어 독백>
> 요즘 GDPR 대응 설계할 때 Delta Lake나 Iceberg 없으면 진짜 고생해. 예전에 Parquet만 쓰던 시절엔 user_id 하나 지우려고 파일 전체 재작성하는 배치 job 따로 만들어야 했거든. 
> 
> 근데 Delta Lake 도입하고 나서 `DELETE FROM visits_users WHERE user_id = 'u123'` 한 줄로 끝나. 설계할 때 immutable 테이블을 Delta Lake로 잡는 게 거의 표준이 됐어. 
> 그리고 Kafka 토픽 설계할 때부터 mutable/immutable 분리해두면 나중에 분리 job 따로 안 만들어도 돼서 훨씬 깔끔해.

<br><br><br><br>


 
# 패턴#40 In-Place Overwriter - <1.데이터삭제 관리 >
## (1) 문제상황

핵심 고통: 레거시 시스템엔 개인정보 삭제 전략이 없는데, 규제는 삭제를 요구한다.

- 이 blog analytics 플랫폼, 5년 전부터 돌아가던 시스템이다.
- 데이터는 전부 날짜 기준 horizontal partition(예: date=2024-01-01/)으로 쌓여있고, 개인정보 삭제 같은 건 애초에 설계에 없었다.
- 이번에 정부 개인정보보호 규정이 새로 생겨서, 유저가 삭제 요청하면 시스템에서도 그 사람 데이터를 실제로 지워야 하는 의무가 생겼다.
- 이상적으로는 Vertical Partitioner처럼 유저 단위로 데이터를 애초에 쪼개서 저장했으면 삭제가 쉬웠을 것이다. 근데 이미 테라바이트 단위로 쌓인 레거시 데이터를 지금 와서 마이그레이션할 시간도, 인력도 없다.
- 결국 기존 저장 구조는 그대로 두고, 데이터를 있는 자리에서(in-place) 덮어써서 지우는 수밖에 없다.

핵심 정리:
- 레거시 시스템: 시간 기준 horizontal partition에 수 TB 데이터 저장, 개인정보 관리 전략 부재
- 신규 규제: 사용자 요청 시 개인정보 삭제(data removal) 의무 발생
- 제약: 리팩토링/마이그레이션할 시간과 리소스 없음 → Vertical Partitioner 적용 불가
- 남은 선택지: 기존 구조를 유지한 채 데이터를 직접 덮어써서 삭제

## (2) 솔루션

주요컨셉: 삭제 대상을 걸러낸 나머지 데이터로, 기존 위치를 안전하게 덮어쓴다.

구현은 스토리지 기술에 완전히 의존한다.

**1. 네이티브 in-place 삭제 지원 스토리지 (Delta Lake, Iceberg)**
- `DELETE ... WHERE user_id = ...` 한 줄로 처리 가능
- 단, delete 해도 time travel 때문에 실제 파일은 안 지워짐. VACUUM 같은 별도 클리닝 작업으로 데이터 블록을 reclaim해야 진짜 삭제됨. 안 그러면 이전 버전 읽기로 개인정보가 복구됨

**2. Deletion Vector 방식 (delete 관리 두 가지 방식)**
- 삭제된 row를 별도의 작은 side file에 마킹만 해두는 방식: write는 가볍지만, read할 때 consumer가 매번 "삭제됨" 여부를 체크하고 걸러야 함
- 삭제 대상 제외한 나머지를 파일로 다시 쓰는 방식: write는 무겁지만 read는 그냥 읽으면 됨

**3. 네이티브 미지원 스토리지 (JSON, CSV)**
- 전체 데이터셋을 읽어서 삭제 대상 유저를 필터링(filter out)하는 job을 직접 돌려야 함
- 원본 위치를 곧바로 덮어쓰면 안 됨: Spark는 분산 + 트랜잭션이 없는(transactionless) 처리 레이어라, job이 중간에 실패/재시도되면 원본 자리가 반쯤 비어있거나 깨진 상태로 남을 수 있음
- 올바른 방식: staging 영역에 먼저 필터링 결과를 전부 쓰고, 완료 확인 후 별도의 승격(promotion) job이 기존 public 데이터셋을 통째로 덮어씀
  - staging job 실패해도 원본은 안전 (데이터 로스 없음)
  - promotion job 실패해도 staging에 완성된 결과가 있으니 재시도만 하면 됨

**4. Kafka (row 개념이 없는 스토리지)**
- tombstone message: 삭제 대상 레코드의 key + null value로 구성된 삭제 마커 메시지
- topic의 cleanup.policy=compact 설정 시, 백그라운드 compaction 프로세스가 tombstone을 인식해 실제 브로커 디스크 로그 파일에서 지움
- 한계: key 하나당 레코드가 유일한(vertically partitioned) 구조에서만 통함. 하나의 key에 여러 이벤트가 붙는 경우(예: 방문 이벤트)엔 최신 값만 남기는 compaction 특성상 부적합

### (질문)
> Q.tombstone 메시지가 뭐야
> A."이 key를 삭제하라"는 목적 하나로만 쓰이는 특수 메시지
> 

Tombstone message
- 정체: key는 있고 value는 null인 특수한 형태의 Kafka 메시지
- 왜 나왔나: Kafka topic은 원래 append-only 로그라서 "삭제"라는 개념 자체가 없었음. 근데 compaction(같은 key의 옛날 값을 지우고 최신 값만 남기는 정리 방식)을 쓰는 topic에서는 "이 key를 아예 지워라"는 신호를 보낼 방법이 필요했음
- 그래서 만든 게 tombstone: 해당 key로 value=null인 메시지를 보내면, 이게 곧 "이 key의 데이터를 삭제하라"는 명령 역할을 함

**동작 흐름**
1. producer가 `(user_id, NULL)` 형태로 tombstone을 보냄
2. 이것도 일단 브로커 디스크에 하나의 레코드로 저장됨
3. 백그라운드 compaction thread가 돌면서, 같은 key의 옛날 레코드들 + tombstone 자체를 실제로 디스크에서 지움 (`delete.retention.ms` 시간 지난 후)

**한계**
- key 하나당 레코드가 유일한 구조(vertically partitioned)에서만 통함
- 방문 이벤트처럼 같은 key로 여러 row가 쌓이는 topic엔 부적합 (compaction이 최신 값 하나만 남기는 방식이라서)


## (3) 결과

**1. 비용(Cost) — 1건 지우려고 전체를 다 읽는다**
- Vertical Partitioner였으면 유저 1명 삭제 = 그 유저 파일 1개만 지우면 끝
- In-Place Overwriter는 파티션 전체를 읽고 다시 써야 함. 2,000 row 파티션에 1명만 삭제 대상이어도 2,000 row 전부 read/write 발생
- 완화책: 삭제 요청을 건별 즉시 처리하지 말고 모아뒀다가(batch) 주기적으로 처리 → job 실행 횟수 자체를 줄임

**2. 롤백(Rollback) 불가능**
- Delta Lake/Iceberg는 원래 time travel(이전 버전 조회/복구) 기능이 있음
- 개인정보 삭제 목적으로 VACUUM까지 돌려서 물리 파일을 reclaim*하면, 그 시점 이전 버전으로 되돌아갈 방법 자체가 사라짐 (의도된 결과)
- VACUUM 안 돌리면 삭제한 척만 하고 실제론 이전 버전에서 데이터 복구 가능 → 규제 위반. 반드시 VACUUM까지 실행해야 진짜 삭제
*Reclaim: 회수하다. 논리적으로만 삭제 표시된 물리 파일을, 디스크에서 실제로 지워서 그 저장 공간을 되돌려 받는(회수하는) 작업 (VACUUM치는거)

#### delete() 실행 시 실제로 일어나는 일

**Before**

|버전|실제로 가리키는 파일|
|---|---|
|v10 (delete 이전)|file_A.parquet (u001, u002, u003 포함)|
|v11 (delete 이후)|file_B.parquet (u001 뺀 u002, u003만 재작성됨)|

- delete() 실행하면 Delta Lake는 file_A를 고치는 게 아니라, u001 뺀 새 파일 file_B를 만들고 v11 로그에 "이제부터 file_B를 봐라"고 기록
- **file_A는 지워지지 않고 디스크에 그대로 남아있음** (v10으로 time travel하면 여전히 file_A를 읽어서 u001을 복구 가능)

#### VACUUM이 지우는 대상

- VACUUM이 지우는 건 **v10이라는 버전 기록 자체**가 아니라, **v10만 참조하고 다른 어떤 최신 버전도 참조하지 않는 file_A라는 물리 파일**
- 조건: retention 기간(기본 7일)이 지나서, 그 파일을 가리키는 버전이 더 이상 "복구 가능 범위" 밖에 있어야 지워짐

**After VACUUM**

|버전|상태|
|---|---|
|v10|로그 기록 자체는 남아있을 수 있음, 근데 v10이 가리키던 file_A가 물리적으로 사라짐 → v10으로 time travel해도 읽을 파일이 없어 실패|
|v11|file_B 그대로 존재, 정상 조회 가능|
**v10으로 time travel이 안 되면, u001뿐 아니라 그 시점의 u002 상태도 같이 확인 불가능해진다.** file_A라는 파일 하나에 u001, u002, u003이 같이 묶여 저장돼 있었기 때문에, 파일이 통째로 지워지면 그 파일에 같이 있던 다른 유저들의 "그 시점 스냅샷"도 함께 사라짐.

## (4) 예시

**Delta Lake — 네이티브 삭제 지원**

~~~python
user_id_to_delete = '140665101097856_0316986e-9e7c-448f-9aac\
					-5727dde96537'
users_table = DeltaTable.forPath(spark_session,\
								 get_delta_users_table_dir())
users_table.delete(f'user_id = "{user_id_to_delete}"')
~~~

- 삭제할 유저 ID를 조건으로 delete() 호출 → 해당 row만 삭제
- ★ 핵심: `users_table.delete(...)` — Delta Lake가 트랜잭션 로그(transaction log) 기반으로 삭제를 원자적으로 기록. 별도 staging 없이 이 한 줄로 안전하게 처리됨
- 이 시점엔 논리적으로만 삭제됨. 이후 VACUUM을 별도로 돌려야 물리 파일이 실제로 지워짐


**Kafka — tombstone message**
- Kafka 브로커한테 "u001이라는 유저의 데이터 지워달라"는 메시지 한 개를 콘솔에서 직접 입력해서 보내는 작업

~~~bash
docker exec -ti ... kafka-console-producer.sh --bootstrap-server .... \
--topic ... \
--property parse.key=true \
--property key.separator=, \
--property null.marker=NULL
140665101097856_0316986e-9e7c-448f-9aac-5727dde96537, NULL
~~~

|옵션|뜻|
|---|---|
|`--bootstrap-server`|접속할 Kafka 브로커 주소 (실무에선 실제 broker 호스트:포트 입력)|
|`--topic`|메시지를 보낼 대상 topic 이름|
|`--property parse.key=true`|입력한 줄을 "key,value" 형태로 나눠서 해석하라는 설정. 이게 없으면 전체를 그냥 value로만 취급함|
|`--property key.separator=,`|key와 value를 구분하는 문자를 콤마(,)로 지정|
|`--property null.marker=NULL`|입력한 텍스트 중 "NULL"이라는 문자열을 실제 null 값으로 취급하라는 설정|

- console producer로 key=삭제할 유저ID, value=NULL인 메시지(tombstone)를 topic에 전송
- ★ 핵심: 마지막 줄 `유저ID,NULL` — key는 있고 value가 null인 조합 자체가 "이 key는 지워라"는 신호
- cleanup.policy=compact 설정에 의해 백그라운드 compaction 프로세스가 해당 key의 모든 메시지+tombstone을 제거



**JSON (flat file) — 네이티브 삭제 미지원**
- JSON 파일 전체를 읽어서 특정 유저 ID를 제외한 나머지만 골라 staging 경로에 새 파일로 저장하는 PySpark 코드
~~~python
input_raw_data = spark_session.read.text(get_input_table_dir())
df_w_user_column = input_raw_data.withColumn(
    'user', F.from_json('value', 'user_id STRING')
)
user_id = '139621130423168_029fba78-15dc-4944-9f65-00636566f75b'
to_save = df_w_user_column.filter(f'user.user_id != "{user_id}"').select('value')
to_save.write.mode('overwrite')\
			 .format('text')\
			 .save(get_staging_table_dir())
~~~

**실행 순서 (위에서 아래로):**

1. `read.text(...)` — 원본 JSON 파일을 한 줄씩 그냥 텍스트로 읽음. 각 row는 `value`라는 컬럼 하나에 통째로 문자열로 들어감
2. `withColumn('user', F.from_json(...))` — `value`의 문자열에서 `user_id` 필드만 뽑아 `user`라는 새 컬럼으로 추가
3. `filter(f'user.user_id != "{user_id}"')` — 삭제 대상 유저 ID와 일치하는 row를 걸러냄 ★ 핵심
4. `select('value')` — 파싱용으로 썼던 `user` 컬럼은 버리고, 원본 raw 텍스트인 `value` 컬럼만 남김
5. `write.mode('overwrite').format('text').save(...)` — 남은 row들을 staging 경로에 새 파일로 저장

**★ 핵심: `filter(f'user.user_id != "{user_id}"')`**

- 이 한 줄이 실질적인 삭제 동작. 나머지 코드는 이 필터링을 가능하게 하기 위한 준비(파싱)이거나 결과 저장을 위한 후처리

**세부 포인트:**

- `select('value')`인 이유: 2번에서 추가한 `user` 컬럼은 결과 파일에 불필요. 원본 raw 텍스트만 그대로 다시 저장해야 원본과 동일한 포맷 유지
- `user_id`만 파싱하는 이유: 필터링에 필요한 값만 뽑아서 불필요한 파싱 연산을 줄임
- `save(get_staging_table_dir())`인 이유: 원본이 아닌 staging에 씀. job 실패/재시도돼도 원본은 안 건드리기 위함. 이후 별도 promotion job이 완성된 결과를 원본 위치로 옮김

## (5) 최신트렌드

목적: 요즘 뭘 쓰고 왜 낫다고 체감하는지

**Delta Lake / Databricks**
- 오픈 테이블 포맷 + Databricks 관리형 삭제 API
- 예전 한계: raw Parquet만 쓰면 row 하나 지우려고 파일 전체를 rewrite해야 했음
- 요즘: DELETE FROM table WHERE ... 한 줄로 처리되고, Deletion Vector 기능이 기본 활성화되면서 파일 전체를 다시 쓰지 않고 삭제 마킹만 함 → 삭제의 write 비용이 훨씬 가벼워짐

**Unity Catalog (Databricks)**
- 개인정보 삭제 요청을 자동화하는 거버넌스 레이어
- 예전 한계: 어느 테이블에 특정 유저 데이터가 있는지 사람이 일일이 추적해야 했음
- 요즘: lineage 추적 기반으로 유저 데이터가 흩어진 테이블들을 자동으로 찾아주고, 삭제 요청 한 번으로 관련 테이블 전체에 전파 가능

**Kafka Tiered Storage / Confluent**
- 오래된 로그 세그먼트를 브로커 로컬 디스크 대신 S3 같은 원격 스토리지로 옮기는 기능
- 예전 한계: compaction이 브로커 로컬 디스크 안에서만 동작해서, 리텐션이 긴 topic은 디스크 용량 압박이 심했음
- 요즘: 오래된 세그먼트는 저렴한 오브젝트 스토리지로 내려가고, compaction/tombstone 삭제는 동일 룰로 동작 → 디스크 걱정 없이 리텐션을 길게 가져가면서도 삭제 규정 준수 가능







<br><br><br><br>
# 패턴#41 Fine-Grained Accessor for Tables - <2.접근제어>

Fine-Grained : "결이 고운, 입자가 미세한"이라는 원래 뜻에서 나온 표현으로, IT/보안 쪽에서는 "아주 세밀한 단위로 나눠서 통제한다"는 의미로 씀

## (1) 문제상황

핵심 고통: 테이블 접근 권한만으론 컬럼/row 단위 세밀한 통제가 안 된다.

- 우리 팀, 예전엔 HDFS/Hive 위에서 워크로드 돌렸는데 이번에 클라우드 데이터 웨어하우스로 마이그레이션했다.
- 보안 요구사항 1단계는 쉬웠다. 새 웨어하우스가 user/group 만들어서 테이블 단위 접근 권한 주는 건 기본 지원하니까.
- 근데 Stakeholder가 하나 더 요구한다. "이 테이블에 접근 권한이 있는 사람이라도, 모든 컬럼/모든 row를 다 보면 안 된다"는 것.
	- Stakeholder : 이 시스템/데이터의 결과에 이해관계가 걸려있는 사람 또는 조직 — 데이터 엔지니어 본인 말고, 요구사항을 제시하거나 그 결과에 영향을 받는 쪽 ex)보안팀(컴플라이언스 요구), 법무팀(규제 대응)
- 예를 들어 `users` 테이블은 마케팅팀도 봐야 하는데, `ssn`이나 `ip` 같은 민감 컬럼은 보안팀만 봐야 하고, row 기준으로는 각자 자기 지역 데이터만 봐야 하는 식이다.
- 테이블 단위 권한 부여로는 이 요구사항을 못 맞춘다. 테이블보다 더 세밀한(fine-grained) 단위 — 컬럼, row 레벨로 접근을 통제하는 메커니즘이 따로 필요하다.


핵심 정리:
- 배경: 클라우드 DW로 마이그레이션 후 기본 user/group 기반 테이블 접근 제어는 이미 구현됨
- 추가 요구사항: 같은 테이블 안에서도 컬럼별, row별로 접근 권한을 다르게 줘야 함
- 한계: 테이블 단위(coarse-grained) 권한 부여로는 컬럼/row 레벨 통제가 불가능
- 필요한 것: table보다 더 낮은 레벨(column, row)에서 동작하는 인가(authorization) 메커니즘

## (2) 솔루션

주요컨셉: 테이블 권한 위에, 컬럼 단위·row 단위로 한 겹 더 세밀한 접근 통제를 얹는다.

**1. GRANT 문으로 컬럼 지정**
- 대상: Amazon Redshift, PostgreSQL
- `GRANT SELECT(col_A, col_B) ON my_table TO some_user;` 형태로, 특정 컬럼만 SELECT 권한을 줌
- DB 표준 문법에 가까워 구현이 제일 단순

**2. 데이터 카탈로그의 정책 태그(policy tag) 기반**
- 대상: GCP BigQuery
- Data Catalog에서 정책 태그를 만들고, 보호할 컬럼에 태그를 붙임
- 사용자에게는 그 태그에 대한 "Fine-Grained Reader" role을 부여
- 여러 테이블에 흩어진 동일 성격 컬럼(예: 모든 테이블의 ip 컬럼)을 태그 단위로 한 번에 관리 가능

**3. 데이터 마스킹(masking) 함수**
- 대상: Databricks(Unity Catalog), Snowflake
- 컬럼 자체는 보이되, 권한 없는 사용자에게는 값이 마스킹 처리됨
- GRANT 방식과 차이: 컬럼 존재 자체는 항상 보이고, 값만 숨김

Row 레벨(Row-level) 접근 통제:
- 쿼리 실행 시 동적으로 WHERE 조건을 자동으로 끼워 넣는 정책 객체를 정의
- 이름은 벤더마다 다름: Databricks는 ROW FILTER, Amazon Redshift는 Row-Level Security, BigQuery/Snowflake는 row access policy
- 동작 원리: 보호 대상 테이블에 SELECT 쿼리가 들어올 때마다, 정책이 자동으로 조건을 추가해서 실행 (예: WHERE login = current_user)




## (3) 결과

**1. Row-level 정책이 쓸 수 있는 속성이 제한적**
- row-level security는 대부분 세션에서 바로 뽑히는 값(username, group, IP)만 조건으로 씀
- 세션 정보에 없는 조건(예: 다른 테이블을 조회해야 아는 담당 리전)은 서브쿼리가 필요한데, 정책 엔진이 매번 이 서브쿼리를 다시 도는 부담 때문에 지원 안 하거나 제한적임
- 대응: 로그인 시점에 세션 변수에 값을 미리 심어두거나, Dataset Materializer 패턴으로 미리 나눠 저장



**2. 복잡한 데이터 타입엔 컬럼 기반 권한이 그대로 안 먹힘**
- 컬럼이 nested structure(중첩 구조)면 컬럼 단위 GRANT를 그대로 적용 못 함 — struct 전체를 주거나 아예 안 주거나만 가능
- 대응: unnest(중첩 풀기)해서 필드들을 최상위 컬럼으로 분리한 별도 테이블을 만들거나, Dataset Materializer 패턴으로 미리 가공



**3. 쿼리 실행 오버헤드**
- row/column-level 보안은 매 쿼리마다 동적으로 조건/함수를 끼워 넣는 방식이라 추가 연산이 붙음
- 지연이 심하면 Dataset Materializer 패턴으로 그룹별 전용 테이블/뷰를 미리 만들어 완화 가능하나, 데이터 중복 저장과 접근 그룹이 늘어날수록 거버넌스 부담이 증가함



## (4) 예시

#### 기술 1: PostgreSQL — 컬럼 레벨 (GRANT 방식)

**Before — `dedp.users` 테이블**

|id|login|registered_datetime|
|---|---|---|
|1|user_a|2023-01-05|
|2|user_b|2023-03-11|

한 줄 요약: PostgreSQL에서 user_a에게 3개 컬럼만 조회 권한을 부여하는 GRANT 문
```sql
GRANT SELECT(id, login, registered_datetime) ON dedp.users TO user_a;
```
- ★ 핵심: `SELECT(컬럼목록)` — 일반 GRANT SELECT ON table과 다르게 컬럼을 괄호로 지정
- 결과: user_a가 SELECT *를 날리면 `ERROR: permission denied for table users` 발생
---



#### 기술 2: PostgreSQL — Row 레벨 (Row-Level Security 방식)

한 줄 요약: PostgreSQL에서 로그인 계정과 login 컬럼이 일치하는 row만 보이게 만드는 row-level 정책
```sql
ALTER TABLE dedp.users ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_row_access ON dedp.users USING (login = current_user);
```
- 첫 줄은 row-level 보안 스위치, 둘째 줄이 실제 규칙 등록
- ★ 핵심: `USING (login = current_user)` — 접속자의 로그인 계정과 login 컬럼 값이 같은 row만 노출
---


#### 기술 3: 네이티브 row-level 미지원 DB — View로 시뮬레이션

한 줄 요약: row-level 보안을 네이티브로 지원 안 하는 DB에서, 조건이 고정된 뷰로 row 필터링을 흉내내는 방법
```sql
CREATE VIEW users_blogs AS
SELECT ... FROM blogs WHERE table.blog_author = current_user
```
- ★ 핵심: `WHERE table.blog_author = current_user` — 뷰 자체에 조건이 고정돼 조회자마다 결과가 달라짐

---

#### 기술 4: AWS DynamoDB (NoSQL) — IAM 정책으로 row 레벨 흉내

한 줄 요약: DynamoDB에서 IAM 정책으로 partition key 기준 row-level 접근을 흉내내는 설정
```json
{
"Statement":[{
  "Effect":"Allow",
  "Action":[...],
  "Resource":["arn:aws:dynamodb:us-west-1:123456789012:table/users"],
  "Condition":{
    "ForAllValues:StringEquals":{
      "dynamodb:LeadingKeys":["${www.amazon.com:user_id}"]
    }
  }
}]
}
```
- ★ 핵심: `dynamodb:LeadingKeys` — partition key가 유저 ID로 시작하는 row만 허용
- 배경: DynamoDB는 SQL의 WHERE절이 없는 key-value 기반 NoSQL이라, IAM 레이어에서 key 매칭 조건으로 우회 구현
---


#### 기술 5: Databricks (Unity Catalog) — 컬럼 레벨 (마스킹 방식)

**Before — `visits` 테이블 (마스킹 미적용 가정)**

|visit_id|ip|
|---|---|
|v001|203.0.113.5|
|v002|198.51.100.7|

한 줄 요약: Databricks에서 engineers 그룹 멤버만 실제 ip 값을 보고 나머지는 가리는 컬럼 마스킹 함수와 테이블 정의

```sql
CREATE FUNCTION ip_mask(ip STRING)
  RETURN CASE WHEN is_member('engineers') THEN ip ELSE '.' END;

CREATE TABLE visits (
  visit_id STRING,
  ip STRING MASK ip_mask
);
```
- ★ 핵심: `CASE WHEN is_member('engineers') THEN ip ELSE '.' END` — 사용자가 engineers 그룹인지 매번 체크해서 원본값/`.`을 결정
- GRANT(기술1) 방식과 차이: SELECT *는 항상 성공하고, 값만 가려짐 (GRANT는 아예 에러남)
---

**분류 요약표**

| 기술                                           | 레벨  | 방식                                                  |
| -------------------------------------------- | --- | --------------------------------------------------- |
| PostgreSQL / Redshift                        | 컬럼  | GRANT SELECT(컬럼목록)                                  |
| BigQuery                                     | 컬럼  | 정책 태그 + Fine-Grained Reader role                    |
| Databricks / Snowflake                       | 컬럼  | 마스킹 함수                                              |
| Databricks / Redshift / BigQuery / Snowflake | Row | ROW FILTER / Row-Level Security / row access policy |
| Row-level 미지원 DB                             | Row | View로 조건 고정                                         |
| DynamoDB (NoSQL)                             | Row | IAM LeadingKeys 조건                                  |


## (5) 최신트렌드

**Databricks Unity Catalog**
- 정체: 여러 워크스페이스/테이블에 걸친 데이터 거버넌스 통합 레이어
- 예전 한계: 테이블·카탈로그마다 GRANT 문/마스킹 함수를 개별 정의해야 해서 정책이 흩어지고 일관성 유지가 어려웠음
- 요즘: 컬럼 마스킹, row filter를 한 군데서 정의하고 카탈로그 전체 테이블에 재사용 가능, 감사 로그도 자동으로 남음

**Snowflake Dynamic Data Masking + Row Access Policies**
- 정체: 컬럼 마스킹 정책, row 접근 정책을 독립 객체로 정의하는 Snowflake 네이티브 기능
- 예전 한계: 권한별로 다른 뷰를 겹겹이 쌓아서 유지보수해야 했고, 뷰가 늘어날수록 스키마가 지저분해졌음
- 요즘: 정책 객체를 한 번 만들어 여러 테이블에 attach만 하면 되니, 뷰 중복 없이 원본 테이블 하나로 여러 권한 그룹 커버 가능

**Apache Ranger (오픈소스, Hadoop/Trino 생태계)**
- 정체: 클러스터 전역의 접근 제어를 중앙에서 관리하는 오픈소스 정책 엔진
- 예전 한계: Hive, Trino, Kafka 등 컴포넌트마다 권한 시스템이 제각각이라 관리 포인트가 여러 개였음
- 요즘: 하나의 UI에서 정책을 정의하면 여러 엔진에 동시에 적용돼서, 온프레미스/오픈소스 스택에서도 통합 거버넌스 가능

**엔지니어 독백:**
- "실무에서 이 세 개 중 뭘 쓰냐는 결국 '어느 웨어하우스/레이크하우스를 쓰느냐'로 거의 결정돼. Databricks 쓰면 Unity Catalog, Snowflake 쓰면 그 안에 이미 내장된 기능을 그냥 쓰면 되는 거고."
- "예전엔 정책이 테이블마다 따로 흩어져 있어서, 신입이 들어오면 '이 컬럼 왜 안 보이지?' 하고 여기저기 찾아다니는 게 일이었어. 지금은 카탈로그 한 군데서 다 확인되니까 온보딩도 훨씬 빨라졌고."
- "Apache Ranger는 특히 클라우드 매니지드 서비스 안 쓰고 자체 Hadoop/Trino 클러스터 운영하는 회사에서 많이 봤어. 관리형 쓸 형편 안 되면 이게 사실상 표준이야."
<br><br><br><br>
# 패턴#42 Fine-Grained Accessor for Resources - <2.접근제어>

## (1) 문제상황

핵심 고통: 클라우드 리소스 권한이 필요 이상으로 넓게 열려있어서, job 하나의 실수가 관련 없는 데이터까지 훼손할 수 있다.

- 이번엔 테이블이 아니라 클라우드 리소스 얘기다. 보안 감사(security audit)에서 지적이 하나 들어왔다.
- 우리 데이터 처리 job 하나가, 원래 자기가 다뤄야 할 데이터셋만 건드려야 하는데, 실제 권한은 object store 안의 **모든** 데이터셋을 덮어쓸 수 있게 열려있었던 것.
- 예를 들어 `visits-processor`라는 job은 원래 `visits` 버킷만 써야 하는데, 실제 IAM 권한은 전체 버킷에 다 열려 있었던 상황. 이 job이 버그로 엉뚱한 경로에 write하면 다른 팀 데이터셋까지 날아갈 수 있는 거다.
- 감사관이 제시한 원칙이 **at-least privilege(최소권한 원칙)** — 각 컴포넌트에게 딱 필요한 만큼만 권한을 준다는 원칙. 지금 우리 시스템엔 이게 안 지켜지고 있던 것.


핵심 정리:
- 배경: 클라우드 오브젝트 스토어 등 리소스에 대한 권한이 필요 이상으로 넓게(overly broad) 부여된 상태
- 위험: 특정 job의 버그/실수가 관련 없는 다른 데이터셋까지 훼손할 수 있음
- 원칙: at-least privilege — 컴포넌트별로 실제 작업 범위만큼만 권한 부여
- 필요한 것: 앞 패턴(#41)이 DB 테이블 안에서의 세밀한 통제였다면, 이번엔 클라우드 리소스(오브젝트 스토어, 스트림 등) 단위에서 세밀한 통제를 구현하는 방법


## (2) 솔루션

주요컨셉: 리소스 자체에 권한을 붙이거나(resource-based), 사용자/서비스 신원에 권한을 붙인다(identity-based).

AWS, Azure, GCP 모두 at-least privilege를 구현할 수 있는 기능을 갖고 있고, 방식은 크게 두 갈래로 나뉜다.

**1. Resource-based (리소스 자체에 정책을 붙임)**
- 정체: "이 리소스(예: 특정 GCS 버킷)에 누가 접근 가능한가"를 리소스 쪽에 직접 정의하는 방식
- 비유: 건물(리소스)의 출입문에 "누구누구 출입 가능"이라는 명단을 붙여두는 것
- GCP GCS 버킷 IAM 정책이 이 방식의 대표 예시

**2. Identity-based (사용자/서비스 신원에 권한을 붙임)**
- 정체: "이 사용자/서비스(identity)가 어떤 리소스에 접근 가능한가"를 신원 쪽에 정의하는 방식
- 비유: 사람(신원)에게 "이 건물, 저 건물 출입 가능"이라는 출입증을 발급하는 것
- AWS는 이 방식을 IAM role로 지원 — 서비스가 role을 assume(위임)해서 임시 권한을 얻음

**추가: Tag 기반 접근 통제 (더 세밀한 응용)**
- resource/identity 두 방식 외에, 리소스에 붙은 태그(tag)를 조건으로 걸 수도 있음
- 예: S3에 파일 업로드 시 특정 태그를 가진 요청만 허용
- 앞 패턴(#41)의 row-level access와 개념적으로 유사 — 리소스 인프라 레벨에서 가장 낮은 단위까지 조건을 거는 방식

### AWS Lambda로 보는 두 방식

#### Identity-based — Lambda 함수(신원)에 권한을 붙임
**상황:** `visits-processor`라는 Lambda 함수가 S3의 `visits` 버킷만 읽고 써야 함

한 줄 요약: Lambda 함수가 assume하는 execution role에 S3 visits 버킷 권한만 부여하는 설정
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::visits/*"
}
```
- 이 정책을 **Lambda의 execution role**(Lambda가 실행될 때 자동으로 assume하는 IAM role)에 붙임
- ★ 핵심: 권한이 **Lambda 함수(신원) 쪽**에 붙어있음. `visits` 버킷은 이 role이 뭘 할 수 있는지 전혀 모름
- 이게 identity-based인 이유: "이 Lambda가 무엇에 접근 가능한가"를 Lambda의 role 정의서에서 관리

#### Resource-based — S3 버킷(리소스) 자체에 정책을 붙임
**같은 상황을 반대 방향으로:**

한 줄 요약: S3 버킷 정책에 특정 Lambda role만 접근 허용하도록 지정하는 설정
```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::123456789012:role/visits-processor-role"},
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::visits/*"
}
```
- 이 정책은 **S3 버킷의 정책(bucket policy)**에 직접 붙임
- ★ 핵심: 권한이 **버킷(리소스) 쪽**에 붙어있음. `visits-processor-role`이 자기가 이 버킷에 접근 가능하다는 걸 몰라도, 버킷이 "얘는 들어와도 돼"라고 허용해주는 구조
- 이게 resource-based인 이유: "이 버킷에 누가 접근 가능한가"를 버킷의 정책 정의서에서 관리

#### Lambda에서 실무적으로 둘을 같이 쓰는 경우
- **Cross-account(다른 AWS 계정) 접근**일 때: identity-based만으론 안 되고 resource-based도 같이 필요함
    - 예: A 계정의 Lambda가 B 계정의 S3 버킷을 읽으려면 → A 계정 Lambda role에 identity-based 권한 부여 **+** B 계정 버킷 정책에도 A 계정 role을 resource-based로 허용해야 함
    - 둘 중 하나만 있으면 접근 실패

**엔지니어 독백:**
- "같은 계정 안에서만 쓸 거면 보통 identity-based(execution role)만으로 충분해. 근데 cross-account 시나리오 만나는 순간 얘기가 달라져. 이때 resource-based를 안 쓰면 아무리 role에 권한을 줘도 상대 버킷이 막아버려서 계속 AccessDenied 떠."
- "신입 때 이거 몰라서 'role에 권한 다 줬는데 왜 안 되지' 하고 몇 시간 삽질한 적 있어. cross-account면 항상 양쪽 다 확인하는 습관 들여."


## (3) 결과

**1. 원칙대로 vs 관리 가능하게 — 트레이드오프**
- at-least privilege를 문자 그대로 지키면, 리소스 하나하나마다 정책을 따로 만들어야 해서 정책 개수가 폭발적으로 늘어남 → 복잡한 환경에선 유지보수가 힘들어짐
- 완화책: `visits*`처럼 prefix 와일드카드로 묶어서 관리
- 근데 이것도 부작용이 있음: `visits*`를 허용하면, 미래에 `visits-2027-new-dataset` 같은 게 새로 생겨도 자동으로 접근 권한이 열려버림 → "지금 필요한 것만 준다"는 원칙 자체를 어기는 셈
- 이 단순화는 반드시 보안팀과 협의해서 결정해야 함
#### Before — 원칙대로: 리소스 하나하나마다 정책을 따로 만든 상태
**한 줄 요약: 버킷 안의 파일 하나하나를 개별 리소스로 지정한 IAM 정책**
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": [
    "arn:aws:s3:::visits/visits-2024-01.parquet",
    "arn:aws:s3:::visits/visits-2024-02.parquet",
    "arn:aws:s3:::visits/visits-2024-03.parquet"
  ]
}
```
- 매달 새 파일이 쌓이는 구조라면, 매달 이 `Resource` 배열에 파일을 하나씩 추가해줘야 함
- **문제 상황(구체적 숫자):** 1년이면 12개, 여러 데이터셋(visits, orders, clicks...) 곱하면 리소스 목록이 수백 줄짜리 정책 파일이 됨. 새 파일 생길 때마다 정책 파일 수정 → 배포 → 리뷰까지 매번 반복해야 해서 유지보수가 마비됨

#### After — 완화책: 와일드카드로 묶기
**한 줄 요약: 접두사가 visits로 시작하는 모든 리소스를 한 번에 허용하는 IAM 정책**
```json
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::visits/visits-*"
}
```
- 정책 파일 한 줄로 끝. 새 파일이 몇 개가 쌓이든 정책을 다시 배포할 필요 없음
- 관리 부담은 확 줄어듦


**2. resource-based/identity-based 혼용 시 복잡도 증가**
- 두 방식을 한 프로젝트에서 섞어 쓰면 "이 권한은 어디서 온 거지?" 추적이 어려워짐
- 가능하면 한 가지 방식으로 통일, 그중에서도 자기 유스케이스를 더 많이 커버하는 쪽 선택

**3. 쿼터(quota) 한계**
- 정책도 결국 클라우드 리소스라 개수 제한이 있음
- AWS IAM은 기본 커스텀 정책 1,500개, GCP IAM은 프로젝트당 커스텀 role 300개 제한
- 일부는 벤더에 요청해서 늘릴 수 있지만 무한정은 아님 → 정책을 세밀하게 쪼갤수록 이 한계에 더 빨리 부딪힘


## (4) 예시

#### 기술 1: AWS S3 — 권한 없을 때 에러

**한 줄 요약: S3 버킷 리스팅을 시도했는데 권한이 없어서 AccessDenied 에러가 나는 상황**
```bash
$ aws s3 ls s3://dedp-visits-301JQN/
An error occurred (AccessDenied) when calling the
ListObjectsV2 operation: Access Denied
```
- 에러 메시지가 명확하게 `AccessDenied`라고 원인을 알려줌
---

#### 기술 2: AWS IAM — Identity-based 해결 (role에 권한 부여)

**한 줄 요약: IAM role에 특정 S3 버킷 읽기 권한만 부여하는 정책**
```json
{"Version": "2012-10-17", "Statement": [{"Sid": "VisitsS3Reader",
"Effect": "Allow", "Action": ["s3:Get*", "s3:List*"],
"Resource": ["arn:aws:s3:::dedp-visits-301JQN/*",
             "arn:aws:s3:::dedp-visits-301JQN"]
}]}
```
- 이 정책을 role에 붙여서, 그 role을 가진 identity(job)에게 `dedp-visits-301JQN` 버킷 읽기 권한을 부여
- ★ 핵심: `Resource` 배열에 정확히 이 버킷 하나만 지정 — 다른 버킷은 이 role로 접근 불가. at-least privilege를 실제로 구현하는 부분
- 참고: `/*` 붙은 ARN은 버킷 안의 객체들, `/*` 없는 ARN은 버킷 자체(리스팅 등)에 대한 권한 — 둘 다 필요해서 두 줄 다 포함
---

#### 기술 3: AWS S3 Bucket Policy — Resource-based 해결 (버킷에 사용자 부여)

**한 줄 요약: S3 버킷 정책에 특정 IAM 유저를 읽기 권한으로 추가하는 명령과 정책 파일**
```bash
$ aws s3api put-bucket-policy --bucket dedp-visits-301JQN --policy file://policy.json
```

```json
{"Statement": [{"Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::123456789012:user/visits-s3-reader"},
  "Action": ["s3:Get*", "s3:List*"],
  "Resource": "arn:aws:s3:::dedp-visits-301JQN/*"
}]}
```
- `aws s3api put-bucket-policy`: 로컬의 policy.json 파일을 읽어서 지정한 버킷(`--bucket`)에 정책을 적용하는 명령
- ★ 핵심: `"Principal": {"AWS": "...user/visits-s3-reader"}` — 기술2와 정반대 구조. 기술2는 identity에 리소스 목록을 붙였다면, 이건 리소스에 identity 목록을 붙인 것
---

#### 기술 4: GCP GCS — Resource-based (Terraform)

**한 줄 요약: GCS 버킷에 admin 권한을 가진 사용자를 IAM 정책으로 바인딩하는 Terraform 코드**
```hcl
data "google_iam_policy" "admin_access" {
  binding {
    role = "roles/storage.admin"
    members = ["user:admingcs@waitingforcode.com"]
  }
}
resource "google_storage_bucket_iam_policy" "policy" {
  bucket      = google_storage_bucket.default.name
  policy_data = data.google_iam_policy.admin_access.policy_data
}
```
- ★ 핵심: `google_storage_bucket_iam_policy` — 정책이 `google_storage_bucket.default`라는 버킷(리소스)에 붙어있는 구조
---


#### 기술 5: AWS Kinesis — Identity-based (Terraform, EMR job)

**한 줄 요약: EMR Spark job이 assume하는 role에 Kinesis visits 스트림 읽기/쓰기 권한만 부여하는 Terraform 코드**
```hcl
resource "aws_iam_role" "job_role" {
  name = "visits-processor-role"
  assume_role_policy = data.aws_iam_policy_document.emr_assume_role.json
}
resource "aws_iam_policy" "visits_read_writer_policy" {
  policy = jsonencode({
    Statement = [{
      Action   = ["kinesis:Get*", "kinesis:Describe*", "kinesis:List*", "kinesis:Put*"]
      Effect   = "Allow"
      Resource = ["arn:aws:kinesis:us-east-1:1234567890:streams/visits"]
    }]
  })
}
```
- ★ 핵심: `Resource = ["arn:...streams/visits"]` — `visits-processor-role`이라는 identity에게 정확히 이 스트림 하나만 읽고 쓸 권한을 부여
---



#### 기술 6: Databricks Unity Catalog — Identity-based (Service Principal)

**한 줄 요약: service principal에게 특정 카탈로그의 스키마 하나에만 읽기 권한을 부여하는 SQL 문**
```sql
GRANT USE CATALOG ON CATALOG dedp_catalog TO `visits-processor-sp`;
GRANT USE SCHEMA, SELECT ON SCHEMA dedp_catalog.visits TO `visits-processor-sp`;
```
- `visits-processor-sp`는 job 전용으로 만든 service principal (사람 계정이 아니라 애플리케이션용 신원)
- ★ 핵심: `ON SCHEMA dedp_catalog.visits` — 카탈로그 전체가 아니라 `visits`라는 스키마 하나에만 권한을 좁혀서 부여. 다른 스키마(예: `payroll`)는 이 service principal로 절대 접근 불가
- Unity Catalog는 계층 구조(catalog → schema → table)라서, 상위 레벨(`USE CATALOG`)은 그냥 "존재를 인식하고 통과"할 권한만 주고, 실제 데이터 접근 권한(`SELECT`)은 원하는 하위 스키마/테이블에만 정확히 지정하는 게 실무 패턴

**이게 왜 identity-based인가:**
- 권한이 리소스(스키마)가 아니라 `visits-processor-sp`라는 신원 쪽 관점에서 "얘는 여기까지만 접근 가능"으로 관리됨 — AWS IAM role, GCP service account와 개념적으로 동일한 위치

**GUI 방식 (Databricks 워크스페이스 콘솔)**
- Admin Console → Service Principals 메뉴에서 생성
- Catalog Explorer에서 스키마/테이블 우클릭 → Permissions 탭에서 service principal 선택하고 권한 체크박스 클릭
- 가장 흔하게 쓰는 방법, 특히 처음 세팅하거나 소수 인원이 수동 관리할 때
---
## (5) 최신트렌드

**AWS IAM Access Analyzer**
- 정체: 실제 사용 로그(CloudTrail)를 분석해서 "이 role이 진짜로 쓴 권한"만 뽑아 최소 권한 정책을 자동 생성해주는 서비스
- 예전 한계: 정책을 처음 짤 때 "혹시 몰라서" 넓게 주고 나중에 좁히는 걸 계속 미루는 경우가 많았음
- 요즘: policy generation 기능으로 일정 기간 실제 API 호출 이력을 보고 최소 정책 초안을 자동 생성 → 감사 대응 시간이 크게 줄어듦

**Terraform + Sentinel / OPA (Open Policy Agent)**
- 정체: IaC 배포 전에 정책 위반을 자동으로 걸러내는 policy-as-code 도구
- 예전 한계: `visits*` 같은 느슨한 와일드카드 정책이 코드 리뷰에서 사람 눈에 안 걸리고 그대로 배포되는 경우가 많았음
- 요즘: "특정 패턴의 Resource ARN은 배포 금지" 같은 룰을 CI 파이프라인에 심어서, 리뷰어가 놓쳐도 배포 단계에서 자동 차단

**Cloud-native 권한 그래프 도구 (AWS IAM Policy Simulator, GCP Policy Analyzer)**
- 정체: "이 identity가 이 리소스에 실제로 어떤 경로로 접근 가능한지"를 그래프로 시각화하는 도구
- 예전 한계: identity-based/resource-based 정책이 여러 겹 쌓이면, 최종적으로 누가 뭘 접근 가능한지 사람이 머리로 추적하기 거의 불가능했음
- 요즘: 정책 변경 전에 시뮬레이션 돌려서 "이 변경이 의도한 대로 동작하는지" 미리 검증 가능 → 배포 후 사고 확률이 줄어듦

**엔지니어 독백:**
- "옛날엔 정책을 넓게 주고 감사 지적받으면 그때야 좁히는 게 일반적이었어. 근데 Access Analyzer 나오고부터는 반대로 가. 처음부터 좁게 주고, 실제 쓰는 걸 보면서 부족하면 늘리는 방향으로 바뀌었어."
- "OPA 같은 policy-as-code는 특히 팀이 커지면서 진가를 봐. 예전엔 리뷰어가 정책 파일 하나하나 눈으로 봐야 했는데, 지금은 CI에서 자동으로 걸러주니까 '이거 통과됐으면 됐겠지' 하고 믿을 수 있어."
- "권한 그래프 도구는 사고 나고 나서야 찾아보는 경우가 많은데, 사실 배포 전에 습관적으로 시뮬레이션 돌려보는 게 훨씬 싸게 먹혀. 사고 나서 추적하는 시간이 훨씬 오래 걸려."<br><br><br><br>
# 패턴#43 Encryptor - <3.데이터보호(암호화,마스킹)>

## (1) 문제상황
핵심 고통: 접근 통제가 뚫리면, 데이터 자체가 평문으로 그대로 노출된다.

- 테이블 권한(#41), 클라우드 리소스 권한(#42)까지 다 세밀하게 잠갔다. 근데 스테이크홀더가 또 물어본다. "만약 그 권한 설정이 뚫리면? 그땐 데이터가 그냥 평문으로 노출되는 거 아니냐"고.
- 맞는 말이다. 접근 통제는 어디까지나 "허가된 사람만 문을 열 수 있게" 하는 것이지, 문이 뚫렸을 때 안에 있는 내용물 자체를 보호하진 못한다.
- 구체적으로 두 가지 시나리오가 걱정거리로 나왔다. 첫째, 스트리밍 브로커랑 job 사이를 오가는 데이터를 누가 네트워크에서 가로채면(intercept)? 둘째, 서버에 물리적으로 접근한 사람이 디스크에서 데이터를 통째로 훔쳐가면(steal)?
- 이 두 시나리오는 각각 "전송 중(in transit)" 데이터랑 "저장된(at rest)" 데이터라는 서로 다른 보호 대상이다. 접근 통제만으론 이 둘을 못 막고, 데이터 자체를 읽어도 못 쓰게 만드는 별도 방법이 필요하다.

핵심 정리:
- 배경: #41, #42로 접근 통제(누가 볼 수 있는가)는 다 갖춤
- 남은 위험: 접근 통제가 뚫리는(우회되는) 경우 — 네트워크 가로채기, 서버 물리 탈취
- 필요한 것: 데이터 자체를 암호화해서, 접근 통제와 별개로 "훔쳐도 못 읽게" 만드는 


## (2) 솔루션

주요컨셉: 저장 데이터(at rest)와 이동 중 데이터(in transit), 두 구간을 각각 암호화한다.
보호해야 할 구간이 두 개니까, 구현도 두 갈래로 나뉜다.


#### 저장 데이터(Data at rest) 암호화 — 방식 2가지

**1. Client-side 암호화**
- 정체: 데이터를 스토리지에 보내기 **전에**, producer(데이터 생성 쪽)가 직접 암호화
- 암호화 키 관리 책임도 producer가 가짐

**2. Server-side 암호화**
- 정체: producer/consumer는 평범하게 요청만 보내고, 암호화·복호화·키 관리를 전부 스토리지 서버 쪽이 알아서 처리
- 클라우드 벤더별 키 관리 서비스: AWS/GCP는 KMS(Key Management Service), Azure는 Key Vault
- 대부분의 실무에서 이 방식을 씀. producer/consumer 코드 변경 거의 없이 스토리지 설정만으로 적용 가능

**Server-side 암호화 동작 순서 (읽기 요청 기준):**
1. 클라이언트가 암호화된 데이터 스토어에 요청을 보냄
2. 스토어가 바로 데이터를 안 주고, KMS(암호화키 저장소)에 복호화 키를 요청
3. 클라이언트에게 권한이 없으면 여기서 요청 실패. 있으면 계속 진행
4. 스토어가 받은 키로 데이터를 복호화한 뒤 클라이언트에게 반환

- 이 전체 과정은 클라우드 사용자 입장에선 완전히 추상화돼서 안 보임. 할 일은 "암호화 정책 설정 + 키 스토어 접근 권한 관리"뿐

#### 이동 중 데이터(Data in transit) 암호화
- 저장 데이터보다 구현이 훨씬 간단함
- 클라이언트 SDK에서 보안 통신 활성화 + 서비스 쪽에 필요한 프로토콜 버전 설정, 이 두 가지면 끝
- 대표 프로토콜: TLS(Transport Layer Security) — HTTPS에서 쓰이는 암호화 채널


## (3) 결과

**1. 암호화/복호화 오버헤드 (CPU 부담)**
- 데이터가 평문이 아니라 변형된(암호화된) 형태로 저장되니까, 읽고 쓸 때마다 암호화·복호화 연산이 추가로 들어감
- 매 read/write 요청마다 CPU에 추가 부하가 걸림


**2. 데이터 로스 위험 (키를 잃어버리면 데이터도 끝)**
- 암호화는 원래 "무단 접근자가 못 읽게" 막는 게 목적인데, 부작용으로 정당한 사용자도 못 읽게 만들 수 있음
- 이런 일이 생기는 유일한 경우: 암호화 키를 잃어버리거나(삭제) 키 접근 권한을 잃어버릴 때
- 완화책: 클라우드 벤더들이 키 스토어에 soft delete(유예 삭제)를 기본 적용 — 삭제 요청해도 즉시 지워지지 않고, 실수로 지운 경우 복구 가능한 유예 기간을 둠


**3. 프로토콜 유지보수 부담 (in transit 쪽)**
- in transit 암호화는 설정 자체는 at rest보다 훨씬 쉽지만, 계속 최신 상태로 유지해야 하는 별도 컴포넌트가 하나 늘어나는 셈
- 예: TLS 1.0, 1.1 버전은 보안 이슈가 발견돼서 폐기(deprecated)됨. 이런 오래된 버전을 쓰는 서비스는 반드시 업그레이드해야 함
- 다만 클라우드 환경에선 이 업그레이드가 대부분 서비스 레벨에서 프로토콜 버전 하나 바꾸는 정도로 단순화되어 있음



## (4) 예시

#### 기술 1: AWS KMS — 암호화 키 정의 (Terraform)

**한 줄 요약: KMS에 암호화 키를 만들고 특정 role에게 암호화/복호화 권한을 부여하는 Terraform 코드**
```hcl
module "kms" {
  source = "terraform-aws-modules/kms/aws"
  key_usage = "ENCRYPT_DECRYPT"
  deletion_window_in_days = 14
  aliases = ["visits-bucket-encryption-key"]
  grants = {
    lambda_doc_convert = {
      grantee_principal = aws_iam_role.iam_key_reader.arn
      operations = ["Encrypt", "Decrypt", "GenerateDataKey"]
    }
  }
}
```
- `key_usage = "ENCRYPT_DECRYPT"`: 이 키를 암호화/복호화 용도로 쓰겠다는 선언
- ★ 핵심: `deletion_window_in_days = 14` — (3)결과에서 본 "키 삭제 시 데이터 로스 위험"을 방지하는 유예 기간 설정. 실수로 키를 삭제 요청해도 14일 동안은 복구 가능
- `grants`: 이 키를 `iam_key_reader` role에게 암호화/복호화 권한으로 내어줌
---

#### 기술 2: AWS S3 — 버킷에 KMS 키 연결 (Terraform)

**한 줄 요약: 방금 만든 KMS 키를 S3 버킷의 기본 서버사이드 암호화 키로 지정하는 Terraform 코드**
```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "visits" {
  bucket = aws_s3_bucket.visits.id
  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = module.kms.key_arn
      sse_algorithm = "aws:kms"
    }
  }
}
```
- ★ 핵심: `kms_master_key_id = module.kms.key_arn` — 기술1에서 만든 키를 이 버킷의 기본 암호화 키로 지정
- 이 설정 이후 `visits` 버킷에 write되는 모든 객체는 자동으로 이 키로 암호화됨. producer/consumer 코드는 전혀 안 건드려도 됨 (server-side 암호화 방식)
---

#### 기술 3: Azure Event Hubs — In-transit 암호화 (최소 TLS 버전 강제)
**한 줄 요약: Event Hubs 네임스페이스에 최소 TLS 버전을 지정해서 오래된 프로토콜 클라이언트 연결을 차단하는 설정**
```hcl
resource "azurerm_eventhub_namespace" "visits" {
  minimum_tls_version = "1.2"
}
```
- ★ 핵심: `minimum_tls_version = "1.2"` — TLS 1.0/1.1 같은 폐기된(deprecated) 버전으로 접속 시도하는 클라이언트는 자동으로 연결 거부됨
- at-rest 설정(기술1~2, 2단계 리소스 정의) 대비 in-transit 설정이 훨씬 가볍다는 걸 코드 줄 수로도 확인 가능




## (5) 최신트렌드

**S3 기본 암호화 (SSE-S3 default encryption)**
- 정체: AWS S3가 버킷 생성 시 서버사이드 암호화를 자동으로 기본 적용하는 기능
- 예전 한계: 개발자가 암호화 설정을 깜빡하면 그냥 평문으로 저장돼버렸고, 보안 감사에서야 뒤늦게 발견되는 경우가 많았음
- 요즘: 별도 설정 없이 새 버킷을 만들면 자동으로 암호화가 걸려있어서, 설정을 빼먹어서 뚫리는 사고 자체가 원천 차단됨


**Envelope Encryption (봉투 암호화) 패턴 확산**
- 정체: 실제 데이터는 빠른 대칭키(data key)로 암호화하고, 그 data key 자체는 KMS의 마스터 키로 다시 한번 암호화해서 보관하는 이중 구조
- 예전 한계: 모든 데이터를 KMS 마스터 키로 직접 암호화하면 API 호출량이 많아져서 느리고 비쌈
- 요즘: data key는 로컬에서 빠르게 쓰고, 마스터 키는 그 data key 하나만 보호하면 되니 KMS 호출 횟수가 확 줄어듦. Databricks, Snowflake 등 대부분의 매니지드 서비스가 내부적으로 이 방식을 씀


**Confidential Computing (기밀 컴퓨팅)**
- 정체: 데이터가 저장(at rest)/전송(in transit) 중일 때뿐 아니라, 연산되는 순간(in use)에도 암호화된 상태를 유지하는 하드웨어 기반 기술 (예: AWS Nitro Enclaves, GCP Confidential VM)
- 예전 한계: 기존 Encryptor 패턴은 at rest, in transit 두 구간만 커버 — 데이터가 메모리에 올라가서 실제로 처리되는 순간엔 결국 평문으로 노출됨
- 요즘: 금융/헬스케어처럼 규제가 엄격한 도메인에서 "연산 중에도 클라우드 관리자조차 데이터를 못 본다"는 요구가 늘면서, 세 번째 보호 구간으로 자리잡는 중


**엔지니어 독백:**
- "예전엔 암호화가 '해야 되는 옵션'이었는데, 요즘은 안 하는 게 오히려 이상한 상황이 됐어. S3 기본 암호화 나오고부터는 신입이 버킷 새로 만들 때 암호화 설정을 아예 신경 안 써도 될 정도야."
- "Envelope Encryption은 처음 보면 왜 굳이 키를 두 겹으로 감싸나 싶은데, 실제로 KMS API 호출량 줄여야 하는 이유가 명확해. 대용량 데이터 파이프라인 돌리면 KMS 직접 호출 비용/레이턴시가 눈에 띄게 체감돼."
- "Confidential Computing은 아직 우리 팀에선 직접 쓸 일 없었는데, 금융권 컨설팅 하는 친구 얘기 들어보면 이게 없으면 아예 계약 자체가 안 되는 케이스도 있더라. 규제 산업 가면 이게 표준이 될 걸로 봐."
