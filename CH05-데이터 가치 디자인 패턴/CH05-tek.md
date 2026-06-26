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
   - 패턴 #26: 메타데이터 데코레이터 (Metadata Decorator)
3. [데이터 집계 (Data Aggregation)](#3-데이터-집계-data-aggregation)
   - 패턴 #27: 분산 집계기 (Distributed Aggregator)
   - 패턴 #28: 로컬 집계기 (Local Aggregator)
4. [세션화 (Sessionization)](#4-세션화-sessionization)
   - 패턴 #29: 증분 세션화 처리기 (Incremental Sessionizer)
   - 패턴 #30: 상태 저장 세션화 처리기 (Stateful Sessionizer)
5. [데이터 정렬 (Data Ordering)](#5-데이터-정렬-data-ordering)
   - 패턴 #31: 빈 팩 정렬기 (Bin Pack Orderer)
   - 패턴 #32: 선입 선출 정렬기 (FIFO Orderer)
6. [요약 (Summary)](#6-요약-summary)

> 본 문서가 챕터 5 **전체(#23~#32)** 를 다룸 — 5.1 Data Enrichment · 5.2 Data Decoration · 5.3 Aggregation · 5.4 Sessionization · 5.5 Data Ordering.
> 챕터 6(데이터 흐름 #33~)은 별도 문서에서 다룸.

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
 본 문서: 챕터 5 전체 (#23 ~ #32)
```

| 패밀리 | 적용 상황 (언제 쓰나) | 핵심 동작 |
|---|---|---|
| **5.1 Data Enrichment** (#23, #24) | raw 데이터에 **다른 데이터셋의 context** 가 없을 때 | 두 데이터셋을 key 로 **JOIN** 해 보강 (#23 정적 / #24 양쪽 스트리밍) |
| **5.2 Data Decoration** (#25, #26) | 데이터가 raw·비정형이라 **추가 가공** 이 필요할 때 | record 에 계산값을 **부착·분리** (#25 Wrapper / #26 Metadata Decorator) |
| **5.3 Data Aggregation** (#27, #28) | 대량 데이터의 **개요(overview)** 가 필요할 때 | 분산/로컬로 **요약** (#27 Distributed: 셔플 기반 / #28 Local: 부분집계 후 병합) |
| **5.4 Sessionization** (#29, #30) | 흩어진 이벤트를 **세션(활동 단위)으로 묶어야** 할 때 | 비활동 gap 으로 세션 경계 (#29 배치 Pending / #30 스트리밍 state store + watermark) |
| **5.5 Data Ordering** (#31, #32) | 다운스트림에 **시간순 전달** 이 중요할 때 | 순서 보존 전달 (#31 Bin Pack: partial commit+bulk / #32 FIFO: ack 받고 하나씩) |

> 참고
> — Data Enrichment(결합)·Data Decoration(개별 속성 계산)은 **소량/문맥 보강** 에 강하지만,
> 대량 데이터의 **개요(overview)** 가 필요하면 Aggregation·Sessionization 으로 넘어가야 함.

### 패턴 흐름 — 앞 패턴의 한계가 다음 패턴을 부른다 (#23 → #33)

강화부터 정렬, 그리고 챕터 6 진입까지 **"앞 패턴이 남긴 문제 → 다음 패턴"** 으로 이어지는 narrative.
패밀리 **안** 의 전환은 "한 패턴의 한계 → 변형", 패밀리 **사이** 의 전환은 "다른 니즈로 전환".

```
[패턴 흐름 — 한 패턴의 한계가 다음 패턴을 부른다]
──────────────────────────────────────────────────────────────────────
 강화 (5.1)  ── "다른 데이터셋의 문맥이 없다"
   #23 Static Joiner — 정적 보강 데이터셋과 JOIN (가장 쉬움)
      │ 한계: 보강 데이터가 snapshot → 시간이 지나면 stale → 정합성 깨짐
      ▼ "보강 쪽도 실시간으로 바뀐다면"
   #24 Dynamic Joiner — 양쪽 스트리밍 결합 (정합성 해결)
      │
      ▼ "외부 결합 말고, record 자체를 가공·정리해야"  (패밀리 전환)
 데코레이션 (5.2)
   #25 Wrapper — 원본과 계산값을 한 겹 씌워 분리
      │ 한계: 계산값이 컨슈머에 그대로 노출돼 오해 유발
      ▼ "이 부가 정보를 컨슈머에게서 숨기려면"
   #26 Metadata Decorator — 계산값을 메타데이터 레이어로 숨김
      │
      ▼ "소량 문맥 말고, 대량 데이터의 개요가 필요"  (패밀리 전환)
 집계 (5.3)
   #27 Distributed Aggregator — 흩어진 데이터를 셔플로 모아 집계
      │ 한계: 셔플 네트워크 비용이 큼
      ▼ "데이터가 이미 잘 분할돼 있다면 셔플을 없애자"
   #28 Local Aggregator — 셔플 없이 로컬 집계 (대신 확장은 어려움)
      │
      ▼ "숫자 요약 말고, 이벤트를 활동 단위로 묶어야"  (패밀리 전환)
 세션화 (5.4)
   #29 Incremental Sessionizer (배치) — 시간별 파티션을 Pending 으로 이어 붙여 세션 완성
      │ 한계: 배치라 결과가 ~1시간 늦음
      ▼ "더 빨리(near real-time) 필요"
   #30 Stateful Sessionizer (스트리밍) — Pending 대신 state store 로 실시간 세션
      │
      ▼ "이번엔 이벤트를 순서대로 전달해야"
 정렬 (5.5)
   #31 Bin Pack Orderer — partial commit + bulk 에서도 순서 보존 (bin 으로 묶어 순차 flush)
      │ 한계: 늘 low latency·대용량인 건 아님
      ▼ "더 단순하게, ASAP"
   #32 FIFO Orderer — record 하나씩 ack 받고 전달 (네트워크 효율은 포기)
      │
      ▼ "그런데 이 잡들을 어떻게 다 엮지?"  → 챕터 6
 데이터 흐름 (Ch6)
   #33 Local Sequencer (별도 문서 CH06) — 큰 잡을 순서 있는 작은 태스크로 분해
──────────────────────────────────────────────────────────────────────
 핵심 — 각 패턴은 "앞 패턴이 남긴 한계"를 푸는 방향으로 등장함.
```

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

코드 구현은 보통 SQL `JOIN` 으로 표현되며, 현대 처리 프레임워크와 고전적 DW 모두에서 동작하는 **범용** 해법.
선언적 SQL 외에 **프로그래밍 API(HTTP 라이브러리)** 로도 보강 가능
— API 뒤의 데이터를 pre-processing 단계에서 **테이블로 materialize** 한 뒤 `JOIN` 에 사용할 수도 있음.

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
       컨슈머 (읽는 쪽)가 uncommitted 파일을 읽지 않는 한 빈 테이블 조인 위험 없음
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
  → 보강된 데이터셋이 금세 **무의미(irrelevant)** 해지고 컨슈머 (다운스트림)에 문제 발생.
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
- **#26 Metadata Decorator** — 계산값을 record 본문이 아닌 **메타데이터 레이어** 에 숨겨, 컨슈머 (데이터를 받아 쓰는 다운스트림)에 노출하지 않음.

---

### 2-1. 패턴 #25: 래퍼 (Wrapper)

> 소프트웨어 엔지니어링에서 *wrapping* 은 객체에 **추가 동작/속성** 을 붙이는 것.
> 데이터 세계에서도 동일하게, **원본(original) 부분과 변환(transformed) 부분을 분리** 하는 데 쓰임.

#### 상황 (Problem)

**책의 use case** — 서로 다른 스키마로 들어오는 방문 데이터:

- 스트리밍 계층이 visits 데이터를 처리하는데, visits 가 **여러 provider** 에서 와서 **출력 스키마가 제각각**.
- 서로 다른 field 를 추출해 **한 곳에 모으는** 잡이 필요. 컨슈머 (다운스트림)가 쉽게 의존할 수 있도록.
- 단, **계산값(computed)을 원본값(original)과 명확히 분리** 해 처리 로직을 단순화하면서도,
  디버깅을 위해 **원본 구조는 그대로 보존** 해야 함.
- **결정적 제약**: 원본 레코드를 **건드리지 않고(무손실)** 유지해야 함.

#### 해결 (Solution)

원본 보존 요구 때문에 단순히 row 를 parse 해 새 구조를 만들 수 없음(초기값 손실). → **Wrapper**.

- 아이디어: **record 레벨에 추상화 한 겹** 을 추가. envelope(봉투)이 원본값을 high-level 로 감싸고,
  거기에 **계산 속성(computed)** — 입력 자체에서 나오거나 실행 문맥(processing time, job version)에서 나오는 — 을 함께 참조.
  예: 방문 이벤트 → **`raw` + `computed`** 로 구성된 이벤트로 변환.
- 스트리밍 포맷(Apache Avro, Protobuf, JSON) 외 **정형 포맷(table)** 도 지원
  — 필요 시 **별도 테이블로 join** 하거나 **같은 테이블의 extra column** 으로 구현.

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
  - 변환/비변환 구분이 명확해지는 장점이 있지만, **데이터 조회가 복잡** 해짐 — 컨슈머 (조회하는 쪽)가 두 위치를 알아야 함.
  - 트레이드오프 — wrapped 데이터를 시스템의 **첫 저장 계층(예: Silver layer)** 에 속한 것으로 보고,
    사용자에게 최종 노출하는 데이터는 아니게 두면 혼란을 줄일 수 있음.
- **Size (크기)**
  - decorated 값은 처리된 레코드의 **본질적 일부** 라 전체 크기와 네트워크 트래픽에 영향
    (메타데이터 레이어에만 두는 #26 Metadata Decorator 와 다른 점).
  - 저장 포맷이 **data source projection(컬럼 프루닝)** 을 지원하면 완화 가능
    — 관심 컬럼만 물리적으로 접근. columnar storage(AWS Redshift, GCP BigQuery)에서 흔한 방식.

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

> ⚠ 이 방식은 원본/계산값 컬럼 이름이 명확히 구분되지 않으면 **컨슈머 (사용하는 쪽)가 둘을 헷갈릴 수 있음** — 명명 규칙에 주의.

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

> **트러블 로그** — Wrapper 로 raw/computed 를 나눠 놓고 컨슈머 (데이터를 받아 쓰는 쪽)에게 "두 곳에 흩어져 있다" 는 사실을 안 알리면 조회가 깨짐.
> 예: 구현 4(두 테이블)로 원본은 `visits_raw`, 계산값은 `visits_computed` 에 두고 `visit_id` 로 join 하게 설계했는데,
> 분석가가 `visits_computed` 만 조회해 `user_id` 를 찾다 없어서 "데이터 유실" 로 잘못 신고하는 식.
> 또 구현 1·2처럼 nested struct 로 감싸면, projection 미지원 포맷에서는 한 컬럼만 필요해도 전체 struct 를 읽어 I/O 가 늘 수 있음.
> Wrapper 의 **저장 위치/구조(어디가 raw·어디가 computed 인지)** 를 스키마 문서로 명시하고,
> columnar 포맷이면 **data source projection(컬럼 프루닝)** 을 켜서 필요한 컬럼만 읽도록 할 것.

---

### 2-2. 패턴 #26: 메타데이터 데코레이터 (Metadata Decorator)

> Wrapper 는 record 를 감싸는 보편적 패턴이지만, 더해진 값이 컨슈머 (데이터를 받아 쓰는 다운스트림)에게
> **직접 노출되면 오해를 부를 수 있음**. 컨슈머는 그 record 가 job 버전 1 에서 나왔는지 2 에서 나왔는지 신경 쓰지 않음.
> 이 문제를 피하려면 추가 record 를 데이터 저장소의 **메타데이터 레이어** 에 숨기면 됨.

#### 상황 (Problem)

**책의 use case** — 자주 배포되는 스트리밍 잡의 기술 문맥 추적:

- 스트리밍 잡이 꽤 자주 진화해서 **거의 매주 한 번** 새 버전을 릴리스함.
- 배포 프로세스는 매끄럽지만, 릴리스된 버전이 생성 데이터에 **어떤 영향을 줬는지 가시성이 부족**함.
- 유지보수를 단순화하려면 각 생성 record 에 **기술 문맥(job 버전 등)을 부착** 해야 함.
- **결정적 제약**: 그 정보를 **최종 사용자에게 보내는 record 에는 포함하고 싶지 않음**
  — 내부 데이터 처리 세부사항이라 컨슈머 (다운스트림)에게는 무관함.

#### 해결 (Solution)

기술 문맥을 record 안에 Wrapper 로 넣는 건 선택지가 아님(컨슈머에게 무관·오해 소지).
→ 데이터 저장소의 **메타데이터 레이어** 를 활용하는 **Metadata Decorator**.

- 구현은 저장소의 **메타데이터 처리 능력** 에 따라 달라짐.
  메타데이터를 기본 지원하면, 기록되는 각 record 에 **전용 메타데이터 속성** 을 붙일 수 있음.
  이 속성이 data producer 의 네이티브 기능이라 구현이 비교적 단순함.
  **Apache Kafka** 가 그런 경우 — key·value 외에 record 마다 **optional header(key-value 목록)** 를 지원.
- **오브젝트 스토어** — 메타데이터 데코레이션이 한 파일의 모든 row 에 적용되면,
  메타데이터 속성을 그 파일에 붙는 **태그(tag)** 로 정의.
- 태그도 메타데이터도 네이티브 지원이 없는 저장소(관계형·NoSQL)라면, 데이터 영역 안에 메타데이터를 넣되
  **외부에 공개하지 않는** 방식으로 데코레이션을 흉내 냄. DW 테이블 예 — 전용 컬럼에 기록한 뒤
  ① 그 컬럼을 뺀 **view 로 노출** 하거나 ② **권한(privilege)으로 비기술 사용자의 읽기를 차단**
  (챕터 7의 **Fine-Grained Accessor for Tables** 패턴과 연결).
- **대안 구현** — 처리 문맥을 **별도 테이블** 에 두고 데이터셋과 join.
  이쪽이 더 단순함 — 그 테이블을 **기술 팀만 접근 가능한 스키마** 에 숨기면 됨.
  메타데이터가 별도 테이블로 **정규화** 되므로 row 마다 중복 저장되지 않음.

```
[Table 5-1] 메타데이터 컬럼을 가진 테이블 — view 나 권한으로 컨슈머 접근을 통제
──────────────────────────────────────────────────────────────────────
 ┌──────────┬────────────────────────┬─────┬───────────────────────────────────────────────────────────┐
 │ event_id │ event_time             │ ... │ processing_context                                        │
 ├──────────┼────────────────────────┼─────┼───────────────────────────────────────────────────────────┤
 │    1     │ 2023-06-10T10:00:59Z   │ ... │ {"job_version":"v1.0.3","processing_time":"2023-06-10T..."}│
 └──────────┴────────────────────────┴─────┴───────────────────────────────────────────────────────────┘
 → 메타데이터를 한 컬럼(processing_context)에 JSON 으로 넣되, 컨슈머에겐 그 컬럼을 숨김
──────────────────────────────────────────────────────────────────────
[Table 5-2 / 5-3] 처리 문맥 테이블 방식 — 메타데이터를 별도 테이블로 정규화
──────────────────────────────────────────────────────────────────────
 (data table)                                  (technical table)
 ┌──────────┬──────────────────────┬───────────────────────┐  ┌───────────────────────┬─────────────┬──────────────────────┐
 │ event_id │ event_time           │ processing_context_id │  │ processing_context_id │ job_version │ processing_time      │
 ├──────────┼──────────────────────┼───────────────────────┤  ├───────────────────────┼─────────────┼──────────────────────┤
 │    1     │ 2023-06-10T10:00:59Z │          1            │  │          1            │   v1.0.3    │ 2023-06-10T10:02:00  │
 └──────────┴──────────────────────┴───────────────────────┘  └───────────────────────┴─────────────┴──────────────────────┘
 → 문맥이 row 마다 중복되지 않음. 컨슈머는 technical table 도 processing_context_id 도 보지 못하게 막음
──────────────────────────────────────────────────────────────────────
```

> **참고 사항 — Wrapper 와 Metadata 의 의미 차이**
> Metadata Decorator 는 Wrapper 와 닮았지만 **의미(semantics)가 다름**.
> 메타데이터는 정의상 "데이터에 대한 데이터" 라 **비즈니스 사용자에게 공개되어선 안 됨**.
> 반면 Wrapper 는 기술 속성에 더해 **비즈니스 속성까지 함께 장식** 하려는 의도라 노출을 전제로 함.

#### 고려사항 (Consequences)

저장소의 메타데이터 지원 여부가 가장 큰 제약이지만, 그것만은 아님.

- **Implementation (구현 난이도)**
  - 상황에서 가정한 스트리밍 브로커조차 **네이티브 메타데이터 지원이 없을 수 있음**
    — 예: **Amazon Kinesis Data Streams 는 header 를 지원하지 않음** → 구현 자체가 불가.
  - 테이블 데이터셋도 까다로움 — 메타데이터를 담을 **추가 컬럼/테이블** 을 정의해야 하는 경우가 잦음.
    동작은 하지만, 네이티브 지원 저장소보다 **품이 더 듦**.
- **Data (메타데이터에 무엇을 넣을지)**
  - 기술적으로는 무엇이든 넣을 수 있으나, **비즈니스 속성(배송 주소·청구 금액 등)은 넣지 말 것**.
    그러면 컨슈머에게 숨겨지는데, 대부분은 메타데이터 영역을 쿼리할 생각조차 하지 않음.
  - 정의상 메타데이터는 "데이터에 대한 데이터" — 외부 사용자는 결국 **데이터 본문** 에만 관심이 있음.

#### 구현 예시 (Examples)

**예시 1 — Apache Kafka 메타데이터 header 추가 (Example 5-10, PySpark)**

메타데이터를 `key`·`value` struct 의 **배열** 로 만들고, `includeHeaders` 로 header 컬럼을 함께 기록:
```python
visits_with_metadata = (visits_to_save.withColumn('headers', F.array(
   F.struct(F.lit('job_version').alias('key'), F.lit(job_version).alias('value')),
   F.struct(F.lit('batch_version').alias('key'),
   F.lit(str(batch_number).encode('UTF-8')).alias('value'))   # value 는 byte 로 인코딩
)))
(visits_with_metadata.write.format('kafka')
 .option('kafka.bootstrap.servers', 'localhost:9094')
 .option('includeHeaders', True).option('topic', 'visits-decorated')   # header 포함 기록
.save())
```

> 두 가지가 핵심 — ① 메타데이터는 **key-value 쌍의 배열**, ② `includeHeaders` 옵션이
> Spark 에게 생성 record 에 header 컬럼을 포함하라고 지시함.

**예시 2 — 외부 메타데이터 테이블 초기화 (Example 5-11, SQL)**

처리 문맥을 별도 테이블로 두는 방식. 이후 join 으로 결합:
```sql
CREATE TABLE dedp.visits_context (
    execution_date_time TIMESTAMPTZ NOT NULL,
    loading_time TIMESTAMPTZ NOT NULL,
    code_version VARCHAR(15) NOT NULL,
    loading_attempt SMALLINT NOT NULL,
    PRIMARY KEY (execution_date_time)
)
```

**예시 3 — 메타데이터 테이블로 새 visits 적재 (Example 5-12, SQL)**

먼저 실행 문맥을 `visits_context` 에 INSERT 한 뒤, 그 PK 를 주간 테이블에 함께 적재:
```sql
{% set weekly_table = get_weekly_table_name(execution_date) %}
INSERT INTO dedp.visits_context
 (execution_date_time, loading_time, code_version, loading_attempt)
VALUES ('{{ execution_date }}', '{{ dag_run.start_date }}',
 '{{ params.code_version }}', {{ task_instance.try_number }});   -- Airflow 실행 문맥

INSERT INTO {{ weekly_table }} (SELECT tmp_devices.*,
   '{{ execution_date }}' AS visits_context_execution_date_time FROM tmp_devices);
```

| 구현 방식 | 어디에 메타데이터를 두나 | 컨슈머 차단 방법 | 주의사항 |
|---|---|---|---|
| 네이티브 header/tag | 브로커 header, 파일 tag | 컨슈머가 안 읽으면 됨 | 브로커가 지원해야 함(Kinesis 미지원) |
| 같은 테이블의 전용 컬럼 | 데이터 테이블의 한 컬럼 | view 노출 / 컬럼 읽기 권한 차단 | row 마다 중복 저장 |
| 별도 정규화 테이블 | 기술 팀 전용 스키마의 테이블 | 스키마 접근 권한 제한 | join 필요, 중복은 없음 |

> **트러블 로그** — 메타데이터 레이어에 **비즈니스 값** 을 넣으면 컨슈머가 영영 못 찾음.
> 예: 결제 record 의 `invoice_amount` 를 "부가 정보" 라며 Kafka header 에 넣었더니,
> 다운스트림 정산 잡은 value 본문만 파싱해 금액 컬럼이 통째로 누락 — 월 마감 집계가 0 원으로 찍힘.
> 또 Kinesis 처럼 header 미지원 브로커에서 같은 설계를 재사용하면 적재 단계에서 조용히 메타데이터가 사라짐.
> 메타데이터에는 **job_version·processing_time 같은 기술 문맥만** 넣고, 비즈니스 값은 데이터 본문(value/컬럼)에 둘 것.

---

## 3. 데이터 집계 (Data Aggregation)

지금까지는 데이터에 정보를 **"더해서"** 가치를 만들었음.
그런데 정보를 **"덜어내는(요약하는)"** 것도 가치를 만드는 방법 — 대량 데이터를 한눈에 보게 요약하는 집계가 그것.

- **#27 Distributed Aggregator** — 데이터가 **여러 머신에 흩어져** 있어 네트워크로 모아 집계해야 할 때.
- **#28 Local Aggregator** — 데이터가 **이미 올바르게 분할** 돼 있거나 단일 머신에 들어가, 네트워크 교환 없이 집계할 때.

---

### 3-1. 패턴 #27: 분산 집계기 (Distributed Aggregator)

> 분산 데이터 처리 프레임워크의 강점 중 하나는 **물리적으로 떨어져 있지만 논리적으로 같은 종류의 item 을 결합** 하는 능력.

#### 상황 (Problem)

**책의 use case** — Silver 레이어 visits 로 OLAP 큐브 만들기:

- raw visit 이벤트를 Bronze 레이어에서 정제해 Silver 레이어에 기록하는 잡을 작성함.
  거기서 여러 컨슈머 (후속 파이프라인)가 각자의 비즈니스 use case 를 구현함.
- 그중 하나가 **OLAP(online analytical processing) 큐브** 구축
  — 모든 visit 을 대시보드에 적합한 **집계 포맷** 으로 줄임.
  결과에는 여러 축(사용자 지역·디바이스 등)에 걸친 **기본 통계(count, 평균 체류시간 등)** 가 들어가야 함.
- 데이터셋은 **일별 event time 파티션** 으로 저장되고, 큐브는 **일간·주간 뷰** 를 표현해야 함.
- **결정적 제약**: 관련 record 가 **여러 물리적 위치에 분산** 돼 있어 단일 머신으로는 집계 불가.

#### 해결 (Solution)

단일 머신에 들어가는 데이터라면 언어의 group by + reduce 로 충분하지만,
빅데이터에서는 관련 record 가 여러 물리적 위치에 쪼개짐. → **Distributed Aggregator**.

- 여러 머신이 모여 하나의 실행 단위인 **클러스터(cluster)** 를 이룸.
  서버 각각은 전체를 처리할 용량이 없지만, 함께 일을 나눠 처리함.
- 하드웨어 차이에도 **코드 구현은 로컬 처리와 동일할 수 있음** — grouping 함수로 관련 row 를 모으고 reduce 적용.
  grouping 은 선택 사항 — 전체를 그대로 합칠 수도 있음(전체 row count, 전역 평균 등).
- 하지만 핵심은 내부 동작에 있음. API 는 같아 보여도, 내부적으로는 서로 다른 머신에 적재된 record 를
  **네트워크로 교환** 하는 단계가 끼어듦 — 그래야 reduce 가 필요한 row 를 한곳에 모아 처리 가능(Figure 5-5).
  이 동작을 **셔플(shuffle)** 이라 부르며, **네트워크 트래픽 비용** 때문에 첫손에 꼽히는 latency 유발원.
- 모든 record 교환이 raw 인 건 아님. **부분 집계(partial aggregation)** 가 가능한 연산은
  셔플 전에 **로컬에서 부분 집계** 해 교환량을 줄일 수 있음.
  대표 예가 count — raw record 를 다 셔플해 최종 노드에서 세는 대신,
  각 초기 노드에서 로컬 key 별로 **부분 count** 를 내고 그 **숫자만 셔플** 해 마지막에 합산.

```
[Figure 5-5 재현] Distributed Aggregator 의 데이터 교환(shuffle)
──────────────────────────────────────────────────────────────────────
 Input file 1          Compute node 1            (shuffle)           Compute node 1
 ┌────────┐            ┌─────────────┐                              ┌─────────────┐
 │ Key=1  │ ──────────►│ Key=1       │                              │ Key=1       │
 │ Key=2  │            │ Key=2       │      Key=3 ─┐    ┌─ Key=1    │ Key=1 ◄─────┼─ node2 에서 옴
 │ Key=3  │            │ Key=3 ──────┼─────────────┼────┼────────►  │ Key=2       │
 └────────┘            └─────────────┘             │    │           └─────────────┘
 Input file 2          Compute node 2                X(교차)          Compute node 2
 ┌────────┐            ┌─────────────┐             │    │           ┌─────────────┐
 │ Key=1  │ ──────────►│ Key=1 ──────┼─────────────┘    └────────►  │ Key=3 ◄─────┼─ node1 에서 옴
 │ Key=6  │            │ Key=6       │                              │ Key=6       │
 │ Key=7  │            │ Key=7       │                              │ Key=7       │
 └────────┘            └─────────────┘                              └─────────────┘
 같은 key 가 서로 다른 노드에 흩어져 있음 → shuffle 로 같은 key 를 한 노드에 collocate 한 뒤 reduce
──────────────────────────────────────────────────────────────────────
```

> **참고 사항 — MapReduce**
> Distributed Aggregator 는 2004년 분산 데이터 처리를 크게 단순화한 **MapReduce** 프로그래밍 모델의 전형적 예.
> 구현은 디스크 기반 Hadoop MapReduce 에서 메모리 우선 Apache Spark 로 진화해 옴.

#### 고려사항 (Consequences)

프레임워크가 패턴을 완전히 구현해 주지만, **데이터 자체** 와 얽힌 함정이 있음.

- **Additional network exchange (추가 네트워크 교환)**
  - 이 패턴은 **두 번의 네트워크 교환** 을 수반함.
    첫째는 입력 데이터를 각 노드로 가져오는 것(요즘 storage·compute 분리 환경에선 피하기 어려움).
    둘째가 바로 Distributed Aggregator 의 셔플 — 관련 데이터를 같은 서버에 모으는 필수 단계.
  - 입력 교환과 달리 이 **셔플은 특정 조건에서 회피 가능** — 다음 절의 Local Aggregator 에서 다룸.
  - 두 교환은 **성격이 다름** — 1차는 사실상 숙명, 2차(셔플)는 조건부 회피 가능. 아래 그림으로 풀어 봄.

```
[추가 설명 — 분산 집계가 네트워크를 타는 두 순간]
──────────────────────────────────────────────────────────────────────
 1차 교환 │ 저장소(S3·GCS·HDFS) ──네트워크──► 각 compute 노드     "입력 적재"
          │ 요즘은 storage·compute 가 분리돼 있어, 계산하려면 데이터를 끌어와야 함 ⇒ 피하기 어려움

 ─ 입력 적재 직후: 집계 key(user_id)가 여러 노드에 흩어져 있음 ─
   ┌── node 1 ──┐   ┌── node 2 ──┐
   │ user=1     │   │ user=1     │   ← user=1 이 두 노드에 분산돼 있음
   │ user=2     │   │ user=3     │
   └────────────┘   └────────────┘

 2차 교환 │ node ◄──네트워크──► node                            "셔플(shuffle)"
          │ 같은 key 를 한 노드로 다시 모음(collocate)
   ┌── node 1 ──┐   ┌── node 2 ──┐
   │ user=1     │   │ user=3     │
   │ user=1     │   └────────────┘   ⇒ 이제 노드별 reduce(합계) 가능
   │ user=2     │
   └────────────┘
──────────────────────────────────────────────────────────────────────
 정리 — 1차(저장소→노드)는 피하기 어렵지만, 2차(노드↔노드 셔플)는
 데이터가 **처음부터 key 별로 올바르게 분할** 돼 있으면 생략 가능 ⇒ #28 Local Aggregator 의 존재 이유.
 (user=1 이 처음부터 한 파티션에만 있으면, 다시 모을 필요가 없음)
```

- **Data skew (데이터 쏠림)**
  - 특정 key 의 출현 빈도가 다른 key 보다 압도적으로 높은 **불균형 데이터셋**.
    그 key 를 네트워크로 옮기고 단일 노드에서 처리하는 비용이 가장 커짐.
  - 완화책으로 **salting** — grouping key 에 추가 값(salt)을 붙여 salted 컬럼으로 1차 grouping 한 뒤,
    원래 key 결과가 필요하면 salted 집계 결과를 **다시 한 번 집계**(Example 5-13).
    프레임워크 네이티브 완화(예: Spark **Adaptive Query Execution**)도 있음.

```
[추가 설명 — 데이터 쏠림과 salting 2단계 집계]
──────────────────────────────────────────────────────────────────────
 ▷ 문제: user_id 로 집계하는데 guest 방문이 전부 user=0 으로 5천만 건 몰림
   ┌─ node 1 ──────────┐  ┌─ node 2 ─┐  ┌─ node 3 ─┐
   │ user=0  (5천만 건)  │  │ user=1   │  │ user=2   │
   └───────────────────┘  └──────────┘  └──────────┘
     ⚠ 혼자 5천만 건 = straggler    수천 건      수천 건
     ✗ 잡은 가장 느린 task 가 끝나야 끝남 ⇒ node 1 하나를 하염없이 대기(최악엔 OOM)

 ▷ salting 1차: 몰린 key 에 무작위 salt(0~2)를 붙여 3갈래로 쪼갬
   groupBy(user_id, salt)
   ┌─ node 1 ─────┐  ┌─ node 2 ─────┐  ┌─ node 3 ─────┐
   │ (0, salt=0)  │  │ (0, salt=1)  │  │ (0, salt=2)  │   ← 5천만 건이 3등분돼 병렬 처리
   │  ~1,667만 건  │  │  ~1,667만 건  │  │  ~1,667만 건  │
   └──────────────┘  └──────────────┘  └──────────────┘
       부분합 A           부분합 B           부분합 C        ✓ straggler 사라짐

 ▷ salting 2차: salt 를 떼고 원래 key 로 부분합을 다시 합쳐 진짜 결과 도출
   groupBy(user_id):  A + B + C = user=0 의 전체 집계 결과
──────────────────────────────────────────────────────────────────────
 2단계가 성립하는 이유 — count·sum 은 결합법칙이 성립("전체 = 부분들의 합")해 쪼갰다 합쳐도 결과 동일.
 (중앙값처럼 부분 결과를 단순 합칠 수 없는 집계엔 이 기법을 그대로 쓰지 못함)
 salt 범위(rand()*3 = 3조각)가 분산 정도를 정함 — 크게 잡으면 더 분산되나 2차에서 합칠 부분값도 늘어남.
 손으로 salting 하기 번거로우면 Spark AQE 가 실행 중 쏠린 파티션을 자동 감지·분할해 줌.
```

- **Scaling (확장성)** — 셔플은 확장성, 특히 **scale-in(다 쓴 노드를 반납해 클러스터를 줄이는 것)** 을 어렵게 만듦. 단계적으로 풀어 봄.
  - **무엇이 문제인가** — reduce(집계)를 끝낸 노드는 더 할 일이 없으니, 이상적으로는 곧바로 반납해 비용을 아껴야 함.
    그런데 분산 집계에서는 일을 끝낸 노드도 **잡이 끝날 때까지 묶여 있어** 반납이 안 됨. 그 이유가 바로 내결함성(fault tolerance).
  - **내결함성(fault tolerance)이란** — 노드나 task 가 도중에 죽어도 잡 전체를 처음부터 다시 돌리지 않고,
    **죽은 부분만 다시 계산** 해 복구하는 능력. 분산 처리는 "노드는 언제든 죽을 수 있다" 는 전제 위에서 동작하므로 반드시 필요한 장치.
  - **왜 끝난 노드가 묶이나** — 어떤 노드가 reduce 중에 죽으면, 그 노드의 입력이었던 **셔플 데이터** 가 남아 있어야
    새 노드에 다시 넘겨 죽은 부분만 재계산할 수 있음(= 전체 reshuffle 회피).
    그래서 reduce 를 이미 끝낸 노드들도 "혹시 다른 노드가 죽으면 데이터를 다시 줘야 함" 때문에
    자기 셔플 데이터를 못 버리고 잡이 끝날 때까지 들고 있음 ⇒ 일은 끝났는데 데이터 보관 탓에 회수 불가.
  - **완화책: shuffle service** — 셔플 데이터를 노드 안이 아니라 **노드 밖 별도 compute 컴포넌트** 에 따로 저장·서빙함.
    노드가 더는 셔플 데이터를 짊어지지 않으니, 죽어도 그 데이터는 shuffle service 에 안전하게 남아 내결함성(fault tolerance)이 유지됨.
    그 결과 일 끝난 노드는 잡 실행 중에도 즉시 회수 가능 ⇒ 탄력적 scale-in 실현.
    구현 예 — Spark **External Shuffle Service**, GCP **Dataflow Shuffle**.

```
[추가 설명 — 셔플 데이터가 노드를 붙잡아 scale-in 을 막는 이유]
──────────────────────────────────────────────────────────────────────
 ▷ 문제: reduce 를 끝낸 노드도 셔플 데이터를 들고 있어야 함 (재시작 시 reshuffle 회피)
   ┌── node 1 ──┐  ┌── node 2 ──┐  ┌── node 3 ──┐
   │ reduce done│  │ reduce ... │  │ reduce ... │
   │ + shuffle  │  │            │  │            │
   └────────────┘  └────────────┘  └────────────┘
     ⚠ node 1 은 일을 끝냈는데 셔플 데이터 때문에 반납 못 함
     ⇒ 잡이 끝날 때까지 점유돼 노는 노드를 회수 못 함, scale-in(축소) 불가

 ▷ 해결: shuffle service — 셔플 데이터를 노드 밖 별도 컴포넌트로 분리
   ┌── node 1 ──┐  ┌── node 2 ──┐
   │ reduce done│  │ reduce ... │  ──►  [ shuffle service : 셔플 데이터만 저장·서빙 ]
   └────────────┘  └────────────┘
     ✓ 노드가 셔플 데이터를 안 들고 있어도 됨
     ⇒ 일 끝난 node 1 을 잡 실행 중에도 즉시 회수 → 탄력적 scale-in 가능
──────────────────────────────────────────────────────────────────────
 구현 예 — Spark External Shuffle Service, GCP Dataflow Shuffle.
 정리 — Local Aggregator(#28)가 "셔플 자체를 없애는" 길이라면,
        shuffle service 는 "셔플은 두되 그 데이터를 노드와 분리해" 노는 노드를 자유롭게 회수하는 길.
```

#### 구현 예시 (Examples)

**예시 1 — skew 컬럼 salting (Example 5-13, PySpark)**

salt 로 1차 grouping 후 원래 key 로 2차 집계:
```python
dataset.withColumn('salt', (rand()*3).cast("int"))   # 0~2 무작위 salt
 .groupBy('group_key', 'salt').agg(...)               # 1차: salted key 로 분산 집계
 .groupBy('group_key').agg(...)                        # 2차: 원래 key 로 재집계
```

**예시 2 — 물리적으로 분리된 두 저장소 집계 (Example 5-14, PySpark)**

JSON 파일(visits)과 PostgreSQL(devices)을 같은 잡에서 `inner` join:
```python
visits: DataFrame = spark_session.read.json(f'{base_dir}/input-visits')   # 파일 시스템
devices: DataFrame = spark_session.read.jdbc(url='jdbc:postgresql:dedp',   # 관계형 DB
   table='dedp.devices', properties={'user': 'dedp_test',
   'password': 'dedp_test', 'driver': 'org.postgresql.Driver'})
visits_with_devices = visits.join(devices,
   [devices.type == visits.context.technical.dev_type,
    devices.version == visits.context.technical.dev_version], 'inner')
```

**예시 3 — 셔플 확인용 실행 계획 (Example 5-15)**

`explain()` 출력에서 `Exchange hashpartitioning` 노드를 찾으면 셔플 발생을 확인할 수 있음:
```
== Physical Plan ==
AdaptiveSparkPlan isFinalPlan=false
+- SortMergeJoin [ctx#8.technical.dev_type, ctx#8.technical.dev_version],..
 :- Sort [ctx#8.technical.dev_type ASC NULLS FIRST, ...
 : +- Exchange hashpartitioning(ctx#8.technical.dev_type, ...   ← 셔플 발생 지점
 :   +- Filter (....)                                            ← 셔플 전 로컬 필터링
 :     +- FileScan json [.....
 +- Sort [type#20 ASC NULLS FIRST, version#22 ASC NULLS FIRST], false, 0
   +- Exchange hashpartitioning(type#20, version#22, 200), ENSURE_REQUIREMENTS,..
     +- Scan JDBCRelation(....
```

> 잡은 셔플 **전에** 필터링 같은 로컬 연산을 먼저 수행해 교환량을 줄임.
> 이 패턴은 DB 에도 적용됨 — GCP BigQuery 는 GCS 오브젝트를 **external table** 로 선언해 테이블과 결합 가능.
> external table 은 AWS Redshift·Azure Synapse Analytics·Snowflake 도 지원함.

---

### 3-2. 패턴 #28: 로컬 집계기 (Local Aggregator)

> 데이터가 입력에서 **이미 올바르게 분할** 돼 있거나 단일 머신에 들어가면, 네트워크 교환(셔플)이 불필요할 수 있음.
> 두 경우 모두 이 패턴으로 해결함.

#### 상황 (Problem)

**책의 use case** — 셔플을 제거하고 싶은 윈도우 생성 잡:

- **분할된 스트리밍 브로커** 에 저장된 incoming visits 로 윈도우를 생성하는 스트리밍 잡이 있음.
- 데이터 볼륨이 **정적** 이고, 기반 분할의 갑작스러운 변동·변경을 예상하지 않음 → **파티션 수가 절대 바뀌지 않음**.
- 프레임워크가 자동으로 추가하는 **grouping 셔플 단계를 제거** 해 잡을 최적화하고 싶음.
- **결정적 제약**: 정적 데이터·일관된 분할이 보장됨 → 굳이 셔플 없이 로컬 집계가 가능한 조건.

#### 해결 (Solution)

비싼 셔플 + 정적 데이터 소스 분할 + 관련 속성이 함께 저장됨 — 이 세 조건이 **Local Aggregator** 의 근거.

- 여전히 집계는 하지만, **입력을 읽는 단 한 번의 네트워크 교환** 만으로 **로컬에서** 처리함.
  고정 분할 스키마와 올바른 입력 분포 덕분 — 특정 grouping key 의 모든 record 가
  **이미 같은 입력 파티션에 존재** 해 다른 곳에서 끌어올 필요가 없음.
- 셔플이 없는 것 외에 또 다른 이점 — task 가 **완전히 격리(fully isolated)** 됨.
  다른 task 의 데이터를 기다리지 않고 전진 가능 — 느린 처리 단위가 전체를 지연시키는 스트리밍에서 특히 유용.
- 구현 노력은 **producer(생산자) 쪽** 에 집중해야 함.
  특정 grouping key 의 record 를 **항상 같은 물리 파티션** 에 쓰도록 보장해야 함
  — **정적 per-record 분할 키** 와 **불변 파티션 수** 로 달성.
- consumer(소비) 쪽 — 일부 도구는 미리 분할된 데이터셋을 셔플 포맷에 맞춰주는 편의 메서드를 제공.
  **Kafka Streams 의 `groupByKey`** 가 그 예. Apache Spark 는 셔플 회피용 힌트는 없지만,
  **`mapPartitions`·`forEachPartition`** 같은 per-partition 연산으로 로컬 집계를 직접 구현 가능.
  또 Spark 는 **같은 key·같은 bucket 수** 로 저장된(bucketing) 데이터셋의 셔플도 회피함.

```
[Distributed vs Local Aggregator — 셔플 유무]
─────────────────────────────────────────────────────────────────────────────────────
 Distributed (#27)   입력 적재 ─► [노드별 부분집계] ─► shuffle (네트워크 비용↑) ─► reduce
                     같은 key 가 여러 노드에 흩어져 있음 ⇒ 모으려면 셔플 필수 (정상 동작이지만 비쌈)

 Local (#28)         입력 적재 ─► [파티션 내 로컬 집계] ─► shuffle 생략 ─► 출력
                     같은 key 가 이미 같은 파티션에 모여 있음 ⇒ 셔플 불필요
                     ⚠ 단, 파티션 수·grouping key 가 불변이어야 성립
─────────────────────────────────────────────────────────────────────────────────────
 핵심 — 셔플은 "틀린 동작"이 아니라 분산 집계의 필수 단계. 다만 네트워크 비용이 커서,
        데이터가 이미 key 별로 잘 분할돼 있을 때 그 비용을 피하는 길이 Local Aggregator.
```

> **참고 사항 — Bucketing 이란**
> Bucketing(= clustering)은 **파티션을 다시 분할** 하는 방식.
> 자세한 내용은 챕터 8의 **Bucket 패턴** 에서 다룸.

비분할이거나 분할됐어도 **작은 데이터셋** 이면 정적 파티션 수를 걱정할 필요 없음
— 전체를 처리 노드에 적재해 **공유 메모리** 로 집계하면 됨.

#### 고려사항 (Consequences)

이 패턴은 저장·grouping key 에 대한 **불변성(immutability)** 을 요구하는데, 늘 가능한 건 아님.

- **Scaling (확장성)**
  - 가장 두드러진 문제. 패턴은 데이터 소스의 **정적 성격** 과 **일관된 분할**(특정 key 가 오직 한 처리 파티션에서만 옴)에 의존.
    이 조건 중 하나라도 못 지키면, 저장 파티션을 바꿀 때 한 key 에 대해 **여러 그룹이 생겨** 동작이 깨짐.
  - 확장하려면 **전용 재구성 task** 로 모든 record 의 파티션 할당을 재생성해야 함 — 전체 처리라 비용 큼.
    스트리밍에선 더 까다로움 — 기존 파티션의 잔여 데이터를 모두 처리하는 **stop-the-world** 이벤트가 필요.
  - 즉 Local Aggregator 는 셔플을 없애는 대신 **확장을 훨씬 어렵게** 만듦.
- **Grouping keys (그룹 키)**
  - Local Aggregator 가 셔플을 없앨 수 있는 비결은 **데이터를 쓸 때 "분할 키 = 집계 키" 로 미리 맞춰 둔 것**.
    예컨대 user_id 로 분할해 두면 같은 user 의 record 가 한 파티션에 모여 있어, user_id 집계를 셔플 없이 로컬로 처리 가능.
  - 그런데 **물리 분할은 오직 한 가지 키로만** 됨. 컨슈머마다 집계하려는 키가 다를 수 있는데,
    데이터를 그 여러 키로 동시에 분할할 수는 없으니 **하나의 grouping key 로직만** 로컬 집계의 혜택을 받음.
  - **구체 예** — user 프로필 변경 스트림을 두 컨슈머가 서로 다르게 집계함:
    - 컨슈머 A: **변경 타입(change type)** 별 집계 (이메일 변경 몇 건, 주소 변경 몇 건 …)
    - 컨슈머 B: **user ID** 별 집계 (사용자별 변경 횟수 …)
    데이터는 change_type **또는** user_id 중 하나로만 물리 분할됨. 만약 change_type 으로 분할하면
    A 는 로컬 집계가 되지만, B 가 찾는 한 user 의 record 는 여러 파티션에 흩어져 있어 셔플이 필요
    ⇒ B 는 **Distributed Aggregator** 를 써야 함.
  - 양쪽 모두 로컬로 처리하려면 **같은 데이터를 change_type 용·user_id 용으로 두 벌 저장** 해야 해서(중복 저장) 부담이 큼.

```
[그룹 키가 분할 키와 어긋나면 로컬 집계가 안 됨]
──────────────────────────────────────────────────────────────────────
 데이터를 change_type 으로 분할해 둔 경우 (한 파티션 = 한 change_type)
   파티션1: change=email   파티션2: change=addr   파티션3: change=phone
   ※ user=1 의 변경들은 email·addr·phone 으로 파티션1·2·3 에 흩어질 수 있음

 컨슈머 A (change_type 별 집계)
   ⇒ 각 파티션이 곧 하나의 change_type, 셔플 없이 로컬 집계 OK   → Local Aggregator

 컨슈머 B (user_id 별 집계)
   ⇒ user=1 이 파티션1·2·3 에 흩어져 있어 한곳에 모아야 함, 셔플 필요   → Distributed Aggregator
──────────────────────────────────────────────────────────────────────
 핵심 — 물리 분할은 한 키로만 가능. 그 키로 집계하는 컨슈머만 로컬 혜택을 받고,
        다른 키로 집계하는 컨슈머는 Distributed Aggregator 로 가야 함 (아니면 키별로 데이터를 중복 저장).
```

#### 구현 예시 (Examples)

**예시 1 — Kafka Streams 로컬 집계 (Example 5-16, Java)**

`groupByKey` 가 각 record 의 key 속성으로 **네트워크 교환 없이** 같은 key 를 묶음:
```java
KStream<String, String> visitsSource = streamsBuilder.stream("visits");
KGroupedStream<String, String> groupedVisits = visitsSource.groupByKey();   // key 명시 없음
KStream<String, AggregatedVisits> aggregatedVisits = groupedVisits
  .aggregate(AggregatedVisits::new, new AggregatedVisitsAggregator(),
    Materialized.with(Serdes.String(), new JsonSerializer<>())).toStream();
aggregatedVisits.to("visits-aggregated", Produced.with(new Serdes.StringSerde(),
  new JsonSerializer<>()));
```

> `groupByKey` 에 key 가 안 보이는 이유 — 각 incoming record 에 **이미 붙어 있는 key 속성** 을 그대로 써서
> 같은 key 를 셔플 없이 결합하기 때문.

**예시 2 — Spark 파티션 내 정렬 후 per-partition 처리 (Example 5-17, PySpark)**

`sortWithinPartitions` 로 파티션 안에서만 정렬한 뒤, 파티션별로 `KafkaWriter` 실행:
```python
sorted_visits: DataFrame = (visits_to_save
  .sortWithinPartitions(['visit_id', 'event_time']))   # 파티션 내부 정렬(셔플 없음)
def write_records_from_spark_partition_to_kafka_topic(visits):
  kafka_writer = KafkaWriter(...)
  for visit in visits:
    kafka_writer.process(visit)
  kafka_writer.close()
sorted_visits.foreachPartition(write_records_from_spark_partition_to_kafka_topic)
```

**예시 3 — per-partition writer (Example 5-18, PySpark)**

정렬된 row 를 받아 한 session(visit)의 페이지들을 모으다, `visit_id` 가 바뀌면 결과를 전송:
```python
class KafkaWriter:
  def __init__(self, bootstrap_server: str, output_topic: str):
    self.in_flight_visit = {'visit_id': None}
  def process(self, row):
    if row.visit_id != self.in_flight_visit['visit_id']:   # visit 이 바뀌면
      send_to_kafka(self.in_flight_visit)                  # 직전 집계 결과 전송
      self.in_flight_visit = {'visit_id': row.visit_id, 'pages': [],...}
    self.in_flight_visit['pages'].append(row.page)
    # ...
```

> 입력이 파티션 내에서 `visit_id` 로 정렬돼 있으므로, "visit_id 가 바뀌는 순간 = 한 session 종료" 로 보고
> 누적 페이지를 한 번에 내보냄 — 셔플 없이 로컬에서 session 집계가 완성됨.

**예시 4 — AWS Redshift 저장 분포 (Example 5-19, SQL)**

클라우드 DW 도 로컬 집계를 지원 — `DISTSTYLE` 로 join 을 셔플 없이 로컬에서 수행:
```sql
CREATE TABLE visits (
  visit_id INT, user_id INT, ...
) ...
DISTSTYLE KEY,
DISTKEY(visit_id);   -- 같은 visit_id 를 같은 노드에 모음 → visit_id 결합은 로컬

CREATE TABLE users (
  user_id INT, ...
) ...
DISTSTYLE ALL;       -- users 를 모든 노드에 복제 → users 와의 join 은 항상 로컬
```

| 분포 방식 | 동작 | 주의사항 |
|---|---|---|
| `DISTSTYLE KEY` + `DISTKEY(col)` | 같은 key 를 같은 노드에 collocate | 그 key 기준 결합만 로컬, 다른 키는 셔플 |
| `DISTSTYLE ALL` | 테이블 전체를 모든 노드에 복제 | 작은 차원 테이블에 적합, 크면 저장 낭비 |

> **트러블 로그** — Local Aggregator 를 쓰다 **파티션 수를 바꾸면** 같은 key 가 여러 파티션에 흩어져 집계가 쪼개짐.
> 예: Kafka `visits` 토픽 파티션을 6→12 로 늘리면, `user_id` 해시 분포가 달라져
> 같은 user 의 이벤트가 두 파티션에 나뉘고, session 집계가 **한 사용자당 2개로 중복 생성** 됨
> — 에러 없이 MAU·세션 수 지표만 부풀어 오름.
> 정적 분할을 전제로 한 패턴이므로 **파티션 수를 불변으로 고정** 하고, 꼭 바꿔야 하면
> 잔여 데이터를 모두 처리하는 재구성(stop-the-world) 절차를 먼저 거칠 것.

---

## 4. 세션화 (Sessionization)

세션(session)은 **같은 활동에 속한 이벤트를 묶는** 특수한 집계
— 시작점, 세션 이벤트, 종료점으로 구성됨(영상 시청·운동·요리처럼 일상에서도 만들어짐).
데이터가 정적(at rest)이냐 움직이냐(in motion)에 따라 두 패턴 중 하나를 택함.

- **#29 Incremental Sessionizer** — **배치 파이프라인** 용. 시간별 파티션을 점진적으로 이어 붙여 세션 생성.
- **#30 Stateful Sessionizer** — **스트림 처리** 용. state store 로 near real-time 세션 생성.

---

### 4-1. 패턴 #29: 증분 세션화 처리기 (Incremental Sessionizer)

> 세션은 실시간 컴포넌트처럼 들리지만, 생성 방식은 **배치 처리도 지원** 함. 이 패턴은 배치 파이프라인에 맞춰져 있음.

#### 상황 (Problem)

**책의 use case** — 시간별 파티션에 걸친 visit 세션화:

- 데이터 수집 팀이 visit 이벤트를 **시간별(hourly) 파티션** 위치에 저장함.
- 이걸 세션으로 집계하고 싶음 — **첫 visit 에서 시작** 하고, 해당 user 의 새 visit 이
  **2시간 안에 없으면 종료**.
- 세션의 통상 길이는 수 분 ~ 3시간 → 한 visit 이 **최대 3개 파티션** 에 걸칠 수 있음.
- 데이터 분석가들이 매번 세션을 맞추느라 고생 — 한 user 에 대해 **여러 연속 파티션** 을 처리해야 하기 때문.
- **결정적 제약**: 한 세션의 record 가 여러 연속 파티션에 존재 → **증분 처리(incremental processing)** 패밀리 문제.
  (챕터 2의 **Incremental Loader** 를 세션화에 응용)

#### 해결 (Solution)

연속 파티션에 걸친 세션이므로 증분 처리로 풂. → **Incremental Sessionizer**.
구현에는 **세 개의 저장 공간** 이 필요함.

- **Input dataset storage (입력 데이터셋)** — 세션으로 묶을 **raw 이벤트(원천 visit)** 가 쌓이는 곳.
  데이터 수집 팀이 적재한 **시간별(hourly) 파티션 visits** 가 여기 들어가며,
  세션화 잡은 **매 실행마다 이 중 이번 시간에 해당하는 새 파티션을 읽어** 처리함.
  세션이 아니라 아직 가공 전의 visit 이벤트를 담는다는 점에서, 세션을 담는 아래 두 저장소와 성격이 다름.
- **Completed sessions storage (완료 세션)** — 끝난 세션을 모두 기록하는 곳.
  진행 중 세션도 여기 쓸 수 있으나, 이상적으로는 **`is_final` 같은 속성** 으로 완료 세션과 구분할 것.
- **Pending sessions storage (대기 세션)** — 다음 실행에서 닫힐, 여러 파티션에 걸친 세션을 보관.
  완료 세션 저장소와 두 가지가 다름 
  — ① **비공개** 여야 함(내부 로직과 함께 진화. 컨슈머가 세부를 알 필요 없고, 알면 진화 유연성이 떨어짐),
  — ② **포맷이 완료 세션과 달라도 됨**(멱등성 보장에 도움 되는 execution ID 같은 기술/내부 정보를 넣을 수 있음).

워크플로 로직 — 입력 데이터셋을 직전 실행에서 생성된 **모든 대기 세션과 결합**.
결합은 세션 엔티티(user·product·visit)별로 일어나며 다음을 생성함.

- **새 세션** — 그 엔티티(user 등)에 직전 Pending 세션이 **없을 때**. 이번 입력의 visit 으로 세션을 **새로 시작**함.
  (예: 그 user 가 이전 시간엔 활동이 없다가 이번 시간에 처음 방문한 경우)
- **복원 + 새 데이터(new data 있음)** — 직전 Pending 세션이 **있고**, 이번 입력에도 그 엔티티의 새 visit 이 **들어온** 경우.
  Pending 세션을 꺼내 새 visit 을 **이어 붙여 연장**함. 마지막 활동 시각이 갱신되므로 **만료 시점도 뒤로 밀림**.
- **복원 + 새 데이터 없음(new data 없음)** — 직전 Pending 세션은 **있는데**, 이번 입력엔 그 엔티티의 visit 이 **없는** 경우.
  세션을 그대로 두되 **마지막 활동 이후 경과 시간만 따짐**.
  비활동 기준(예: 2시간)을 넘기면 이번/다음 실행에서 **종료(만료)** 되고, 아직이면 다시 Pending 으로 남김.

| 직전 Pending | 이번 새 visit | 결합 결과 | 동작 |
|---|---|---|---|
| 없음 | 있음 | ① 새 세션 | 이번 visit 으로 **새로 시작** |
| 있음 | 있음 | ② 복원 + 새 데이터 | 기존 세션에 **이어 붙여 연장** (만료 뒤로 밀림) |
| 있음 | 없음 | ③ 복원 + 데이터 없음 | **유지하며 만료 카운트** (기준 넘으면 종료) |
| 없음 | 없음 | — | 그 엔티티는 이번 실행과 무관 → **등장 안 함** |

결합이 끝나면 처리할 세션 데이터(이전+신규 record)가 나옴. 여기에 **세 가지 상태** 의 세션화 로직을 적용함.

- **Initialization (시작)** — 세션이 시작될 때. 예: 블로그 홈페이지 방문 같은 특정 이벤트 타입 발생 시.
- **Accumulation (누적)** — 세션이 살아 있을 때. 새 데이터를 어떻게 다룰지 정의(예: 방문 페이지를 순서대로 저장).
- **Finalization (종료)** — 세션이 멈출 때. 특정 이벤트 타입 발생 또는 **비활동 기간**(예: 3시간 이탈)으로 종료.

```
[Figure 5-6 재현] 세 저장 공간을 쓰는 Incremental Sessionizer
──────────────────────────────────────────────────────────────────────
   ┌──────────────────┐
   │  Input dataset   │──┐
   └──────────────────┘  │     ┌────────────────────────┐   Completed   ┌──────────────────┐
                         ├────►│ • 세션 생성              │   sessions    │ Output Sessions  │
   ┌──────────────────┐  │     │ • 세션 복원 (new data O) │──────────────►│  (완료 세션 공개)   │
   │ Pending Sessions │──┘     │ • 세션 복원 (new data X) │               └──────────────────┘
   │  (비공개)          │◄───┐   │ • 세션 종료              │
   └──────────────────┘    │   └────────────────────────┘
                           └── 시작됐지만 완료되지 않은 세션을 다시 pending 으로 기록
──────────────────────────────────────────────────────────────────────
 직전 실행의 pending 세션 + 입력 신규 데이터를 결합 → 완료 세션은 공개 저장소로, 미완료는 pending 으로 회귀
```

> 처리 로직은 `WINDOW` 함수나 `GROUP BY` 식 + 언어별 커스텀 매핑 함수로 정의함.
> 로직이 복잡하면 SQL 보다 **프로그래밍 API** 가 더 쉬울 수 있음.

**쉽게 풀어 보면** — 세션은 여러 시간대 파티션에 걸쳐 있어(최대 3시간 → 3개 파티션) 한 번에 끝낼 수 없음.
그래서 **아직 안 끝난 세션을 Pending 에 넣어 다음 실행으로 넘기고**, 한 시간(파티션)씩 이어 붙여 완성함.
매 실행은 **(이번 시간 입력) + (직전 Pending)** 만 처리하고 전체 과거는 다시 안 돌림. 그래서 이름이 "증분(Incremental)".

- 핵심은 **Pending(대기) 저장소** — "이번 시간엔 안 끝났지만 다음 시간에 이어질 수 있는 세션" 을 잠시 모셔 뒀다가
  다음 실행 때 새 입력과 합쳐 이어감. 끝나면 비로소 **Completed(공개)** 로 내보냄.
- 매 실행의 결합 결과는 user 별로 셋 중 하나.
  ① **새 세션**(pending 없음) · ② **복원 + 새 데이터**(기존 세션 연장) · ③ **복원 + 새 데이터 없음**(만료에 가까워짐)

```
[Incremental Sessionizer — user=1 의 세션이 시간대를 넘어 완성되는 과정]
──────────────────────────────────────────────────────────────────────
 09:00 파티션
   입력: user=1 → 09:50 에 page A 방문   /   직전 Pending: 없음
   ⇒ ① 새 세션 (A) 생성. 09:50 뒤 2시간 안 지남 → Completed 아님, Pending 으로 넘김

 10:00 파티션
   입력: user=1 → 10:30 에 page B 방문   /   직전 Pending: 세션 (A) 있음
   ⇒ ② 복원+새 데이터: (A) → (A,B) 로 연장. 아직 2시간 안 지남 → 다시 Pending

 11:00 · 12:00 파티션
   입력: user=1 visit 없음              /   직전 Pending: 세션 (A,B) 있음
   ⇒ ③ 복원+데이터 없음: 그대로 유지하며 만료 시점만 체크
   12:30 = 마지막 visit(10:30) + 2시간 경과 ⇒ Finalization(종료)
        → Completed 에 (A,B) 기록, Pending 에서 제거
──────────────────────────────────────────────────────────────────────
 ⚠ 흩어진 visit 을 한 번에 못 끝내는 게 핵심 난점 — Pending 을 징검다리 삼아 시간대를 넘겨 한 세션으로 이어 붙임.
 (이번 시간에 끝났는지 알려면 다음 시간 데이터가 필요하므로, 미완 세션은 항상 Pending 에 남겨 둠)
```

#### 고려사항 (Consequences)

- **Inactivity period (비활동 기간)**
  - 세션을 얼마나 오래 열어둘지 정의. 길수록 더 많은 late data 를 세션에 담지만, **compute·storage 자원이 더 듦**.
    비즈니스 로직과 비용 사이 균형이 필요.
  - 긴 비활동 기간은 세션을 그만큼 **숨김 공간(pending)** 에 오래 둠.
    컨슈머가 **부분 세션(partial session)** 을 받아들일 수 있으면 출력 세션 저장소로 내보낼 수도 있으나, **정합성 위험** 을 알려야 함.
  - 부분 세션은 완료 세션이 아니라 **다음 버전에서 바뀔 수 있음**.
    예: 은행 사기 탐지의 부분 세션이 첫 파티션에선 "위험 없음" 으로 분류됐다가 다음 파티션에서 "위험" 으로 바뀜
    — 컨슈머가 부분 세션을 최종으로 간주하면 잘못된 로직을 적용함.
  - 따라서 진행 중 세션은 **`is_completed: false` 같은 속성으로 표시** 해, 완료 상태만 보는 다운스트림이 무시하게 할 것.
- **Data freshness (데이터 신선도)**
  - Incremental Sessionizer 는 배치 파이프라인용(여전히 데이터 팀의 1차 선택) → 인사이트가 실시간 대비 **매우 늦게** 나옴.
    완화하려면 위의 부분 세션을 만들 수 있음.
- **Late data·event time 파티션·backfilling**
  - 세션화가 event time 파티셔닝에 의존하면, 이미 처리된 파티션의 세션을 **놓칠 수 있음**.
  - 세션은 **forward dependent(앞으로 의존)** 한 특수 자산
    — 09:00 파티션 세션이 10:00 에 직접 영향을 주고, 10:00 이 11:00 에 영향을 줌.
    한 엔티티의 late data 를 통합해도 그게 **다음 세션들에 영향을 안 준다는 보장이 없음**.
  - backfilling 도 마찬가지 — 한 파티션을 재실행하면 **이후 모든 파티션** 도 재실행해야 해 금세 비싸짐.
    만능 해법은 없음 — 전체 재생은 코드는 쉽지만 비싸고, 영리한 탐지로 대상만 backfill 하면 비용은 줄지만 복잡함.

```
[참고 — 부분 세션(partial session)은 뒤집힐 수 있음 (정합성 위험)]
──────────────────────────────────────────────────────────────────────
 끝나기 전 세션을 미리 내보낸 경우 (예: 은행 사기 탐지)
   09:00 까지 본 부분 세션  ⇒ 판정 "위험 없음"   ← 컨슈머가 이걸 최종으로 오해
   10:00 데이터가 더 붙음    ⇒ 판정 "위험"        ← 결론이 뒤집힘
 ⚠ 부분 세션은 완료본이 아니라 다음 실행에서 바뀔 수 있음
 ⇒ 진행 중 세션엔 is_completed=false 를 달아, 완료본만 보는 다운스트림이 무시하게 함
──────────────────────────────────────────────────────────────────────
```

```
[참고 — 세션은 forward dependent: 앞 세션이 뒤 세션에 연쇄로 영향]
──────────────────────────────────────────────────────────────────────
   09:00 세션 ──(Pending 으로 이어짐)──► 10:00 세션 ──► 11:00 세션 ──► …
   (Pending 이 다음 실행으로 넘어가며 앞뒤가 줄줄이 엮임)

 ▷ Late data — 이미 처리한 09:00 에 visit 하나가 뒤늦게 도착
   ⇒ 09:00 세션만 고쳐선 끝이 아님. 10:00·11:00… 까지 연쇄로 바뀔 수 있음

 ▷ Backfilling — 09:00 파티션을 다시 돌리면
   ⇒ 사슬 때문에 이후 모든 파티션도 재실행해야 함 → 금세 비싸짐
──────────────────────────────────────────────────────────────────────
 만능 해법 없음 — ⓐ 영향 시점부터 전부 재생(쉽지만 비쌈) vs ⓑ 영향 엔티티만 골라 재처리(싸지만 복잡).
```

#### 구현 예시 (Examples)

**예시 1 — Incremental Sessionizer 단계 (Example 5-20, Airflow)**

완료/대기 세션을 정리하는 두 task 를 동시에 돌린 뒤 세션을 생성:
```python
clean_previous_runs_sessions = PostgresOperator(...)
clean_previous_runs_pending_sessions = PostgresOperator(...)
generate_sessions = PostgresOperator(...)

([clean_previous_runs_sessions, clean_previous_runs_pending_sessions]
 >> generate_sessions)
```

**예시 2 — 멱등성 컴포넌트 (Example 5-21, SQL)**

Airflow 의 불변 `ds` 파라미터로 이번 및 이후 실행분을 지워 재실행을 멱등하게 만듦:
```sql
DELETE FROM dedp.sessions WHERE execution_time_id >= '{{ ds }}';
DELETE FROM dedp.pending_sessions WHERE execution_time_id >= '{{ ds }}';
```

**예시 3 — 새 데이터 적재 (Example 5-22, SQL)**

입력을 **session-scoped 임시 테이블** 에 먼저 적재 — 세션 쿼리가 실패해도 적재를 다시 안 해도 됨:
```sql
CREATE TEMPORARY TABLE visits_{{ ds_nodash }} (# ...);
COPY visits_{{ ds_nodash }} FROM '/data_to_load/date={{ ds_nodash }}/dataset.csv' CSV
```

**예시 4 — 세션 생성 로직 (Example 5-23, SQL)**

신규 데이터와 직전 실행의 pending 세션을 `FULL OUTER JOIN` 으로 결합:
```sql
CREATE TEMPORARY TABLE sessions_to_classify AS
 SELECT
  COALESCE(p.session_id, n.session_id) AS session_id,   -- 첫 비-NULL 선택
  LEAST(p.start_time, n.start_time) AS start_time,        -- 더 이른 시작
  GREATEST(p.last_visit_time, n.start_time) AS last_visit_time,  -- 더 늦은 마지막
  ARRAY_CAT(p.pages, n.pages) AS pages,                   -- 두 배열 이어 붙임
  CASE
    WHEN n.user_id IS NULL THEN p.expiration_batch_id     -- 새 데이터 없으면 기존 만료 유지
    ELSE '{{ macros.ds_add(ds, 2) }}'                     -- 새 데이터 있으면 만료 2일 연장
  END AS expiration_batch_id
 FROM (SELECT ... FROM visits_{{ ds_nodash }}
  WINDOW visits_window AS (PARTITION BY visit_id, user_id ORDER BY event_time)
 ) AS n
 FULL OUTER JOIN (
  SELECT ... FROM dedp.pending_sessions WHERE execution_time_id = '{{ prev_ds }}')
  AS p ON n.session_id = p.session_id;
```

> `COALESCE`(첫 비-NULL)·`LEAST`/`GREATEST`(최소/최대)·`ARRAY_CAT`(배열 연결) 같은 편의 함수로 결합하고,
> **만료 시각은 세션에 새 데이터가 있을 때만** 갱신함.

**예시 5 — 세션 기록 (Example 5-24, SQL)**

만료 시각이 현재 실행과 다르면 pending 으로, 같으면 완료 세션으로 INSERT:
```sql
INSERT INTO dedp.pending_sessions (...)
    SELECT ... FROM sessions_to_classify WHERE expiration_batch_id != '{{ ds }}';
INSERT INTO dedp.sessions (...)
    SELECT ... FROM sessions_to_classify WHERE expiration_batch_id = '{{ ds }}';
```

> **트러블 로그** — 세션은 forward dependent 라, late data 나 backfill 시 **이후 파티션을 함께 재처리하지 않으면** 세션이 어긋남.
> 예: 09:00 파티션에 늦게 도착한 visit 을 그 파티션만 다시 돌려 반영하면, 그 visit 으로 이어졌어야 할
> 10:00·11:00 세션은 여전히 옛 결과라 — 한 user 의 세션이 09:00 에선 "종료", 10:00 에선 "신규 시작" 으로 쪼개져 중복 집계됨.
> 진행 중 세션은 `is_completed: false` 로 표시하고, backfill 은 **수정된 파티션 이후 전 구간을 재실행**(또는 영향 엔티티만 추적해 재처리)할 것.

---

### 4-2. 패턴 #30: 상태 저장 세션화 처리기 (Stateful Sessionizer)

> 데이터 신선도가 문제라면 Incremental Sessionizer 로는 부족함.
> **스트림 처리 계층 위** 에서 더 잦고 작은 반복으로 도는 세션화 패턴이 필요함.

#### 상황 (Problem)

**책의 use case** — 더 낮은 latency 의 세션이 필요해진 상황:

- 이해관계자들이 세션 가용성에 만족하던 중, 점점 더 많은 이들이 **더 낮은 latency** 로 세션에 접근하길 원함.
- Incremental Sessionizer 로는 불가능 — 파티션이 시간 단위라 **최선의 latency 가 1시간**.
- 다행히 visits 는 **스트리밍 브로커에 수 초 안에** 도착함.
- **결정적 제약**: 배치 파이프라인을 다시 써서 **near real-time** 으로 세션을 생성해야 함.

#### 해결 (Solution)

배치로는 "as soon as possible" 보장이 어렵고, 기본 스트리밍은 **stateless** 라 역시 안 됨.
→ 더 진보한 **Stateful Sessionizer**.

- stateless 스트리밍과의 차이 — stateful 파이프라인은 **state store** 라는 추가 컴포넌트를 가짐
  (챕터 4의 **Windowed Deduplicator** 에서 본 그 state store).
  세션화 맥락에서 state store 는 Incremental Sessionizer 의 **pending 세션 저장소와 같은 역할**
  — 모든 in-flight 세션을 저장해 처리 내내 유지함.
- 워크플로는 Incremental Sessionizer 와 동일함.
  ① state store 에 있으면 세션을 **재개**, 없으면 **생성**.
  ② 생성/재개된 세션을 비즈니스 로직대로 새 incoming record 와 **결합**(예: 방문 페이지를 순서대로 저장).
  ③ 세션이 완료됐거나 부분 세션을 컨슈머에게 노출해야 하면 **최종 출력에 기록**.
  미완료면 **새 상태를 state store 에 기록**.

```
[Figure 5-7 재현] 데이터 처리 잡과 state store 의 상호작용
──────────────────────────────────────────────────────────────────────
                   ┌─────────── Stateful job ───────────┐
   ┌────────┐      │   ┌──────────────────────────┐     │      ┌────────┐
   │ Input  │─────►│   │ State store (in-memory)  │     │─────►│ Output │
   └────────┘      │   │  빠른 접근, 휘발성           │     │      └────────┘
                   └───────────────┬────────────────────┘
                                   │ 주기적 동기화(checkpoint)
                                   ▼
                   ┌ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┐
                       State store (resilient)
                   │   내결함성 용 내구 저장          │
                   └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ┘
──────────────────────────────────────────────────────────────────────
 state store 는 두 종류 — ① 빠른 in-memory(휘발성), ② 장애 대비 내구 저장.
 잡은 상태를 내구 저장소로 주기 동기화해 재시작 시 유실을 막음.
```

데이터 처리 로직은 다음 추상화에 기댈 수 있음.

- **Session windows (세션 윈도우)** — 각 session key 마다 만들어지는 윈도우.
  길이는 **gap duration**(같은 session key 의 두 이벤트 사이 허용 최대 비활동 기간)으로 지정.
  두 이벤트 간격이 gap duration 보다 크면 **새 세션 윈도우**, 작으면 기존 세션에 포함.
- **Arbitrary stateful processing (임의 상태 저장 처리)** — 세션 윈도우보다 품은 더 들지만 유연함.
  gap duration 로직을 직접 정의 — 정적 타이머 또는 session key 마다 다른 동적 연산도 가능.
  Spark Structured Streaming·Flink·GCP Dataflow 가 기본 제공.

```
[Figure 5-8 재현] gap duration 20분일 때 같은 key 의 두 세션 윈도우
──────────────────────────────────────────────────────────────────────────
  ┌─ session window 1 ───────────────────┐          ┌─ session window 2 ─┐
  │  10:00    10:05         10:15        │          │      10:45         │
  └──────┬───────┬─────────────┬─────────┘          └─────────┬──────────┘
      10:05-10:00   10:15-10:05                       10:45-10:15
       = 5분         = 10분                              = 30분 (> 20분)
       (≤20, 같은 세션)  (≤20, 같은 세션)                  ⇒ 새 세션 윈도우
──────────────────────────────────────────────────────────────────────────
 간격이 gap duration(20분) 이하면 같은 세션, 초과하면 새 세션이 시작됨.
```

**쉽게 풀어 보면** — #30 은 #29(배치)와 **워크플로가 똑같음**. 단 하나, **Pending 저장소가 state store 로** 바뀌고 배치가 스트리밍이 됨.
새 이벤트가 올 때마다 state store 에서 그 key 의 세션을 꺼내(있으면 재개, 없으면 생성) 이어 붙이고,
gap(비활동)을 넘으면 방출, 아니면 state store 에 다시 저장하며 주기적으로 checkpoint 백업함.

```
[쉽게 풀어 보면 — #30 은 실시간 루프 (state store 가 #29 의 Pending 역할)]
──────────────────────────────────────────────────────────────────────
 새 visit 이벤트 도착 (key = visit_id)
        │
        ▼
   state store 조회
     ├─ 있음 → 세션 재개(resume)
     └─ 없음 → 세션 생성(create)
        │
        ▼
   새 이벤트 결합 (예: 방문 페이지 누적)
        │
        ▼
   gap(비활동) 넘었나? ── 예 ──►  세션 방출(emit) → 출력, state 제거
        │ 아니오
        ▼
   state store 에 갱신 저장 ──(주기적)──► checkpoint(내구 저장)로 백업
──────────────────────────────────────────────────────────────────────
 #29 와 워크플로 동일, "Pending 저장소" 가 "state store" 로 바뀐 실시간 버전.
 ⚠ 만료는 event time 기준(processing time 은 지연 시 조기 만료 위험), session key 는 불변값(visit_id)으로.
```

#### 고려사항 (Consequences)

- **At-least-once processing (적어도 한 번 처리)**
  - 상태를 내구 저장소에 쓰는 일(=**checkpointing**)은 상태 갱신마다가 아니라 **불규칙하게** 일어남.
    멈춘 잡은 마지막 성공 checkpoint 부터 재시작 → **at-least-once** 의미.
  - 나쁜 건 아니지만, **session key 를 실행마다 바뀌는 속성**(예: 실시간 시각)으로 만드는 등
    세션 멱등성을 깨는 부작용은 피해야 함 — 재시작마다 값이 달라지기 때문.
- **Scaling (확장성)**
  - stateful 맥락에서 compute 용량 변경은 **state rebalancing** 을 수반
    — 특정 state key 가 새 워커에 할당될 때까지 처리가 멈춤.
    불가능하진 않지만 stateless 잡보다 **비쌈**.
- **Inactivity period length (비활동 기간 길이)**
  - Incremental Sessionizer 와 같이 비용과 세션 포함률의 균형이 필요
    — 길수록 하드웨어 압박과 출력 신선도가 함께 커짐.
- **Inactivity period time (만료 기준 시간)**
  - 완료 세션의 상태 만료를 관리해야 함. 정리는 두 시간축에 기댐
    — ① **event time**(incoming 이벤트의 시각, 더 신뢰할 만해 대개 선호),
    ② **processing time**(현재 시각 기반, 다소 위험).
  - processing time 은 예기치 못한 latency(쓰기 재시도 등)로 세션이 **너무 일찍 만료** 될 수 있음
    — 이 예측 불가능성 때문에 stateful 파이프라인에선 **event time 으로 추론하는 게 항상 더 쉬움**.

#### 구현 예시 (Examples)

**예시 1 — PySpark stateful mapping (Example 5-25)**

만료 컬럼(`event_time`)·grouping key(`visit_id`)를 정의하고 `applyInPandasWithState` 호출:
```python
grouped_visits = (visits_from_kafka.withWatermark('event_time', '1 minute')
  .groupBy(F.col('visit_id')))

visited_pages_type = ArrayType(StructType([StructField("page", StringType()),
  StructField("event_time_as_ms", LongType())]))

sessions = grouped_visits.applyInPandasWithState(
  func=map_visits_to_session,                       # 상태 저장 로직
  outputStructType=StructType([...]),               # 출력 세션 구조
  stateStructType=StructType([                      # state store 와 주고받는 pending 세션 구조
    StructField("visits", visited_pages_type),
    StructField("user_id", StringType())]),
  outputMode="update",                              # 갱신된 row 만 출력
  timeoutConf="EventTimeTimeout"                    # event time 기반 만료
)
```

> 이 부분은 순수 선언부 — 실제 생성·누적 로직은 `map_visits_to_session` 함수에 있음.

**예시 2 — 세션 생성 로직 개요 (Example 5-26, PySpark)**

상태가 타임아웃됐는지 먼저 보고, 살아 있으면 누적 로직 적용:
```python
def map_visits_to_session(visit_id_tuple, input_rows, current_state: GroupState):
  session_expiration_time_10min_as_ms = 10 * 60 * 1000
  visit_id = visit_id_tuple[0]
  visit_to_return = None
  if current_state.hasTimedOut:                     # 만료됨 → 최종 포맷으로 집계 후 제거
    visits, user_id, = current_state.get
    visit_to_return = get_session_to_return(visits, user_id)
    current_state.remove()
  else:
    # ... accumulation logic (누적)
  if visit_to_return:
    yield pandas.DataFrame(visit_to_return)
```

**예시 3 — 만료 기준 시각 탐지 (Example 5-27, PySpark)**

첫 반복엔 watermark 가 없으므로 event time 으로 대체:
```python
should_use_event_time_for_watermark = current_state.getCurrentWatermarkMs() == 0
base_watermark = current_state.getCurrentWatermarkMs()
new_visits = []
user_id = None
for input_df_for_group in input_rows:
  input_df_for_group['event_time_as_ms'] = input_df_for_group['event_time'] \
    .apply(lambda x: int(pandas.Timestamp(x).timestamp()) * 1000)
  if should_use_event_time_for_watermark:           # watermark 없으면 event time 사용
    base_watermark = int(input_df_for_group['event_time_as_ms'].max())
# ... visits accumulation, 생략
```

```
[Table 5-4 재현] event time vs watermark 만료 전략 (watermark 1분, 상태 만료 10분)
──────────────────────────────────────────────────────────────────────
 ┌───────────┬────────────┬───────────────────────────────┬───────────────┐
 │ State key │ Event time │ Expiration times              │ New watermark │
 ├───────────┼────────────┼───────────────────────────────┼───────────────┤
 │     A     │   10:00    │ Watermark 10:10 / Event 10:10 │     09:59     │
 │     A     │   10:01    │ Watermark 10:09 / Event 10:11 │     10:00     │
 │     A     │   10:08    │ Watermark 10:10 / Event 10:18 │     10:07     │
 │     B     │   10:15    │ Watermark 10:17 / Event 10:25 │     10:14     │
 └───────────┴────────────┴───────────────────────────────┴───────────────┘
 key A 는 8분간 활성. watermark 전략이면 B 의 첫 원소 처리 직후(10:10 < 10:14) 방출 가능하나,
 event-time 전략이면 만료(10:18)가 새 watermark(10:14)를 넘어 아직 방출 안 됨.
 ⇒ watermark 는 잡 진행도에 연동되므로 event time 직접 사용보다 더 합리적
──────────────────────────────────────────────────────────────────────
```

**Table 5-4 깊이 읽기 — 워터마크 만료가 event time 만료보다 합리적인 이유**

> **워터마크가 헷갈릴 때** — 워터마크 = "스트림이 '이 시각까지의 데이터는 다 봤다' 고 믿는 시점" 표시.
> 계산은 `워터마크 = (지금까지 본 event time 최대값) − 허용 지연(여기선 1분)` 으로, **항상 최신 이벤트보다 1분 뒤처져** 따라옴.
> 늦게(순서 어긋나) 도착하는 이벤트를 놓치지 않으려는 안전선이며, Spark `EventTimeTimeout` 은 **워터마크가 만료 시각을 넘는 순간** 세션을 타임아웃시킴 
— 즉 **만료를 판정하는 시계 자체가 워터마크**.

[시간순 흐름 — 위 Table 5-4 를 행 순서로]
```
────────────────────────────────────────────────────────────────────────────────────
 10:00 (A) → 워터마크 09:59 / Watermark만료 10:10 / Event만료 10:10  · A 생성
 10:01 (A) → 워터마크 10:00 / Watermark만료 10:09 / Event만료 10:11  · A 누적
 10:08 (A) → 워터마크 10:07 / Watermark만료 10:10 / Event만료 10:18  · A 누적(마지막 활동)
 10:15 (B) → 워터마크 10:14 / Watermark만료 10:17 / Event만료 10:25  · B 생성 + A 만료·방출
────────────────────────────────────────────────────────────────────────────────────
 워터마크 = event − 1분 · Watermark만료 = 직전워터마크 + 10분 · Event만료 = 이벤트시각 + 10분
 * 1행은 워터마크가 아직 0 → 코드가 event time(10:00)으로 대체 → 두 만료가 10:10 으로 동일.
```

**(1) state 가 유지되는 기준 / 바뀌는 기준**

표의 state key 는 곧 세션 키(`visit_id`). A 는 한 방문 세션, B 는 다른 세션.

```
[세션 A 의 일생 — state store 안에서]
──────────────────────────────────────────────────────────────────
 10:00  이벤트(A) ─► A 세션 생성, 만료시각 설정
 10:01  이벤트(A) ─► 기존 A 불러와 누적, 만료시각 미래로 갱신(setTimeoutTimestamp)
 10:08  이벤트(A) ─► 또 누적, 만료시각 또 갱신
        ......... (A 에 새 이벤트 없는 비활동 구간) .........
        워터마크가 A 만료시각 통과 ─► 타임아웃 ─► A 집계·방출 + state 제거
──────────────────────────────────────────────────────────────────
```

- **유지(누적)** — 같은 key 에 **만료 전** 새 이벤트가 계속 도착 → 올 때마다 만료 타임스탬프를 미래로 다시 밀어줌. 1~3행이 모두 A 인 이유.
- **변경(방출·제거)** — ① 워터마크가 그 key 의 만료시각을 **넘으면** 타임아웃(`hasTimedOut`) → 최종 집계 후 `yield`, `current_state.remove()`. ② 새 key(B) 등장 = 별개 세션 신규 생성.

**(2) "New watermark" 칼럼 vs "Expiration times" 안의 watermark / event 차이**

두 곳의 'watermark' 는 서로 다른 것.

- **New watermark 칼럼** = **워터마크 값 그 자체** = `event time − 1분`
  (`10:00→09:59 / 10:01→10:00 / 10:08→10:07 / 10:15→10:14`).
- **Expiration times 칼럼** = 만료 **후보 시각 두 개**(전략 비교). 둘 다 `base + 10분(상태 만료)` 이되 **base 가 다름**:
  - `Watermark NN = (직전 워터마크) + 10분`  ← 잡 진행도 기준
  - `Event NN = (이 이벤트 시각) + 10분`  ← 원시 이벤트 시각 기준

3행(A, 10:08) 검산:
```
 직전 워터마크 = 2행이 남긴 10:00
   Watermark 전략: 10:00 + 10분 = 10:10
   Event 전략:     10:08 + 10분 = 10:18
```
⚠ Watermark 전략의 base 는 "이번 이벤트" 가 아니라 **직전 이벤트가 남긴 워터마크(10:00)** — 워터마크는 마이크로배치 단위로 갱신되어 한 박자 뒤처짐. 그래서 10:08 이벤트인데도 만료가 10:10 으로 잡힘.

**(3) 왜 워터마크 만료가 event time 만료보다 합리적인가**

만료를 닫는 심판이 어차피 워터마크이므로, `event time + 10분` 으로 잡으면 기준은 원시 시각인데 닫는 시계는 1분 뒤처진 워터마크라 **실제로는 `10분 + 워터마크 지연` 만큼 더 오래** 세션이 잔류 → 메모리 점유·출력 지연.

표의 결정적 장면 — B 의 첫 이벤트(10:15) 도착으로 워터마크가 10:14 로 전진:
```
[B 첫 이벤트(10:15) 도착 → 워터마크 10:14 로 전진 → A 닫을지 판정]
──────────────────────────────────────────────────────────────────
 현재 워터마크 = 10:14 (심판 시계)
   ├─ Watermark 전략: A 만료 10:10  →  10:10 < 10:14  ⇒ A 즉시 방출 ✓
   └─ Event 전략:     A 만료 10:18  →  10:18 > 10:14  ⇒ 아직 잔류 ✗
──────────────────────────────────────────────────────────────────
```
⇒ 워터마크는 **잡의 실제 진행도에 연동**되므로, 원시 event time 직접 사용보다 상태 정리가 제때·예측가능하게 일어남.

**(4) "1~3행이 모두 A 인 것도 expiration watermark ≥ new watermark 때문인가?" — 정확히는 아님**

같은 행 안에서 `expiration watermark ≥ new watermark` 는 **모든 행(B 포함)에서 항상 참** — `만료 = 직전워터마크 + 10분` 이라 10분 여유 때문에 늘 큼. 그래서 이 비교로는 A 와 B 를 구분 못 함.

- **key 가 A 인 것** — 입력 데이터의 `visit_id` 가 A 라서. 키는 데이터가 정함.
- **세 이벤트가 한 세션 A 로 유지되는 진짜 기준** — **다음 이벤트가 도착해 전진한 워터마크 vs 직전에 저장돼 있던 만료시각** 의 시간차 비교(같은 행 안이 아님):

```
[유지/만료 판정 — 전진한 워터마크 vs 직전 저장 만료시각]
──────────────────────────────────────────────────────────────────
 Row2 도착: 전진 워터마크 10:00  <  Row1 저장만료 10:10  ⇒ 생존(누적)
 Row3 도착: 전진 워터마크 10:07  <  Row2 저장만료 10:09  ⇒ 생존(누적, 여유 2분)
 Row4(B)도착: 전진 워터마크 10:14 ≥ Row3 저장만료 10:10  ⇒ A 만료·방출
──────────────────────────────────────────────────────────────────
 다음 워터마크 < 직전 저장 만료 → 유지  /  ≥ → 만료
```

즉 1~3행이 A 인 이유는 **세 이벤트가 비활동 10분을 안 넘기고 연달아 도착**해 매번 만료가 미뤄졌기 때문. B 가 와서 워터마크가 10:14 까지 전진하자 비로소 A 의 만료(10:10)를 넘겨 A 가 방출됨.

> **트러블 로그** — Table 5-4 를 읽을 때 **같은 행의 expiration 과 new watermark 를 비교**해 만료 여부를 판단하면 오해함. 만료는 늘 **다음 배치에서 전진한 워터마크**가 **이전에 저장된 만료시각**을 넘는지로 결정됨.
> 예: 위 표에서 3행만 보고 "10:10 ≥ 10:07 이니 A 살아있다" 고 읽으면, 정작 A 를 닫는 사건(B 도착으로 워터마크 10:14)이 4행에 있다는 걸 놓쳐 세션 방출 시점을 한 행 앞으로 잘못 예측하게 됨.
> 만료 추론은 항상 **"전진한 워터마크 vs 저장된 timeout_timestamp"** 한 쌍으로 볼 것.

**예시 4 — 상태·만료 시각 갱신 (Example 5-28, PySpark)**

기존 상태를 불러와 새 visit 을 합치고, 만료 타임스탬프를 다시 설정:
```python
visits_so_far = []
if current_state.exists:
  visits_so_far, user_id, = current_state.get
visits_for_state = visits_so_far + new_visits
current_state.update((visits_for_state, user_id,))     # 상태 갱신

timeout_timestamp = base_watermark + session_expiration_time_10min_as_ms
current_state.setTimeoutTimestamp(timeout_timestamp)   # 만료 시각 재설정
```

**예시 5 — Apache Flink 세션 윈도우 (Example 5-29)**

세션 윈도우는 훨씬 단순 — gap 10분, allowed lateness 15분으로 선언:
```java
sessions: DataStream = (visits_input_data_stream
  .key_by(VisitIdSelector())                              // 세션 키 추출
  .window(EventTimeSessionWindows.with_gap(Time.minutes(10)))   // gap duration 10분
  .allowed_lateness(Time.minutes(15).to_milliseconds())   // 이미 방출된 윈도우에 늦게 합류 허용 15분
  .process(VisitToSessionConverter(), Types.STRING()).uid('sessionizer'))
```

**워터마크 직접 관리(Table 5-4) vs 세션 윈도우(예시 5) — 같은 패턴, 다른 추상화 레벨**

Table 5-4 의 예시 1~4(Spark `applyInPandasWithState`)와 예시 5(Flink `EventTimeSessionWindows`)는
**다른 패턴이 아니라 #30 Stateful Sessionizer 를 두 추상화 레벨로 구현한 것**.
전자는 세션 윈도우를 손으로 조립, 후자는 프레임워크가 세션 윈도우를 기본 제공.

```
[같은 목표 "비활동 10분이면 세션 닫기", 다른 추상화 레벨]
──────────────────────────────────────────────────────────────────
 Table 5-4 방식 (Spark applyInPandasWithState) — 세션 머신을 직접 조립
   · state store 직접 읽기/쓰기 (current_state.get / update / remove)
   · 워터마크 직접 조회 (getCurrentWatermarkMs)
   · 만료시각 직접 계산·설정 (base_watermark + 10분 → setTimeoutTimestamp)
   · 타임아웃 직접 분기 (if current_state.hasTimedOut: …)

 예시 5 방식 (Flink EventTimeSessionWindows) — 파라미터만 선언, 나머지 자동
   · .window(EventTimeSessionWindows.with_gap(10분))
   · .allowed_lateness(15분)
   · 상태·워터마크·만료·세션 병합 전부 프레임워크 내부 처리
──────────────────────────────────────────────────────────────────
```

| 항목 | Table 5-4 방식 (Spark 저수준 stateful) | 예시 5 방식 (Flink 세션 윈도우) |
|---|---|---|
| 추상화 레벨 | 저수준 — 세션 로직 직접 구현 | 고수준 — 세션 윈도우 기본 제공 |
| 상태 관리 주체 | 개발자 (get/update/remove 직접 호출) | 프레임워크 (내부 자동) |
| 워터마크·만료 | 직접 조회·계산·설정 | 내부 자동 (gap 만 선언) |
| 만료 기준 선택 | 자유 — watermark / event / processing time 선택 | 프레임워크가 정함 (event time 기반) |
| 늦은 데이터(late) | 워터마크 지연(1분) 넘으면 버림. 방출·제거된 세션은 안 되살아남 | `allowed_lateness(15분)` 으로 방출된 윈도우에 늦게 합류·재방출 |
| 세션 병합 | 키별 단일 상태에 누적(병합 직접 구현) | gap 사이 이벤트면 두 윈도우 자동 병합 |
| 출력 포맷 | 자유 (직접 yield, 집계 형태 커스텀) | 프레임워크 윈도우 결과 형태 |
| 코드량·실수 위험 | 많음·높음 (만료·멱등성 직접 관리) | 적음·낮음 |

**결정적 차이 — 늦은 데이터(late event) 처리**

실무에서 가장 큰 차이.

- **Table 5-4 방식** — 세션 타임아웃 시 `current_state.remove()` 로 상태를 **지움**.
  같은 `visit_id` 의 늦은 이벤트가 뒤늦게 오면 지워진 상태라 별개의 **새 세션**이 됨(되살릴 로직을 직접 안 짜는 한).
- **예시 5 방식** — `allowed_lateness(15분)` 으로 이미 방출된 세션 윈도우에 **15분 안 늦은 이벤트가 다시 합류**해 결과를 갱신·재방출.
  이 "방출 후 재합류" 가 프레임워크 기본 기능.

```
[late event 가 왔을 때 — visit_id A 세션이 이미 방출·제거된 뒤]
──────────────────────────────────────────────────────────────────
 ✗ Table 5-4 방식:  remove() 된 A 에 늦은 이벤트 도착
                     ⇒ 새 세션 A 가 또 생성 → 한 방문이 둘로 쪼개져 집계
 ✓ 예시 5 방식:     allowed_lateness 15분 안이면 기존 A 윈도우에 합류
                     ⇒ 한 세션으로 갱신·재방출
──────────────────────────────────────────────────────────────────
 ⇒ 표준 gap 세션화 + 늦은 데이터 재합류면 예시 5 가 짧고 안전.
   세션 정의가 비표준이거나 만료 전략·출력 포맷을 직접 통제해야 하면 Table 5-4 방식.
```

> 비유 — 예시 5 는 **자동 변속기**(gap 만 정하면 알아서), Table 5-4 방식은 **수동 변속기**
> (클러치·기어 = 상태·워터마크·만료를 직접 조작). 같은 목적지(세션화)에 가지만 통제권과 책임의 위치가 다름.

> **트러블 로그** — "스트림이니 무조건 저수준 stateful 로 짠다" 고 덤비면 만료·멱등성 함정을 직접 다 떠안음.
> 예: 늦게 5분 도착하는 이벤트를 살려야 하는데 저수준 stateful 로 짜면 `remove()` 된 세션이 새 세션으로 쪼개져
> 한 방문이 둘로 집계됨 — `allowed_lateness` 같은 재합류 로직을 직접 구현해야 함.
> 표준 비활동 gap 세션이면 Flink `EventTimeSessionWindows` 한 줄이 Table 5-4 의 워터마크·만료 코드 수십 줄을 대체하므로,
> 비표준 세션 로직·커스텀 만료가 꼭 필요할 때만 저수준으로 내려가고 아니면 세션 윈도우 API 를 먼저 검토할 것.

| 항목 | Incremental Sessionizer (#29) | Stateful Sessionizer (#30) |
|---|---|---|
| 처리 모드 | 배치(시간별 파티션) | 스트림(near real-time) |
| pending 세션 저장 | 별도 pending 저장소 | state store(in-memory + 내구 동기화) |
| 최선 latency | 약 1시간 | 수 초 |
| 멱등성 처리 | Airflow `ds` 기반 DELETE | checkpoint, at-least-once 유의 |
| 만료 기준 | `expiration_batch_id` | event time(권장) / processing time(위험) |

> **트러블 로그** — Stateful Sessionizer 에서 **session key 나 만료를 processing time 으로** 잡으면 재시작·지연 때 세션이 깨짐.
> 예: session key 에 `현재시각` 을 섞으면, 잡이 checkpoint 부터 at-least-once 로 재처리될 때 같은 visit 이
> 다른 key 로 들어가 세션이 둘로 쪼개짐. 또 processing time 만료를 쓰면 쓰기 재시도로 5분 지연된 사이
> 멀쩡한 세션이 **너무 일찍 만료** 돼 페이지 절반이 누락된 세션이 방출됨.
> session key 는 실행마다 불변인 값(`visit_id` 등)으로 잡고, 만료는 **event time + watermark** 기준으로 둘 것.

---

## 5. 데이터 정렬 (Data Ordering)

집계·결합으로 가치를 더하는 것 말고도, **순서(order)** 자체가 중요한 가치가 됨 —
이벤트를 시간순(chronological)으로 다운스트림 컨슈머 (받아 쓰는 쪽)에게 전달하는 경우.
SQL `ORDER BY` 로 줄여 생각하기 쉽지만, **정렬된 전달(ordered delivery)** 은 의외로 까다로움.

> 책의 비유 — 차량 추적 시스템에서 차들의 위치 순서가 어긋나면,
> 차가 건물을 뛰어넘거나 강을 헤엄치는 것처럼 보임. 그만큼 순서(chronology)가 결정적인 use case 가 많음.

- **#31 Bin Pack Orderer** — **partial commit(부분 커밋)** 저장소에서 bulk 전달을 쓰면서도 순서를 지킬 때.
- **#32 FIFO Orderer** — low latency·대용량이라 단순함이 우선일 때(네트워크 교환을 희생).

---

### 5-1. 패턴 #31: 빈 팩 정렬기 (Bin Pack Orderer)

> 대규모 정렬 전달의 악몽은 **partial commit** 임. bulk API 의 효율은 살리면서, 부분 커밋이 순서를 깨지 않게 묶어 보내는 패턴.

#### 상황 (Problem)

**책의 use case** — 외부 API 로 노출하는 visit 동기화 잡:

- 블로깅 플랫폼이 외부 사이트에 페이지를 embed → 그 embedding 이 만든 visit 이벤트를 자체 이벤트로 처리.
  내부 보관뿐 아니라 **외부 API 로도 노출**(분석용)해야 함.
- 동기화 잡은 **모든 파트너 공통** — **10분 processing time 윈도우** 로 **분당 집계** 를 만들고,
  끝에 파트너별 다른 출력으로 flush. 이벤트는 **분·파트너별 개별 전달, event time 순서** 여야 함.
- **결정적 제약**: API 적재 저장소가 **partial commit 시맨틱의 스트리밍 브로커** 라,
  retry 가 순서를 깰 수 있어 정렬 로직에 각별히 주의해야 함.

> **참고 사항 — Partial Commit (부분 커밋)**
> 고전적 commit 은 success/failure 두 상태지만, partial commit 은 **세 번째 상태** 가 있음 — record 의 **일부만** 적재.
> 왜 순서를 깨나? 10:00·10:10·10:20 세 record 를 bulk 로 보내면 DB 가 **전부·둘·하나·없음** 중 무엇이든 쓸 수 있음.
> 중간 결과(둘·하나)가 위험함 — 어떤 record 가 미전달인지, 언제 성공할지 알 수 없기 때문.
> 예: 10:20 만 써졌으면 10:00·10:10 을 retry → **순서가 뒤집힌 채** 기록됨.
> 스트리밍에서 가장 잘 드러나지만 data-at-rest 에서도 중요 — 일시적으로 일부만 빈 데이터셋이 downstream 처리를 잘못 trigger.
> partial commit 예 — Kinesis `PutRecords`, DynamoDB `BatchWriteItem`, Elasticsearch bulk.

**보충 — "data-at-rest 에서도 중요" 가 무슨 뜻인가**

데이터는 두 상태로 존재. partial commit 문제는 둘 다에 걸침.

```
[data-in-motion vs data-at-rest]
──────────────────────────────────────────────────────────────────
 data-in-motion (= 스트리밍)        data-at-rest (= 정적 저장)
   "흐르는 중" 인 데이터               "멈춰 저장돼 있는" 데이터
   Kafka·Kinesis 토픽,                S3/HDFS 파일, Hive·Delta 파티션,
   브로커를 통과 중                    DB 테이블, Parquet 객체
   비유: 도로를 달리는 택배 트럭        비유: 창고에 적재된 택배 상자
──────────────────────────────────────────────────────────────────
```

문장의 의미 — partial commit 은 **흐르는 데이터만의 문제가 아니라 저장되는 데이터에서도 똑같이 위험**.
스트리밍에선 retry 가 **순서를 뒤집고**, data-at-rest 에선 **완성 안 된 부분 데이터셋을 다운스트림이 완성본으로 오인**해 잘못 처리.

```
[data-at-rest 에서 partial commit 사고]
──────────────────────────────────────────────────────────────────
 적재 잡: 파티션 dt=2026-06-26 에 1000건 bulk 쓰기
   t0  파티션 0건
   t1  partial commit 으로 600건만 써진 순간  ← 일시적 "일부만 찬" 상태
   t2  나머지 400건 마저 써짐 (완성)

 ✗ 다운스트림이 t1 에 파티션을 보고 "데이터 도착" 으로 trigger
    ⇒ 600건만 읽어 집계 → 400건 누락된 잘못된 리포트
──────────────────────────────────────────────────────────────────
 ⇒ 저장 데이터라도 "쓰는 중(부분 상태)" 을 완성으로 오인하면 downstream 이 깨짐
```

반대로 data-in-motion(스트리밍)에선 같은 partial commit 이 **순서 역전**으로 나타남.
Kafka 예 — visit 동기화 잡이 한 파트너 이벤트 3건을 시간순(10:00 → 10:10 → 10:20)으로 한 파티션에 전송:

```
[data-in-motion(Kafka) 에서 partial commit 사고 — 순서 역전]
──────────────────────────────────────────────────────────────────
 프로듀서: 한 배치로 3건 전송 (같은 파티션, key=partner)
   보낸 순서:  [ visit@10:00,  visit@10:10,  visit@10:20 ]

 t1  배치 부분 실패 ── 10:20 만 ack, 10:00·10:10 은 실패(타임아웃/NotEnoughReplicas)
        파티션 로그:  offset0 = 10:20

 t2  실패분 retry ── 10:00·10:10 을 재전송 → 로그 뒤에 append
        파티션 로그:  offset0 = 10:20, offset1 = 10:00, offset2 = 10:10
                       ▲ 가장 늦은 이벤트가 맨 앞 offset
──────────────────────────────────────────────────────────────────
 ✗ 컨슈머가 offset 순서대로 읽음:  10:20 → 10:00 → 10:10
    ⇒ 차량 추적이면 차가 10:20 위치에서 10:00 위치로 "시간을 거슬러 점프"
    ⇒ 책 비유처럼 차가 건물을 뛰어넘거나 강을 헤엄치는 것처럼 보임
──────────────────────────────────────────────────────────────────
 ⚠ 원인 — retries 켜짐 + max.in.flight.requests.per.connection > 1 이면 retry 배치가
    뒤늦게 append 되어 파티션 안에서 순서가 뒤집힘.
 ✓ Kafka 자체 완화 — enable.idempotence=true (멱등 프로듀서) 면 브로커가 시퀀스 번호로
    재정렬을 막아 파티션 내 순서를 보존. 단 여러 파티션·여러 토픽 간 순서는 여전히 보장 못 함.
```

**핵심 대비** — 같은 partial commit, 다른 증상:

| 상태 | partial commit 으로 일부만 써진 순간 → 증상 |
|---|---|
| data-in-motion (Kafka 스트리밍) | retry 분이 뒤 offset 에 붙어 **도착 순서 역전** ⇒ 컨슈머가 시간 거꾸로 된 이벤트 처리 |
| data-at-rest (정적 저장) | 부분만 찬 데이터셋을 **완성으로 오인** ⇒ 누락된 채 집계(앞 예시 400건 누락) |

> **참고 사항 — Readiness Marker 와의 연결**
> data-at-rest 의 "부분 완성을 완성으로 오인" 은 **Readiness Marker(준비 완료 표식)** 패턴으로 방어.
> 1000건을 다 쓴 뒤 마지막에 `_SUCCESS` 같은 마커를 찍고, 다운스트림은 **마커가 있을 때만** 읽게 해 t1 같은 중간 상태를 못 보게 함.

#### 해결 (Solution)

record 를 **개별 전달** 하면 순서는 지키지만 네트워크 오버헤드가 큼(record 수만큼 요청을 초기화).
bulk operation 에 **Bin Pack Orderer** 를 결합하면, partial commit 맥락에서도 순서를 보장하면서 bulk 의 효율을 살림.

두 단계로 동작함.
- **① 정렬** — 관련 이벤트를 모아 **grouping key + event time 으로 sort**.
- **② bin packing** — 정렬된 row 를 **delivery bin 으로 묶되, 한 bin 에 grouping key 가 하나씩만** 오게 함
  (서로 다른 엔티티의 "같은 순번" 이벤트를 같은 bin 에 모음).
  ⇒ completeness·중복·partial commit 걱정 없이 bulk API 로 보낼 수 있는 **격리된 부분집합(isolated subset)** 이 생김.

워크플로:
- record 를 grouping key·time 으로 정렬.
- 정렬된 row 를 delivery bin 에 배치 — 각 bin 엔 grouping key 가 **한 번만**. (bin = array/list)
- bin 을 **순차(sequential) emit**. 한 bin 안에서 retry 가 나도 그 bin 에 국한됨 —
  bin 당 grouping key 가 한 번만 등장하니 retry 가 그 key 의 순서를 못 깸.
  현재 bin 이 완전히 쓰이기 전엔 다음 bin 을 안 보냄.

```
[Figure 5-9 재현] Bin packer — 3개 bulk 요청으로 순서를 지키며 전달
──────────────────────────────────────────────────────────────────────
 입력(정렬 전)        ──ORDER BY key, time──►   정렬 후
   Key=1  10:00                                  Key=1  09:04
   Key=2  09:04                                  Key=1  10:00
   Key=1  09:04                                  Key=1  10:04
   Key=1  10:04                                  Key=2  09:04
   Key=3  10:15                                  Key=2  10:00
   Key=2  10:00                                  Key=3  10:15
                                                     │ Delivery Bins 생성 (bin 당 key 1회)
                                                     ▼
 Delivery Bin 1     Delivery Bin 2     Delivery Bin 3
 ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
 │ Key=1  09:04 │   │ Key=1  10:00 │   │ Key=1  10:04 │
 │ Key=2  09:04 │   │ Key=2  10:00 │   └──────────────┘
 │ Key=3  10:15 │   └──────────────┘
 └──────────────┘
       │ ①                │ ②                │ ③
       └──────────────────┴───────────────────┴────►  [출력 스토어]  flush one by one, in order
──────────────────────────────────────────────────────────────────────
 각 key 의 1·2·3번째 이벤트가 bin 1·2·3 에 흩어짐 → bin 을 순서대로 flush 하면 key 안의 순서가 보존됨.
 한 bin 엔 key 가 한 번만 등장 ⇒ 그 bin 의 partial commit·retry 가 key 순서를 깨지 못함.
```

**쉽게 풀어 보면** — 핵심은 **"한 bulk 요청(=bin) 안에 같은 key 를 두 번 넣지 않는 것"**.
한 요청에 같은 key 의 여러 이벤트가 섞여 있으면 partial commit 으로 일부만 써지고 나머지가 retry 돼 순서가 뒤집힘.
key 의 이벤트들을 **서로 다른 bin 에 한 건씩** 흩고 bin 을 순서대로 flush 하면, 한 요청이 부분 실패해도 그 key 순서는 그대로 유지됨.

```
[쉽게 풀어 보면 — 왜 bin 으로 나누면 partial commit 이 순서를 못 깨나]
──────────────────────────────────────────────────────────────────────
 ✗ bin 없이: 한 key 의 3건을 한 bulk 에 담음
   bulk = [ Key1@09:04, Key1@10:00, Key1@10:04 ]
   partial commit 으로 10:04 만 성공, 나머지는 나중에 retry
   ⇒ 10:00 이 10:04 "뒤" 에 기록됨 → 순서 깨짐

 ✓ bin 으로 나눔: 한 key 는 bin 마다 1건씩
   Bin1 = [ Key1@09:04 … ]   Bin2 = [ Key1@10:00 … ]   Bin3 = [ Key1@10:04 … ]
   Bin1 이 완전히 성공해야 Bin2 전송 (순차 flush)
   ⇒ 한 bin 안엔 Key1 이 1번뿐 → 그 bin 이 부분 실패·retry 돼도 Key1 순서는 불변
──────────────────────────────────────────────────────────────────────
```

#### 고려사항 (Consequences)

순서 보장에는 좋아 보이지만, compute 런타임이 받쳐주지 않으면 일부 item 이 여전히 어긋날 수 있음.

- **Retries (재시도)**
  - 패턴은 **같은 실행 내** 에서만 순서를 보장함.
    파이프라인 전체가 실패해 retry 하면, **이미 emit 된 결과까지 다시** 섞이게 됨 →
    패턴을 써도 전체 순서가 깨질 수 있음.
- **Complexity (복잡도)**
  - bin packer 는 고전적 sort 보다 구현이 확실히 어려움 — 커스텀 정렬·bin 생성 로직이 필요한 반면,
    고전 정렬은 적절한 정렬 함수 한 번 호출이면 끝.

#### 구현 예시 (Examples)

**예시 1 — bin packer 준비 단계 (Example 5-30, PySpark)**

파티션 내에서 `visit_id`·`event_time` 으로 로컬 정렬(네트워크 교환 없음) 후 파티션별로 Kinesis 전송:
```python
(events.sortWithinPartitions([F.col('visit_id'), F.col('event_time')])
   .foreachPartition(lambda rows: write_records_to_kinesis(...)))
```

**예시 2 — Kinesis 용 Bin Pack Orderer (Example 5-31)**

정렬된 row 를 돌며, `visit_id` 가 바뀌면 bin 위치를 0으로 리셋해 같은 key 의 연속 등장을 서로 다른 bin 에 배치:
```python
def write_records_to_kinesis(output_stream, visits_rows):
  producer = boto3.client('kinesis')
  delivery_groups = []
  groups_index = 0
  last_visit_id: Optional[str] = None
  for visit in visits_rows:
    if visit.visit_id != last_visit_id:   # key 가 바뀌면
      last_visit_id = visit.visit_id
      groups_index = 0                     # 다시 bin 0 부터
    if len(delivery_groups) <= groups_index:
      delivery_groups.append([])           # 필요한 만큼 bin 생성
    delivery_groups[groups_index].append(visit)
    groups_index += 1                       # 같은 key 의 다음 등장은 다음 bin 으로
```

> 정렬 덕분에 같은 `visit_id` 가 연속으로 나오고, 그 1·2·3번째가 bin 0·1·2 로 흩어짐.
> 이후 bin 을 0→1→2 순서로 flush 하면 각 key 의 순서가 그대로 지켜짐.

| 항목 | Bin Pack Orderer (#31) | FIFO Orderer (#32) |
|---|---|---|
| 전제 | partial commit + bulk 효율 둘 다 필요 | 단순함·ASAP 전달 우선 |
| 동작 | 정렬 후 bin 으로 묶어 순차 bulk flush | record 하나씩 ack 받고 다음 전송 |
| 비용 | 구현 복잡, 네트워크 효율↑ | 구현 단순, I/O 오버헤드↑ |

> **트러블 로그** — partial commit 저장소에 bin packing 없이 bulk 로 보내면, 부분 커밋 + retry 로 순서가 뒤집힘.
> 예: Kinesis `PutRecords` 로 한 차량의 위치 이벤트 10건을 한 번에 보냈는데 3·5번째만 실패 → 그 둘만 retry →
> 5번째가 6·7번째 **뒤에** 기록돼, 지도에서 차가 뒤로 점프하는 것처럼 보임.
> 한 bin 에 한 차량의 위치를 **한 건만** 담고 bin 을 순서대로 flush 하면, 한 bin 의 부분 실패가 그 차량 순서를 못 깸.

---

### 5-2. 패턴 #32: 선입 선출 정렬기 (FIFO Orderer)

> Bin Pack Orderer 가 partial commit + bulk 최적화를 노린다면, FIFO Orderer 는 **단순함을 위해 네트워크 효율을 포기** 하는 더 가벼운 대안.

#### 상황 (Problem)

**책의 use case** — ASAP 전달이 필요한 스트리밍 잡:

- visits 에서 특정 이벤트 **subset 을 감지** 해, 처리 순서대로 다른 스트림에 전달하는 스트리밍 잡.
- **각 record 를 가능한 빨리(ASAP) 전달** 해야 함 → 네트워크를 아끼려는 buffering 은 선택지가 아님.
- **결정적 제약**: low latency 가 우선이라 bulk·버퍼링을 못 씀 → 가볍고 단순한 순서 보장이 필요.

#### 해결 (Solution)

전달 제약이 더 느슨하다면 Bin Pack 보다 단순한 **FIFO Orderer** 가 맞음.
정렬 알고리즘이 필요 없음 — 요구가 **first in, first out** 으로 보내는 것뿐이라, record 를 감지해 전달 요청하면 됨.

- 핵심은 **각 record 의 전달 ack 를 받은 뒤 다음으로** 진행하는 것. 안 그러면 순서가 깨지거나 유실됨.
- 구현은 두 가지.
  - **개별 전달 API** — Kinesis `PutRecord`, Kafka `send(...)` + 동기 `flush(...)`.
  - **bulk API + concurrency=1** — full commit 시맨틱 저장소에서만. 동시에 너무 많은 bulk 를 보내지 않게 함
    (하나라도 실패하면 out of order). Kafka 는 `max.in.flight.requests.per.connection=1`,
    또는 **idempotent producer**(최대 5 concurrent 를 허용하면서도 순서 보장).

> **참고 사항 — In-Flight Requests (전송 중 요청)**
> in-flight 요청은 throughput 최적화에 좋음 — 첫 bulk 의 응답을 안 기다리고 다음을 만들어 보냄.
> 하지만 순서를 깰 수 있음 — 두 in-flight 중 **둘째가 성공하고 첫째가 retry** 되면 그 사이 순서가 깨짐.

**보충 — "ack 받은 뒤 다음으로 진행" 이 무슨 뜻인가**

ack(acknowledgment) = "나 이거 잘 받았어" 라는 **수신 확인 응답**. record 를 보내면 받는 쪽(브로커·저장소)이 도착 완료 신호를 돌려줌.
FIFO Orderer 는 정렬 알고리즘 없이, 순서대로 들어온 record 를 **"하나 보내고 → ack 받고 → 다음 보내고"** 식으로 **한 번에 하나씩 직렬** 처리.

```
[FIFO 정상 — ack 받고 다음 (concurrency = 1)]
──────────────────────────────────────────────────────────────────
 R1(10:00) send ─► ack ✓ ─► R2(10:10) send ─► ack ✓ ─► R3(10:20) send ─► ack ✓
                   │                          │
             이 확인을 받아야           그제서야 다음을 보냄
──────────────────────────────────────────────────────────────────
 ⇒ 이전 게 확실히 도착한 뒤에만 다음을 보내므로, 정렬 없이도 항상 오래된 것부터 = FIFO
```

ack 를 안 기다리고 **여러 개를 동시에(in-flight) 쏘면** 두 사고가 남.

```
[✗ 사고1 — ack 안 기다림 → 순서 역전]
──────────────────────────────────────────────────────────────────
 R1, R2, R3 를 ack 안 기다리고 동시 전송
   R1(10:00) 실패 → retry      R2(10:10) 성공      R3(10:20) 성공
   retry 된 R1 이 뒤늦게 도착
 ⇒ 도착 순서: 10:10, 10:20, 10:00   ← 가장 오래된 게 맨 뒤로 밀림
──────────────────────────────────────────────────────────────────

[✗ 사고2 — ack 안 기다림 → 유실]
──────────────────────────────────────────────────────────────────
 R1(10:00) send ── (전송 실패했지만 ack 확인 안 함) ── 바로 R2 로 넘어감
 ⇒ R1 은 도착 못 했는데 아무도 모름 → 10:00 이벤트 영구 유실
    (ack 를 기다렸다면 "R1 실패" 를 알고 재전송했을 것)
──────────────────────────────────────────────────────────────────
```

⇒ ack 를 기다려야 ① 앞의 것이 도착한 뒤의 것이 가게되어 **순서 보장**, ② 실패를 ack 로 잡아내 **유실 방지**.
그래서 구현도 개별 전달 API(`PutRecord`, Kafka `send` + 동기 `flush`) 또는 bulk 라도 `max.in.flight.requests.per.connection=1` 로 **동시에 하나만 전송 중** 이게 강제.
- `max.in.flight.requests.per.connection`: 카프카 프로듀서가 브로커로부터 ACK(응답)를 받지 않고도 전송할 수 있는 최대 요청(배치) 수를 의미하는 핵심 성능 및 순서 보장 옵션
    - 높은 처리량: 값을 높이면 대기 시간 없이 여러 요청을 동시에 전송할 수 있어 네트워크 처리량이 향상.
    - 메시지 순서 역전 문제: 이 값이 1보다 클 때(예: 5), 첫 번째 요청(1)이 실패하고 두 번째 요청(2)이 성공하면, 브로커에는 2가 먼저 기록된 후 재시도된 1이 기록되어 순서가 뒤바뀔 위험
    - 순서가 중요할 때: 순서가 생명인 서비스(예: 은행 잔고 변경, 로그 순서)라면 이 값을 1로 설정

**쉽게 풀어 보면** — FIFO 는 **"하나 보내고 그 ack 를 받은 뒤에야 다음을 보내는 것"** — 그래서 정렬 알고리즘 없이도 항상 오래된 것부터 전달됨.
대신 record 마다 왕복이 생겨 I/O·지연이 커지고, **exactly-once 는 보장하지 못함**(ack 가 실패하면 같은 record 를 재전송 → 중복).

```
[쉽게 풀어 보면 — FIFO 는 "하나 보내고 ack 받고 다음", 단 중복은 못 막음]
──────────────────────────────────────────────────────────────────────
 정상 흐름 (순서 보장)
   R1 send → ack ✓ → R2 send → ack ✓ → R3 send → ack ✓
   (이전 ack 를 받아야 다음 전송 ⇒ 항상 오래된 것부터 = FIFO)

 ✗ exactly-once 는 아님
   R1 send 성공 → ack 전송 중 실패 → 잡 재시작
   ⇒ producer 가 R1 을 "미전달" 로 보고 재전송 → R1 중복
   완화: 챕터 4 멱등성 패턴(중복을 무해하게)
──────────────────────────────────────────────────────────────────────
 순서는 지키지만 record 마다 왕복(ack)이라 I/O·지연↑ → 멀티스레딩 시 엔티티별로 scope 를 고정할 것.
```

#### 고려사항 (Consequences)

단순함이 매력이지만, FIFO 전달이 걸린 모든 use case 에 무턱대고 쓸 건 아님.

- **I/O overhead and latency (I/O 오버헤드·지연)**
  - 가장 큰 단점 — record 마다 한 요청이라 I/O 오버헤드와 지연이 커짐.
    전달할 데이터가 많고 분당 전달 건수를 보는 모니터링 대시보드가 있으면 특히 두드러짐.
  - **멀티스레딩** 으로 완화 가능(여러 프로세스에서 개별 요청). 단 프로세스 간 순서 보장이 문제 —
    서로 격리돼 있기 때문. 좋은 전략은 **정렬 대상의 scope 를 엔티티별로 만드는 것** —
    한 user/product 의 모든 record 를 같은 프로세스에 할당.
    (bin 과 비슷하나, 각 컨테이너가 같은 엔티티의 모든 record 를 담아 비동기로 개별 전달함.)

**보충 — "프로세스" 의 의미, 그리고 엔티티·레코드·프로세스의 관계**

  여기서 **프로세스**는 OS 의 그 프로세스라기보다 **독립적으로 동시에 record 를 전달하는 병렬 실행 단위(일꾼)**.
  실무 도구로는 멀티스레딩의 스레드/워커, Spark executor task, Kafka consumer group 의 컨슈머 인스턴스, Flink subtask 등.
  FIFO 는 record 1건마다 왕복이라 느린데, 여러 프로세스가 동시에 전달해 속도를 올림 — 그러나 프로세스가 서로 격리돼 순서 문제가 생김.

  ```
  세 개념
  ──────────────────────────────────────────────────────────────────
   레코드(record)   = 데이터 한 건(이벤트 하나)            예: visit@10:00
   엔티티(entity)   = 순서가 의미 있는 단위 = grouping key   예: user A, product 17
                      └ 한 엔티티는 여러 레코드를 가짐
   프로세스(process) = 레코드를 병렬 전달하는 일꾼            예: 스레드 P1, P2
  ──────────────────────────────────────────────────────────────────
   핵심 — 순서는 "엔티티 안에서만" 의미가 있음.
   user A 의 이벤트끼리는 시간순이 중요하지만, user A 와 user B 사이 순서는 무의미.
  ```

  관계 — **레코드 ∈ 엔티티 → 엔티티를 프로세스에 통째로 할당**.

  ```
  [✗ 잘못 — 레코드를 프로세스에 라운드로빈 분배]
  ──────────────────────────────────────────────────────────────────
   P1 ← A1, A3, A5, B2      P2 ← A2, A4, B1, B3
     같은 user A 레코드가 P1·P2 에 흩어짐 → 비동기 전달로 A2 가 A1 보다 먼저 도착 가능
   ⇒ user A 순서 깨짐
  ──────────────────────────────────────────────────────────────────

  [✓ 올바름 — 엔티티 단위로 프로세스에 고정 (= scope 를 엔티티별로)]
  ──────────────────────────────────────────────────────────────────
   P1 ← user A 전체 (A1→A2→A3→A4→A5, 프로세스 안에서 FIFO)
   P2 ← user B 전체 (B1→B2→B3, 프로세스 안에서 FIFO)
   ⇒ A 순서 보장 + B 순서 보장, 그리고 A·B 는 병렬 ⇒ 빠르면서 순서도 OK
  ──────────────────────────────────────────────────────────────────
   * Kafka 파티셔닝과 같은 원리 — key=user_id 면 같은 user 는 늘 같은 파티션,
     파티션 안에선 순서 보장, 컨슈머는 파티션 단위로 병렬 처리.
  ```

- **FIFO is not exactly once (FIFO 는 정확히 한 번이 아님)**
  - FIFO 는 **가장 오래된 것부터 전달** 한다는 뜻일 뿐, exactly-once 를 보장하지 않음.
    예: `producer.send(message)` → `consumer.ack(message)` 에서 send 성공 후 **ack 가 실패** 할 수 있음 →
    재시작 시 producer 가 **이미 전달된 record 를 재전송** 함.
  - 완화 — 챕터 4의 멱등성 패턴에 기댐.

**보충 — "ack 실패 → 재전송 → 중복" 이 왜 생기나**

핵심은 재전송 여부가 아니라, **프로듀서가 "ack 없음" 으로는 무슨 일이 났는지 구분 못 함(모호함)**.

```
[ack 를 못 받았다 — 진짜 무슨 일이 일어났나?]
──────────────────────────────────────────────────────────────────
 시나리오 A (record 유실)
   프로듀서 ──R1──✗(가는 길에 사라짐)   브로커는 R1 받은 적 없음
   ⇒ 재전송해야 맞음 (안 하면 R1 영구 유실)

 시나리오 B (record 도착, ack 만 유실)
   프로듀서 ──R1──► 브로커 (R1 정상 기록됨!)
   브로커 ──ack──✗(돌아오는 길에 사라짐)
   ⇒ 재전송하면 R1 또 기록 → 중복
──────────────────────────────────────────────────────────────────
 프로듀서가 보는 건 둘 다 똑같음: "R1 보냈는데 ack 안 왔다." A·B 구분 불가.
   · 재전송 안 함 → A 일 때 유실
   · 재전송 함   → B 일 때 중복
 at-least-once 는 유실을 더 나쁘게 봐 무조건 재전송 → B 에서 중복이 대가.
──────────────────────────────────────────────────────────────────
```

⚠ 흔한 오해 — "ack 가 돌아오다 실패하면 프로듀서가 안 보낸다" 는 **반대**.
프로듀서는 ack 를 **못 받은 것 자체가 재전송 트리거** — 타임아웃까지 ack 안 오면(가다 유실이든 돌아오다 유실이든 똑같이 안 옴) `retries` 로 재전송. 이미 기록됐는지는 알 도리가 없음.

⚠ `max.in.flight.requests.per.connection=1` 의 역할 혼동 주의 — 이건 **순서 보장**이지 중복 방지가 아님.
"동시 응답 대기 1개" 라 retry 가 뒤 record 를 추월 못 해 **순서 역전만 막음**. 그 하나를 재전송하는 중복은 그대로.

```
[max.in.flight=1 이어도 중복은 남음]
──────────────────────────────────────────────────────────────────
 R1 send ─► (브로커 기록 성공) ─► ack 유실 ─► 타임아웃 ─► R1 재전송 ─► R1 또 기록
   동시 요청은 내내 1개(순서 OK)였지만 R1 이 두 번 기록됨(중복)
──────────────────────────────────────────────────────────────────
```

✓ 중복을 막는 건 `enable.idempotence=true`(멱등 프로듀서) — record 에 **producer ID + 시퀀스 번호**를 붙이면,
브로커가 이미 기록한 시퀀스 번호의 재전송을 **버리고도 ack 는 정상 응답** → Kafka 내부 exactly-once.

```
 일반 프로듀서:  재전송 R1        ─► 브로커 그냥 또 기록            → 중복
 멱등 프로듀서:  재전송 R1(seq=5) ─► 브로커 "seq=5 이미 있음" 버림 + ack ✓ → 중복 없음
```

**상세 — 멱등 프로듀서가 중복을 알아보는 메커니즘**

핵심은 **재전송이 와도 브로커가 "이미 쓴 것" 인지 알아보게 만드는 지문**. 그 지문이 PID + 시퀀스 번호.

```
[두 가지 지문]
──────────────────────────────────────────────────────────────────
 PID (Producer ID) = 프로듀서가 처음 연결할 때 브로커가 발급하는 고유 번호
                     (InitProducerId 요청 → 예: PID=4242). 프로듀서 세션마다 1개.
 시퀀스 번호(seq)   = (PID, 파티션) 조합마다 0 부터 1씩 증가하는 일련번호
                     R1=seq0, R2=seq1, R3=seq2 ... (각 record/배치에 매겨짐)
──────────────────────────────────────────────────────────────────
 ⇒ 모든 record 에 "누가(PID) 보낸, 몇 번째(seq) record 인가" 지문이 찍힘.
```

브로커(파티션 리더)는 **PID 별 "마지막으로 기록한 seq"**(`lastSeq`)를 저장해 두고, 들어온 seq 와 비교해 3분기 처리.

```
[브로커의 시퀀스 판정]
──────────────────────────────────────────────────────────────────
 seq == lastSeq + 1  → 정상(다음 차례) → 기록, lastSeq 증가, ack ✓
 seq <= lastSeq      → 이미 쓴 것 = 중복 재전송 → 안 씀(버림) + ack ✓ (성공 응답)
 seq >  lastSeq + 1   → 중간이 빔(gap) → OutOfOrderSequenceException (뭔가 유실)
──────────────────────────────────────────────────────────────────
 "버리고도 ack 정상 응답" = 가운데 줄. 중복이라 안 쓰되, 프로듀서가 성공으로 알고
 다음으로 넘어가도록 성공 ack 를 돌려줌(오류로 응답하면 무한 재시도·정지하므로).
```

시나리오 B(record 는 기록됐는데 ack 만 유실)를 seq 로 따라가면:

```
[멱등 프로듀서 — ack 유실 시 중복이 안 생기는 흐름]
──────────────────────────────────────────────────────────────────
 init → 브로커가 PID=4242 발급, lastSeq[4242] = -1

 R1 전송 (PID=4242, seq=0)
   브로커: 0 == (-1)+1 → 기록 ✓, lastSeq=0
   브로커 ──ack──✗ (돌아오는 길에 유실)

 프로듀서: ack 못 받음 → 타임아웃 → R1 재전송 (PID=4242, seq=0)  ← 같은 seq!
   브로커: 0 <= lastSeq(0) → 이미 있음 → 안 씀(버림) + ack ✓ (성공)

 프로듀서: 이제 ack 받음 → R2(seq=1) 진행
 ⇒ R1 은 정확히 한 번만 기록됨
──────────────────────────────────────────────────────────────────
 일반 프로듀서면 재전송 R1 을 또 써서 중복. 멱등은 재전송도 seq=0 그대로라 알아보고 버림.
```

덤 — 순서·동시성 동시 확보. `seq > lastSeq+1`(중간 빈) 배치를 브로커가 거부하므로,
여러 요청이 동시에 떠 있어도(in-flight) 순서 어긋남을 브로커가 잡아냄.
그래서 `max.in.flight.requests.per.connection` 을 1 까지 안 낮추고 **5 까지 허용하면서도 순서 보장**
(앞 예시 `enable.idempotence=true # 순서 보장하며 동시성↑` 의 의미).

> **참고 사항 — 전제 설정**
> 멱등 프로듀서는 내부적으로 `acks=all`, `retries>0`, `max.in.flight<=5` 를 요구.
> Kafka 3.0+ 부터는 `enable.idempotence=true` 가 기본값.

**상세 — 멱등성을 켜면 `max.in.flight=1` 은 불필요(상위 호환)**

멱등성이 켜지면 **중복 방지 + (파티션 내) 순서 보장이 함께** 따라옴 → `max.in.flight=1` 로 따로 낮출 필요 없음.

| 설정 | 중복 방지 | 순서 보장(파티션 내) | 처리량 |
|---|---|---|---|
| `max.in.flight=1` (멱등 off) | ✗ 재전송 시 중복 | ✓ 한 번에 하나라 추월 불가 | 낮음 |
| `max.in.flight=5` (멱등 off) | ✗ | ✗ retry 가 뒤 요청 추월 | 높음 |
| **`enable.idempotence=true`** (max.in.flight≤5) | ✓ seq 로 중복 버림 | ✓ seq 로 순서 검증 | 높음 |

- `max.in.flight=1` **단독은 순서만 잡고 중복은 못 잡음** — 한 번에 하나만 보내도 그 ack 가 유실돼 재전송하면 중복(시나리오 B). **반쪽짜리**.
- `enable.idempotence=true` 는 **중복·순서 둘 다** 잡음 → `max.in.flight=1` 의 **상위 호환**. 멱등성만 켜면 순서는 자동으로 따라옴.

```
[언제 무엇을 쓰나]
──────────────────────────────────────────────────────────────────
 멱등성 쓸 수 있으면      → enable.idempotence=true (이걸로 끝, max.in.flight=1 불필요)
 멱등성 못 쓰는 환경이면   → max.in.flight=1 로 순서만 지킴 (중복은 챕터 4 패턴으로 따로)
   예: Kafka 0.11 미만(멱등성 미지원), acks=all 등 전제를 못 맞추는 특수 제약.
   요즘(3.0+)은 기본값이라 사실상 항상 켜져 있음.
──────────────────────────────────────────────────────────────────
 역사 — 멱등성 이전엔 retries>0 + max.in.flight>1 이면 순서가 깨져서,
        순서 지키는 유일한 방법이 max.in.flight=1(느림)이었음.
        멱등 프로듀서가 이 "순서 ↔ 처리량" 트레이드오프를 없앰.
```

⚠ 단 멱등성의 순서 보장 범위는 **"한 파티션 안 + 한 프로듀서 세션"** (`max.in.flight=1` 도 같은 한계).
여러 파티션에 걸친 **전역 순서**나 **잡 재시작을 가로지른 순서·중복**은 둘 다 보장 못 함 → 트랜잭션(`transactional.id`) 필요.
⇒ FIFO Orderer 에서 "엔티티별 순서" 가 목표면 **같은 엔티티 → 같은 파티션**(key 파티셔닝) + `enable.idempotence=true` 면 순서·중복이 한 번에 해결.

단 멱등 프로듀서도 **한 프로듀서 세션 + Kafka 내부**에서만 막음. **잡 재시작**(새 producer ID 로 시퀀스 추적 끊김)이나
**소스 쪽 ack 실패**(`send` 성공 후 `consumer.ack` 실패 → 재시작 시 소스 메시지 다시 읽어 또 전송)는 여전히 중복 → 챕터 4 멱등성 패턴으로 보완.

#### 구현 예시 (Examples)

**예시 1 — 개별 전달 (Example 5-33, Kafka)**

record 하나 produce 후 즉시 flush — 쉽지만 record 당 한 요청이라 네트워크 비쌈:
```python
producer.produce(...)
producer.flush()
```

**예시 2 — bulk + concurrency=1 (Example 5-34)**

`flush` 없이 producer 가 최대 1건씩만 동시 처리하게 해 동시 쓰기로 인한 순서 붕괴를 막음:
```python
producer = Producer({
   'max.in.flight.requests.per.connection': 1,   # 동시 1건만
   'queue.buffering.max.ms': 1000
})
producer.produce(...)
```

**예시 3 — idempotent producer (Example 5-35)**

idempotent producer(KIP-98)는 **최대 5 concurrent 를 허용하면서도 순서 보장** — 버퍼링 시간을 늘려 bulk 를 더 채움:
```python
producer = Producer({
  'max.in.flight.requests.per.connection': 5,
  'enable.idempotence': True,                    # 순서 보장하며 동시성↑
  'queue.buffering.max.ms': 2000
})
producer.produce(...)
```

**예시 4 — Kinesis SequenceNumberForOrdering (Example 5-36)**

bulk 불가(partial commit) 저장소면 개별 요청. 직전 sequence number 를 넘겨 순서를 강제:
```python
records_to_deliver = [...]
previous_sequence_number = None
for record in records_to_deliver:
  put_result = client.put_record(StreamName=..., Data=...,
    SequenceNumberForOrdering=previous_sequence_number)   # 없으면 rough order
  previous_sequence_number = put_result.sequence_number
```

**예시 5 — GCP Pub/Sub ordering_key (Example 5-37)**

`ordering_key` 는 grouping key 처럼 작동 — 같은 key 를 공유하는 record 를 FIFO 로 전달(같은 publisher·single region 내에서만):
```python
records_to_deliver = [Record(data=..., ordering_key="a"),
  Record(data=..., ordering_key="b"), Record(data=..., ordering_key="c"),
  Record(data=..., ordering_key="a")]
for record in records_to_deliver:
  publisher.publish(..., data=..., ordering_key=record.ordering_key)
```

> **트러블 로그** — FIFO 를 멀티스레딩으로 가속하면서 엔티티 scope 를 안 나누면 순서가 깨짐.
> 예: `user_id` 무관하게 4개 스레드로 개별 전송하면, 같은 user 의 `click → purchase` 가 서로 다른 스레드에서 경합해
> `purchase` 가 먼저 도착 → 전환 분석이 역전됨.
> 같은 엔티티(user)의 record 는 **같은 스레드/프로세스에 고정** 할 것.
> 또 ack 실패로 재전송될 수 있으니, exactly-once 가 필요하면 챕터 4의 멱등성 패턴으로 보완할 것.

---

## 6. 요약 (Summary)

챕터 5는 데이터셋의 **가치를 높이는** 방법을 다뤘음. 크게 두 시나리오로 나뉨.

- **정보를 더해서** 가치를 높이는 경우
- **정보를 줄여서** 더 이해하기 쉽게 만드는 경우

```
[챕터 5 한눈에 — 데이터 가치 패턴 5갈래]
──────────────────────────────────────────────────────────────────────
 정보 추가 ┌ 5.1 Data Enrichment    다른 데이터셋과 결합       #23 Static / #24 Dynamic Joiner
          └ 5.2 Data Decoration   raw record 에 주석     #25 Wrapper / #26 Metadata Decorator

 정보 축약 ┌ 5.3 Data Aggregation   대량을 요약             #27 Distributed / #28 Local Aggregator
          └ 5.4 Sessionization    이벤트를 세션으로 묶음     #29 Incremental / #30 Stateful Sessionizer

 순서 보장   5.5 Data Ordering      시간순 전달             #31 Bin Pack / #32 FIFO Orderer
──────────────────────────────────────────────────────────────────────
```

- **정보 추가 시나리오**
  - **Data Enrichment** — 데이터셋 결합. 동질 파이프라인뿐 아니라 **스트리밍 + 배치** 이질 환경에서도 가능.
  - **Data Decoration** — raw record 에 주석. **Wrapper** 는 원본과 계산값을 분리(데이터 표현의 일관성 확보),
    **Metadata Decorator** 는 그 계산값을 **메타데이터 레이어에 숨겨** 최종 사용자에게 노출하지 않음.
- **정보 축약 시나리오**
  - **Data Aggregation** — 분산(Distributed) 또는 로컬(Local) 환경에서 요약.
  - **Sessionization** — 증분 배치(Incremental) 또는 실시간 스트리밍(Stateful)으로 사용자 경험을 세션으로 요약.
- **순서 보장**
  - **Data Ordering** — **Bin Pack Orderer** 는 partial commit 맥락에서 순서를 지키고,
    **FIFO Orderer** 는 네트워크 교환을 희생해 단순함을 택하는 대안.

> 다음 여정 — 지금까지 **수집(2장) · 오류 관리(3장) · 멱등성(4장) · 데이터 가치(5장)** 패턴을 익혔음.
> 아직 한 가지가 남음 — **이 모든 걸 어떻게 연결하나?** 그게 챕터 6(데이터 흐름)의 주제.
