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

<br><br>


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


<br><br><br><br><br><br><br><br>
