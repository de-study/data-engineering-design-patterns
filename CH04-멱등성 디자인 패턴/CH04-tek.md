# 데이터 엔지니어링 디자인 패턴 - 멱등성 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 4 | 실무 데이터 엔지니어링 관점 정리

---

## 목차

1. [덮어쓰기 (Overwriting)](#1-덮어쓰기-overwriting)
   - 패턴 #16: 빠른 메타데이터 정리기 (Fast Metadata Cleaner)
   - 패턴 #17: 데이터 덮어쓰기 (Data Overwrite)
2. [갱신 (Updates)](#2-갱신-updates)
   - 패턴 #18: 병합기 (Merger)
   - 패턴 #19: 상태 저장 병합기 (Stateful Merger)
3. [데이터베이스 (Database)](#3-데이터베이스-database)
   - 패턴 #20: 키 기반 멱등성 (Keyed Idempotency)
   - 패턴 #21: 트랜잭션 기반 작성자 (Transactional Writer)

---

## 책의 use case (전체 챕터 공통)

챕터 3의 오류 관리 패턴이 **대부분의** 문제를 해결한다고 했지만, 전부는 아님.
대표적으로 자동 재시도(automatic retries) 가 데이터 정합성을 직접 흔듦 — 재시도된 task/job 은
이미 성공한 write 를 한 번 더 수행해서 같은 데이터를 두 번 적재할 수 있음.

- **Best case**: 중복이 들어가도 컨슈머가 그것을 중복이라고 식별 가능 → 제거 가능.
- **Worst case**: 중복이 같은 데이터인지조차 알 수 없음 → 데이터셋 신뢰도가 무너짐.

> **사이드바 — 멱등성(Idempotency) 한 줄 정의**
> `absolute(-1) == absolute(absolute(absolute(-1)))` — 몇 번 호출해도 같은 결과를 주는 함수의 성질.
> 데이터 엔지니어링에서는 **"같은 잡을 몇 번 돌려도 일관된 출력"** 을 보장하는 것 — 중복이 아예 없거나, 있더라도 "식별 가능한 중복" 으로 남음.
> producer 측 메시징 시스템이 transactional 하지 않으면 재시도로 인한 중복 발생 자체는 막을 수 없지만, **멱등 처리**가 적용된 컨슈머는 그 중복을 식별·정리할 수 있음.

```
챕터 4 가 다루는 영역
==========================================================================

[챕터 3 패턴들]                       [현실 = 자동 재시도]
   - Dead-Letter                        - task retry
   - Windowed Deduplicator              - job restart
   - Late Data Integrator               - producer 재전송
      │                                       │
      └───────────────────────────────────────┘
                          │
                          ▼
              [중복 / 부분 적재 / 불일치]
                          │
                          ▼
                ┌─────── 멱등성 패턴 ───────┐
                │                            │
        ┌───────┴────────┐         ┌─────────┴──────────┐
        ▼                ▼         ▼                    ▼
  [4.1 Overwriting] [4.2 Updates] [4.3 Database] [4.4 Immutable]
   #16, #17          #18, #19      #20, #21       #22

  본 문서가 다루는 범위: 4.1 Overwriting (#16, #17) / 4.2 Updates (#18, #19) / 4.3 Database (#20, #21)
==========================================================================

원칙: 잡을 N번 돌려도 결과 데이터셋 상태가 동일해야 함
      ─ 재시도·백필이 일상이라는 전제에서 출발하는 설계
```

> 참고 — Maxime Beauchemin 의 2018년 글 "Functional Data Engineering: A Modern Paradigm for Batch Data Processing" 이 데이터 엔지니어링에 멱등성 개념을 대중화시킨 출발점. 책 저자가 챕터 도입부에서 직접 언급.

---

## 1. 덮어쓰기 (Overwriting)

멱등성 패턴의 첫 패밀리는 **data removal** 시나리오를 다룸 — "기존 데이터를 지운 뒤 새 데이터를 쓴다" 는 가장 단순한 접근.
다만 큰 데이터셋에서는 비용이 큼.
그래서 제거 방식을 **data-based** (실제 데이터 파일을 건드림) 와 **metadata-based** (메타데이터 레이어만 건드림) 둘로 나누어 다룸.

- **#16 Fast Metadata Cleaner** — metadata 연산 (`TRUNCATE`, `DROP`) 으로 빠르게 정리. 데이터 파일 자체를 안 건드림.
- **#17 Data Overwrite** — metadata 레이어가 없거나 사용 불가할 때, data 레이어에서 직접 덮어쓰기.

### 1-1. 패턴 #16: 빠른 메타데이터 정리기 (Fast Metadata Cleaner)

데이터 파일을 직접 지우는 대신, 데이터 파일을 설명하는 **훨씬 작은 메타데이터 레이어** 만 건드려 빠르게 정리하는 패턴.
"논리적(logical) 레벨에서만 동작" 하기 때문에 시간 복잡도가 행 수와 무관.

#### 상황 (Problem)

**책의 use case:**
- 일일 배치 잡이 **500 GB ~ 1.5 TB** 의 visit 이벤트를 처리 중.
- 멱등성을 보장하기 위해 두 단계로 구성:
  1. `DELETE` — 이전 run 이 추가한 모든 행 제거
  2. `INSERT` — 새로 처리한 행 삽입
- 3주간은 정상 동작했으나, 테이블이 커지면서 **`DELETE` task 의 성능이 급격히 저하** → latency 문제.
- **결정적 제약**: 계속 증가하는 테이블에서 daily batch 의 멱등성을 보장하면서도 scale-out 가능한 설계가 필요.
  `DELETE` 는 행 식별 + 데이터 파일 재작성을 동반해 데이터 양에 선형 비례.

#### 해결 (Solution)

`DELETE` 대신 **메타데이터 연산** 인 `TRUNCATE TABLE` 또는 `DROP TABLE` 을 활용.
이 두 명령은 데이터 파일을 스캔하지 않고 카탈로그/메타스토어 레이어만 바꿈 → 데이터 크기와 무관하게 빠름.

> **사이드바 — `TRUNCATE TABLE table_a` ≒ `DELETE FROM table_a`**
> 의미상 둘 다 모든 행을 제거. 그러나 내부 동작이 다름 — `TRUNCATE` 는 테이블 스캔을 하지 않으므로 **metadata operation** 으로 분류됨.

핵심 발상: 데이터셋을 monolithic 한 단일 테이블로 보지 말고, **여러 물리적 분할 테이블을 하나의 논리 단위로 노출** 하는 것.
예 — 1년치 visits 를 weekly 테이블 52개 + 단일 view 로 노출.

**idempotency granularity** 라는 개념이 핵심:
- 위 예에서 granularity = 1주.
- granularity 가 곧 "TRUNCATE/DROP 으로 정리하는 단위" 이자 후술하는 **백필 단위** 이기도 함.

데이터 오케스트레이션에 추가해야 할 단계:
1. **실행일 분석** — 새 granularity 시작인지 (예: 월요일/1월 1일), 이전 것을 이어갈지를 판정.
   `Exclusive Choice` 패턴 (Chapter 6) 으로 분기 처리.
2. **idempotency 환경 준비** — `TRUNCATE TABLE` 또는 `DROP TABLE` 수행.
   - `TRUNCATE`: 보통 "테이블이 없으면 만들기" 단계가 선행됨.
   - `DROP`: 자연스럽게 "테이블 재생성" 단계가 후행됨.
3. **단일 abstraction 갱신** — view 같은 단일 진입점을 새 테이블에 맞춰 업데이트.
   `DROP` 기반에서는 사용자가 view 를 조회하는 도중 drop 이 일어나면 에러가 날 수 있으므로,
   **"view 에서 테이블 제거 → DROP → CREATE → view 에 다시 포함"** 순서를 권장.

incremental/partitioned 가 아니라 **full dataset** 에도 적용 가능 — 매 load 마다 단순히 테이블 재생성으로 단순화하거나,
대안으로 다음 절의 `Data Overwrite` 패턴을 사용.

```
주간 파티션 + 단일 view 노출 구조
--------------------------------------------------------------
[Pipeline, week 1, 2024]  ──►  [Visits, week 1, 2024]  ┐
[Pipeline, week 2, 2024]  ──►  [Visits, week 2, 2024]  │
                                                       ├── view: visits_2024
        ...                          ...               │   (사용자 단일 진입점)
[Pipeline, week 52, 2024] ──►  [Visits, week 52, 2024] ┘

idempotency 동작 (TRUNCATE 기반):
  실행일 = 월요일/1월 1일 ?
       ├─ Yes → 테이블 없으면 생성 → view 에 포함 → TRUNCATE
       │                                            └─► daily 데이터 처리 → weekly 테이블에 적재
       └─ No  → daily 데이터 처리 → weekly 테이블에 적재

idempotency 동작 (DROP 기반):
  실행일 = 월요일/1월 1일 ?
       ├─ Yes → view 에서 테이블 제거 → DROP → 테이블 재생성 → view 에 다시 포함
       │                                                       └─► daily 처리 → 적재
       └─ No  → daily 처리 → 적재
--------------------------------------------------------------

비교 — 기존 DELETE+INSERT vs Fast Metadata Cleaner
  - DELETE+INSERT  : O(데이터 크기), 행 스캔 + 파일 재작성 필요 → 시간이 갈수록 무거워짐
  - TRUNCATE/DROP  : O(1), 메타스토어 변경만으로 끝남 → 데이터 크기와 무관
```

#### 고려사항 (Consequences)

`fast` 와 `data removal` 키워드가 매력적이지만, 다음 함정이 있음.

**Granularity and backfilling boundary — 정리 단위 = 백필 단위**
> "지우고 다시 채우는 작업(백필 경계)을, 내가 원하는 만큼 정밀하게 쪼개서(세분화) 처리할 수 없다"
- idempotency granularity 가 곧 **backfilling granularity**. 파이프라인을 replay 하려면 **파티션 테이블 생성 task부터** 다시 돌려야 함. 아니면 데이터셋이 불일치 상태가 됨.
- 예: weekly partition 인데 화요일 하루만 백필 필요 → 그 주 전체를 재실행해야 함. 단, 다른 요일들은 data loading step 만 selective replay 가능 (full pipeline 재실행은 불필요).
- 더 fine-grained 한 단위 (예: 한 user/customer) 의 백필에는 **부적합** — 메타데이터 연산은 항상 whole table 에 동작하기 때문.

**Metadata limits — 데이터 스토어의 한도**
- 패턴은 dedicated partition/테이블을 계속 만들어 쌓는 구조 → 그러나 무한 생성 불가.
- 예: **GCP BigQuery 는 테이블당 partition 4,000개**, **AWS Redshift 는 클러스터당 200,000 테이블** 한도.
- 여러 파이프라인에서 이 패턴을 동시에 쓰면 한도에 빠르게 도달.
- 완화책 — **freezing step**: 일정 기간이 지난 mutable idempotent 테이블을 immutable 한 더 큰 단위로 합치기 (예: weekly → monthly 또는 yearly).
- 또한 **메타데이터 연산을 지원하는 DB** (data warehouse, lakehouse, RDBMS) 에서만 동작. object store 위에서는 사실상 불가 → `Data Overwrite` 로 대체.

**Data exposition layer — 단일 진입점 유지**
- 데이터셋이 더 이상 한 곳에 존재하지 않음 → 사용자가 "내부 분할 구조" 를 몰라도 되게 view 같은 logical 구조로 통합 노출 필요.

**Schema evolution — 스키마 변경의 비용**
- idempotent 테이블에 새 컬럼이 추가되면 **기존 테이블들의 스키마를 별도 파이프라인으로 업데이트** 해야 함.
  Fast Metadata Cleaner 안에서 처리하면 자동으로 과거 데이터를 reprocessing 하게 되어 비효율적.
- 단, "새 required field 추가" 시나리오는 예외 — 어차피 과거 run 을 replay 하면서 새 필드가 자동 채워지므로 패턴 안에서 처리 가능.

#### 구현 예시 (Examples)

**예시 1 — Apache Airflow: 실행일에 따라 분기하는 idempotency router**

`BranchPythonOperator` 로 "주간 테이블 새로 만들 날인지" 판정:
```python
# (1) execution_date 가 월요일이거나 1월 1일이면 → 테이블 생성 분기로
def retrieve_path_for_table_creation(**context):
    ex_date = context['execution_date']
    should_create_table = ex_date.day_of_week == 1 or ex_date.day_of_year == 1
    return 'create_weekly_table' if should_create_table else 'dummy_task'

check_if_monday_or_first_january_at_midnight = BranchPythonOperator(
    task_id='check_if_monday_or_first_january_at_midnight',
    provide_context=True,
    python_callable=retrieve_path_for_table_creation
)
```

**예시 2 — Apache Airflow: 주간 테이블 생성 + view 갱신 분기**

새 주간 테이블을 만들고 view 를 그 테이블 포함해 다시 정의:
```python
# (2) 주차 번호가 들어간 새 weekly 테이블 생성
create_weekly_table = PostgresOperator(  # ...
    sql='/sql/create_weekly_table.sql'
)

# (3) visits 라는 통합 view 를 새 weekly 테이블 포함하도록 재정의
recreate_view = PostgresViewManagerOperator(  # ...
    view_name='visits',
    sql='/sql/recreate_view.sql'
)
```

이후 두 분기 (생성 / 직행) 모두 공통의 "input 을 weekly 테이블로 적재" 단계를 거침.

| 구현 방식 | 동작 | 주의사항 |
|----------|------|----------|
| `TRUNCATE` 기반 | 테이블 유지 + 행만 비움 | 컨텍스트 테이블이 미리 있어야 함 (없으면 CREATE 선행 task) |
| `DROP` 기반 | 테이블 drop 후 CREATE | drop 도중 view 조회 시 에러 → "view 에서 빼기 → drop → create → view 에 넣기" 순서 권장 |
| full dataset 변형 | 매 load 마다 단순 재생성 | 분할 구조가 없어 schema evolution 시 운영 단순. 단 큰 데이터에는 비효율 → #17 대신 고려 |

> **트러블 로그** — granularity 와 백필 단위가 같다는 점을 놓치면 한 번에 큰 데이터가 사라짐.
> 예: weekly partition 으로 운영 중인데 어느 화요일 데이터에 버그 발견 → "그 화요일만 다시 처리" 한다고 단일 day 백필을 돌리면,
> Fast Metadata Cleaner 가 그 주 weekly 테이블 전체를 `TRUNCATE` 하고 화요일 데이터만 다시 적재 → **월·수·목·금·토·일 데이터가 통째로 증발**.
> 다음 날 다운스트림 주간 KPI 가 평소의 1/7 로 떨어져 발견됨.
> 또 다른 흔한 사고 — daily granularity 로 운영하다 BigQuery 의 **테이블당 4,000 partition 한도** 에 11년차쯤 도달.
> 11년치 daily partition = 약 4,015개. partition 생성이 거부되면서 일자별 적재가 줄줄이 실패.
> **백필 시에는 반드시 "granularity 단위 전체" 를 replay** 하거나, data loading step 만 selective 로 다시 돌릴 것.
> **장기 운영 잡에는 freezing 단계 (weekly → monthly/yearly 통합) 를 미리 설계** 해 둘 것.

---

### 1-2. 패턴 #17: 데이터 덮어쓰기 (Data Overwrite)

메타데이터 연산이 불가하거나 비용이 큰 경우, **데이터 레이어 자체에서 dataset 을 통째로 (또는 일부 조건만) 교체** 하는 패턴.

#### 상황 (Problem)

**책의 use case:**
- 일일 배치 잡이 visit 데이터셋을 **object store 의 event-time partitioned 위치** 에 저장.
- 적절한 멱등성 전략이 없어 매 backfilling 액션이 중복 레코드를 만들고 있음.
- Fast Metadata Cleaner 를 알고는 있지만, **object store 에는 적절한 메타데이터 레이어가 없어** 적용 불가.
- **결정적 제약**: `TRUNCATE`/`DROP` 같은 metadata 연산이 없는 저장소에서도 멱등성을 보장할 수 있어야 함.

#### 해결 (Solution)

메타데이터 레이어가 없거나 사용이 어려우면, **data layer 의 native replacement command** 를 사용.
구체적 구현은 스택에 따라 다름.

**Data processing framework 사용 시 (configuration-driven)**
- Apache Spark — `save mode` 옵션 (`overwrite`) 으로 writer 설정.
- Apache Flink — write mode properties.
- 프레임워크가 알아서 기존 파일을 정리하고 새로 씀.
- **Delta Lake** 는 선택적 덮어쓰기 (`replaceWhere`) 도 지원 — 조건에 매칭되는 부분만 덮어쓰기.

**SQL 직접 사용 시**
- `DELETE FROM` + `INSERT INTO` — 가장 익숙한 조합. 단순하지만 두 번의 동작이 필요.
- `INSERT OVERWRITE` — 더 간결. 한 문장에 전체 테이블을 INSERT 부분의 레코드로 교체. 단점: row 단위 selective 덮어쓰기는 안 됨.
- 데이터 스토어의 native loading command — 예: BigQuery 의 `LOAD DATA OVERWRITE`. 그 외는 보통 `TRUNCATE TABLE` 선행 필요.

> **사이드바 — Not Only List of Columns**
> `INSERT` 는 값을 명시적으로 나열할 필요가 없음. 다른 테이블에서 `SELECT` 로 가져와도 됨.
> 예: `INSERT INTO visits (id, v_time) SELECT visit_id, visit_time FROM visits_raw` —
> `visits_raw` 의 모든 행을 별도 declaration 없이 그대로 삽입.

**중요한 차이 — overwriting ≠ 즉시 디스크 제거**
- time travel 을 지원하는 저장소 (table file format, BigQuery, Snowflake 등) 는 overwrite 후에도 옛 데이터 블록을 retention 기간 동안 남겨둠.
- 사용자에게는 안 보이지만 디스크는 여전히 점유 → retention 만료 또는 `VACUUM` 으로 정리 필요.

```
Data Overwrite 동작 (Apache Spark + 모던 table file format)
--------------------------------------------------------------
[기존 데이터 파일 (parquet 등)]      [transaction log]
       │                                   │
       │  write.mode('overwrite')          │
       ▼                                   ▼
[새 parquet 파일 작성]            [기존 파일을 "삭제됨" 으로 commit]
       │                                   │
       └────────────┬──────────────────────┘
                    ▼
          [retention 기간 / VACUUM 명령]
                    │
                    ▼
          [실제 디스크에서 제거]

옵션 (구현체별):
- Spark 전체 덮어쓰기  : write.mode('overwrite')
- Delta 선택 덮어쓰기  : .option('replaceWhere', "date = '2024-01-01'")
- SQL 등가 1          : INSERT OVERWRITE INTO ... SELECT ...
- SQL 등가 2          : DELETE FROM ... + INSERT INTO ... (selective)
- BigQuery native     : LOAD DATA OVERWRITE / bq load --replace=true
--------------------------------------------------------------
```

#### 고려사항 (Consequences)

data 레이어를 직접 건드리므로 적용 범위는 Fast Metadata Cleaner 보다 넓지만, 다음 비용이 따름.

**Data overhead — 데이터 크기에 비례하는 비용**
- 데이터셋이 크고 **partitioned 되어 있지 않으면** overwrite 자체가 느려짐 — 시간이 갈수록 처리량이 줄어드는 구조.
- 완화책 — partitioning 같은 storage optimization 적용. 덮어쓸 데이터 양 자체를 줄이면 교체가 빨라짐.
- (책의 후반에 storage 패턴 패밀리가 별도로 다뤄짐.)

**Vacuum need — 사라진 것 같지만 디스크에는 남는 데이터**
- `DELETE` (또는 overwrite 의 내부 delete) 는 디스크에서 데이터를 **즉시** 제거하지 않을 수 있음.
- table file format, RDBMS 에서 발생 — 사용자 `SELECT` 에는 안 보이지만 dead row 가 디스크에 남음.
- 공간 회수를 위해 `VACUUM` (또는 동등한 정리 프로세스) 을 주기적으로 실행해야 함.

#### 구현 예시 (Examples)

**예시 1 — SQL: `INSERT OVERWRITE`**

테이블/파티션 콘텐츠를 truncate 후 새 데이터 삽입을 한 문장으로:
```sql
-- devices 테이블 전체를 devices_staging 의 'valid' 행들로 교체
INSERT OVERWRITE INTO devices SELECT * FROM devices_staging WHERE state = 'valid';
```

**예시 2 — BigQuery: 로딩 시 prior truncation (`bq load --replace=true`)**

`writeDisposition` 의 `WRITE_TRUNCATE` 와 동일한 단축형:
```bash
bq load dedp.devices gs://devices/in_20240101.csv ./info_schema.json --replace=true
```

**예시 3 — Apache Spark (PySpark) save mode**

writer 설정만 변경:
```python
input_data.write.mode('overwrite').text(job_arguments.output_dir)
```

| 구현 방식 | 동작 | 주의사항 |
|----------|------|----------|
| Spark `mode('overwrite')` | 출력 디렉토리의 기존 파일을 새 파일로 교체 | save mode 자체는 **transactional 이 아님** — 잡 중간 실패 시 일부만 새 파일/일부는 옛 파일로 남을 수 있음. 모던 table file format (Delta/Iceberg) 이 commit 기반으로 이 문제를 해결 |
| Delta Lake `replaceWhere` | 조건 매칭 파티션만 선택 덮어쓰기 | 조건 컬럼은 partition 컬럼이어야 효율적. 비파티션 컬럼이면 풀스캔 발생 |
| SQL `INSERT OVERWRITE` | 전체 테이블/파티션 교체 | row 단위 selective 덮어쓰기는 안 됨 → 부분만 바꾸려면 `DELETE` + `INSERT` 조합 |
| BigQuery `LOAD DATA OVERWRITE` | 로드 시점에 truncate + 적재 | time travel 보존 기간 만료 전엔 디스크 공간 차지 |

> **트러블 로그** — "save mode = overwrite" 가 transactional 이라고 착각하면 데이터 corruption 위험.
> 예: plain parquet 디렉토리 (Delta/Iceberg 미적용) 에 매일 `df.write.mode('overwrite').parquet(path)` 로 적재 중,
> 새벽 잡이 80% 진행 시점에 OOM 으로 죽음 → 디렉토리에는 옛 파일 일부 + 새 파일 일부가 섞여 남음.
> 후속 reader 가 그걸 읽으면 "어제 데이터 절반 + 오늘 데이터 절반" 의 정체불명 데이터셋이 됨.
> Delta/Iceberg 같은 table file format 에서는 overwrite 가 transaction log 의 새 commit 으로 처리되어 commit 이전엔 reader 가 옛 스냅샷을 그대로 보지만, plain parquet 디렉토리에는 그 보호장치가 없음.
> 또 다른 사고 — 100GB Iceberg 테이블에 일일 `INSERT OVERWRITE` 만 돌리고 `VACUUM` 을 한 번도 안 돌려서, 6개월 뒤 retention 안에 누적된 옛 스냅샷이 **30 TB** 까지 부풀어 객체 저장소 비용이 폭증.
> **overwrite 패턴은 (a) transaction 보장이 있는 포맷 위에서 쓰고, (b) retention 정책과 VACUUM 스케줄을 함께 설계** 할 것.

---

## 2. 갱신 (Updates)

데이터셋을 통째로 지우고 다시 쓰는 덮어쓰기는 단순하지만, **모든 데이터셋에 맞는 건 아님**.
대표적으로 **증분 갱신 데이터셋(updated incremental dataset)** 이 문제 — 데이터 제공자가 매 버전마다
**변경/추가된 일부 행만** 보내는 경우, 전체를 덮어쓰려면 "엔티티별 최신 버전만 남기는" 전처리를 매번 해야 함.

이럴 때는 지우고 다시 쓰는 대신, **새 변경분을 기존 데이터셋에 합치는(merge)** 접근이 더 적합함.

- **#18 Merger** — `MERGE`(UPSERT) 로 신규 행은 insert, 기존 행은 update. stateless.
- **#19 Stateful Merger** — Merger 에 **상태 테이블(state table)** 을 더해, 백필 시 옛 버전으로 **복원(restore)** 후 병합. 일관성 보장.

### 2-1. 패턴 #18: 병합기 (Merger)

전체 데이터셋이 없고 **증분 변경분만** 들어올 때, 기존 데이터셋과 새 변경분을 `MERGE`(UPSERT) 로 결합하는 패턴.

#### 상황 (Problem)

**책의 use case:**
- Apache Kafka 토픽에서 **CDC(Change Data Capture)** 로 동기화된 변경 스트림을 처리하는 배치 파이프라인 구축 중.
- 이 변경분을 **Delta Lake 테이블** 에 반영해야 함.
- 테이블은 **데이터 소스의 특정 시점 상태를 그대로** 반영해야 함 → 중복이 있으면 안 됨.
- **결정적 제약**: 전체 데이터셋이 손에 없고 **증분 변경분만** 들어옴 → 덮어쓰기(delete-and-replace)로는 처리 불가.
  엔티티 식별자를 기준으로 새 행과 기존 행을 합쳐야만 함.

> **사이드바 — 전체 데이터셋이면 그냥 Overwrite 가 쉬움**
> 완전한 데이터셋을 매번 받는다면 덮어쓰기 패턴(#16/#17)이 더 단순함 — 그냥 지우고 새로 쓰면 끝.
> Merger 는 "기존 행과 새 행을 데이터 레벨에서 결합" 해야 하므로 더 무거움.
> 따라서 Merger 는 **증분 데이터셋** 처럼 delete-and-replace 로 다루기 어려운 경우에 한정해 사용.

#### 해결 (Solution)

**가장 중요한 첫 단계는 결합 키(combine attributes) 정의** — 새 데이터셋과 기존 데이터셋을 무엇으로 매칭할지 결정.
- 유일성이 보장되면 단일 속성(예: `user_id`) 사용.
- 그렇지 않으면 복합 키(예: 방문 이벤트의 `visit_id` + `visit_time`).

그다음, 처리 계층에서 데이터셋을 결합. 오늘날 대부분의 솔루션(데이터 처리 프레임워크, 테이블 파일 포맷, 데이터 웨어하우스)이
`MERGE`(=`UPSERT`) 명령을 지원하며, 이것이 Merger 패턴 구현의 정답.

`MERGE` 가 다뤄야 할 **세 가지 시나리오**:

- **Insert** — 새 데이터셋의 행이 현재 테이블에 없음 → 신규 레코드로 추가.
- **Update** — 양쪽에 다 있음 → 새 데이터셋의 갱신된 버전으로 업데이트.
- **Delete** — **가장 까다로움**. Merger 는 hard delete 를 직접 지원하지 않음.
  병합할 데이터셋에서 행이 그냥 빠지면 **아무 일도 일어나지 않기 때문**.
  따라서 삭제는 **soft delete**(삭제 표시 속성을 가진 update) 로 표현해야 detect 후 hard/soft delete 적용 가능.

> **사이드바 — `is_deleted` 플래그는 INSERT 절에도 필요**
> soft delete 구현 시 `WHEN NOT MATCHED ... INSERT` 절에도 `is_deleted = false` 조건을 둬야 함.
> 안 그러면 **첫 실행** 때 이미 삭제된 레코드까지 insert 되고, 이후엔 매칭으로 잡히지 않아 **영원히 못 지움**.

```
Merger 동작 (MERGE INTO target USING input)
--------------------------------------------------------------
[기존 테이블 target]          [새 증분 input]
   type=A, v=1                  type=A, v=1 (full_name 변경)
   type=B, v=1                  type=C, v=1 (신규)
       │                            │
       └──────────── ON type,version ┘
                     │
        ┌────────────┼─────────────────────┐
        ▼            ▼                      ▼
  WHEN MATCHED   WHEN MATCHED          WHEN NOT MATCHED
  + is_deleted   + !is_deleted         + !is_deleted
     → DELETE       → UPDATE              → INSERT
  (soft delete)  (A 행 갱신)            (C 행 추가)
--------------------------------------------------------------
핵심: input 에서 빠진 행은 "아무 일 없음" → hard delete 불가
      삭제는 is_deleted 플래그로 들어와야 detect 됨
```

#### 고려사항 (Consequences)

단순해 보이지만, 데이터셋 성격(증분/전체)에 따른 함정이 숨어 있음.

- **Uniqueness — 식별자 불변성이 첫 번째이자 가장 중요한 요구사항**
  - 데이터 제공자 또는 생성 잡이 각 레코드를 안전하게 식별할 **불변 속성** 을 정의해야 함.
  - 식별자가 흔들리면 백필 시 update 되어야 할 행이 새 행으로 **insert** 되어 **불일치 중복** 발생.

- **I/O — data-based 패턴의 비용**
  - Fast Metadata Cleaner 와 달리 Merger 는 데이터 블록 레벨에서 동작 → **compute 부하가 큼**.
  - 완화책 — 모던 DB/테이블 파일 포맷은 메타데이터 레이어에서 영향받는 레코드를 먼저 찾아 **불필요한 파일 스캔을 skip**.

- **Incremental datasets with backfilling — 백필 중 일시적 불일치**
  - 증분 데이터셋이므로 백필은 **가장 최근 버전** 에서 시작함. 그런데 그 최신 버전에는 백필 대상 시점 **이후** 의 행들(M, N, O 등)이 이미 들어 있음.
  - 결과적으로 백필이 과거 시점부터 다시 채워지는 동안 **컨슈머는 정상 run 과 다른 데이터** 를 보게 됨(테이블이 서서히 정상으로 수렴).
  - 완화책 — 파이프라인 **외부에 복원(restore) 메커니즘** 을 두어 첫 replay 시점으로 롤백. time travel 같은 버저닝 기능이 있으면 쉬움.
    단 이렇게 하면 stateless Merger 가 stateful 로 바뀜 → 다음 **#19 Stateful Merger** 로 이어짐.

> **사이드바 — 데이터 제공자의 실수는 백필이 답이 아님**
> 제공자가 잘못된 값을 넣었다면 파이프라인을 replay 할 필요 없음.
> **수정된 데이터셋을 새 증분으로 받아 `MERGE`** 하면 잘못된 행이 올바른 값으로 덮어써짐.
> 단, 행의 **식별자가 그대로** 여야 함(옛 행과 새 행을 매칭할 수 있어야 함).

#### 구현 예시 (Examples)

**예시 1 — Apache Airflow + SQL: 임시 테이블에 신규 파일 로드**

`LIKE` 연산자로 대상 테이블 스키마를 그대로 복제 → 속성 일일이 선언할 필요 없음(메타데이터 비동기화 방지). 트랜잭션 종료 시 자동 파기됨:
```sql
-- (1) 새 CSV 를 임시 테이블로 로드. LIKE 로 dedp.devices 스키마 그대로 복제
CREATE TEMPORARY TABLE changed_devices (LIKE dedp.devices);
COPY changed_devices FROM '/data_to_load/dataset.csv' CSV DELIMITER ';' HEADER;
```

**예시 2 — `MERGE` 본체 (증분 파일, deletes 없음)**

입력이 증분(신규/변경분만)이라 delete 절은 생략:
```sql
-- (2) type + version 으로 매칭 → 있으면 UPDATE, 없으면 INSERT
MERGE INTO dedp.devices AS d USING changed_devices AS c_d
ON c_d.type = d.type AND c_d.version = d.version
WHEN MATCHED THEN
  UPDATE SET full_name = c_d.full_name
WHEN NOT MATCHED THEN
  INSERT (type, full_name, version) VALUES (c_d.type, c_d.full_name, c_d.version)
```

**예시 3 — soft delete 까지 포함한 `MERGE`**

`is_deleted` 플래그로 삭제를 표현. INSERT 절에도 `is_deleted = false` 조건을 둬 첫 실행 시 삭제행 유입 방지:
```sql
MERGE INTO dedp.devices_output AS target
USING dedp.devices_input AS input
ON target.type = input.type AND target.version = input.version
WHEN MATCHED AND input.is_deleted = true THEN
  DELETE
WHEN MATCHED AND input.is_deleted = false THEN
  UPDATE SET full_name = input.full_name
WHEN NOT MATCHED AND input.is_deleted = false THEN
  INSERT (full_name, version, type) VALUES (input.full_name, input.version, input.type)
```

| 시나리오 | MERGE 절 | 주의사항 |
|----------|----------|----------|
| 신규 행 | `WHEN NOT MATCHED THEN INSERT` | soft delete 사용 시 `is_deleted = false` 조건 필수 |
| 갱신 | `WHEN MATCHED THEN UPDATE` | 결합 키는 불변 속성이어야 함 |
| 삭제 | `WHEN MATCHED AND is_deleted = true THEN DELETE` | hard delete 는 직접 불가 → soft delete 로 표현 |
| 입력에서 빠진 행 | (해당 절 없음) | 아무 일도 안 일어남 — 삭제로 처리되지 않음에 주의 |

> **트러블 로그** — Merger 의 결합 키를 mutable 속성으로 잡으면 백필 시 조용히 중복이 쌓임.
> 예: 디바이스 테이블을 `serial_number` 대신 매 ETL 마다 새로 부여되는 `surrogate_id` 로 매칭하도록 짜 두면,
> 백필로 같은 디바이스를 재처리할 때 `surrogate_id` 가 달라져 `WHEN MATCHED` 가 안 잡히고 `INSERT` 로 빠짐 →
> 같은 디바이스가 2건, 3건씩 쌓이고도 에러는 안 나서 한참 뒤 다운스트림 집계가 부풀어서야 발견됨.
> 또 하나 — soft delete 의 INSERT 절에서 `is_deleted = false` 조건을 빠뜨리면, 최초 적재 때 이미 삭제된 레코드가 테이블에 들어가
> 이후 어떤 증분으로도 매칭되지 않아 **영구 잔류** 함.
> **결합 키는 데이터 제공자가 보장하는 불변 식별자만 쓰고, soft delete 의 INSERT 절에는 반드시 `is_deleted = false` 를 넣을 것.**

---

### 2-2. 패턴 #19: 상태 저장 병합기 (Stateful Merger)

Merger 에 **상태 테이블(state table)** 을 더해, 백필 시 테이블을 마지막 정상 버전으로 **복원한 뒤** 병합함으로써 백필 중 일관성을 보장하는 패턴.

#### 상황 (Problem)

**책의 use case:**
- Merger 패턴으로 두 Delta Lake 테이블 간 변경을 동기화하는 데 성공.
- 1주일 뒤 병합된 데이터셋에서 **이슈를 발견** → 비즈니스 사용자가 백필 요청.
- 사용자는 **일관성** 을 중시 → 백필 전에 **마지막 정상 버전으로 데이터셋을 복원** 해 줄 것을 요구.
- **결정적 제약**: Merger 는 병합 동작에만 집중하므로 복원 기능이 없음 → 백필 중에도 컨슈머에게 일관된 상태를 보여주려면 복원 능력이 필요.

#### 해결 (Solution)

데이터셋을 복원하려면 병합만으로는 부족함. **추가 상태 테이블** 로 데이터 복원 능력을 제공하는 것이 Stateful Merger.

워크플로에 단계 2개가 추가됨:
1. **(시작 단계) 복원** — 필요 시 병합 테이블을 옛 버전으로 되돌림.
2. **(종료 단계) 상태 테이블 갱신** — `MERGE` 가 만든 새 테이블 버전을 실행 시각과 함께 기록.

**상태 테이블** 은 `(실행 시각 → 테이블 버전)` 매핑을 저장. 예를 들어 09:00 실행이 버전 5, 10:00 실행이 버전 6 을 만들면 그대로 기록.

**백필 감지 로직** — 복원은 **백필 모드일 때만** 동작(정상 run 이면 아무 것도 안 함):
- 오케스트레이터가 실행 컨텍스트로 모드(백필/정상)를 알려주면 그걸 그대로 활용.
- 아니면 상태 테이블로 직접 판정:
  1. **이전 run 이 만든 버전** 조회. 없으면 → 첫 실행이거나 첫 실행을 백필하는 것 → `TRUNCATE TABLE` 후 바로 병합.
  2. **현재 테이블 최신 버전 vs 이전 run 버전** 비교:
     - **같으면** → 정상 run → 복원 불필요.
     - **다르면** → 백필 → 병합 전에 테이블을 이전 버전으로 복원.

```
Stateful Merger 워크플로
--------------------------------------------------------------
  [① 복원 단계]  →  [② MERGE]  →  [③ 상태 테이블 갱신]
       │                              │
       └─ 백필 모드일 때만 동작        └─ 새 버전을 (실행시각→버전) 으로 기록

상태 테이블 예 (일일 잡 4회 실행 후)
  실행 시각        버전
  2024-10-05        1
  2024-10-06        2
  2024-10-07        3
  2024-10-08        4

복원 판정 (2024-10-07 을 두 번째로 실행 = 백필):
  이전 run(2024-10-06) 버전 = 2,  현재 최신 버전 = 4   →  2 ≠ 4  →  백필!
       └─► 테이블을 버전 2 로 restore 후 MERGE → 새 버전 5 기록
--------------------------------------------------------------
```

**버저닝 없는 데이터 스토어에서의 변형:**
테이블을 버저닝하는 대신, **원시 데이터 테이블(`devices_history`)** 에 `execution_time` 컬럼과 함께 모든 원시 데이터를 적재.
백필 감지는 "미래 실행 시각의 레코드가 이미 있는지" 로 판정하고, 복원은 해당 시점 이후 행을 지운 뒤 **Windowed Deduplicator** 로 테이블을 재구성.

#### 고려사항 (Consequences)

Merger 의 백필 이슈는 해결하지만, 고유의 함정이 따라옴.

- **Versioned data stores — 버저닝 가능한 스토어가 전제**
  - 제시된 구현은 각 write 가 새 테이블 버전을 만드는 **버저닝 스토어** 를 요구(테이블 파일 포맷 등). 그래야 상태 추적·복원이 가능.
  - 버저닝이 없으면 → 위의 `devices_history` 기반 변형으로 적응.

- **Vacuum operations — retention 만료가 복원 한계를 정함**
  - Delta Lake/Iceberg 같은 버저닝 데이터셋도 **retention 기간이 지나면 옛 파일을 제거** → 그 시점 이전 버전은 복원 불가.
  - 완화책 — retention 기간을 늘리면 백필 가능 범위가 늘지만 **저장 비용 증가**. 또는 "retention 너머는 백필 불가" 를 수용.

- **Metadata operations — compaction 이 만드는 버전 점프**
  - `VACUUM` 외에 **compaction**(작은 파일을 큰 파일로 합치기, Compactor 패턴)도 데이터를 안 바꾸면서 **새 버전을 만듦**.
  - 따라서 복원 시 항상 "이전 버전" 을 쓰면, 두 병합 run 사이의 compaction 을 놓침.
  - 완화책 — 버전이 1씩 증가한다는 가정하에, **"현재 실행 시각의 버전 − 1"** 을 복원 버전으로 사용.

```
compaction 을 고려한 복원 버전 계산
--------------------------------------------------------------
  version_to_restore = version_for_current_execution_time - 1

  예) 2024-10-07 실행의 버전 = 9  →  복원 버전 = 8
      (버전 8 = 2024-10-07 와 2024-10-08 run 사이에 compaction 된 테이블)
--------------------------------------------------------------
```

#### 구현 예시 (Examples)

**예시 1 — Delta Lake 상태 테이블 정의**

실행 시각과 그때 생성된 Delta 버전 두 컬럼:
```sql
CREATE TABLE IF NOT EXISTS `default`.`versions`
(execution_time STRING NOT NULL, delta_table_version INT NOT NULL)
```

**예시 2 — 현재/과거 버전 조회 (복원 단계 진입)**

`DESCRIBE HISTORY` 로 마지막 `MERGE` 버전을, 상태 테이블로 이전 실행 버전을 가져옴:
```python
# 마지막 MERGE 가 만든 테이블 버전
last_merge_version = (spark.sql('DESCRIBE HISTORY default.devices')
    .filter('operation = "MERGE"')
    .selectExpr('MAX(version) AS last_version').collect()[0].last_version)

# 이전 실행 시각에 기록된 버전
maybe_previous_job_version = spark.sql(f'''SELECT delta_table_version FROM versions
    WHERE execution_time = "{previous_execution_time}"''').collect()
```

**예시 3 — 버전 비교 후 truncate 또는 restore**

이전 버전 기록이 없으면 `TRUNCATE`, 있으면서 최신보다 낮으면 백필로 판단해 `restoreToVersion`:
```python
if not maybe_previous_job_version:
    spark.sql('TRUNCATE TABLE default.devices')          # 첫 실행/첫 백필
else:
    previous_job_version = maybe_previous_job_version[0].delta_table_version
    if previous_job_version < last_merge_version:        # 백필 모드
        (DeltaTable.forName(spark, 'devices')
            .restoreToVersion(previous_job_version))
```

**예시 4 — `MERGE` 후 상태 테이블 갱신**

새 버전을 다시 `MERGE` 로 기록(백필이면 update, 정상 run 이면 insert):
```python
last_version = (spark.sql('DESCRIBE HISTORY default.devices')
    .selectExpr('MAX(version) AS last_version').collect()[0].last_version)
new_version_df = spark.createDataFrame([
    Row(execution_time=current_execution_time, delta_table_version=last_version)])
(DeltaTable.forName(spark_session, 'versions').alias('old_versions')
    .merge(new_version.alias('new_version'),
           'old_versions.execution_time = new_version.execution_time')
    .whenMatchedUpdateAll().whenNotMatchedInsertAll().execute())
```

> **트러블 로그** — Stateful Merger 를 buffer 길게 잡지 않고 짧은 retention 위에 올리면, 정작 백필이 필요한 순간 복원할 버전이 사라져 있음.
> 예: Delta 테이블 retention 을 기본 7일로 둔 채 운영하다, 3주 전 데이터의 정합성 이슈로 백필을 시도 →
> `restoreToVersion(120)` 호출이 "version 120 의 파일이 VACUUM 으로 제거됨" 에러로 실패하고 복원 자체가 불가능해짐.
> 또 다른 함정 — 야간에 compaction 잡이 돌아 버전이 1 점프했는데 복원 로직이 "이전 버전" 을 그대로 쓰면,
> compaction 이 정리한 파일 구성을 놓쳐 복원 후 테이블이 깨진 상태가 됨.
> **백필 가능 기간만큼 retention 을 확보하고, compaction 이 도는 환경에서는 복원 버전을 "현재 실행 버전 − 1" 로 계산할 것.**

---

## 3. 데이터베이스 (Database)

앞의 패턴들은 오케스트레이션 계층을 손보거나 정교한 write 연산을 짜야 했음.
**일이 너무 많다면** — 때로는 **데이터베이스 자체의 기능** 에 멱등성을 위임하는 지름길을 쓸 수 있음.

- **#20 Keyed Idempotency** — 키 기반 스토어 + **멱등 키 생성 전략** → 몇 번 저장해도 정확히 한 번 기록됨.
- **#21 Transactional Writer** — DB의 **트랜잭션(all-or-nothing)** 으로, 커밋 전까지 부분 데이터가 컨슈머에 노출되지 않게 함.

### 3-1. 패턴 #20: 키 기반 멱등성 (Keyed Idempotency)

키 기반 데이터 스토어에서 **항상 같은 키를 생성** 하도록 만들어, 같은 레코드를 몇 번 저장해도 한 번만 쓰이게 하는 패턴.

#### 상황 (Problem)

**책의 use case:**
- 스트리밍 파이프라인이 방문 이벤트를 처리해 **사용자 세션** 을 생성.
- 사용자별 시간 윈도우 동안 메시지를 버퍼링하고, 갱신된 세션을 **key-value 스토어** 에 기록.
- task 재시도 시 중복이 생기지 않도록 이 파이프라인도 **멱등** 해야 함.
- **결정적 제약**: 같은 사용자의 모든 방문 이벤트가 **항상 동일한 세션 ID** 를 받아야 함 → 그래야 재시도해도 한 번만 write 됨.

#### 해결 (Solution)

키 기반 DB에서 멱등성은 **처리 측의 키 생성 로직** 에 달려 있음. 같은 사용자의 모든 방문에 **같은 세션 ID** 를 생성하면 한 번만 쓰임.

구현은 **불변(immutable) 속성으로 키를 생성** 하는 데서 시작:
- 운이 좋으면 입력 데이터에 유일 속성이 이미 있음(예: "최신 활동" 이면 `user_id` 그대로).
- 없으면 조합해서 만들어야 함 — 예: `user_id` + `first_visit_time`.

**여기에 함정이 있음 — event time 은 mutable.**
잡이 런타임 에러로 멈췄다 재시작된 사이에 **늦게 도착한(late) 데이터** 가 입력 스토어에 쓰이면,
"첫 방문 시각" 이 바뀌어 **세션 ID 가 달라짐** → 멱등성 깨짐.

```
late data 가 키를 흔드는 시나리오
--------------------------------------------------------------
[최초 run]   user=U 의 first_visit = 10:00  →  session_id = hash(10:00)
                  │  (잡이 런타임 에러로 중단)
                  ▼
[재시작 후]  09:55 의 late 레코드가 입력에 추가됨
             user=U 의 first_visit = 09:55  →  session_id = hash(09:55)  ✗ 새 세션!
--------------------------------------------------------------
해결: event time(가변) 대신 append/ingestion time(불변) 으로 키 생성
```

**해법 — 불변 시각 속성 사용:**
- event time 대신 **append time**(스트리밍 브로커에 물리적으로 기록된 시각) 같은 불변 값을 키에 사용.
- late 이벤트가 와도 append time 은 안 변하므로 키가 유지됨.
- Apache Kafka 에서는 `append time`, Amazon Kinesis Data Streams 에서는 `approximate arrival timestamp` 라 부름.
- 정적 데이터 스토어(data-at-rest)에서는 `added/ingestion/insertion time` 등으로 불리며, 보통 "현재 시각" 기본값으로 구현됨.

이 전략은 **key-value 스토어뿐 아니라 파일/파티션/테이블 컨테이너에도** 적용됨.
예 — 일일 배치가 실행 시각으로 파일명을 지으면(`20_11_2024`, `21_11_2024` …), replay 해도 항상 같은 파일 하나만 생성됨.
핵심은 동일 — **불변 속성으로 이름(키)을 짓는 것.**

#### 고려사항 (Consequences)

단순하지만, 대부분 **데이터베이스 종류** 와 얽힌 함정이 있음.

- **Database dependent — DB 종류에 따라 적용 가능성이 다름**
  - **key-based(NoSQL)** — Cassandra, ScyllaDB, HBase 등에서 가장 잘 맞음(같은 키 = 덮어쓰기).
  - **관계형 DB** — 같은 PK 로 insert 하면 **덮어쓰기가 아니라 에러**. 그래서 write 가 `INSERT` 대신 `MERGE` 가 되어 Merger 패턴만큼 복잡해짐.
  - **Apache Kafka** — 키를 지원하지만 append-only 로그라 **삽입 시점에 dedup 하지 않음**. 대신 **비동기 compaction** 으로 나중에 정리 → 그 사이 컨슈머는 중복을 볼 수 있음(단 같은 키라 구분은 쉬움).

- **Mutable data source — compaction 이 첫 이벤트를 지우면 키가 흔들림**
  - Kafka compaction 은 너무 오래된 이벤트를 제거하도록 설정 가능.
  - 키 생성에 쓴 **첫 이벤트가 compaction 으로 삭제** 된 뒤 잡을 재시작하면, 로그의 다음 레코드를 집어 키가 달라짐 → 멱등성 깨짐.
  - 단, 데이터 자체가 바뀐 것이므로 레코드 모양이 달라진 만큼 다른 키를 쓰는 게 합리적이기도 함.

> **사이드바 — Kafka 의 timestamp 속성**
> Kafka 프로듀서가 append time 을 외부에서 지정할 수도 있지만, 패턴 관점에서는 **브로커가 완전히 통제하는 값** 보다 덜 신뢰됨.
> 토픽 설정은 `log.message.timestamp.type` 속성으로 확인 가능.

#### 구현 예시 (Examples)

**예시 1 — ScyllaDB 세션 테이블 (복합 PK)**

`session_id` + `user_id` 를 유일 키로 → 멱등 세션 생성의 보증:
```sql
CREATE TABLE sessions (
  session_id BIGINT,
  user_id BIGINT,
  pages LIST<TEXT>,
  ingestion_time TIMESTAMP,
  PRIMARY KEY(session_id, user_id));
```

**예시 2 — ingestion time 기반 윈도우 (오름차순 정렬)**

오름차순이라 새 데이터가 추가돼도 "첫 기록 활동" 이 흔들리지 않음:
```sql
SELECT ... OVER (PARTITION BY user_id ORDER BY ingestion_time ASC, visit_time ASC)
```

**예시 3 — Spark Structured Streaming: append time(timestamp) 추출**

Kafka 레코드의 `timestamp`(= append time)를 키 생성용으로 보존:
```python
(input_data.selectExpr('CAST(value AS STRING)', 'timestamp').select(F.from_json(
    F.col('value'), 'user_id LONG, page STRING, event_time TIMESTAMP')
    .alias('visit'), F.col('timestamp'))
  .selectExpr('visit.*', 'UNIX_TIMESTAMP(timestamp) AS append_time')
  .withWatermark('event_time', '10 seconds').groupBy(F.col('user_id')))
```

**예시 4 — 멱등 세션 ID 생성 (만료 상태 처리)**

상태에 누적된 **최소 append time** 을 해시해 `session_id` 로 사용 → 재처리해도 동일:
```python
def map_visit_to_session(user_tuple, input_rows, current_state):
    user_id = user_tuple[0]
    if current_state.hasTimedOut:
        min_append_time, pages, = current_state.get
        session_to_return = {
            'user_id': [user_id],
            'session_id': [hash(str(min_append_time))],   # 불변 append time 으로 ID 생성
            'pages': [pages]
        }
    else:
        # 아래 누적 로직으로 진행
        ...
```

**예시 5 — append time 상태 누적**

각 윈도우의 **가장 이른 append time** 을 세션 첫 버전에 고정. Kafka append time 은 증가하므로 이후엔 첫 값을 그대로 유지:
```python
else:
    data_min_append_time = 0
    for input_df_for_group in input_rows:
        data_min_append_time = int(input_df_for_group['append_time'].min()) * 1000
    if current_state.exists:
        min_append_time, current_pages, = current_state.get
        visited_pages = current_pages + pages
        current_state.update((min_append_time, visited_pages,))   # 기존 append time 유지
    else:
        current_state.update((data_min_append_time, pages,))       # 첫 값 고정
```

다른 브로커도 **속성명만 다를 뿐** 동일하게 구현됨. 예 — Amazon Kinesis 는 읽기 부분만 바꿔 `approximateArrivalTimestamp` 를 `append_time` 으로 매핑:
```python
(spark_session.readStream.format("kinesis") # ...
  .load().selectExpr("CAST(data AS STRING)",
    "approximateArrivalTimestamp AS append_time"))
```

| 데이터 스토어 | write 동작 | 주의사항 |
|--------------|-----------|----------|
| NoSQL (Cassandra/ScyllaDB/HBase) | 같은 키 = 덮어쓰기 | 가장 자연스러운 적용 대상 |
| 관계형 DB | 같은 PK insert = 에러 | `INSERT` 대신 `MERGE` 필요 → 복잡도 증가 |
| Apache Kafka | append-only, 삽입 시 dedup 없음 | 비동기 compaction 전까지 중복 노출(같은 키) |
| 파일/파티션/테이블 | 실행 시각 등 불변값으로 명명 | replay 해도 같은 이름 하나만 생성 |

> **트러블 로그** — 세션/엔티티 키를 event time(가변)으로 만들면 late data 한 건에 멱등성이 통째로 깨짐.
> 예: 사용자 세션 ID 를 `hash(user_id + first_event_time)` 으로 만들어 두고 운영 중,
> 잡이 OOM 으로 재시작된 사이 09:55 짜리 지연 이벤트가 도착 → 같은 사용자의 first_event_time 이 10:00 에서 09:55 로 바뀌면서
> **세션 ID 가 달라져 한 세션이 두 개로 쪼개짐**. 다음 날 세션 수 지표가 평소의 1.3배로 튀어서야 발견됨.
> **키 생성에는 반드시 append/ingestion time 같은 불변 속성을 쓰고, Kafka 라면 compaction 이 첫 이벤트를 지우지 않도록 retention 을 키 생성 윈도우보다 길게 둘 것.**

---

### 3-2. 패턴 #21: 트랜잭션 기반 작성자 (Transactional Writer)

DB의 **트랜잭션(all-or-nothing)** 을 활용해, **커밋 전까지** 진행 중인 변경분이 컨슈머에게 보이지 않게 함으로써 부분/중복 데이터 노출을 막는 패턴.

#### 상황 (Problem)

**책의 use case:**
- 비용 절감을 위해 클라우드의 **유휴 컴퓨팅(spot/preemptible 등)** 위에서 배치 잡을 돌려 인프라 비용 **60%** 절감.
- 그런데 다운스트림 컨슈머가 **데이터 품질** 불만을 제기.
- 클라우드가 노드를 회수할 때마다 실행 중 task 들이 실패 → 다른 노드에서 재시도 → **데이터를 다시 write** → 컨슈머가 **중복·불완전 레코드** 를 봄.
- **결정적 제약**: 잡이 **부분 데이터(partial data)를 절대 노출하지 않아야** 함.

#### 해결 (Solution)

트랜잭션을 활용. 커밋되지 않은(in-progress) 변경은 다운스트림 reader 에게 보이지 않게 함.

**3단계로 구성:**
1. **트랜잭션 초기화** — 명시적(`START TRANSACTION`/`BEGIN`) 또는 처리 계층이 알아서 여는 암묵적 방식.
2. **데이터 write** — 기록된 변경은 DB에 추가되지만 **트랜잭션 범위 안에서만 private**.
3. **commit** — 새 레코드를 공개. 문제가 생기면 commit 대신 **rollback** 으로 폐기.

```
Transactional Writer 흐름
--------------------------------------------------------------
  [BEGIN] → [write (private)] → 정상 → [COMMIT] → 컨슈머에 공개
                                  └ 오류 → [ROLLBACK] → 폐기
--------------------------------------------------------------
커밋 전까지 컨슈머는 아무 것도 못 봄 (read uncommitted 제외)
```

**처리 모델별 구현이 갈림:**
- **standalone / ELT (DW 직접 처리, BigQuery·Redshift·Snowflake)** — 트랜잭션이 보통 **선언적이고 DB가 완전 관리**. 내부적으로 분산 처리 가능.
- **분산 데이터 처리(ETL)** — 여러 task 가 같은 출력에 병렬 write. 두 가지 구현:
  - **로컬(task 기반) 트랜잭션** — task 마다 독립 트랜잭션. 잡 재시도가 없을 때만 안전(아래 고려사항 참고).
  - **잡 전체 트랜잭션** — 잡이 시작 전 트랜잭션을 열고 모든 task 완료 후 commit. 더 강한 보증, 하지만 더 어려움.
    예: Spark + Delta Lake 는 writer 가 commit log 디렉토리에 새 엔트리를 만들 때 commit 됨. 이 단계가 실패하면 데이터 파일은 남아 따로 치워야 함.

**멱등성 범위 주의** — all-or-nothing 은 **현재 run 안에서만** 멱등.
백필하면 writer 가 **새 트랜잭션** 을 열어 같은 레코드를 또 insert 함. 즉 멱등성은 트랜잭션 단위로만 보장됨.

> **사이드바 — Read Uncommitted = dirty reads**
> writer 가 트랜잭션을 써도, reader 가 격리 수준을 **read uncommitted** 로 설정하면 커밋 안 된 레코드를 볼 수 있음.
> 롤백될 수도 있는 레코드를 읽는 이 현상이 **dirty read**.

**지원 기술** — 모던 테이블 파일 포맷(Delta Lake, Iceberg, Hudi), 스트리밍 브로커(Kafka), DW(Redshift, BigQuery), RDBMS(PostgreSQL, MySQL, Oracle, SQL Server).
단, 분산 처리 도구와의 통합은 제각각 — 테이블 파일 포맷은 Flink/Spark 가 잘 지원하지만, **Kafka 트랜잭션 프로듀서는 Flink 에서만** 가능.

#### 고려사항 (Consequences)

다른 패턴보다 구현이 쉬움(적절한 명령만 호출하면 됨). 그래도 함정이 있음.

- **Commit step — 커밋이 만드는 지연(latency)**
  - 비트랜잭션 write 와 달리 트랜잭션은 **열기/커밋** 두 단계 + 양쪽에서 충돌 해소가 추가됨.
  - 예: JSON/CSV 파일은 만들자마자 보이지만, Delta Lake 파일은 **commit logfile 이 생긴 뒤에야** 보임 → 컨슈머는 **가장 느린 task 가 끝날 때까지** 대기.
  - 이 조율 오버헤드는 "완전한 데이터셋만 노출" 하기 위한 필요 비용.

- **Distributed processing — 분산 프레임워크 지원이 제각각**
  - 트랜잭션 지원이 전역적이지 않음. 예: 인기 있는 **Kafka 트랜잭션이 Spark 에서는 미지원** → 패턴 적용 범위가 크게 줄어듦.

- **Idempotency scope — 멱등성은 트랜잭션 안으로 한정**
  - 분산 프레임워크가 **로컬(task 기반) 트랜잭션** 을 쓰면서 이미 커밋된 task 를 추적하지 않으면, **잡 재시작 시 커밋된 트랜잭션의 데이터까지 다시 write**.
  - 백필도 마찬가지 — 재처리가 새 트랜잭션을 열어 같은 레코드를 또 추가함.

#### 구현 예시 (Examples)

**예시 1 — 한 트랜잭션 안의 두 MERGE (부분 성공 불가)**

두 번째 파일에 컬럼 길이 초과 행이 있어 두 번째 `MERGE` 가 실패 → 첫 번째도 commit 되지 않음(SQL runner 가 commit 단계에 도달 못 함):
```sql
CREATE TEMPORARY TABLE changed_devices_file1 (LIKE dedp.devices);
COPY changed_devices_file1 FROM '/data_to_load/dataset_1.csv' CSV DELIMITER ';' HEADER;
MERGE INTO dedp.devices AS d USING changed_devices_file1 AS c_d
-- ... 생략

CREATE TEMPORARY TABLE changed_devices_file2 (LIKE dedp.devices);
COPY changed_devices_file2 FROM '/data_to_load/dataset_too_long_type.csv' CSV DELIMITER ';' HEADER;
MERGE INTO dedp.devices AS d USING changed_devices_file2 AS c_d
-- ... 생략

COMMIT;   -- 여기에 도달하지 못하면 첫 MERGE 도 반영 안 됨 = all-or-nothing
```

**예시 2 — Apache Flink: 트랜잭션 Kafka 프로듀서**

`EXACTLY_ONCE` 전송 보증 + `transaction.timeout.ms` 설정이 핵심:
```python
kafka_sink_valid_data = (KafkaSink.builder().set_bootstrap_servers("localhost:9094")
    .set_record_serializer(KafkaRecordSerializationSchema.builder()
        .set_topic('reduced_visits')
        .set_value_serialization_schema(SimpleStringSchema())
        .build())
    .set_delivery_guarantee(DeliveryGuarantee.EXACTLY_ONCE)   # 트랜잭션 전송
    .set_property('transaction.timeout.ms', str(1 * 60 * 1000))  # checkpoint 보다 길게
    .build())
```

> exactly-once 전송은 Flink **checkpointing** 에 의존하며 시간이 걸릴 수 있음.
> timeout 이 너무 짧아 checkpoint 가 그보다 오래 걸리면, Flink 가 **트랜잭션 만료로 commit 에 실패**함.

| 처리 모델 | 트랜잭션 방식 | 주의사항 |
|----------|--------------|----------|
| standalone / ELT (DW) | DB 관리 선언적 트랜잭션 | 내부적으로 분산 처리 가능, 구현 단순 |
| 분산 ETL — 로컬(task) 트랜잭션 | task 별 독립 commit | 잡 재시도 시 커밋된 task 가 재기록됨 |
| 분산 ETL — 잡 전체 트랜잭션 | 잡 단위 commit | 강한 보증, 구현 어려움(commit log 실패 시 파일 정리 필요) |
| Kafka + Flink | exactly-once 프로듀서 | `transaction.timeout.ms` > checkpoint 소요시간 |

> **트러블 로그** — 트랜잭션을 썼다고 멱등성이 영구 보장된다고 믿으면 백필에서 그대로 중복이 쌓임.
> 예: Spark + Delta Lake 잡이 task 기반 로컬 트랜잭션으로 동작하는데, spot 노드 회수로 잡이 통째로 재시작되면
> **이미 commit 된 task 의 출력이 다시 write** 되어 같은 레코드가 2배로 들어감 — 트랜잭션 단위로는 "정상" 이라 에러도 안 남.
> 또 하나 — Flink Kafka 싱크의 `transaction.timeout.ms` 를 30초로 두고 checkpoint 가 평소 45초 걸리는 잡을 돌리면,
> 트랜잭션이 만료돼 commit 이 실패하고 출력 토픽이 비어버림.
> **멱등성은 트랜잭션 범위 안으로만 보장됨을 전제로 백필/재시작 시 중복 처리를 별도 설계하고(예: 잡 전체 트랜잭션 + 처리 이력 추적), Flink 라면 transaction.timeout.ms 를 checkpoint 소요시간보다 넉넉히 길게 잡을 것.**
