# 데이터 엔지니어링 디자인 패턴 - 멱등성 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 4 | 실무 데이터 엔지니어링 관점 정리

---

## 목차

1. [덮어쓰기 (Overwriting)](#1-덮어쓰기-overwriting)
   - 패턴 #16: 빠른 메타데이터 정리기 (Fast Metadata Cleaner)
   - 패턴 #17: 데이터 덮어쓰기 (Data Overwrite)

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

  본 문서가 다루는 범위: 4.1 Overwriting (#16, #17)
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
- 데이터셋이 더 이상 single place 에 살지 않음 → 사용자가 "내부 분할 구조" 를 몰라도 되게 view 같은 logical 구조로 통합 노출 필요.

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
