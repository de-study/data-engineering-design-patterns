# 데이터 엔지니어링 디자인 패턴 - 데이터 보안 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 7 | 실무 데이터 엔지니어링 관점 정리

> **도식 기호 범례** — `⚠` 위험·함정(조심해야 할 동작) · `✗` 잘못된 결과(깨진 상태) · `✓` 올바른 결과 · `⇒` 그 결과 도출

---

## 목차

1. [데이터 제거 (Data Removal)](#1-데이터-제거-data-removal)
   - 패턴 #41: 수직 파티셔너 (Vertical Partitioner)
2. [요약](#2-요약)

> 본 문서가 다루는 범위는 챕터 7 중 **#41 (7.1 Data Removal 의 Vertical Partitioner)**.
> 나머지(#42 In-Place Overwriter, 7.2 Access Control · 7.3 Data Protection · 7.4 Connectivity)는 후속 문서에서 다룸.
> ※ 이름이 같은 챕터 8 의 **#51 Vertical Partitioner(스토리지 최적화용)** 와 혼동 주의 — 본 #41 은 **개인정보 삭제(잊힐 권리)** 를 겨냥한 보안 패턴.

---

## 책의 use case (챕터 도입)

> 데이터 가치·데이터 흐름 패턴으로 만든 **접근하기 쉽고 가치 있는 데이터셋** 은 중요한 비즈니스 자산이자,
> **악의적 행위자를 포함한** 다른 시장 참여자들의 시샘 대상이기도 함. 그래서 데이터 엔지니어링은 처리 잡을 짜는 데서 멈출 수 없음.

- **Compliance (규정 준수)** — 최근 몇 년간 큰 주목을 받은 영역. **GDPR(유럽)**·**CCPA(미국)** 같은 데이터 프라이버시 법이 **나(데이터 프로바이더)와 고객(데이터 컨슈머)** 사이의 경계를 정밀하게 규정.
- **Access control (접근 제어)** — 조직 안에서 데이터셋이 열려 있으면 **다른 팀이 실수로 덮어쓸** 수 있고, 그 여파가 나와 다운스트림 컨슈머 모두에게 큼.
- **Data protection (데이터 보호)** — 데이터셋 위치 접근권을 실수로 줘도, **암호화** 같은 보호 계층을 더해 뒀다면 컨슈머는 **복호화 키** 가 있어야 데이터를 읽을 수 있음.
- **Connectivity (연결성)** — 사람이든 애플리케이션이든 데이터 저장소에 연결해 읽고 씀.
  **자격 증명(credentials)을 Git 저장소에 두면** 유출 위험이 커짐 → 외부의 더 안전한 곳에 보관해야 함.

```
[챕터 7 데이터 보안 패턴 — 네 카테고리]
──────────────────────────────────────────────────────────────────────
 7.1 Data Removal     잊힐 권리 대응(개인정보 삭제)      #41 Vertical Partitioner / #42 In-Place Overwriter
 7.2 Access Control   세밀한 접근 제어                #43 Fine-Grained Accessor (Tables/Resources #44)
 7.3 Data Protection  암호화·익명화                   #45 Encryptor / #46 Anonymizer / #47 Pseudo-Anonymizer
 7.4 Connectivity     자격 증명 없이 안전하게 연결       #48 Secrets Pointer / #49 Secretless Connector
──────────────────────────────────────────────────────────────────────
 본 문서: 7.1 Data Removal 의 Vertical Partitioner(#41) 만 다룸.
```

### 패턴 흐름 — 챕터 6에서 챕터 7 으로

데이터를 잇고 나눠 **가치 있는 자산** 을 만들었으니(챕터 6), 이제 그 자산을 **어떻게 지키나** 가 데이터 보안 패턴의 출발점.
그 첫 카테고리가 **데이터 제거(Data Removal)** — 사용자가 삭제를 요청하면 그의 데이터를 지워야 하는 **잊힐 권리** 대응.

```
[패턴 흐름 — 챕터 6에서 챕터 7 으로]
──────────────────────────────────────────────────────────────────────
 챕터 6: 데이터 흐름으로 가치 있는 데이터셋을 만들고 팀 간 공유
      │ 새 과제: 이 자산을 규정에 맞게 "지켜야" 함 (GDPR·CCPA)
      ▼ "사용자가 '내 데이터 지워줘' 라고 하면 어떻게 지우지?"
 7.1 Data Removal (개인정보 삭제)
   #41 Vertical Partitioner — 불변 개인정보를 "딱 한 번만" 저장하도록 컬럼 단위로 분리 → 지울 대상 자체를 줄임
      │ (다음) 리팩터링할 시간·자원이 없는 레거시라면?
      ▼
   #42 In-Place Overwriter — 기존 저장소를 제자리에서 덮어써 삭제 (본 문서 범위 밖)
──────────────────────────────────────────────────────────────────────
 핵심 — 데이터 제거는 "어떻게 지우나" 보다 "지울 데이터를 어떻게 적게 만드나(분리)" 가 먼저.
```

---

## 1. 데이터 제거 (Data Removal)

CCPA·GDPR 같은 프라이버시 규정은 여러 준수 요구사항을 정의하는데, 그중 하나가 **개인정보 삭제 요청(personal data removal request)**
— 사용자로부터 삭제 요청을 받으면 그 사용자의 데이터를 **지워야** 함. 이 절은 그 요구를 다루는 **두 구현 접근** 을 소개.

- **#41 Vertical Partitioner** — 데이터 조직을 **컬럼 단위로 분리** 해 삭제할 개인정보를 최소화 (아래 1-1 절).
- **#42 In-Place Overwriter** — 리팩터링 여유가 없을 때 **기존 저장소를 제자리에서 덮어써** 삭제 (본 문서 범위 밖).

---

### 1-1. 패턴 #41: 수직 파티셔너 (Vertical Partitioner)

> 첫 데이터 제거 패턴 — **똑똑한 데이터 조직(또는 워크플로 격리)** 이 까다로운 문제도 풀어준다는 원칙의 사례.
> 지울 데이터를 나중에 열심히 찾는 대신, **애초에 지울 대상이 적게** 데이터를 배치함.

#### 상황 (Problem)

**책의 use case** — 개인정보 삭제 파이프라인 설계 리뷰에서 지적된 스토리지 오버헤드:

- 새 **개인정보 삭제 파이프라인** 의 첫 설계 문서를 발표. 동료들 반응은 좋았으나 **중요한 스토리지 오버헤드** 를 지적받음.
- 데이터셋의 **여러 컬럼이 불변(immutable)** — 절대 바뀌지 않는데도 **매 레코드마다 반복 저장** 됨.
  예: **생일(birthday)**, **개인 식별 번호(personal ID number)**.
- 동료들의 피드백 — 각 불변 속성을 **딱 한 번만 저장** 하라. 그러면 삭제 요청이 왔을 때 **지울 데이터가 줄어듦**.
- **결정적 제약**: 잊힐 권리(개인정보 삭제)에 대응해야 하는데, 불변 개인정보가 **모든 레코드에 중복** 저장돼 저장·삭제 비용이 과다.

#### 해결 (Solution)

데이터셋을 **가변(mutable) 부분** 과 **불변(immutable) 부분** 두 갈래로 나눔 → **Vertical Partitioner**.

- **수평 vs 수직 파티셔닝** — 파티셔닝에는 두 방향이 있음.
  - **수평(horizontal)** — **관련 속성을 가진 레코드들을 함께** 둠. 대표 예가 배치 증분 처리에 흔한 **날짜 기반 파티션**. (챕터 8에서 다룸)
  - **수직(vertical)** — **각 row 를 쪼개** parts 를 서로 다른 곳에 씀. 이 **속성 기반 분할** 이 효율적인 개인정보 삭제 파이프라인을 가능케 함.
- **분할 절차** — 두 단계.
  - ① **쪼갤 컬럼** 과, 나중에 쪼갠 row 들을 **다시 합칠 때 쓸 속성**(예: `user_id`)을 식별.
  - ② 데이터 수집 잡에 **속성 기반 split 로직** 을 추가 — 한 row 를 두 부분으로 나눠 **각각 별도 저장소** 에 씀.
    매 레코드마다 값이 달라지는 **event 속성** 은 한 저장소로, 안 바뀌는 것·**개인정보 범위** 에 속하는 것은 다른 곳으로.
- **가장 쉬운 구현** — `SELECT` 문(또는 데이터 처리 프레임워크의 attribute projection). 
  **여러 쿼리** 를 날려 각각 **다른 컬럼 집합** 을 타깃으로 잡아 **다른 데이터스토어** 에 씀. 각 쿼리에 **중복 제거(dedup)** 같은 비즈니스 규칙을 붙일 수 있음.

```
[Figure 7-1 재현] Vertical Partitioner — PII·불변 속성을 전용 저장소로 분리
──────────────────────────────────────────────────────────────────────
                                 event_id: 1029384
                                 user_id: 10
 Input          Vertical         address: 12, Dummy Street
 location  ───► partitioner ───► city: Neverland
                                 action: click
                                 visited_page: home.html
                                       │  vertical partitioning
                     ┌─────────────────┴─────────────────┐
                     ▼                                   ▼
           event_id: 1029384                     user_id: 10
           user_id: 10                           address: 12, Dummy Street
           action: click                         city: Neverland
           visited_page: home.html
                     │                                   │
                     ▼                                   ▼
              [ Event data ]                       [ PII data ]
──────────────────────────────────────────────────────────────────────
 매 레코드마다 값이 바뀌는 event 속성 → Event data, 안 바뀌는 PII·불변 속성 → PII data.
 user_id 는 양쪽에 남겨, 컨슈머가 나중에 두 조각을 다시 join 하는 키로 씀.
```

**쉽게 풀어 보면 (방문 로그 예)** — 블로그 방문 이벤트 한 줄에 **매번 바뀌는 것** 과 **절대 안 바뀌는 개인정보** 가 섞여 있음.

- **한 줄에 섞인 두 종류** — `action`·`visited_page` 는 클릭마다 달라지지만, `address`·`city`(개인정보)는 그 사용자에 대해 항상 같음.
- **수직 분할** — 이 한 줄을 **Event data**(가변)와 **PII data**(불변) 두 저장소로 쪼개고, 양쪽에 `user_id` 만 남겨 나중에 이어붙일 키로 씀.
- **삭제가 쉬워지는 이유** — 사용자가 삭제를 요청하면, 매 레코드에 흩어진 개인정보를 다 뒤질 필요 없이
  **PII data 의 그 `user_id` 한 행만** 지우면 됨.

```
[왜 지울 게 줄어드나 — user_id=10, 방문 1,000건 예]
──────────────────────────────────────────────────────────────────────
 ✗ 비파티션 (한 테이블에 다)
   visit_id | user_id | address        | city      | action | visited_page
   1        | 10      | 12, Dummy St   | Neverland | click  | home.html
   2        | 10      | 12, Dummy St   | Neverland | scroll | post.html
   ...      | 10      | 12, Dummy St   | Neverland | ...    | ...        ← address·city 를 1,000벌 중복 저장
   ⇒ 삭제 요청 시 user_id=10 의 1,000행을 스캔·수정

 ✓ 수직 분할 (두 저장소)
   Event data (1,000행, 개인정보 없음)         PII data (user_id 당 1행)
   visit_id | user_id | action | page      user_id | address       | city
   1        | 10      | click  | home      10      | 12, Dummy St  | Neverland   ← 개인정보 1벌만
   2        | 10      | scroll | post
   ...                                     ⇒ 삭제 요청 = PII data 의 1행 delete
──────────────────────────────────────────────────────────────────────
 방문이 많은 사용자일수록 절감 폭이 큼 — 중복 저장되던 개인정보 벌수만큼 그대로 이득.
```

#### 고려사항 (Consequences)

삭제 use case 에는 좋은 성능 최적화지만, **컨슈머 (조회하는 쪽)** 에겐 몇 가지 단점이 따름.

- **Query performance (조회 성능)**
  - 수직 파티셔닝은 일종의 **데이터 정규화(normalization)** — 불변 속성이 가변 속성과 **따로 떨어져** 삶.
    **쓰기(write)** 는 볼륨을 줄여 최적화되지만, **읽기(read)** 는 저하됨 — reader 가 **쪼개진 row 들을 join** 해야 하기 때문.
  - 읽기 연산이 **네트워크 트래픽** 을 수반. 비파티션 버전이라면 같은 row 안에서 **로컬로** 끝났을 읽기가, 이제는 두 저장소를 오가야 함.
- **Querying complexity (조회 복잡도)**
  - 데이터 분리는 쿼리에 **추가 복잡도** 를 가져옴 — 컨슈머는 일부 속성이 **다른 곳에 있음** 을 알아야 함.
  - 완화가 어렵지 않음 — **단일 진입점(view 등)** 으로 노출하거나, **데이터 문서화**(예: data catalog)를 제공하거나,
    **데이터 리니지**(챕터 10 **Dataset Tracker 패턴**)로 분리 구조를 명확히 드러내면 됨.
- **Complexity in a polyglot world (폴리글랏 환경의 복잡도)**
  - 한 데이터셋이 **서로 다른 종류의 저장소**(예: NoSQL DB + 관계형 DB)에 **동시에** 사는 경우.
    **폴리글랏 지속성(polyglot persistence)** 은 reader 에게 좋음 — 각 컨슈머에 **적합한 기술** 로 레코드를 노출.
    예: 검색 기능은 **검색 최적화 DB**, 저지연 마이크로서비스는 **key-value store** 에서 같은 레코드를 원함.
  - 이 경우 **여러 저장소 시스템 전반** 에 수직 파티셔닝을 적용해야 할 수 있음(Figure 7-2).
- **Raw data (원본 데이터)**
  - **원본(분할 전) 데이터** 를 일정 기간 보관해야 하면, 삭제를 위한 **보완책** 이 따로 필요.
    Vertical Partitioner 는 **첫 변환 단계부터만** 적용되기 때문.
  - 쉬운 해법은 분할 전 데이터에 **짧은 보존 기간(retention)** 을 두는 것(삭제 요청 지연 한도를 지키는 선에서).
    단 이는 **backfill 용 데이터 가용성** 을 낮춤.

```
[Figure 7-2 재현] Polyglot persistence + vertical partitioning
──────────────────────────────────────────────────────────────────────
                          ┌─► Immutable personal data ─► RDBMS writer         ─► [ Mutable | Immutable ]
 Input raw ─► Vertical ───┤
      data    partitioner └─► Mutable data            ─► Search engine writer ─► [ Search engine ]
──────────────────────────────────────────────────────────────────────
 같은 레코드를 컨슈머 특성에 맞는 다른 저장소로 — 불변 PII 는 RDBMS, 가변 데이터는 검색엔진.
 잡이 각 row 를 쪼개면, 전용 컨슈머가 자기 저장소 계층에 맞게 처리.
```

| 파티셔닝 | 무엇을 나누나 | 대표 예 | 이 패턴에서의 쓰임 |
|---|---|---|---|
| **수평(horizontal)** | **행(row)** 을 그룹으로 | 날짜 기반 파티션(배치 증분) | 챕터 8 저장 패턴 |
| **수직(vertical)** | **열(column)** 을 저장소별로 | 가변 event ↔ 불변 PII 분리 | 삭제 대상(개인정보)을 한 곳에 모아 최소화 |

#### 구현 예시 (Examples)

**예시 1 — Spark `foreachBatch` 로 수직 분할 (Example 7-1)**

들어오는 레코드를 두 데이터셋으로 쪼개 **각각 다른 Kafka 토픽** 에 씀. 공통 입력이므로 `persist()` 로 한 번만 읽음:
```python
def split_visit_attributes(visits_to_save: DataFrame, batch_number: int):
    visits_to_save.persist()

    # 브랜치 1 — 방문(event) 데이터에서 user 컨텍스트를 제거
    visits_without_user_context = (visits_to_save
        .filter('user_id IS NOT NULL AND context.user.login IS NOT NULL')
        .withColumn('context', F.col('context').dropFields('user'))   # 개인정보 필드 제거
        .select(F.col('visit_id').alias('key'), F.to_json(F.struct('*')).alias('value')))
    # save to visits_without_user_context (Event data 토픽)

    # 브랜치 2 — user 속성만 뽑아 user_context 로 (불변 PII)
    user_context_to_save = (visits_to_save.selectExpr('context.user.*', 'user_id')
        .select(F.col('user_id').alias('key'), F.to_json(F.struct('*')).alias('value')))
    # save to user_context_to_save (PII 토픽)

    visits_to_save.unpersist()
```

> 단순한 **컬럼 기반 변환** 으로 방문 데이터에서 user 정보를 떼어내고, user 속성만 별도 토픽으로 보냄.
> `visit_id` 를 key 로 쓰는 방문 토픽과 달리, user 토픽은 `user_id` 를 key 로 써서 **사용자당 한 레코드** 로 정리됨.

**예시 2 — Delta Lake `MERGE` 로 dedup (Example 7-2)**

분할한 user_context 토픽을 Delta 테이블로 옮기며 `MERGE` 로 **user_id 당 최신 한 건만** 유지:
```python
def save_most_recent_user_context(context_to_save: DataFrame, batch_number: int):
    deduplicated_context = context_to_save.dropDuplicates(['user_id']).alias('new')
    current_table = DeltaTable.forPath(spark_session, get_delta_users_table_dir())
    (current_table.alias('current')
        .merge(deduplicated_context, 'current.user_id = new.user_id')
        .whenMatchedUpdateAll().whenNotMatchedInsertAll()
        .execute())
```

> 불변 속성을 **딱 한 번만** 저장하려는 목표를 MERGE 로 달성 — 같은 `user_id` 는 갱신, 없으면 삽입.

**예시 3 — Delta Lake 삭제 (Example 7-3)**

이렇게 분리해 두면 삭제는 **PII 테이블의 한 행** 을 지우는 것으로 끝:
```python
user_id_to_delete = '140665101097856_0316986e-9e7c-448f-9aac-5727dde96537'
users_table = DeltaTable.forPath(spark_session, get_delta_users_table_dir())
users_table.delete(f'user_id = "{user_id_to_delete}"')
```

> `delete` 만으로는 부족 — **`VACUUM`** 을 추가로 돌려 보존 기간을 넘긴 파일까지 물리 삭제해야 함.
> 안 그러면 **이전 버전(older version)을 읽어** 삭제된 사용자의 데이터를 여전히 복구할 수 있음(Delta 의 time travel).

**예시 4 — Kafka tombstone 메시지 (Example 7-4)**

Kafka 에서는 삭제를 **tombstone(묘비) 메시지** — 삭제할 레코드의 **key + `null` 값** — 로 표현.
`cleanup.policy=compact` 토픽에 보내면 background compaction 이 해당 tombstone 을 지움:
```bash
docker exec -ti ... kafka-console-producer.sh --bootstrap-server .... \
  --topic ... --property parse.key=true --property key.separator=, \
  --property null.marker=NULL

140665101097856_0316986e-9e7c-448f-9aac-5727dde96537,NULL
```

> compaction 후 그 `user_id` 는 토픽에서 사라짐. 단 이 방식은 **key 당 한 개** 인 `user_context` 토픽에서만 유효
> — `visits` 처럼 **여러 이벤트가 같은 visit id key 를 공유** 하는 토픽에 compaction 을 걸면 **마지막 방문만 남아** 데이터가 훼손됨.

```
[Examples 7-1 ~ 7-4 한 흐름 — 분리부터 삭제까지]
──────────────────────────────────────────────────────────────────────
 raw visits ─► split_visit_attributes (7-1, foreachBatch)
                  ├─► visits 토픽         (key=visit_id · 가변 event, 개인정보 제거)
                  └─► user_context 토픽   (key=user_id · 불변 PII)
                             │
                             ▼  MERGE + dropDuplicates(user_id)  (7-2)
                     [ Delta users 테이블 ]   ← user_id 당 1건으로 정리
                             │
        삭제 요청 ──────────►  │  delete(user_id = ...)  (7-3)  ─► + VACUUM (물리 삭제)
                             │
   (Kafka 저장이면)  tombstone  user_id,NULL  ─► compaction 이 제거  (7-4)
──────────────────────────────────────────────────────────────────────
 분리(7-1) → 중복 제거(7-2) → 논리 삭제(7-3) → 물리 정리(VACUUM 또는 compaction) 4단계.
 ⚠ VACUUM/compaction 을 빠뜨리면 이전 버전으로 복구 가능 → 삭제가 "완료" 되지 않음.
```

> **트러블 로그** — Vertical Partitioner 로 개인정보를 분리해 놓고 **원본(분할 전) 데이터의 보존·삭제** 를 잊으면,
> 정작 삭제 요청을 받은 사용자의 데이터가 **원본 레이어에 그대로 남아** 규정 위반이 됨.
>
> **예 —** `visits_raw` 원본을 backfill 용으로 90일 보관하면서 PII 테이블에서만 `user_id` 를 지우면,
> 삭제 요청 뒤에도 원본 JSON 에 `address`·`city` 가 남아 GDPR/CCPA 삭제 의무를 못 지킴.
> 또 Delta 에서 `delete` 만 하고 `VACUUM` 을 빼면 time travel 로 이전 버전을 읽어 복구까지 가능.
>
> **권장 —** 수직 분할은 **첫 변환 단계부터만** 적용됨을 인지하고, 원본 데이터에는 **삭제 지연 한도 내의 짧은 retention** 을 걸 것.
> Delta 는 `delete` 후 반드시 **`VACUUM`** 으로 물리 삭제, Kafka 는 **key 당 1건 토픽에만** compaction 기반 tombstone 을 쓸 것.

---

## 2. 요약

챕터 7의 데이터 보안 패턴은 **"만든 데이터 자산을 어떻게 지키나"** 를 다루며, 그 첫 카테고리가 **데이터 제거(잊힐 권리)**.
#41 **Vertical Partitioner** 는 삭제를 "어떻게 지우나" 가 아니라 **"지울 대상을 어떻게 적게 만드나"** 로 뒤집는 패턴
— 불변 개인정보를 **컬럼 단위로 떼어 딱 한 번만** 저장해, 삭제 요청 시 **PII 저장소의 한 행** 만 지우면 되게 함.

| 패턴 | 카테고리 | 한 줄 요약 | 핵심 트레이드오프 |
|---|---|---|---|
| #41 Vertical Partitioner | Data Removal | 가변 event ↔ 불변 PII 를 컬럼 단위로 분리해 삭제 대상 최소화 | 쓰기·삭제 비용↓ / <br>읽기 시 join · 네트워크 비용↑ · 조회 복잡도 (원본 데이터 보존 별도 관리) |

```
[#41 한눈에 — 지울 데이터를 애초에 줄인다]
──────────────────────────────────────────────────────────────────────
 ✗ 비파티션        레코드마다 address·city 반복 저장 → 삭제 시 전 레코드를 뒤져야 함
 ✓ Vertical 분할   PII 는 user_id 당 1건(PII data) + 가변 event 는 따로(Event data)
                   ⇒ 삭제 요청 = PII data 의 그 user_id 한 행 delete (+ VACUUM / tombstone)
──────────────────────────────────────────────────────────────────────
 대가 — 조회 시 두 조각을 user_id 로 다시 join(네트워크·복잡도). view ·data catalog·리니지로 완화.
```

**정리** — Vertical Partitioner 는 **삭제 최적화** 를 위한 **쓰기 시점의 데이터 조직** 선택이고,
그 대가는 **조회 시 join 비용과 복잡도**. 리팩터링·마이그레이션 여유가 없는 레거시라면
다음 패턴 **#42 In-Place Overwriter**(기존 저장소를 제자리에서 덮어써 삭제)로 넘어가며,
이어 **접근 제어(#43·#44) · 데이터 보호(#45~#47) · 연결성(#48·#49)** 이 데이터 보안을 완성함.
