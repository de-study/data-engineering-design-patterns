# 데이터 엔지니어링 디자인 패턴 - 데이터 흐름 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 6 | 실무 데이터 엔지니어링 관점 정리

> **도식 기호 범례** — `⚠` 위험·함정(조심해야 할 동작) · `✗` 잘못된 결과(깨진 상태) · `✓` 올바른 결과 · `⇒` 그 결과 도출

---

## 목차

1. [시퀀스 (Sequence)](#1-시퀀스-sequence)
   - 패턴 #33: 로컬 시퀀서 (Local Sequencer)
   - 패턴 #34: 독립 시퀀서 (Isolated Sequencer)
2. [팬인 (Fan-In)](#2-팬인-fan-in)
   - 패턴 #35: 정렬된 팬인 (Aligned Fan-In)
   - 패턴 #36: 비정렬 팬인 (Unaligned Fan-In)
3. [팬아웃 (Fan-Out)](#3-팬아웃-fan-out)
   - 패턴 #37: 병렬 분할 (Parallel Split)
   - 패턴 #38: 배타적 선택 (Exclusive Choice)
4. [오케스트레이션 (Orchestration)](#4-오케스트레이션-orchestration)
   - 패턴 #39: 단일 실행기 (Single Runner)
   - 패턴 #40: 동시 실행기 (Concurrent Runner)
5. [요약](#5-요약)

> 본 문서는 챕터 6 **전체(#33 ~ #40)** 를 다룸 — 6.1 Sequence · 6.2 Fan-In · 6.3 Fan-Out · 6.4 Orchestration.

---

## 책의 use case (챕터 도입)

> raw 데이터에서 비즈니스 가치를 만들면 **사실 기반(fact-based) 의사결정** 이 가능해짐.
> 챕터 5의 데이터 가치 패턴이 그 길을 열어줌. 하지만 그렇게 만든 인사이트가 **나(내 팀) 안에만(local) 머물면** 아쉬움.

- **양방향 가치 교환** — 가치 있는 데이터셋을 **조직 내 다른 팀에 노출** 하면
  그들이 자기 use case 를 강화해 데이터 가치 자산이 더 커짐.
  반대로 다른 팀의 데이터셋을 받아 **내 데이터 가치** 를 키울 수도 있음.
- **별도 패밀리인 이유** — 데이터 가치 패밀리처럼 보이지만 **적용 규칙이 달라서**,
  이를 **데이터 흐름(Data Flow) 패턴** 으로 따로 다룸.
- **목표** — 데이터셋 생성에 필요한 **모든 단계를 설계·조율**. 구체적으로:
  - 태스크를 파이프라인으로 chaining
  - 병렬·배타 분기 생성
  - 물리적으로 분리된 파이프라인의 의존성 관리

```
[챕터 6 데이터 흐름 패턴 — 두 레벨에서 동작]
──────────────────────────────────────────────────────────────────────
 ① Data orchestration   파이프라인 안/여러 파이프라인을 조율 — cross-team 협업에 유용
 ② Data processing      잡 내부 환경 — 비즈니스 로직을 더 명확·유지보수 쉽게 조직화
──────────────────────────────────────────────────────────────────────
 6.1 Sequence    단계의 순서 조율         #33 Local / #34 Isolated Sequencer
 6.2 Fan-In      여러 입력의 합류         #35 Aligned / #36 Unaligned Fan-In
 6.3 Fan-Out     하나에서 여러 분기       #37 Parallel Split / #38 Exclusive Choice
 6.4 Orchestration 파이프라인 동시성 관리   #39 Single / #40 Concurrent Runner
──────────────────────────────────────────────────────────────────────
 본 문서: 챕터 6 전체 — 6.1 Sequence(#33·#34) · 6.2 Fan-In(#35·#36) · 6.3 Fan-Out(#37·#38) · 6.4 Orchestration(#39·#40)
```

### 패턴 흐름 — 챕터 5에서 챕터 6 으로

가치는 만들었는데(챕터 5) **어떻게 연결·공유하지?** 가 데이터 흐름 패턴의 출발점이고, 그 첫걸음이 **순서(Sequence)** 임.

```
[패턴 흐름 — 챕터 5에서 챕터 6 으로]
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
   #34 Isolated Sequencer — 팀·조직 경계로 분리된 파이프라인을 잇기 (프로바이더 ↔ 컨슈머)
      │ (다음) 여러 입력이 한 지점으로 합류한다면? 하나에서 여러 갈래로 나뉜다면?
      ▼
 6.2 Fan-In   #35 Aligned / #36 Unaligned — 여러 부모 → 한 자식 (합류)
 6.3 Fan-Out  #37 Parallel Split / #38 Exclusive Choice — 한 부모 → 여러 자식 (병렬 전부 / 택일)
      │ (다음) 이 파이프라인들을 "몇 개나 동시에" 돌릴까?
      ▼
 6.4 Orchestration  #39 Single / #40 Concurrent Runner — 동시성 1(순서 지킴) / N(처리량 올림)
──────────────────────────────────────────────────────────────────────
 핵심 — 데이터 흐름 패턴은 "만든 가치를 어떻게 잇고·합치고·나누고·돌리나" 를 다루고, 그 첫걸음이 순서(Sequence).
```

---

## 1. 시퀀스 (Sequence)

데이터 흐름을 설계할 때 가장 먼저 만나는 카테고리가 **단계의 순서(sequence)** 임.
이 순서가 파이프라인의 **복잡도·성능·유지보수** 에 두루 영향을 줌.

> 예 — 처리한 데이터셋을 **여러 곳에 쓰는** 잡을 하나의 단일 단위로 워크플로에 두면,
> 한 DB 의 적재(loading)만 다시 돌리고 싶어도 **전체 실행을 재시작** 해야 함. 시퀀스 패턴이 이 문제를 풀어줌.

- **#33 Local Sequencer** — **한 파이프라인/잡 안에서** 태스크를 순차로 조율할 때.
- **#34 Isolated Sequencer** — **물리적으로 분리된 여러 파이프라인** 을 잇는 경우 (아래 1-2 절).

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

복잡한 로직을 단순화하는 좋은 방법은 **작게 분해(decompose)** 하는 것 
— 데이터 엔지니어링에서도 분해가 **가독성** 과 **관심사 분리(separation of concerns)** 를 높여줌. → **Local Sequencer**.

- **분해 방식** — 큰 컴포넌트 하나를 **연결된 작은 item 여러 개로 쪼개 순차 실행** 함.
  태스크 간 의존성은 **데이터셋 의존성** 에 맞춰 정함 — 예: task B 가 task A 의 데이터를 필요로 하면 **B 는 A 다음**.
- **적용 예 (챕터 2 full 데이터 수집)** — **Readiness Marker**(데이터 존재 확인) + **Full Loader**(물리 적재) 두 태스크로 구성.
  이를 두 방식 중 하나로 배치(Figure 6-1)
  - ① **data orchestration layer** 의 **두 의존 태스크** 로 분리
  - ② **data processing layer** 의 **단일 태스크** 로 통합

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
  - 경험 법칙 (rule of thumb) 은 **재시작 경계(restart boundaries)** 로 생각하는 것 — "어떤 태스크가 개별적으로 재시작돼야 하나?"
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

두 스텝을 순차 실행하되 `ActionOnFailure=TERMINATE_CLUSTER` 로 실패 시 클러스터 종료
— 부분 데이터 노출이나 "성공한 척" 을 막음:
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

**CRON 이 아는 것은 오직 "언제 시작하느냐(시각)" 하나뿐.**
잡이 내부에서 무엇을 하는지, 성공했는지 실패했는지, 앞 단계가 끝났는지는 전혀 관여하지 않음 
— 시간이 되면 명령을 던지고 끝.
그래서 CRON 으로 여러 단계를 엮으려면 **순서를 시각으로 추측** 하게 됨.

```
[CRON 으로 다단계를 엮을 때 — 시각으로 순서를 "추측"하는 안티패턴]
──────────────────────────────────────────────────────────────────────
 0  2 * * *  readiness_check.sh    # 2시:    데이터 준비됐길 "바라며" 체크
 30 2 * * *  load_data.sh          # 2시 30분: 30분이면 끝났겠지...
 0  3 * * *  expose_table.sh       # 3시:    적재 끝났겠지...
──────────────────────────────────────────────────────────────────────
 ⚠ 순서를 시각으로 추측 — readiness 가 35분 걸리면 load_data 가 ✗ 준비 안 된 데이터를 적재
 ⚠ 실패를 모름   — readiness 가 실패해도 2시 30분이 되면 load_data 는 ✗ 그냥 실행됨
 ⚠ 상태가 없음   — 3번 실패한 잡을 재시작하면 ✗ readiness 부터 처음부터 다시
```

**data orchestrator 가 "그 이상"인 지점** — CRON 의 위 한계를 다음 4가지로 메움.

- **의존성으로 순서 보장** — 시각 추측이 아니라 `>>`(왼쪽 성공 → 오른쪽 시작)로 순서를 정함.
  readiness 가 5분이 걸리든 35분이 걸리든 **끝난 그 순간** 다음 태스크가 출발 → "30분이면 되겠지" 추측 불필요.
- **실패 전파(failure propagation)** — upstream 실패를 앎. readiness 가 실패하면 load_data 는 **아예 실행 안 됨**.
  예시 2의 `ActionOnFailure=TERMINATE_CLUSTER` 처럼 "성공한 척" 하며 부분 데이터를 노출하는 사고를 차단.
- **태스크별 재시작(restart boundary)** — "readiness·집계 성공, 적재만 실패" 를 기억 → **실패한 적재만** 다시 돌림.
  유료 API 10회 readiness 를 매번 반복하는 비용 폭증(아래 트러블 로그)을 방지.
- **공통 태스크 추상화** — 센서(데이터 도착 대기)·SQL 실행·API 호출을 **기본 컴포넌트로 제공**.
  CRON 이면 폴링 스크립트까지 직접 짜야 할 것을, orchestrator 는 `input_data_sensor` 처럼 가져다 씀.

| 비교 축 | CRON (단순 시간 트리거) | Local Sequencer (data orchestrator) |
|---|---|---|
| 순서 보장 | 시각으로 **추측** | 의존성(`>>`)으로 **보장** |
| 실패 처리 | 모름 — 그냥 다음 실행 | 실패 전파 — 뒤를 **차단** |
| 재시작 | 상태 없음 → **전부 다시** | 태스크별 → **실패분만** |
| 공통 연산 | 직접 구현 | 추상화 **기본 제공** |

즉 "CRON 그 이상" 은 **단순 시간 트리거** 에서 
**데이터 의존성·실패·상태를 이해하는 처리 로직 구성(고급 orchestration)** 으로 올라간다는 뜻.

**그런데 왜 "CRON 도 여전히 유효" 인가** — 위 4가지 이점은 전부 **태스크가 여럿이고 서로 의존할 때** 빛남.
잡이 단 하나, 앞뒤로 엮인 단계도 없고, 실패하면 통째로 다시 돌리면 그만인 **격리된 use case** 라면
orchestrator 의 의존성·재시작·실패전파가 **쓸 일이 없음** → 무거운 도구를 띄우는 건 운영 부담만 늘림.

```
[의존성 0개짜리 격리된 단일 잡 — CRON 으로 충분]
 0 4 * * *  /run/standalone_report.sh   # 다른 무엇에도 의존 안 함, 실패하면 통째로 재실행
```

| 레벨 | 무엇으로 순서를 표현 | 적합한 상황 |
|---|---|---|
| Data orchestration | Airflow `>>`, EMR Step API 등 | 태스크별 개별 재시작·공통 태스크 추상화가 필요할 때 |
| Data processing | 변수 chaining(앞 결과 → 다음 입력) | 한 잡 안에서 변환을 순서대로 엮을 때 |

> **트러블 로그** — 모든 단계를 한 잡에 **단일 단위** 로 욱여넣으면, 일부만 실패해도 **비싼 단계까지 통째로** 재실행됨.
>
> **예 —** readiness check(유료 API 10회) + 무거운 집계 + 적재를 한 PySpark 잡에 넣었는데 적재에서 실패하면,
> 재시도마다 유료 API 10회 호출과 집계가 **처음부터** 다시 돌아 비용·시간이 폭증.
>
> **권장 —** **재시작 경계** 를 기준으로 Readiness Marker / 집계 / 적재를 orchestration 태스크로 쪼개, 실패한 단계만 다시 시작하도록 할 것.
>
> **반대 함정 —** orchestrator 의 이점은 전부 "의존성" 에서 나옴.
> 의존성 0개짜리 단일 배치(예: 매일 한 번 도는 독립 리포트)까지 Airflow DAG 로 감싸면
> scheduler·webserver·metadata DB 유지보수에 정작 잡 로직보다 더 많은 시간을 쓰게 됨.
> → 태스크가 **2개 이상이고 서로 데이터로 엮일 때만** orchestrator 를 도입하고, 격리된 단일 잡은 CRON 으로 둘 것.

---

### 1-2. 패턴 #34: 독립 시퀀서 (Isolated Sequencer)

> Local Sequencer 로 만든 파이프라인이 **최종 단계가 아닌** 경우가 많음 — 여러 격리된 워크플로가 협업해 최종 인사이트를 만듦.
> 규칙은 비슷하게 들려도 **트리거 조건과 제약이 완전히 다름**.

#### 상황 (Problem)

**책의 use case** — 데이터 시각화 팀에 정제·보강 데이터셋을 넘기는 경우:

- 우리 팀은 raw 데이터셋을 **정제·보강** 해 시각화 도구가 쓰는 여러 뷰로 노출함.
- 시각화 팀과의 기술 미팅 결과, **대시보드용 데이터셋 변환** 을 우리 준비 파이프라인에 넣지 않기로 합의 
  — 그 부분은 우리 팀 책임이 아니기 때문.
- 시각화 팀은 **정제·보강된 데이터셋만** 받고, 변환은 자기들이 직접 수행하기로 함.
- **결정적 제약**: 두 파이프라인이 데이터를 주고받지만, **조직상 분리** 되어 하나의 프로세스로 합칠 수 없음.

#### 해결 (Solution)

한 파이프라인이 다른 파이프라인에 데이터를 공급하지만 조직 분리 때문에 단일 프로세스로 병합 불가한 상황 → **Isolated Sequencer**.

- **목표** — **물리적으로 격리된 파이프라인을 결합**. Local Sequencer 처럼 핵심 관심사는 **경계(boundary) 식별**.
  가장 쉬운 경계는 **컨슈머 (받는 쪽) / 프로바이더 (주는 쪽)** 또는 팀 단위 
  — 내 팀이 만든 데이터셋을 다른 팀에 넘기면 자연히 두 개의 격리된 파이프라인으로 경계가 그어짐.
- **프로바이더 겸 컨슈머** — 프로바이더이면서 동시에 컨슈머인 경우도 있음(우리 팀 안의 다른 파이프라인이 그 데이터셋으로 또 다른 데이터셋을 만들 때).
  이때는 **복잡도** 로 판단 — 프로바이더+컨슈머 로직을 한 곳에 두면 읽기 힘들어지면 분리, 아니면 합쳐서 **Local Sequencer**.

경계를 정한 다음 단계는 **트리거 메커니즘**. 두 전략 — **데이터 기반(data-based)** / **태스크 기반(task-based)**.

- **데이터 기반** — **Readiness Marker 패턴(챕터 2)** 기반. 프로바이더가 데이터셋 + marker 파일 생성 →
  컨슈머가 marker 파일을 **감지하는 순간** 작업 시작.
- **태스크 기반** — marker 없이, 프로바이더가 컨슈머 파이프라인을 **직접 트리거**.

두 방식의 **결합도·책임** 이 다름.

- **결합도** — 데이터 기반 = **느슨한 결합(loosely coupled)** 으로 진화 여지가 큼(유일한 요구사항은 marker 규약 준수).
  태스크 기반 = **강한 결합(tightly coupled)** — 물리적으로 분리돼 있어도 독립 생존 불가.
  컨슈머 쪽 태스크/파이프라인 **이름만 바꿔도** 프로바이더가 존재하지 않는 태스크를 트리거하려다 **의존 체인 전체가 깨짐**.
- **어느 쪽이 바뀔 수 있나** — 데이터 기반: 컨슈머 자유도 큼, 프로바이더에 알리지 않고 다른 데이터셋으로 갈아탈 수도 있음.
  태스크 기반: 컨슈머가 데이터셋을 임의로 건너뛸 수 없음(프로바이더가 직접 트리거). 데이터셋이 사라지면 프로바이더 파이프라인이 실패할 수 있음.

```
[Figure 6-2 재현] 두 파이프라인 의존성 구현 — 데이터 기반 vs 태스크 기반
──────────────────────────────────────────────────────────────────────
 ① Data-based approach (느슨한 결합 / loosely coupled)
   [Data provider pipeline]                 [Data consumer pipeline]
     Data generation job ─► (DB)   ┈┈감지┈►   Readiness marker ─► Data generation job
   ⇒ 프로바이더는 데이터+marker 만 생성, 컨슈머가 marker 를 보고 스스로 시작

 ② Task-based approach (강한 결합 / tightly coupled)
   [Data provider pipeline]                 [Data consumer pipeline]
     Data generation job ─► Trigger task ──►  Data generation job
            └─<writes>─► (DB)
   ⇒ 프로바이더가 컨슈머 파이프라인을 직접 트리거 (공유 데이터셋이 없어도 됨)
──────────────────────────────────────────────────────────────────────
 데이터 기반 = marker(데이터 존재) 규약만 지키면 됨 · 태스크 기반 = 직접 트리거(이름만 바뀌어도 체인 붕괴)
```

**쉽게 풀어 보면 (주문 데이터 예)** — 두 팀이 주문 데이터를 주고받는 상황에 대입하면 개념이 한 번에 이어짐.

- **등장인물** — **데이터 플랫폼팀**(프로바이더)이 raw 주문 로그를 정제해 `orders_cleaned` 를 매일 07:00 생성,
  **마케팅 분석팀**(컨슈머)이 그걸 받아 `campaign_roi` 집계를 생성.
  두 팀은 **다른 조직** 이라 한 DAG 로 못 합침 → 딱 **Isolated Sequencer** 상황.

```
[주문 데이터로 본 Isolated Sequencer]
──────────────────────────────────────────────────────────────────────
 ① 경계 (팀 단위)
    [플랫폼팀] raw_orders ─► orders_cleaned   ┆   [마케팅팀] orders_cleaned ─► campaign_roi
                        (프로바이더)      └─ 경계 ─┘            (컨슈머)
 ② 프로바이더 겸 컨슈머
    플랫폼팀의 orders_cleaned 를 사내 inventory_forecast(재고 예측)도 입력으로 씀
    ⇒ 복잡도로 판단: 한 잡에 몰아 읽기 힘들면 분리 / 단순하면 합쳐서 Local Sequencer
 ③ 트리거 메커니즘
    데이터 기반: 플랫폼팀 orders_cleaned + _SUCCESS 마커 → 마케팅팀 FileSensor 가 감지하는 순간 시작
    태스크 기반: 플랫폼팀이 마케팅팀 campaign_roi_start 태스크를 직접 트리거
──────────────────────────────────────────────────────────────────────
 핵심 — 상대가 말없이 바꿔도 안 터지는 쪽이 데이터 기반 ⇒ cross-team 의 기본값
```

- **결합도 차이 (핵심)** — 마케팅팀이 협의 없이 태스크 이름 `campaign_roi_start` → `roi_v2` 로 바꾸면:
  - 데이터 기반 ⇒ 플랫폼팀은 `_SUCCESS` 마커만 남기므로 `✓` 무사.
  - 태스크 기반 ⇒ 플랫폼팀이 없는 태스크를 트리거 → `✗` 매일 실패(플랫폼팀이 영문도 모르고 장애 대응).
- **누가 바꿀 수 있나** — 데이터 기반: 마케팅팀이 소스를 `orders_enriched` 로 갈아타도 플랫폼팀 무관.
  태스크 기반: 플랫폼팀이 `orders_cleaned` 를 없애면 트리거 대상이 사라져 `✗` 플랫폼팀 파이프라인이 실패.

> **참고 사항 — 실무에서는 무엇을 주로 쓰나**
> **팀 경계를 넘는 의존은 데이터 기반(느슨한 결합)이 사실상 표준.** 위 결합도 차이 때문
> — 상대 팀이 말없이 바꿔도 안 터짐.
>
> 최근 오케스트레이터도 이 방향
> — Airflow 의 **Dataset(데이터 인지 스케줄링, 2.4+)**, Dagster 의 **software-defined assets(에셋 리니지 기반)** 는
> "데이터가 갱신되면 컨슈머가 뜬다" 는 데이터 기반 모델을 1급 기능으로 제공.
>
> **1급 기능이란** — 데이터셋을 **이름 붙은 객체** 로 API 에서 직접 참조한다는 뜻.
> 예전엔 `_SUCCESS` 마커를 손으로 만들고 `FileSensor` 로 폴링해 구현하던 데이터 기반 결합을,
> 이제 프레임워크가 **내장 구독 방식** 으로 대신 해 줌.
> Airflow 는 프로바이더가 `outlets=[orders]` 로 데이터셋을 "생산" 표시하고, 컨슈머는 `schedule=[orders]` 로 그 데이터셋을 "구독" 하면
> 프로바이더 성공 시 Airflow 가 컨슈머를 자동 실행. (비유: "파일 생겼나 5초마다 확인하는 알바" → "생기면 알림 오는 구독 버튼")
>
> **태스크 기반(직접 트리거)** 은
> ① **같은 팀·같은 레포** 라 이름 변경을 서로 통제할 수 있거나,
> ② **공유 데이터셋이 아예 없을 때**(예: 완료 후 알림 이메일 발송)에 씀 — `TriggerDagRunOperator`·`ExternalTaskSensor` 가 대표적.
>
> 정리 — **cross-team = 데이터 기반**, **팀 내부·비(非)데이터 트리거 = 태스크 기반** 을 기본값으로 잡을 것.

#### 고려사항 (Consequences)

가장 큰 난제는 **모든 파이프라인의 동기화 유지**. 가장 큰 단점은 **논리적 격리가 운영 제약을 하나 더** 추가한다는 것.

- **Scheduling (스케줄링)**
  - 태스크 기반은 **스케줄 빈도** 가 얽힘. 이상적으로 두 파이프라인이 같은 스케줄을 공유 → 프로바이더가 컨슈머를 바로 트리거.
  - 그렇지 않으면 한쪽이 복잡도를 떠안음 — 프로바이더면 일부 실행 스케줄에서 **트리거를 건너뛰는 조건** 을 추가,
    컨슈머면 프로바이더에 스케줄을 맞추고 **필요할 때만** 실제 처리를 실행.
- **Communication (커뮤니케이션)**
  - Isolated Sequencer 는 **서로 다른 팀** 이 관리하는 파이프라인을 다룸. 조직에 좋은 소통 문화가 없으면 문제.
  - 극복이 쉽지 않음 — 프로바이더/컨슈머 어느 쪽이든 상대편 조건 변경을 확인하는 메커니즘을 넣을 수 있으나 노력이 많이 들고,
    **보안상 상대가 인바운드 연결을 안 받으면** 구현 자체가 불가능할 수도.
  - 더 견고한 접근은 **조직적 변화** — 각 격리 팀이 자기 입출력을 인지하고, 격리된 파이프라인은 서로 코드를 못 보므로, 컬럼명·경로·의미를 바꾸는 팀이 그 데이터를 쓰는 모든 컨슈머에게 미리 알려야 한다

#### 구현 예시 (Examples)

**예시 1 — 데이터 기반 (Example 6-4, Apache Airflow)**

`devices_loader`(프로바이더)와 `devices_aggregator`(컨슈머)의 **암묵적 의존**.
컨슈머의 `input_data_sensor` 가 일종의 **의존성 강제자(dependency enforcer)** — 실패하면 프로바이더가 약속대로 데이터를 못 내놨다는 뜻.
(단, 두 파이프라인이 상호의존인지 **코드만 봐선 알 방법이 없음**):
```python
# devices_loader (프로바이더) — 내부 스토리지에 데이터셋 생성
@task
def load_new_devices_to_internal_storage():
    ctx = get_current_context()
    partitioned_dir = f'{devices_file_location}/{ctx["ds_nodash"]}'
    internal_file_location = f'{partitioned_dir}/dataset.csv'
    shutil.copyfile(input_devices_file, internal_file_location)
input_data_sensor >> load_new_devices_to_internal_storage()

# devices_aggregator (컨슈머) — FileSensor 로 데이터 존재(marker)를 감지
input_data_sensor = FileSensor(
    task_id='input_data_sensor',
    filepath=devices_file_location + '/{{ ds_nodash }}/dataset.csv',
)
input_data_sensor >> load_data_to_table >> refresh_aggregates
```

**예시 2 — 태스크 기반 (Example 6-5, Apache Airflow)**

`ExternalTaskMarker` + `ExternalTaskSensor` 로 두 파이프라인을 연결. 센서는 컨슈머가 프로바이더 태스크 실행을 **기다림** 을 표현.
마커는 **backfill 자동화** — `devices_loader` 에서 일어난 backfill 을 감지해 컨슈머 쪽에서 자동 실행:
```python
# devices_loader (프로바이더) — 실행 성공을 다운스트림에 알리는 마커
success_execution_marker = ExternalTaskMarker(
    task_id='trigger_downstream_consumers',
    external_dag_id='devices_aggregator',
    external_task_id='downstream_trigger_sensor',
)
(input_data_sensor >> load_new_devices_to_internal_storage()
    >> success_execution_marker)

# devices_aggregator (컨슈머) — 프로바이더의 태스크 실행을 감지
parent_dag_sensor = ExternalTaskSensor(
    task_id='downstream_trigger_sensor',
    external_dag_id='devices_loader',
    external_task_id='trigger_downstream_consumers',
    allowed_states=['success'],
    failed_states=['failed', 'skipped']
)
parent_dag_sensor >> load_data_to_table >> refresh_aggregates
```

> **참고 사항 — 데이터 리니지(Data Lineage)**
> 복잡한 데이터 시스템에서 데이터셋 간 의존성을 표현하는 중요한 컴포넌트가 **데이터 리니지 도구**.
> 위 예에서 `devices_loader` 의 출력을 **어떤 데이터셋·파이프라인이 소비하는지** 볼 수 있음.
> 챕터 10에서 두 가지 데이터 리니지 패턴을 다루며, 오늘날 주요 데이터 도구를 지원하는 오픈소스 표준 **OpenLineage** 로도 리니지를 배울 수 있음.

> **트러블 로그** — 태스크 기반(task-based) 의존을 쓰면서 컨슈머 팀이 프로바이더에 알리지 않고 태스크 이름(`external_task_id`)을 바꾸면,
> 프로바이더의 `ExternalTaskMarker` 가 **존재하지 않는 태스크** 를 트리거하려다 의존 체인 전체가 실패.
>
> **예 —** `devices_aggregator` 가 `downstream_trigger_sensor` 를 `trigger_sensor_v2` 로 리네이밍한 순간,
> `devices_loader` 쪽 마커(`external_task_id='downstream_trigger_sensor'`)가 매 실행마다 깨져 프로바이더 DAG 까지 빨간불.
>
> **권장 —** 두 팀이 인터페이스 변경을 사전에 공유할 문화가 없다면, 강결합인 태스크 기반 대신
> **marker 파일 규약만 지키면 되는 데이터 기반(loosely coupled)** 으로 의존을 표현할 것.

---

## 2. 팬인 (Fan-In)

앞의 두 패턴(#33·#34)은 **시퀀스** — 단계가 특정 순서로 이어짐.
하지만 파이프라인이 늘 그렇게 단순하진 않음. 여러 브랜치가 만들어졌다가 **어느 지점에서 다시 합류(merge)** 함.
이 합류 지점이 **팬인(Fan-In)** 패턴 패밀리가 관여하는 곳.

- **#35 Aligned Fan-In** — **모든 직속 부모** 태스크가 성공해야 진행.
- **#36 Unaligned Fan-In** — **일부 부모가 실패해도** 진행(부분 데이터 허용 / 실패 처리).

---

### 2-1. 패턴 #35: 정렬된 팬인 (Aligned Fan-In)

> 가장 설명하기 쉬운 팬인 패턴 — **모든 직속 부모 태스크가 성공** 해야 다음으로 진행.

#### 상황 (Problem)

**책의 use case** — 시간별로 파티션된 블로그 방문 데이터를 **일(day) 단위** 로 집계:

- 파이프라인이 raw 방문 이벤트에서 블로그 방문의 **일별 집계** 를 생성.
- 그런데 데이터셋은 **시간(hour) 단위로 파티션** 돼 있음 — 조직 대부분의 use case 에 그 구성이 맞기 때문.
- 컨슈머 (다운스트림) 는 **전체 뷰(full view)** 에만 관심 → 굳이 일 단위로만 돌릴 이유는 없음.
  다만 **시간별 파티셔닝을 활용** 해 단일 잡이 너무 많은 데이터를 처리하지 않게 하고 싶음.
- **결정적 제약**: 여러 부모 태스크(시간별 부분 집계)와 그 뒤에 도는 **한 태스크**(일별 최종 집계) 사이의 의존.

#### 해결 (Solution)

여러 부모 태스크 + 그 후에 도는 한 태스크의 의존 → **Aligned Fan-In**.

- **데이터 오케스트레이션 레이어** — 간단. **공통 태스크로 합류하는 별도 브랜치들** 을 정의.
  예에서 브랜치 = 하루 각 시간의 부분 집계를 만드는 처리 잡들. 전부 완료되면 merge 태스크가 시간별 결과를 받아 **하루치 최종 출력** 을 계산.
- **데이터 처리 레이어** — 브랜치를 정의하되, **브랜치들이 서로 어떻게 상호작용하는지** 도 정의해야 함. SQL 로 이해하는 게 최선.
  - **UNION** — 여러 출력이 만든 하나의 데이터셋. **행 수는 늘고 열 수는 동일** (세로 정렬).
  - **JOIN** — 행 방향으로 결합해 **열을 추가** (가로 정렬). JOIN 은 항상 UNION 보다 행이 적음.
- **큰 이점 — 피드백 루프 최적화**. 패턴을 안 쓰고 마지막 시간 처리에 문제가 있으면, 하루치를 한꺼번에 처리하므로 **실행이 끝날 무렵에야** 에러가 남.
  반면 로직을 분리하면 **24개 작은 잡이 작은 데이터를 동시에** 처리 → 문제를 **더 빨리** 반환.
- **실패 관점 이점** — 24시간 중 마지막만 실패하면 **그 시간만** 고쳐 재실행.
  Aligned Fan-In 없이 24시간을 한 잡으로 처리했다면 **나머지 23시간도 다시** 처리해야 함. (Local Sequencer 처럼 **backfill 용이성** 기준으로 분리)

```
[Figure 6-3 재현] Aligned Fan-In — 시간별 부분 집계 → 일별 최종 집계
──────────────────────────────────────────────────────────────────────
 Readiness marker h1  ─►  Partial aggregates h1  ┐
 Readiness marker h2  ─►  Partial aggregates h2  ┤
         ...                     ...             ├─►  Final aggregates
 Readiness marker h24 ─►  Partial aggregates h24 ┘
──────────────────────────────────────────────────────────────────────
 24개 시간별 브랜치가 "모두 성공"해야 최종 집계(merge) 태스크가 실행됨.
```

**쉽게 풀어 보면 (UNION 세로 vs JOIN 가로)** — 두 레이어의 관심사가 다름
— **데이터 오케스트레이션 = "언제 합류하나"(순서)**, **데이터 처리 = "어떻게 결합하나"(데이터)**.
데이터 처리 레이어에서 브랜치 결과를 실제로 합치는 방식이 UNION / JOIN 둘.

- **UNION (세로로 쌓기)** — **같은 스키마** 결과를 위아래로 이어붙임. Aligned Fan-In 의 기본형
  — 24시간이 전부 `(hour, visits)` 로 같으니 쌓아서 하루치를 만듦. ⇒ **행 ↑ · 열 그대로**.
- **JOIN (가로로 붙이기)** — **같은 키(hour)** 기준으로 **다른 지표** 를 옆에 붙임(예: 방문수 + 매출).
  ⇒ **열 ↑ · 행 그대로**.

```
[UNION — 세로로 쌓기: 행 ↑, 열 그대로]
──────────────────────────────────────────────
 h1 결과             h2 결과
 | hour | visits |   | hour | visits |
 |  01  |  100   |   |  02  |  150   |
         │ UNION
         ▼
 | hour | visits |
 |  01  |  100   |
 |  02  |  150   |   ← 행이 쌓임(2행), 열은 2개 그대로

[JOIN ON hour — 가로로 붙이기: 열 ↑, 행 그대로]
──────────────────────────────────────────────
 방문 브랜치          매출 브랜치
 | hour | visits |   | hour | revenue |
 |  01  |  100   |   |  01  |  5000   |
 |  02  |  150   |   |  02  |  7000   |
         │ JOIN ON hour
         ▼
 | hour | visits | revenue |
 |  01  |  100   |  5000   |
 |  02  |  150   |  7000   |   ← 열이 늘어남(visits+revenue), 행은 2행 그대로
──────────────────────────────────────────────
 같은 두 데이터셋(각 24행) → UNION = 48행(쌓기) · JOIN = 24행(키 매칭). 그래서 "JOIN 이 항상 행이 적음".
```

| 연산 | 언제 | 결과 | Fan-In 예 |
|---|---|---|---|
| **UNION** (세로) | 브랜치들이 **같은 스키마**, 다른 조각(h1~h24) | **행 ↑**, 열 그대로 | 24시간 방문수를 쌓아 하루치 |
| **JOIN** (가로) | 브랜치들이 **다른 지표**, 같은 키 | **열 ↑**, 행 그대로 | 시간별 방문수 + 매출을 옆에 붙이기 |

#### 고려사항 (Consequences)

decoupling을 통한 느슨한 결합이 유익하지만 **함정** 이 있음.

- **Infrastructure spikes (인프라 스파이크)**
  - 일별 집계가 **24개 잡을 동시에** 돌림. 탄력적 프로비저닝이면 문제 없음. 아니면 균형을 찾아 **동시 실행 수를 줄여야** 함.
  - 대안 — 브랜치를 가능한 한 빨리, **완전 증분** 으로 실행. 24개 시간 브랜치라면 매시간 각 브랜치를 돌리고 **하루 마지막 실행에서만** 결과를 집계.
- **Scheduling skew (스케줄링 스큐)**
  - 자식 태스크가 **모든 부모의 성공** 을 요구하므로, 부모들의 실행 시간이 불균형하면 스큐 발생.
    성공한 태스크들이 **가장 느린 하나** 를 기다림 → 자식 트리거 시각이 **가장 긴 부모** 에 좌우됨.
- **Scheduling overhead (스케줄링 오버헤드)**
  - 너무 잘게 쪼갠 파이프라인은 **스케줄링 오버헤드** 도 큼. 오케스트레이터가 태스크 스케줄·조율에 자원 대부분을 써야 함.
    하지만 decouple·가독성 개선의 대가.
- **Complexity (복잡도)**
  - 많이 decouple 할수록 파이프라인이 길어짐 → 가독성·이해도 저하. **만능 해법 없음**, 파이프라인마다 올바른 경계를 개별 식별.
  - 핵심 질문 — "어떤 태스크들이 **하나의 실행 단위** 에 속하나?", "어떤 연산을 **개별적으로 backfill** 해야 하나?"

#### 구현 예시 (Examples)

**예시 1 — 동적 태스크 생성 (Example 6-6, Apache Airflow)**

`for` 루프 안에서 시퀀스를 **한 번만** 정의 → Airflow 가 반복마다 전용 태스크를 만들고 공통 `clear_context` 부모에 연결:
```python
clear_context = PostgresOperator(...)
generate_trends = PostgresOperator(...)

for hour_to_load in [f"{hour:02d}" for hour in range(24)]:
    file_sensor = FileSensor(
        task_id=f'wait_for_{hour_to_load}',
        filepath=input_dir + '/date={{ ds_nodash }}/hour=' + hour_to_load + '/dataset.csv'
    )
    visits_loader = PostgresOperator(
        task_id=f'load_hourly_visits_{hour_to_load}',
        params={'hour': hour_to_load}
    )
    clear_context >> file_sensor >> visits_loader >> generate_trends
```

> 정적 태스크 목록이면 같은 레벨을 `[...]` 로 선언 가능(예: `[load_data_1, load_data_2] >> process_data`).
> 더 장황하지만 **브랜치-자식 연결이 명시적** 이라 가독성이 좋음.

**예시 2 — UNION (Example 6-7, PySpark)**

두 데이터셋을 하나로 합쳐 그 위에 **하나의 변환** 을 적용:
```python
input_dataset_1: DataFrame = ...
input_dataset_2: DataFrame = ...
output_dataset = input_dataset_1.unionByName(input_dataset_2)   # 열 "이름" 기준 결합
```

**예시 3 — 위치 기반 오류 (Example 6-8, SQL)**

`UNION` 은 기본이 **위치(position) 기반** → 호환 안 되는 필드도 그대로 합쳐져 오염된 결과:
```sql
SELECT a, b, c FROM abc
UNION
SELECT c, b, a FROM cba
```

> 위 연산은 `a+c`, `b+b`, `c+a` 가 합쳐진 **비호환 데이터셋** 을 만듦.
> Spark 의 `unionByName` 은 위치가 아니라 **열 이름** 으로 결합해 이를 완화. 다만 널리 쓰이진 않으니 기본 `UNION` 이 위치 기반임을 유의.

> **트러블 로그** — `UNION` 을 열 순서만 믿고 쓰면 스키마가 미묘하게 어긋난 두 브랜치가 **조용히 잘못 합쳐짐**.
>
> **예 —** 방문 로그 브랜치 A 는 `(user_id, page, ts)`, 브랜치 B 는 `(page, user_id, ts)` 순서인데
> `SELECT * ... UNION SELECT * ...` 로 합치면 `user_id` 열에 `page` 값이 섞여 들어가 집계가 깨짐(에러 없이 값만 오염되기도 함).
>
> **권장 —** 스키마가 100% 동일하다고 보장 못 하면 위치 기반 `UNION` 대신 **`unionByName`(또는 컬럼을 명시한 SELECT)** 으로 이름 기준 결합할 것.

---

### 2-2. 패턴 #36: 비정렬 팬인 (Unaligned Fan-In)

> "모든 부모가 성공해야 한다" 는 조건은 지연만 늘리는 게 아니라 **의미상 틀릴 수도** 있음. 그래서 Aligned Fan-In 의 변형이 필요.

#### 상황 (Problem)

**책의 use case** — 시간별 방문 집계를 Aligned Fan-In 으로 몇 주간 잘 돌렸으나:

- **실패한 태스크를 관리하는** 제대로 된 방법이 없음.
- 몇 번, 한 시간이 제대로 처리되지 않아 다운스트림용 집계 뷰를 **아예 만들지 못함**.
- 미팅 결과, **부분 데이터셋이라도 릴리스** 하고 나중에 빈틈을 채우는 게 낫다고 합의.
- **결정적 제약**: Aligned Fan-In 의 **"모든 부모 성공" 의존을 완화** 해야 함.

#### 해결 (Solution)

부모 결과에 대한 의존을 완화 → **Unaligned Fan-In**. **일부 부모가 실패해도** 자식 태스크 실행 가능. 두 시나리오:

- **일부 부모 성공 + 나머지 실패가 허용 가능** → 자식 태스크를 그냥 트리거.
  단 부분 입력으로 작업하므로 → **부분 데이터셋(partial dataset) 문제** 발생.
- **모든 부모 실패** → 성공 기준 대신 **실패 기준** 으로 태스크 스케줄. 예: **fallback 또는 오류 관리 태스크**.
- 구현 — 오케스트레이션 도구의 **트리거 조건** 설정. Airflow 는 `trigger_rule` 속성을 가용 상태 중 하나로.
  덜 선언적인 환경에선 부모 결과를 **명시적으로 평가**.

**쉽게 풀어 보면 (블로그 방문 집계 예)** — "모두 성공" 조건을 풀면 두 갈래가 생김.
24시간 부분 집계 → 일별 집계 상황에 대입.

- **시나리오 1 — 일부만 성공 (부분 데이터셋)** — 13시(h13) 소스 파일이 안 들어와 실패, 나머지 23시간 성공.
  - Aligned 이면 ⇒ 자식(일별 집계)이 **아예 안 돌아** 그날 대시보드가 통째로 공백.
  - Unaligned 이면 ⇒ **23시간치로 그냥 일별 집계를 생성**(trigger anyway). 단 이건 **95.8%(23/24)짜리 부분 데이터셋**
    — 컨슈머가 완결 데이터로 오인하지 않게 **`is_approximate=true`** 를 함께 실어야 함.
- **시나리오 2 — 전부 실패 (fallback)** — 업스트림 소스 시스템 전체 장애로 24시간 **모두 실패**.
  - 성공 기준으로는 자식이 **영영 안 돎** → 그냥 방치하면 장애를 아무도 모름.
  - 대신 **"실패 기준"** 으로 별도 태스크를 스케줄 — 예: **어제 스냅샷을 그대로 유지(fallback)** + **온콜 알림·incident 티켓(오류 관리)**.

```
[두 갈래 — 트리거 기준으로 갈림]
──────────────────────────────────────────────────────────────────────
 h1~h24 부분 집계
    ├─ 23개 성공 · h13 실패  ─(ALL_DONE)──►  일별 집계 실행 (부분 데이터셋, is_approximate=true)
    └─ 24개 전부 실패        ─(ALL_FAILED)─►  fallback + 오류 관리 (어제 값 유지 · 온콜 알림)
──────────────────────────────────────────────────────────────────────
 ⚠ 성공 기준(ALL_SUCCESS)만 쓰면 두 경우 다 "자식이 안 돎" → 대시보드 공백. 실패도 하나의 경로로 다뤄야 함.
```

Airflow `trigger_rule` 로 이 갈래를 표현:

| trigger_rule | 자식이 실행되는 조건 | 이 패턴에서의 쓰임 |
|---|---|---|
| `ALL_SUCCESS` (기본) | 모든 부모 **성공** | Aligned Fan-In (#35) |
| `ALL_DONE` | 결과 무관, 모든 부모 **완료** | 시나리오 1 — 부분이라도 진행 |
| `ALL_FAILED` | 모든 부모 **실패** | 시나리오 2 — fallback·오류 관리 |

#### 고려사항 (Consequences)

Aligned Fan-In 의 단점을 **공유하면서 새 단점도** 추가.

```
[Figure 6-4 재현] 혼란스러운 Unaligned Fan-In — 성공 갈래·실패 갈래 공존
──────────────────────────────────────────────────────────────────────
 Partial aggregates h1  ┐
 Partial aggregates h2  ┤        ┌─►  Final aggregates      (모든 부모 성공 시)
         ...            ├───────►┤
 Partial aggregates h24 ┘        └─►  Partial aggregates     (일부 실패 시)
──────────────────────────────────────────────────────────────────────
 ⚠ 성공/실패 두 갈래가 그래프에 동시에 걸려, 코드를 안 보면 실행 흐름 판단이 어려움.
```

- **Readability (가독성)**
  - 데이터 흐름의 가독성·이해도가 떨어질 수 있음. 특히 다운스트림 태스크를 **두 종류**(모든 부모 성공 시 / 실패가 있을 때)로 넣으면,
    **코드를 안 보곤** 실행 흐름을 알기 어려워 파이프라인이 혼란스러워짐(Figure 6-4).
  - 완화 — 오케스트레이션 도구가 **기타 시나리오용 커스텀 함수** 를 제공하는지 확인.
    Airflow 는 `on_failure_callback` 으로 실패 호출을 관리, AWS Step Functions 는 `Catch` 필드로 실패 핸들러를 직접 트리거.
- **Partial data (부분 데이터)**
  - 부분 성공한 부모로 데이터셋을 만들면, **이 사실을 컨슈머에게 반드시 공유**. 안 그러면 완전성(completeness)에 대해 잘못된 가정을 함.
  - 방법 ① 가장 단순: 생성 데이터셋과 짝을 이루는 **completeness 테이블** 에 완전성 지표 저장(예: 24개 중 12개 성공 → 50%, 6개 → 25%).
    ② **메타데이터 레이어** 에 추가(예: 오브젝트 스토리지 태그). 
    ③ 정적 정보 + **다운스트림 알림**(예: 이메일).
    ④ 부분 데이터셋을 **비공개** 로 둘 수도 — 이때는 완전성을 먼저 판정해 **100%가 아니면 내부 테이블/폴더** 에 기록.

#### 구현 예시 (Examples)

**예시 1 — 트리거 조건 (Example 6-9, Apache Airflow)**

Example 6-6 과 **동일하되 `trigger_rule` 만** 다름 — 부모 성공 여부와 무관하게 "모두 완료" 면 실행:
```python
clear_context = PostgresOperator(...)
generate_cube = PostgresOperator(
    # ...
    trigger_rule=TriggerRule.ALL_DONE   # 결과 무관, 모든 부모가 "완료"되기만 하면 실행
)

for hour_to_load in [f"{hour:02d}" for hour in range(24)]:
    file_sensor = FileSensor(
        task_id=f'wait_for_{hour_to_load}',
        filepath=input_dir + '/date={{ ds_nodash }}/hour=' + hour_to_load + '/dataset.csv'
    )
    visits_loader = PostgresOperator(
        task_id=f'load_hourly_visits_{hour_to_load}',
        params={'hour': hour_to_load}
    )
    clear_context >> file_sensor >> visits_loader >> generate_cube
```

> Airflow 기본은 **모든 부모 성공** 시에만 태스크 시작. 여기선 `ALL_DONE` — 결과와 무관하게 모든 부모가 완료되기만 하면 실행.
> 다만 이 규칙이 **코드에 숨어** 있어 그래프만 봐선 드러나지 않음.

**예시 2 — 근사(approximate) 플래그 (Example 6-10, SQL)**

처리된 시간 수가 **24와 다르면** 해당 행을 근사로 표시(서브쿼리로 완전성 판정):
```sql
INSERT INTO dedp.visits_cube (..., is_approximate)
SELECT ...,
 (SELECT CASE WHEN hours_subquery.all_hours = 24 THEN false ELSE true END FROM
   (SELECT COUNT(DISTINCT execution_time_id) AS all_hours FROM dedp.visits_raw
     WHERE execution_time_id = '{{ ds }}')
  AS hours_subquery)
FROM dedp.visits_raw GROUP BY CUBE(...);
```

**예시 3 — lambda 함수 3개 (Example 6-11, AWS Step Functions)**

`Map` 태스크로 각 파티션을 개별 처리하고, 마지막에 오류가 있으면 테이블에 `is_partial` 을 붙임:
```python
# lambda-partitions-detector — 처리할 새 파티션 탐지 (1회 실행)
def lambda_handler(event, context):
    partitions_to_process = []
    return partitions_to_process

# lambda-partitions-processor — Map 태스크, 각 파티션마다 개별 실행 → 성공/실패 플래그
def lambda_handler(event, context):
    processing_result: bool
    return processing_result

# lambda-table-creator — 처리 플래그를 모아, 하나라도 실패면 is_partial 표시
def lambda_handler(event, context):
    table_metadata = {}
    if False in event['ProcessorResults']:
        table_metadata['is_partial'] = True
    return True
```

> **트러블 로그** — Unaligned Fan-In 으로 부분 데이터셋을 내보내면서 **완전성(completeness)을 컨슈머에 알리지 않으면**,
> 다운스트림이 24시간치 완결 데이터로 오인해 잘못된 의사결정을 함.
>
> **예 —** 24시간 중 6시간만 성공한 방문 집계(실제 **25% 커버리지**)를 그대로 대시보드에 노출하면,
> 마케팅팀이 "오늘 방문이 전일 대비 급감" 이라 오판해 캠페인을 중단할 수 있음.
>
> **권장 —** `is_approximate`/`is_partial` 플래그나 completeness 테이블(성공 부모 수 ÷ 전체)로 **완전성 비율을 함께 실어** 보내고,
> 100%가 아니면 **부분 데이터임을 명시** 할 것.

---

## 3. 팬아웃 (Fan-Out)

지금까지는 **공통 태스크로 합류** 하는 파이프라인을 봤음(팬인). 하지만 마지막 유형 — **한 태스크가 여러 태스크의 입력** 이 되는 경우 — 이 빠져 있음.
하나의 데이터셋을 **여러 팀이 서로 다른 목적**(데이터 분석, 데이터 사이언스 등)으로 쓸 때 유용.

- **#37 Parallel Split** — 한 부모에서 **둘 이상의 자식 브랜치를 병렬 실행**.
- **#38 Exclusive Choice** — 조건에 따라 **하나의 브랜치만** 실행 (아래 3-2 절).

---

### 3-1. 패턴 #37: 병렬 분할 (Parallel Split)

> 첫 팬아웃 패턴. **한 부모 태스크가 둘 이상의 자식 태스크의 전제조건**.
> 자식들은 로직이 격리돼 **병렬 실행 가능** 하며, 유일한 공통점은 **같은 부모 요구**.

#### 상황 (Problem)

**책의 use case** — 레거시 C# 프레임워크를 최신 Python 으로 마이그레이션:

- 조직에서 **아무도 모르는** C# 데이터 처리 프레임워크를 교체하려 함. 유지보수자들이 **유용한 문서 없이** 퇴사.
- 리버스 엔지니어링으로 로직을 최신 오픈소스 Python 라이브러리로 재작성 중.
- 파이프라인을 마이그레이션해야 하는데, 리버스 엔지니어링이 완벽하지 않을 수 있어
  **컨슈머가 새 솔루션으로 옮길 때까지 옛 파이프라인을 계속** 돌리고 싶음.
- **결정적 제약**: 마이그레이션 동안 처리된 데이터셋을 **두 곳(old/new)에 써야** 함.

#### 해결 (Solution)

두 다른 연산이 **하나의 공통 부모** 에 의존 → 팬아웃. 첫 패턴 **Parallel Split** — 작업을 **병렬로 쪼갬**.

- **오케스트레이션 레이어** — 간단. 데이터 흐름 정의 API(**DSL** 또는 함수 같은 고수준 추상)에 의존.
- **데이터 처리 세계** — 그리 간단치 않음. 주의점 세 가지:
  - **공통 계산은 한 번만** — split 처리가 별도의 데이터 읽기를 유발하면 안 됨.
    SQL 임시 테이블이나 Spark `.persist()` 로 **중간 데이터셋을 물질화(materialize)** 해 달성.
  - **브랜치 간 비간섭** — 코드가 전역·공유 변수를 쓰면 **읽기 전용** 이거나 모든 쓰기 프로세스에 **호환되는 수정** 이어야 함.
  - **실행 시간 민감** 하면 **전용 컴퓨트 할당** 또는 오토스케일링으로 병렬 워크로드를 수용.


**쉽게 풀어 보면 (마이그레이션 예: 한 번 읽어 두 곳에 쓰기)** — 정제·처리한 입력을 **old(csv) · new(delta)** 두 포맷으로 동시에 내보내는 상황.
한 부모(read + 정제)가 두 브랜치(delta 저장 · csv 저장)의 공통 전제 → **Parallel Split**.

```
[한 번 읽어 두 곳에 쓰기 — Parallel Split]
──────────────────────────────────────────────────────────────────────
                         ┌─►  write delta  (new)   ← 브랜치 A
 read + 정제 → persist() ─┤
                         └─►  write csv    (old)   ← 브랜치 B
──────────────────────────────────────────────────────────────────────
 공통 부모(read + 정제)는 1회만, 결과를 persist 로 캐싱해 두 브랜치가 공유.
```

세 주의점을 **✗ 하기 쉬운 실수 / ✓ 올바른 처리** 로 대비하면:

- **주의 1 · 공통 계산 1회 (materialize)**
  - `✗` 브랜치마다 `spark.read...` 를 따로 호출 → 원본을 **2회 스캔**(비용 2배 + 그 사이 새 파일 유입 시 브랜치별 스냅샷이 어긋남).
  - `✓` 한 번만 읽어 `input_dataset.persist()`(또는 SQL 임시 테이블) → 두 브랜치가 **캐시된 결과를 공유**.
- **주의 2 · 브랜치 간 비간섭**
  - `✗` 두 브랜치가 **공유 `config` 딕셔너리** 를 각자 수정(A: `mode='overwrite'`, B: `mode='append'`) → 경쟁 조건으로 결과 오염.
  - `✗` 두 브랜치가 **같은 경로/테이블** 에 씀 → 쓰기 충돌.
  - `✓` 공유 객체는 **읽기 전용**, 브랜치별 설정은 **각자 복사본**, 출력 타깃은 **서로 다른 경로**(delta 경로 ≠ csv 경로).
- **주의 3 · 실행 시간 민감 → 전용 컴퓨트**
  - `✗` 두 브랜치가 한 클러스터에서 **자원 경쟁** → 둘 다 느려져 SLA 위반.
  - `✓` SLA 민감하면 **브랜치별 전용 컴퓨트/오토스케일링**. 한쪽이 CPU 편중·한쪽이 메모리 편중이면 Figure 6-5 처럼 **하드웨어를 분리**.

#### 고려사항 (Consequences)

- **Blocked execution (실행 차단)**
  - 오케스트레이션 레이어 + **이전 실행이 성공해야 다음이 도는** 시간 의존 파이프라인에 해당.
    Parallel Split 은 트리거 조건이 **가장 느린 브랜치** 기준 → 파이프라인이 가장 느린 브랜치 완료를 기다림.
  - 실패가 있으면 더 나쁨 — **후속 실행이 아예 안 돎**. 이 위험이 있으면 느린 브랜치를 **전용 파이프라인** 으로 빼고,
    **Isolated Sequencer 의 트리거 규칙(데이터/태스크 기반 의존)** 으로 조건화하는 것을 고려.
- **Hardware (하드웨어)**
  - 데이터 처리 레이어 관점. 메인 잡이 두 다른 잡을 위한 중간 데이터셋을 만들어야 하면 두 잡이 **같은 하드웨어 기대치** 를 가져야 함.
    하나는 **CPU 편중**, 하나는 **메모리 편중** 이면 딱 맞는 인프라를 쓸 수 없음.
  - 완화 — Local Sequencer 에서처럼 잡을 나눠 **중간 데이터셋을 만든 뒤**, 병렬 잡들을 각자 **전용 하드웨어** 에서 시작(Figure 6-5).

```
[Figure 6-5 재현] 같은 기반 데이터셋 + 다른 컴퓨트 요구 → 잡 분할
──────────────────────────────────────────────────────────────────────
 ··►  Intermediary dataset generator  ┬─►  CPU-bounded job      (CPU 편중)
                                      └─►  Memory-bounded job   (메모리 편중)
──────────────────────────────────────────────────────────────────────
 중간 데이터셋을 먼저 물질화한 뒤, 두 잡을 각자 전용 하드웨어에서 병렬 실행.
 (점선 ··► = 이 예와 무관한 이전 태스크)
```

**쉽게 풀어 보면 (마이그레이션 예 이어서)** — delta·csv 두 브랜치로 나눈 뒤 생기는 두 함정.

- **Blocked execution — 가장 느린/취약한 브랜치가 전체를 잡음**
  - 매일 도는 마이그레이션 DAG(어제 성공해야 오늘 실행되는 **시간 의존**)에서, 브랜치 A(delta)는 5분, 브랜치 B(csv, 레거시 시스템)는 40분이 걸림.
  - ⇒ 부모 "완료" = 가장 느린 40분 → delta 가 5분에 끝나도 **전체가 40분 대기**.
  - `✗` csv 브랜치가 레거시 장애로 실패 → 오늘 run 실패 → **(어제 성공 의존) 내일 run 이 안 떠** 마이그레이션 전체가 정지.
  - `✓` 느린 csv 를 **전용 파이프라인으로 빼** 데이터 기반 트리거(Isolated Sequencer)로 연결 → 메인 DAG 는 delta 만 보고 진행.

```
[Blocked execution — 느린 브랜치가 전체를 잡고, 실패는 다음 실행까지 막음]
─────────────────────────────────────────────────────────────────────────────────────
 read+정제 ─┬─► delta 쓰기  ▓▓                     (5분, 금방 끝남)
           └─► csv 쓰기    ▓▓▓▓▓▓▓▓▓▓             (40분, 레거시 느림)
 Parallel Split 실행(run)이 끝났다고 인정되는 시점 = 40분  ⇒ delta 가 끝나도 전체는 ⚠ 40분 대기
 ✗ csv 실패 → 오늘 run 실패 → (어제 성공 의존) 내일 run 안 뜸 → 마이그레이션 정지
─────────────────────────────────────────────────────────────────────────────────────
 타임라인 예) Day1 02:00 시작 → 02:08 delta ✓ / 02:20 csv ✗ → Day1 실패
              → Day2 run 시작 안 됨 → Day3·Day4 줄줄이 정지
```

위 `✓` 완화책 — **느린 브랜치를 전용 파이프라인으로 떼어내고 Isolated Sequencer 트리거로 잇기** — 를 그림으로:

```
[Blocked execution 완화 — 느린 브랜치를 전용 파이프라인으로 분리]
──────────────────────────────────────────────────────────────────────
 ✗ Before — 한 파이프라인 안에 느린 csv 까지 묶음 (Parallel Split)
    read+정제 ─┬─► delta 쓰기  (5분)
               └─► csv 쓰기    (40분, 느리고 잘 깨짐)
    ⇒ 부모 완료 = 40분 · csv 실패 시 run 전체 ✗ → 다음 날 run 정지

 ✓ After — 느린 csv 를 별도 파이프라인으로 빼고, 데이터 기반 트리거로 연결
    [메인 파이프라인]                                [csv 전용 파이프라인]
    read+정제 ─► delta 쓰기 ─► _SUCCESS 마커  ┈감지┈►  csv 쓰기 (40분)
    ⇒ 메인은 5분에 끝 · csv 가 실패/지연돼도 메인은 무관 (Isolated Sequencer)
──────────────────────────────────────────────────────────────────────
 핵심 — 병렬(팬아웃) 대신 "빠른 본류 + 느린 곁가지" 로 분리해 실행 차단·연쇄 실패를 끊음.
 (트리거는 #34 의 데이터 기반 = 마커 감지 / 태스크 기반 = 직접 트리거 중 선택)
```

- **Hardware — 브랜치마다 다른 자원을 원함**
  - 브랜치 A (delta 쓰기 + Z-order 최적화)는 **CPU 편중**(압축·정렬), 브랜치 B(대용량 csv join/집계)는 **메모리 편중**(큰 shuffle).
  - `✗` 둘을 한 클러스터/잡에 묶으면 **최악 기준으로 사이징** → 메모리 최적 인스턴스는 CPU 를 놀리고, 반대도 마찬가지. 딱 맞는 인프라를 못 씀.
  - `✓` **중간 데이터셋을 먼저 물질화**(Local Sequencer 식)한 뒤, A 는 CPU 최적·B 는 메모리 최적 인스턴스에서 **별도 잡** 으로 실행(Figure 6-5).

| 브랜치 | 특성 | 맞는 인스턴스 |
|---|---|---|
| A · delta 쓰기 + Z-order | **CPU 편중** | CPU 최적 (예: AWS c-계열) |
| B · 대용량 csv join | **메모리 편중** | 메모리 최적 (예: AWS r-계열) |
| (한 클러스터로 묶으면) | 둘 다 만족하는 크기로 **과다 프로비저닝** → 비용 ↑, 그래도 최적 아님 | |

> **용어 — Z-order** = 관련 데이터를 **같은 파일에 뭉쳐** 쿼리가 불필요 파일을 건너뛰게(data skipping) 하는 Delta Lake 정렬(`OPTIMIZE ... ZORDER BY (col)`).
> 여러 컬럼을 Z자 곡선으로 섞어 **어느 컬럼으로 필터해도** 스킵이 됨. **정렬 + 압축** 연산이라 **CPU 를 많이 씀** → 그래서 브랜치 A 가 CPU 편중.
> (최근 Databricks 는 후속인 **Liquid Clustering** 을 밀지만 "관련 데이터를 뭉쳐 스킵을 늘린다" 는 개념은 같음.)

#### 구현 예시 (Examples)

**예시 1 — Apache Airflow (Example 6-12)**

팬인 예시와 같은 모델이나, 시퀀스 끝이 **공통 태스크로 끝나지 않는** 미묘한 차이. 유일한 공통점은 **첫 공통 연산(`file_sensor`)**:
```python
file_sensor = FileSensor(
    task_id='input_dataset_waiter'
)

for output_format in ['delta', 'csv']:
    load_job_trigger = SparkKubernetesOperator(
        task_id=f'load_job_trigger_{output_format}',
        params={'output_format': output_format}
    )
    load_job_sensor = SparkKubernetesSensor(
        task_id=f'load_job_sensor_{output_format}')

    file_sensor >> load_job_trigger >> load_job_sensor
```

**예시 2 — 입력 캐싱 (Example 6-13, PySpark)**

같은 입력에 다른 처리 로직을 얹으므로 **한 번만 읽으면** 됨 → `persist()`:
```python
input_dataset = (spark_session.read
    .schema('type STRING, full_name STRING, version STRING').format('json')
    .load(DemoConfiguration.INPUT_PATH))
input_dataset.persist(StorageLevel.MEMORY_ONLY)   # 읽은 데이터를 메모리에 캐싱 (공통 계산 1회)
input_dataset.write...   # 브랜치 1
input_dataset.write...   # 브랜치 2
```

> RAM 이 부족할까 걱정되면 `MEMORY_AND_DISK` 등으로 설정해 공간이 차면 **디스크로** 넘기게 할 수 있음.

**예시 3 — 재시도 이중 쓰기 방지 (Example 6-14, Delta Lake)**

`txnVersion`·`txnAppId` 로 **재시도 시 데이터 이중 쓰기** 를 방지 — 값이 안 바뀌므로 **첫 실행만** 물리적으로 쓰고 이후는 무시:
```python
batch_id = 1
app_id = 'devices-loader-v1'

input_dataset.write.mode('append').format('delta')
    .option('txnVersion', batch_id).option('txnAppId', app_id)
    .save(DemoConfiguration.DEVICES_TABLE)

(input_dataset.withColumn('loading_time', functions.current_timestamp())
    .withColumn('full_name',
        functions.concat_ws(' ', input_dataset.full_name, input_dataset.version))
    .write.mode('append').format('delta')
    .option('txnVersion', batch_id).option('txnAppId', app_id)
    .save(DemoConfiguration.DEVICES_TABLE_ENRICHED))
```

> 같은 `batch_id`·`app_id` 면 재실행이 무시됨. 도구에 이런 **쓰기 중복 제거** 기능이 없으면 **챕터 4의 멱등성 패턴** 을 활용.

> **트러블 로그** — Parallel Split 을 데이터 처리 레이어에서 구현하며 **공통 입력을 각 브랜치가 따로 읽게** 두면,
> 같은 소스를 브랜치 수만큼 반복 스캔해 비용·시간이 배로 늘고, 소스가 변경 가능하면 브랜치마다 **다른 스냅샷** 을 읽어 결과가 어긋남.
>
> **예 —** `delta`·`csv` 두 포맷으로 내보내는 잡에서 `spark.read...` 를 브랜치마다 호출하면 원본 JSON 을 **2회 스캔**
> — 그 사이 새 파일이 유입되면 delta 에는 있고 csv 에는 없는 레코드가 생김.
>
> **권장 —** 공통 계산은 **`persist()`(또는 임시 테이블)로 한 번만 물질화** 한 뒤 브랜치들이 그 결과를 공유하게 하고,
> 재시도 이중 쓰기는 **Delta 의 `txnVersion`/`txnAppId`** 나 챕터 4 멱등성 패턴으로 막을 것.

---

### 3-2. 패턴 #38: 배타적 선택 (Exclusive Choice)

> 두 번째 팬아웃 패턴. Parallel Split 과 똑같이 **하나의 공통 부모** 에 의존하지만,
> 자식들을 **병렬로 다 돌리는 대신** 조건에 따라 **딱 하나만** 고름.

#### 상황 (Problem)

**책의 use case** — Parallel Split 로 끝낸 마이그레이션을 다시 진화시키는 경우:

- #37 로 완성한 마이그레이션이 완벽히 잘 돌아감.
- 이제 파이프라인을 진화시켜 **새 잡 버전** 을 **2024년 1월 1일부터만** 실행하려 함.
- 단 **backfill** 할 때(과거 날짜 재처리)는 그 이전 날짜가 여전히 **예전(legacy) 잡** 으로 돌아야 함.
- **결정적 제약**: 이 진화를 **새 파이프라인을 만들지 않고** 해내야 함 — 전체 실행 이력(execution history)을 한 곳에 유지하기 위해.

#### 해결 (Solution)

여전히 자식 태스크는 둘이지만, 이번엔 **한 번에 하나만** 돌아야 함 → **Exclusive Choice**.

- **Parallel Split 과의 공통점** — 최소 둘 이상의 다운스트림 프로세스를 선언하는 것은 동일(오케스트레이터 DSL 또는 프로그래밍 언어 함수).
- **결정적 차이 — 분기 직전 한 단계** — 마지막 공통 지점 바로 뒤에 자식들을 곧장 매다는 게 아니라,
  **어느 경로로 갈지 정하는 조건 평가(condition evaluator) 태스크** 를 하나 끼워 넣음.
- **도구 지원** — 현대 오케스트레이터는 분기 조건 평가를 기본 제공.
  - Airflow(오픈소스) — 전용 태스크 타입인 **branch operator**.
  - Azure Data Factory(서버리스) — **if condition activity** 로 구현.
- **데이터 처리 레이어** — 훨씬 단순. 컴퓨트 레이어 언어의 **`if-else`/`switch`** 문을 그대로 씀.

```
[Figure 6-6 재현] Exclusive Choice — 날짜 조건으로 새 잡 vs 레거시 잡 택일
──────────────────────────────────────────────────────────────────────
                                             ┌─ Yes ─►  New job version    (새 잡)
 Readiness marker ─► Is the date 2024-01-01? ┤
                                             └─ No ──►  Legacy job version (레거시 잡)
──────────────────────────────────────────────────────────────────────
 공통 부모(readiness) 뒤에 "조건 평가" 태스크를 두고, 결과에 따라 두 브랜치 중 하나만 실행.
```

**쉽게 풀어 보면 (마이그레이션 예시 이어서)**
#37 Parallel Split 은 delta·csv 를 **둘 다** 썼지만,
#38 Exclusive Choice 는 날짜(또는 조건)를 보고 **한 잡만** 고름. 둘 다 "공통 부모 + 여러 자식" 이지만 **실행 방식이 정반대**.

| 비교 축 | Parallel Split (#37) | Exclusive Choice (#38) |
|---|---|---|
| 공통 부모 | 있음 | 있음 |
| 자식 실행 | **둘 이상 병렬로 전부** | 조건 평가 후 **하나만** |
| 분기 직전 | 곧장 분기 | **조건 평가 태스크** 를 삽입 |
| 데이터 처리 구현 | `persist()` 후 두 브랜치 | `if-else` / `switch` |
| 대표 위험 | 느린 브랜치가 전체를 막음 | 분기 남발로 가독성 급락 |

#### 고려사항 (Consequences)

가장 큰 위험은 이 패턴의 **유연성** 그 자체 — 브랜치를 원하는 만큼 추가할 수 있고, 브랜치가 많아지면 파이프라인 이해도가 급락.

- **Complexity factory (복잡도 공장)**
  - 앱 코드에서 `if-else` 를 여러 개 쓰는 건 흔해서, 오케스트레이션 레이어에도 똑같이 반복하고 싶어짐.
    하지만 이 레이어에서 조건이 여러 개면 = **실행 브랜치가 여러 개** → 나중에 다시 합류하거나 더 갈라져 **컴포넌트가 거의 읽을 수 없게** 됨.
  - `if-else` 조건 개수에 정해진 임계값은 없음. 
    - 이상 징후를 확인하는 법: **최근 합류한 가상의 동료에게 파이프라인을 설명** 해 보기 → 설명이 간결하지 않고 질문·답변이 꼬리를 물면 파이프라인이 너무 복잡한 것.

```
[Exclusive Choice 남용 — 브랜치가 합류·재분기하며 읽을 수 없게 됨]
──────────────────────────────────────────────────────────────────────
 ✓ 절제된 분기 (조건 1개, 택일 후 곧장 합류)
    readiness ─► 날짜? ─┬─ 이후 ─► new_job ─┐
                       └─ 이전 ─► legacy  ─┴─► publish

 ✗ 브랜치 스파게티 (조건 여러 개가 겹쳐 합류·재분기)
    readiness ─► A? ─┬─Y─► jobA ─► B? ─┬─Y─► jobB ─┐
                     └─N─► jobC ──┐    └─N─► jobD ─┼─► C? ─┬─► jobF
                                  └──────► jobE ───┘       └─► jobG
    ⇒ 그래프만 봐선 "어떤 조건일 때 무엇이 도나" 추적 불가 → 코드를 열어야 앎
──────────────────────────────────────────────────────────────────────
 판단법 — 위 "가상 동료 설명" 이 간결하지 않으면, 분기를 별도 파이프라인으로 쪼갤 신호.
```

- **Hidden logic (숨은 로직)**
  - 데이터 처리 구현에 해당. 잡 안에 조건문·분기가 들어가 **서로 다른 데이터셋을 만들거나 다른 출력 저장소** 와 상호작용하면 나중에 문제.
  - 코드를 쓴 며칠 · 몇 주는 기억하지만, 시간이 지나면 **이 잡에 여러 결과 경로가 있다는 사실 자체를 잊음**.
  - 그래서 Exclusive Choice 는 처리 레이어에 숨기기보다 **데이터 오케스트레이션 레이어 위에** 두는 편이 나음 (Example 6-16 vs 6-15).
- **Heavy conditions (무거운 조건)**
  - 데이터 처리 레이어에서 구현하는데 조건이 **데이터를 처리해야** 알 수 있으면 잡 실행 시간에 영향.
    가능하면 **메타데이터 기반 조건**(데이터셋을 건드리지 않아 더 빠름)을 선호.
  - 데이터 처리를 피할 수 없으면 — 여러 번 처리하지 않도록, 전체 데이터셋을 매번 읽지 않는 **증분 처리** 로 최적화.

> **참고 사항 — Rubber Duck Debugging(고무 오리 디버깅)**
> 가상의 동료(또는 책상 위 고무 오리)에게 소리 내어 설명하는 방법은 흔히 **디버깅 기법** 으로 알려져 있지만,
> 파이프라인의 **복잡도를 진단** 하는 데도 똑같이 쓸 수 있음 — 설명이 장황해지면 그 자체가 "너무 복잡하다" 는 신호.

#### 구현 예시 (Examples)

**예시 1 — 처리 레이어 vs 오케스트레이션 레이어 (Example 6-15 / 6-16)**

Exclusive Choice 는 조건 분기(`if-else`)라서 **어디든 넣을 수 있음**. 그런데 데이터 파이프라인에는 분기를 놓을 자리가 **두 레이어** 있고, 코드는 거의 같아 보여도 **결과는 크게 달라짐**.

- **데이터 처리 레이어 (data processing layer)** — 실제 데이터를 다루는 **잡 코드 내부**(PySpark·SQL 등). 조건이 잡 안에서 평가돼, 오케스트레이터 입장에선 **태스크 하나** 로만 보임.
- **데이터 오케스트레이션 레이어 (data orchestration layer)** — 어떤 잡을 언제 돌릴지 정하는 **DAG/스케줄러**. 조건이 **"어느 잡(진입점)을 부를지"** 를 정해, 분기가 **그래프에 그대로 드러남**.

같은 조건 분기지만 **어느 레이어에 두느냐** 가 유지보수를 가름:
```python
# Example 6-15 — 데이터 처리 레이어: 잡 "안"에 조건 분기 (숨은 로직 위험)
if condition_a:
    process_condition_a()      # 한 잡 안에서 함수만 갈라짐
else:
    process_default()
```
```bash
# Example 6-16 — 데이터 오케스트레이션 레이어(권장): 조건에 따라 "다른 진입점(잡)"을 실행
if condition_a:
    python job_for_condition_a.py    # 아예 다른 잡을 트리거
else:
    python job_for_default.py
```

**무엇이 다른가 — 오케스트레이터가 "보는" 그래프가 다름.**
마이그레이션 예 (2024-01-01 이후 새 잡 / 이전 레거시 잡)에 대입하면, 두 코드가 만드는 파이프라인 모습이 이렇게 갈림:

```
[같은 분기, 다른 레이어 — 오케스트레이션 그래프에 무엇이 보이나]
──────────────────────────────────────────────────────────────────────
 ✗ Example 6-15 (처리 레이어)          ✓ Example 6-16 (오케스트레이션 레이어)
   readiness ─► [ single_job ]           readiness ─► [ 날짜 라우터 ] ─┬─► [ new_job ]
                 └ 내부에 if-else 숨음                                 └─► [ legacy_job ]
   ⇒ 그래프엔 태스크 "1개" 로만 보임        ⇒ 분기와 두 잡이 그래프에 그대로 보임
──────────────────────────────────────────────────────────────────────
 핵심 — 6-15 는 결과 경로가 "코드 안"에, 6-16 은 "그래프 위"에 있음. 이 차이가 아래 4가지로 번짐.
```

**왜 이 차이가 실무에서 중요한가 — 네 가지로 번짐.**

- **① 가시성·온보딩** — 6-16 은 그래프만 봐도 "날짜에 따라 두 잡으로 갈린다" 를 앎.
  6-15 는 태스크가 하나라, 새 팀원이 코드를 열기 전엔 **이 잡에 결과 경로가 여럿이라는 사실 자체를 모름** (→ Hidden logic 함정).
- **② 모니터링·알림** — 6-16 은 `new_job`·`legacy_job` 이 **별도 태스크** 라 각각의 성공/실패·실행시간·재시도를 따로 추적.
  6-15 는 한 태스크라 **어느 경로가 돌았는지** 로그를 심지 않으면 알 수 없고, 실패해도 "어느 브랜치에서" 인지 불명확.
- **③ 재시작 경계 (restart boundary, #33 Local Sequencer 와 연결)** — 6-16 은 실패한 **그 브랜치 잡만** 다시 돌림.
  6-15 는 분기가 잡 안이라 실패 시 **잡 전체를 재시작** → 조건 평가 전 공통 처리까지 다시 함.
- **④ 리소스 배분** — 6-16 은 브랜치별로 **다른 컴퓨트** 를 붙일 수 있음 (예: Delta 브랜치는 CPU 최적, CSV 브랜치는 메모리 최적 — #37 Parallel Split 의 Hardware 참고).
  6-15 는 한 잡이라 **가장 무거운 브랜치 기준으로 오버프로비저닝** 해야 함.

| 관점 | Example 6-15 (처리 레이어) | Example 6-16 (오케스트레이션 레이어) |
|---|---|---|
| 오케스트레이터가 보는 그래프 | 태스크 **1개** (분기 숨음) | **라우터 + 브랜치 잡들** (분기 보임) |
| 온보딩·디버깅 | 코드 열어야 결과 경로 파악 | 그래프만 봐도 파악 |
| 브랜치별 모니터링·재시작 | 불가 (잡 통째로) | 가능 (브랜치별로) |
| 브랜치별 컴퓨트 | 불가 (한 잡) | 가능 |

**그럼 처리 레이어(6-15)는 언제 쓰나 — 책이 6-15 를 든 건 "쓰지 말라" 가 아님.** 분기가 **가벼워** 위 네 가지가 문제 안 될 때는 처리 레이어가 더 단순.

- 분기가 **출력 저장소 · 데이터셋을 바꾸지 않고**(같은 곳에 쓰되 값만 살짝 다름), 결과 경로가 뻔하며, 브랜치별로 따로 모니터링·재시작할 이유가 없을 때.
- 반대로 조건이 **데이터를 실제로 처리해야만 알 수 있는** 경우(Heavy conditions)엔 처리 레이어가 불가피할 수도 — 이땐 전체 데이터셋을 매번 읽지 않는 **증분 처리** 로 비용을 줄일 것.

> **정리** — 분기가 **데이터셋·출력 저장소를 바꾸는 "무거운" 결정** 이면 6-16(오케스트레이션 레이어)으로 **드러내고**,
> 값만 살짝 바꾸는 **"가벼운" 결정** 이면 6-15(처리 레이어)로 **단순하게** 둠. 판단 기준은 "이 분기를 남이 그래프에서 봐야 하나?" 하나.

**예시 2 — Airflow BranchPythonOperator (Example 6-17)**

`BranchPythonOperator` 가 실행 날짜에 따라 다음 브랜치를 **라우팅**:
```python
def get_output_format_route(**context):
    migration_date = pendulum.datetime(2024, 2, 3)
    execution_date = context['execution_date']
    if execution_date >= migration_date:
        return 'load_job_trigger_delta'   # 마이그레이션 날짜 이후 → 새 포맷(Delta)
    else:
        return 'load_job_trigger_csv'      # 이전 → 레거시 포맷(CSV)

format_router = BranchPythonOperator(
    task_id='format_router',
    python_callable=get_output_format_route,
    provide_context=True
)
```

> 실행 날짜로 Delta/CSV 를 고름. 브랜치는 얼마든 늘릴 수 있지만 **늘수록 가독성이 떨어짐**.
> 분기 대신 `start_date`·`end_date` 로 시간 유효성을 나눈 **별도 파이프라인** 을 만드는 선택지도 있음.

**예시 3 — PySpark 잡 파라미터 + Factory (Example 6-18 / 6-19)**

외부 파라미터(`--output_type`)로 브랜치를 고르고, 생성 로직은 **Factory 디자인 패턴(SWE)** 뒤에 숨김:
```python
class OutputType(str, Enum):
    delta_lake = 'delta'
    csv = 'csv'

parser = argparse.ArgumentParser(prog='...')
parser.add_argument('--output_type', required=True, type=OutputType)
args = parser.parse_args()

output_generation_factory = OutputGenerationFactory(args.output_type)   # 조건을 인터페이스 뒤로 은닉
spark_session = output_generation_factory.get_spark_session()
raw_data = (spark_session.read...)
output_generation_factory.write_devices_data(raw_data, args.output_dir)
```

Factory 본문(Example 6-19) — 클라이언트는 `output_type` 만 넘기고, **분기는 전부 Factory 안에** 숨김:
```python
class OutputGenerationFactory:
    def __init__(self, output_type: OutputType):
        self.type = output_type

    def get_spark_session(self) -> SparkSession:
        if self.type == OutputType.delta_lake:              # Delta 면 delta-pip 로 세션 구성
            return configure_spark_with_delta_pip(SparkSession...)
        else:
            return SparkSession.builder...getOrCreate()

    def write_devices_data(self, devices_data: DataFrame, output_location: str):
        if self.type == OutputType.delta_lake:
            devices_data.write.format('delta')...            # Delta 브랜치
        else:
            devices_data.coalesce(1).write...format('csv')...  # CSV 브랜치
```

```
[Factory 로 조건 은닉 — if-else 를 클라이언트에서 걷어냄]
─────────────────────────────────────────────────────────────────────────────────────
 클라이언트 코드             OutputGenerationFactory (인터페이스)               실제 구현
 --output_type=delta ─►  get_spark_session() / write_devices_data() ─┬─► Delta 로직
                         (내부에서 self.type 으로 분기)                   └─► CSV 로직
─────────────────────────────────────────────────────────────────────────────────────
 ⚠ 남은 약점 — 위처럼 두 메서드가 각자 if-else 를 반복. 메서드가 늘면 조건이 곳곳에 흩어짐.
 ✓ 더 나아가려면 — Delta 전용·CSV 전용 "클래스" 를 Factory 가 통째로 돌려줘, 조건 자체를 1회로.
```

**예시 4 — 스키마 변화 기반 (Example 6-20)**

외부 파라미터 대신 **데이터셋 특성(스키마)** 으로 경로를 정할 수도 있음:
```python
input_schema = detect_schema(input_dataset)
output_location = DemoConfiguration.DEVICES_TABLE_LEGACY
if len(input_schema.fields) >= 3:
    output_location = DemoConfiguration.DEVICES_TABLE_SCHEMA_CHANGED
```

> 필드가 3개 이상이면 새 위치, 아니면 레거시 테이블로. **메타데이터 레이어(스키마)** 로 판단해 빠름
> — 단 데이터 자체를 조건으로 쓰면(Heavy conditions) 더 비쌀 수 있음.

> **트러블 로그** — Exclusive Choice 를 **데이터 처리 레이어의 `if-else` 로만** 숨겨 두면, 오케스트레이터에겐 태스크가 **하나로만** 보여서
> 몇 주·몇 달 뒤 "이 잡이 조건에 따라 여러 결과 경로를 낸다" 는 사실 자체를 잊고, 장애·변경 때 엉뚱한 곳을 헤매게 됨.
>
> **예 —** `write_devices_data()` 하나가 `output_type` 에 따라 Delta 테이블 또는 CSV 를 만드는데, DAG 에는 `devices_export` 태스크 **하나만** 초록불로 뜸.
> ① **온콜 디버깅** — "어떤 날은 CSV, 어떤 날은 Delta 가 생긴다" 는 문의가 와도 그래프엔 성공한 태스크 하나뿐이라 원인을 못 짚고,
> 결국 잡 코드를 `grep` 해 숨은 `if output_type == ...` 을 찾아야 함 (경로가 안 보이니 성공/실패 로그·알림도 브랜치별로 못 나눔).
> ② **backfill 사고** — 옛 데이터를 고치려 2024-01-01 **이전** 날짜를 재처리하는데 잡 파라미터가 새 포맷 기본값이라,
> 레거시 날짜까지 **새 잡으로 덮어써** 버림 → #38 이 지키려던 "전체 실행 이력 보존" 이 오히려 깨짐.
>
> **권장 —** 분기가 **데이터셋·출력 저장소를 바꾸는 "무거운" 결정** 이면 처리 레이어에 숨기지 말고
> **오케스트레이션 레이어의 branch operator(Example 6-17)** 로 드러내, `new_job`·`legacy_job` 을 **별도 태스크** 로 그래프에 노출할 것.
> 그래야 브랜치별로 **보이고·모니터링되고·개별 재시작** 됨. 조건은 `execution_date`·스키마 필드 수 같은 **메타데이터 기반** 으로 둬 라우터를 가볍게 유지.
> 처리 레이어에 둘 수밖에 없어 **Factory 패턴(Example 6-19)** 을 써도, 그건 `if-else` **코드 중복** 만 정리할 뿐 오케스트레이터엔 여전히 태스크 1개로 보여 **Hidden logic 은 안 풀림**.
>
> **반대 함정 —** 그렇다고 **사소한 분기까지 전부** 오케스트레이션 레이어로 올리면, 이번엔 브랜치가 합류·재분기하며 그래프가 스파게티가 됨(Complexity factory).
> 판단 기준은 **"이 분기를 남이 그래프에서 봐야 하나?"**
> — 출력 저장소·데이터셋을 바꾸면 Yes(오케스트레이션 레이어), 값만 살짝 바꾸면 No(처리 레이어).

---

## 4. 오케스트레이션 (Orchestration)

지금까지는 개별 흐름을 내부·외부 의존성과 함께 **정적으로 조직** 했을 뿐 — 오케스트레이션 레이어 위에 놓인 정적 리소스에 가까웠음.
마지막 패턴 패밀리에서 이들이 **데이터 처리 태스크를 실제로 돌리는 동적 컴포넌트** 가 됨.

핵심 질문 — "선언한 파이프라인을 오케스트레이터가 **어떤 동시성(concurrency)으로 실행하나?**"

- **#39 Single Runner** — 한 파이프라인의 실행이 **항상 하나뿐**(concurrency = 1). 증분·순차 처리에 필수.
- **#40 Concurrent Runner** — **여럿을 병렬 실행**(concurrency > 1). 서로 독립적인 실행을 가속(예: backfill).

```
[6.4 Orchestration — 같은 파이프라인을 몇 개까지 동시에 돌리나]
──────────────────────────────────────────────────────────────────────
 #39 Single Runner      run1 ─► run2 ─► run3        (concurrency=1, 앞이 끝나야 뒤가 시작)
 #40 Concurrent Runner  run1 ┐
                        run2 ┼─ 동시 실행 (concurrency=N)
                        run3 ┘
──────────────────────────────────────────────────────────────────────
 순차성이 결과 정합성의 전제면 Single, 실행끼리 독립이면 Concurrent.
```

---

### 4-1. 패턴 #39: 단일 실행기 (Single Runner)

> 가장 보편적인 오케스트레이션 패턴. 보편적인 대신 **런타임 비용** 이 따름.

#### 상황 (Problem)

**책의 use case** — 세션화 파이프라인을 릴리스에 올리는 경우:

- 최근 프로젝트에서 **Incremental Sessionizer 패턴(챕터 5)** 으로 세션화 파이프라인을 구현.
- POC 였던 터라 오케스트레이션은 범위 밖 — 세션을 비즈니스 오너와 검증하려고 잡을 **수동으로 on-demand** 실행해 옴.
- 이제 프로젝트가 릴리스 사이클에 들어가 데이터 오케스트레이션을 붙여야 함.
- **결정적 제약**: 파이프라인 그래프는 있지만 **실행 방법** 이 없음. 그리고 세션화는 **증분 = 순차** 라 **동시에 둘 이상 돌리면 결과가 틀림**.

#### 해결 (Solution)

runner 가 필요. Single Runner 는 **주어진 파이프라인의 실행이 항상 하나** 임을 보장 → **Single Runner**.

- **구현** — 데이터 흐름의 **동시성 수준(concurrency)을 설정**. Airflow·Azure Data Factory 는 concurrency 속성을 **1** 로 두면 됨.
- **네이티브 지원이 없으면** — **Readiness Marker 패턴(챕터 2)** 으로 이전 실행 완료를 기다리게 구현.
- **작은 함정** — 이 방식은 **미래 실행의 트리거 자체를 막지는 않음**. 모든 후속 실행이 앞선 실행 완료를 기다리며 **줄 서게** 됨.

#### 고려사항 (Consequences)

제한된 동시성은 논리적으로는 타당하지만, 파이프라인의 일상 운영에 중요한 영향을 줌.

- **Backfilling (백필)**
  - backfill 만 안 하면 제한된 동시성은 별문제 아님. 하지만 backfill 하면 순차성 때문에 **재처리가 매우 느림**.
  - 완화가 어려움 — 단일 동시성은 **완화 불가한 비즈니스 요구**(병렬로 돌리면 결과가 틀림).
  - 할 수 있는 건 — 매번 **전체 파이프라인을 backfill 해야 하는지** 재검토.
    예: 처리 단계 + 적재 단계로 구성됐고 **설정만 바꿔 다른 위치에 적재만** 하면 되는 경우, 무거운 데이터 처리 단계를 건너뛸 수 있음.
- **Latency (지연)**
  - backfill 이 최악의 시나리오. 덜 심각한 건 **straggler**(일부 실행이 유독 느림).
    예: 시간별 파이프라인이 전체 **30분** 이면 되던 게 최근 **1.5시간** 으로 늘면 데이터가 점점 지연되고,
    **모든 다운스트림 컨슈머** 까지 그 지연을 떠안음.
  - 확장 가능한 인프라라면 **컴퓨트 추가·처리 로직 개선** 으로 완화.

```
[Single Runner + straggler — 지연이 매시간 누적되는 원리 (스케줄: 매시 정각, concurrency=1)]
──────────────────────────────────────────────────────────────────────
 ✓ 정상 (처리 30분) — 다음 실행 전에 여유 30분, 지연 0
   00:00 ▓▓▓ 00:30끝    01:00 ▓▓▓ 01:30끝    02:00 ▓▓▓ 02:30끝

 ✗ 스트래글러 (처리 1.5시간) — 밀린 실행이 큐에서 대기하며 누적
   00:00 예정·시작 ▓▓▓▓▓▓▓▓▓ 01:30끝
   01:00 예정 → 01:30 시작 ▓▓▓▓▓▓▓▓▓ 03:00끝     (지연 30분)
   02:00 예정 → 03:00 시작 ▓▓▓▓▓▓▓▓▓ 04:30끝     (지연 60분)
   03:00 예정 → 04:30 시작 ▓▓▓▓▓▓▓▓▓ 06:00끝     (지연 90분 → 계속 누적)
──────────────────────────────────────────────────────────────────────
 ⚠ concurrency=1 이라 밀린 실행이 줄 서며 지연이 매시간 +30분씩 벌어짐 → 다운스트림까지 전파.
 ✓ 처리 시간을 스케줄 간격(1h) 밑으로 되돌려야(컴퓨트↑/로직 개선) 누적이 멈춤.
```

#### 구현 예시 (Examples)

**예시 1 — Apache Airflow (Example 6-21)**

`max_active_runs=1` 로 동시 실행을 1로 제한하고, `depends_on_past=True` 로 이전 실행 성공을 전제:
```python
with DAG('visits_trend_generator', max_active_runs=1, default_args={
    'depends_on_past': True,
    # ...
}):
    ...
```

> `max_active_runs=1` — 동시 실행 파이프라인 수를 1로 제한(순차성 보장).
> `depends_on_past=True` — 각 태스크가 **이전 실행의 같은 태스크 성공** 을 전제. 이전 run 의 태스크 A 가 실패하면 이번 run 의 A 는 시작 안 함.

- **클러스터 단위 제어** — AWS EMR 의 **`StepConcurrencyLevel`** 로 한 클러스터에서 병렬 실행할 잡 수를 정함.
- **클라우드 함정** — Airflow 는 활성 run 이 있으면 새 run 을 **아예 스케줄 안 함**.
  반면 **Azure Data Factory 는 만들어서 최대 100개까지 큐에 쌓음** — 큐가 차면 스케줄링이 **429("Too many requests")** 를 반환.

| 도구 | concurrency=1 설정 | 초과 시 동작 |
|---|---|---|
| Apache Airflow | `max_active_runs=1` | 활성 run 있으면 **새 run 스케줄 안 함** |
| Azure Data Factory | `Concurrency=1` | 나머지는 **최대 100개 큐** 에 적재, 초과 시 **429** |
| AWS EMR | `StepConcurrencyLevel` | 클러스터 내 **병렬 잡 수** 를 제한 |

> **트러블 로그** — 증분·순차가 전제인 파이프라인(세션화·상태 누적 등)을 동시성 제한 없이 두면,
> 앞 실행이 늘어질 때 다음 실행이 겹쳐 떠서 **상태가 중복·유실** 됨.
>
> **예 —** Incremental Sessionizer 를 `max_active_runs` 기본값으로 돌리다가 한 배치가 1.5시간으로 늘어난 날,
> 다음 시간 run 이 앞 run 종료 전에 시작돼 **같은 세션 윈도를 두 번 갱신** → 세션 카운트가 부풀려짐.
>
> **권장 —** 결과 정합성이 **순서에 의존** 하면 `max_active_runs=1` + `depends_on_past=True` 로 **단일 실행** 을 강제하고,
> backfill 이 느린 건 "전체 재처리가 정말 필요한지" 를 먼저 따져 적재만 다시 하는 식으로 줄일 것.

---

### 4-2. 패턴 #40: 동시 실행기 (Concurrent Runner)

> Single Runner 의 backfill·지연 문제를 푸는 다음 패턴. **동시성 제약을 완화** 하기만 하면 됨.

#### 상황 (Problem)

**책의 use case** — 외부 소스를 최대한 빨리 적재해야 하는 데이터 수집 팀:

- 데이터 수집 팀. 목표는 외부 at-rest 소스 → 내부 DB 로 **가능한 한 빨리** 적재.
- 수집 주기는 보통 30분~1시간인데 **가끔 전체 과정이 더 오래** 걸림. Single Runner 라서 그 사이 **모든 후속 배송이 지연**.
- **결정적 제약**: 적재하는 데이터셋들이 **서로 독립** — 순차 실행이 꼭 필요한지 의문.

> **용어 — data at rest (정지 데이터)** = 스토리지에 **가만히 저장돼 있는** 데이터. 파일(S3·오브젝트 스토리지), DB 테이블, FTP 덤프, 데이터 레이크 스냅샷 등 **배치로 읽어 가는 정적 소스**.
> 흘러가는 **in motion(Kafka·스트림)**, 처리 중인 **in use(메모리)** 와 구분됨 (보안 용어 "encryption at rest" 와는 다른 맥락).
> 여기서 소스가 at-rest 라는 점이 곧 **Concurrent Runner 의 근거** — 이미 완결된 정적 데이터라 배송끼리 순서에 안 얽혀 **아무 순서로·동시에** 당겨올 수 있음.
> (반대로 소스가 스트림이라 순서·상태가 중요했다면 오히려 #39 Single Runner 가 안전.)

#### 해결 (Solution)

데이터셋이 독립이므로 **순서 무관하게, 완화된 동시성 제약** 으로 수집 가능 → **Concurrent Runner**.

- **구현** — **동시성을 1보다 크게** 정의.
- **"큰 힘엔 큰 책임"** — 인프라 맥락에서 **다른 파이프라인까지 고려한 균형** 이 필요.
- **동작** — 오케스트레이터가 현재 실행 중 인스턴스가 **최대 동시성에 못 미치는 한** 다음 실행을 계속 집어감.

#### 고려사항 (Consequences)

여러 파이프라인을 동시에 돌리는 것은 중요한 영향을 줄 수 있고, 특히 동시성 수준이 지나치게 관대할 때 그러함.

- **Resource starvation (자원 고갈)**
  - **멀티테넌트 환경**(여러 팀이 한 오케스트레이터 공유)에서 특히. 여러 파이프라인이 높은 동시성으로 동시에 backfill 하면
    스케줄러가 **다른 파이프라인을 시작할 용량이 없음**.
  - 최선책 — **workload management** 로 동시성 제어: 사용자 그룹에 특정 컴퓨트 용량을 할당 →
    팀이 동시성을 아무리 높여도 **할당 임계값 이상은 못 씀**.
  - 서버리스 오케스트레이터는 대체로 문제 아님 — 예: AWS Step Functions 는 최대 **10,000개** 병렬 자식 워크플로 허용.
- **Shared state (공유 상태)**
  - 동시 실행을 허용할 때 공유 상태는 흔한 함정. 파이프라인이 **공유 컴포넌트** 에 작업하면, 동시·비결정적 실행이 예기치 못한 side effect 로 문제를 일으킴.
  - 예: **Dynamic Late Data Integrator 패턴(챕터 4)** — 동시 실행이 잘돼야 backfill 을 여러 번 트리거, 최악이면 **일부 실행 날짜에 아예 트리거 안 됨**.

```
[Concurrent Runner + 공유 상태 — 두 run 이 같은 버전을 읽어 생기는 race condition]
──────────────────────────────────────────────────────────────────────
 공유 state 테이블: last_processed_version = 100   (max_active_runs=5 로 run#1·#2 동시 시작)

   run#1  read last=100 ─► v101~v110 처리 ─► write last=110
   run#2  read last=100 ─► v101~v110 처리 ─► write last=110   ← run#1 과 똑같은 구간!
          (run#1 이 쓰기 전에 100 을 읽어 버림)
──────────────────────────────────────────────────────────────────────
 ✗ v101~v110 이 두 번 적재(중복) · v111~ 는 아무도 안 집어 감(누락)
 ⚠ 실행 타이밍에 따라 중복·누락 구간이 매번 달라짐 → 재현·디버깅이 어려움(비결정적)
 ✓ 순서가 정합성의 전제면 Single Runner(=1) 유지. 굳이 병렬이면 파티션별로 상태를 쪼개 공유 자체를 없앰.
```

**쉽게 풀어 보면 (왜 겹쳐 읽으면 중복·누락이 "동시에" 나나)**

- **공유 state 테이블이란** — 증분(incremental) 잡은 "지난번 어디까지 처리했나" 를 **북마크(offset·high-water mark)** 로 `last_processed_version` 같은 값에 저장하고, 다음 run 이 그 지점부터 **그 다음 구간만** 읽음. 이 북마크를 **모든 run 이 공유** 함. (도구별 이름 — AWS Glue `job bookmark` · Kafka `offset` · Spark `checkpoint`. 스트리밍의 `watermark` 는 이벤트 시간 경계라 초점이 다름.)
- **버그의 정체 = read-modify-write race (lost update)** — 처리 순서가 "① 읽고 → ② 처리하고 → ③ 갱신" 인데, **락 없이** 동시에 돌면 run#1 이 ③을 쓰기도 전에 run#2 가 ①에서 **같은 값(100)** 을 읽어 버림. 두 run 이 같은 출발점을 보는 순간 사고 확정.
- **① 중복 (duplication)** — 둘 다 `last=100` 을 읽었으니 **똑같이 v101~v110 을 "다음 구간" 으로 판단** → 같은 데이터를 두 번 적재. write 가 멱등하지 않으면 다운스트림에 **중복 행·이중 집계** 로 번짐.
- **② 누락 (omission)** — 원래 5개 run(`max_active_runs=5`)이 **나눠서** v101~v150 을 커버해야 했는데, 다 같은 100 을 읽어 **앞 구간만 겹쳐** 처리하고 각자 북마크를 110 근처로만 밀어 놓음 → **v111~ 이후는 이번 라운드에서 아무 run 에도 안 잡힘**. 예약된 후속 run 이 없으면 그대로 방치됨.
- **왜 "비결정적" 인가** — 중복·누락 구간이 **누가 언제 읽고 쓰느냐(스케줄러 타이밍·처리 속도)** 에 좌우됨. 같은 코드·같은 입력인데도 재현이 안 돼, 로그만 봐선 원인 추적이 지독히 어려움.
<!-- - **챕터 4 Dynamic Late Data Integrator 와의 연결** — 거기선 "지연 데이터 감지 → 다운스트림 backfill 트리거 → 북마크 전진" 순서인데, run#2 가 트리거를 쏘기도 전에 run#1 이 북마크를 밀면 run#2 는 **"이미 처리됨" 으로 착각** 해 그 버전의 backfill 을 영영 안 쏨. -->

**어떻게 막나 — 세 갈래.**

- **① 그냥 Single Runner(=1)** — 순서가 정합성의 전제면 **동시성을 포기** 하는 게 가장 단순·안전 (#39). backfill 이 느린 건 감수.
- **② 상태를 쪼개 공유 자체를 제거** — run 마다 **독립된 북마크**(파티션·날짜·소스별 `last_processed_version`)를 두면 서로 밟을 일이 없어 병렬 OK. #40 의 "데이터셋이 서로 독립" 전제와 맞물림.
- **③ 원자적 read-modify-write** — 하나의 상태를 꼭 공유해야 하면 **락 / 트랜잭션 / CAS(compare-and-set)** 로 "읽고-갱신" 을 **한 단위** 로 묶어 lost update 를 원천 차단.

```
[✓ 고친 흐름 — 상태를 쪼개거나(②) 원자적으로 갱신(③)]
──────────────────────────────────────────────────────────────────────────────────────────────────────
 ② 상태 분리    run#1: partitionA 북마크     run#2: partitionB 북마크   ← 공유 안 함, 밟을 일 없음
 ③ 원자적 갱신  run#1  [락] read 100 → write 110 [해제]
              run#2               [대기]………………………………[락] read 110 → write 120 [해제]   ← 항상 최신 값을 읽음
──────────────────────────────────────────────────────────────────────────────────────────────────────
 공유를 없애거나(②) · 공유하되 read-modify-write 를 한 단위로 묶으면(③) ⇒ 같은 100 을 두 번 읽는 일이 사라짐.
```

#### 구현 예시 (Examples)

**예시 1 — Apache Airflow (Example 6-22)**

`max_active_runs=5` 로 동시 실행을 허용하고, 독립적이므로 `depends_on_past=False`:
```python
with DAG('devices_loader', max_active_runs=5, default_args={
    'depends_on_past': False,
    # ...
}):
    ...
```

> `max_active_runs=5` — 최대 5개 run 동시 실행. `depends_on_past=False` — 이전 실행에 의존하지 않음(독립 데이터셋이라 가능).
> 특정 태스크만 이전 실행에 의존시키려면 `True` 로 바꿀 수 있고, 이는 **동시성 설정과 무관** 하지만 이전 실행이 실패한 태스크에서 파이프라인이 멈추게 할 수 있음.
> 이 유연성은 모든 오케스트레이터에 있는 건 아님 — Azure Data Factory 는 동시성과 **트리거 기반 의존** 은 되지만 **태스크 기반 의존** 은 안 됨.

| 패턴 | concurrency | 언제 | 핵심 위험 |
|---|---|---|---|
| #39 Single Runner | 1 | 증분·순차(세션화, 상태 누적) | backfill 느림·지연이 다운스트림까지 전파 |
| #40 Concurrent Runner | >1 | 실행끼리 **독립**(무관한 데이터셋 수집) | 자원 고갈(멀티테넌트)·공유 상태 side effect |

> **트러블 로그** — 독립이라 믿고 Concurrent Runner 로 동시성을 높였는데, 실은 파이프라인이 **공유 상태(state 테이블 등)** 를 건드리면
> 동시 실행이 서로의 결과를 덮어써 데이터가 조용히 틀어짐.
>
> **예 —** 챕터 4 Dynamic Late Data Integrator 처럼 `last_processed_version` 을 공유하는 잡을 `max_active_runs=5` 로 돌리면,
> 두 run 이 같은 버전을 읽고 각자 갱신해 어떤 날짜는 backfill 이 **중복 트리거**, 어떤 날짜는 **누락** 됨.
>
> **권장 —** 동시성을 1보다 올리기 전에 파이프라인이 **공유 컴포넌트에 쓰는지** 먼저 확인하고,
> 순차성이 결과 정합성의 전제라면 **Single Runner(concurrency = 1)** 를 유지할 것. 멀티테넌트면 **workload management** 로 팀별 용량을 캡.

---

## 5. 요약

챕터 6의 데이터 흐름 패턴은 **"만든 데이터 가치를 어떻게 잇고(sequence)·합치고(fan-in)·나누고(fan-out)·돌리나(orchestration)"** 를 다룸.
#33(Local Sequencer)이 **한 파이프라인 안의 순서** 였다면, #34~#38 은 **파이프라인·팀 경계를 넘는 흐름** 을,
#39~#40 은 그 흐름을 **어떤 동시성으로 실행할지** 를 조율함.

| 패턴 | 카테고리 | 한 줄 요약 | 핵심 트레이드오프 |
|---|---|---|---|
| #33 Local Sequencer | Sequence | 거대 잡을 순서 있는 작은 태스크로 분해(같은 파이프라인 안) | 재시작·backfill 경계 설정 / 격리된 단일 잡엔 오케스트레이터가 과함 |
| #34 Isolated Sequencer | Sequence | 조직·팀 경계로 분리된 두 파이프라인을 데이터/태스크 트리거로 연결 | 데이터 기반=느슨(진화 자유) / 태스크 기반=강결합(이름 변경에 취약)·팀 간 소통 의존 |
| #35 Aligned Fan-In | Fan-In | 여러 부모(시간별 부분 집계)가 **모두 성공** 해야 자식(일별 최종) 실행 | backfill·피드백 루프 유리 / 인프라 스파이크·스케줄링 스큐 |
| #36 Unaligned Fan-In | Fan-In | 일부 부모가 실패해도 자식 실행 — **부분 데이터** 허용 또는 실패 태스크 트리거 | 가독성 저하·완전성(completeness) 공유 필수 |
| #37 Parallel Split | Fan-Out | 한 부모 → 둘 이상 자식 **병렬 실행**(같은 입력, 다른 처리) | 공통 계산 1회 물질화·재시도 이중 쓰기 방지 / 가장 느린 브랜치가 전체를 막음·HW 요구 불일치 |
| #38 Exclusive Choice | Fan-Out | 조건 평가 후 여러 자식 중 **하나만** 실행(if-else/branch) | 분기 남발 시 가독성 급락·처리 레이어의 숨은 로직 / 메타데이터 조건 선호 |
| #39 Single Runner | Orchestration | 파이프라인 실행이 **항상 하나**(concurrency = 1) | 증분·순차엔 필수 / backfill 느림·지연이 다운스트림까지 전파 |
| #40 Concurrent Runner | Orchestration | 여럿 **병렬 실행**(concurrency > 1) | 독립 실행 가속(backfill) / 자원 고갈(멀티테넌트)·공유 상태 side effect |

```
[#33 ~ #40 한눈에 — 시퀀스 → 팬인 → 팬아웃 → 오케스트레이션]
──────────────────────────────────────────────────────────────────────
 Sequence   task A ─► task B ─► task C                       #33 Local Sequencer
            provider ──(데이터/태스크 트리거)──► consumer         #34 Isolated Sequencer

 Fan-In     h1·h2·…·h24 ──► (merge) ──► final                #35 Aligned  (모두 성공)
                        └─► partial(+실패 갈래)                #36 Unaligned(일부 성공)

 Fan-Out    parent ──┬─► child A (병렬 전부)                   #37 Parallel Split
                     └─► child B
            parent ─► 조건? ─┬─► child A (택일)                #38 Exclusive Choice
                            └─► child B

 Orchestr.  run1 ─► run2 ─► run3       (concurrency=1)       #39 Single Runner
            run1·run2·run3 동시 실행   (concurrency=N)         #40 Concurrent Runner
──────────────────────────────────────────────────────────────────────
 공통 원칙 — 경계(boundary)는 "재시작 단위·backfill 단위" 로 잡고, 팀 경계를 넘을수록 소통·완전성 공유가 핵심.
```

**정리** — 세 카테고리는 각각 "무엇을 고르나" 와 "그 대가·관건" 으로 요약됨.

- **팬인 (Fan-In)** — **"모두 기다릴지(Aligned)"** vs **"부분이라도 갈지(Unaligned)"**
  - 트레이드오프 — 각각 **지연(스큐)** / **완전성 관리 부담**
- **팬아웃 (Fan-Out)** — **"모두 병렬로(Parallel Split)"** vs **"하나만 택일(Exclusive Choice)"**
  - 트레이드오프 — 각각 **공통 계산 1회·중복 쓰기 방지** / **분기를 오케스트레이션 레이어로 드러내기**
- **오케스트레이션 (Orchestration)** — **동시성 1(Single)** 로 순서 지키기 vs **동시성 N(Concurrent)** 으로 처리량 올리기
  - 트레이드오프 — 실행끼리 **독립인지** / **공유 상태를 건드리는지**

다음 챕터(7 데이터 보안)에서는 이렇게 만든 데이터셋을 **어떻게 지우고·접근 제어하고·보호하나** 로 넘어감.
