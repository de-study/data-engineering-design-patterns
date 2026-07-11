## 6.3 팬아웃(Fan-Out)

- 패턴#37 병렬 분할(Parallel Split)
- 직전 서브챕터(팬인)는 여러 브랜치를 하나로 합쳤다.
- 팬아웃은 반대로, 한 태스크 결과를 여러 브랜치로 **가른다**.

---

## 패턴#36 병렬 분할(Parallel Split)

### (1)문제상황

- 결론: legacy 파이프라인을 new 파이프라인으로 바꾸는 중, 같은 결과를 old/new 두 저장소에 동시에 써야 한다.
- 엔지니어 독백:
    
    > 신입한테. 이거 migration 실무에서 진짜 자주 만난다. 나는 예전에 C#으로 짠 구 파이프라인을 Python으로 갈아엎는 일을 맡았는데, 원 코드 짠 사람은 이미 퇴사했고 문서도 없었다. reverse engineering으로 로직 파악해서 다시 짜는데, 이게 100% 똑같다는 확신이 안 선다. 그래서 new를 바로 켜고 old를 끄면? 사고 나면 되돌릴 게 없다. 그래서 한동안 old랑 new를 같이 돌리면서 결과를 대조한다. 이때 필요한 게 "한 번 처리한 결과를 두 곳에 뿌리는" 구조다.
    
- 등장 컴포넌트:
    - 레거시 프로그램 코드 : C#으로 작성된 구 파이프라인. 담당자 퇴사, 문서 없음.
    - 새로 도입하는 파이프라인 : Python으로 재작성 중인 new 파이프라인.
- 문제 흐름:
    - 기존 방식 : C# 로직을 reverse engineering으로 파악해 Python 재작성.
    - 실패 장면 : 재작성 정확성 보장 없음. new 결과가 old와 미세하게 어긋날 수 있음.
    - 개선 욕구 : consumer가 new로 완전히 넘어갈 때까지 old도 병행해 두 결과를 대조·검증.
    - 새 문제(패턴이 푸는 지점) : 같은 처리 결과를 old store, new store 두 곳에 동시 write. parent 하나(공통 처리)에서 child 둘(old write, new write)로 갈라짐. 이 "하나 → 여럿" 갈래가 Parallel Split.

### (2)솔루션

- 주요컨셉: parent가 공통 연산을 1회 수행해 중간 데이터셋으로 저장하고, 여러 child가 그것을 각자 읽어 나눠 쓴다.
- 왜 중간 데이터셋이 필요한가:
    - 이전 방식 : child마다 원천을 각자 읽어 처리.
    - 한계 : 공통 연산이 child 수만큼 중복 실행 → 자원·비용 낭비.
    - 해결 : 공통 결과를 한 번만 저장해 child가 그것만 읽게 함.
- 예시로 본 문제 (중간 데이터셋 없을 때):
    - 원천 : S3의 `raw_events.parquet` (방문 로그 1억 건)
    - 공통 처리 : 정제(clean). 무거움.
    - child①(old write) : `raw_events.parquet` 읽음 → 정제 → old store write
    - child②(new write) : `raw_events.parquet` 또 읽음 → 정제 또 함 → new store write
    - 결과 : 무거운 정제가 2번 실행. 1억 건 처리 시간·비용 2배.
- 해결 후 (중간 데이터셋 사용):
    - parent : `raw_events.parquet` 읽음 → 정제 → 중간 데이터셋 `cleaned`로 저장 (1회)
    - child① : `cleaned` 읽음 → old store write
    - child② : `cleaned` 읽음 → new store write
    - 결과 : 정제 1번. child는 정제된 결과만 소비.
- 처리 계층 3대 주의점:
    - 공통 연산 중복 방지 : 중간 데이터셋 저장으로 1회만 계산.
    - branch 간섭 방지 : 공유·전역 변수는 read-only거나 모든 write에 호환되게.
    - 자원 분리 : child별 요구 자원이 다르면 전용 compute 할당 또는 autoscale.
- 중간 데이터셋 저장 방식 (선택지):
    - SQL temp table : 임시 테이블로 저장. SQL 파이프라인용.
    - Spark `.persist()` : 중간 dataframe을 메모리·디스크에 유지. 한 Spark 잡 내부용.
    - 파일 write (parquet) : 스토리지에 기록. child가 별도 잡일 때용.
- 실무 선호 (상황별):
    - 한 Spark 잡 내부 분기 → `.persist()`. 잡 경계 안이라 메모리 재사용이 가장 빠름.
    - child가 별도 잡·별도 스케줄 → parquet write. 잡 간 공유는 파일이 안전.
    - 판단 기준 : 한 잡 내부면 `.persist()`, 잡 경계 넘으면 파일.


### (3)결과

- 결론: 공통 연산 1회 실행을 얻는 대가로, 단점 2가지를 감수한다.
- 단점 1 — 느린 child가 전체를 막음 (blocked execution):
    - 용어 : blocked execution = 한 갈래가 막혀 뒤 단계 전체가 멈추는 상태.
    - 배경 : Parallel Split은 child 여럿이 전부 끝나야 다음 단계로 넘어간다.
    - 문제 : child 하나가 느리거나 실패하면 나머지가 멀쩡해도 다음 단계 시작 불가. 가장 느린 child가 전체 속도를 결정.
    - 예시 : old write(1분) + new write(외부 API 지연 30분) → 파이프라인은 30분 뒤에야 다음 단계.
    - 대응 : 느린 child를 별도 파이프라인으로 분리, data-based 의존(#34)으로 느슨하게 연결.
- 단점 2 — child별 필요 장비가 다름 (hardware mismatch):
    - 용어 : hardware mismatch = child들의 자원 요구가 서로 안 맞는 상태.
    - 배경 : parent가 만든 중간 데이터셋을 child 여럿이 나눠 쓴다.
    - 문제 : child A는 CPU 중심, child B는 memory 중심이면 한 종류 장비로 둘 다 만족 불가.
    - 대응 : 중간 데이터셋 생성과 child 잡을 분리, child별 전용 장비에서 실행.
- 엔지니어 독백:
    
    > 신입한테. 제일 자주 데는 게 blocked execution이다. child 3개 묶었는데 하나가 느린 외부 API에 붙어 있으면, 나머지 2개 멀쩡해도 그날 파이프라인이 그거 기다리다 다 밀린다. child들 SLA가 제각각이면 "한 트리거에 묶는 게 맞나"부터 다시 본다. 묶으면 편하지만 느린 놈이 전체 발목 잡는다.



### (4)예시

- 결론: 오케스트레이션은 parent → child 리스트로 fan-out, 처리 계층은 `.persist()`로 중간 데이터셋 1회 계산이 핵심.
- 도메인: 방문 로그 migration. S3 `raw_events.parquet`를 정제해 old/new 두 store에 병행 write.

---

- 예시 1) 오케스트레이션 fan-out (Airflow)
- 한 줄 요약: parent 하나가 child 두 개로 갈라진다.

```
generate_dataset >> [write_to_legacy_store, write_to_new_store]
```

- 실행 순서:
    - ① `generate_dataset` : 공통 정제 → 중간 데이터셋 생성.
    - ② 완료 후 `write_to_legacy_store`, `write_to_new_store` 병렬 실행.
- 핵심 라인: `[write_to_legacy_store, write_to_new_store]` 리스트 표기. Airflow에서 parent 뒤 리스트 = fan-out 분기.

---

- 예시 2) 처리 계층 중간 데이터셋 재사용 (Spark)
- 한 줄 요약: 정제 결과를 `.persist()`로 캐시해 두 write가 재계산 없이 공유.

```
df = spark.read.parquet("s3://.../raw_events.parquet")  # 원천 읽기
cleaned = clean(df)          # 공통 정제
cleaned.persist()            # 핵심: 중간 데이터셋 캐시
cleaned.write.parquet("s3://.../legacy/")  # child1
cleaned.write.parquet("s3://.../new/")     # child2
```

- 실행 순서:
    - ① 원천 읽기 → 정제.
    - ② `.persist()` : 정제 결과를 메모리·디스크에 유지.
    - ③ 두 write가 캐시된 `cleaned`를 읽음. 원천 재읽기·재정제 없음.
- 핵심 라인: `cleaned.persist()`. 없으면 각 write가 원천부터 다시 읽어 정제가 2번 실행.
- 각주:
    - `.persist()` : 중간 결과 캐시. `.cache()`(메모리만)와 달리 저장 레벨 지정 가능.
    - `.write.parquet(path)` : DataFrame을 parquet로 저장.

---

- 엔지니어 독백:
    
    > 신입한테. `.persist()` 빼먹으면 로그상 잘 도는데 원천을 child 수만큼 다시 읽는다. 큰 parquet면 그냥 2배, 3배 요금이다. 한 DataFrame을 두 번 이상 write하면 persist 걸었나부터 봐라. 그리고 persist 걸었으면 잡 끝에 `.unpersist()`로 풀어줘라. 안 풀면 메모리 물고 있다가 다음 스테이지에서 OOM 난다.


### (5)최신트렌드

- 결론: 요즘은 수동 `.persist()` 대신, 저장 계층·엔진이 중간 데이터셋 재사용을 알아서 처리하는 쪽으로 간다.  
    
- Delta Lake 중간 테이블:
    - 정체 : ACID 트랜잭션을 지원하는 테이블 포맷.
    - 이전 한계 : `.persist()`는 잡이 끝나면 사라져, 다른 잡·다른 스케줄의 child는 재사용 불가.
    - 왜 요즘 쓰나 : 중간 결과를 Delta 테이블로 한 번 쓰면, 여러 child 잡이 시간·경계 넘어 재사용. 잡 간 공유가 편해서 많이 쓴다.  
        
- Spark AQE (reuse-exchange):
    - 정체 : 런타임 통계로 쿼리를 자동 최적화하는 Spark 기능.
    - 이전 한계 : 동일 연산 재사용을 개발자가 수동으로 `.persist()` 걸어 관리, 실수 잦음.
    - 왜 요즘 쓰나 : 같은 연산 갈래를 엔진이 자동 감지해 재사용. 설정 한 줄이라 부담 없이 켠다.  
        
- dbt (팬아웃 모델):
    - 정체 : SQL 변환을 모델 단위로 관리하는 도구.
    - 이전 한계 : 한 결과를 여러 다운스트림에 수동 연결, 의존이 코드에 안 드러남.
    - 왜 요즘 쓰나 : 하나의 모델을 여러 모델이 `ref()`로 참조 → 팬아웃 의존을 선언적으로 표현. SQL 팀이 선호.
    
- 실무 선호 (상황별):
    - Spark 잡 내부 최적화 → AQE. 켜두면 자동이라 기본값.
    - child가 별도 잡·재사용 필요 → Delta 중간 테이블.
    - SQL 중심 파이프라인 → dbt.  
        
- 엔지니어 독백:
    
    > 신입한테. AQE는 Spark 3 쓰면 그냥 켜라(`spark.sql.adaptive.enabled=true`). 근데 이거 켰다고 `.persist()`가 필요 없어지는 건 아니다. 잡 경계를 넘는 재사용은 여전히 Delta 중간 테이블로 명시적으로 써야 한다. 나는 "한 잡 안이면 persist/AQE, 잡 넘으면 Delta 테이블"로 딱 나눠서 판단한다.   


### (6)persist() vs cache()


- 결론: `.persist()`는 계산 결과를 메모리·디스크에 저장해 재계산을 막는 Spark 함수다.

#### persist가 나온 배경

- Spark는 lazy evaluation이다. `action`(write, count 등)이 호출될 때마다 그 DataFrame의 계산 과정을 처음부터 다시 실행한다.
- 한계 : 같은 DataFrame을 여러 번 쓰면 그때마다 원천 읽기·변환이 반복된다.
- 해결 : `.persist()`로 한 번 계산한 결과를 저장 → 이후엔 저장본을 재사용.

#### 헷갈리는 함수들

- `.cache()`
    - 정체 : `.persist()`의 축약형. 저장 위치를 `MEMORY_AND_DISK`(기본값)로 고정.
    - 차이 : 저장 레벨을 못 고른다.
- `.persist(StorageLevel)`
    - 정체 : 저장 레벨을 직접 지정하는 버전.
    - 옵션 예 : `MEMORY_ONLY`(메모리만), `MEMORY_AND_DISK`(메모리 넘치면 디스크), `DISK_ONLY`(디스크만).
- `.checkpoint()`
    - 정체 : 결과를 안정적 스토리지(HDFS/S3)에 저장하고 lineage(계산 계보)를 끊음.
    - 차이 : persist는 lineage를 유지(장애 시 재계산 가능), checkpoint는 아예 끊음(긴 lineage로 인한 성능 저하·OOM 방지).

#### 실무 선호

- 대부분 상황 → `.cache()`. 기본 레벨로 충분하고 짧다.
- 메모리 빠듯한 대용량 → `.persist(DISK_ONLY)`. 메모리 압박 회피.
- lineage가 수십 단계로 길어져 느려짐 → `.checkpoint()`. 계보를 끊어 안정화.
- 판단 기준 : 재사용만 필요하면 cache/persist, 계보 자체가 문제면 checkpoint.
- 엔지니어 독백:

    > 신입한테. cache랑 persist 구분에 너무 힘 빼지 마라. 실무 90%는 그냥 `.cache()`다. 진짜 중요한 건 "언제 거냐"인데, 한 DataFrame을 2번 이상 action에 쓸 때만 걸어라. 한 번만 쓰는데 캐시하면 메모리만 잡아먹고 이득 없다. 그리고 다 쓰면 `.unpersist()`로 꼭 풀어라 — 안 풀면 다음 스테이지에서 메모리 모자라 OOM 난다.
    

<br><br><br><br><br>




## 패턴#38 배타적 선택(Exclusive Choice)

### (1) 문제상황

**핵심 고통: 같은 파이프라인 안에서 날짜에 따라 다른 잡을 돌려야 하는데, Parallel Split은 항상 둘 다 돌린다.**

> 등장 컴포넌트
> 
> - **Airflow DAG**: 태스크 순서와 의존성을 선언하는 오케스트레이션 레이어. 데이터를 직접 만지지 않고 "무엇을 언제 실행할지"만 결정.
> - **Spark 배치 잡**: 실제 devices 원본 파일을 읽어 가공·저장하는 데이터 처리 레이어.
> - **CSV 출력 / Delta Lake 테이블**: 같은 데이터를 담는 두 종류의 싱크(sink). 마이그레이션 대상.

**엔지니어 독백**
> 원래 devices 파이프라인은 CSV 하나만 뱉었다. 레거시 BI 툴이 CSV만 읽을 줄 알아서다. 그런데 Delta Lake로 옮기기로 했고, 검증 기간 동안 병렬 출력(#37 Parallel Split)으로 CSV와 Delta를 동시에 썼다. 두 결과가 같은지 두 달 비교했고, 문제없었다.
>이제 2024-01-01부터는 Delta만 쓰기로 정해졌다. 그런데 문제는 **백필(backfill)** 이다. 2023년 11월치 30일분을 다시 돌릴 일이 생겼는데, DAG는 여전히 CSV 잡과 Delta 잡을 둘 다 실행한다.


"그럼 새 DAG를 하나 더 만들면 되지 않나?" 싶었다. 그런데 DAG ID가 바뀌면 Airflow 메타 DB의 task instance 기록이 새 DAG ID로 분리된다. 지난 2년치 실행 이력·SLA 통계·백필 로그가 전부 끊긴다.

결국 필요한 건 **하나의 DAG 안에서, 공통 부모 태스크 다음에 두 브랜치 중 하나만 고르는 지점**이다. 둘 다가 아니라 **배타적으로 하나만**. 그게 Exclusive Choice다.

- 백필도 delta로 실행되면 기존 csv파일이 삭제처리되지않고 delta만 추가됨
- Delta 테이블에 2023-11-15 데이터가 append로 다시 들어감 → 중복 레코드 발생

|시기|Parallel Split(#37) 상태|2023-11-15 파티션|
|---|---|---|
|2023-11-15 새벽 02:00 (정기 실행)|CSV 잡 + Delta 잡 **둘 다 실행** (검증 기간)|CSV 생성 ✅ / **Delta 생성 ✅**|
|2023-12-31|검증 종료|—|
|2024-01-01|Delta만 쓰기로 결정|—|
|2024-03-20 (오늘)|**DAG는 아직 Parallel Split 그대로**|—|
|2024-03-20 백필 실행|CSV 잡 + Delta 잡 **또 둘 다 실행**|Delta에 **두 번째로** 씀|

**핵심: 2023-11-15는 검증 기간 한복판이었다. 그날 Delta 잡이 이미 정상적으로 돌았고, Delta 테이블에 4,000만 건이 들어갔다.**

그런데 오늘 백필을 돌리면 DAG는 여전히 두 브랜치를 다 던진다 → Delta 잡이 같은 4,000만 건을 `append` 로 또 쓴다.

<결과>
```
s3://.../devices_delta/dt=2023-11-15/
├── part-00000-a1b2.snappy.parquet   ← 2023-11-15 실행분 (4,000만 건)
├── part-00001-c3d4.snappy.parquet
├── part-00000-e5f6.snappy.parquet   ← 2024-03-20 백필분 (같은 4,000만 건)
└── part-00001-g7h8.snappy.parquet
```

`SELECT COUNT(*) FROM devices WHERE dt='2023-11-15'` → **8,000만 건**
- 브라우저별 방문자 수 2배



### (2) 솔루션

**핵심 해결책**: 분기 전에 "조건 평가 태스크"를 하나 추가해서, 그 결과에 따라 하나의 경로만 실행한다.


**구현 위치 선택지 — 2가지**

1. 오케스트레이션 레이어 (Airflow DAG 수준)
2. 데이터 처리 레이어 (PySpark job 코드 수준)


**1. 오케스트레이션 레이어 구현**

- Airflow의 `BranchPythonOperator` 사용
- 정체: 조건을 평가해서 다음에 실행할 태스크 ID를 **문자열로 반환**하는 전용 태스크 타입
- 반환된 태스크 ID의 경로만 실행, 나머지는 자동으로 **skip** 처리
- 구조:

```
공통 태스크
    ↓
[BranchPythonOperator] ← 조건 평가
    ↓              ↓
load_delta     load_csv     ← 하나만 실행됨
```

- 조건 평가 기준 예시
- 실행 날짜 기반: `execution_date >= 2024-01-01` → Delta Lake 경로
- 입력 스키마 기반: 컬럼 수가 3개 이상 → 신규 테이블 경로
- 외부 파라미터 기반: `--output_type delta` 인자값 → Delta Lake 경로


**2. 데이터 처리 레이어 구현**

- PySpark job 코드 안에서 if-else로 분기
- `argparse`로 `--output_type` 파라미터를 받아서 조건 처리
- Factory 패턴(SWE 디자인 패턴) 활용
- 정체: 조건 분기 로직을 `OutputGenerationFactory` 클래스 안에 숨기고, 외부에는 단일 인터페이스만 노출
- 클라이언트 코드는 조건을 모른 채 `factory.write()`만 호출하면 됨



**어디서 구현할지 — 트레이드오프**

**상황 설정**: 디바이스 로그 파이프라인, 2024-01-01 이전 → CSV, 이후 → Delta Lake

- 오케스트레이션 레이어 구현 시 DAG 모습:
```
[ingest_data] → [format_router] → [load_csv]
                               → [load_delta]
```

- 처리 레이어 구현 시 DAG 모습:
```
[ingest_data] → [load_job]
```

- `load_job.py` 안을 열어야만 분기 존재를 알 수 있음

|구현 위치|장점|단점|적합한 상황|
|---|---|---|---|
|오케스트레이션 레이어|분기가 DAG에서 눈에 보임, 히스토리 추적 용이|태스크 수 증가, 관리 복잡도 증가|경로마다 다른 job을 실행해야 할 때|
|처리 레이어|코드 한 곳에서 관리, 배포 단순|분기 로직이 코드 안에 숨어있음|같은 job에서 출력 형식만 바꿀 때|

**실무 선호**: 오케스트레이션 레이어
- 이유: 6개월 뒤 신입 팀원이 장애 디버깅 시
- 오케스트레이션: DAG 화면만 봐도 분기 구조 즉시 파악
- 처리 레이어: DAG엔 태스크 하나뿐 → 단순해 보임 → job 코드 뜯어봐야 분기 조건 발견 → 디버깅 수 시간 낭비
- 분기 로직이 코드 안에 숨어있으면 → 운영 리스크
---


### (3) 결과 (Consequences)

**핵심**: 분기가 많아질수록 파이프라인 복잡도가 기하급수적으로 증가한다.



**위험 1 — 복잡도 폭발**
- Exclusive Choice는 분기를 원하는 만큼 추가할 수 있음
- 분기가 늘어날수록 DAG이 이렇게 변함:
```
[ingest_data] → [format_router] → [load_csv]
                               → [load_delta]
                               → [load_parquet]
                               → [load_iceberg]
```

- 각 분기가 또 다른 분기를 낳으면:
```
[format_router] → [load_delta] → [region_router] → [load_us]
                                                 → [load_eu]
               → [load_csv]   → [version_router] → [load_v1]
                                                 → [load_v2]
```

- 결과: DAG을 처음 보는 사람은 흐름 파악 자체가 불가능해짐



**위험 2 — 숨겨진 로직 (Hidden Logic)**
- 처리 레이어에 분기를 구현하면 job 코드 안에 분기가 묻힘
- 시간이 지나면 본인도 "이 job이 조건에 따라 다른 결과를 낸다"는 사실을 잊음
- 장애 발생 시 → DAG엔 단순해 보임 → 원인 찾는 데 수 시간 소요


**위험 3 — 무거운 조건 (Heavy Conditions)**
- 조건 평가 시 데이터를 직접 읽어야 하는 경우 발생
- 예) "파일 안의 레코드 수가 100만 건 이상이면 Delta Lake로, 아니면 CSV로"
- 이 조건 하나 때문에 전체 데이터셋을 스캔해야 함 → 실행 시간 급증
- 실무 권고: 가능하면 **메타데이터 기반 조건** 사용
- 좋은 예: 실행 날짜, 파일 크기, 스키마 컬럼 수 → 데이터 스캔 없이 빠르게 평가
- 나쁜 예: 레코드 수, 특정 컬럼의 값 분포 → 데이터 전체 읽어야 함


**복잡도 자가 진단법 — Rubber Duck Debugging**
- 방법: 파이프라인을 "프로젝트에 새로 합류한 팀원"에게 설명한다고 가정하고 말로 설명해봄
- 설명이 길어지고 질문이 많이 나온다면 → 파이프라인이 너무 복잡하다는 신호
- 이 시점에서 고려할 대안:
- 분기를 늘리는 대신 `start_date`, `end_date` 파라미터를 받는 별도 파이프라인으로 분리
- 분기 조건 자체를 단순화
---



### (4) 예시

**핵심**: 오케스트레이션 레이어(Airflow)와 처리 레이어(PySpark) 각각의 구현 코드를 본다.

---

**예시 1 — Airflow BranchPythonOperator (오케스트레이션 레이어)**

```python
def get_output_format_route(**context):
    migration_date = pendulum.datetime(2024, 2, 3)
    execution_date = context['execution_date']
    if execution_date >= migration_date:
        return 'load_job_trigger_delta'   # 이 태스크 ID만 실행
    else:
        return 'load_job_trigger_csv'     # 이 태스크 ID만 실행

format_router = BranchPythonOperator(
    task_id='format_router',
    python_callable=get_output_format_route,
    provide_context=True
)
```

- `get_output_format_route`: 실행 날짜를 보고 어느 태스크 ID를 실행할지 결정하는 함수
- `context['execution_date']`: Airflow가 자동으로 넘겨주는 현재 실행 날짜
- **핵심 라인**: `return 'load_job_trigger_delta'`
- 태스크 ID 문자열을 반환하는 순간, Airflow가 해당 태스크만 실행하고 나머지는 skip 처리
- `provide_context=True`: Airflow context(실행 날짜 등)를 함수 인자로 전달하도록 설정

---

**예시 2 — PySpark Factory 패턴 (처리 레이어, 외부 파라미터 기반)**

```
class OutputType(str, Enum):
    delta_lake = 'delta'
    csv = 'csv'

parser = argparse.ArgumentParser()
parser.add_argument('--output_type', required=True, type=OutputType)
args = parser.parse_args()

output_generation_factory = OutputGenerationFactory(args.output_type)
spark_session = output_generation_factory.get_spark_session()
raw_data = spark_session.read.parquet('/data/devices/')
output_generation_factory.write_devices_data(raw_data, args.output_dir)
```

```
class OutputGenerationFactory:
    def __init__(self, output_type: OutputType):
        self.type = output_type

    def get_spark_session(self) -> SparkSession:
        if self.type == OutputType.delta_lake:
            return configure_spark_with_delta_pip(SparkSession.builder).getOrCreate()
        else:
            return SparkSession.builder.getOrCreate()

    def write_devices_data(self, devices_data: DataFrame, output_location: str):
        if self.type == OutputType.delta_lake:
            devices_data.write.format('delta').save(output_location)
        else:
            devices_data.coalesce(1).write.format('csv').save(output_location)
```

- `OutputType`: delta / csv 두 가지 출력 타입을 Enum으로 정의 → 오타 방지
- `--output_type`: job 실행 시 외부에서 주입하는 파라미터 (고정 명칭)
- **핵심 라인**: `OutputGenerationFactory(args.output_type)`
- 이 한 줄로 분기 로직 전체가 factory 안으로 들어감
- 클라이언트 코드는 조건을 몰라도 됨, `write_devices_data()` 호출만 하면 알아서 분기
- `coalesce(1)`: CSV 출력 시 파일을 1개로 합치는 옵션

---

**예시 3 — PySpark 스키마 기반 분기 (처리 레이어, 메타데이터 기반)**

```
input_dataset = spark.read.parquet('/data/devices/')
input_schema = input_dataset.schema

output_location = DemoConfiguration.DEVICES_TABLE_LEGACY
if len(input_schema.fields) >= 3:
    output_location = DemoConfiguration.DEVICES_TABLE_SCHEMA_CHANGED
```

- `input_dataset.schema`: 데이터를 전부 읽지 않고 **스키마 메타데이터만** 읽음 → 빠름
- **핵심 라인**: `len(input_schema.fields) >= 3`
- 컬럼 수라는 메타데이터만으로 분기 결정 → 데이터 스캔 없음 → 실행 비용 최소화
- 이전 Consequences에서 언급한 "메타데이터 기반 조건"의 실제 구현 예시

---

> 엔지니어 독백
> 
> 신입 때 처리 레이어에 if-else 분기 잔뜩 넣고 "코드 한 곳에서 관리되니까 깔끔하다"고 생각했어. 근데 3개월 뒤 장애 났을 때 DAG 봐도 아무것도 안 보이고, job 코드 열어서 분기 10개 찾는 데 반나절 날렸어. 그 이후로 무조건 오케스트레이션 레이어에 분기 표현해. 코드가 좀 늘어나도 DAG에서 눈에 보이는 게 훨씬 낫거든. 그리고 조건 평가할 때 데이터 스캔하는 실수도 자주 봐 — 스키마 컬럼 수, 실행 날짜 같은 메타데이터로 해결되는 경우가 대부분이니까 데이터 건드리기 전에 메타데이터로 먼저 해결할 수 있는지 확인하는 습관 들여.


### (5) 질문
> 팩토리 패턴?
> **Factory 패턴**: 조건에 따라 다른 객체를 만들어주는 공장.


**예시 — 음료 주문**
```
class DrinkFactory:
    def get_drink(self, drink_type: str):
        if drink_type == 'coffee':
            return Coffee()
        else:
            return Tea()

class Coffee:
    def drink(self):
        print("커피 마심")

class Tea:
    def drink(self):
        print("차 마심")
```

**사용하는 쪽**

```
factory = DrinkFactory()
drink = factory.get_drink('coffee')
drink.drink()  # 커피 마심
```


**핵심**
- 사용하는 쪽은 `'coffee'` 만 넘김
- 어떤 객체를 만들지는 Factory 안에서 결정
- 사용하는 쪽은 Coffee, Tea 클래스 존재 자체를 몰라도 됨

### (6) 최신트렌드

**핵심**: Exclusive Choice 패턴은 요즘 오케스트레이션 도구의 발전으로 더 쉽고 명시적으로 구현 가능해졌다.

**1. Airflow Dynamic Task Mapping (Airflow 2.3+)**

- 이전 한계: `BranchPythonOperator`는 분기 경로를 코드에 **하드코딩** 해야 했음
- 분기 추가할 때마다 DAG 코드 수정 필요
- 요즘 쓰는 이유: Dynamic Task Mapping으로 분기 경로를 **런타임에 동적으로** 결정 가능
- 조건 목록이 바뀌어도 DAG 코드 수정 없이 대응 가능
- 실무 체감: 출력 타입이 2개 → 5개로 늘어날 때 DAG 코드 안 건드려도 됨

**2. AWS Step Functions**

- 이전 한계: Airflow는 서버 띄우고 관리해야 하는 운영 부담 있음
- 요즘 쓰는 이유: 서버리스라 인프라 관리 없이 Exclusive Choice 패턴 구현 가능
- `Choice` 스테이트가 내장되어 있어 조건 분기를 JSON 설정만으로 표현
- 실무 체감: 소규모 파이프라인에서 Airflow 띄우기 부담스러울 때 Step Functions이 훨씬 빠름

**3. Azure Data Factory — If Condition Activity**

- 이전 한계: 조건 분기를 코드로 직접 구현해야 했음
- 요즘 쓰는 이유: UI에서 드래그앤드롭으로 Exclusive Choice 패턴 구성 가능
- 코드 없이 분기 로직을 시각적으로 표현
- 실무 체감: 데이터 엔지니어링 경험 없는 팀원도 파이프라인 분기 구조 파악 가능

**4. Databricks Workflows**

- 이전 한계: Spark job의 조건 분기를 Airflow DAG과 따로 관리해야 했음
- 요즘 쓰는 이유: Spark job 실행과 조건 분기를 **같은 플랫폼**에서 관리 가능
- `if/elseif` 조건을 Workflow UI에서 직접 설정
- 실무 체감: Airflow 없이 Databricks 안에서 오케스트레이션까지 해결되니까 스택이 단순해짐

---

> 엔지니어 독백
> 
> 요즘 팀마다 오케스트레이션 도구가 다른데, 공통적으로 느끼는 건 "분기 로직을 코드 밖으로 꺼내는 방향"으로 다들 가고 있다는 거야. Airflow BranchPythonOperator도 결국 Python 코드인데, Step Functions이나 ADF는 아예 설정 파일이나 UI로 분기를 표현하거든. 코드가 줄어드는 만큼 실수도 줄고 신입도 파악하기 쉬워. 근데 무조건 UI 도구가 좋은 건 아니야 — 복잡한 조건은 결국 코드가 더 명확할 때가 있어서 팀 상황에 맞게 선택해야 해.







## 패턴#39 Single Runner 패턴
### (1) 문제상황

**핵심 고통**: 파이프라인 여러 개가 동시에 실행되면, 이전 실행 결과에 의존하는 증분 처리(incremental processing)가 망가진다.

---

**배경 — 증분 처리가 뭔지부터**

- 매일 쌓이는 로그 1억 건을 매번 전부 처리하면 비용, 시간 낭비
- 그래서 "어제까지 처리했으니 오늘치만 처리하자" → 증분 처리
- **핵심 전제**: 어제 처리가 완전히 끝나야 오늘치 시작점을 알 수 있음


**세션화가 왜 순서에 민감한지**
- 유저 A의 클릭 이벤트 로그:
```
1월 1일: 클릭 3번 → 세션 종료 시각 23:50
1월 2일: 클릭 5번 → 세션 시작 시각 = 1월 1일 세션 종료 시각 기준으로 계산
1월 3일: 클릭 2번 → 세션 시작 시각 = 1월 2일 세션 종료 시각 기준으로 계산
```
- 1월 2일 세션을 계산하려면 → 1월 1일 세션 종료 시각이 DB에 저장되어 있어야 함
- 1월 1일 처리가 안 끝난 상태에서 1월 2일 처리 시작하면 → 시작점이 NULL → 세션 계산 오류



**실무 시나리오 — 장애 발생**
- 상황: Airflow DAG이 매일 자정 실행, 1월 1일~3일 서버 장애로 3일치 밀림
- 1월 4일 서버 복구 → Airflow가 밀린 3일치를 **동시에** 실행 시작
```
[1월 1일 job] ──┐
[1월 2일 job] ──┼── 동시 실행
[1월 3일 job] ──┘
```



**세션화 처리 방식 — 먼저 이해**
- 세션화 job은 매일 실행될 때 이런 순서로 동작함:
1. DB에서 "어제 마지막 세션 종료 시각" 조회
2. 그 시각 이후부터 오늘 로그 읽기 시작
3. 세션 계산
4. 결과를 DB에 저장 (다음 날 job이 이걸 읽어감)


**정상 실행 흐름**
```
[1월 1일 job 실행]
1. DB 조회: "12월 31일 세션 종료 시각 = 23:50" 읽음
2. 23:50 이후 로그 읽기
3. 세션 계산 완료
4. DB 저장: "1월 1일 세션 종료 시각 = 23:55" 저장
             ↑
             이게 저장되어야 1월 2일 job이 읽을 수 있음

[1월 2일 job 실행] ← 1월 1일 job 완전히 끝난 후 시작
1. DB 조회: "1월 1일 세션 종료 시각 = 23:55" 읽음 ← 정상
2. 23:55 이후 로그 읽기
3. 세션 계산 완료
```


**동시 실행 시 실패 흐름**
```
[1월 1일 job] 실행 중 (아직 DB 저장 전)
[1월 2일 job] 동시에 실행 시작

1월 2일 job:
1. DB 조회: "1월 1일 세션 종료 시각" 읽으려고 함
            ↑
            1월 1일 job이 아직 계산 중 → DB에 없음 → NULL 반환

2. 시작점이 NULL이니까 → 로그 전체를 처음부터 읽거나 / 에러 발생
3. 잘못된 시작점으로 세션 계산 → 데이터 오염
```

**장애 발생 시 — 왜 동시에 도냐**
- 1월 1일~3일 서버 장애 → 3일치 job이 실행 못 함 → **밀린 상태(backlog)**로 쌓임
- 장애 후: 밀린 것들이 한꺼번에 몰려서 동시 실행 → `max_active_runs` 제한 안 걸려있으면 그냥 다 동시에 돌아버림
- Airflow 기본 설정: `max_active_runs` 값이 기본적으로 **제한 없음(또는 크게 설정)**

**핵심 포인트**
- 1월 2일 job 입장에서는 DB에 값이 있을 거라고 **가정**하고 시작함
- 근데 1월 1일 job이 아직 안 끝났으니 DB에 값이 없음
- 없는 값(NULL)으로 계산 시작 → 세션 시작점 자체가 틀림


**결과**
- 1월 2일, 3일 세션 데이터 전체 오염
- 수동으로 1월 2일, 3일 데이터 삭제 후 재처리 필요
- 재처리하는 동안 다운스트림 대시보드, ML 피처 전부 영향받음



**개선 욕구 → 새로운 문제**
- "무조건 1월 1일 → 1월 2일 → 1월 3일 순서대로 하나씩만 실행하고 싶다"
- 근데 Airflow 기본 설정은 밀린 실행을 빠르게 처리하려고 동시 실행이 기본값
- 동시 실행 자체를 막는 설정이 필요 → **Single Runner 패턴** 등장
- 이름 의미: 파이프라인 인스턴스를 항상 **단 하나(Single)만** 실행되도록 보장


### (2)솔루션

**핵심 해결책**: `max_active_runs = 1` 설정으로 파이프라인 인스턴스가 항상 하나만 실행되도록 강제한다.

---

**솔루션 구조**

- Airflow DAG 선언 시 `max_active_runs = 1` 설정
- 효과: 현재 실행 중인 DAG 인스턴스가 있으면 다음 인스턴스는 **대기(queued)** 상태로 멈춤
- 이전 인스턴스 완료 후 → 다음 인스턴스 자동 시작

```
[1월 1일 job] 실행 중
[1월 2일 job] 대기 (queued)   ← max_active_runs=1 이라 못 시작
[1월 3일 job] 대기 (queued)

→ 1월 1일 job 완료
→ 1월 2일 job 시작
→ 1월 2일 job 완료
→ 1월 3일 job 시작
```

---

**Airflow 설정 방법**

```
from airflow import DAG
from datetime import datetime

dag = DAG(
    dag_id='sessionization_pipeline',   # DAG 이름 (내가 정하는 변수)
    schedule_interval='@daily',         # 매일 실행
    max_active_runs=1,                  # 핵심 설정 — 동시 실행 1개로 제한
    start_date=datetime(2024, 1, 1),
)
```

- `dag_id`: 내가 정하는 DAG 이름
- `schedule_interval`: 실행 주기 (`@daily`, `@hourly`, `0 * * * *` 크론 표현식 등)
- **핵심 라인**: `max_active_runs=1`
- 이 한 줄이 동시 실행을 막는 전부
- Airflow가 현재 실행 중인 인스턴스 수를 체크 → 1개 초과 시 다음 인스턴스 대기

---

**Azure Data Factory 설정 방법**

- ADF는 `concurrency` 속성을 `1`로 설정
- Airflow의 `max_active_runs=1`과 동일한 효과

---

**주의사항**

- `max_active_runs=1`은 순서를 보장하지만 **백필 속도가 느려짐**
- 3일치 밀리면 → 순차 실행이라 3배 시간 소요
- 병렬로 돌리면 빠르지만 데이터 오염 → 트레이드오프


### (3)결과

**핵심**: 순서 보장을 얻는 대신 백필 성능을 잃는다. 이게 Single Runner의 본질적인 트레이드오프야.


**장점**
- 실행 순서 100% 보장
- 이전 job 완료 후 다음 job 시작 → 증분 처리 데이터 오염 없음
- 설정 한 줄(`max_active_runs=1`)로 구현 가능
- 파이프라인 인스턴스 간 상태 충돌 원천 차단


**단점 1 — 백필 성능 저하**
- 30일치 밀린 상황에서 Single Runner는 순차 실행
- job 1개당 1시간 걸리면 → 30일치 백필 = 30시간 소요
- 병렬로 돌리면 1~2시간이면 끝날 걸 30배 시간 낭비


**단점 2 — 지연(Latency) 누적**
- 1월 1일 job이 어떤 이유로 오래 걸리면  
    → 1월 2일 job은 그만큼 대기  
    → 1월 3일 job은 더 오래 대기
- 하나가 느려지면 뒤에 줄 선 것들이 전부 밀림


**단점 3 — 병목 지점 명확**
- 파이프라인 하나가 실패하면 → 뒤에 줄 선 모든 인스턴스가 블록됨
- 실패한 job 수동 재시작 전까지 전체 파이프라인 멈춤
```
[1월 1일 job] 실패 ← 여기서 멈춤
[1월 2일 job] 대기 중
[1월 3일 job] 대기 중
     ↑
수동으로 1월 1일 job 고치고 재시작해야 풀림
```


**트레이드오프 결론**

|상황|선택|
|---|---|
|증분 처리, 이전 결과에 의존하는 파이프라인|Single Runner 필수|
|백필이 자주 발생하는 파이프라인|백필 전용 파이프라인 별도 구성 고려|
|각 실행이 독립적인 파이프라인|Single Runner 불필요 → Concurrent Runner 적합|

---

> 엔지니어 독백
> 
> Single Runner 설정해놓고 안심했는데, 어느 날 30일치 백필 요청이 들어온 거야. 순차 실행이라 30시간 걸린다고 하니까 비즈니스팀이 난리났지. 그때 배운 게 백필 전용 파이프라인을 따로 만들어두는 거야. 운영용은 `max_active_runs=1`로 순서 보장하고, 백필용은 날짜 범위 파라미터 받아서 별도로 돌리는 식으로. 두 개를 분리해두면 운영 파이프라인 건드리지 않고 백필만 빠르게 처리할 수 있어.

<질문>
> Q.`max_active_runs=1` 이 설정은 그러면 어느 단위에 적용되는것인가? 
> 어떤건 싱글러너로하고 어떤건 병렬도 되도록할려면?
> 
> A. **`max_active_runs=1`은 DAG 단위로 적용돼.**

**DAG 단위 적용 의미**
- Airflow에서 DAG = 파이프라인 하나
- `max_active_runs=1`은 해당 DAG의 인스턴스(실행 횟수)를 1개로 제한
- 다른 DAG에는 영향 없음

```
DAG A (sessionization)     max_active_runs=1   ← 순차 실행
DAG B (user_stats)         max_active_runs=16  ← 병렬 실행 허용
DAG C (revenue_report)     max_active_runs=1   ← 순차 실행
```

- DAG A가 실행 중이어도 DAG B는 얼마든지 동시에 실행 가능
- 제한은 **같은 DAG의 인스턴스끼리만** 적용



### (4) 예시

**핵심**: Airflow DAG에서 `max_active_runs=1` 설정으로 순차 실행을 보장하는 실제 구현.


**예시 — 디바이스 세션화 파이프라인**
```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

def run_sessionization(**context):
    execution_date = context['execution_date']
    # DB에서 이전 세션 종료 시각 조회
    last_session_end = get_last_session_end(execution_date)
    # 이전 세션 종료 시각 이후 로그만 처리
    process_sessions(start_from=last_session_end)
    # 처리 결과 DB에 저장 → 다음 job이 읽어감
    save_session_end(execution_date)

dag = DAG(
    dag_id='sessionization_pipeline',
    schedule_interval='@daily',
    start_date=datetime(2024, 1, 1),
    max_active_runs=1,              # 핵심 설정 ###################
    catchup=True,                   # 밀린 날짜 백필 허용
)

sessionization_task = PythonOperator(
    task_id='run_sessionization',
    python_callable=run_sessionization,
    provide_context=True,
    dag=dag,
)
```

- `dag_id`: 내가 정하는 DAG 이름
- **핵심 라인**: `max_active_runs=1`
- 현재 실행 중인 DAG 인스턴스가 있으면 다음 인스턴스 대기
- 완료 후 자동으로 다음 인스턴스 시작
- `catchup=True`: 밀린 날짜 자동 백필 활성화
- `max_active_runs=1`과 함께 쓰면 밀린 날짜를 **순서대로** 하나씩 처리
- `provide_context=True`: `execution_date` 등 Airflow context를 함수 인자로 전달

---

**실행 흐름 — 3일치 밀린 상황**

```
1월 4일 서버 복구

T+0분:  1월 1일 job 시작
T+60분: 1월 1일 job 완료 → DB에 세션 종료 시각 저장
T+60분: 1월 2일 job 시작 ← 1월 1일 결과 정상적으로 읽음
T+120분: 1월 2일 job 완료
T+120분: 1월 3일 job 시작
T+180분: 1월 3일 job 완료
```

---

**비교 — `max_active_runs` 설정값별 동작**
```
# 동시 실행 허용 (기본값)
max_active_runs=16

# 실행 흐름
T+0분: 1월 1일, 2일, 3일 job 동시 시작 → 데이터 오염


# 순차 실행 (Single Runner)
max_active_runs=1

# 실행 흐름
T+0분:   1월 1일 job 시작
T+60분:  1월 2일 job 시작
T+120분: 1월 3일 job 시작 → 순서 보장
```

< 엔지니어 독백>
> `max_active_runs=1` 설정 자체는 한 줄이라 쉬운데, `catchup=True`랑 같이 쓸 때 주의해야 해. `catchup=True`는 밀린 날짜를 전부 백필하겠다는 설정인데, 파이프라인 처음 배포할 때 `start_date`를 과거로 잡으면 그날부터 오늘까지 전부 순차 실행하려고 해. 수백 일치가 밀려있으면 며칠씩 걸리는 거야. 그래서 신규 파이프라인 배포할 때는 `start_date`를 배포일 기준으로 잡거나, `catchup=False`로 설정하고 백필은 별도로 제어하는 게 안전해.



**예시 1 — Apache Airflow: DAG 단위 + 태스크 단위 동시 제한 (Example 6-21)**
오늘치 데이터를 어제치와 비교하는 파이프라인 — 순서가 깨지면 비교 자체가 틀려지는 케이스.
```
with DAG('visits_trend_generator',
    max_active_runs=1,          # DAG 단위: 동시 실행 1개로 제한
    default_args={
        'depends_on_past': True,  # 태스크 단위: 이전 실행 태스크 성공해야 현재 실행 시작
    }
):
```
- `max_active_runs=1`: DAG 인스턴스 전체를 1개로 제한
- **핵심 라인**: `depends_on_past=True`
- DAG 레벨 제한과 별개로 **태스크 레벨**에서도 순서 보장
- 예) 1월 1일 `load_data` 태스크가 실패하면 → 1월 2일 `load_data` 태스크는 시작 안 함
- `max_active_runs=1`만 있으면 DAG은 하나씩 실행되지만, 태스크 단위 실패 시 다음 DAG이 그냥 시작될 수 있음
- 두 설정의 역할 차이:

```
max_active_runs=1   → DAG 인스턴스 단위 제한 (파이프라인 전체)
depends_on_past=True → 태스크 단위 제한 (태스크 하나하나)
```

<엔지니어 독백>
>실무에서 Single Runner 설정은 파이프라인 배포 전 체크리스트 1번이야. "이 파이프라인이 이전 실행 결과에 의존하냐?" → YES면 무조건 `max_active_runs=1` 박고 시작해. 근데 `max_active_runs=1`만 설정하고 `depends_on_past`를 빠뜨리는 실수를 자주 봐. DAG 단위는 막혔는데 태스크 단위 보호가 없으면 구멍이 생기거든.



**예시 2 — AWS EMR: 클러스터 단위 동시 실행 제한**
Airflow 없이 EMR에서 직접 Spark job을 순차 실행해야 할 때.
- `StepConcurrencyLevel` 파라미터로 클러스터 내 동시 실행 job 수 제한
- `StepConcurrencyLevel=1` → 클러스터 안에서 job 하나씩만 실행
```
aws emr create-cluster \
    --name "sessionization-cluster" \
    --step-concurrency-level 1    # 핵심: 클러스터 단위 동시 실행 1개로 제한
```

- Airflow의 `max_active_runs`가 DAG 단위라면, 이건 **EMR 클러스터 단위** 제한
- Airflow 없이 EMR만 쓰는 환경에서 Single Runner 패턴 구현하는 방법



**예시 3 — Azure Data Factory: Concurrency 설정**

ADF UI에서 파이프라인 설정 → `Concurrency = 1`
- Airflow와 **동작 방식이 다름** — 주의 필요
- Airflow `max_active_runs=1`: 실행 중인 인스턴스 있으면 **스케줄 자체를 안 함**
- ADF `Concurrency=1`: 실행을 **큐(queue)에 쌓아둠** → 큐 최대 100개 → 초과 시 `429 Too Many Requests` 에러
```
# Airflow 동작
실행 중 → 다음 스케줄 건너뜀 (skip)

# ADF 동작
실행 중 → 다음 실행을 큐에 대기 → 큐 100개 초과 시 429 에러
```
- ADF는 "막는" 게 아니라 "줄 세우는" 방식 → 큐가 쌓이면 에러 발생 가능



**예시 4 — 네이티브 지원 없는 환경: Readiness Marker 패턴**
위 3가지 프레임워크 외 환경에서 Single Runner 구현이 필요할 때.
- 이전 실행 완료 여부를 체크하는 **마커 파일/레코드**를 두고 다음 실행이 대기
- 단점: 마커 체크 로직을 직접 구현해야 함, 다음 실행 트리거 자체는 막지 못함

---
<엔지니어 독백>
> ADF 쓰다가 큐 100개 꽉 차서 `429` 에러 터지는 거 처음 보면 당황해. Airflow처럼 그냥 skip될 거라고 기대하거든. ADF는 큐에 쌓는 방식이라 장애 복구 후 밀린 실행이 100개 넘으면 그때부터 에러야. 그래서 ADF 쓸 때는 밀린 실행 수 모니터링 알람을 꼭 걸어둬야 해.



<질문>
> 1.emr은 저렇게 클러스터 생성할때 까먹고 설정안했을때 어디가서 수정해서 설정해주나? 설정파일이 있을거 같은데 ? 
> 
> 2.azure , ADF는 그럼 Concurrency=1 로 설정하면 큐에 넣는다는건지 원래 큐를 기반으로 돌아가는건지? 그러면 동시성 작업수를 설정할 필요가 없잖아? 공식문서 기반으로 답변 작성해줘.


**1. EMR — 클러스터 생성 후 수정 방법**
실행 중인 클러스터에서 `ModifyCluster` API로 `StepConcurrencyLevel`을 수정할 수 있다. 
```
# 실행 중인 클러스터의 동시 실행 수 변경
aws emr modify-cluster \
    --cluster-id j-2AXXXXXXGAPLF \   # 클러스터 ID (AWS 콘솔에서 확인)
    --step-concurrency-level 1        # 변경할 동시 실행 수
```
- `--cluster-id`: 수정할 클러스터 ID (고정 명칭, 값은 AWS 콘솔에서 확인)
- `--step-concurrency-level`: 동시 실행 가능한 step 수 (내가 정하는 값, 1~256)

**AWS 콘솔에서도 수정 가능**
- EMR 콘솔 → Clusters → 클러스터 선택 → Steps 탭 → Attributes 섹션 → Edit → 값 변경

**주의사항**
- step이 실행 중일 때 변경하면 `ActionOnFailure` 설정이 의도대로 동작하지 않을 수 있으므로 step이 없을 때 변경하는 걸 권고 
- 기본값은 1 (Single Runner) → 설정 안 하면 이미 Single Runner 상태 



**2. ADF — Concurrency 동작 방식**
- Concurrency 미설정 시 최대값 없음 → Concurrency 한도 도달 시 추가 실행은 큐에 대기 → 이전 실행 완료 후 자동 시작 
- 즉, Concurrency=1로 설정하면 1개 인스턴스만 실행되고 나머지는 큐에서 대기 —
- → 이게 ADF의 기본 동작 방식이야. 

**Airflow vs ADF 동작 차이**

|-|Airflow|ADF|
|---|---|---|
|실행 중 인스턴스 있을 때|다음 스케줄 **skip**|큐에 **대기**|
|한도 초과 시|새 실행 안 만듦|큐에 쌓다가 100개 초과 시 `429` 에러|
|기본값|`max_active_runs=16`|Concurrency 무제한|

**"그러면 Concurrency 설정이 왜 필요하냐"는 질문에 대해**
- Concurrency 미설정 시 여러 실행이 동시에 겹쳐서 실행될 수 있고, 특히 Tumbling Window 트리거에서는 최대 50개까지 동시 실행될 수 있음
- 즉 **큐에 넣는 기준**을 설정하는 거야
- `Concurrency=1`: 1개 실행 중이면 나머지 전부 큐 대기 → Single Runner
- `Concurrency=5`: 5개까지 동시 실행, 나머지 큐 대기 → Concurrent Runner
- 미설정: 한도 없이 전부 동시 실행





### (5) 최신트렌드

**핵심**: 요즘은 동시 실행 제어를 설정 한 줄이 아니라 플랫폼 수준에서 더 정교하게 다룬다.


**1. Databricks Workflows — Job Cluster 단위 동시 실행 제어**
- 이전 한계: Airflow `max_active_runs=1`은 DAG 단위 제한이라 Spark job 내부 동시성은 별도 제어 필요
- 요즘 쓰는 이유: Databricks Workflows에서 `Max concurrent runs=1` 설정 한 줄로 Spark job 실행부터 오케스트레이션까지 같은 플랫폼에서 제어 가능
- 체감 이점: Airflow 서버 따로 안 띄워도 됨 → 스택 단순화



**2. Apache Airflow 2.x — `depends_on_past` + Dataset-aware Scheduling**
- 이전 한계: `max_active_runs=1`만으로는 "데이터가 준비됐을 때" 실행 시작을 보장 못 함 → 시간 기반 스케줄이라 데이터 없어도 실행 시도
- 요즘 쓰는 이유: Airflow 2.4+의 Dataset-aware Scheduling — 특정 데이터셋이 업데이트될 때만 DAG 실행
- 시간 기반이 아니라 **데이터 준비 완료** 기반으로 트리거
- 이전 실행 결과(데이터셋)가 없으면 아예 실행 안 함 → Single Runner보다 더 안전한 순서 보장
- 체감 이점: 이전 job이 늦게 끝나도 다음 job이 무작정 대기하는 게 아니라 데이터 준비되면 자동 시작



**3. Google Cloud Composer + Workflows**
- 이전 한계: 멀티 클라우드 환경에서 Airflow 단독으로 동시 실행 제어하면 클라우드별 리소스 제한과 충돌
- 요즘 쓰는 이유: Google Cloud Workflows가 `concurrency` 블록을 내장 지원 → 스텝 단위로 동시 실행 수 제어 가능
- 체감 이점: GCP 환경에서는 Airflow 없이 Workflows만으로 Single Runner 패턴 구현 가능



**4. AWS Step Functions — `maxConcurrency=1`**
- 이전 한계: Lambda 기반 파이프라인에서 동시 실행 제어를 코드로 직접 구현해야 했음
- 요즘 쓰는 이유: Step Functions Map 스테이트에서 `maxConcurrency=1` 설정으로 서버리스 환경에서도 Single Runner 패턴 구현
- 체감 이점: 서버 관리 없이 증분 처리 파이프라인 순서 보장 가능



<엔지니어 독백>
> 요즘 팀들이 Airflow 대신 Databricks Workflows나 Step Functions으로 옮겨가는 추세야. 이유가 뭐냐면 Airflow는 서버 자체를 관리해야 하거든 — 버전 업그레이드, 플러그인 호환성, 스케줄러 장애 이런 것들. 근데 Databricks나 Step Functions은 관리 포인트가 없어. 단, 복잡한 DAG 의존성이나 세밀한 태스크 단위 제어가 필요하면 아직은 Airflow가 낫긴 해. Single Runner 패턴 자체는 어느 도구든 설정 한 줄이라 쉬운데, 진짜 중요한 건 "이 파이프라인이 정말 순차 실행이 필요한가"를 설계 단계에서 판단하는 거야. 무조건 `max_active_runs=1` 박으면 백필할 때 고생해.






## 패턴#40 동시실행기 (Concurrent Runner)
### (1)문제상황

**핵심 고통**: 각 실행이 서로 독립적인데 Single Runner처럼 순차 실행하면 처리 속도가 너무 느리다.



**배경 — Single Runner와 뭐가 다른가**
- Single Runner: 이전 실행 결과에 다음 실행이 의존 → 순서 필수
- Concurrent Runner: 각 실행이 완전히 독립적 → 순서 상관없음 → 굳이 기다릴 이유 없음

**배경2 — Parallel Split와 뭐가 다른가  
- 핵심 차이 — 뭘 병렬로 돌리냐(대상)

| -     | Parallel Split           | Concurrent Runner          |
| ----- | ------------------------ | -------------------------- |
| 병렬 단위 | **태스크** (하나의 DAG 실행 안에서) | **DAG 인스턴스** (실행 자체를 여러 개) |
| 언제 분기 | 하나의 실행 안에서 분기            | 여러 실행이 동시에 존재              |
| 트리거   | 공통 부모 태스크 완료 후           | 스케줄러가 새 인스턴스 생성            |


**실무 시나리오**
- 데이터 수집팀에서 외부 소스 여러 곳에서 데이터를 내부 DB로 적재하는 파이프라인 운영 중
- 수집 주기: 30분~1시간마다 실행
- 각 실행은 완전히 독립적
- 9시 수집 데이터 ↔ 9시 30분 수집 데이터 → 서로 참조 안 함
- A소스 데이터 ↔ B소스 데이터 → 서로 참조 안 함


**구체적 실패 장면**
- 어느 날 외부 소스에서 데이터가 평소보다 3배 많이 들어옴
- 9시 수집 job: 평소 30분 → 이날 1시간 30분 소요
- Single Runner 설정 상태:
```
[9:00 job]  실행 중 (1시간 30분 소요)
[9:30 job]  대기 중 ← 9:00 job 끝날 때까지 못 시작
[10:00 job] 대기 중
[10:30 job] 대기 중
```
- 결과: 9시 30분에 들어온 데이터가 실제로 적재되는 시각 → 오전 11시 이후
- 다운스트림 대시보드, 분석팀 전부 1~2시간 지연 데이터 보게 됨
- 근데 이 데이터들은 서로 독립적이라 동시에 돌려도 아무 문제 없음



**개선 욕구 → 새로운 문제**
- "어차피 독립적인 실행인데 동시에 돌리면 안 되나?"
- Single Runner는 이런 케이스에서 오히려 불필요한 병목을 만드는 안티패턴
- 동시 실행을 허용하되 리소스 한도는 지켜야 함 → **Concurrent Runner 패턴** 등장
- 이름 의미: 여러 인스턴스를 동시에(Concurrent) 실행(Run)하도록 허용

### (2) 솔루션

**핵심 컨셉**: `max_active_runs` 값을 1보다 크게 설정해서 동시 실행을 허용한다.


**구조**
- Single Runner와 설정 방법은 동일 — `max_active_runs` 값만 다름
- `max_active_runs=5` → 동시에 최대 5개 인스턴스까지 실행 허용
- 5개 초과분은 큐에 대기 → 실행 중인 인스턴스 완료되면 자동 
	- 아마도 리소스 터지는걸 방지하는 범위에서 설정하기?
```
max_active_runs=5 설정 시:

[9:00 job]  실행 중 ┐
[9:30 job]  실행 중 ├ 동시 실행 (최대 5개)
[10:00 job] 실행 중 ┘
[10:30 job] 대기 중 ← 5개 초과 시 대기
```



**주의사항 — 공유 컴포넌트가 있는 경우**
- 각 실행이 독립적이어야 Concurrent Runner 적용 가능
- 공유 상태(shared state)가 있으면 동시 실행 시 충돌 발생
- 예) Dynamic Late Data Integrator 패턴처럼 공유 컴포넌트를 쓰는 파이프라인:
- 동시 실행 시 백필이 여러 번 중복 트리거되거나
- 특정 실행 날짜의 백필이 아예 안 트리거될 수 있음



**`depends_on_past` 설정 병행 가능
- `max_active_runs > 1`이어도 특정 태스크는 순서 보장하고 싶을 때
- `depends_on_past=True`를 태스크 단위로 설정하면 됨
- DAG 전체는 병렬로 돌되, 특정 태스크만 이전 실행 성공 후 시작

```
DAG 전체: max_active_runs=5  → 병렬 허용
특정 태스크: depends_on_past=True → 이 태스크만 순서 보장
```



**Single Runner vs Concurrent Runner 선택 기준**

| 질문                       | 답   | 선택                      |
| ------------------------ | --- | ----------------------- |
| 이전 실행 결과를 다음 실행에서 읽나?    | YES | Single Runner           |
| 각 실행이 완전히 독립적인가?         | YES | Concurrent Runner       |
| 공유 컴포넌트(DB, 파일 등)를 건드리나? | YES | Single Runner 또는 신중히 검토 |
| 백필 속도가 중요한가?             | YES | Concurrent Runner       |


<질문>
> Q. 프레임워크별 기본값은 몇인가?
> A.

|프레임워크|기본값|의미|
|---|---|---|
|Airflow|16|설정 안 하면 동시 16개 허용|
|AWS EMR|1|설정 안 하면 Single Runner|
|ADF|1 (일반 트리거) / 최대 50 (Tumbling Window)|트리거 종류마다 다름|

**1. Apache Airflow
- `max_active_runs_per_dag` 글로벌 기본값: **16**
- DAG 코드에서 `max_active_runs` 미설정 시 → 글로벌 설정값(16) 상속
- 즉 설정 안 하면 동시에 16개 DAG 인스턴스까지 실행 허용

```
# airflow.cfg 글로벌 설정
max_active_runs_per_dag = 16   # 기본값

# DAG 코드에서 미설정 시 → 위 값 상속
# DAG 코드에서 명시 설정 시 → 코드 값이 우선
```


**2. AWS EMR**
- `StepConcurrencyLevel` 기본값: **1** → 기본이 Single Runner 
- 설정 안 하면 순차 실행이 기본



**3. Azure Data Factory**
- Concurrency 미설정 시 기본값: **1** → 직렬 실행
- Tumbling Window 트리거 사용 시 Concurrency 미설정하면 최대 50개까지 동시 실행될 수 있으므로 주의 


### (3)결과(Consequences)

**핵심**: 동시 실행을 허용하는 대신 리소스 고갈과 공유 상태 충돌이라는 새로운 위험이 생긴다.



**위험 1 — 리소스 고갈 (Resource Starvation)**
- 동시 실행 수에 제한을 안 걸면 인스턴스가 무한정 쌓임
- 각 인스턴스가 CPU, 메모리, DB 커넥션을 점유 → 클러스터 전체 성능 저하
- 최악의 경우 다른 파이프라인까지 영향받음
```
max_active_runs=16 (기본값) 상태에서 30일치 백필:

[1일치]  실행 중 ┐
[2일치]  실행 중 │
...              ├ 16개 동시 실행 → 클러스터 CPU 100%
[16일치] 실행 중 ┘
[17일치] 대기 중
```
- 해결: `max_active_runs`를 적절한 값으로 제한 → 리소스 여유분 확보



**위험 2 — 공유 상태 충돌 (Shared State)**
- 각 실행이 독립적이어야 Concurrent Runner 적용 가능
- 공유 컴포넌트(DB 테이블, 파일 경로, 캐시 등)를 건드리면 동시 실행 시 충돌
- 예) Dynamic Late Data Integrator 패턴 (늦게 도착한 데이터를 감지해 해당 날짜 파티션을 자동 백필하는 패턴):
- 동시 실행 A: "백필 필요" 감지 → 백필 트리거
- 동시 실행 B: 동시에 같은 조건 감지 → 백필 중복 트리거
- 결과: 같은 데이터 두 번 처리 → 중복 데이터 적재



**위험 3 — 비결정적 실행 (Nondeterministic Execution)**
- 동시 실행은 실행 순서가 보장되지 않음
- 공유 컴포넌트가 있는 파이프라인에서 실행 순서에 따라 결과가 달라질 수 있음
- 재현하기 어려운 간헐적 버그로 이어짐



**트레이드오프 결론**

|상황|선택|
|---|---|
|각 실행이 완전히 독립적|Concurrent Runner 적합|
|공유 DB 테이블, 파일 경로 있음|Single Runner 또는 신중히 검토|
|백필 속도가 중요|Concurrent Runner + 적절한 `max_active_runs` 값|
|리소스가 제한적|`max_active_runs` 낮게 설정|



<엔지니어 독백>
> `max_active_runs` 기본값 16인 거 모르고 그냥 쓰다가 백필 때 클러스터 터지는 경험 한 번씩은 해봤을 거야. 30일치 백필 돌리면 16개가 동시에 뜨면서 다른 파이프라인까지 느려지거든. 그래서 나는 파이프라인 배포할 때 항상 `max_active_runs`를 명시적으로 설정해. 독립적인 파이프라인이라도 무한정 허용하는 건 위험해. 리소스 여유분 보고 3~5 정도로 잡는 게 실무에서 안전해.



### (4)예시

<엔지니어 독백>
> Concurrent Runner 설정은 단순한데 함정이 있어. `max_active_runs` 높게 잡으면 되는 거 아니냐고 생각하는데, 파이프라인 안에 공유 컴포넌트가 섞여 있으면 그 태스크만 순차로 묶어야 해. 전체 DAG을 병렬로 돌리면서 일부 태스크만 순서 보장하는 게 핵심이야.



**예시 1 — Apache Airflow: 기본 Concurrent Runner (Example 6-22)**
- 디바이스 로그를 외부 소스에서 수집해서 내부 DB로 적재하는 파이프라인
- 각 실행이 독립적이라 동시 실행 허용.

```python
with DAG('devices_loader',
    max_active_runs=5,           # 동시에 최대 5개 인스턴스 실행 허용
    default_args={
        'depends_on_past': False, # 태스크 단위 순서 의존성 없음
    }
):
```

- `max_active_runs=5`: 동시에 5개 DAG 인스턴스까지 실행 허용
- **핵심 라인**: `depends_on_past=False`
- 이전 실행 성공 여부와 무관하게 다음 실행 즉시 시작
- 각 실행이 독립적이기 때문에 가능
- ADF는 `Concurrency` 설정으로 동일하게 구현 가능하지만, 태스크 단위 `depends_on_past` 같은 세밀한 제어는 지원하지 않음

---

**예시 2 — DAG 전체 병렬 + 특정 태스크만 순차 (Example 3-20)**
- Dynamic Late Data Integrator 패턴 (늦게 도착한 데이터를 감지해 해당 날짜 파티션을 자동 백필하는 패턴)처럼 공유 컴포넌트가 일부 있는 파이프라인
- DAG 전체는 병렬로 돌리되 공유 컴포넌트를 건드리는 태스크만 순차 실행.
```
with DAG('devices_loader',
    max_active_runs=5,                      # DAG 전체: 동시 5개 허용
    default_args={'depends_on_past': False}, # 기본: 순서 의존성 없음
) as dag:

    processing_marker = SparkKubernetesOperator(
        task_id='mark_partition_as_being_processed',
        depends_on_past=True   # 이 태스크만 순차 실행 강제#########
    )

    backfill_creation_job = SparkKubernetesOperator(
        task_id='get_late_partitions_and_mark_them_as_being_processed',
        depends_on_past=True   # 이 태스크만 순차 실행 강제###########
    )
```
- `default_args={'depends_on_past': False}`: DAG 전체 기본값은 병렬 허용
- **핵심 라인**: 특정 태스크에만 `depends_on_past=True` 개별 설정
- `processing_marker`: 파티션 처리 중 마킹 → 공유 상태 건드림 → 순차 필수
- `backfill_creation_job`: 백필 태스크 생성 → 중복 트리거 방지 → 순차 필수
- 나머지 태스크는 병렬로 실행됨


**두 예시 비교**
```
예시 1 — 완전 독립적 파이프라인
DAG 전체: 병렬 (max_active_runs=5)
모든 태스크: 병렬 (depends_on_past=False)

예시 2 — 공유 컴포넌트 일부 있는 파이프라인
DAG 전체: 병렬 (max_active_runs=5)
일반 태스크: 병렬 (depends_on_past=False)
공유 태스크: 순차 (depends_on_past=True) ← 태스크 단위 개별 설정
```

**예시3 ㅡ ADF 주의사항**
- ADF는 `depends_on_past` 같은 태스크 단위 의존성 설정 미지원
- 트리거 기반 의존성만 설정 가능
- Concurrent Runner 구현 시 `Concurrency` 값만 조정 가능
- 태스크 단위 세밀한 제어가 필요하면 Airflow 선택


<엔지니어 독백>
> 예시 2가 실무에서 훨씬 자주 보이는 패턴이야. 
> 파이프라인이 완전히 독립적인 경우는 생각보다 드물거든. 
> 대부분은 일부 태스크가 공유 DB나 파일을 건드려. 
> 
> 그럴 때 DAG 전체를 Single Runner로 잡으면 너무 느리고, 
> 전체를 Concurrent Runner로 풀면 충돌 나.
>  그래서 `default_args`로 전체는 병렬로 열어두고, 
>  공유 컴포넌트 건드리는 태스크만 `depends_on_past=True` 개별로 걸어주는 게 정석이야. 
>  처음에 이 패턴 몰라서 전체 DAG을 Single Runner로 잡았다가 백필 때 엄청 느렸던 경험이 있어.




### (5) 최신트렌드

**핵심**: 요즘은 동시 실행 수를 고정값으로 박지 않고, 리소스 상태에 따라 동적으로 제어하는 방향으로 가고 있다.


**1. Airflow 2.x — Dynamic Task Mapping**
- 이전 한계: `max_active_runs=5`처럼 동시 실행 수를 코드에 하드코딩해야 했음
- 데이터가 적은 날도 5개, 많은 날도 5개 → 비효율
- 요즘 쓰는 이유: Dynamic Task Mapping으로 입력 데이터 크기에 따라 실행 수를 런타임에 동적으로 결정
- 데이터 많은 날 → 자동으로 더 많은 태스크 생성
- 데이터 적은 날 → 자동으로 더 적게 생성
- 체감 이점: 리소스 낭비 없이 처리량 최적화 가능


**2. Databricks Workflows — Job Cluster 자동 스케일링**
- 이전 한계: 동시 실행 수 늘리면 클러스터 리소스가 고정돼 있어서 리소스 고갈 위험
- 요즘 쓰는 이유: Databricks Job Cluster가 동시 실행 수에 맞춰 워커 노드를 자동으로 늘렸다 줄임
- `max_concurrent_runs` 설정 + Auto Scaling 조합
- 동시 실행 5개 뜨면 클러스터가 자동으로 확장 → 리소스 고갈 없음
- 체감 이점: 동시 실행 수 늘려도 리소스 걱정 없이 백필 빠르게 처리 가능



**3. AWS Step Functions — Map State 병렬 처리**
- 이전 한계: Airflow 없이 Lambda 기반 파이프라인에서 동시 실행 수 제어를 코드로 직접 구현해야 했음
- 요즘 쓰는 이유: Step Functions Map State의 `maxConcurrency` 파라미터로 서버리스 환경에서 동시 실행 수 제어
- `maxConcurrency=0` → 무제한 병렬
- `maxConcurrency=5` → 최대 5개 동시 실행
- 체감 이점: 서버 관리 없이 Concurrent Runner 패턴 구현, 비용도 실행 수만큼만 과금



**4. Kubernetes 기반 Airflow (KubernetesPodOperator)**
- 이전 한계: 동시 실행 수 늘리면 단일 서버 리소스 한계에 금방 부딪힘
- 요즘 쓰는 이유: 각 DAG 인스턴스를 독립된 Pod로 실행 → 클러스터 노드 수만큼 수평 확장 가능
- `max_active_runs=20`으로 설정해도 Pod가 각각 독립 실행
- 노드 부족하면 Kubernetes가 자동으로 노드 추가
- 체감 이점: 리소스 고갈 걱정 없이 Concurrent Runner 공격적으로 활용 가능


< 엔지니어 독백>

> 예전엔 `max_active_runs` 값 정하는 게 감으로 하는 거였어. 
> 클러스터 CPU 보고 "3이면 되겠지" 이런 식으로. 
> 
> 근데 요즘은 Databricks Auto Scaling이나 Kubernetes 조합으로 가면 
> 동시 실행 수 제한 자체가 의미가 없어지는 경우도 있어. 
> 클러스터가 알아서 늘어나니까. 
> 
> 단, 무조건 병렬로 때리면 다운스트림 DB가 커넥션 한도 초과로 터지는 경우가 있거든. 
> 클러스터 리소스만 보지 말고 싱크(Sink) 쪽 DB 커넥션 한도도 같이 봐야 해. 
> 그게 진짜 병목이 되는 경우가 더 많아.



