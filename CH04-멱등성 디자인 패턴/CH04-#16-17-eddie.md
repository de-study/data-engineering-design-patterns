# CH04. 멱등성 디자인 패턴 

> 3장에서 체크포인트만으로는 중복을 막을 수 없다고 했어. 
> 그 중복을 막는 게 바로 Idempotency야.

한 줄 정의:

> 몇 번을 실행해도 항상 같은 결과가 나오는 성질

수학의 절댓값 함수가 대표 예시야.

```
abs(-1)           = 1
abs(abs(-1))      = 1
abs(abs(abs(-1))) = 1  ← 몇 번을 해도 결과가 같음
```

데이터 파이프라인에 적용하면 이거야.

```
결제 파이프라인을 1번 실행해도 → Delta Lake에 결제 데이터 12,000건
결제 파이프라인을 3번 실행해도 → Delta Lake에 결제 데이터 12,000건
                                   (중복 없음)
```

# 4.1 덮어쓰기(Overwriting)

## 패턴 #16 Fast Metadata Cleaner

### 4.1.1 Problem — DELETE가 느려진다

#### 배경 — 왜 이 패턴이 생겼나

idempotency를 구현하는 가장 직관적인 방법은 이거다.

~~~
1. 이전 실행이 넣은 데이터를 DELETE로 지운다
2. 새 데이터를 INSERT한다
~~~

처음엔 잘 돼. 근데 테이블이 커질수록 DELETE가 느려진다.

DELETE는 두 단계로 동작한다.

~~~
1단계: 테이블 전체를 스캔해서 조건에 맞는 row 찾기
2단계: 찾은 row가 있는 데이터 파일 전부 다시 쓰기
→ 테이블이 클수록 선형으로 느려짐
~~~

---

#### 실무 시나리오 — 결제 트랜잭션 배치 파이프라인

**등장인물 정의**

- Airflow DAG: 매일 새벽 2시 실행되는 배치 잡. 결제 데이터를 처리해서 PostgreSQL에 적재
- PostgreSQL: 결제 데이터가 최종 적재되는 저장소

idempotency를 위해 이렇게 구현했다.

~~~
1. DELETE FROM payments WHERE date = '2026-05-23'
2. INSERT INTO payments SELECT ...
~~~

3개월이 지나면서 DELETE 소요시간이 급격히 증가한다.

~~~
1일차:  DELETE 소요시간 → 2초
30일차: DELETE 소요시간 → 45초
90일차: DELETE 소요시간 → 8분
~~~

DELETE는 WHERE 조건이 있어도 테이블 전체를 스캔해야 지울 row를 찾을 수 있다.
테이블이 클수록 느려지는 구조다.

---

#### 엔지니어 독백

처음엔 진짜 괜찮아. 데이터 몇 백만 건일 때는 DELETE가 금방 끝나거든.
근데 6개월, 1년 지나면서 테이블이 수십억 건이 되면 그때부터 문제야.

새벽 2시에 시작한 Airflow DAG가 DELETE 하나 때문에 새벽 4시까지 안 끝나는 거야.
그러면 다음 DAG 실행이 밀리고, 아침에 출근한 데이터 분석팀이 "어제 데이터 왜 없어요?" 슬랙 보내는 거야.
이게 단순히 느린 게 아니야. 비즈니스 임팩트야.

---

### 4.1.2 Solution — TRUNCATE/DROP으로 메타데이터 레이어에서 지운다

#### DELETE vs TRUNCATE 동작 차이

~~~
DELETE FROM payments WHERE date = '2026-05-23'
→ 1단계: 테이블 전체 스캔해서 조건에 맞는 row 찾기
→ 2단계: 찾은 row가 있는 데이터 파일 전부 다시 쓰기
→ 테이블이 클수록 느려짐

TRUNCATE TABLE payments_2026_week21
→ 메타데이터만 수정 (테이블이 비어있다고 표시)
→ 데이터 파일 스캔 없음
→ 테이블 크기와 무관하게 항상 빠름
~~~

TRUNCATE는 데이터 파일을 건드리지 않는다.
메타데이터 레이어에서 "이 테이블 비어있음"이라고 표시만 바꾸는 것이다.
그래서 Fast Metadata Cleaner라는 이름이 붙었다.

---

#### 핵심 구조 변경 — 테이블을 쪼갠다

TRUNCATE의 문제는 테이블 전체를 비워버린다는 것이다. 하루치만 지울 수가 없다.
그래서 이 패턴은 테이블 자체를 단위별로 쪼개고 View로 하나처럼 노출한다.

~~~
기존: payments 테이블 하나에 전체 데이터

변경:
payments_2026_week20  (5/11~5/17)
payments_2026_week21  (5/18~5/24)  ← 이번 주 테이블만 TRUNCATE
payments_2026_week22  (5/25~5/31)
         ↓
payments (View) = week20 UNION ALL week21 UNION ALL week22
~~~

재실행이 필요하면 해당 주 테이블만 TRUNCATE하고 다시 INSERT한다.
다른 주 데이터는 건드리지 않는다.

---

#### Airflow DAG 처리 흐름

~~~
Airflow DAG 매일 실행
        ↓
[BranchPythonOperator]
오늘이 월요일인가?
    ↓ YES                    ↓ NO
새 주차 테이블 생성         현재 주차 테이블 TRUNCATE
View 갱신
        ↓
결제 데이터 INSERT
~~~

---

#### TRUNCATE vs DROP 선택

| 방식 | 동작 | 장점 | 단점 |
|---|---|---|---|
| TRUNCATE TABLE | 테이블 구조 유지, 데이터만 삭제 | View 연결 유지됨 | 테이블이 미리 존재해야 함 |
| DROP TABLE | 테이블 자체 삭제 후 재생성 | 완전 초기화 | View가 DROP 순간 깨질 수 있음 |

실무에서는 TRUNCATE가 더 안전하다.
DROP은 View가 테이블을 참조하는 순간 사용자가 오류를 볼 수 있다.

---

#### Idempotency Granularity — 멱등성 세분화

테이블을 쪼개는 단위가 곧 재실행할 수 있는 최소 단위다.

~~~
일별 세분화:
payments_2026_05_23  ← 하루치만 TRUNCATE 후 재실행 가능
→ Idempotency Granularity = 1일

주별 세분화:
payments_2026_week21 ← 이번 주 전체를 TRUNCATE 후 재실행
→ Idempotency Granularity = 1주
→ 수요일 하루만 잘못돼도 7일치 전체 재처리
~~~

세분화 단위 선택 기준:

| 세분화 단위 | 장점 | 단점 |
|---|---|---|
| 일별 | backfill 범위 최소화 | 테이블 수 많음 (1년 = 365개) |
| 주별 | 테이블 수 적음 | backfill 시 최대 7일치 재처리 |
| 월별 | 테이블 수 더 적음 | backfill 시 최대 31일치 재처리 |

---

### 4.1.3 Consequences — 트레이드오프 3가지 + Schema Evolution

#### 1. Granularity = Backfilling Boundary

idempotency 세분화 단위가 backfill 최소 단위를 결정한다.

~~~
주별 세분화 상황에서 5월 20일(수) 데이터만 잘못됨

→ payments_2026_week21 전체를 TRUNCATE
→ 5월 18일~24일 일주일 전체 재처리
→ 하루치 문제인데 7일치 재처리
~~~

더 심각한 케이스: 특정 가맹점 데이터만 잘못된 경우

~~~
merchant_id = 'M001' 데이터만 잘못됨
→ TRUNCATE는 테이블 단위로만 동작
→ M001 데이터만 골라서 지울 수 없음
→ 주 전체를 TRUNCATE하고 전체 재처리
~~~

---

#### 2. Metadata Limits — 테이블/파티션 한도

| 데이터 웨어하우스 | 한도 |
|---|---|
| GCP BigQuery | 파티션 4,000개 |
| AWS Redshift | 테이블 200,000개 |

일별 세분화 10개 파이프라인이면 1년에 3,650개 테이블이 생긴다.

해결책은 Freezing이다. 일정 기간이 지난 테이블은 변경 가능성이 없다고 판단되면
더 큰 단위로 합쳐버린다.

~~~
payments_2026_05_01 ~ payments_2026_05_31
→ 6월이 되면 변경 가능성 없음
→ payments_2026_05 로 합쳐버림 (freezing)
→ 테이블 수 31개 → 1개로 줄어듦
~~~

---

#### 3. Data Exposition Layer — View 관리 복잡도

데이터가 여러 테이블에 쪼개져 있으니 사용자에게는 View로 하나처럼 보여줘야 한다.

~~~
payments (View)
= payments_2026_week19
  UNION ALL payments_2026_week20
  UNION ALL payments_2026_week21
~~~

새 테이블이 생길 때마다 View를 갱신해야 한다.
테이블을 DROP할 때는 View에서 먼저 제거해야 한다.

~~~
잘못된 순서:
DROP TABLE payments_2026_week21  ← View가 이 테이블 참조 중
→ 사용자가 payments View 조회 시 오류 발생

올바른 순서:
1. View에서 payments_2026_week21 제거
2. DROP TABLE payments_2026_week21
~~~

---

#### 4. Schema Evolution — 스키마 변경 시 동작

| 케이스 | 동작 | 이 패턴의 도움 여부 |
|---|---|---|
| Optional 컬럼 추가 | 기존 테이블 수동 ALTER TABLE 필요 | 도움 안 됨. 별도 마이그레이션 필요 |
| Required 컬럼 추가 | 어차피 전체 재처리 필요 | 재처리 시 TRUNCATE + INSERT로 자동 해결 |

---

#### 이 패턴을 쓸 수 있는 저장소

| 저장소 | 사용 가능 여부 |
|---|---|
| Data Warehouse (BigQuery, Redshift) | 가능 |
| Lakehouse (Delta Lake, Iceberg) | 가능 |
| 관계형 DB (PostgreSQL) | 가능 |
| Object Store (S3 단독) | 불가능 → Data Overwrite 패턴 사용 |

---

### 4.1.4 Examples — Airflow DAG 구현

**등장인물 정의**

- Airflow DAG: 매일 실행되는 배치 잡. 결제 데이터를 처리해서 PostgreSQL에 적재
- PostgreSQL: 결제 데이터가 최종 적재되는 저장소. 주차별 테이블 + View로 구성
- BranchPythonOperator: Airflow의 분기 오퍼레이터. 오늘이 월요일인지 판단해서 다른 태스크로 분기

---

**Step 1 — 분기 판단: BranchPythonOperator**

~~~
def retrieve_path_for_table_creation(**context):
    ex_date = context['execution_date']       # Airflow가 주입하는 실행 날짜 (내가 정하는 값 아님)

    should_create_table = (
        ex_date.day_of_week == 1              # 월요일 = 새 주 시작
        or ex_date.day_of_year == 1           # 1월 1일 = 새 해 시작
    )

    # 반환한 태스크 ID로 Airflow가 분기
    # 나머지 태스크는 자동으로 skip
    return 'create_weekly_table' if should_create_table else 'truncate_weekly_table'

branch_task = BranchPythonOperator(
    task_id='check_if_new_week',
    python_callable=retrieve_path_for_table_creation
)
~~~

- `execution_date`: Airflow가 자동으로 주입하는 고정 변수. 이 DAG가 처리하는 날짜
- 반환값이 태스크 ID 문자열 → Airflow가 해당 태스크로 분기, 나머지는 skip

---

**Step 2 — 새 주차 테이블 생성 + View 갱신 (월요일)**

~~~
create_weekly_table = PostgresOperator(
    task_id='create_weekly_table',
    sql='''
        CREATE TABLE IF NOT EXISTS
        payments_{{ execution_date.strftime("%Y_week%V") }}
        (LIKE payments_template INCLUDING ALL);
        -- payments_2026_week21 같은 테이블이 자동 생성됨
    '''
)

recreate_view = PostgresOperator(
    task_id='recreate_view',
    sql='''
        CREATE OR REPLACE VIEW payments AS
            SELECT * FROM payments_2026_week20
            UNION ALL
            SELECT * FROM payments_2026_week21;
    '''
)
~~~

---

**Step 3 — 현재 주차 테이블 TRUNCATE (월요일 외)**

~~~
truncate_weekly_table = PostgresOperator(
    task_id='truncate_weekly_table',
    sql='''
        TRUNCATE TABLE
        payments_{{ execution_date.strftime("%Y_week%V") }};
        -- 테이블 스캔 없이 메타데이터만 수정 → 빠름
    '''
)
~~~

---

**Step 4 — 데이터 INSERT (공통)**

~~~
insert_payments = PostgresOperator(
    task_id='insert_payments',
    sql='''
        INSERT INTO payments_{{ execution_date.strftime("%Y_week%V") }}
        SELECT * FROM payments_raw
        WHERE DATE(event_time) = '{{ ds }}';
    '''
)

# DAG 전체 흐름 연결
branch_task >> [create_weekly_table, truncate_weekly_table]
create_weekly_table >> recreate_view >> insert_payments
truncate_weekly_table >> insert_payments
~~~

---

**재실행(backfill) 시 동작**

~~~
5월 20일(수) 데이터 잘못됨 → Airflow에서 5월 20일 DAG 재실행

→ 오늘이 월요일 아님
→ truncate_weekly_table 실행
→ payments_2026_week21 전체 TRUNCATE
→ 5월 18일~24일 데이터 전체 다시 INSERT

주의: 5월 20일 하루치 문제인데
      5월 18일~24일 7일치 전체 재처리됨 (Backfilling Boundary)
~~~

---

#### 엔지니어 독백

BranchPythonOperator 처음 보면 복잡해 보여.
근데 하는 일은 단순해.
"오늘이 새 주의 시작이면 테이블 새로 만들고, 아니면 기존 테이블 비우고 다시 써." 이게 전부야.

핵심은 `execution_date`야. Airflow가 자동으로 주입해주는 변수인데,
이걸 기준으로 테이블명을 만들어.
`payments_2026_week21` 같은 테이블명이 내가 직접 정하는 게 아니라
Airflow 실행 날짜에서 자동으로 생성돼.
그래서 재실행해도 항상 같은 테이블명이 나오고,
그 테이블을 TRUNCATE하고 다시 INSERT하니까 idempotent한 거야.

View 갱신 빼먹으면 새 주차 테이블 만들었는데
사용자가 payments View 조회할 때 새 데이터가 안 보여.
이 실수 진짜 많이 해.
`create_weekly_table >> recreate_view` 순서를 반드시 DAG에 명시해야 해.

세분화 단위 결정할 때 "비즈니스 팀이 요청하는 backfill 최소 단위가 뭐냐"를 반드시 먼저 물어봐야 해.
그게 일이면 일별, 주면 주별로 잡는 거야.
나중에 바꾸려면 테이블 전체를 재구성해야 해.
처음 설계가 진짜 중요한 이유야.



### 4.2 패턴 #17 Data Overwrite

### 4.2.1 Problem — Object Store에는 TRUNCATE가 없다

#### Fast Metadata Cleaner와의 차이

Fast Metadata Cleaner는 TRUNCATE/DROP 명령어가 있는 저장소에서만 동작한다.
S3 같은 Object Store는 TRUNCATE/DROP 명령어가 없다.
그래서 Object Store 기반 파이프라인은 다른 방식이 필요하다.

---

#### 실무 시나리오 — 결제 트랜잭션 배치 파이프라인

**등장인물 정의**

- Airflow DAG: 매일 실행되는 배치 잡. 결제 데이터를 처리해서 S3에 Parquet로 적재하는 Spark 배치 잡
- S3: 결제 데이터가 적재되는 Object Store. TRUNCATE/DROP 명령어 없음
- Spark 배치 잡: Airflow DAG가 실행하는 데이터 처리 잡

파이프라인이 backfill 실행될 때마다 S3에 중복 데이터가 쌓인다.

~~~
최초 실행:
s3://bucket/payments/date=2026-05-23/part-001.parquet  (12,000건)

backfill 재실행:
s3://bucket/payments/date=2026-05-23/part-001.parquet  (12,000건)
s3://bucket/payments/date=2026-05-23/part-002.parquet  (12,000건)  ← 중복
~~~

TRUNCATE가 없으니까 기존 파일을 지우지 않고 새 파일이 추가되는 거야.

---

### 4.2.2 Solution — 쓰기 전에 덮어쓴다

#### 핵심 아이디어

TRUNCATE 대신 쓰기 작업 자체에 "기존 데이터를 먼저 지우고 쓴다" 옵션을 붙이는 것이다.

구현 방식은 3가지다.

---

#### 방식 1 — Spark 배치 잡: overwrite 모드

~~~
# Spark 배치 잡에서 S3에 적재할 때
input_data.write \
    .mode('overwrite') \            # 기존 파일 삭제 후 새로 씀
    .partitionBy('date') \
    .parquet('s3://bucket/payments/')

# Delta Lake에서 특정 파티션만 덮어쓰기
input_data.write \
    .mode('overwrite') \
    .option('replaceWhere', "date = '2026-05-23'") \  # 해당 날짜만 덮어씀
    .format('delta') \
    .save('s3://bucket/payments/')
~~~

- `mode('overwrite')`: Spark가 쓰기 전에 기존 파일을 먼저 삭제하고 새로 씀
- `replaceWhere`: Delta Lake 전용. 조건에 맞는 파티션만 선택적으로 덮어씀. 다른 날짜 데이터는 건드리지 않음

---

#### 방식 2 — SQL: INSERT OVERWRITE

~~~
-- 테이블 전체를 devices_staging 내용으로 덮어씀
INSERT OVERWRITE INTO payments
SELECT * FROM payments_staging
WHERE date = '2026-05-23';
~~~

- Databricks, Snowflake에서 지원
- DELETE + INSERT를 한 번에 처리
- 단, 조건으로 특정 row만 골라서 덮어쓰는 것은 불가능. 테이블 전체가 대상

---

#### 방식 3 — BigQuery: writeDisposition

~~~
bq load dedp.payments \
    gs://bucket/payments_20260523.csv \
    ./schema.json \
    --replace=true    # 기존 데이터 전부 삭제 후 새 데이터 적재
~~~

- `--replace=true`: BigQuery 내부적으로 WRITE_TRUNCATE 동작
- 테이블 전체를 비우고 새 데이터를 넣음

---

#### 방식 선택 기준

| 상황 | 권장 방식 |
|---|---|
| Spark + S3 (Object Store) | `mode('overwrite')` |
| Spark + Delta Lake (특정 파티션만) | `replaceWhere` |
| Databricks / Snowflake SQL | `INSERT OVERWRITE` |
| BigQuery | `--replace=true` (bq load) |

---

### 4.2.3 Consequences — 트레이드오프 2가지

#### 1. Data Overhead — 데이터가 클수록 느려진다

Fast Metadata Cleaner는 메타데이터만 건드려서 빨랐어.
Data Overwrite는 실제 데이터 파일을 삭제하고 다시 쓰는 작업이야.

~~~
파티셔닝 없이 전체 덮어쓰기:
payments 테이블 전체 (90일치) → 전부 삭제 후 재작성
→ 데이터가 쌓일수록 점점 느려짐

파티셔닝 + 선택적 덮어쓰기 (replaceWhere):
date=2026-05-23 파티션만 → 해당 파티션만 삭제 후 재작성
→ 데이터 크기에 비교적 덜 영향받음
~~~

파티셔닝으로 덮어쓸 범위를 줄이는 게 핵심이야.

---

#### 2. Vacuum 필요 — 삭제해도 디스크에 남는다

Delta Lake, Apache Iceberg 같은 Table File Format과 일부 관계형 DB는
덮어쓴 후에도 기존 데이터 파일이 디스크에 남아있어.

Time Travel 기능을 위해 의도적으로 보관하는 거야.

~~~
Delta Lake 예시:
overwrite 실행 후 → 기존 파일은 숨겨지고 새 파일이 활성화됨
                    기존 파일은 디스크에 존재 (사용자는 못 읽음)

→ VACUUM 명령어로 실제 삭제
VACUUM delta.`s3://bucket/payments/` RETAIN 7 HOURS;
~~~

VACUUM을 안 하면 S3 비용이 계속 누적돼. 주기적으로 실행해야 해.

---

### 4.2.4 Examples — 실제 구현

#### Spark 배치 잡 + S3 (Object Store)

~~~
# parquet 파일에서 읽기
input_data = spark.read.parquet('s3://bucket/payments/raw/date=2026-05-23/')

# S3에 overwrite 모드로 적재
# 기존 date=2026-05-23 파티션 파일들 삭제 후 새로 씀
input_data.write \
    .mode('overwrite') \
    .partitionBy('date') \
    .parquet('s3://bucket/payments/valid/')
~~~

---

#### Spark 배치 잡 + Delta Lake (선택적 덮어쓰기)

~~~
# parquet 파일에서 읽기
input_data = spark.read.parquet('s3://bucket/payments/raw/date=2026-05-23/')

# date=2026-05-23 파티션만 덮어씀
# 다른 날짜 데이터는 건드리지 않음  ← Fast Metadata Cleaner와 유사한 효과
input_data.write \
    .mode('overwrite') \
    .option('replaceWhere', "date = '2026-05-23'") \
    .format('delta') \
    .save('s3://bucket/payments/valid/')

# 주기적으로 VACUUM 실행 (오래된 파일 실제 삭제)
from delta.tables import DeltaTable
DeltaTable.forPath(spark, 's3://bucket/payments/valid/') \
    .vacuum(retentionHours=168)   # 7일치 보관 후 삭제
~~~

---

#### Fast Metadata Cleaner vs Data Overwrite 비교

| 항목 | Fast Metadata Cleaner | Data Overwrite |
|---|---|---|
| 사용 가능 저장소 | DW, Lakehouse, RDB | Object Store 포함 모두 |
| 삭제 방식 | 메타데이터 조작 (빠름) | 실제 파일 삭제 (느림) |
| 선택적 삭제 | 테이블 단위만 가능 | 파티션 단위 가능 (replaceWhere) |
| Vacuum 필요 | 불필요 | Delta Lake 등은 필요 |
| 오케스트레이션 의존도 | Airflow DAG 필수 | Spark 설정으로 대부분 해결 |

---

#### 엔지니어 독백

S3 기반 파이프라인에서 idempotency 구현할 때 제일 흔한 실수가
`mode('overwrite')`를 파티셔닝 없이 쓰는 거야.

~~~
# 위험한 코드
input_data.write.mode('overwrite').parquet('s3://bucket/payments/')
→ payments 전체를 오늘 데이터로 덮어씀
→ 어제, 그제 데이터 전부 날아감
~~~

반드시 `partitionBy`를 함께 써서 해당 날짜 파티션만 덮어쓰도록 해야 해.

Delta Lake 쓴다면 `replaceWhere`가 제일 안전해.
조건을 명시적으로 지정하니까 의도치 않게 다른 파티션을 건드릴 위험이 없어.

VACUUM은 Airflow DAG에 주기적으로 넣어놔야 해.
안 하면 S3 비용이 몇 달 지나면서 2배, 3배로 불어나.
나중에 "왜 S3 비용이 이렇게 많이 나와요?" 하는 상황 막으려면 처음부터 넣어놔야 해.
