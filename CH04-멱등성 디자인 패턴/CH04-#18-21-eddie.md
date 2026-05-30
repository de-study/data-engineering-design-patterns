# 4.2 갱신(Update)
- 패턴#18 병합기(Merger)
- 패턴#19 상태저장병합기(Stateful Merger)
- 패턴#20 키 기반 멱등성(Keyed Idempotency)

<br><br><br>
<br><br><br>










# 패턴#18 병합기(Merger)

## (1)문제상황 — 증분데이터만 유입
### 배경 — 왜 이 패턴이 생겼나

- Fast Metadata Cleaner, Data Overwrite →  "전체 데이터를 지우고 다시 쓴다"
- 전체 데이터를 매번 받을 수 없는 케이스가 있어.
- Overwrite 계열 패턴(#16, #17) → 전제가 "매 실행마다 전체 데이터를 받는다"
```
Overwrite 전제:
오늘 실행 → 전체 12,000건 받음 → 기존 데이터 삭제 → 12,000건 INSERT
재실행해도 → 전체 12,000건 받음 → 기존 데이터 삭제 → 12,000건 INSERT
→ 항상 같은 결과 = idempotent
```

- CDC(Change Data Capture) 기반 파이프라인 → 변경된 행만 Kafka로 받음

```
CDC 상황:
기존 테이블: 10,000건 (누적 상태)
오늘 변경분: 10건 (UPDATE 7건 + INSERT 3건)

만약 overwrite 시도하면:
기존 10,000건 삭제 → 변경분 10건만 INSERT
→ 9,990건 데이터 유실
```

- Merge의 필요성 : 전체를 덮어쓰면 기존 데이터가 사라지는 구조. 그래서 merge가 필요해. 

### 실무 시나리오 — 디바이스 정보 CDC 파이프라인

**등장인물 정의**
- Kafka topic: 디바이스 마스터 테이블의 변경분(INSERT/UPDATE)이 유입되는 스트리밍 브로커
- Airflow DAG: 매 시간 실행되는 배치 잡. Kafka에서 변경분을 읽어 Delta Lake 테이블에 반영
- Delta Lake 테이블: 디바이스 정보가 최종 적재되는 저장소. 전체 최신 상태를 유지해야 함

```
Kafka에서 오는 변경분 예시:
- type=laptop, version=1.0 → full_name 변경 (UPDATE)
- type=tablet, version=3.0 → 신규 모델 추가 (INSERT)
```

> 이 변경분을 기존 테이블에 합쳐야 해. 삭제하고 다시 쓸 수 없어.




## (2) 솔루션 — MERGE 연산으로 기존 데이터와 합친다

### 솔루션 컨셉
1. 고유 키(identity) 정의 → 새 데이터와 기존 데이터를 어떤 컬럼으로 구분하고, 조인할지
2. MERGE 연산 → 키가 일치하면 UPDATE, 없으면 INSERT

```
MERGE INTO devices AS d
USING changed_devices AS c_d
  ON c_d.type = d.type AND c_d.version = d.version
WHEN MATCHED THEN
  UPDATE SET full_name = c_d.full_name    ← 기존 row 업데이트
WHEN NOT MATCHED THEN
  INSERT (type, full_name, version)       ← 새 row 삽입
  VALUES (c_d.type, c_d.full_name, c_d.version)
```

- 고유 키: `type + version` 조합 → 이 두 컬럼으로 기존 데이터와 매칭


## (3)결과 — 트레이드오프 4가지
- 고유 키 필수
    - row를 식별할 수 있는 고유 키가 없으면 MERGE 자체가 불가능
    - 키가 없으면 패턴 적용 전에 키 설계부터 해야 함
- I/O 오버헤드
    - 단순 INSERT가 아니라 기존 데이터와 매칭하는 과정이 있어서 성능이 더 무거움
    - 다만 현대 DW(BigQuery, Snowflake 등)는 메타데이터 통계를 활용해 최적화
- DELETE 미지원
    - 이 패턴은 incremental 데이터에서 INSERT/UPDATE만 처리
    - 소스에서 삭제가 발생하면 별도 soft delete 처리 필요 → Delete 표기만(컬럼에다가)
- Backfill 시 일관성 문제
    - MERGE는 현재 상태 기준으로 동작 → backfill 재실행 시 의도치 않은 결과 가능
    - 이걸 해결하려면 패턴 #19 Stateful Merger가 필요
### Backfill 시 일관성 문제 - 예시 
정상 실행 시나리오 (devices 테이블):
```
10-05 실행: INSERT laptop/1.0 (MacBook Pro 2024)
10-06 실행: UPDATE laptop/1.0 → MacBook Pro 2025
10-07 실행: UPDATE laptop/1.0 → MacBook Pro 2026

현재 테이블 상태:
laptop/1.0 | MacBook Pro 2026
```

여기서 10-06 데이터에 문제가 생겨서 backfill 재실행:
```
10-06 변경분: laptop/1.0 → MacBook Pro 2025

현재 테이블 상태 (10-07까지 실행된 상태):
laptop/1.0 | MacBook Pro 2026

10-06 변경분 MERGE 실행:
→ MATCHED → UPDATE → MacBook Pro 2025로 덮어씀

결과:
laptop/1.0 | MacBook Pro 2025  ← 10-07 변경분이 사라짐
```


문제가 뭐냐면:
```
원래 의도한 backfill 결과:
10-06 시점으로 되돌린 다음 → 10-06 데이터 수정 → 10-07 재실행

실제로 일어난 일:
10-07까지 반영된 현재 상태에 → 10-06 변경분을 그냥 덮어씀
→ 10-07 변경분이 유실됨
```

> MERGE는 "지금 테이블이 어떤 상태인지" 신경 안 써. 
> 그냥 키 매칭해서 덮어쓰는 거야. 과거 시점으로 되돌리는 기능이 없어.

> Q.scd 문제 맞지?
> A.맞아, 정확히 같은 구조야.
>    SCD(Slowly Changing Dimension) Type 1이 딱 이 문제야.
>    Stateful Merger가 해결하는 방식은 SCD Type 2랑 유사해.

```
SCD Type 2:
변경 시점마다 버전을 따로 저장
→ 10-06 버전, 10-07 버전이 별도로 존재
→ backfill 시 해당 시점 버전으로 되돌릴 수 있음
```




## (4)예시

**등장인물 정의**
- Airflow DAG: 매 시간 실행되는 배치 잡. 변경분 CSV를 읽어 PostgreSQL에 반영
- PostgreSQL: 디바이스 정보가 최종 적재되는 저장소
- changed_devices: 변경분을 담는 임시 테이블. 트랜잭션 종료 시 자동 삭제


**Step 1 — 변경분을 임시 테이블에 로드**
```
CREATE TEMPORARY TABLE changed_devices (LIKE dedp.devices);
-- LIKE dedp.devices: 타겟 테이블과 동일한 스키마로 생성 (스키마 불일치 방지)

COPY changed_devices
FROM '/data_to_load/dataset.csv'
CSV DELIMITER ';' HEADER;
-- 변경분 파일을 임시 테이블에 올림
```


**Step 2 — MERGE 연산 (핵심)**
```
MERGE INTO dedp.devices AS d
USING changed_devices AS c_d
  ON c_d.type = d.type AND c_d.version = d.version
  -- type + version 조합으로 기존 row와 매칭 (고유 키)

WHEN MATCHED THEN
  UPDATE SET full_name = c_d.full_name
  -- 키가 일치하면 → UPDATE

WHEN NOT MATCHED THEN
  INSERT (type, full_name, version)
  VALUES (c_d.type, c_d.full_name, c_d.version);
  -- 키가 없으면 → INSERT
```


**실행 결과 예시**
```
변경분 파일 내용:
type=laptop, version=1.0, full_name=MacBook Pro 2026  ← 기존 row (UPDATE 대상)
type=tablet, version=3.0, full_name=iPad Ultra        ← 신규 row (INSERT 대상)

MERGE 실행 후:
laptop/1.0 → full_name이 MacBook Pro 2026으로 수정됨
tablet/3.0 → 새 row로 추가됨
나머지 기존 데이터 → 건드리지 않음
```


**재실행(backfill) 시 동작**
```
같은 변경분 파일로 재실행:
laptop/1.0 → MATCHED → 동일한 값으로 UPDATE (결과 동일)
tablet/3.0 → MATCHED → 이미 존재하므로 UPDATE (중복 INSERT 없음)
→ idempotent 보장
```


##### 엔지니어 독백
>`LIKE dedp.devices` 이거 처음엔 왜 쓰는지 모를 수 있어. 그냥 컬럼 직접 써도 되거든. 
>근데 나중에 타겟 테이블에 컬럼 추가되면? 임시 테이블 생성 쿼리도 같이 수정해야 해. 
>두 군데를 관리하는 거야. 
>`LIKE`를 쓰면 타겟 테이블 스키마가 바뀌어도 임시 테이블이 자동으로 따라가. 
>이런 사소한 선택이 6개월 뒤 스키마 변경할 때 네 야근을 줄여줘.


<br><br><br>
<br>
<br>
<br>
<br><br><br>





























# 패턴#19 - Stateful Merger
### (1) 문제상황

Merger 패턴(#18)은 backfill 시 일관성을 보장하지 못해. 
MERGE는 "현재 테이블 상태" 기준으로 동작하기 때문이야.

**실제 상황:**

```
정상 실행:
10-05 실행 → laptop/1.0 INSERT (MacBook Pro 2024)  → 테이블 버전 1
10-06 실행 → laptop/1.0 UPDATE (MacBook Pro 2025)  → 테이블 버전 2
10-07 실행 → laptop/1.0 UPDATE (MacBook Pro 2026)  → 테이블 버전 3
```

10-06 데이터에 문제가 생겨서 backfill 재실행:

```
현재 테이블: MacBook Pro 2026 (버전 3 상태)
10-06 변경분 MERGE 실행 → MacBook Pro 2025로 덮어씀
→ 10-07 변경분(MacBook Pro 2026) 유실
```

MERGE는 과거 시점으로 되돌리는 기능이 없어. backfill 하려면 
먼저 테이블을 10-05 시점으로 복원한 다음에 10-06을 재실행해야 하는데, (원복 후 Merge)
Merger 패턴엔 그 복원 메커니즘이 없어.

> Q. 백필이라는게 원래 현재시점까지 적용하는거 아닌가? 원래 10-07까지 실행하는거니까 문제가 없는거 아닌지?
> A.맞아, 제대로 된 backfill이라면 10-06 재실행 후 10-07도 연쇄 재실행해.
> 근데 문제는 Merger(#18)에서 10-07 연쇄 재실행이 항상 보장되지 않는 케이스야.

```
케이스 1 — 10-07 실행 중 장애 발생:
10-06 재실행 완료 → 10-07 재실행 시작 → 중간에 클러스터 죽음
→ 테이블 상태: MacBook Pro 2025 (10-07 유실)

케이스 2 — 수동 backfill:
엔지니어가 실수로 10-06만 재실행하고 10-07은 안 함
→ 테이블 상태: MacBook Pro 2025 (10-07 유실)

케이스 3 — 10-06 변경분 자체가 잘못됨:
10-06 데이터 fix 후 재실행 → 10-07은 이미 올바른 데이터
→ 10-07까지 재실행하면 불필요한 재처리
→ 10-06만 재실행하고 싶은데 Merger는 그게 안전하지 않음
```

**Stateful Merger가 해결하는 핵심:**
- 연쇄 재실행이 실패하거나 부분 실행되더라도 테이블 일관성이 깨지지 않는 것
```
10-06만 재실행해도 안전한 이유:
→ 복원 단계에서 테이블을 10-05 버전으로 되돌림
→ 10-06 MERGE 실행
→ 10-07 재실행 시 버전 불일치 감지 → 자동으로 올바른 시점에서 이어받음
```


### (2) 솔루션

- 별도 state 테이블을 두고, 매 실행마다 "실행시간 → 테이블 버전" 을 기록해. 
- Backfill 감지 시 state 테이블을 보고 해당 시점 버전으로 먼저 복원한 다음 MERGE를 실행해.

**전체 흐름:**
```
[복원 단계] → [MERGE 단계] → [state 테이블 업데이트]
```

**state 테이블 구조:**
```
execution_time  | table_version
2024-10-05      | 1
2024-10-06      | 2
2024-10-07      | 3
```

**backfill 감지 로직:**
```
정상 실행 감지:
→ 이전 실행(10-07)이 만든 버전 == 현재 테이블 최신 버전
→ 복원 없이 바로 MERGE 진행

backfill 감지:
→ 이전 실행(10-06)이 만든 버전 != 현재 테이블 최신 버전
→ 10-05가 만든 버전으로 테이블 복원 후 MERGE 진행
```

Delta Lake의 Time Travel 기능으로 특정 버전으로 복원가능 :
```
# Delta Lake Time Travel로 버전 복원

# 핵심: RESTORE 명령어로 테이블을 특정 버전으로 되돌림
RESTORE TABLE dedp.devices TO VERSION AS OF 1;

#버전 번호는 state 테이블에서 조회한 값
```


### (3) 결과

트레이드오프 3가지:
1. 버전 관리 가능한 저장소 필수
    - Delta Lake, Apache Iceberg, Apache Hudi 같은 table file format이어야 함
    - 버전 관리가 안 되는 저장소라면 별도 raw 테이블에 실행시간 컬럼을 추가해서 우회 가능
2. state 테이블 관리 부담
    - state 테이블이 손상되거나 누락되면 backfill 감지 자체가 불가능
    - state 테이블도 별도 백업/모니터링 필요
    - state 테이블이 망가지면 Merger(#18)와 동일한 문제가 다시 터짐
3. Merger보다 구현 복잡도 높음
    - backfill 감지 로직 + 복원 로직 + state 업데이트 로직 세 가지를 모두 관리해야 함
    - backfill 요건이 없으면 Merger(#18)로 충분
4. Vacuum 필요
	- RESTORE로 되돌린 후 덮어쓴 파일들이 디스크에 그대로 남아있음
	- Delta Lake, Iceberg는 Time Travel을 위해 의도적으로 보관하는 구조
~~~
VACUUM delta.`s3://bucket/devices/` RETAIN 168 HOURS;
   -- 7일치 보관 후 오래된 파일 실제 삭제
   -- 안 하면 S3 비용 계속 누적
~~~
5. 메타데이터 오버헤드
	- RESTORE, DESCRIBE HISTORY 같은 메타데이터 작업이 매 실행마다 추가됨
	- 테이블 버전이 쌓일수록 DESCRIBE HISTORY 조회가 느려질 수 있음
	- 주기적인 테이블 최적화(OPTIMIZE) 병행 필요


### (4) 예시
#### 엔지니어 독백
>실무에서 Stateful Merger를 쓰게 되는 순간은 보통 이래. Merger(#18)로 잘 돌아가다가 어느 날 데이터 팀에서 슬랙이 와. "어제 디바이스 데이터 이상한데요, 10-06 데이터 잘못 들어온 것 같아요."
>
>Airflow에서 10-06 DAG 재실행 눌렀는데, 10-07까지 연쇄 재실행이 되어야 하는데 중간에 클러스터가 죽거나, 실수로 10-06만 재실행하면 그 순간 테이블이 꼬여. 이런 상황을 한 번 겪고 나면 "backfill 시 복원 메커니즘이 필요하다"는 걸 뼈저리게 느껴. 그때부터 Stateful Merger로 재설계하게 되는 거야.
>
>구조는 단순해. DAG 시작 전에 "지금 backfill이냐 정상 실행이냐"를 판단하고, backfill이면 먼저 테이블을 올바른 시점으로 되돌린 다음 MERGE를 실행해.
**등장인물 정의**

- Airflow DAG: 매일 실행되는 배치 잡. 디바이스 변경분을 Delta Lake 테이블에 반영
- Delta Lake 테이블(dedp.devices): 디바이스 정보 최종 적재 저장소. 버전 관리 지원
- state 테이블(dedp.devices_state): 실행시간과 테이블 버전을 매핑해서 저장하는 테이블



**Step 1 — backfill 감지 및 복원**

```
# Airflow PythonOperator에서 실행
# 이전 실행이 만든 버전과 현재 최신 버전을 비교해서 backfill 여부 판단


def restore_if_backfill(**context):
    ex_date = context['execution_date']    # Airflow 주입 변수. 현재 실행 날짜
    prev_ex_date = context['prev_execution_date']  # 이전 실행 날짜


    # state 테이블에서 이전 실행이 기록한 버전 조회
    prev_version = query("""
        SELECT table_version FROM dedp.devices_state
        WHERE execution_time = %(prev_date)s
    """, prev_date=prev_ex_date)


    # Delta Lake 현재 최신 버전 조회
    current_version = spark.sql(
        "DESCRIBE HISTORY dedp.devices LIMIT 1"
    ).first()['version']

    if prev_version is None:
        # 첫 실행 또는 첫 번째 backfill → 테이블 전체 초기화
        spark.sql("TRUNCATE TABLE dedp.devices")

    elif prev_version != current_version:
        # ★ 핵심: 버전 불일치 = backfill 감지 → 이전 실행 시점 버전으로 복원
        spark.sql(f"""
            RESTORE TABLE dedp.devices TO VERSION AS OF {prev_version}
        """)
    
    
    # 버전 일치 = 정상 실행 → 복원 없이 통과
```

- `RESTORE TABLE ... TO VERSION AS OF`: Delta Lake Time Travel 명령어. 테이블을 특정 버전으로 되돌림
- 이게 이 패턴의 핵심. Merger와의 유일한 차이점이야



**Step 2 — MERGE 실행 (Merger 패턴과 동일)**

```
# 변경분 파일을 임시 테이블에 로드 후 MERGE
# Merger 패턴(#18)과 동일한 로직

MERGE INTO dedp.devices AS d
USING changed_devices AS c_d
  ON c_d.type = d.type AND c_d.version = d.version
WHEN MATCHED THEN
  UPDATE SET full_name = c_d.full_name
WHEN NOT MATCHED THEN
  INSERT (type, full_name, version)
  VALUES (c_d.type, c_d.full_name, c_d.version)
```


**Step 3 — state 테이블 업데이트**
```
# MERGE 완료 후 새로 생성된 버전 번호를 state 테이블에 기록
# 다음 실행의 backfill 감지에 사용됨

def update_state(**context):
    ex_date = context['execution_date']

    # MERGE로 새로 생성된 최신 버전 조회
    new_version = spark.sql(
        "DESCRIBE HISTORY dedp.devices LIMIT 1"
    ).first()['version']

    # ★ 핵심: 실행시간 → 버전 매핑을 state 테이블에 저장
    query("""
        INSERT INTO dedp.devices_state (execution_time, table_version)
        VALUES (%(ex_date)s, %(version)s)
    """, ex_date=ex_date, version=new_version)
```


**backfill 시나리오 전체 흐름:**
```
state 테이블 현황:
10-05 → 버전 1
10-06 → 버전 2
10-07 → 버전 3 (현재 최신)

10-06 backfill 재실행:
→ 이전 실행(10-05)이 만든 버전: 1
→ 현재 최신 버전: 3
→ 버전 불일치 감지 → 버전 1로 RESTORE
→ 10-06 변경분 MERGE 실행 → 버전 4 생성
→ state 테이블 업데이트: 10-06 → 버전 4

이후 10-07 자동 backfill:
→ 이전 실행(10-06)이 만든 버전: 4
→ 현재 최신 버전: 4
→ 버전 일치 → 복원 없이 MERGE 실행 → 버전 5 생성
```

### (5) 최신트렌드
Delta Lake, Apache Iceberg, Apache Hudi 이 세 가지를 
**Table File Format** 이라고 부르는데, 공통적으로 이걸 해결하려고 나온 거야.

**이전 기술의 한계 — 순수 S3 + Parquet:**
```
S3에 Parquet로 저장하면:
- 버전 관리 없음 → RESTORE 불가능
- 트랜잭션 없음 → 쓰다가 실패하면 파일 일부만 존재
- MERGE 없음 → 직접 파일 읽고 처리 후 전체 재작성
```

**Table File Format이 추가한 것:**
```
Parquet 데이터 파일은 그대로 + 그 위에 메타데이터 레이어 추가
→ commit log (Delta Lake), metadata log (Iceberg)

이 로그 덕분에:
- 버전 관리 → Time Travel, RESTORE 가능
- 트랜잭션 → all-or-nothing 보장
- MERGE 최적화 → 변경된 파일만 재작성
```

- Stateful Merger에서 `RESTORE TABLE ... TO VERSION AS OF` 이게 가능한 이유가 바로 commit log가 있어서야.
- 순수 S3 + Parquet 였으면 Stateful Merger 자체를 구현할 수 없어. 버전이라는 개념 자체가 없으니까.

#### 엔지니어 독백
>Stateful Merger 처음 설계할 때 state 테이블을 가볍게 보는 경우가 많아. "그냥 버전 숫자 저장하는 테이블 아니야?" 하고.
>근데 이 state 테이블이 망가지면 backfill 감지가 아예 안 돼. 그러면 backfill 실행했을 때 복원 없이 MERGE가 그냥 실행되고, Merger 패턴(#18)이랑 똑같은 문제가 다시 터져.
>state 테이블은 반드시 별도 모니터링 대상에 넣어야 해. 그리고 Delta Lake 없이 이 패턴 쓰려면 raw 테이블에 execution_time 컬럼 추가하는 방식으로 우회해야 하는데, 그건 쿼리가 훨씬 복잡해져. 처음부터 Delta Lake나 Iceberg 위에서 설계하는 게 훨씬 편해.

<br>
<br>
<br>
<br><br><br>
<br><br><br>























# 4.3 데이터베이스(Database)

- 패턴 #20 키 기반 멱등성(Keyed Idempotency)
- 패턴 #21 트랜잭션 작성기(Transactional Writer)


<br>
<br><br><br>

# 패턴 #20 — Keyed Idempotency (키 기반 멱등성)

> 패턴#19 vs 패턴#20 차이가 뭐야 둘다 id로 중복방지 하는거 아닌가?

패턴 #19 Stateful Merger: 
- 배치 파이프라인 (Airflow DAG) 
- 오케스트레이터가 있어서 "실행 전 복원" 단계를 넣을 수 있음 
- 문제: backfill 시 테이블 상태가 꼬임 
- 해결: state 테이블로 버전 추적 → 실행 전 RESTORE 
패턴 #20 Keyed Idempotency: 
- 스트리밍 파이프라인 (Spark Structured Streaming) 
- 오케스트레이터가 없어서 "실행 전 뭔가를 한다"는 개념 자체가 없음 
- 문제: 태스크 재시도 시 키가 달라져서 중복 발생 
- 해결: 키 생성 자체를 불변값 기반으로 만들어서 재시도해도 항상 같은 키

### (1) 문제상황

스트리밍 파이프라인은 오케스트레이터가 없다. 태스크 실패 시 자동 재시도가 발생하고, 재시도 과정에서 중복 데이터가 적재된다.

---

**용어 정의**

- event_time: 이벤트가 실제로 발생한 시간
    - 클라이언트(앱, 브라우저)가 생성해서 payload에 담아 전송
    - 네트워크 지연으로 늦게 도착 가능 → 가변값
- append_time: Kafka 브로커가 메시지를 수신해서 로그에 기록한 시간
    - 브로커가 직접 생성
    - 한번 기록되면 변경 불가 → 불변값

```
user_id=1이 09:55에 검색했는데 네트워크 지연으로 10:05에 Kafka 도착:
event_time  = 09:55  ← 이벤트 발생 시간. 늦게 도착해도 이 값은 09:55
append_time = 10:05  ← Kafka 수신 시간. 절대 안 바뀜
```

---

**등장인물 정의**

- Spark Structured Streaming 잡: Kafka에서 방문 이벤트를 읽어 세션을 생성하고 ScyllaDB에 저장
- ScyllaDB: 생성된 세션이 적재되는 key-value store. PRIMARY KEY로 row를 식별. 같은 키면 덮어씀
- Kafka topic: 방문 이벤트가 흘러들어오는 스트리밍 브로커

---

**ScyllaDB 적재 구조**

ScyllaDB는 PRIMARY KEY가 같으면 에러 없이 덮어쓴다. session_id가 항상 동일하게 생성되는 한 재시도해도 같은 row를 덮어쓰기 때문에 중복이 발생하지 않는다.

```
PRIMARY KEY = (session_id, user_id)

session_id | user_id | pages            | ingestion_time
-----------|---------|------------------|---------------
1001       | 1       | [home, product]  | 10:30
1002       | 2       | [home, cart]     | 11:00
```

문제는 session_id가 재시도할 때마다 다르게 생성된다는 것이다. Spark 잡이 세션 키 생성에 event_time(가변값)을 사용하기 때문이다.

```
session_id 생성 로직:
session_id = hash(user_id, min(event_time))  ← 가변값 사용
```

1차 실행:

```
Kafka에서 읽은 이벤트:
user_id=1, event_time=10:00, page=home
user_id=1, event_time=10:15, page=product
user_id=1, event_time=10:30, page=cart

min(event_time) = 10:00
session_id = hash(1, 10:00) = 1001

ScyllaDB 적재:
session_id=1001 | user_id=1 | pages=[home, product, cart]
```

태스크 실패 후 재시도. 재시도 타이밍에 09:55 late data가 Kafka에 추가됨:

```
Kafka에서 읽은 이벤트 (09:55 late data 포함):
user_id=1, event_time=09:55, page=search  ← late data
user_id=1, event_time=10:00, page=home
user_id=1, event_time=10:15, page=product
user_id=1, event_time=10:30, page=cart

min(event_time) = 09:55  ← 바뀜
session_id = hash(1, 09:55) = 9999  ← 다른 키 생성

ScyllaDB 적재 결과:
session_id=1001 | user_id=1 | pages=[home, product, cart]       ← 1차 실행
session_id=9999 | user_id=1 | pages=[search, home, product, cart] ← 재시도
→ user_id=1의 세션 2개 존재. 중복 발생
```

ScyllaDB는 1001과 9999를 완전히 다른 키로 인식한다. 덮어쓸 대상을 찾지 못해 새 row로 INSERT한다.

비즈니스 임팩트:

```
분석팀 쿼리:
SELECT COUNT(DISTINCT session_id) FROM sessions WHERE user_id = 1
→ 결과: 2  (실제로는 1번의 방문)

구매 전환율 계산:
실제 방문 세션: 1개 / 집계된 세션: 2개
→ 전환율이 절반으로 줄어든 것처럼 보임
```

---

### (2) 솔루션

세션 키 생성에 append_time(불변값)을 사용한다. late data가 들어와도 append_time은 Kafka 브로커가 기록한 값이기 때문에 변경되지 않는다.

```
키 생성 전략 변경:
Before: session_id = hash(user_id, min(event_time))   ← 가변값
After:  session_id = hash(user_id, min(append_time))  ← 불변값

재시도 시:
1차 실행: session_id = hash(1, append_time=10:01) = 1001
재시도:   session_id = hash(1, append_time=10:01) = 1001  ← 동일
→ ScyllaDB에 같은 키로 덮어씀 → 중복 없음
```

---

**단, append_time도 안전하지 않은 케이스가 있다 — Kafka compaction**

- 잡이 세션 키를 만들 때 그룹 내 min(append_time)을 사용
- compaction으로 가장 오래된 메시지가 삭제되면 min(append_time) 기준이 바뀜 → 키 생성 기준 붕괴

Kafka compaction 정의:

- 같은 키의 메시지 중 최신 것만 남기고 오래된 것을 삭제하는 정리 작업
- 목적: 디스크 절약 + 소비자 초기 읽기 속도 향상

```
compaction 전:
offset 1: user_id=1, append_time=10:00  ← 삭제 대상
offset 2: user_id=1, append_time=10:15  ← 삭제 대상
offset 3: user_id=1, append_time=10:30  ← 최신. 유지

compaction 후:
offset 3: user_id=1, append_time=10:30  만 남음
```

잡이 장애로 오래 멈춰있는 사이 compaction 실행 시:

```
잡 재시작 후 재처리:
min(append_time) = 10:30  ← 10:00이 삭제됐으니 기준 변경
session_id = hash(1, 10:30) = 9999  ← 원래 1001과 다른 키

ScyllaDB 결과:
session_id=1001 | user_id=1  ← 1차 실행 결과
session_id=9999 | user_id=1  ← 재시작 후 추가
→ 중복 발생
```

---

### (3) 결과

1. key-value store 의존
    - NoSQL(ScyllaDB, Cassandra): 같은 키 → 자동 덮어씀 → 이 패턴이 가장 자연스럽게 동작
    - PostgreSQL: 같은 PK INSERT 시 에러 발생 → INSERT 대신 MERGE(UPSERT) 사용 필요

```
     재시도:
     INSERT INTO sessions VALUES (1001, 1, ...)  → ERROR: duplicate key
```

- Kafka: append-only log라서 compaction 전까지 소비자가 중복 데이터를 볼 수 있음

```
     재시도:
     session_id=1001 produce → 로그에 또 추가
     → compaction 전까지 중복 2개 존재
```

2. compaction 주의
    - 잡이 오래 멈춰있으면 compaction으로 min(append_time) 기준이 바뀔 수 있음
    - compaction 주기와 잡 재시작 시나리오를 함께 고려해야 함


### (4) 예시

##### 엔지니어 독백

> 이 패턴을 도입하게 되는 계기는 대부분 장애 이후다. 처음엔 event_time으로 키를 만들어도 잘 돌아간다. 운영하다 보면 어느 날 ScyllaDB에 같은 유저의 세션이 두 개씩 생겨있는 걸 발견하게 된다. 로그를 뒤져보면 태스크 재시도 타이밍에 late data가 들어온 거다.

> 키 생성에 무엇을 쓰냐는 코드 한 줄 수정이지만, 그 판단을 처음 설계할 때 하느냐 장애 나고 하느냐는 완전히 다른 얘기다. 설계 단계에서 반드시 확인해야 할 것은 딱 하나다. "이 값이 재시도 시에도 동일하게 나오는가."



**ScyllaDB 테이블 정의**

PRIMARY KEY 설계가 idempotency를 결정한다. 같은 PRIMARY KEY로 INSERT 시 ScyllaDB가 자동으로 덮어쓰기 때문이다.

```
-- ★ 핵심: PRIMARY KEY = (session_id, user_id)
-- 같은 키로 INSERT 시 자동 덮어씀

CREATE TABLE sessions (
    session_id     BIGINT,
    user_id        BIGINT,
    pages          LIST<TEXT>,
    ingestion_time TIMESTAMP,
    PRIMARY KEY(session_id, user_id)
);
```



**Spark Structured Streaming 잡 — append_time 기반 키 생성**

Kafka에서 방문 이벤트를 읽을 때 append_time을 함께 추출한다.

```
# ★ 핵심: event_time이 아닌 timestamp(append_time)를 키 생성에 사용
# timestamp = Kafka 브로커가 메시지 수신한 시간 (불변값)

input_df = (
    spark.readStream
    .format("kafka")
    .option("kafka.bootstrap.servers", "localhost:9094")
    .option("subscribe", "visits")
    .load()
    .selectExpr(
        "CAST(value AS STRING)",
        "timestamp AS append_time"
    )
)
```

append_time 기준으로 세션 키를 생성한다.

```
def generate_session(key, input_rows, current_state):
    user_id = key[0]

    # ★ 핵심: min(append_time)으로 session_id 생성
    # late data가 들어와도 append_time은 변경되지 않음
    min_append_time = min(
        row['append_time'] for row in input_rows
    )
    session_id = hash((user_id, min_append_time))

    pages = [row['page'] for row in input_rows]

    if current_state.exists:
        existing_session_id, existing_pages = current_state.get
        current_state.update((existing_session_id, existing_pages + pages))
    else:
        current_state.update((session_id, pages))

    yield current_state.get
```



### (5) 최신트렌드

- Apache Flink + Kafka 조합에서 Keyed Idempotency 대신 패턴 #21 Transactional Writer로 대체하는 케이스 증가
    - Flink는 Kafka transactional producer를 지원 → 트랜잭션이 키 기반 중복 제거보다 강한 보장을 제공
    - Spark + Kafka 조합에서는 트랜잭션 미지원 → Keyed Idempotency가 여전히 주력
- ScyllaDB 대신 Apache Cassandra 5.0 도입 증가
    - LWT(Lightweight Transaction) 개선으로 키 기반 upsert 성능 향상
- DynamoDB, Bigtable 같은 managed key-value store 활용 증가
    - 인프라 관리 부담 없이 같은 패턴 적용 가능
    - 특히 AWS EMR + DynamoDB 조합이 실무에서 자주 쓰임






<br><br><br>
<br><br><br>
<br>
<br>
<br>













---

# 패턴 #21 트랜잭션 작성기(Transactional Writer)

### (1) 문제상황

클라우드 환경에서 비용 절감을 위해 스팟 인스턴스(Spot Instance)를 사용하는 배치 잡은 노드가 언제든 회수될 수 있다. 노드 회수 시 실행 중인 태스크가 실패하고 재시도가 발생한다. 이때 이미 성공한 태스크가 데이터를 다시 쓰면서 소비자가 중복 또는 불완전한 데이터를 보게 된다.

**용어 정의**
- 스팟 인스턴스(Spot Instance): 클라우드 공급자가 유휴 컴퓨팅 자원을 저렴하게 제공하는 인스턴스. 공급자가 필요하면 언제든 회수 가능
- 트랜잭션(Transaction): 여러 쓰기 작업을 하나의 단위로 묶는 메커니즘. commit 전까지 소비자에게 변경사항이 노출되지 않음
- commit: 트랜잭션 내 모든 쓰기 작업을 소비자에게 공개하는 확정 단계
- rollback: 트랜잭션 내 모든 쓰기 작업을 취소하는 단계
- 소비자(Consumer): 
	- 적재 완료된 데이터를 읽어가는 주체 - SQL 쿼리, BI 툴, 다른 배치 잡 등 
	- COMMIT 이전 데이터는 읽을 수 없음 (트랜잭션 격리)

**등장인물 정의**
- Airflow DAG: 스팟 인스턴스 클러스터에서 실행되는 배치 잡. 디바이스 데이터 2개 파일을 읽어 PostgreSQL에 적재
- PostgreSQL: 디바이스 데이터가 최종 적재되는 저장소
- 스팟 인스턴스 클러스터: AWS EMR 스팟 인스턴스로 구성된 클러스터. 노드 회수 시 실행 중 태스크 실패

**문제 발생 시나리오:**
파이프라인이 디바이스 데이터 파일 2개를 순차적으로 처리해서 PostgreSQL에 적재한다.
```
처리 순서:
1. dataset_1.csv 읽기 → MERGE INTO devices
2. dataset_2.csv 읽기 → MERGE INTO devices
```

1번 파일 처리 완료 후 2번 파일 처리 중 스팟 인스턴스 회수 발생:
```
1번 파일 MERGE 완료 → devices 테이블에 즉시 반영 (소비자에게 노출)
2번 파일 MERGE 중 → 노드 회수로 태스크 실패

소비자가 보는 devices 테이블:
1번 파일 데이터만 반영된 불완전한 상태
→ 데이터 분석팀이 불완전한 데이터로 리포트 생성
```

태스크 재시도 시:
```
1번 파일 MERGE 재실행 → 이미 적재된 데이터에 중복 발생 가능
2번 파일 MERGE 재실행 → 정상 완료

소비자가 보는 devices 테이블:
1번 파일 데이터 중복 + 2번 파일 데이터 혼재
```

핵심 문제: 트랜잭션 없이 각 MERGE가 즉시 commit되기 때문에 파이프라인 중간 실패 시 소비자가 불완전한 데이터를 보게 된다.

> transaction이 필요한 작업의 예시로는
> 기간단위의 매출을 위한 작업이면 이해가 쉬울듯!










### (2) 솔루션
>트랜잭션으로 작업을 묶어서 처리한다

2개의 MERGE 작업을 하나의 트랜잭션으로 묶는다. 두 작업이 모두 성공했을 때만 commit해서 소비자에게 공개한다. 중간에 실패하면 rollback해서 소비자가 불완전한 데이터를 보지 못하게 한다.
```
트랜잭션 적용 전:
MERGE (1번 파일) → 즉시 소비자에게 노출
MERGE (2번 파일) → 즉시 소비자에게 노출

트랜잭션 적용 후:
BEGIN TRANSACTION
  MERGE (1번 파일) → 트랜잭션 내부에만 존재. 소비자에게 미노출
  MERGE (2번 파일) → 트랜잭션 내부에만 존재. 소비자에게 미노출
COMMIT               → 두 작업 동시에 소비자에게 공개
```

2번 파일 처리 중 실패 시:
```
BEGIN TRANSACTION
  MERGE (1번 파일) → 트랜잭션 내부에만 존재
  MERGE (2번 파일) → 실패
ROLLBACK             → 1번 파일 작업도 취소
→ 소비자는 변경사항을 전혀 볼 수 없음
```

**분산 처리 환경에서의 트랜잭션 구현 방식 2가지:**
Spark 실행 계층 구조:
```
Application (스파크 앱 전체)
└── Job (Action 하나당 1개 생성)
    └── Stage (셔플 경계로 나뉨)
        └── Task (파티션 하나당 1개 생성)
```

**1. 태스크 단위 로컬 트랜잭션**
Spark의 각 태스크가 독립적으로 트랜잭션을 열고 commit한다.
```
Spark 잡 실행 구조:
Driver
├── Task 1 → partition 1 처리 → BEGIN → MERGE → COMMIT
├── Task 2 → partition 2 처리 → BEGIN → MERGE → COMMIT
└── Task 3 → partition 3 처리 → BEGIN → MERGE → COMMIT
```

문제: 잡 재시도 시 이미 commit된 태스크가 데이터를 다시 씀
```
1차 실행:
Task 1 COMMIT 완료 → devices에 반영됨
Task 2 COMMIT 완료 → devices에 반영됨
Task 3 실패 → 잡 전체 재시도

재시도:
Task 1 재실행 → 이미 COMMIT된 데이터에 중복 INSERT
Task 2 재실행 → 이미 COMMIT된 데이터에 중복 INSERT
Task 3 재실행 → 정상 완료
→ Task 1, 2 데이터 중복 발생
```


**2. 잡 단위 글로벌 트랜잭션**
잡 시작 시 트랜잭션을 열고 모든 태스크가 완료된 후 Driver가 한 번에 commit한다.
```
Spark + Delta Lake 실행 구조:
Driver → 트랜잭션 시작
├── Task 1 → partition 1 처리 → 데이터 파일 S3에 씀 (미노출)
├── Task 2 → partition 2 처리 → 데이터 파일 S3에 씀 (미노출)
└── Task 3 → partition 3 처리 → 데이터 파일 S3에 씀 (미노출)
Driver → 모든 태스크 완료 → commit log 파일 생성 → 소비자에게 노출

→ COMMIT 미도달 
→ 전체 rollback 
→ Task 1, 2가 쓴 파일도 무효화 
→ 재시도 시 처음부터 깨끗하게 재실행 → 중복 없음
```

Delta Lake의 commit 방식:
```
S3 구조:
s3://bucket/devices/
├── _delta_log/
│   ├── 000001.json  ← commit log. 이 파일이 생성되는 순간이 COMMIT
│   └── 000002.json
├── part-001.parquet  ← 실제 데이터 파일 (commit log 없으면 소비자가 못 읽음)
└── part-002.parquet

Task 3 실패 시:
→ commit log 파일 미생성
→ part-001, part-002는 S3에 존재하지만 소비자가 읽을 수 없음
→ 잡 재시도 시 새 데이터 파일로 덮어씀
→ 중복 없음
```

**실무 결론:**
- 스팟 인스턴스 환경이면 무조건 잡 단위 글로벌 트랜잭션을 써야 한다
    - 태스크 단위 트랜잭션은 "각 태스크 내부의 원자성"만 보장, "잡 전체의 원자성"은 보장 못해.
    - 노드 회수가 언제 발생할지 모름 → 태스크 단위로는 중복을 막을 수 없음
    - Delta Lake를 쓰면 commit log 방식으로 글로벌 트랜잭션이 자동 구현됨 → 별도 구현 불필요
- RDB(PostgreSQL)를 싱크로 쓰는 경우 글로벌 트랜잭션 구현이 까다로움
    - Spark Driver에서 DB 커넥션을 열고 유지하면서 모든 Task가 같은 트랜잭션을 공유해야 함
    - 커넥션 수 제한, 트랜잭션 타임아웃 문제 발생 가능
    - 이 경우 태스크 단위 로컬 트랜잭션 + Keyed Idempotency(패턴 #20) 조합이 현실적인 선택








### (3) 결과

1. commit 단계 오버헤드 — 트랜잭션을 쓰면 소비자가 데이터를 늦게 받는다
    - 트랜잭션은 commit 전까지 데이터를 소비자에게 노출하지 않음
    - 가장 느린 태스크가 완료될 때까지 소비자가 데이터를 기다려야 함
    - JSON, CSV처럼 트랜잭션 없는 포맷은 파일 생성 즉시 소비자가 읽을 수 있음 → 빠름
    - Delta Lake처럼 트랜잭션 있는 포맷은 모든 태스크 완료 후 commit 시점에만 읽을 수 있음 → 느림
```
Task 1~2는 5분, Task 3이 20분 걸리는 잡:
JSON/CSV  → 소비자가 5분 후부터 부분 데이터를 읽을 수 있음
Delta Lake → 소비자가 20분을 전부 기다려야 완전한 데이터를 읽을 수 있음
```

2. 분산 처리 프레임워크 지원 제한 — 저장소가 트랜잭션을 지원해도 처리 프레임워크가 연동을 지원하지 않으면 사용할 수 없다
    - 설계 단계에서 저장소의 트랜잭션 지원 여부와 프레임워크의 연동 지원 여부를 반드시 함께 확인해야 함

```
Spark + Kafka    → Kafka 트랜잭션 지원 O, Spark 연동 X → 사용 불가
Flink + Kafka    → Kafka 트랜잭션 지원 O, Flink 연동 O → 사용 가능
Spark + Delta Lake → Delta Lake 트랜잭션 지원 O, Spark 연동 O → 사용 가능
```

3. idempotency 범위 제한 — 트랜잭션은 현재 실행 단위 내에서만 idempotency를 보장한다
    - backfill 재실행 시 새 트랜잭션이 열리면서 동일한 데이터가 다시 적재됨 → 중복 발생
    - 트랜잭션만으로는 backfill idempotency를 보장할 수 없음
    - backfill이 필요한 파이프라인은 트랜잭션 앞단에 실행 전 해당 파티션 TRUNCATE 또는 replaceWhere 덮어쓰기(패턴#16, #17) 를 함께 설계해야 함

```
backfill 시나리오:
1차 실행: 트랜잭션 열림 → 데이터 적재 → COMMIT → devices에 반영
backfill 재실행: 새 트랜잭션 열림 → 동일한 데이터 적재 → COMMIT → 중복 발생
→ 트랜잭션은 "이미 적재된 데이터가 있는지"를 확인하지 않음

올바른 설계:
backfill 재실행 전 → 해당 날짜 파티션 TRUNCATE 또는 replaceWhere로 기존 데이터 제거
(원복)
→ 트랜잭션으로 새 데이터 적재
→ 중복 없음
```
---








### (4) 예시

#### 엔지니어 독백
> 이 패턴을 도입하게 되는 계기는 대부분 스팟 인스턴스 장애 이후다. 비용 절감을 위해 스팟 인스턴스를 쓰다 보면 어느 날 새벽에 노드가 회수되면서 파이프라인이 중간에 죽는다. 아침에 출근한 분석팀이 "디바이스 데이터가 절반밖에 없어요" 슬랙을 보내는 거다.

> 트랜잭션 구현 자체는 단순하다. SQL에 BEGIN, COMMIT 감싸면 끝이다. 근데 설계 전에 반드시 확인해야 할 것이 있다. 저장소가 트랜잭션을 지원하는지, 그리고 사용하는 프레임워크가 그 저장소의 트랜잭션 연동을 지원하는지. Spark + Kafka 조합에서는 Kafka 트랜잭션 연동이 안 된다. 이 제약을 모르고 설계하면 나중에 전체를 다시 짜야 한다.


**배치 파이프라인 — PostgreSQL 트랜잭션**

디바이스 데이터 파일 2개를 하나의 트랜잭션으로 묶어서 PostgreSQL에 적재하는 코드다. 2개의 MERGE 작업이 모두 성공했을 때만 COMMIT해서 소비자에게 공개한다.

```
-- 1번 파일을 임시 테이블에 로드 후 devices 테이블에 MERGE
CREATE TEMPORARY TABLE changed_devices_file1 (LIKE dedp.devices);
COPY changed_devices_file1
FROM '/data_to_load/dataset_1.csv' CSV DELIMITER ';' HEADER;

MERGE INTO dedp.devices AS d
USING changed_devices_file1 AS c_d
  ON c_d.type = d.type AND c_d.version = d.version
WHEN MATCHED THEN
  UPDATE SET full_name = c_d.full_name
WHEN NOT MATCHED THEN
  INSERT (type, full_name, version)
  VALUES (c_d.type, c_d.full_name, c_d.version);

-- 2번 파일을 임시 테이블에 로드 후 devices 테이블에 MERGE
CREATE TEMPORARY TABLE changed_devices_file2 (LIKE dedp.devices);
COPY changed_devices_file2
FROM '/data_to_load/dataset_2.csv' CSV DELIMITER ';' HEADER;

MERGE INTO dedp.devices AS d
USING changed_devices_file2 AS c_d
  ON c_d.type = d.type AND c_d.version = d.version
WHEN MATCHED THEN
  UPDATE SET full_name = c_d.full_name
WHEN NOT MATCHED THEN
  INSERT (type, full_name, version)
  VALUES (c_d.type, c_d.full_name, c_d.version);

-- ★ 핵심: 두 MERGE가 모두 성공했을 때만 COMMIT
-- COMMIT 전까지 소비자는 변경사항을 볼 수 없음
COMMIT;
```

2번 파일에 컬럼 길이 초과 데이터가 있을 경우:
```
1번 파일 MERGE 완료
2번 파일 MERGE 실패 → COMMIT 미도달 → 자동 ROLLBACK
→ 1번 파일 작업도 취소
→ 소비자는 변경사항을 전혀 볼 수 없음
→ 재시도 시 처음부터 깨끗하게 재실행
```

---

**스트리밍 파이프라인 — Apache Flink + Kafka 트랜잭션**

Flink가 처리한 결과를 Kafka topic에 전송할 때 exactly-once를 보장하는 설정 코드다.

exactly-once 동작 흐름:
```
1. Flink checkpoint 시작
   → Flink가 현재 처리 상태를 S3/HDFS에 저장 시작

2. Kafka 트랜잭션 열림
   → Flink가 reduced_visits topic에 데이터 전송 시작
   → 소비자는 아직 이 데이터를 볼 수 없음

3. Flink checkpoint 완료
   → 상태 저장 완료 확인
   → Kafka에 COMMIT 신호 전송
   → 소비자가 데이터를 읽을 수 있게 됨

장애 발생 시:
   → checkpoint 미완료 → COMMIT 미도달
   → Flink 재시작 → 마지막 checkpoint 시점으로 복구
   → 해당 구간 데이터 재처리 → COMMIT
   → 소비자는 완전한 데이터만 읽음
```

```
# ★ 핵심 1: DeliveryGuarantee.EXACTLY_ONCE → Kafka 트랜잭션 활성화
# ★ 핵심 2: transaction.timeout.ms → checkpoint 완료 전 트랜잭션 만료 방지
# transaction.timeout.ms는 반드시 checkpoint 완료 예상 시간보다 크게 설정해야 함

kafka_sink = (
    KafkaSink.builder()
    .set_bootstrap_servers("localhost:9094")
    .set_record_serializer(
        KafkaRecordSerializationSchema.builder()
        .set_topic('reduced_visits')
        .set_value_serialization_schema(SimpleStringSchema())
        .build()
    )
    .set_delivery_guarantee(DeliveryGuarantee.EXACTLY_ONCE)
    .set_property(
        'transaction.timeout.ms',
        str(10 * 60 * 1000)  # checkpoint 최대 완료 시간이 5분이면 10분으로 설정
    )
    .build()
)
```

transaction.timeout.ms를 잘못 설정한 경우:
```
checkpoint 완료 시간: 2분
transaction.timeout.ms: 1분 (기본값)

0분: Kafka 트랜잭션 열림, 데이터 전송 시작
1분: transaction.timeout.ms 만료
     → Kafka가 트랜잭션 강제 취소(abort)
     → 트랜잭션 내 데이터 전부 취소

2분: Flink checkpoint 완료
     → Kafka에 COMMIT 신호 보내려 했으나 이미 abort
     → 데이터 유실
```

transaction.timeout.ms 설정 기준:
```
1. Flink UI → Jobs → Checkpoints 탭에서 최대 checkpoint 완료 시간 확인
   예: 평균 3분, 최대 5분

2. transaction.timeout.ms = 최대 checkpoint 시간 × 2 로 설정
   → 10분

3. Kafka broker의 transaction.max.timeout.ms 확인
   → transaction.timeout.ms가 이 값을 초과하면 Kafka가 에러 반환
   → 기본값 15분이므로 10분 설정은 안전
```


#### 엔지니어 독백 — 운영 시 주의사항
- transaction.timeout.ms 기본값(1분)으로 운영하면 checkpoint가 오래 걸리는 순간 데이터가 조용히 유실된다. 에러 로그도 안 남는 경우가 있어서 한참 뒤에 발견하게 된다. 반드시 Flink UI에서 checkpoint 완료 시간을 확인하고 그 2배로 설정해야 한다
- Spark + Kafka 조합에서 exactly-once가 필요하면 Flink로 전환하거나 Keyed Idempotency(불변 키 기반 중복 제거, #20)를 사용해야 한다
- backfill 시 트랜잭션만으로는 중복을 막을 수 없다. 트랜잭션 앞단에 해당 파티션 TRUNCATE 또는 replaceWhere 덮어쓰기(#16, #17)를 함께 설계해야 한다



### (5) 최신트렌드

- Delta Lake + Spark 조합이 배치 트랜잭션의 사실상 표준으로 자리잡음
    - commit log 기반 트랜잭션으로 분산 환경에서 잡 단위 글로벌 트랜잭션 구현 가능
    - AWS EMR, Databricks 모두 기본 지원
- Apache Flink + Kafka 조합이 스트리밍 exactly-once의 표준으로 자리잡음
    - Spark + Kafka는 트랜잭션 미지원 → 스트리밍 exactly-once가 필요하면 Flink로 전환하는 케이스 증가
- Apache Iceberg의 트랜잭션 지원 강화
    - Spark, Flink 모두 Iceberg 트랜잭션 지원
    - Delta Lake 대비 멀티 엔진 지원이 강점 → 벤더 종속성을 줄이고 싶은 팀에서 선호
 
<br><br><br> 
