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
