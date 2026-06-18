# 데이터 엔지니어링 디자인 패턴 - 데이터 흐름 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 6 | 실무 데이터 엔지니어링 관점 정리

> **도식 기호 범례** — `⚠` 위험·함정(조심해야 할 동작) · `✗` 잘못된 결과(깨진 상태) · `✓` 올바른 결과 · `⇒` 그 결과 도출

---

## 목차

1. [시퀀스 (Sequence)](#1-시퀀스-sequence)
   - 패턴 #33: 로컬 시퀀서 (Local Sequencer)

> 본 문서가 다루는 범위는 챕터 6 중 **#33** (6.1 Sequence 의 Local Sequencer).
> 나머지(#34 Isolated Sequencer, 6.2 Fan-In, 6.3 Fan-Out, 6.4 Orchestration)는 후속 문서에서 다룸.

---

## 책의 use case (챕터 도입)

> raw 데이터에서 비즈니스 가치를 만들면 **사실 기반(fact-based) 의사결정** 이 가능해짐.
> 챕터 5의 데이터 가치 패턴이 그 길을 열어줌. 하지만 그렇게 만든 인사이트가 **나(내 팀) 안에만(local) 머물면** 아쉬움.

- 가치 있는 데이터셋을 **조직 내 다른 팀에 노출** 하면, 그들이 자기 use case 를 강화하며 데이터 가치 자산이 더 커짐.
  반대로 다른 팀의 데이터셋을 받아 **내 데이터 가치** 를 키울 수도 있음.
- 데이터 가치 패밀리처럼 보이지만 **적용 규칙이 달라서**, 이를 **데이터 흐름(Data Flow) 패턴** 으로 따로 다룸.
- 목표 — 데이터셋 생성에 필요한 **모든 단계를 설계·조율**.
  태스크를 파이프라인으로 chaining, 병렬·배타 분기 생성, 물리적으로 분리된 파이프라인의 의존성 관리 등.

```
[챕터 6 데이터 흐름 패턴 — 두 레벨에서 동작]
──────────────────────────────────────────────────────────────────────
 ① Data orchestration   파이프라인 안/여러 파이프라인을 조율 — cross-team 협업에 유용
 ② Data processing      잡 내부 환경 — 비즈니스 로직을 더 명확·유지보수 쉽게 조직화
──────────────────────────────────────────────────────────────────────
 6.1 Sequence    단계의 순서 조율        #33 Local / #34 Isolated Sequencer
 6.2 Fan-In      여러 입력의 합류         #35 Aligned / #36 Unaligned Fan-In
 6.3 Fan-Out     하나에서 여러 분기       #37 Parallel Split / #38 Exclusive Choice
 6.4 Orchestration 파이프라인 동시성 관리   #39 Single / #40 Concurrent Runner
──────────────────────────────────────────────────────────────────────
 본 문서: 6.1 Sequence 의 Local Sequencer (#33)
```

### 패턴 흐름 — 챕터 5에서 챕터 6 #33 으로

가치는 만들었는데(챕터 5) **어떻게 연결·공유하지?** 가 데이터 흐름 패턴의 출발점이고, 그 첫걸음이 **순서(Sequence)** 임.

```
[패턴 흐름 — 챕터 5에서 챕터 6 #33 으로]
──────────────────────────────────────────────────────────────────────
 챕터 5: raw → 데이터 가치(보강·집계·세션·정렬) 생성
      │ 한계: 그 가치가 "내 팀 안(local)" 에만 머묾
      ▼ "다른 팀과 주고받아 더 넓게 쓰자"
 챕터 6: 데이터 흐름 (파이프라인 설계·조율)
      │ 첫 관문: 단계의 "순서(sequence)"
      ▼ "오래된 거대 잡이 매번 처음부터 재실행돼 디버깅이 고통"
 6.1 Sequence
   #33 Local Sequencer — 큰 잡을 순서 있는 작은 태스크로 분해 (같은 파이프라인 안)
      │ (다음) 물리적으로 분리된 파이프라인을 잇는다면?
      ▼
   #34 Isolated Sequencer (후속 문서)
──────────────────────────────────────────────────────────────────────
 핵심 — 데이터 흐름 패턴은 "만든 가치를 어떻게 잇고 공유하나" 를 다루고, 그 첫걸음이 순서(Sequence).
```

---

## 1. 시퀀스 (Sequence)

데이터 흐름을 설계할 때 가장 먼저 만나는 카테고리가 **단계의 순서(sequence)** 임.
이 순서가 파이프라인의 **복잡도·성능·유지보수** 에 두루 영향을 줌.

> 예 — 처리한 데이터셋을 **여러 곳에 쓰는** 잡을 하나의 단일 단위로 워크플로에 두면,
> 한 DB 의 적재(loading)만 다시 돌리고 싶어도 **전체 실행을 재시작** 해야 함. 시퀀스 패턴이 이 문제를 풀어줌.

- **#33 Local Sequencer** — **한 파이프라인/잡 안에서** 태스크를 순차로 조율할 때.
- (#34 Isolated Sequencer — **물리적으로 분리된 여러 파이프라인** 을 잇는 경우. 본 문서 범위 밖.)

---

### 1-1. 패턴 #33: 로컬 시퀀서 (Local Sequencer)

> 이 절의 첫 패턴이자 가장 쉬운 패턴 — **같은 파이프라인(또는 데이터 처리 잡) 안에서** 태스크를 로컬로 조율함.

#### 상황 (Problem)

**책의 use case** — 너무 커져 버린 오래된 분석 잡:

- 데이터 분석 부서에서 **가장 오래된 잡** 하나를 맡음.
- 세월이 지나며 코드가 수십 줄 → **수백 줄** 로 불었고, transformation 개수도 **3배** 로 늘어남.
- 잡이 **자주 실패** 하는데, 매번 **처음부터 다시** 시작해야 해서 디버깅이 길고 고통스러움.
- **결정적 제약**: 코드를 단순화하고 유지보수를 개선해야 하지만, **비즈니스 로직은 제거할 수 없음**.

#### 해결 (Solution)

복잡한 로직을 단순화하는 좋은 방법은 **작게 분해(decompose)** 하는 것 —
데이터 엔지니어링에서도 분해가 **가독성** 과 **관심사 분리(separation of concerns)** 를 높여줌. → **Local Sequencer**.

- 큰 컴포넌트 하나를 **연결된 작은 item 여러 개로 쪼개 순차 실행** 함.
  태스크 간 의존성은 **데이터셋 의존성** 에 맞춰 정함 — 예: task B 가 task A 의 데이터를 필요로 하면 **B 는 A 다음**.
- 챕터 2의 full 데이터 수집 예 — **Readiness Marker**(데이터 존재 확인) + **Full Loader**(물리 적재) 두 태스크로 구성.
  이를 ① **data orchestration layer** 의 두 의존 태스크로 두거나, ② **data processing layer** 의 단일 태스크로 합칠 수 있음(Figure 6-1).

```
[Figure 6-1 재현] Local Sequencer — orchestration 두 태스크 vs 단일 태스크
──────────────────────────────────────────────────────────────────────
 ① data orchestration layer (두 의존 태스크)
    ┌──────────────────┐      ┌─────────────┐
    │ Readiness Marker │ ───► │ Full Loader │
    └──────────────────┘      └─────────────┘

 ② data processing layer (단일 태스크)
    ┌────────────────────────────────┐
    │ Readiness Marker + Full Loader │
    └────────────────────────────────┘
──────────────────────────────────────────────────────────────────────
 같은 시퀀스를 orchestration 의 두 태스크로 쪼갤지, processing 의 한 태스크로 합칠지 선택.
```

다음 **세 기준** 으로 orchestration layer vs processing layer 를 정함.

- **Separation of concerns (관심사 분리)** — 한 item 에 모든 연산을 몰면 이해가 어려워짐.
  좋은 지표가 **이름 짓기** — 태스크 이름이 잘 안 떠오르거나 너무 길어지면, 한 태스크에 연산을 너무 많이 넣었다는 신호.
- **Maintainability (유지보수)** — 데이터 처리 순차성에 기대면 backfill·retry 때 **성공한 태스크까지 다시** 계산하게 됨.
  예: readiness check 가 **유료 API 10개** 호출인데 3번 실패한 잡을 재시작하면, 그 API 호출까지 매번 반복돼 비용이 급증.
- **Implementation effort (구현 부담)** — orchestrator 는 SQL 쿼리·API 호출 같은 공통 태스크의 **추상화를 기본 제공** 함.
  모든 걸 한 단위로 합치면 그 편의 기능을 못 써서 **바퀴를 재발명** 하게 됨.

**쉽게 풀어 보면** — 거대한 잡 하나를 **순서 있는 작은 태스크들로 쪼개는 것** 이 전부.
가장 큰 이득은 **실패 시 재시작 범위** — monolith 면 일부만 실패해도 전부 다시 돌지만, 쪼개 두면 **실패한 태스크만** 다시 시작함.

```
[쉽게 풀어 보면 — 왜 쪼개나: 실패 시 재시작 범위가 다르다]
──────────────────────────────────────────────────────────────────────
 ⚠ 단일 거대 잡 (monolith)
   [ readiness + 무거운 집계 + 적재 ]   ← 적재에서 실패
   ⇒ 재시도하면 readiness·집계까지 처음부터 다시 (비싼 단계 반복)

 ✓ 순서 있는 작은 태스크 (Local Sequencer)
   [readiness] ─► [집계] ─► [적재]      ← 적재에서 실패
   ⇒ 성공한 readiness·집계는 그대로, 적재만 재시작
──────────────────────────────────────────────────────────────────────
 경계(boundary)는 "어디서부터 다시 시작해야 하나(restart boundary)" 로 정함.
 ⚠ 단, 반드시 한 단위여야 하는 연산(트랜잭션)은 묶어 둘 것.
```

#### 고려사항 (Consequences)

앞의 팁이 도움은 되지만 첫날부터 완벽한 구성을 보장하진 못함. **경계(boundary)를 정의** 하는 일이 핵심 난제.

- **Boundaries (경계)**
  - 경계를 잘못 잡으면 실행 시간이 너무 길어지거나, 스케줄러를 확장 못 할 경우 **다른 파이프라인까지 영향** 을 줌.
    scope 와 태스크 수 사이의 균형이 필요.
  - rule of thumb 은 **재시작 경계(restart boundaries)** 로 생각하는 것 — "어떤 태스크가 개별적으로 재시작돼야 하나?"
    예: Readiness Marker 와 Full Loader 는 **개별 실패** 해야 하고 서로 영향을 주면 안 됨.
    적재가 실패했다고 Readiness Marker 를 다시 돌릴 필요는 없음 — 적재 실행 자체가 **데이터가 이미 있었음** 을 뜻하기 때문.
    (단, source 가 immutable 이라는 가정. producer 가 데이터를 지울 수 있으면 Readiness Marker 도 다시 돌려야 함.)
  - 데이터 처리 잡에서는 경계를 **가장 compute-비싼 연산 사이** 에 둠.
    예: 최종 변환 전 **중간 데이터셋 생성** 이 비싸면 거기에 경계를 둬서, 최종 변환이 실패해도 그 비싼 중간 생성을 매번 다시 안 함.
  - **트랜잭션(transactions) 관점** 의 분리도 있음 — 두 연산이 **반드시 한 단위** 여야 하면 묶어 둠.
    예: 챕터 3의 **Dynamic Late Data Integrator** 는 데이터 처리와 **state 테이블의 last processed version 갱신** 이 서로 의존하므로 단일 단위로 실행.

#### 구현 예시 (Examples)

**예시 1 — Apache Airflow (Example 6-1)**

`>>` 로 의존성을 표현(왼쪽이 먼저). "데이터를 기다려 → 내부 테이블에 적재 → 사용자에게 노출" 흐름:
```python
input_data_sensor >> load_data_to_table >> expose_new_table
```

**예시 2 — AWS EMR Step API (Example 6-2)**

두 스텝을 순차 실행하되 `ActionOnFailure=TERMINATE_CLUSTER` 로 실패 시 클러스터 종료 —
부분 데이터 노출이나 "성공한 척" 을 막음:
```bash
aws emr add-steps --cluster-id j=cluster_id --steps Type=Spark,Name="Spark Program",\
  ActionOnFailure=TERMINATE_CLUSTER,Args=[--class com.waitingforcode.DataLoader]
aws emr add-steps --cluster-id j=cluster_id --steps Type=Spark,Name="Spark Program",\
  ActionOnFailure=TERMINATE_CLUSTER,Args=[--class com.waitingforcode.DataPublisher]
```

> Airflow 는 기본적으로 **upstream 성공** 기반 순차 실행이라(부모가 성공해야 자식 진행) 이런 설정이 필요 없음.

**예시 3 — PySpark 데이터 처리 레벨 (Example 6-3)**

명시적 chaining 기호가 없어도, **앞 단계 변수를 다음 단계 입력으로** 쓰는 것만으로 순서가 자연스럽게 표현됨:
```python
input_dataset: DataFrame = spark_session.read...
valid_and_enriched_dataset_to_write: DataFrame = input_dataset...   # 앞 결과를 입력으로
valid_and_enriched_dataset_to_write.write...                         # 마지막에 기록
```

> **참고 사항 — CRON 그 이상**
> Local Sequencer 는 단순 CRON 식 대비 **data orchestrator 의 힘** 을 보여줌(고급 처리 로직 구성).
> 단, **의존성이 전혀 없는 격리된 use case** 라면 CRON 식도 여전히 유효한 선택.

| 레벨 | 무엇으로 순서를 표현 | 적합한 상황 |
|---|---|---|
| Data orchestration | Airflow `>>`, EMR Step API 등 | 태스크별 개별 재시작·공통 태스크 추상화가 필요할 때 |
| Data processing | 변수 chaining(앞 결과 → 다음 입력) | 한 잡 안에서 변환을 순서대로 엮을 때 |

> **트러블 로그** — 모든 단계를 한 잡에 **단일 단위** 로 욱여넣으면, 일부만 실패해도 **비싼 단계까지 통째로** 재실행됨.
> 예: readiness check(유료 API 10회) + 무거운 집계 + 적재를 한 PySpark 잡에 넣었는데 적재에서 실패하면,
> 재시도마다 유료 API 10회 호출과 집계가 **처음부터** 다시 돌아 비용·시간이 폭증.
> **재시작 경계** 를 기준으로 Readiness Marker / 집계 / 적재를 orchestration 태스크로 쪼개,
> 실패한 단계만 다시 시작하도록 할 것.
