# 데이터 엔지니어링 디자인 패턴 - 데이터 보안 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 7 | 실무 데이터 엔지니어링 관점 정리

> **도식 기호 범례** — `⚠` 위험·함정(조심해야 할 동작) · `✗` 잘못된 결과(깨진 상태) · `✓` 올바른 결과 · `⇒` 그 결과 도출

---

## 목차

1. [데이터 제거 (Data Removal)](#1-데이터-제거-data-removal)
   - 패턴 #41: 수직 파티셔너 (Vertical Partitioner)
   - 패턴 #42: 제자리 덮어쓰기 (In-Place Overwriter)
2. [접근 제어 (Access Control)](#2-접근-제어-access-control)
   - 패턴 #43: 테이블 세분화 접근자 (Fine-Grained Accessor for Tables)
   - 패턴 #44: 리소스 세분화 접근자 (Fine-Grained Accessor for Resources)
3. [데이터 보호 (Data Protection)](#3-데이터-보호-data-protection)
   - 패턴 #45: 암호화기 (Encryptor)
4. [연결성 (Connectivity)](#4-연결성-connectivity) — 범위 밖(후속 문서)
5. [요약](#5-요약)

> 본 문서가 다루는 범위는 챕터 7 중 **#41·#42(7.1 Data Removal), #43·#44(7.2 Access Control), #45(7.3 Data Protection)**.
> 아직 안 다룬 패턴 — **#46 Anonymizer · #47 Pseudo-Anonymizer(7.3) · #48 Secrets Pointer · #49 Secretless Connector(7.4 Connectivity)** 는 후속 문서 몫(각 자리에 "범위 밖" 표시만 남김).
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
 본 문서: 7.1 Data Removal(#41·#42) · 7.2 Access Control(#43·#44) · 7.3 의 #45 Encryptor 까지. #46~#49 는 후속 문서.
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
   #42 In-Place Overwriter — 기존 저장소를 제자리에서 덮어써 삭제 (staging→승격, 블록 회수 필수)
──────────────────────────────────────────────────────────────────────
 핵심 — 데이터 제거는 "어떻게 지우나" 보다 "지울 데이터를 어떻게 적게 만드나(분리)" 가 먼저.
```

---

## 1. 데이터 제거 (Data Removal)

CCPA·GDPR 같은 프라이버시 규정은 여러 준수 요구사항을 정의하는데, 그중 하나가 **개인정보 삭제 요청(personal data removal request)**
— 사용자로부터 삭제 요청을 받으면 그 사용자의 데이터를 **지워야** 함. 이 절은 그 요구를 다루는 **두 구현 접근** 을 소개.

- **#41 Vertical Partitioner** — 데이터 조직을 **컬럼 단위로 분리** 해 삭제할 개인정보를 최소화 (아래 1-1 절).
- **#42 In-Place Overwriter** — 리팩터링 여유가 없을 때 **기존 저장소를 제자리에서 덮어써** 삭제 (아래 1-2 절).

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

<details>
<summary><b>⚠ 트러블 로그</b> — Vertical Partitioner로 개인정보만 분리하고 원본(분할 전) 데이터의 보존·삭제를 잊으면 삭제 요청 사용자의 데이터가 원본 레이어에 남아 규정 위반.</summary>
<div markdown="1">

**예 —** `visits_raw` 원본을 backfill 용으로 90일 보관하면서 PII 테이블에서만 `user_id` 를 지우면,
삭제 요청 뒤에도 원본 JSON 에 `address`·`city` 가 남아 GDPR/CCPA 삭제 의무를 못 지킴.
또 Delta 에서 `delete` 만 하고 `VACUUM` 을 빼면 time travel 로 이전 버전을 읽어 복구까지 가능.

**권장 —** 수직 분할은 **첫 변환 단계부터만** 적용됨을 인지하고, 원본 데이터에는 **삭제 지연 한도 내의 짧은 retention** 을 걸 것.
Delta 는 `delete` 후 반드시 **`VACUUM`** 으로 물리 삭제, Kafka 는 **key 당 1건 토픽에만** compaction 기반 tombstone 을 쓸 것.

</div>
</details>

---

### 1-2. 패턴 #42: 제자리 덮어쓰기 (In-Place Overwriter)

> #41 Vertical Partitioner 는 **새 프로젝트를 시작하거나** 기존 워크로드를 마이그레이션할 **시간·컴퓨팅 자원이 충분** 할 때 좋음.
> 그 편안한 위치에 있지 못하면, 검증된 **덮어쓰기 전략** 에 기댈 수밖에 없음.

#### 상황 (Problem)

**책의 use case** — 개인정보 관리 전략이 없는 레거시 시스템을 물려받음:

- **테라바이트급 데이터** 가 **시간 기반 수평 파티션** 에 저장된 레거시 시스템을 인수. **개인정보 관리 전략이 정의돼 있지 않음**.
- 레거시임에도 조직에서 **널리 사용** 중. 그래서 정부의 새 프라이버시 규정을 준수해야 함 — 사용자 요청 시 개인정보를 즉시 제거.
- **결정적 제약**: 아키텍처를 **리팩터링할 자원이 없음**. #41 처럼 처음부터 컬럼을 나눠 설계할 수 없는 상황.

#### 해결 (Solution)

리팩터링 여유가 없으니 **In-Place Overwriter** — 기존 저장소를 **제자리에서 덮어써** 삭제. 구현은 스토리지 기술에 크게 의존.

- **네이티브 in-place 삭제 지원 시** — 제거 대상을 **WHERE 조건** 으로 겨냥해 `DELETE` 실행.
  - Iceberg·Delta Lake 같은 오픈 테이블 포맷은 잘못된 작업을 되돌리는 **time travel** 제공.
    단 개인정보를 삭제해도 **제거된 레코드를 담은 데이터 블록을 회수(reclaim)하지 않으면** 데이터가 그대로 남음(→ #41 의 `VACUUM` 과 같은 맥락).
- **네이티브 삭제 미지원 시**(JSON·CSV 등 원시 포맷) — 전체 데이터셋을 처리해 제거 사용자 레코드를 걸러내는 **시뮬레이션 잡** 실행(컴퓨팅 집약적).
  - **직접 교체 금지** — 잡이 재시도·실패하면 데이터 손실 가능. 대신 **스테이징 영역(staging area)** 에 결과를 쓰고,
    삭제가 성공한 뒤에만 기존 공개 데이터셋을 덮어쓰는 **데이터 승격(promotion) 잡** 실행. 승격 잡이 실패해도 스테이징에 이미 계산돼 있어 데이터 손실로 안 이어짐.
- **Compactable 저장소**(compaction 켠 Kafka 토픽 + key 기반 레코드) — ① 토픽 레코드 읽기 → ② 제거 대상이면 **record key + null payload** 전송.
  compaction 이 key 별 최신 항목만 남기므로, 개인정보가 든 이전 레코드가 제거되고 빈 delete marker 만 남음(#41 예시의 tombstone 과 동일).

```
[In-Place Overwriter — 저장소 기술별 삭제 구현]
──────────────────────────────────────────────────────────────────────
 삭제 요청(user_id)
   │
   ├─ 네이티브 삭제 지원(Delta·Iceberg)   DELETE WHERE user_id=... → 블록 회수(VACUUM) 필수
   │     └ 회수 안 하면 time travel 로 복구 가능 ⚠
   │
   ├─ 원시 파일(JSON·CSV)                전체 스캔·필터 → staging 기록 → 승격(promote)
   │     └ 최종 위치에 직접 덮어쓰기 금지(실패 시 손실) ⚠
   │
   └─ Compactable(Kafka, key 기반)       key + null(tombstone) → compaction 이 이전 레코드 제거
──────────────────────────────────────────────────────────────────────
 공통 대가 — 전체 데이터를 읽고 다시 씀 → I/O·비용이 #41 Vertical Partitioner 보다 큼.
```

> **참고 사항 — 삭제 벡터 (Deletion Vector)**
> 테이블 파일 포맷의 삭제 관리 방식은 둘. ① 제거된 행을 식별해 **작은 side file** 에 기록(쓰기 풋프린트 축소)하는 **deletion vector** — 컨슈머가 읽는 시점에 삭제 행을 직접 걸러냄. ② 반대로 제거 항목을 뺀 전체 데이터를 파일에 새로 씀 — 쓰기 집약적이지만 컨슈머는 바로 사용.

> **참고 사항 — 보존 기간 (Retention Period)**
> 데이터 보존 기간이 삭제 조치 지연 한도보다 짧으면, 그 자체를 삭제 전략으로 볼 수 있음(법정 기간 내 자동 제거). 단 구현 제안일 뿐, 최종 채택 전 **CDO·법무팀과 확인** 할 것.

```
[Figure 7-3 재현] 스테이징을 거친 데이터 제거
──────────────────────────────────────────────────────────────────────
 [ JSON public dataset ] ─► [ 데이터 제거 잡 ] ─► [ Staging area ]
                                                       │ 제거 성공 시에만
                                                       ▼
                               [ 데이터 승격 잡 ] ─► [ JSON public dataset ]
──────────────────────────────────────────────────────────────────────
 제거 잡은 결과를 staging 에만 씀 → 성공 후 승격 잡이 공개 데이터셋을 덮어씀.
 승격 잡이 실패해도 staging 에 이미 계산돼 있어 데이터 손실로 안 이어짐.
```

#### 고려사항 (Consequences)

읽기·쓰기 연산이 많은 패턴 — 둘 다 시스템에 부담.

- **I/O overhead(I/O 오버헤드)**
  - 파일을 읽고 덮어쓰기가 **심각한 I/O** 를 유발. 시간이 지나며 저장 공간이 거의 **2배** 로 커지고 처리량도 늘어남.
  - 완화 — 스토리지 계층이 필터 조건과 무관한 파일 읽기를 피할 수 있으면 작아짐. **Parquet·Delta·Iceberg** 는 데이터 블록별 통계(메타데이터)를 저장 → 쿼리 엔진이 제거 대상 행이 없는 블록을 **스킵**.
- **Cost(비용)**
  - 전체 데이터를 읽어야 하므로 **#41 Vertical Partitioner 보다 비쌈**. 예: 제거 엔티티 1건에 레코드 **2,000건** 이 있으면, Vertical Partitioner 는 **1건만** 읽고 drop, In-Place Overwriter 는 **2,000건** 이 영향받음.
  - 완화 — 삭제 요청을 **묶어서(batch)**, 요청마다 파이프라인 하나씩이 아니라 전체 요청에 대해 한 번 실행.

> **참고 사항 — 되돌릴 수 없는 롤백 (Impossible Rollback)**
> 스테이징 방식도 완벽치 않음. 삭제 잡에 버그가 있어 재실행해야 하는데, **원본이 이미 덮어써져** 원래 데이터셋을 못 씀. 완화 — Proxy 패턴에 기대거나 인프라 수준에서 **데이터 버저닝** 활성화(S3·Azure Storage·GCS 모두 버저닝 지원).

#### 구현 예시 (Examples)

**예시 1 — PySpark 로 플랫 파일에서 레코드 제거 (Example 7-5)**

Delta 예시는 #41 의 Example 7-3 과 동일 코드라 생략. **플랫 파일** 이 더 까다로움 — 전체를 필터링해 staging 에 새로 씀:
```python
input_raw_data = spark_session.read.text(get_input_table_dir())
df_w_user_column = input_raw_data.withColumn(
    'user', F.from_json('value', 'user_id STRING'))       # 필터용 user_id 만 추출
user_id = '139621130423168_029fba78-15dc-4944-9f65-00636566f75b'
to_save = df_w_user_column.filter(f"user.user_id != '{user_id}'").select('value')
to_save.write.mode('overwrite').format('text').save(get_staging_table_dir())  # 최종이 아닌 staging 에
```

> 데이터셋 변경을 최소화하려 **text API** 를 쓰고, 공간 절약을 위해 필터에 필요한 `user_id` 만 추출.
> Spark 는 **분산·비트랜잭션** 처리 계층이라 최종 위치에 바로 덮어쓰지 않고, staging 에 만든 뒤 **rename-like 명령** 으로 승격.

**예시 2 — AWS CLI 로 최종 위치 승격 (Example 7-6)**

```bash
aws s3 rm ${BUCKET}/output --recursive
aws s3 mv ${BUCKET}/staging ${BUCKET}/output --recursive
```

> 클라우드 오브젝트 스토어의 rename 은 로컬 파일시스템과 같은 트랜잭션 시맨틱이 아님
> — 흔히 **copy-and-remove** 로 구현돼, 실패 시 빈 유효 상태(empty valid state)를 남길 수 있음.

<details>
<summary><b>⚠ 트러블 로그</b> — In-Place Overwriter로 삭제하면서 staging을 건너뛰거나 블록 회수를 잊으면 데이터가 손실·잔존.</summary>
<div markdown="1">

**예 —** 시뮬레이션 잡이 최종 경로에 바로 `mode('overwrite')` 로 쓰다가 중간에 죽으면, 공개 데이터셋이 **반쯤 지워진 상태** 로 남아 다운스트림이 깨짐.
반대로 Delta 에서 `DELETE` 만 하고 블록 회수(`VACUUM`)를 빼면, time travel 로 이전 버전을 읽어 삭제한 사용자를 **복구** 할 수 있어 규정 위반.

**권장 —** 원시 포맷은 반드시 **staging → 승격** 2단계로 처리하고, 테이블 포맷은 `DELETE` 뒤 **블록 회수(VACUUM/expire)** 까지 돌릴 것. 원본은 인프라 **버저닝** 으로 롤백 경로를 확보.

</div>
</details>

---

## 2. 접근 제어 (Access Control)

효율적인 데이터 제거만으로는 기본 보안이 안 됨 — 가장 중요한 데이터 구획에는 **인가된 사용자만** 접근하게 해야 함.
개인정보를 비공개로 두는 것도 중요하지만, **데이터 그 자체가 최대 경쟁 자산** 이기도 함.

- **#43 Fine-Grained Accessor for Tables** — 테이블의 **컬럼/행 단위** 접근 제어(고전적 분석 세계). (아래 2-1)
- **#44 Fine-Grained Accessor for Resources** — **클라우드 리소스 단위** 접근 제어(최소 권한 원칙). (아래 2-2)

---

### 2-1. 패턴 #43: 테이블 세분화 접근자 (Fine-Grained Accessor for Tables)

> 사용자·그룹을 만들어 특정 테이블 권한을 주는 고전적 방식에 딱 맞음. 그런데 그보다 **더 세밀한** 제어가 가능.

#### 상황 (Problem)

**책의 use case** — HDFS/Hive 를 클라우드 DW 로 옮기며 마주친 저수준 인가 요구:

- 이전 **HDFS/Hive 워크로드를 클라우드 DW 로 마이그레이션** 한 뒤 보안 접근 정책을 구현해야 함.
- 첫 요구(사용자·그룹으로 **테이블** 접근 관리)는 쉬움 — 새 DW 가 고전적 사용자·그룹을 지원.
- 그러나 이해관계자의 추가 요구 — 테이블 접근 권한이 있어도 **모든 컬럼·행을 읽을 권한은 없을** 수 있음. 이 **저수준 리소스** 에도 인가 메커니즘이 필요.
- **결정적 제약**: 테이블 단위가 아니라 **컬럼·행 단위** 접근 제어가 필요.

#### 해결 (Solution)

**컬럼 기반 접근** 은 세 가지 구현.

- **① GRANT 연산자** — 인가 액션 스코프 안의 컬럼을 정의. Amazon Redshift·PostgreSQL 지원.
  (Example 7-7 — 두 컬럼만 읽기 허용: `GRANT SELECT(col_A, col_B) ON my_table TO some_user;`)
- **② data catalog + 정책 태그** — GCP BigQuery 방식. Data Catalog 에 **정책 태그** 를 만들어 보호 컬럼에 할당하고, 사용자에게 태그별 **Fine-Grained Reader** 역할을 부여.
- **③ data masking** — Databricks(Unity Catalog)·Snowflake 방식. 사용자가 보호 컬럼을 보되 접근권이 없으면 **내용이 숨겨짐**.
  (Example 7-8 — `engineers` 그룹만 `ip` 컬럼을 보는 마스킹 함수)
  ```sql
  CREATE FUNCTION ip_mask(ip STRING)
    RETURN CASE WHEN is_member('engineers') THEN ip ELSE '.' END;

  CREATE TABLE visits (
    visit_id STRING,
    ip STRING MASK ip_mask);          -- 컬럼에 마스킹 함수 부착
  ```

**행 기반 접근(row-level security)** 은 실행 쿼리에 **WHERE 조건을 동적으로** 추가. 이름은 제각각
— Databricks `ROW FILTER`, Amazon Redshift Row-Level Security, GCP BigQuery·Snowflake row access policies.
보호 테이블의 모든 select 에 동적 조건을 더하는 **별도 DB 객체** 를 정의.

- 네이티브 미지원 시 — **view + access guard 조건** 으로 시뮬레이션.
  (Example 7-9 — 쿼리 발행 사용자 소유 blog 만 반환)
  ```sql
  CREATE VIEW users_blogs AS
  SELECT ... FROM blogs WHERE table.blog_author = current_user
  ```
  조건이 사용자마다 달라져, 각 사용자가 **전용 view** 를 가진 것처럼 동작.

```
[Fine-Grained Accessor — 같은 테이블, 사용자마다 다른 뷰]
──────────────────────────────────────────────────────────────────────
 원본 visits     visit_id | city  | ip         | user_id
                 v1       | Seoul | 10.0.0.1   | u-10
                 v2       | Busan | 10.0.0.2   | u-20

 analyst (engineers 그룹 아님, 세션 user_id=u-10)
   컬럼: ip 마스킹(MASK)  +  행: RLS(user_id=current_user)
                 visit_id | city  | ip  | user_id
                 v1       | Seoul | .   | u-10        ← ip 가려짐 + u-20 행은 안 보임

 engineer (engineers 그룹)
                 visit_id | city  | ip         | user_id
                 v1       | Seoul | 10.0.0.1   | u-10   ← ip 원본
                 v2       | Busan | 10.0.0.2   | u-20   ← 모든 행
──────────────────────────────────────────────────────────────────────
 "어느 열(컬럼 마스킹) × 어느 행(row 필터)" 을 곱해 사용자별 뷰가 만들어짐.
```

#### 고려사항 (Consequences)

DB 네이티브 지원이라 앞선 패턴들보다 단점이 상대적으로 적음.

- **Row-level security limits(행 수준 보안의 한계)**
  - 대부분의 row-level 구현은 **연결 세션에서 직접 얻는 속성**(사용자명·그룹·IP)으로 적용 범위가 제한됨.
- **Data type(데이터 타입)**
  - 컬럼이 **nested structure** 같은 복합 타입이면 단순 컬럼 기반 전략을 못 씀. 먼저 **unnest** 해 다른 테이블로 노출하거나(Dataset Materializer 패턴), fine-grained 권한을 지원하면 그걸 사용.
- **Query overhead(쿼리 오버헤드)**
  - row/column 보호가 쿼리에 **동적으로 추가되는 SQL 함수** 로 표현됨 → 예기치 않은 지연 유발 시, 허용된 데이터만 담은 **전용 테이블·view**(Dataset Materializer)로 완화. 단 데이터 중복·거버넌스 부담이 따름.

#### 구현 예시 (Examples)

**예시 — PostgreSQL(컬럼·행) + DynamoDB (Example 7-10 ~ 7-12)**

컬럼 접근 — 나열한 컬럼만 허용, `SELECT *` 는 거부(Example 7-10):
```sql
GRANT SELECT(id, login, registered_datetime) ON dedp.users TO user_a;
-- user_a 가 미포함 컬럼을 조회하면: ERROR: permission denied for table users
```

행 접근 — 정책으로 `login = current_user` 조건을 모든 조회에 주입(Example 7-11):
```sql
ALTER TABLE dedp.users ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_row_access ON dedp.users USING (login = current_user);
```

NoSQL 도 지원 — DynamoDB 는 IAM 정책의 `dynamodb:LeadingKeys` 로 **자기 user_id 로 시작하는 행만** 읽게 함(Example 7-12):
```json
{
 "Statement":[{
   "Sid": "...",
   "Effect":"Allow",
   "Action":["..."],
   "Resource":["arn:aws:dynamodb:us-west-1:123456789012:table/users"],
   "Condition":{
     "ForAllValues:StringEquals":{
       "dynamodb:LeadingKeys":["${www.amazon.com:user_id}"]
     }
   }
 }]
}
```

<details>
<summary><b>⚠ 트러블 로그</b> — 컬럼 마스킹만 걸고 row-level을 빠뜨리면 다른 사용자의 행이 통째로 노출됨.</summary>
<div markdown="1">

**예 —** `ip` 컬럼에 마스킹 함수를 걸어 안심했지만 `ENABLE ROW LEVEL SECURITY` 를 안 걸면,
분석가가 `SELECT visit_id, city FROM visits` 로 **모든 사용자의 방문 행** 을 그대로 조회 — 컬럼 하나만 가려졌을 뿐 행 필터는 없음.
또 `context` 같은 **nested 컬럼** 엔 컬럼 GRANT 가 안 먹어, 통째로 열리거나 통째로 막힘.

**권장 —** 컬럼 보호와 행 보호는 **별개** 임을 기억하고 둘을 함께 설계할 것. 복합 타입 컬럼은 먼저 unnest 해 평탄화한 뒤 권한을 적용.

</div>
</details>

---

### 2-2. 패턴 #44: 리소스 세분화 접근자 (Fine-Grained Accessor for Resources)

> 접근 기반 패턴은 **테이블** 데이터셋에 좋음. 그런데 DB 만 쓰는 건 아님 — 클라우드 제공자가 관리하는 다른 데이터 스토어(오브젝트 스토어·큐 등)에도 적용.

#### 상황 (Problem)

**책의 use case** — 보안 감사가 지적한 과도한 권한:

- 보안 감사가 클라우드 계정의 **과도하게 넓은 권한** 을 탐지. 위험 중 하나 — 한 데이터 처리 잡이 계정의 **모든 데이터셋을 덮어쓸** 가능성.
- 감사관이 **최소 권한(at-least privilege)** 모범 사례를 제시 — 각 컴포넌트에 **필요한 최소 권한만** 부여해, 데이터 처리 잡이 **실제 작업하는 데이터셋만** 다루게.
- **결정적 제약**: 클라우드 제공자에서 최소 권한을 **기술적으로 구현** 해야 함.

#### 해결 (Solution)

모든 주요 클라우드(AWS·Azure·GCP)가 최소 권한 구현을 제공 = 이 패턴의 backbone. **두 전략**.

- **① resource 기반** — 접근 스코프를 **리소스 수준에서 직접** 정의.
  (Example 7-13 — GCS 버킷에 IAM 정책 부착, Terraform)
  ```hcl
  data "google_iam_policy" "admin_access" {
    binding {
      role    = "roles/storage.admin"
      members = ["user:admingcs@waitingforcode.com",]
    }
  }
  resource "google_storage_bucket_iam_policy" "policy" {
    bucket      = google_storage_bucket.default.name
    policy_data = data.google_iam_policy.admin_access.policy_data
  }
  ```
- **② identity 기반** — 접근 권한을 **identity 수준**(사람 또는 애플리케이션 사용자)에서 정의. AWS 는 서비스가 assume 하는 IAM role 로 지원.
  (Example 7-14 — Spark EMR 잡이 Kinesis Data Streams 를 읽고 쓰게)
  ```hcl
  data "aws_iam_policy_document" "emr_assume_role" {
    statement {
      effect     = "Allow"
      principals { type = "Service"; identifiers = ["elasticmapreduce.amazonaws.com"] }
      actions    = ["sts:AssumeRole"]
    }
  }
  resource "aws_iam_role" "job_role" {
    name               = "visits-processor-role"
    assume_role_policy = data.aws_iam_policy_document.emr_assume_role.json
  }
  resource "aws_iam_policy" "visits_read_writer_policy" {
    name   = "visits_rw"
    policy = jsonencode({
      Version   = "2012-10-17"
      Statement = [{
        Action   = ["kinesis:Get*", "kinesis:Describe*", "kinesis:List*", "kinesis:Put*"]
        Effect   = "Allow"
        Resource = ["arn:aws:kinesis:us-east-1:1234567890:streams/visits"]}]
    })
  }
  resource "aws_iam_role_policy_attachment" "policy_attachment" {
    role       = aws_iam_role.job_role.name
    policy_arn = aws_iam_policy.visits_read_writer_policy.arn
  }
  ```

fine-grained 권한은 유연 — 특정 리소스, 같은 **prefix 로 시작하는 리소스 집합**, 심지어 **runtime 조건 기반** 리소스도 타깃 가능.
tag 기반도 가능(Example 7-15 — S3 에서 `aws:TagKeys` 로 `user_id` 태그가 붙은 리소스만 PutObject):
```json
"Statement": [{
 "Effect": "Allow", "Action": "s3:PutObject",
 "Resource": "*", "Condition": {
  "ForAllValues:StringEquals": {"aws:TagKeys": ["${www.amazon.com:user_id}"]}}
}]
```

```
[Figure 7-4 재현] resource 기반 vs identity 기반 접근 제어
──────────────────────────────────────────────────────────────────────
 resource 기반   User A ──► [ Object store ]    ← 접근 정책이 "리소스"에 붙음
                            (admin: user A / reader: group B·user C / writer: user B)

 identity 기반   User A ──► [ Database A ]       ← 접근 정책이 "사용자"에 붙음
                 (User A 정책: admin: database A / reader: message queue B / writer: object store C)
──────────────────────────────────────────────────────────────────────
 resource 기반 = 정책을 각 리소스에 부착 / identity 기반 = 정책을 각 사용자에 부착.
```

#### 고려사항 (Consequences)

IaC 나 커스텀 스크립트로 정의하니 기술적으론 어렵지 않음. 하지만 트레이드오프가 있음.

- **Security by the book trade-off(원칙과 현실의 트레이드오프)**
  - 최소 권한 원칙은 훌륭하나, **많은 작은 접근 정책** 을 낳아 복잡한 환경에서 유지보수가 어려움.
  - 완화 — **wildcard/prefix 기반**(예: `visits*`)으로 리소스 수를 관리 가능하게. 단 이는 최소 권한 위반 소지 — **미래에 생길** visits-prefixed 리소스 접근까지 허용될 수 있음. 보안팀과 논의 필요.
- **Complexity(복잡도)**
  - resource 기반과 identity 기반을 한 프로젝트에 **둘 다** 쓰면 복잡도 증가. 가능하면 use case 를 더 많이 커버하는 **한 방식** 을 선호.
- **Quotas(쿼터)**
  - 접근 정책에도 한도가 있음. 예: AWS IAM 기본 커스텀 정책 **1,500개**, GCP IAM 프로젝트당 커스텀 role **300개**. 일부는 유연해 제공자에 상향 요청 가능.

#### 구현 예시 (Examples)

**예시 — 권한 없는 접근의 예외와 두 부여 방식 (Example 7-16 ~ 7-18)**

권한 없이 S3 버킷을 읽으면 예외(Example 7-16):
```bash
$ aws s3 ls s3://dedp-visits-301JQN/
# An error occurred (AccessDenied) when calling the
# ListObjectsV2 operation: Access Denied
```

identity 기반 — 역할에 S3 읽기 액션을 스코프(Example 7-17):
```json
{"Version": "2012-10-17", "Statement": [{"Sid": "VisitsS3Reader",
 "Effect": "Allow", "Action": ["s3:Get*", "s3:List*"],
 "Resource": ["arn:aws:s3:::dedp-visits-301JQN/*",
   "arn:aws:s3:::dedp-visits-301JQN"]
}]}
```

resource 기반 대안 — 버킷 정책으로 특정 user 를 인가(Example 7-18):
```bash
$ aws s3api put-bucket-policy --bucket dedp-visits-301JQN --policy file://policy.json
# policy.json
{"Statement": [{"Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::123456789012:user/visits-s3-reader"},
  "Action": ["s3:Get*", "s3:List*"],
  "Resource": "arn:aws:s3:::dedp-visits-301JQN/*"
}]}
```

<details>
<summary><b>⚠ 트러블 로그</b> — 유지보수 편하려고 wildcard 권한을 넓게 주면 최소 권한이 무너짐.</summary>
<div markdown="1">

**예 —** 정책 개수를 줄이려 `arn:aws:s3:::visits*` 처럼 prefix 로 묶으면, 나중에 만든 `visits-raw-pii` 버킷까지 **자동으로 읽기 가능** 해져 감사에서 지적받음.
반대로 resource 기반과 identity 기반을 한 계정에 섞어 쓰면, 같은 버킷에 정책이 두 군데 붙어 **어느 쪽이 거부인지** 추적이 어려움.

**권장 —** wildcard 는 미래 리소스까지 포함됨을 전제로 **보안팀 승인 후** 최소 범위로만 쓰고, resource·identity 중 **한 방식** 으로 통일할 것. 정책 쿼터(AWS 1,500 / GCP 300)도 사전에 확인.

</div>
</details>

---

## 3. 데이터 보호 (Data Protection)

데이터 접근을 **논리 수준**(DB·클라우드 서비스 스코프)에서 통제하는 것만으로는 완전히 안전한 시스템이 못 됨.
빠진 조각은 **데이터 그 자체를 보호** 해 예상치 못한 사용에 대비하는 것.

- **#45 Encryptor** — 저장(at rest)·전송(in transit) 데이터를 **암호화**. 접근 통제가 뚫려도 **복호화 키** 없이는 못 읽게 함. (아래 3-1)
- **#46 Anonymizer** *(범위 밖 — 후속 문서)* — 민감 데이터를 **제거·변형** 해 재식별 불가능하게(제거·교란·합성). 최강 보호지만 분석 가치를 훼손.
- **#47 Pseudo-Anonymizer** *(범위 밖 — 후속 문서)* — 민감 값을 **쓸모 있는 가짜 값** 으로 대체(마스킹·토큰화·해싱·암호화). 분석은 가능하나 조합 재식별 위험.

---

### 3-1. 패턴 #45: 암호화기 (Encryptor)

> 클라우드에서 잡을 돌려도 데이터는 **물리적으로 어딘가 저장** 되고, 인가되지 않은 사람이 읽으려 할 수 있음.
> 접근 정책(#43·#44)에 더해, **접근 통제가 뚫려도 데이터를 못 쓰게** 만드는 계층이 암호화.

#### 상황 (Problem)

**책의 use case** — 저장·전송 데이터 보안을 강제하라는 과제:

- 테이블·클라우드 리소스에 fine-grained 접근 정책(#43·#44)을 구현한 뒤, **저장(at rest)·전송(in transit) 데이터 보안** 을 강제하는 과제를 받음.
- 이해관계자 우려 — 인가되지 않은 사람이 **스트리밍 브로커와 잡 사이 전송 데이터** 를 가로채거나, **서버에서 데이터를 물리적으로** 훔칠 수 있음.
- **결정적 제약**: 접근 통제가 뚫려도 데이터가 읽히면 안 됨 → 데이터 자체를 암호화.

#### 해결 (Solution)

**Encryptor** — 두 보호 수준이 필요하니 **두 구현**.

- **① 저장 데이터(at rest)** — **client-side** 또는 **server-side** 암호화. 차이는 **암호화 키 관리** 주체.
  - **client-side** — 데이터 프로듀서가 저장 전 암호화하고 **키 관리도 책임**.
  - **server-side** — 모든 암/복호화를 서버가 수행. 컨슈머·프로듀서는 요청만 보내고 **키 관리 포함 전부** 서버가 처리.
    공개 클라우드가 널리 지원 — **AWS·GCP는 KMS(Key Management Service), Azure는 Key Vault**.
- **② 전송 데이터(in transit)** — 클라이언트↔저장소가 네트워크로 데이터를 교환하는 계층.
  클라우드 구현은 비교적 쉬움 — 클라이언트 **SDK 수준에서 보안 통신** 활성화 + 서비스에 필요한 **프로토콜 버전(TLS)** 설정.

```
[Figure 7-5 재현] 저장 데이터 server-side 암호화 워크플로
──────────────────────────────────────────────────────────────────────
                          ┌─ Encryption key store ─┐
                     ③ 키 │  (KMS / Key Vault)      │
                    ┌────►└──────────▲──────────────┘
                    │            ② 키 요청│
        ① 요청       │                  │
 [ Client ] ─────────┴──► [ Encryptable data store ]
      ▲                              │
      └──────── ④ 복호화된 데이터 ─────┘
──────────────────────────────────────────────────────────────────────
 ① 요청이 암호화 저장소에 도달 → ② 저장소가 키 저장소에 복호화 키 요청(인가 안 되면 실패)
 → ③ 키 획득 → ④ 복호화한 레코드를 클라이언트에 반환. 클라우드에선 이 교환이 완전히 추상화됨.
```

#### 고려사항 (Consequences)

물리 저장 수준의 보안이라 **공짜는 아님**.

- **Encryption/decryption overhead(암/복호화 오버헤드)**
  - 데이터가 평문이 아니라 변형된 형태로 저장 → 암/복호화 없이는 못 씀. 매 읽기·쓰기가 **CPU 에 추가 부담**.
- **Data loss risk(데이터 손실 위험)**
  - 저장 데이터를 보호하지만, 부작용으로 **인가 사용자 접근도 차단** 될 수 있음 — **암호화 키를 잃거나** 키 접근을 잃으면.
  - 완화 — 클라우드는 암호화 저장소에 **soft delete** 를 구현. 삭제 요청이 즉시 반영되지 않고 **유예 기간** 동안 실수를 복원 가능.
- **Protocol updates(프로토콜 갱신)**
  - 전송 암호화는 설정이 쉽지만 **최신 유지가 필요** — TLS 는 1.0·1.1 이 deprecated(보안 이슈)라 그 버전 쓰는 서비스는 업그레이드 필요. 클라우드에선 **서비스 수준에서 버전 업그레이드** 로 단순화됨.

#### 구현 예시 (Examples)

**예시 — AWS KMS 암호화 + S3 연결 + Azure TLS 강제 (Example 7-19 ~ 7-21)**

KMS 암호화 키 정의 + 다른 서비스(Lambda IAM role)에 암/복호화 권한 부여(Example 7-19):
```hcl
module "kms" {
  source                  = "terraform-aws-modules/kms/aws"
  key_usage               = "ENCRYPT_DECRYPT"
  deletion_window_in_days = 14                       # data loss 완화용 키 복원 창(soft delete)
  aliases                 = ["visits-bucket-encryption-key"]
  grants = {
    lambda_doc_convert = {
      grantee_principal = aws_iam_role.iam_key_reader.arn
      operations        = ["Encrypt", "Decrypt", "GenerateDataKey"]
    }
  }
}
```

정의한 키를 S3 버킷에 연결 — 기본 server-side 암호화(Example 7-20):
```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "visits" {
  bucket = aws_s3_bucket.visits.id
  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = module.kms.key_arn
      sse_algorithm     = "aws:kms"
    }
  }
}
```

전송 암호화 — Azure Event Hubs 에 **최소 TLS 버전** 강제(Example 7-21):
```hcl
resource "azurerm_eventhub_namespace" "visits" {
  name                = "visits-namespace"
  location            = azurerm_resource_group.dedp.location
  resource_group_name = azurerm_resource_group.dedp.name
  sku                 = "Standard"
  capacity            = 2
  minimum_tls_version = "1.2"                         # 구버전 TLS 클라이언트 차단
}
```

> 요약하면 암호화는 **키를 암호화 대상 리소스·인가 identity 와 연결** 하는 일. 저장은 KMS 키를 버킷에 붙이고, 전송은 서비스에 최소 TLS 버전을 설정.

<details>
<summary><b>⚠ 트러블 로그</b> — 암호화 키를 성급히 삭제하거나 구버전 TLS를 방치하면 데이터를 못 읽거나 전송이 가로채짐.</summary>
<div markdown="1">

**예 —** 정리하겠다고 KMS 키를 즉시 삭제하면, 그 키로 암호화된 S3 객체 전체가 **영구히 복호화 불가**
— soft delete 유예 없이 지우면 인가 사용자도 못 읽음.
반대로 Event Hubs 의 `minimum_tls_version` 을 안 걸어 TLS 1.0 클라이언트를 허용하면, 스트리밍 구간에서 데이터가 평문에 가깝게 **가로채질** 수 있음.

**권장 —** 키에는 반드시 **삭제 유예 창(`deletion_window_in_days`)** 을 두고, 전송에는 **`minimum_tls_version = "1.2"` 이상** 을 강제할 것. 키 접근권을 잃으면 데이터도 잃는다는 점을 운영 문서에 명시.

</div>
</details>

---

## 4. 연결성 (Connectivity)

> **범위 밖(후속 문서)** — 데이터 저장소에 **안전하게 연결** 하는 카테고리. 자격 증명(credentials) 관리가 핵심.

- **#48 Secrets Pointer** *(범위 밖)* — 자격 증명을 코드/Git 에 두지 않고 **시크릿 매니저의 이름(참조)** 으로 가리킴. 유출 위험을 낮추되 캐시 무효화·로그 유출을 주의.
- **#49 Secretless Connector** *(범위 밖)* — 아예 **관리할 자격 증명 없이** IAM·인증서 기반으로 연결. 관리할 시크릿은 0 이지만 설정·로테이션 부담이 있음.

---

## 5. 요약

챕터 7의 데이터 보안은 **"만든 데이터 자산을 어떻게 지키나"** 를 네 카테고리로 다룸 — **데이터 제거 · 접근 제어 · 데이터 보호 · 연결성**.
본 문서는 그중 **#41·#42(데이터 제거) · #43·#44(접근 제어) · #45(데이터 보호)** 를 정리했고, **#46·#47(데이터 보호 나머지)과 #48·#49(연결성)** 은 후속 문서 몫.

- **데이터 제거** — 잊힐 권리 대응. #41 **Vertical Partitioner** 는 **지울 대상을 애초에 줄이고**(컬럼 분리), #42 **In-Place Overwriter** 는 리팩터링 여유가 없을 때 **기존 저장소를 제자리에서 덮어써** 삭제(staging→승격, 블록 회수 필수).
- **접근 제어** — 인가된 사용자만 접근. #43 **Fine-Grained Accessor for Tables** 는 **컬럼/행 단위**(GRANT·마스킹·row-level), #44 **for Resources** 는 **클라우드 리소스 단위**(최소 권한, IAM resource/identity 기반).
- **데이터 보호** — 데이터 자체를 보호. #45 **Encryptor** 는 저장·전송 데이터를 **암호화** 해, 접근 통제가 뚫려도 **키 없이는 못 읽게** 함.

| 패턴 | 카테고리 | 한 줄 요약 | 핵심 트레이드오프 |
|---|---|---|---|
| #41 Vertical Partitioner | Data Removal | 가변 event ↔ 불변 PII 를 컬럼 단위로 분리해 삭제 대상 최소화 | 쓰기·삭제↓ / <br>읽기 시 join·복잡도↑ |
| #42 In-Place Overwriter | Data Removal | 기존 저장소를 제자리에서 덮어써 삭제(staging→승격) | 리팩터링 불필요 / <br>I/O·비용↑(전체 스캔) |
| #43 Fine-Grained Accessor (Tables) | Access Control | 테이블 컬럼/행 단위 접근 제어(GRANT·마스킹·RLS) | DB 네이티브 / <br>복합 타입·쿼리 오버헤드 |
| #44 Fine-Grained Accessor (Resources) | Access Control | 클라우드 리소스 단위 최소 권한(IAM) | 유출 반경↓ / <br>정책 폭증·쿼터·복잡도 |
| #45 Encryptor | Data Protection | 저장·전송 데이터를 암호화, 키 없으면 못 읽음 | 접근 뚫려도 보호 / <br>CPU 오버헤드·키 분실 위험 |

```
[챕터 7 전반 선택 가이드 — #41~#45]
──────────────────────────────────────────────────────────────────────
 삭제 대상을 지울 때
   새 설계·마이그레이션 여유 있음        ─► #41 Vertical Partitioner (컬럼 분리)
   레거시·자원 부족                     ─► #42 In-Place Overwriter (제자리 덮어쓰기)

 접근을 좁힐 때
   테이블의 컬럼/행 단위                 ─► #43 Fine-Grained Accessor for Tables
   클라우드 리소스(버킷·스트림) 단위      ─► #44 Fine-Grained Accessor for Resources

 데이터 자체를 지킬 때
   저장·전송 데이터를 못 읽게            ─► #45 Encryptor (KMS/Key Vault·TLS)
──────────────────────────────────────────────────────────────────────
 ⚠ 삭제는 논리 삭제 뒤 물리 회수(VACUUM/블록 reclaim)까지, 암호화는 키 삭제에 유예 창을 둘 것.
```

**정리** — 챕터 7 전반부는 **"지울 데이터를 줄이거나 확실히 지우고(#41·#42) → 접근을 컬럼·행·리소스 단위로 좁히고(#43·#44) → 뚫려도 못 읽게 암호화(#45)"** 로 이어지는 방어선.
남은 조각은 데이터를 **비식별화(#46·#47)** 해 안전하게 공유하고, 저장소 **연결에서 자격 증명을 숨기거나 없애는(#48·#49)** 패턴 — 후속 문서에서 다룸.
