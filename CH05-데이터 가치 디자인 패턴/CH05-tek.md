# 데이터 엔지니어링 디자인 패턴 - 데이터 가치 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 5 | 실무 데이터 엔지니어링 관점 정리

> **도식 기호 범례** — `⚠` 위험·함정(조심해야 할 동작) · `✗` 잘못된 결과(깨진 상태) · `✓` 올바른 결과 · `⇒` 그 결과 도출

---

## 목차

1. [데이터 강화 (Data Enrichment)](#1-데이터-강화-data-enrichment)
   - 패턴 #23: 정적 조이너 (Static Joiner)
   - 패턴 #24: 동적 조이너 (Dynamic Joiner)
2. [데이터 데코레이션 (Data Decoration)](#2-데이터-데코레이션-data-decoration)
   - 패턴 #25: 래퍼 (Wrapper)

> 본 문서가 다루는 범위는 챕터 5 중 **#23~#25** (5.1 Data Enrichment 전체 + 5.2 Data Decoration 의 Wrapper)임.
> 나머지(#26 Metadata Decorator, 5.3 Aggregation, 5.4 Sessionization, 5.5 Ordering)는 후속 문서에서 다룸.

---

## 책의 use case (챕터 도입)

> 데이터는 저장소에 그냥 놓여 있다고 해서 자산(asset)이 아님. 수집된 직후의 데이터는 대개 빈약하고 품질 문제가 많음.

책의 예시 — 스트리밍 브로커로 들어오는 **방문(visit) 이벤트**:

- 스트리밍 계층의 데이터 producer 는 **웹 브라우저** → 브라우저 버전·언어·OS 같은 기술 정보는 얻을 수 있음.
- 하지만 "특정 브라우저를 쓰는 방문자들의 공통점" 같은 걸 알려면? 각 방문 이벤트는 **명시적 관계 없이 개별 item** 으로 수집되므로
  추가 작업 없이는 상관관계를 만들 수 없음.
- 이처럼 **데이터셋을 더 유용하게 만드는** 게 데이터 가치(Data Value) 패턴의 목적.

```
[챕터 5 데이터 가치 패턴 — 무엇으로 가치를 더하는가]
==========================================================================
 5.1 Data Enrichment   "다른 데이터셋과 결합" → #23 Static / #24 Dynamic Joiner
 5.2 Data Decoration   "개별 속성을 계산·부착" → #25 Wrapper / #26 Metadata Decorator
 5.3 Data Aggregation  "대량 데이터를 요약"     → #27 Distributed / #28 Local Aggregator
 5.4 Sessionization    "이벤트를 세션으로 묶음"  → #29 Incremental / #30 Stateful Sessionizer
 5.5 Data Ordering     "순서가 중요할 때 정렬"   → #31 Bin Pack / #32 FIFO Orderer
==========================================================================
 본 문서: 5.1 (#23, #24) + 5.2 의 Wrapper (#25)
```

| 패밀리 | 적용 상황 (언제 쓰나) | 핵심 동작 |
|---|---|---|
| **5.1 Data Enrichment** (#23, #24) | raw 데이터에 **다른 데이터셋의 context** 가 없을 때 | 두 데이터셋을 key 로 **JOIN** 해 보강 (#23 정적 / #24 양쪽 스트리밍) |
| **5.2 Data Decoration** (#25, #26) | 데이터가 raw·비정형이라 **추가 가공** 이 필요할 때 | record 에 계산값을 **부착·분리** (#25 Wrapper / #26 Metadata Decorator) |

> 참고
> — Data Enrichment(결합)·Data Decoration(개별 속성 계산)은 **소량/문맥 보강** 에 강하지만,
> 대량 데이터의 **개요(overview)** 가 필요하면 Aggregation·Sessionization 으로 넘어가야 함.

---

## 1. 데이터 강화 (Data Enrichment)

raw 데이터는 기술적 제약 때문에 빈약한 경우가 많음 — 특히 **이벤트** 가 대표적.
이벤트는 시간·공간 속성은 잘 담지만 **확장된 문맥(extended context)** 은 부족함
(예: 방문 이벤트에 작성자는 있어도 그 사용자의 마지막 접속일·프로필 점수는 없음).
Data Enrichment 패턴은 이 한계를 메워 데이터를 **여러 이해관계자에게 더 유용하게** 만듦.

- **#23 Static Joiner** — 보강용 데이터셋이 **정적(static)** 일 때. 가장 쉬운 상황.
- **#24 Dynamic Joiner** — **양쪽 모두 움직이는(streaming)** 데이터셋을 결합할 때.

---

### 1-1. 패턴 #23: 정적 조이너 (Static Joiner)

#### 상황 (Problem)

**책의 use case** — 사용자 등록일과 일별 활동의 관계를 단순화하는 데이터셋 요청:

- 팀이 만든 데이터셋을 비즈니스 이해관계자들이 활발히 사용 중.
- 새 프로젝트에서 **사용자 등록일(registration date)과 일별 활동** 의 의존성을 쉽게 볼 수 있는 데이터셋을 요청받음.
- 그런데 **raw 데이터셋에는 user context 가 없음** — 그 정보는 **정적 사용자 참조 데이터셋(static user reference dataset)** 에만 있음.
- **결정적 제약**: 보강용 데이터셋의 성격이 **정적(at-rest)** 임 → 참조 데이터를 사용자 활동에 끌어와 붙이면 됨.

#### 해결 (Solution)

조인 결과가 at-rest 성격이라 **Static Joiner** 가 딱 맞음.
놀랍게도 이 정적 데이터의 특성에도 불구하고 **스트리밍 파이프라인에서도** 동작함.

- 구현에는 두 데이터셋을 결합할 **공통 속성 목록** 이 필요 (예: visits·users 양쪽의 `user_id`).
- 이 keyed 조건 외에 **시간 제약(time constraint)** 이 추가로 필요할 수 있음
  — 특히 보강용 데이터셋이 **SCD(slowly changing dimensions)** 형태일 때.
  이 경우 **time-sensitive static joiner** 변형을 구현함.

> **참고 사항 — Slowly Changing Dimensions (SCD)**
> 시간 민감 보강이 필요하면 **SCD** 로 구현 — 엔티티의 시간에 따른 변화를 추적하는 데이터 모델링 전략.
> 책의 해결책에서는 **SCD type 2 또는 4** 를 권장 (둘 다 row 변화를 추적, 기술적 구현만 다름).
> - **SCD type 2**: validity date(`from_date`/`to_date`)로 추적. 현재 값은 `to_date` 가 비어 있음(NULL).
> - **SCD type 4**: 테이블 2개. 하나는 엔티티별 **현재 값**, 다른 하나는 **현재 포함 전체 이력**.

```
[Figure 5-1 재현] SCD type 2 / type 4 — user emails 이력
──────────────────────────────────────────────────────────────────────
 SCD Type 2 (User emails history) — 한 테이블에 현재+과거 공존
 ┌─────────┬───────┬────────────┬────────────┬────────────┐
 │ User_id │ Email │ From_date  │ To_date    │ is_current │
 ├─────────┼───────┼────────────┼────────────┼────────────┤
 │   1     │ a@... │ 2024-06-01 │ 2024-06-21 │   false    │
 │   1     │ b@... │ 2024-06-21 │ 2024-07-05 │   false    │
 │   1     │ c@... │ 2024-07-05 │   NULL     │   true     │  ← 현재 값(To_date 비어 있음)
 └─────────┴───────┴────────────┴────────────┴────────────┘

 SCD Type 4 — 현재 테이블 + 이력 테이블 분리
 (User emails current)                 (User emails history)
 ┌─────────┬───────┬────────────┐      ┌─────────┬───────┬────────────┬────────────┐
 │ User_id │ Email │ From_date  │      │ User_id │ Email │ From_date  │ To_date    │
 ├─────────┼───────┼────────────┤      ├─────────┼───────┼────────────┼────────────┤
 │   1     │ c@... │ 2024-07-05 │      │   1     │ a@... │ 2024-06-01 │ 2024-06-21 │
 └─────────┴───────┴────────────┘      │   1     │ b@... │ 2024-06-21 │ 2024-07-05 │
                                       └─────────┴───────┴────────────┴────────────┘
──────────────────────────────────────────────────────────────────────
```

이메일이 a→b→c 로 바뀐 사용자의 변화 이력을 type 2(한 테이블)와 type 4(두 테이블)로 각각 표현한 그림.

코드 구현은 보통 SQL `JOIN` 으로 표현되며, 현대 처리 프레임워크와 고전적 DW 모두에서 동작하는 **범용** 해법임.
선언적 SQL 외에 **프로그래밍 API(HTTP 라이브러리)** 로도 보강 가능 —
API 뒤의 데이터를 pre-processing 단계에서 **테이블로 materialize** 한 뒤 `JOIN` 에 사용할 수도 있음.

```
[Figure 5-2 재현] 정적 보강 — 실시간 API vs materialized API
──────────────────────────────────────────────────────────────────────
 Real-time API enrichment
   [데이터 처리] ──(API 직접 호출)──► API
        └──► [처리된 데이터셋을 출력 테이블에 적재]

 Materialized API static data enrichment   ← 멱등성 신경 쓰면 이쪽
   [API 동기화] ──► [내부 저장소]
                        └──► [데이터 처리] ──► [출력 테이블에 적재]
──────────────────────────────────────────────────────────────────────
 멱등성을 고려하면 API 를 materialize 한 뒤 SCD 형태로 두는 쪽을 택할 것
 ⇒ 처리 로직을 리플레이해도 항상 같은 데이터셋을 사용 = 결과 동일
```

#### 고려사항 (Consequences)

구현 자체는 단순해 보이지만, 멱등성을 비롯한 함정이 있음.

- **Late data and consistency (지연 데이터·정합성)**
  - 이상적으로는 user 가 event 와 같은 속도로 진화해야 함(프로필 변경 직후 방문하면 최신 변경이 반영되어야).
    하지만 챕터 3에서 본 것처럼 **현실에서는 그렇지 않을 수 있음**.
  - 스트리밍 파이프라인의 latency 완화 → **Dynamic Joiner(#24)** 사용.
  - 배치 파이프라인은 **오케스트레이션** 으로 보강 데이터셋이 준비될 때까지 대기 (챕터 2의 **Readiness Marker** 활용).
- **Idempotency (멱등성)**
  - 배치 백필 시 **보강 데이터셋도 멱등** 이어야 하는지 자문할 것.
    그래야 한다면, 그리고 data provider 가 시간 기반 쿼리를 허용하지 않으면,
    보강 데이터셋을 **내 데이터 계층으로 가져와** 시간 측면을 직접 제어한 뒤 조인해야 함.
  - API 뒤에 숨은 외부 데이터셋이면 더 까다로움 — 이상적으론 시간 기반 쿼리가 가능해야 하나 불가능할 수 있음.
    내부 저장소에 temporality 를 더해 모든 보강 레코드를 거기에 기록(Figure 5-2)하는 게 해법.
  - 두 경우 모두 **SCD 사용이 좋은 선택**.

#### 구현 예시 (Examples)

**예시 1 — SCD type 2 테이블 정의 (Example 5-1)**

`start_date`/`end_date` 로 각 사용자 속성의 유효 구간을 표현:
```sql
CREATE TABLE dedp.users ( # ...
 id TEXT NOT NULL,
 login VARCHAR(45) NOT NULL,
 start_date TIMESTAMP NOT NULL DEFAULT NOW(),
 end_date TIMESTAMP NOT NULL DEFAULT '9999-12-31'::timestamp,  -- 현재 값은 먼 미래로
 PRIMARY KEY(id, start_date)
);

CREATE TABLE dedp.visits ( # ...
 visit_id CHAR(36) NOT NULL,
 event_time TIMESTAMP NOT NULL,
 PRIMARY KEY(visit_id, event_time)
);
```

**예시 2 — SCD type 2 조인 (Example 5-2)**

`event_time` 시점에 유효했던 user 행만 결합 (`NOW()` 자리에 실행 시각 등 임의 기준 사용 가능):
```sql
SELECT v.visit_id, v.event_time, v.page, u.id, u.login, u.email
FROM dedp.visits v JOIN dedp.users u ON u.id = v.user_id
  AND NOW() BETWEEN start_date AND end_date;   -- 기준 시각이 유효구간 안에 드는 user 행만
```

> `NOW()` 대신 data orchestrator 가 주는 **execution time** 을 쓰면, 파이프라인 실행에 묶인 **불변(immutable) 속성** 이 되어
> 멱등성에 도움이 됨. (SCD type 4 는 같은 쿼리라 책에서는 생략 — 현재값을 별도 테이블에 둔다는 점만 다름.)

**예시 3 — PySpark 스트림-배치 조인 (Example 5-3)**

스트리밍 visits 에 정적 devices 참조를 `left_outer` 로 결합:
```python
devices: DataFrame = spark.read.format('delta').load(...)          # 정적 참조 (table file format)
visits: DataFrame = (spark.readStream.format('kafka').load()...)   # 스트리밍

(visits.join(devices_table, [visits.device_type == devices.type,
   visits.device_version == devices.version], 'left_outer'))       # 매칭 없어도 OK(left)
```

```
[Example 5-3 의 함정 — 별도 lifecycle]
──────────────────────────────────────────────────────────────────────
  정적 dataset 과 스트리밍 job 은 생명주기가 다름 → 스트리밍 job 은 정적 dataset 갱신을 기다리지 않음
  ⚠ 정적 dataset 을 raw 포맷(JSON/CSV)으로 full rewrite 하는 중이라면:
       그 순간 스트리밍 job 이 읽으면 → ✗ join miss, 최악엔 "빈 참조 테이블" 과 조인
  ✓ table file format(Delta/Iceberg)이면 atomicity·consistency 보장:
       소비자가 uncommitted 파일을 읽지 않는 한 빈 테이블 조인 위험 없음
──────────────────────────────────────────────────────────────────────
```

**예시 4 — API 로 보강하는 PySpark writer (Example 5-4)**

레코드를 버퍼링하다 임계치에 도달하면 **고유 IP 목록으로 한 번에(bulk)** API 호출:
```python
class KafkaWriterWithEnricher:
 BUFFER_THRESHOLD = 100
# ...
 def process(self, row):
    if len(self.buffered_to_enrich) == self.BUFFER_THRESHOLD:
       self._enrich_ips()        # 버퍼가 차면 일괄 보강
       self._flush_records()
    else:
       self.buffered_to_enrich.append(row)

 def _enrich_ips(self):
    ips = (','.join(set(visit.ip for visit in self.buffered_to_enrich
       if visit.ip not in self.enriched_ips)))                   # 미보강 IP만 중복 제거
    fetched_ips = requests.get(f'http://localhost:8080/geolocation/fetch?ips={ips}',
                  headers={'Content-Type': 'application/json','Charset': 'UTF-8'})
    if fetched_ips.status_code == 200:
       mapped_ips = json.loads(fetched_ips.content)['mapped']
       self.enriched_ips.update(mapped_ips)
```

> bulk 호출은 데이터 보강에서 가장 비싼 연산인 **네트워크 throughput** 을 최적화함 — 레코드마다 호출하지 말 것.

> **트러블 로그** — SCD 없이 현재값만 담긴 참조 테이블을 백필 조인에 쓰면, 과거 사실이 현재 값으로 오염됨.
> 예: `users` 가 현재 이메일(c@...)만 들고 있는데 2024-06-10 시점 방문을 조인하면,
> 당시 이메일이 a@... 였는데도 결과에는 c@... 가 붙음 — 6월 마케팅 코호트 분석이 통째로 어긋남.
> 시간 민감 보강은 SCD type 2/4 로 유효 구간을 남기고, 조인 시 `event_time BETWEEN start_date AND end_date`
> 처럼 **이벤트 시점 기준** 으로 매칭할 것.

---

### 1-2. 패턴 #24: 동적 조이너 (Dynamic Joiner)

#### 상황 (Problem)

**책의 use case** — Static Joiner 로는 부족해진 users-to-visits:

- users-to-visits 에 Static Joiner 를 구현했지만 결과가 만족스럽지 않음.
- 매주 **수천 명의 신규 사용자** 가 유입되며 **프로필 변경(profile change)** 이 급증.
- 각 사용자 변경이 **스트리밍 브로커(CDC, Change Data Capture)** 에 등록됨
  → 보강된 데이터셋이 금세 **무의미(irrelevant)** 해지고 downstream 소비자에 문제 발생.
- **결정적 제약**: 이제 **양쪽 데이터셋이 모두 움직임(in motion, streaming)** → 정적 결합으로는 불가.

#### 해결 (Solution)

둘 다 움직이므로 Static Joiner 불가 → **Dynamic Joiner**.
Static Joiner 와 공유하는 부분(key 식별, join method 정의)에 더해 **time boundaries(시간 경계)** 라는 요구가 추가됨.

- 시간 관리 전략이 없으면 **조인 다수가 비어버림(empty join)** — 두 데이터셋의 latency 가 달라
  보강 데이터가 본 데이터보다 늦거나 빠르게 도착하기 때문. → time condition 으로 완화.
- **양쪽 스트림에 time-bounded buffer** 를 둠. 빠른 소스가 느린 소스와 시간 의미를 맞추도록,
  버퍼가 **추가 시간(allowed latency)** 을 벌어 매칭 기회를 줌
  (예: users 가 visits 보다 1시간 늦으면, visits 를 1시간 더 버퍼에 보관).
- 버퍼 무한 증가를 막기 위해 **GC(garbage collection) watermark** 도입
  — 너무 오래된 이벤트는 버퍼에서 제거. 늦게 온 레코드의 조인은 잃지만, **버퍼 크기를 관리** 하는 대가.
- 이 버퍼링 로직은 데이터 처리 프레임워크가 **네이티브로** 구현해 줌 (직접 짤 필요 없음).

```
[Figure 5-3 재현] Dynamic Joiner — latency 다른 두 스트림의 버퍼 조인
──────────────────────────────────────────────────────────────────────
   Buffer (Stream A)          Stream A        Stream B      Buffer (Stream B)
   Key=1 09:56                Key=10 10:20    Key=1 10:00   (비어 있음)
   Key=2 09:57                Key=11 10:21    Key=2 10:01
   Key=3 09:58                Key=12 10:23    Key=3 10:02
   Key=10 10:20 ◄─(A가 빠름, 매칭 대기)
   Key=11 10:21
   Key=12 10:23
                         │                        │
                         └──────────┬─────────────┘
                                    ▼ JOIN 결과
                    Key=1 09:56  Key=2 09:57  Key=3 09:58
                    Key=1 10:00  Key=2 10:01  Key=3 10:02
──────────────────────────────────────────────────────────────────────
 Stream A 가 B 보다 몇 분 빠름 → A 는 매칭 안 된 key 를 일정 시간 버퍼링.
 B 가 따라잡으면 incoming 또는 버퍼에서 대응 row 를 찾아 조인.
 매칭 없으면 GC watermark 가 B 의 가장 오래된 event time 보다 낡은 레코드를 제거.
```

#### 고려사항 (Consequences)

Static Joiner 의 큰 문제(정합성)를 해결하지만, Dynamic Joiner 만의 단점이 있음.

- **Space versus exactness trade-off (공간 대 정확도)**
  - GC watermark 와 time boundary 때문에 **가능한 모든 조인을 다 얻지는 못함**.
  - buffer space 를 늘리면 효율(매칭률)↑ 이지만 **하드웨어 비용↑**.
  - 줄이면 저장공간↓ 이지만 latency 차이가 크면 **매칭 가능성↓**.
  - **만능 공식은 없음** — 비즈니스 요구와 통상적 latency 차이에 맞춰 균형을 잡아야 함.
- **Late data (지연 데이터)**
  - 스트림 처리는 latency 가 낮은 만큼 late data 내성이 약함.
    예: users 스트림의 일시적 연결 문제로 일부 이벤트가 지연되면, **GC watermark 가 먼저 진행** 되어
    버퍼가 무효화되고 그 늦은 이벤트들은 무시됨.
  - 두 enrichment 패턴(#23·#24) **모두 추가 노력 없이는 조인 결과를 100% 보장하지 못함**
    → 한계 극복은 챕터 3 방식으로 **late data 를 추적·통합** 해야 함.

#### 구현 예시 (Examples)

**예시 1 — Apache Spark Structured Streaming 시간 기반 조인 (Example 5-5)**

양쪽에 `withWatermark` 를 주고, 조인 조건에 **시간 윈도우(BETWEEN)** 를 명시:
```python
visits_from_kafka: DataFrame = (visits_data_stream # ...
   .withWatermark('event_time', '10 minutes'))          # late 허용 10분

ads_from_kafka: DataFrame = (ads_data_stream # ...
   .withWatermark('display_time', '10 minutes'))

visits_with_ads = visits_from_kafka.join(ads_from_kafka, F.expr('''
    page = visit_page AND
    display_time BETWEEN event_time AND event_time + INTERVAL 2 minutes   -- 비즈니스+기술 규칙
   '''), 'left_outer')
```

> 조인 조건이 **비즈니스 의미**(광고는 방문 후 최대 2분 안에 노출)와 **기술 의미**(late data 여지)를 동시에 표현함.
> `withWatermark` 가 양쪽에서 최대 10분까지 늦은 레코드를 허용.

**예시 2 — Apache Flink temporal table join (Example 5-6, Java)**

Python API 가 temporal table join 을 지원하지 않아 Java 로 작성. 각 토픽에 watermark 를 주고
`TemporalTableFunction` 으로 **event_time 시점의 최신 광고** 를 매칭:
```java
tableEnv
 .createTemporaryTable("visits_tmp_table", TableDescriptor.forConnector("kafka")
 .schema(Schema.newBuilder().fromColumns(SchemaBuilders.forVisits())
    .watermark("event_time", "event_time - INTERVAL '5' MINUTES")
    .build()).option("topic", "visits")
// ...
TemporalTableFunction adsLookupFunction = adsTable.createTemporalTableFunction(
  $("update_time"), $("ad_page"));                          // update_time <= event_time 인 최신 광고
tableEnv.createTemporarySystemFunction("adsLookupFunction", adsLookupFunction);

Table joinResult = visitsTable.joinLateral(call("adsLookupFunction",
 $("event_time")), $("ad_page")).isEqual($("page")).select($("*"));
```

> `TemporalTableFunction` 이 각 페이지에 대해 **`update_time <= event_time` 인 가장 최근 광고** 를 가져옴
> — "방문 시점에 실제로 유효했던 광고" 를 정확히 매칭하는 역할.

| 항목 | Static Joiner (#23) | Dynamic Joiner (#24) |
|---|---|---|
| 데이터 성격 | 한쪽이 정적(at-rest) | 양쪽 모두 스트리밍(in motion) |
| 추가 요구 | (선택) SCD 시간 조건 | **time boundaries + 버퍼 + GC watermark** 필수 |
| 핵심 위험 | full rewrite 중 빈 테이블 조인 | latency 차이로 empty join, late data 누락 |
| 대표 구현 | SQL JOIN, materialized API | Spark watermark join, Flink temporal table join |

> **트러블 로그** — Dynamic Joiner 에서 watermark/버퍼 시간을 latency 차이보다 짧게 잡으면 조인이 조용히 비어버림.
> 예: users 스트림이 평소 visits 보다 30분 늦는데 `withWatermark('event_time', '10 minutes')` 로 두면,
> users 가 도착하기 전에 GC watermark 가 visits 버퍼를 비워 → 보강 컬럼이 전부 NULL 인 `left_outer` 결과만 쌓임.
> 에러도 안 나서 한참 뒤 대시보드 지표가 비어 보이고 나서야 발견됨.
> 운영 전에 두 스트림의 **실제 latency 분포** 를 측정해 watermark·버퍼 시간을 그보다 넉넉히 잡고,
> 끝내 못 맞춘 조인은 챕터 3의 late data 통합으로 별도 보정할 것.

---

## 2. 데이터 데코레이션 (Data Decoration)

Data Enrichment 로 가치를 더했다면, 그걸로 충분한가?
데이터는 현대 조직의 핵심 자산이지만 보강 시나리오에서도 여전히 **raw·비정형** 상태라
이해하고 활용하기 어려울 수 있음. 이때 **Data Decoration** 패턴이 도움이 됨.

- **#25 Wrapper** — record 에 추상화 한 겹을 씌워 **원본과 계산값을 분리**.
- (#26 Metadata Decorator — 계산값을 **메타데이터 레이어** 에 숨김. 본 문서 범위 밖.)

---

### 2-1. 패턴 #25: 래퍼 (Wrapper)

> 소프트웨어 엔지니어링에서 *wrapping* 은 객체에 **추가 동작/속성** 을 붙이는 것.
> 데이터 세계에서도 동일하게, **원본(original) 부분과 변환(transformed) 부분을 분리** 하는 데 쓰임.

#### 상황 (Problem)

**책의 use case** — 서로 다른 스키마로 들어오는 방문 데이터:

- 스트리밍 계층이 visits 데이터를 처리하는데, visits 가 **여러 provider** 에서 와서 **출력 스키마가 제각각**.
- 서로 다른 field 를 추출해 **한 곳에 모으는** 잡이 필요. downstream 소비자가 쉽게 의존할 수 있도록.
- 단, **계산값(computed)을 원본값(original)과 명확히 분리** 해 처리 로직을 단순화하면서도,
  디버깅을 위해 **원본 구조는 그대로 보존** 해야 함.
- **결정적 제약**: 원본 레코드를 **건드리지 않고(무손실)** 유지해야 함.

#### 해결 (Solution)

원본 보존 요구 때문에 단순히 row 를 parse 해 새 구조를 만들 수 없음(초기값 손실). → **Wrapper**.

- 아이디어: **record 레벨에 추상화 한 겹** 을 추가. envelope(봉투)이 원본값을 high-level 로 감싸고,
  거기에 **계산 속성(computed)** — 입력 자체에서 나오거나 실행 문맥(processing time, job version)에서 나오는 — 을 함께 참조.
  예: 방문 이벤트 → **`raw` + `computed`** 로 구성된 이벤트로 변환.
- 스트리밍 포맷(Apache Avro, Protobuf, JSON) 외 **정형 포맷(table)** 도 지원 —
  필요 시 **별도 테이블로 join** 하거나 **같은 테이블의 extra column** 으로 구현.

```
[Figure 5-4 재현] 정형 데이터를 위한 Wrapper 4가지 구현
──────────────────────────────────────────────────────────────────────
 구현 1  [ Original row struct<...> ][ Computed col 1 ] ... [ Computed col n ]
         → 원본을 nested struct, 계산값을 flat 컬럼

 구현 2  [ Original col 1 ] ... [ Original col n ][ Computed row struct<...> ]
         → 원본을 flat 컬럼, 계산값을 nested struct (구현 1의 반대)

 구현 3  [ Original col 1 ] ... [ Original col n ][ Computed col 1 ] ... [ Computed col n ]
         → 모두 같은 레벨의 flat 컬럼

 구현 4  Table 1: [ Original col 1 ] ... [ Original col n ]
         Table 2: [ Computed col 1 ] ... [ Computed col n ]   (unique key 로 나중에 join)
──────────────────────────────────────────────────────────────────────
```

- **구현 1·2 = 비정규화(denormalization)** — 읽기가 빠를 수 있음.
- **구현 4(두 테이블) = 정규화(normalized)** — 읽기는 느릴 수 있으나, 데이터셋을 **논리적으로 분리** 하거나
  원본 구조를 **바꿀 수 없을 때** 더 나은 선택.
- 네 방식 모두 **스키마 관리** 가 필요 (챕터 9의 schema consistency 패턴과 연결).

> **참고 사항 — A Wrapper in a Table? (테이블 안의 래퍼)**
> 정형 포맷에서는 envelope 이 직접 보이지 않지만 **여전히 존재** 함 — envelope 이 곧 테이블의 **한 row** 임.
> 따라서 일반 columnar 포맷을 깨지 않고도, 예컨대 여러 컬럼의 field 를 **nested attribute 를 가진 한 컬럼** 으로 묶을 수 있음.

#### 고려사항 (Consequences)

논리적·물리적 분리가 데이터셋에 꽤 중요한 결과를 가져옴 — 주로 **저장 공간(storage footprint)** 측면.

- **Domain split (도메인 분리)**
  - 논리적 함의 — 한 도메인의 속성을 raw/computed 로 쪼갬.
    예: user 에 Wrapper 적용 시 user 관련 field 가 **raw 와 computed 두 high-level 구조** 에 흩어짐.
  - 변환/비변환 구분이 명확해지는 장점이 있지만, **데이터 조회가 복잡** 해짐 — 소비자가 두 위치를 알아야 함.
  - 트레이드오프 — wrapped 데이터를 시스템의 **첫 저장 계층(예: Silver layer)** 에 속한 것으로 보고,
    사용자에게 최종 노출하는 데이터는 아니게 두면 혼란을 줄일 수 있음.
- **Size (크기)**
  - decorated 값은 처리된 레코드의 **본질적 일부** 라 전체 크기와 네트워크 트래픽에 영향
    (메타데이터 레이어에만 두는 #26 Metadata Decorator 와 다른 점).
  - 저장 포맷이 **data source projection(컬럼 프루닝)** 을 지원하면 완화 가능 —
    관심 컬럼만 물리적으로 접근. columnar storage(AWS Redshift, GCP BigQuery)에서 흔한 방식.

#### 구현 예시 (Examples)

**예시 1 — PySpark 로 메타데이터 래핑 (Example 5-7)**

실행 문맥(job version, batch number)을 struct 로 묶고, 원본값과 함께 JSON 으로 감쌈:
```python
visits_w_processing_context = (visits.withColumn('processing_context', F.struct(
   F.lit(job_version).alias('job_version'), F.lit(batch_number).alias('batch_version')
)))

visits_to_save = (visits_w_processing_context.withColumn('value', F.to_json(
   F.struct(F.col('value').cast('string').alias('raw_data'),   # 원본 = raw_data
   F.col('processing_context')))))                             # 계산/문맥 = processing_context
```

> 메타데이터를 새 컬럼으로 정의한 뒤, 초기값(`raw_data`)과 기술 문맥(`processing_context`)을
> 하나의 struct 로 묶어 JSON 으로 출력 DB 에 기록.

**예시 2 — SQL 로 원본은 그대로, 계산값을 struct 로 (Example 5-8)**

`SELECT *` 로 원본 컬럼을 모두 두고, 계산값만 `NAMED_STRUCT` 로 묶어 `decorated` 컬럼에:
```sql
SELECT *, NAMED_STRUCT(
  'is_connected',
  CASE WHEN context.user.connected_since IS NULL
     THEN false ELSE true END,
  'page_referral_key', CONCAT_WS('-', page, context.referral)
) AS decorated FROM input_visits
```

> `NAMED_STRUCT` 는 `key1, value1, key2, value2, ...` 형식으로 struct 의 키 이름을 직접 지정하는 편리한 함수.

**예시 3 — SQL 로 계산값을 1급 컬럼, 원본을 struct 로 (Example 5-9, 5-8의 반대)**

계산값을 테이블 컬럼으로 승격하고, 원본값을 `raw` struct 로 격리:
```sql
SELECT
  CASE WHEN context.user.connected_since IS NULL
     THEN false ELSE true END AS is_connected,
  CONCAT_WS('-', page, context.referral) AS page_referral_key,
  STRUCT(visit_id, event_time, user_id, page, context) AS raw   -- 원본은 raw struct 로
FROM input_visits
```

> ⚠ 이 방식은 원본/계산값 컬럼 이름이 명확히 구분되지 않으면 **소비자가 둘을 헷갈릴 수 있음** — 명명 규칙에 주의.

```
[Wrapper 핵심 — 한 record 안의 raw / computed 분리]
──────────────────────────────────────────────────────────────────────
  입력 visit 이벤트
        │  Wrapper 적용
        ▼
  ┌──────────────────────── envelope (= 한 row) ───────────────────────┐
  │  raw       : { visit_id, event_time, user_id, page, context, ... } │ ← 디버깅용 원본 보존
  │  computed  : { is_connected, page_referral_key, job_version, ... } │ ← 변환·문맥값
  └────────────────────────────────────────────────────────────────────┘
  ⇒ 처리 로직은 computed 만 보면 되고, 문제 추적 시 raw 로 원본 복원 가능
──────────────────────────────────────────────────────────────────────
```

> **트러블 로그** — Wrapper 로 raw/computed 를 나눠 놓고 소비자에게 "두 곳에 흩어져 있다" 는 사실을 안 알리면 조회가 깨짐.
> 예: 구현 4(두 테이블)로 원본은 `visits_raw`, 계산값은 `visits_computed` 에 두고 `visit_id` 로 join 하게 설계했는데,
> 분석가가 `visits_computed` 만 조회해 `user_id` 를 찾다 없어서 "데이터 유실" 로 잘못 신고하는 식.
> 또 구현 1·2처럼 nested struct 로 감싸면, projection 미지원 포맷에서는 한 컬럼만 필요해도 전체 struct 를 읽어 I/O 가 늘 수 있음.
> Wrapper 의 **저장 위치/구조(어디가 raw·어디가 computed 인지)** 를 스키마 문서로 명시하고,
> columnar 포맷이면 **data source projection(컬럼 프루닝)** 을 켜서 필요한 컬럼만 읽도록 할 것.
