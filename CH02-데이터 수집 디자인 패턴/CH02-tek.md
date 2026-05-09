# 데이터 엔지니어링 디자인 패턴 - 데이터 수집 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 2 | 실무 데이터 엔지니어링 관점 정리

---

## 목차

1. [전체 적재 (Full Load)](#1-전체-적재-full-load)
   - 패턴 #01: 전체 로더 (Full Loader)
2. [증분 적재 (Incremental Load)](#2-증분-적재-incremental-load)
   - 패턴 #02: 증분 로더 (Incremental Loader)
   - 패턴 #03: 변경 데이터 캡처 (Change Data Capture)
3. [복제 (Replication)](#3-복제-replication)
   - 패턴 #04: 패스스루 복제기 (Passthrough Replicator)
   - 패턴 #05: 변환 복제기 (Transformation Replicator)
4. [데이터 컴팩션 (Data Compaction)](#4-데이터-컴팩션-data-compaction)
   - 패턴 #06: 컴팩터 (Compactor)
5. [데이터 준비 (Data Readiness)](#5-데이터-준비-data-readiness)
   - 패턴 #07: 준비 마커 (Readiness Marker)
6. [이벤트 주도 (Event Driven)](#6-이벤트-주도-event-driven)
   - 패턴 #08: 외부 트리거 (External Trigger)
7. [요약](#7-요약)

---

## 책의 use case (전체 패턴 공통)

이 책 전체가 **블로그 분석 플랫폼(blog analytics platform)** 케이스 스터디 위에서 진행됨.
챕터 2의 모든 패턴은 이 플랫폼의 데이터 수집 단계에서 발생하는 문제를 해결.

```
블로그 분석 플랫폼 데이터 흐름 (Medallion 아키텍처)
==========================================================================

[방문 이벤트 — 실시간]                [참조 데이터 — 시간당]
  (사용자가 블로그 방문)                (디바이스 정보, 외부 API 등)
    │                                       │
    ▼                                       ▼
  [API Gateway]                       [Data Provider 1, 2]
    │                                       │
    ▼                                       │
  [Raw data topic]                          │
    (Kafka)                                 │
    │                                       ▼
    │                                 [Data ingestion job 1, 2]
    ├──────► [Stateful job]                 │
    ├──────► [Synchronization job]          │
    │                                       │
    └─────────────────┬─────────────────────┘
                      ▼
            [Lakehouse storage — Medallion]
            ┌────────────────────────────────────┐
            │ Bronze : 원시 데이터 (가공 없음)     │
            │ Silver : 정제 + 보강 데이터         │
            │ Gold   : 비즈니스 사용자용 최종 형식  │
            └────────────────────────────────────┘
                      │
                      ▼
            [BI 대시보드 / ML 모델 / 데이터 마트]

==========================================================================

각 패턴이 적용되는 위치:
  #01 전체 로더    : 외부 → Silver (디바이스 참조 데이터)
  #02 증분 로더    : 레거시 트랜잭션 DB → Bronze (방문 이벤트)
  #03 CDC          : 트랜잭션 DB → Streaming broker (저지연 요구)
  #04 패스스루 복제 : production → dev/staging (참조 데이터)
  #05 변환 복제    : production → staging (PII 제거)
  #06 컴팩터       : Bronze 작은 파일 → 큰 파일 병합
  #07 준비 마커    : Silver 적재 완료 알림
  #08 외부 트리거  : 백엔드 release 이벤트로 적재 트리거
```

---

## 1. 전체 적재 (Full Load)

전체 적재는 매번 데이터셋 전체를 가져오는 가장 단순한 데이터 수집 시나리오.
데이터베이스 부트스트랩이나 참조 데이터셋(reference dataset) 생성에 유용.

### 1-1. 패턴 #01: 전체 로더 (Full Loader)

데이터 수집의 가장 단순한 두 단계 구조 (extract + load) 패턴.
구조는 단순하지만 함정이 있음.

#### 상황 (Problem)

**책의 use case:**
- Silver 레이어를 구축 중. 변환 작업 중 하나가 외부 데이터 제공자로부터 디바이스 정보를 추가로 받아야 함.
- 이 디바이스 데이터셋은 일주일에 몇 번만 변경되는 **느리게 진화하는(slowly evolving)** 엔티티.
- 전체 행 수는 100만 건을 넘지 않음.
- **결정적 제약**: 데이터 제공자가 마지막 적재 이후 변경된 행을 식별할 수 있는 어떤 속성도 제공하지 않음.

→ 변경분을 가려낼 방법이 없으므로 매번 전체를 가져올 수밖에 없음.

#### 해결 (Solution)

마지막 갱신 컬럼이 없다는 것이 오히려 Full Loader 패턴이 이상적인 해결책이 되는 이유.

가장 단순한 구현은 두 단계 — **EL (Extract-Load)**:
- 데이터 저장소의 native 명령으로 한 DB에서 export 하고 다른 DB로 import
- 동질(homogeneous) 데이터 저장소 간이라면 변환 불필요

**Passthrough Jobs** — Extract-load 작업은 데이터가 소스에서 목적지로 단순 통과하므로 패스스루 작업이라고도 불림.

이종(heterogeneous) DB 간이라면 입력 형식을 출력 형식에 맞추는 얇은 변환 레이어가 필요 → **ETL** 작업이 됨.

```
전체 로더 동작 흐름
--------------------------------------------------------------
[t = T1]                          [t = T2]
원천: 100만 건                    원천: 102만 건
  │                                 │
  └─► 전체 SELECT                   └─► 전체 SELECT
        │                                 │
        ▼                                 ▼
타겟 (overwrite)                  타겟 (overwrite)
  - 100만 건                        - 102만 건 (T1 데이터 완전 교체)
  - T1 상태 사라짐                  - 삭제된 행도 자동 반영됨
--------------------------------------------------------------
```

#### 고려사항 (Consequences)

데이터를 두 저장소 사이에 옮기는 단순한 작업처럼 보여도 함정이 있음.

**Data volume — 데이터 볼륨**
- 보통은 정기 스케줄로 도는 배치 작업
- 데이터 볼륨이 천천히 늘어나면 거의 일정한 컴퓨트로 오래 잘 돌아감
- 더 동적으로 진화하는 데이터셋이라면 정적 컴퓨트 자원으로는 부족
  - 예: 데이터셋이 하루 사이 두 배가 되면 적재가 느려지거나 하드웨어 한계로 실패
- 대응: 처리 레이어의 **자동 스케일링(auto-scaling)** 활용

**Data consistency — 데이터 일관성**
- 데이터를 완전히 덮어써야 하므로 매 실행에서 `drop-and-insert` 가 가능
- 그러나 적재 중에 컨슈머가 읽으면 부분 데이터 또는 빈 결과 발생
- 트랜잭션이 가장 쉬운 완화책 — 가시성을 자동 관리
- 트랜잭션 미지원 저장소라면 **단일 데이터 노출 추상화(single data exposition abstraction)** — 예: view — 를 활용해 내부 기술 테이블만 교체

```
단일 데이터 노출 추상화 (Single Data Exposition Abstraction)
--------------------------------------------------------------
적재 전:                          적재 후:
  Table_1 (현재 사용 중)            Table_1
  Table_2 (대기)                   Table_2 (현재 사용 중)
       └── Public view ──┐              └── Public view ──┐
                         ▼                                ▼
                  컨슈머가 view 통해 접근           컨슈머는 끊김 없이 새 데이터
--------------------------------------------------------------
```

**이전 버전 보존 필요**
- 예상치 못한 문제 발생 시 이전 버전을 사용해야 할 수 있음
- 완전히 덮어쓰면 복구 불가 — Delta Lake / Apache Iceberg / GCP BigQuery 같은 **time travel** 지원 형식 사용 권장

#### 구현 예시 (Examples)

**예시 1 — 호환 가능한 데이터 저장소 간 (단순 스크립트)**

S3 버킷 간 동기화. `--delete` 인자는 소스에 없는 객체를 목적지에서 제거함:
```bash
aws s3 sync s3://input-bucket s3://output-bucket --delete
```

**예시 2 — Apache Spark + Delta Lake**

대용량 처리를 위해 분산 프레임워크 사용. Delta Lake로 트랜잭션/버저닝 무료 제공:
```python
input_data = spark.read.schema(input_data_schema).json("s3://devices/list")
input_data.write.format("delta").save("s3://master/devices")
```

**예시 3 — Apache Airflow + PostgreSQL (native versioning 없는 DB)**

데이터 적재 task와 데이터 노출 task를 분리. 명시적으로 버전된 테이블에 적재 후 view를 새 테이블로 가리키게 함:
```sql
-- 데이터 적재 task: 새 버전 테이블 생성
COPY devices_${version} FROM '/data_to_load/dataset.csv' CSV DELIMITER ';' HEADER;

-- 데이터 노출 task: view를 새 테이블로 교체
CREATE OR REPLACE VIEW devices AS SELECT * FROM devices_${version};
```

| 구현 방식 | 장점 | 적합 상황 |
|----------|------|----------|
| `aws s3 sync` 같은 native 명령 | 가장 단순, 변환 비용 0 | 동질 저장소 간 |
| Spark + Delta Lake | autoscaling + ACID + time travel 무료 | 분산 처리가 필요한 대규모 |
| Airflow + 버전 테이블 + view | 트랜잭션 미지원 DB도 안전 교체 가능 | PostgreSQL 등 native versioning 없는 DB |

> **트러블 로그** — `drop-and-insert` 를 view 추상화 없이 prod 서비스 테이블에 직접 적용하면
> 적재가 진행되는 1~2분 동안 BI 대시보드가 빈 결과로 보여 경영진에 잘못된 0원 매출이 표시되는 사고 발생.
> 예: 디바이스 참조 테이블 100만 건을 매일 새벽 4시에 drop-insert 하다가
> 그 시간대 호주 지사 대시보드(현지 시각 12시)가 평균 90초 동안 device 정보가 비어
> 디바이스별 세션 통계가 모두 NULL로 표시된 사례가 있음.
> 반드시 `Table_v1 → Table_v2` 전환을 view 또는 트랜잭션으로 감싸 atomic swap 으로 만들 것.

---

## 2. 증분 적재 (Incremental Load)

전체 적재는 단순하지만 계속 커지는 데이터셋에는 비용이 큼.
증분 적재는 데이터셋의 작은 부분만 — 보통 더 높은 빈도로 — 가져옴.

### 2-1. 패턴 #02: 증분 로더 (Incremental Loader)

데이터셋의 새로운 부분만 처리하는 첫 번째 증분 디자인 패턴.

#### 상황 (Problem)

**책의 use case:**
- 블로그 분석 플랫폼에서 대부분의 방문 이벤트는 실시간 스트리밍 브로커에서 옴.
- 그런데 일부 이벤트는 여전히 **레거시 producer가 트랜잭션 DB**에 쓰고 있음.
- 이 레거시 방문 이벤트를 Bronze 레이어로 가져오는 전용 적재 프로세스가 필요.
- 데이터 볼륨이 계속 증가하므로 **마지막 실행 이후 추가된 방문만** 통합해야 함.
- 각 방문 이벤트는 immutable (수정/삭제 없음).

#### 해결 (Solution)

지속적으로 증가하는 데이터셋에 적합. 입력 데이터 구조에 따라 두 가지 구현:

**구현 1: Delta column 방식**
- 마지막 실행 이후 추가된 행을 식별하기 위한 **delta column** 사용
- 이벤트 기반 데이터(예: immutable 방문)에서는 보통 **ingestion time**
- 마지막 ingestion time 값을 기억해야 함

**구현 2: Time-partitioned datasets 방식**
- 시간 기반 파티션을 사용해 새로 적재할 레코드 그룹을 감지
- 데이터가 이미 스토리지 레이어에서 논리적으로 정리되어 있어 프로세스 단순화
- 새 파티션이 적재 가능한지 확인하려면 **Readiness Marker 패턴(#07)** 활용
- 실행 날짜로부터 처리할 파티션을 암시적으로 결정 — 예: 11:00에 실행되면 이전 시간 파티션 처리

```
두 가지 증분 로더 구현
--------------------------------------------------------------

[Delta column 구현]
  event_id  ingestion_time
     1         09:59           ┌─────────────────────────────┐
     2         10:01     ───►  │ Incremental loader          │ ───► Output DB
     3         10:02           │ last ingestion time = 10:00 │      (write 2, 3)
                               └─────────────────────────────┘
                                     ▼
                               Set new last ingestion time to 10:02

[Partition-based 구현]
  Partition = 9:00              ┌─────────────────────────────┐
  Partition = 10:00      ───►  │ Incremental loader          │ ───► Partition = 9:00
                               │ run at 11:00 for 10:00      │
                               │ partition is being written  │
                               └─────────────────────────────┘
--------------------------------------------------------------
```

> **이벤트 시간 사용 시 주의**
> Delta column으로 `event time`을 쓰는 것은 위험.
> 데이터 producer가 이미 처리된 event time에 대해 **늦은 데이터(late data)** 를 발행하면 적재 프로세스가 그 레코드를 놓침.

#### 고려사항 (Consequences)

증분 방식은 적재 데이터 볼륨을 줄이는 장점이 있지만 까다로운 점도 있음.

**Hard deletes — 물리적 삭제**
- mutable 데이터에는 사용이 까다로움
- delta column으로는 **삭제된 행을 감지할 수 없음** (delta column 자체가 사라짐)
- 대응 1: **Soft deletes** — producer가 물리적 제거 대신 "삭제됨" 마킹 (DELETE 대신 UPDATE)
- 대응 2: **Insert-only tables** — INSERT만 받고 데이터 재구성 책임을 컨슈머가 짐 (append-only)

**Backfilling — 백필 위험**
- 두 달치 데이터 처리 후 백필 요청이 들어오면 증분이 아니라 전체 적재가 됨
- 추가 행을 수용할 추가 자원 필요
- 완화책: **적재 윈도우 제한** — `delta_column BETWEEN ingestion_time AND ingestion_time + INTERVAL '1 HOUR'`
  - 백필 시에도 컴퓨트 수요가 예측 가능
  - 동일 백필 작업 동시 실행 가능 (입력 저장소가 지원하는 한)
- Partition-based 구현은 한 번에 한 파티션씩 작업하므로 이 문제 자체가 없음

#### 구현 예시 (Examples)

**예시 1 — Partition prefix 동기화:**
```bash
# date=2024-01-01 prefix만 동기화 (target에 prefix 유지)
aws s3 sync s3://input/date=2024-01-01 s3://output/date=2024-01-01 --delete

# 주의: date=2024-01-01 prefix를 우측에서 빠뜨리면 출력 레이아웃이 평탄화됨
```

**예시 2 — Airflow + Spark의 Partition-based 구현:**
```python
# 다음 파티션이 준비될 때까지 대기 (FileSensor)
next_partition_sensor = FileSensor(
    task_id='input_partition_sensor',
    filepath=get_data_location_base_dir() + '/{{ data_interval_end | ds }}',
    mode='reschedule',
)

# 파티션 준비되면 Spark 작업 트리거
load_job_trigger = SparkKubernetesOperator(application_file='load_job_spec.yaml')
load_job_sensor = SparkKubernetesSensor()

next_partition_sensor >> load_job_trigger >> load_job_sensor
```

```yaml
# Spark 작업의 입력/출력은 immutable execution time 표현 사용 — 백필 단순화
arguments:
  - "/data_for_demo/input/date={{ ds }}"
  - "/data_for_demo/output/date={{ ds }}"
```

**예시 3 — Delta column + 시간 경계 필터:**
```python
in_data = (spark_session.read.text(input_path)
    .select('value', functions.from_json(
        functions.col('value'), 'ingestion_time TIMESTAMP')))

# delta column으로 시간 윈도우 필터링 — 재실행해도 동일 윈도우만 처리
input_to_write = in_data.filter(
    f'ingestion_time BETWEEN "{date_from}" AND "{date_to}"'
)

input_to_write.mode('append').select('value').write.text(output_path)
```

> **트러블 로그** — Delta column으로 `event_time` 을 쓰면 producer 의 늦은 데이터에 그대로 당함.
> 예: 모바일 앱에서 오프라인 큐잉된 방문 이벤트가 사용자가 다음날 와이파이 잡힌 뒤 한꺼번에 들어옴.
> 적재 작업이 어제 14:00 까지 처리해 워터마크가 14:00이면, 새로 도착한 어제 11:30 이벤트는
> `event_time < 14:00` 이라 영원히 누락. MAU 100만 앱에서 일일 0.3% (3,000건) 가 매일 손실되어
> 한 달 누적 9만 건 매출 통계가 빠짐.
> 가능하면 `ingestion_time` 을 delta column 으로 사용하고, event_time 을 써야 한다면
> 별도 **Late Data 패턴**(챕터 3) 결합 필요.

### 2-2. 패턴 #03: 변경 데이터 캡처 (Change Data Capture)

증분 로더는 모든 use case에 맞지 않음. 더 낮은 적재 지연 또는 물리적 삭제 지원이 필요할 때 CDC가 더 나은 선택.

#### 상황 (Problem)

**책의 use case:**
- Incremental Loader로 통합한 레거시 방문 이벤트가 진화해야 함.
- 적재 속도가 너무 느려 다운스트림 컨슈머가 데이터 대기 시간이 너무 길다고 불평.
- product manager가 트랜잭션 레코드를 가능한 빨리 streaming broker로 통합 요청.
- **요구사항**: 적재 작업이 테이블의 각 변경을 **30초 내**에 캡처해 중앙 토픽에서 다른 컨슈머가 사용할 수 있게 함.

#### 해결 (Solution)

지연 요구사항 때문에 Incremental Loader는 사용 불가 — 작업 스케줄링과 쿼리 실행 오버헤드가 있음.

**CDC (Change Data Capture)** 가 더 나은 후보. 내부 적재 메커니즘으로 더 낮은 지연 보장.

CDC 동작:
- **DB의 commit log에서 직접** 모든 수정된 행을 지속적으로 적재
- commit log는 append-only 구조 — 기존 행에 대한 모든 작업을 logfile 끝에 기록
- CDC 컨슈머가 그 변경을 streaming broker 또는 다른 출력으로 보냄
- 컨슈머는 변경 이력 전체를 저장하거나 각 행의 최신 값만 유지하는 등 자유롭게 활용

**CDC가 Incremental Loader 대비 우월한 점:**
- 더 낮은 지연 보장
- **모든 종류의 데이터 작업 가로챔 (hard deletes 포함)**
- → 데이터 producer에게 soft delete를 요청할 필요 없음

```
CDC 데이터 흐름 (Debezium 아키텍처)
--------------------------------------------------------------
[원천 DB]                    [Kafka Connect]            [Kafka 토픽]
┌──────────┐                                            ┌──────────────┐
│ Table A  │                  ┌─────────────┐    ───►  │ Topic Table A│
│ Table B  │  ── commit log ─►│ Debezium    │           ├──────────────┤
│   ...    │                  │ Connector   │    ───►  │ Topic Table B│
└──────────┘                  └─────────────┘           └──────────────┘
--------------------------------------------------------------

각 이벤트는 commit log에서 추출되어 Kafka 토픽으로:
  - INSERT (op='c')
  - UPDATE (op='u', before/after 모두 포함)
  - DELETE (op='d')
```

#### 고려사항 (Consequences)

낮은 지연 약속은 좋지만 다른 엔지니어링 컴포넌트처럼 자체 도전과제 동반.

**Complexity — 복잡성**
- Full Loader/Incremental Loader는 데이터 엔지니어 혼자 구현 가능
- CDC는 **운영팀의 도움 필요** — 예: 서버에서 commit log 활성화

**Data scope — 데이터 범위**
- 클라이언트 구현에 따라 **클라이언트 프로세스 시작 이후의 변경만** 가져올 수 있음
- 이전 변경에 관심이 있다면 다른 적재 패턴과 결합 필요

**Payload — 페이로드**
- CDC는 레코드와 함께 추가 메타데이터를 가져옴 — operation type, modification time, column type
- 컨슈머는 무관한 속성을 무시하도록 처리 로직 적응 필요

**Data semantics — 데이터 의미론**
- CDC는 **data at rest** 를 적재하지만 부수효과로 그 정적 행이 **data in motion** 이 됨
- data in motion 은 다른 처리 의미를 가짐
- 예: JOIN 작업 — 정적 테이블에서 매칭이 없으면 "매칭 데이터가 없음". 두 동적 스트리밍 소스에서 매칭이 없으면 "데이터가 **아직** 없음" — 한 스트림이 다른 스트림보다 단순히 늦었을 수 있음
- → CDC 컨슈머의 데이터를 정적 데이터로 간주하면 안 됨

#### 구현 예시 (Examples)

**예시 1 — Debezium Kafka Connect for PostgreSQL:**
```json
{
  "name": "visits-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "postgres", "database.port": "5432",
    "database.user": "postgres", "database.password": "postgres",
    "database.dbname": "postgres", "database.server.name": "dbserver1",
    "schema.include.list": "dedp_schema",
    "topic.prefix": "dedp"
  }
}
```
→ `dedp_schema.events` 테이블의 모든 변경이 `dedp.dedp_schema.events` 토픽에 기록됨.
PostgreSQL 측 사전 준비: `pgoutput` 플러그인으로 logical replication stream 활성화 + 권한 사용자.

**예시 2 — Delta Lake Change Data Feed (CDF):**

Delta Lake에는 native CDC 기능 — `readChangeFeed`. 글로벌 세션 속성 또는 테이블 속성으로 활성화:
```python
# 방법 1: 세션 속성
spark_session_builder.config(
    'spark.databricks.delta.properties.defaults.enableChangeDataFeed', 'true'
)

# 방법 2: 테이블 속성
spark_session.sql('''
    CREATE TABLE events (
        visit_id STRING, event_time TIMESTAMP, user_id STRING, page STRING
    )
    TBLPROPERTIES (delta.enableChangeDataFeed = true)
''')
```

CDF 사용 — 처음부터 4 파일씩:
```python
events = (spark_session.readStream.format('delta')
    .option('maxFilesPerTrigger', 4)
    .option('readChangeFeed', 'true')
    .option('startingVersion', 0)
    .table('events'))
query = events.writeStream.format('console').start()
```

CDF 출력 테이블에는 추가 컬럼:
```
+----------+-----------+--------------+---------------+-------------------+
| visit_id | event_time| _change_type | _commit_version | _commit_timestamp |
+----------+-----------+--------------+---------------+-------------------+
| 14008..0 | 23-11-24  | insert       | 6             | 2023-12-03 13:28  |
| 14008..1 | 23-11-24  | insert       | 6             | 2023-12-03 13:28  |
+----------+-----------+--------------+---------------+-------------------+

UPDATE 적용 시: update_preimage / update_postimage 두 행으로 나타남
```

| 항목 | 증분 로더 (#02) | CDC (#03) |
|------|----------------|-----------|
| 삭제 감지 | 불가 (soft delete 필요) | 가능 (hard delete 자동 캡처) |
| 지연 | 작업 스케줄 주기에 의존 | 초 단위 |
| 운영 복잡도 | 낮음 (DE 단독 구현) | 높음 (Ops팀 협업 필요) |
| 페이로드 | 비즈니스 컬럼만 | 메타데이터 (op, timestamp, version) 포함 |
| JOIN 의미 | 정적 데이터 의미 | 동적 데이터 의미 (지연 매칭 가능) |

> **트러블 로그** — CDC 컨슈머에서 두 토픽을 JOIN 할 때 정적 데이터처럼 inner join 을 쓰면
> "아직 도착 안 한" 데이터가 매칭 실패로 영원히 누락됨.
> 예: `orders` 토픽과 `payments` 토픽을 inner join 했는데, payment 가 order 보다 평균 200ms 늦게 commit log 에 기록되어
> 결과 토픽에 매일 ~5% 의 주문이 영원히 빠짐. 일 매출 100억 환경에서 5억이 사라지는 사고.
> CDC 데이터에는 **stream-stream join + watermark + 적절한 join window** 를 반드시 사용하고,
> 그래도 매칭 못 한 한쪽은 별도 토픽에 적재해 추후 재시도하는 구조로 가야 함.

---

## 3. 복제 (Replication)

복제는 데이터 수집 패턴의 또 다른 계열. 주 목표는 데이터를 한 위치에서 다른 위치로 **있는 그대로** 복사.
다만 현실에서는 규제 준수 등으로 입력을 변경해야 하는 경우가 많음.

> **Data Loading vs Replication**
> 두 개념은 비슷해 보이지만 차이가 있음.
> 복제는 **같은 종류의 저장소** 사이에 데이터를 옮기고 모든 메타데이터 속성(예: DB의 PK, streaming broker의 이벤트 위치)을 보존하는 것이 이상적.
> 적재는 더 유연하고 이런 동질 환경 제약이 없음.

### 3-1. 패턴 #04: 패스스루 복제기 (Passthrough Replicator)

데이터 적재처럼, 복제 영역에도 결과를 받아들일 수 있다면 기본으로 사용할 수 있는 **패스스루 모드** 가 있음.

#### 상황 (Problem)

**책의 use case:**
- 배포 프로세스가 3개 환경 — development, staging, production.
- 많은 작업이 외부 API에서 매일 production 에 적재되는 **디바이스 파라미터 참조 데이터셋** 사용.
- 더 나은 개발 경험과 버그 탐지를 위해 이 데이터셋을 나머지 환경에도 두고 싶음.
- **결정적 제약**: 참조 데이터셋 적재 프로세스가 third-party API를 사용하고 **idempotent 하지 않음** — 같은 API 호출이 하루 중 다른 결과를 반환할 수 있음.
- → dev/staging 에서 적재 파이프라인을 단순 복사/재실행할 수 없음. **production 과 동일한 데이터** 가 필요.

#### 해결 (Solution)

idempotent 하지 않은 데이터 제공자 + 환경 간 일관성 요구 → Passthrough Replicator 패턴의 좋은 이유.
컴퓨트 레벨 또는 인프라 레벨로 설정 가능.

**컴퓨트 구현**: EL job — read+write 두 단계만. 이상적으로 입력에서 그대로 복사 (변환 없음).
- 변환을 넣으면 데이터 품질 이슈 발생 가능 (string→date 타입 변환, float 반올림 등)

**인프라 구현**: **replication policy** 문서로 입력/출력 위치 설정 후 데이터 저장소가 알아서 복제 수행.

#### 고려사항 (Consequences)

핵심 학습은 구현을 단순하게 유지하는 것. 단순한 구현조차 해결할 도전과제가 있음.

**Keep it simple — 단순함 유지**
- 데이터를 있는 그대로 필요로 함 — 가능한 가장 단순한 복제 작업, 이상적으로는 DB의 data copy 명령
- 명령이 없고 텍스트 형식(JSON 등)에 데이터 처리 프레임워크를 써야 한다면 **JSON I/O API 대신 raw text API** 사용 (선해석 없이 라인 그대로 복사)
- 같은 파일 수, 같은 파일명이 중요하다면 그 파라미터를 커스터마이즈 못 하는 분산 처리 프레임워크는 피할 것

**Security and isolation — 보안과 격리**
- 환경 간 통신은 항상 까다롭고 복제 작업에 버그가 있으면 오류 발생 가능
- 타겟 환경에 부정적 영향을 줘 불안정하게 만들 위험
- → **pull 대신 push** 접근 사용 — 데이터셋 소유 환경이 다른 환경으로 복사, 빈도와 throughput 제어
- 그래도 이슈 가능 — 클라우드 서브넷에서 마지막 IP 주소를 사용해 다른 작업이 진행 못 하는 경우 등

**PII data — 개인 식별 정보**
- 복제 데이터셋이 PII 또는 production 환경에서 propagate 되면 안 되는 정보를 저장한다면 → **Transformation Replicator 패턴 (#05)** 사용

**Latency — 지연**
- 인프라 기반 구현은 종종 추가 지연이 있음 — 클라우드 제공자의 SLA 확인 필요

**Metadata — 메타데이터**
- 메타데이터 부분 무시하면 복제 데이터셋이 사용 불가능해질 수 있음
- 예: Delta Lake 테이블의 Apache Parquet 파일만 복제하면 부족 (트랜잭션 로그도 필요)
- Apache Kafka — 키와 값뿐 아니라 **헤더와 파티션 내 이벤트 순서** 도 신경

#### 구현 예시 (Examples)

**예시 1 — Apache Spark JSON 복제 (raw text API):**
```python
# JSON I/O API 대신 raw text — 데이터 자체에 어떤 간섭도 없이 라인 단위 복사
input_dataset = spark_session.read.text(f'{base_dir}/input/date=2023-11-01')
input_dataset.write.mode('overwrite').text(f'{base_dir}/output-raw/date=2023-11-01')
```
→ 주의: 이 스니펫은 파일 수를 보존하지 않음 (소스/목적지 파일 수가 다를 수 있음, 데이터는 동일).

**예시 2 — Kafka 토픽 복제 with ordering semantic:**
```python
events_to_replicate = (input_data_stream
    .selectExpr('key', 'value', 'partition', 'headers', 'offset'))

def write_sorted_events(events: DataFrame, batch_number: int):
    (events.sortWithinPartitions('offset', ascending=True).drop('offset').write
        .format('kafka').option('kafka.bootstrap.servers', 'localhost:9094')
        .option('topic', 'events-replicated').option('includeHeaders', 'true').save())

write_data_stream = (events_to_replicate.writeStream
    .option('checkpointLocation', f'{get_base_dir()}/checkpoint-kafka-replicator')
    .foreachBatch(write_sorted_events))
```
→ `sortWithinPartitions('offset')` 와 `includeHeaders` 가 핵심 — 메타데이터와 순서 보존.

**예시 3 — AWS S3 bucket replication (Terraform, 인프라 기반):**
```hcl
resource "aws_s3_bucket_replication_configuration" "replication" {
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.devices_production.id

  rule {
    id     = "devices"
    status = "Enabled"
    destination {
      bucket        = aws_s3_bucket.devices_staging.arn
      storage_class = "STANDARD"
    }
  }
}
```
→ Kafka 토픽 복제는 MirrorMaker 유틸리티 활용.

> **트러블 로그** — Delta Lake 테이블을 "그냥 Parquet 파일들이니까" 라며 `aws s3 sync` 로 복제하면
> 타겟에서 테이블 자체가 깨짐. `_delta_log/` 디렉터리 (트랜잭션 로그)를 빠뜨리거나 timing 이 어긋나면
> Delta reader 가 "이 버전에 해당하는 파일이 없음" 또는 "트랜잭션 충돌" 오류로 read 자체 실패.
> 예: 1TB 테이블을 dev 환경에 복제하면서 Parquet 파일은 모두 옮겼지만 `_delta_log/` 만 빠져
> dev 팀 전체가 3일간 테스트 못 한 사례. Delta/Iceberg/Hudi 같은 트랜잭션 형식은 반드시 native 복제 도구
> (Delta CLONE, S3 Replication 등) 사용 — 메타데이터 일관성을 보장.

### 3-2. 패턴 #05: 변환 복제기 (Transformation Replicator)

Kafka 예시도 복잡해 보였지만, 복제 시나리오가 더 많은 코드 노력을 요구할 수 있음 — Transformation Replicator를 사용할 때.

#### 상황 (Problem)

**책의 use case:**
- 새 버전의 데이터 처리 작업을 release 하기 전에 production 의 실제 데이터로 테스트하고 싶음 (synthetic data로는 데이터 품질 이슈 시뮬레이션 불가).
- production 데이터를 staging 환경에 복제해야 함.
- **결정적 제약**: 복제할 데이터셋에 **PII** 가 포함되어 있어 production 외부에서 접근 불가.
- → 단순한 Passthrough Replicator 작업 사용 불가.

#### 해결 (Solution)

데이터 시스템 테스트의 큰 문제는 데이터 자체. 데이터 제공자가 일관된 스키마와 값을 보장 못 하면 production 데이터 사용이 불가피.
하지만 production 데이터는 다른 환경으로 옮길 수 없는 민감 속성을 자주 가짐.

**Transformation Replicator 패턴** — Passthrough 의 read/write 에 더해 그 사이에 **transformation layer** 를 둠.

Transformation 구현 옵션:
- 데이터 처리 프레임워크(Spark, Flink) 사용 시 **custom mapping function**
- SQL로 표현 가능하면 **SQL SELECT 문**

Transformation 내용:
- 복제하면 안 되는 속성을 **대체** (예: **Anonymizer 패턴**)
- 또는 처리에 불필요하면 **제거**

#### 고려사항 (Consequences)

custom logic을 작성하므로 데이터셋이 깨질 위험이 Passthrough 보다 높음.

**Transformation risk for text file formats — 텍스트 파일 형식의 변환 위험**
- 예: JSON/CSV 위에 변환 — datetime 형식이 데이터 처리 프레임워크 표준과 다른 경우 발생
- 결과: 복제된 데이터셋에 timestamp 컬럼이 모두 포함되지 않고 staging 작업 실패
- → "keep it simple" 접근 — timestamp 컬럼을 있는 그대로 두지 말고 **string 으로 설정** 해서 silent transformation 회피

**Desynchronization — 비동기화**
- 데이터가 지속 진화 — 오늘의 privacy 필드가 미래에도 유효한 보장 없음
- 새 PII가 등장하거나 현재 PII 가 아닌 속성이 PII 로 재분류될 수 있음
- 대응: **data governance tool** (data catalog, data contract) — 민감 필드를 태깅. 자동화 가능.

#### 구현 예시 (Examples)

**예시 1 — Data reduction (EXCEPT operator):**

Databricks/BigQuery 등이 `EXCEPT` 지원:
```sql
-- ip, latitude, longitude 제외한 모든 컬럼 선택
SELECT * EXCEPT (ip, latitude, longitude)
```

**예시 2 — PySpark drop:**
```python
input_delta_dataset = spark_session.read.format('delta').load(users_table_path)
users_no_pii = input_delta_dataset.drop('ip', 'latitude', 'longitude')
```

**예시 3 — Column-level access (AWS Redshift):**
```sql
-- user_a 가 visits 테이블에서 visit_id, event_time, user_id 만 SELECT 가능
GRANT SELECT (visit_id, event_time, user_id) ON TABLE visits TO user_a;
```
→ EXCEPT 기반보다 verbose 하지만 추가 보호 레이어 (Fine-Grained Accessor 패턴)

**예시 4 — Column-based transformation:**
```python
# full_name 의 첫 글자 마스킹
devices_trunc_full_name = (input_delta_dataset
    .withColumn('full_name',
        functions.expr('SUBSTRING(full_name, 2, LENGTH(full_name))')))
```

**예시 5 — Mapping function (Scala Spark, 강타입):**
```scala
case class Device(`type`: String, full_name: String, version: String) {
  lazy val transformed = {
    if (version.startsWith("1.")) {
      this.copy(full_name = full_name.substring(1), version = "invalid")
    } else { this }
  }
}
inputDataset.as[Device].map(device => device.transformed)
```

> **트러블 로그** — Transformation Replicator 에서 timestamp 컬럼을 `to_timestamp()` 로 명시적 변환하면
> 한 milliseconds 자릿수 차이로 silent fail 발생.
> 예: production 의 ISO 8601 timestamp 가 `2026-05-09T14:23:01.234Z` 인데
> Spark 의 to_timestamp 기본 패턴은 milliseconds 처리 안 해서 NULL 로 변환됨.
> 결과: staging 데이터 30%가 timestamp NULL → 다운스트림 partitioning 실패 →
> staging 작업이 production 배포 전에 통과해도 prod 에서 첫 batch 부터 실패하는 사고.
> 책 권장대로 텍스트 형식 PII 마스킹 시에는 timestamp 등 민감하지 않은 컬럼은 **string 그대로 두고**
> 다운스트림에서 변환할 것.

---

## 4. 데이터 컴팩션 (Data Compaction)

완벽한 데이터셋도 시간이 지나면 새 데이터로 인해 병목이 될 수 있음.
어느 시점에는 listing files 같은 메타데이터 작업이 데이터 처리 변환보다 더 오래 걸리게 됨.

### 4-1. 패턴 #06: 컴팩터 (Compactor)

성장하는 데이터셋 이슈를 다루는 가장 쉬운 방법은 underlying 파일의 storage footprint 줄이기.

#### 상황 (Problem)

**책의 use case:**
- 실시간 데이터 적재 파이프라인이 streaming broker에서 object store로 이벤트를 동기화.
- 주 목표: 최대 10분 내에 batch jobs가 데이터를 사용할 수 있게 함.
- 단순 패스스루 작업이라 파이프라인은 문제 없이 실행 중.
- 그러나 **3개월 후 모든 batch jobs가 metadata overhead 문제로 고통** 받기 시작:
  - 실행 시간의 **70% 를 listing files** 에 사용
  - 나머지 30% 만 실제 데이터 처리에 사용
  - pay-as-you-go 서비스 사용 시 심각한 latency/cost 영향

#### 해결 (Solution)

작은 파일이 많은 것은 잘 알려진 문제 — Hadoop 시대부터 존재, 현대 object store 기반 lakehouse 에도 여전히 있음.
많은 작은 파일 저장 → 더 긴 listing 작업 + 파일 열고 닫는 더 무거운 I/O.

자연스러운 해결책: **더 적은 파일 저장**.

**Compactor 패턴**: 여러 작은 파일을 더 큰 파일로 결합 → 읽기의 전체 I/O 오버헤드 감소.

기술별 구현:
- **Apache Iceberg**: `rewrite_data_files` action — 트랜잭션 분산 처리 작업이 새 commit의 일부로 작은 파일을 더 큰 파일로 병합
- **Delta Lake**: `OPTIMIZE` 명령
- **Apache Hudi**: **MoR (merge-on-read)** 테이블로 설정 — 데이터셋이 columnar 형식으로 쓰이고 후속 변경은 row 형식으로 쓰임. 읽기 최적화를 위해 row storage 의 변경을 columnar storage 와 병합.
  - Delta/Iceberg는 homogeneous columnar 형식만 — 이 점이 다름

**Lake 외 데이터 저장소**:
- **Apache Kafka** (append-only key-based logs system): configuration 기반 구현 — compaction frequency 설정만, 데이터 저장소가 알아서 관리
- key-based 시스템에서는 컴팩션이 **주어진 record key의 가장 최근 entry만 유지** 하는 형태로 storage 최적화. 이 경우 현재 데이터를 덮어씀 (table file format 의 컴팩션과는 다름).

```
Compactor 효과
--------------------------------------------------------------
[컴팩션 전 — 3개월치 5초 트리거 스트리밍]
  /events/dt=.../
    ├─ part-00001.parquet  (1MB)
    ├─ part-00002.parquet  (2MB)
    ├─ ...
    └─ part-XXXXX.parquet  (1MB)
  파일 수: 수만 개

  Batch job 실행 시간:
    - listing files : 70%
    - 실제 처리     : 30%
            │
            ▼ OPTIMIZE / rewrite_data_files
            │
[컴팩션 후]
  /events/dt=.../
    ├─ part-00001.parquet  (256MB)
    └─ part-00002.parquet  (256MB)
  파일 수: 수십 개

  Batch job 실행 시간:
    - listing files : 5%
    - 실제 처리     : 95%
--------------------------------------------------------------
```

#### 고려사항 (Consequences)

겉보기엔 무해하고 많은 데이터 저장소에서 native 지원하지만, 상당한 설계 노력이 필요할 수 있음.

**Cost vs performance trade-offs — 비용 대 성능 트레이드오프**
- 컴팩션 작업은 큰 테이블에서 컴퓨트 집약적
- 비용만 보면 드물게 — 하루 한 번, 이상적으로 근무 시간 외, 데이터셋 생성 파이프라인 외부 — 실행
- 그러나 드문 실행은 아직 컴팩션 안 된 데이터에 작용하는 작업에 문제 — 최적화 혜택을 못 받음
- one-size-fits-all 해결책 없음. 비용/성능 양쪽 perspective 에서 완벽하지 않을 수 있음을 받아들여야 함.
- 때로는 컴팩션을 **데이터 적재 프로세스에 포함** 시키는 게 나음 — 적재 throughput 을 penalize 하지만 컨슈머에 더 큰 영향을 주지 않음.

**Consistency — 일관성**
- 컴팩션은 이미 존재하는 데이터를 단순 재작성 — 컨슈머가 사용할 데이터와 컴팩션 중인 데이터를 구분하기 어려움
- → **ACID 형식 (Delta Lake, Apache Iceberg) 에서 구현이 훨씬 단순하고 안전** (raw 형식 JSON/CSV 보다)

**Cleaning — 정리**
- 컴팩션 작업은 source 파일을 **보존** 할 수 있음 → 작은 파일이 여전히 남아 metadata 작업에 영향
- 컴팩션만으로는 부족 — 이미 컴팩션된 파일이 차지한 공간을 회수하는 **cleaning job** 으로 보완
- `VACUUM` 같은 명령 사용 (Delta Lake, Apache Iceberg, PostgreSQL, Redshift)
- 단, cleaning 전략은 신중히 선택 — 삭제된 컴팩션 파일 기준으로 데이터셋의 그 버전으로 복구 못 할 수 있음

#### 구현 예시 (Examples)

**예시 1 — Delta Lake 컴팩션 + cleaning:**
```python
# 컴팩션: 작업 시작 시 사용 가능한 파일들만 재구성 — reader/writer 와 충돌 없음
devices_table = DeltaTable.forPath(spark_session, table_dir)
devices_table.optimize().executeCompaction()
```

```python
# Cleaning: 보존 임계값보다 오래된 파일만 적용 — 진행 중 쓰기 파일 보호
devices_table = DeltaTable.forPath(spark_session, table_dir)
devices_table.vacuum()
```

**예시 2 — Apache Kafka log compaction:**
```properties
# 토픽 생성 시 log compaction 설정
log.cleanup.policy = compact          # 컴팩션 전략
log.cleaner.min.compaction.lag.ms = ...
log.cleaner.max.compaction.lag.ms = ...   # 컴팩션 빈도
```
→ **주의**: Delta Lake와 달리 Kafka 컴팩션은 **비결정적** — 정기 스케줄을 기대할 수 없고, 항상 각 키당 unique 레코드를 보장하지 않음.

| 데이터 저장소 | 컴팩션 명령 | Cleaning 명령 | 동시성 안전성 |
|--------------|-----------|--------------|--------------|
| Delta Lake | `OPTIMIZE` / `optimize().executeCompaction()` | `VACUUM` / `vacuum()` | 안전 (ACID) |
| Apache Iceberg | `rewrite_data_files` action | expire snapshots + remove orphan | 안전 (ACID) |
| Apache Hudi | MoR auto-compaction (configuration) | clean service | 안전 |
| Apache Kafka | `log.cleanup.policy=compact` | 자동 | 비결정적 |
| PostgreSQL | `VACUUM` | `VACUUM FULL` | 안전 |

> **트러블 로그** — Delta `VACUUM` 의 기본 retention 7일을 모르고 더 짧게 줄이면 데이터 손실 위험.
> 예: 운영팀이 스토리지 비용 줄이려고 `VACUUM RETAIN 1 HOURS` 를 야간 cron 에 걸었더니
> 같은 시간대 진행 중이던 1시간짜리 long-running ETL 작업의 입력 파일이 컴팩션 후 vacuum 으로 삭제되어
> 작업이 "FileNotFoundException" 으로 실패. 그 작업의 idempotent 재실행도 source 파일이 사라져 불가능.
> 책 권장대로 retention 단축은 신중히 — 진행 중인 long-running 쿼리/스트리밍 의 최대 실행 시간 + 충분한 buffer 가 필요.
> Delta 는 기본 보호 장치(retentionDurationCheck)를 끄지 않으면 168시간(7일) 미만으로 못 줄이게 막는데, 이걸 강제로 끄지 말 것.

---

## 5. 데이터 준비 (Data Readiness)

지금까지 다룬 다른 데이터 적재 의미론만이 이 작업에서 마주칠 수 있는 문제는 아님.
다음 문제 질문은 "**언제 적재 프로세스를 시작해야 하는가?**"

### 5-1. 패턴 #07: 준비 마커 (Readiness Marker)

가장 적절한 순간에 적재 프로세스를 트리거하도록 돕는 패턴.
목표: **완전한 데이터셋의 적재를 보장**.

#### 상황 (Problem)

**책의 use case:**
- 매시간 batch job이 Medallion 아키텍처의 Silver 레이어에 데이터 준비.
- 데이터셋은 모든 알려진 데이터 품질 이슈가 수정되고 사용자 DB와 외부 API 서비스에서 로드된 추가 컨텍스트로 보강됨.
- 다른 팀들이 ML 모델과 BI 대시보드 생성에 이 데이터셋을 사용.
- **문제**: 다른 팀들이 종종 **불완전한 데이터셋**에 대해 불평. 컨슈밍을 시작해도 되는 시점을 직간접적으로 알리는 메커니즘 구현 요청.

#### 해결 (Solution)

이 이슈는 논리적으로 의존하지만 물리적으로 격리된 (다른 팀이 유지하는) 파이프라인에서 특히 두드러짐.
격리된 워크로드 때문에 작업이 다운스트림 파이프라인을 직접 트리거할 수 없음.
대신 **Readiness Marker 패턴**으로 데이터셋을 처리 준비 완료로 표시.

구현은 파일 형식과 스토리지 조직에 따라 다름.

**구현 1: Flag file 기반**
- object store / 분산 파일 시스템 인기 덕에 가장 쉬운 구현
- 성공적인 데이터 생성 후 만들어지는 **flag file**
- 데이터 처리 레이어에서 native 지원: Apache Spark는 raw file 형식에 `_SUCCESS` 자동 작성. Delta Lake 는 새 commit log.
- native 지원이 없으면 데이터 오케스트레이션 레이어에서 별도 task로 구현

**구현 2: Convention 기반 (partitioned data sources)**
- 시간 기반 테이블/위치를 생성한다면 Readiness Marker 가 conventional 가능
- 예: 작업이 매시간 실행되어 hourly partition에 데이터 작성 — 10:00 실행은 partition 10에, 11:00 실행은 partition 11에
- 컨슈머가 partition 10을 처리하려면 작업이 partition 11에 작업 중일 때까지 단순 대기

#### 고려사항 (Consequences)

Readiness Marker는 reader가 데이터 가져오기를 제어하는 **pull 접근** 의존. 구현이 암묵적이라 주의점이 있음.

**Lack of enforcement — 강제 부재**
- flag file이나 다음 partition 감지 기반의 conventional readiness를 강제할 쉬운 방법 없음
- 어느 쪽이든 컨슈머가 데이터 생성 중에 컨슘 시작할 수 있음
- 이 암묵성 때문에 **컨슈머와 소통**, 컨슈머 측에서 처리를 트리거할 조건에 동의가 매우 중요
- 추가로, readiness convention 미준수 위험을 명확히 설명해야 함

**Reliability for late data — 늦은 데이터에 대한 신뢰성**
- partition이 event time 기반이면 partition-based 구현은 늦은 데이터 이슈 겪음
- 예: 8, 9, 10 partition을 닫음 → 컨슈머가 모두 처리 → 9시 partition에 늦은 데이터 감지 → 9시 partition에 통합 → 그러나 컨슈머가 그 partition 닫힘으로 간주해 동일 처리 못 함
- 대응: partition을 **immutable한 닫힌 부분**으로 간주하거나, mutability 조건을 명확히 정의/공유
- 컨슈머에 partition 업데이트 알림 + 처리할 새 데이터 알림 — **Late Data 패턴** (챕터 3) 사용

#### 구현 예시 (Examples)

**예시 1 — PySpark가 자동 생성하는 `_SUCCESS`:**
```python
# raw file 형식에서는 _SUCCESS 자동 생성
dataset = (spark_session.read.schema('...').json(f'{base_dir}/input'))
dataset.write.mode('overwrite').format('parquet').save('devices-parquet')
# → devices-parquet/_SUCCESS 자동 생성됨 (모든 task 성공 후)
```

**예시 2 — Airflow FileSensor 가 `_SUCCESS` 대기:**
```python
FileSensor(
    filepath=f'{input_data_file_path}/_SUCCESS',
    mode='reschedule'        # worker 슬롯 점유 안 함 (sleep + reschedule)
)
```
→ `mode='reschedule'` 이 핵심 — sensor task 가 worker 를 잡고 있지 않고, 깨어나서 확인 후 다시 자러 감.

**예시 3 — Airflow @task로 별도 readiness file 생성:**

데이터 처리 작업에서 native 가 안 될 때 오케스트레이션 레이어에서 직접 생성:
```python
@task
def delete_dataset():
    shutil.rmtree(dataset_dir, ignore_errors=True)

@task
def generate_dataset():
    pass    # 처리 부분 (생략)

@task
def create_readiness_file():
    with open(f'{dataset_dir}/COMPLETED', 'w') as marker_file:
        marker_file.write('')

# readiness marker 는 항상 파이프라인의 마지막 단계 — 마지막 변환 후
delete_dataset() >> generate_dataset() >> create_readiness_file()
```

> **트러블 로그** — `_SUCCESS` 만 보고 다운스트림이 곧장 처리하면 빈 파티션 사고가 남.
> 예: 업스트림이 그날 데이터가 0건이어도 정상 종료하면 빈 디렉터리 + `_SUCCESS` 가 생성됨.
> 다운스트림이 "정상 적재" 로 인식하고 그날 통계 테이블에 0을 INSERT — 일일 매출 대시보드가
> 갑자기 0원으로 떨어져 경영진 알람이 울리는 사례가 흔함.
> 책의 권장사항대로 컨슈머와 readiness 의 의미를 명확히 합의하고,
> readiness file 안에 `row_count` 같은 메타데이터를 함께 적어 다운스트림이 임계값 검증 (예: 평균의 30% 미만이면 실패) 할 수 있게 할 것.

---

## 6. 이벤트 주도 (Event Driven)

이상적인 데이터 적재 시나리오에서는 새 데이터셋이 정기 스케줄에 사용 가능 — Readiness Marker 사용 가능.
이상보다 못한 시나리오에서는 **incoming frequency 예측이 어려움** — 정적 적재에서 **이벤트 주도 적재** 로 mindset 전환 필요.

### 6-1. 패턴 #08: 외부 트리거 (External Trigger)

이전 섹션의 Readiness Marker 패턴은 **pull semantics** (컨슈머가 새 데이터 확인 책임).
이벤트 주도 데이터셋의 본질은 **push semantics** 선호 (producer 가 데이터 가용성을 컨슈머에 알림).

#### 상황 (Problem)

**책의 use case:**
- 조직의 백엔드 팀이 일주일에 최대 한 번 (월~목 사이) 새 features를 release.
- 각 release 가 **참조 데이터셋** 보강 — 웹사이트의 모든 features 를 어느 시점이든 보관.
- 지금까지 이 데이터셋의 refresh job은 **하루 한 번** 스케줄 — 변경 없을 때도 reload 해서 컴퓨트 자원 낭비.
- 비용 절감 목표로 스케줄링 메커니즘 변경 — **새로 처리할 게 있을 때만** 파이프라인 실행.
- 백엔드 팀이 새 feature publish 할 때마다 **중앙 message bus 에 notification event** 보냄.

#### 해결 (Solution)

예측 불가한 데이터 생성은 종종 데이터의 이벤트 주도 본질 때문.
시간 스케줄 작업으로 전체 데이터셋 복사하거나 자주 확인할 수도 있지만, 컴퓨트 자원 낭비 + 불필요한 운영 부담.
더 나은 접근 — **이벤트 주도 External Trigger 패턴**.

> **Not Only Trigger**: 트리거할 작업이 없다면 적재 프로세스를 notification handler 에서 직접 실행 가능.
> 그래서 **이벤트 주도 데이터 적재** 라고 부름.

패턴은 3개의 주요 액션으로 구성:

```
External Trigger 3단계
--------------------------------------------------------------

[1. 알림 채널 구독]
   외부 (이벤트 주도 producer) ↔ 내 파이프라인 사이 연결 설정
                  │
                  ▼
[2. 알림에 반응]
   Events handler 가 이벤트 분석
   → 데이터 오케스트레이션/처리 레이어에서 파이프라인 트리거할지 결정
   → 메시지 버스가 지원하면 특정 이벤트만 구독 (예: 특정 테이블의 데이터 생성)
   → 필터링 미지원이면 handler 가 직접 구현
                  │
                  ▼
[3. 적재 파이프라인 트리거]
   Handler 가 데이터 오케스트레이션 워크플로우 트리거
   또는 작업 직접 트리거
   → 기본: 한 이벤트 → 한 파이프라인. 동일 데이터셋이 여러 워크로드의 입력이면 여러 개 트리거 가능
--------------------------------------------------------------

전체 그림:
   ┌──────────────┐
   │  Messages    │
   │   (📧 📧)     │
   └──────┬───────┘
          │
          ▼
   ┌──────────────┐    Trigger for event              ┌─────────────────┐
   │ Notification │───►"Write data to table orders"──►│ Data processing │
   │   handler    │    Trigger for event              └─────────────────┘
   └──────┬───────┘    "Write data to table visits"
          │                    │
          ▼                    ▼
   ┌──────────────────────────────┐
   │ Data orchestration           │
   │  ┌──┐  ┌──┐  ┌──┐            │
   │  │  │─►│  │─►│  │ Event-driven job
   │  └──┘  └──┘  └──┘            │
   └──────────────────────────────┘
```

#### 고려사항 (Consequences)

이벤트 주도 특성이 자원 낭비 줄여 매력적이지만 데이터 스택에 영향이 있음.

**Push vs pull**
- External Trigger 는 pull 또는 push 의미론 구현 가능. 차이가 시스템 영향 이해의 핵심.
- **pull-based**: 새 이벤트 있는지 짧은 정기 간격으로 지속 확인. 기술적으로 유효하지만, 작업이 대부분 시간을 zero message pulling 에 써서 최적화 안 됨.
- **push-based**: 데이터 소스가 endpoint(s) 에 새 메시지를 알림. 각 알림이 새 컨슈머 인스턴스 시작 → 이벤트 반응 후 종료. **더 나은 대안**.

**Execution context — 실행 컨텍스트**
- 외부 트리거가 **단순 ping 메커니즘** 이 될 위험 — 데이터 오케스트레이터 endpoint 만 호출
- 매력적이지만, 트리거되는 적재 파이프라인은 다른 파이프라인처럼 유지 필요
- 단순 트리거만 하면 **왜 트리거되었는지, 무엇을 처리해야 하는지** 충분한 컨텍스트 부족
- → 트리거 호출에 **적절한 메타데이터 정보** 풍부히 — trigger job 버전, notification envelope, processing time, event time
- 일상 모니터링과 실패 원인 조사 시 유용

**Error management — 오류 관리**
- 이벤트가 핵심 요소 — 없으면 작업 트리거 불가
- 어떤 일이 있어도 이벤트를 보존하는 목표로 **실패 대비 트리거 설계**
- **Dead-Letter 패턴** (다음 챕터) 같은 패턴 활용

#### 구현 예시 (Examples)

클라우드(AWS, Azure, GCP)는 serverless function 서비스를 제공해 이벤트 주도 능력 활성화 — 트리거의 좋은 후보.
대부분의 데이터 오케스트레이터는 파이프라인 시작에 사용할 API 노출.

**예시 — AWS Lambda 함수가 Apache Airflow 연결:**

```python
# 외부 트리거되는 DAG 정의 — schedule_interval=None 이 핵심
with DAG('devices-loader', max_active_runs=5, schedule_interval=None,
         default_args={'depends_on_past': False}) as dag:
    # 파이프라인은 트리거에서 파일을 단순 복사 (생략)
    pass
```
→ `schedule_interval=None` 이라 Airflow 스케줄러가 plan 단계에서 무시 — **명시적 trigger action** 으로만 실행.

```python
# AWS Lambda handler — 모니터링 중인 S3 버킷에 새 객체 등장 시 실행
def lambda_handler(event, ctx):
    payload = {
        'event': json.dumps(event),
        'trigger': {
            'function_name': ctx.function_name,
            'function_version': ctx.function_version,
            'lambda_request_id': ctx.aws_request_id,
        },
        'file_to_load': urllib.parse.unquote_plus(
            event['Records'][0]['s3']['object']['key'], encoding='utf-8'
        ),
        'dag_run_id': f'External-{ctx.aws_request_id}',
    }
    trigger_response = requests.post(
        'http://localhost:8080/api/v1/dags/devices-loader/dagRuns',
        data=json.dumps({'conf': payload}),
        auth=('dedp', 'dedp'),
        headers=headers,
    )
    if trigger_response.status_code != 200:
        raise Exception(f"Couldn't trigger devices-loader DAG. {trigger_response}")
    return True
```
→ **Resiliency**: AWS Lambda 가 인프라 레벨에서 **failed-event destinations** 로 처리.
실패한 호출 레코드를 별도 destination 에 보냄. batch size, concurrency level 도 설정 가능.

> **Explicitness of Code Snippets**
> 책의 코드는 가독성 위해 hardcoded credentials 사용. 일반 규칙으로 코드에 credentials 직접 hardcode 는 나쁜 관행 — 쉽게 leak.
> 완화책: **Data Security 디자인 패턴 (챕터 7)** 사용.

> **트러블 로그** — S3 → Lambda → Airflow 트리거에서 Lambda 가 가끔 같은 이벤트로 2번 호출되는 일이 발생.
> S3 이벤트는 **at-least-once** 보장이라 동일 객체에 대해 중복 트리거 가능.
> 예: 같은 reference 데이터셋 적재가 2번 실행되어 Silver 레이어 테이블에 동일 데이터가 두 번 INSERT,
> 다운스트림 KPI 가 정확히 2배로 표시되어 마케팅 팀이 "캠페인 효과 200% 증가" 로 잘못 해석하고
> 광고 예산 2배 집행한 사례.
> 책 권장대로 trigger payload 의 **`lambda_request_id`** (또는 `dag_run_id`) 를 idempotency key 로 활용해
> Airflow DAG 시작 시 "이 request_id 가 이미 처리되었는가" 를 체크하는 deduplication 단계를 둘 것.

---

## 7. 요약

데이터 수집은 기술적으로 도전적이지 않다고 여겨지지만, 이번 챕터는 그 반대를 증명함.
한 곳에서 다른 곳으로 데이터를 옮기는 단순한 작업조차 도전과제가 있음:
- **Readiness Marker** 없으면 컨슈머로서 불완전한 데이터 적재 또는 producer 로서 사용자 사이 나쁜 평판
- **Compactor** 없으면 사실상 무제한 lakehouse도 API 호출 때문에 빠르게 성능 병목

이번 챕터의 패턴 대부분은 ETL/ELT 파이프라인의 **extract step** 의 좋은 후보 — 더 비즈니스 지향 작업에서도 사용 가능.

### 7-1. 8개 패턴 한눈 비교

| # | 패턴 | 카테고리 | 책의 use case 시나리오 | 핵심 트레이드오프 |
|---|------|---------|----------------------|----------------|
| 01 | Full Loader | 적재 | 외부 API → Silver 디바이스 참조 데이터 (변경 추적 attribute 없음) | 단순 ↔ 데이터 볼륨 증가 시 비용 |
| 02 | Incremental Loader | 적재 | 레거시 트랜잭션 DB → Bronze 방문 이벤트 | 효율 ↔ hard delete 미감지 + late data |
| 03 | CDC | 적재 | 트랜잭션 DB → 30초 SLA streaming broker | 정확성 ↔ Ops 협업 + JOIN 의미론 변화 |
| 04 | Passthrough Replicator | 복제 | production 참조 데이터 → dev/staging (idempotent 아닌 API) | 단순 ↔ 메타데이터 보존 부담 |
| 05 | Transformation Replicator | 복제 | production → staging (PII 제거) | 보안 ↔ 변환 silent fail 위험 |
| 06 | Compactor | 후처리 | 5초 트리거 스트리밍 결과 → batch 처리용 | 쿼리 성능 ↔ 컴퓨트 비용 + cleaning |
| 07 | Readiness Marker | 신호 | Silver 시간당 batch → 다운스트림 ML/BI 팀 | 단순 ↔ enforcement 부재 + late data |
| 08 | External Trigger | 신호 | 백엔드 release → 참조 데이터 적재 | 자원 절약 ↔ 컨텍스트/오류 관리 |

### 7-2. 패턴 선택 의사결정 가이드

```
"새 데이터 소스를 수집해야 한다"
            │
            ▼
[Q1] 변경 추적 컬럼이 있는가?
  ├─ 없음 ─────────► #01 Full Loader (느리게 진화하는 작은 데이터셋)
  └─ 있음
            │
            ▼
[Q2] 지연 요구는?
  ├─ 분/시간 단위 + 삭제 추적 불필요 ──► #02 Incremental Loader
  └─ 초/분 단위 + 또는 hard delete 추적 ──► #03 CDC
            │
            ▼
[Q3] 데이터 복사 필요?
  ├─ production data 그대로 ─► #04 Passthrough Replicator
  └─ PII 제거/마스킹 필요 ──► #05 Transformation Replicator
            │
            ▼
[Q4] 적재 결과가 작은 파일인가?
  ├─ 그렇다 (스트리밍 등) ──► #06 Compactor (+ VACUUM cleaning)
  └─ 적정 크기 ──► 컴팩션 불필요
            │
            ▼
[Q5] 다운스트림 트리거 방식은?
  ├─ 정기 스케줄 (시간/일) ──► #07 Readiness Marker (pull)
  └─ 불규칙 발생 ──► #08 External Trigger (push)
```

### 7-3. 실무 조합 예시 (책 use case 기반)

**시나리오 1: 외부 디바이스 참조 데이터 동기화**
- #01 Full Loader (Silver) + #07 Readiness Marker (다운스트림 알림) + #04 Passthrough Replicator (dev/staging 복제)

**시나리오 2: 레거시 트랜잭션 DB → 실시간 분석**
- #03 CDC (Debezium + Kafka) + #06 Compactor (Bronze의 작은 파일 정리) + #08 External Trigger (다운스트림 자동 트리거)

**시나리오 3: production → staging (PII 제거)**
- #05 Transformation Replicator + Anonymizer 패턴 (챕터 7)

**시나리오 4: hourly Silver 적재 → ML/BI 팀**
- #02 Incremental Loader (partition-based) + #07 Readiness Marker (`_SUCCESS` 또는 `COMPLETED`) + Airflow FileSensor

### 7-4. 책에서 강조한 공통 원칙

- **단순함 유지 (Keep it simple)** — 특히 Passthrough Replicator: raw text API > JSON I/O API, datetime > string 강제 변환 안 함
- **트랜잭션 형식의 가치** — Compactor 의 일관성, Full Loader 의 time travel 등 ACID 가 단순화시키는 영역이 많음
- **메타데이터 보존** — Delta Lake 의 `_delta_log/`, Kafka 의 headers/offset, CDC 의 op/timestamp 모두 빠뜨리면 데이터셋 자체가 사용 불가능해질 수 있음
- **convention 보다 명시적 합의** — Readiness Marker 의 enforcement 부재처럼, 컨슈머와 의미를 명확히 합의 + 메타데이터로 검증 가능하게 만들 것
- **이벤트는 always at-least-once 가정** — External Trigger 에서 idempotency key 로 중복 처리 방어
