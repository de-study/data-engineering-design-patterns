# 데이터 엔지니어링 디자인 패턴 - 오류 관리 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 3 | 실무 데이터 엔지니어링 관점 정리

---

## 목차

1. [처리할 수 없는 레코드 (Unprocessable Records)](#1-처리할-수-없는-레코드-unprocessable-records)
   - 패턴 #09: 데드 레터 (Dead-Letter)
2. [중복된 레코드 (Duplicated Records)](#2-중복된-레코드-duplicated-records)
   - 패턴 #10: 윈도 중복 제거 (Windowed Deduplicator)
3. [지연 데이터 (Late Data)](#3-지연-데이터-late-data)
   - 패턴 #11: 지연 데이터 탐지기 (Late Data Detector)
   - 패턴 #12: 정적 지연 데이터 통합기 (Static Late Data Integrator)
   - 패턴 #13: 동적 지연 데이터 통합기 (Dynamic Late Data Integrator)
4. [요약](#4-요약)

---

## 책의 use case (전체 패턴 공통)

챕터 2와 동일하게 **블로그 분석 플랫폼(blog analytics platform)** 케이스 스터디 위에서 진행됨.
이 챕터는 데이터 수집 이후 단계에서 "원본 데이터가 깨끗하지 않다"는 현실에서 출발함.

데이터는 본질적으로 동적이고, 우리는 늘 **남이 만든 데이터**를 처리함.
따라서 다음과 같은 오류는 피할 수 없음:
- 깨진 페이로드, 스키마 위반 → 처리할 수 없는 레코드
- 프로듀서의 자동 재시도 → 중복 레코드
- 네트워크 단절 후 일괄 업로드 → 지연 데이터

```
오류가 들어오는 지점과 각 패턴의 위치
==========================================================================

[데이터 프로듀서]                       [네트워크 / 인프라]
  - 잘못된 JSON                          - 일시적 단절
  - 누락된 필드                          - 자동 재시도
  - 잘못된 타임존                        - 늦은 전달
    │                                       │
    └───────────────────────────────────────┘
                      ▼
              [Streaming broker / API]
                      │
        ┌─────────────┼──────────────┐
        ▼             ▼              ▼
   [#09 데드 레터]  [#10 윈도 중복   [#11 지연 데이터
    bad record       제거]            탐지기]
    → DLQ            중복 제거         워터마크 계산
        │             │                 │
        │             │                 ├─► on-time → 메인 흐름
        │             │                 └─► late    → #12/#13
        │             │                                 (배치 통합)
        ▼             ▼                                 │
   [재처리 옵션]   [Silver / Gold]                       ▼
                                                  [Silver / Gold 보정]

==========================================================================

전이/비전이 오류 구분 (책의 사이드바)
- Transient (전이): 일시적 DB 단절 등 → 자동 재시도로 복구됨
- Nontransient (비전이): 처리할 수 없는 레코드(poison pill) 등 →
                          자동 복구 불가, 수동 개입 필요
```

---

## 1. 처리할 수 없는 레코드 (Unprocessable Records)

데이터 품질 문제는 데이터 프로젝트에서 가장 흔한 이슈.
"처리할 수 없는 레코드"는 잡으면 잡(job)을 멈추게 만드는 비전이(nontransient) 오류 — **poison pill** 이라고도 부름.
배치라면 fail-fast 로 끝낼 수도 있지만, **장시간 실행되는 스트리밍 잡**에서는 fail-fast 가 거의 불가능함.

### 1-1. 패턴 #09: 데드 레터 (Dead-Letter)

처리할 수 없는 레코드를 메인 흐름에서 분리해 별도 저장소로 보내는 패턴.
"건너뛰지만 잃어버리지는 않는다" 가 핵심.

#### 상황 (Problem)

**책의 use case:**
- 스트리밍 잡이 Apache Kafka 토픽의 방문 이벤트를 객체 저장소로 적재 중.
- 최근 프로듀서가 처리할 수 없는 레코드를 생성하기 시작 → 잡이 계속 실패.
- 3일 연속 수동으로 잡을 재시작하고 체크포인트 파일의 오프셋을 직접 수정하는 중.
- **결정적 제약**: 잘못된 레코드 하나 때문에 파이프라인 전체가 멈춤. 운영자가 매일 새벽 개입해야 하는 상황이 누적되면서 더 이상 지속 불가능.

→ 잘못된 레코드를 만나도 멈추지 않고, 나중에 조사할 수 있게 따로 저장할 방법이 필요.

#### 해결 (Solution)

먼저 코드에서 **실패 가능 지점(likely fail spots)** 을 식별.
- 커스텀 매핑 함수가 될 수도 있고, 프레임워크가 관리하는 error-safe 변환이 될 수도 있음.
- 매핑 함수라면 `try-catch` 블록으로, error-safe 변환이라면 `if-else` 로 안전장치를 둠.
- 실패한 메시지는 메타데이터로 함께 저장 (Metadata Decorator 패턴과 결합).

다음으로 **오류 레코드용 별도 출력(sink)** 을 구성.
DLQ(Dead Letter Queue) 저장소 선택 시 고려 항목:
- **Resiliency** — 데드 레터 저장소 자체에 또 다시 데드 레터 전략을 짤 필요는 없어야 함
- **Monitoring ease** — "최근 얼마나 많은 레코드가 dead-letter 됐는가" 가 핵심 지표
- **Writing performance** — 추가 쓰기 비용은 전체 잡 실행 시간에 영향을 줌

좋은 후보는 **클라우드 객체 저장소** 또는 **스트리밍 브로커** (가용성·속도·모니터링 용이성).

마지막으로 선택적으로 **replay pipeline** 을 둬서 dead-lettered 레코드를 메인 흐름에 다시 통합.

```
데드 레터 패턴 구성요소
--------------------------------------------------------------
[Source]
   │
   ▼
[Job — try/catch 또는 error-safe transformation]
   │
   ├──► 성공 ──► [Main sink (예: object store, Kafka)]
   │
   └──► 실패 ──► [Dead-letter sink]
                    │
                    ▼
              [Monitoring (drop rate, alert)]
                    │
                    ▼ (선택)
              [Replay pipeline ──► Main sink]
--------------------------------------------------------------

스트리밍 vs 배치 차이:
- 스트리밍: 레코드 1건 단위로 dead-letter 에 즉시 기록
- 배치    : 한 번에 dead-letter 대상 레코드 집합(subset)을 기록
```

#### 고려사항 (Consequences)

오류를 무시하는 행위는 단순해 보여도 부작용이 큼.

**Snowball backfilling effect — 백필 눈덩이 효과**
- Fail-fast 라면 컨슈머가 멈추기 때문에 시스템 전체가 정합성을 유지함.
- 데드 레터 + replay 를 쓰면 컨슈머는 일단 부분 데이터로 진행함.
- 나중에 replay 된 레코드가 이미 처리된 파티션에 속하면 → 다운스트림이 백필을 해야 하고, 그들의 다운스트림도 다시 백필 → 눈덩이.
- 완화책: ①replay 자체를 포기 (데이터 손실) ②백필 통보 후 진행 (운영 부담). silver bullet 없음.

**Dead-lettered records identification — 출처 식별**
- replay 된 레코드를 정상 적재 레코드와 구분할 필요가 자주 생김.
- `was_dead_lettered` boolean 컬럼 또는 잡 이름·버전·replay 시각을 담은 메타데이터를 부착 → Data Decorator 패턴과 자연스럽게 결합.

**Ordering and consistency — 순서·정합성**
- IoT 센서가 매분 이벤트를 보내는데 5분 연속으로 데드 레터에 빠지면, 5분 비활성 윈도우로 세션을 닫는 컨슈머는 "세션 종료" 라고 잘못 판단함 → 세션 정합성 파괴.
- 정렬이 보장돼야 하는 경우 더 심각함. `10:00, 10:01, 10:02` 순으로 와야 하는데 `10:01` 만 실패 후 replay 되면 결과는 `10:00, 10:02, 10:01` 이 됨.

**Error-safe functions — 안전 함수의 함정**
- `CONCAT` 같이 NULL 반환으로 처리하는 error-safe 함수는 예외를 던지지 않음 → catch 로 잡을 게 없음.
- 입력은 있는데 출력이 NULL 이라면 "처리 실패" 일 수 있음을 비교 로직으로 따져야 함.
- 함수마다 error-safety 의미가 달라서, 선언형 언어(SQL)에서 패턴을 적용하면 쿼리가 매우 verbose 해짐.

**Error or failure? — 진짜 멈춰야 할 실패의 은폐**
- 데드 레터 패턴은 정의상 오류를 숨기는 패턴 → 정말 심각한 시스템 장애도 같이 숨길 위험.
- 반드시 **드롭 카운트 기반 알람**을 둬서 임계치 초과 시 잡을 강제 정지시킬 것.

#### 구현 예시 (Examples)

**예시 1 — Apache Flink (스트리밍, side output)**

Flink 의 `side output` 으로 별도 destination 을 만들고, mapping 함수에서 try/catch 로 분기:
```python
# OutputTag 로 데드 레터용 사이드 채널 선언
invalid_data_output: OutputTag = OutputTag('invalid_visits', Types.STRING())

# 매핑 함수: 성공이면 메인 스트림, 실패면 사이드 아웃풋으로 분기
def map_rows(self, json_payload: str) -> str:
    try:
        evt = json.loads(json_payload)
        evt_time = int(datetime.datetime.fromisoformat(evt['event_time']).timestamp())
        yield json.dumps({'visit_id': evt['visit_id'],
                          'event_time': evt_time,
                          'page': evt['page']})
    except Exception as e:
        # 실패한 원본 페이로드 + 에러 메시지를 함께 사이드 아웃풋으로
        yield self.invalid_data_output, _wrap_input_with_error(json_payload, e)

# 두 개의 Kafka sink — 정상/비정상 분리
visits.get_side_output(invalid_data_output).sink_to(kafka_sink_invalid_data)
visits.sink_to(kafka_sink_valid_data)
```

**예시 2 — Apache Spark SQL + Delta Lake (배치, error-safe 변환)**

`CONCAT` 처럼 예외를 던지지 않고 NULL 만 돌려주는 함수는 입력/출력 비교로 실패를 추정:
```python
# 1단계 — 변환 결과의 유효성 플래그를 함께 계산
spark_session.sql('''
 SELECT type, full_name, version, name_with_version,
   CASE
     WHEN (full_name IS NOT NULL OR version IS NOT NULL)
       AND name_with_version IS NULL THEN false
     ELSE true
   END AS is_valid
 FROM (SELECT type, full_name, version,
        CONCAT(full_name, version) AS name_w_version
       FROM devices_to_load)''')

# 2단계 — persist() 로 캐싱 후 두 곳에 분리 저장 (쿼리 중복 실행 방지)
devices_to_load_with_validity_flag.persist()

(devices_to_load_with_validity_flag.filter('is_valid IS TRUE')
   .drop('is_valid').write.mode('overwrite')
   .format('delta').save(f'{base_dir}/output/devices-table'))

(devices_to_load_with_validity_flag.filter('is_valid IS FALSE')
   .drop('is_valid').write.mode('overwrite')
   .format('delta').save(f'{base_dir}/output/devices-dead-letter-table'))
```

| 구현 방식 | 동작 | 주의사항 |
|----------|------|----------|
| Flink side output | 레코드 단위로 try/catch 분기 → DLQ topic | 진짜 예외를 던지는 변환에 적합 |
| Spark SQL + error-safe func | NULL 반환을 실패로 해석, persist 후 분리 저장 | 쿼리가 길어짐 / `.persist()` 빠뜨리면 변환이 두 번 실행됨 |
| 배치 후처리 (Iceberg/Delta 등) | 적재 후 별도 검증 쿼리로 invalid 분리 | DLQ가 메인과 동일 저장소 → 모니터링 분리 필요 |

> **트러블 로그** — 데드 레터 패턴을 깔 때 "DLQ에 떨어진 건수" 알람을 빼먹으면 가장 위험.
> 예: 프로듀서가 새 버전 배포 직후 `event_time` 필드 이름을 `eventTime` 으로 바꿔 보내기 시작.
> 잡은 안 죽고 잘 도는데 실제로는 100% 의 레코드가 DLQ 로 들어가는 중. 메인 토픽 처리량은 정상으로 보이고
> downstream 대시보드만 "오늘 방문자 0명" 이 되어 24시간 뒤 발견됨.
> 반드시 **드롭률(예: 직전 5분 평균 대비 3배 이상)** 기준 알람을 설정하고,
> 임계치 초과 시 잡을 자동 정지시켜 잘못된 데이터의 downstream 전파를 막을 것.

---

## 2. 중복된 레코드 (Duplicated Records)

분산 시스템에서 exactly-once 전달은 매우 어렵기 때문에 보통은 at-least-once 환경에서 살게 됨.
프로듀서의 자동 재시도, 컨슈머 재시작 등으로 같은 레코드가 여러 번 도착하는 것은 일상.
**"각 이벤트를 정확히 한 번만 처리"** 를 보장하려면 deduplication 이 필요함.

### 2-1. 패턴 #10: 윈도 중복 제거 (Windowed Deduplicator)

배치와 스트리밍 모두에서 적용 가능한 중복 제거 패턴.
핵심은 **"데이터를 유한한 범위로 가둔다"** 는 발상.

#### 상황 (Problem)

**책의 use case:**
- 배치 잡이 스트리밍 레이어에서 객체 저장소로 동기화된 방문 이벤트를 처리.
- 결과를 비즈니스 사용자에게 직접 노출 → exactly-once 처리 보장 필요.
- 스트리밍 레이어에는 프로듀서 자동 재시도로 인한 중복 이벤트가 빈번하게 섞여 있음.
- **결정적 제약**: 무한히 들어오는 데이터에서 "이미 본 키" 를 영원히 기억할 수는 없음 → 어딘가에서 범위를 끊어야 함.

#### 해결 (Solution)

1. **deduplication attributes 결정** — 레코드의 유일성을 결정하는 컬럼 집합 (예: `visit_id + visit_time`).
2. **deduplication scope 결정**:
   - **배치**: 현재 처리 중인 데이터셋이 곧 scope. 과거 데이터까지 포함시키면 비용·시간 증가.
   - **스트리밍**: 데이터셋이 무한하므로 **시간 기반 윈도우(state store)** 로 가상의 "완료된 데이터셋"을 만듦.

배치 구현은 단순:
- `DISTINCT`, 또는 `ROW_NUMBER() OVER (PARTITION BY ...)` + `position = 1` 필터.
- "배치에서 윈도우" 라는 표현은 명시적 시간 윈도우가 아니라 **암묵적인 global window** (= 처리 데이터셋 전체) 를 의미.

스트리밍 구현은 더 복잡 — **state store** 에 이미 본 키를 윈도우 기간만큼 보관.

```
스트리밍에서 윈도 중복 제거 동작
--------------------------------------------------------------
[Stream input]
    │
    ▼
[Watermark = visit_time - 10 min]   ←─ "10분 이전 데이터는 이미 본 적 있어도 제거"
    │
    ▼
[State Store]                       ← 이미 본 키 (visit_id, visit_time) 보관
    │   ┌─────────────────────────┐
    │   │ key=v1, ts=10:00 (유지) │
    │   │ key=v2, ts=10:05 (유지) │
    │   │ key=v3, ts=09:45 (만료) │
    │   └─────────────────────────┘
    │
    ▼
[중복이면 drop / 신규면 forward + state에 기록]
    │
    ▼
[Sink]
--------------------------------------------------------------

State store 의 세 가지 종류 (모두 성능 vs 정합성 트레이드오프):
- Local                  : 메모리만, 가장 빠름, 실패 시 상태 소실
- Local + fault-tolerance: 메모리 + 원격 저장, 빠르지만 영속화 비용 발생
- Remote                 : 원격 전용, 내장 fault-tolerance 이지만 지연·비용 증가
```

#### 고려사항 (Consequences)

Exactly-once 처리가 곧 exactly-once 전달을 의미하지 않는다는 점이 중요.

**Space vs Time — 공간/시간 트레이드오프 (스트리밍 한정)**
- 짧은 윈도우 → state store 작음, 자원 적게 씀, 그러나 윈도우 밖 중복은 놓침.
- 긴 윈도우 → 더 안전하지만, 보존할 unique key 수가 폭증 → state store 가 GB 단위로 커질 수 있음.

**Idempotent producer — 전달 보장은 별개 문제**
- 컨슈머에서 중복을 깨끗하게 잘 제거해도, **프로듀서의 재시도** 로 인한 exactly-once 전달은 별도 문제.
- 진짜 exactly-once 전달은 Chapter 4 의 idempotency 패턴이 필요함.

**Automatic retries — 재시도와의 충돌**
- 잡 자체가 runtime error 로 재시작되면, 재시작 직전에 처리된 레코드가 다시 들어와 deduplication 로직을 통과할 수도 있음.
- 자동 재시도와 deduplication 의 동시 보장은 어렵고, 보통은 자동 transient error 복구를 우선시함.

#### 구현 예시 (Examples)

**예시 1 — Apache Spark 배치 (`dropDuplicates`)**

지정한 컬럼 기준으로 중복 제거. 미지정 시 전체 스키마 사용:
```python
dataset = (session.read.schema('...').format('json').load(f'{base_dir}/input'))
deduplicated = dataset.dropDuplicates(['type', 'full_name', 'version'])
```

**예시 2 — 일반 SQL (WINDOW 함수)**

native dedup 함수가 없는 환경에서는 `ROW_NUMBER` + `position = 1` 패턴:
```sql
SELECT type, full_name, version FROM (
  SELECT type, full_name, version,
    ROW_NUMBER() OVER (PARTITION BY type, full_name, version ORDER BY 1) AS position
  FROM duplicated_devices
) WHERE position = 1
```

**예시 3 — Apache Spark Structured Streaming (`dropDuplicates` + watermark)**

스트리밍은 반드시 watermark 와 결합 — state store 의 무한 증가를 막기 위함:
```python
event_schema = StructType([
    StructField("visit_id", StringType()),
    StructField("visit_time", TimestampType())
])

# (1) 입력 파싱 — JSON 에서 visit_time/visit_id 추출
deduplicated_visits = (input
    .select(F.from_json("value", event_schema).alias("value_struct"), "value")
    .select("value_struct.visit_time", "value_struct.visit_id", "value")

# (2) 워터마크: visit_time 기준 10분 이전 키는 state 에서 자동 제거
    .withWatermark("visit_time", "10 minutes")
    .dropDuplicates(["visit_id", "visit_time"])
    .drop("visit_time", "visit_id"))
```

| 구현 방식 | 동작 | 주의사항 |
|----------|------|----------|
| 배치 `dropDuplicates` | 데이터셋 전체에서 중복 제거 | scope 가 곧 한 번의 잡 → 과거 배치와의 중복은 별도 처리 |
| 배치 `ROW_NUMBER` | 그룹 내 첫 행만 유지 | `ORDER BY` 절을 빼면 비결정적 결과 |
| 스트리밍 `dropDuplicates + watermark` | state store 에 키를 windowed 보관 | watermark 없이 쓰면 state 가 무한히 자람 |

> **트러블 로그** — 스트리밍에서 `dropDuplicates` 를 watermark 없이 쓰는 게 가장 흔한 실수.
> 예: visit_id 카디널리티가 일 1,000만 건인 트래픽에서 watermark 미설정으로 띄워두면
> state store 가 RocksDB 기준 하루에 수십 GB씩 증가, 일주일 만에 executor OOM 으로 파이프라인이 죽고
> 체크포인트는 살아있는 채로 잡만 멈춰 운영 새벽 호출이 발생함.
> 반드시 `withWatermark(...)` 를 먼저 선언해 state 만료 기준을 명시할 것.
> 윈도우 길이는 "실제 운영에서 관측된 최대 지연 + 안전 마진" 으로 잡고, 처음부터 길게 두지 말 것.

---

## 3. 지연 데이터 (Late Data)

처리 불능 레코드, 중복 다음으로 만나는 가장 골치 아픈 문제가 지연 데이터.
이름은 평범해 보이지만 파이프라인 결과의 **정확성**을 직접 흔든다.

### 책의 시간 개념 정리

- **Event time** : 그 사건(이벤트)이 실제로 발생한 시각. 늦게 도착할 수 있음.
- **Processing time** : 파이프라인이 그 데이터를 다룬 시각. 정의상 절대 "늦지" 않음.
- **Watermark** : `MAX(event time) - allowed lateness`. 이 값보다 오래된 event 는 "지연" 으로 간주.

지연 데이터 처리는 3단계로 나뉨:
1. **탐지** — Late Data Detector (#11)
2. **고정 기간 통합** — Static Late Data Integrator (#12)
3. **가변 기간 통합** — Dynamic Late Data Integrator (#13)

### 3-1. 패턴 #11: 지연 데이터 탐지기 (Late Data Detector)

지연 데이터를 처리하기 위한 첫 단계 — "이 레코드가 늦었는가?" 를 판정하는 패턴.

#### 상황 (Problem)

**책의 use case:**
- 블로그 방문자 대부분은 실시간으로 이벤트 생성 (생성 후 15초 이내 도착).
- 그러나 네트워크가 끊긴 사용자는 로컬에 버퍼링 후 연결 복구 시점에 한꺼번에 flush.
- **결정적 제약**: 잡이 어떤 레코드를 "늦었다" 고 판단할 수 있어야, "무시 / 별도 저장 / 재통합" 등 use case 별 전략을 적용 가능.

#### 해결 (Solution)

1. **시간 속성 정의** — 이벤트가 언제 일어났는지 나타내는 단일 time attribute (event time) 가 있어야 함. 없으면 분류 자체가 불가.
2. **파티션 단위 집계 전략(latency aggregation)** — 잡이 멈추는 일을 피하려면 **단조 증가(monotonically increasing)** 여야 함.
   - 가장 일반적으로 **`MAX(event_time)`** 를 사용 — 시간이 거꾸로 가지 않음.
   - **`MIN` 은 stuck-in-the-past** 문제를 일으키므로 보통 회피.
3. **전체 데이터소스 통합 전략** — 여러 파티션·여러 소스의 progress 를 하나로 합치는 함수 선택:
   - `MIN`: 가장 느린 의존성 추종 → 더 많은 데이터를 on-time 으로 인정. 단, event time 기반 버퍼링이 크면 버퍼가 비대해짐.
   - `MAX`: 가장 빠른 의존성 추종 → 버퍼는 작아지지만 느린 소스의 데이터를 잃을 위험.
   - `MIN + MAX 혼합`: 소스별로 다른 전략 적용 후 전체 합산 가능.
4. **allowed lateness 정의** — 프로듀서 측 일시적 지연을 허용하는 여유분.
   - 최종 watermark = `MAX(event_time) - allowed_lateness`.

```
워터마크 계산 예시 (allowed lateness = 30분)
--------------------------------------------------------------
Step 1: 첫 배치 도착 — 10:00, 10:05, 10:06
   input watermark   : 없음
   watermark candidate: MAX(10:06) - 30분 = 09:36
   output watermark   : 09:36                ← 새로 정의
   ignored records    : 없음

Step 2: 두 번째 배치 — 9:20, 9:31, 10:07
   input watermark   : 09:36
   watermark candidate: MAX(10:07) - 30분 = 09:37
   output watermark   : MAX(09:36, 09:37) = 09:37   ← 더 큰 쪽으로 전진
   ignored records    : 09:20, 09:31                ← 09:36 이전 → late
--------------------------------------------------------------

핵심: output watermark 는 절대 뒤로 가지 않음 (MAX 누적).
      이게 깨지면 stateful 잡이 "이미 닫힌 윈도우" 를 다시 열어야 함.
```

#### 고려사항 (Consequences)

탐지 자체에도 함정이 많음.

**Late data capture — 프레임워크별 지원 차이**
- Apache Spark Structured Streaming: 지연 데이터를 **탐지·무시** 는 하지만 **캡처 API 가 없음** → 별도 저장이 어려움.
- Apache Flink: 탐지·캡처 모두 유연하게 지원 → side output 으로 분리 저장 가능.
- "탐지만 되고 캡처가 안 되는" 환경이라면 Late Data Integrator 패턴 적용도 제한됨.

**MIN strategy / stuck-in-the-past**
- 파티션 추적에 `MIN` 을 쓰면 가장 늦은 이벤트가 워터마크를 끌어내림 → 두 가지 심각한 문제 발생:
  - **Open-close-open 무한 루프**: 이미 emit 한 윈도우를 다시 열어야 함.
  - **Stuck in the past**: 지연 데이터가 계속 들어오면 워터마크가 영원히 전진 못 함 → state 가 무한 증가.
- 그래서 partition-level 에서는 사실상 `MAX` 만 권장.

**MAX strategy / event skew**
- `MAX` 전략은 highly skewed 환경에서 너무 공격적일 수 있음.
- 예: 5개 소스 중 4개가 네트워크 이슈로 40분 지연. 워터마크는 1개의 빠른 소스 기준으로 전진 → 나머지 4개는 전부 late 로 drop.
- 완화책: 적절한 모니터링 + 지연 발생 시 재통합 (다음 두 패턴).

#### 구현 예시 (Examples)

**예시 1 — Apache Spark Structured Streaming (`withWatermark`)**

탐지·무시를 자동으로 해주는 가장 단순한 형태:
```python
visits_events = (input_data.selectExpr('CAST(value AS STRING)')
    .select(F.from_json('value',
            'visit_id INT, event_time TIMESTAMP, page STRING').alias('visit'))
    .selectExpr('visit.*'))

# event_time 기준 1시간까지의 지연 허용 → 1시간보다 늦은 이벤트는 자동 drop
session_window: DataFrame = (visits_events
    .withWatermark('event_time', '1 hour')
    .groupBy(F.window(F.col('event_time'), '10 minutes'))
    .count())
```

이 코드 동작을 표로 보면 (2023-06-30 기준):

| event_time | 워터마크 (전→후) | 버퍼링된 윈도우 | emit 된 윈도우 |
|-----------|------------------|-----------------|----------------|
| 03:15 | 1970-01-01T00:00 → 02:15 | [03:10-03:20] | [] |
| 03:00 | 02:15 → 02:15 | [03:00-03:10, 03:10-03:20] | [] |
| 01:50 | 02:15 → 02:15 (drop) | [03:00-03:10, 03:10-03:20] | [] |
| 03:11 | 02:15 → 02:15 | [03:00-03:10, 03:10-03:20] | [] |
| 04:31 | 02:15 → 03:31 | [04:30-04:40] | [03:00-03:10, 03:10-03:20] |

**예시 2 — Apache Flink (탐지 + 캡처 모두)**

Flink 는 현재 워터마크 값을 ProcessFunction context 에서 직접 읽을 수 있음 → 늦은 이벤트를 side output 으로 캡처 가능:
```python
# (1) 이벤트에서 event_time 추출하는 timestamp assigner
class VisitTimestampAssigner(TimestampAssigner):
    def extract_timestamp(self, value, record_timestamp):
        event = json.loads(value)
        event_time = datetime.datetime.fromisoformat(event['event_time'])
        return int(event_time.timestamp())

# (2) 워터마크 인지 processor — 늦은 이벤트는 별도 OutputTag 로 분기
class VisitLateDataProcessor(ProcessFunction):
    def __init__(self, late_data_output: OutputTag):
        self.late_data_output = late_data_output

    def process_element(self, value: Visit, ctx):
        current_watermark = ctx.timer_service().current_watermark()
        if current_watermark > value.event_time:
            yield (self.late_data_output,
                   json.dumps(VisitWithStatus(visit=value, is_late=True).to_dict()))
        else:
            yield json.dumps(VisitWithStatus(visit=value, is_late=False).to_dict())

# (3) 파이프라인 — out-of-orderness 5초까지 허용
watermark_strategy = (WatermarkStrategy
    .for_bounded_out_of_orderness(Duration.of_seconds(5))
    .with_timestamp_assigner(VisitTimestampAssigner()))

late_data_output: OutputTag = OutputTag('late_events', Types.STRING())
visits: DataStream = (data_source.map(map_json_to_visit)
    .process(VisitLateDataProcessor(late_data_output), Types.STRING()))

# 두 sink 로 분리 — 정상 / 지연
visits.get_side_output(late_data_output).sink_to(kafka_sink_late_visits)
visits.sink_to(kafka_sink_valid_data)
```

| 항목 | Spark Structured Streaming | Apache Flink |
|------|---------------------------|--------------|
| 탐지 | O (`withWatermark`) | O (워터마크 컨텍스트) |
| 캡처(별도 저장) | X (공식 API 없음) | O (side output) |
| 추천 use case | "그냥 무시해도 OK" 한 잡 | 지연 데이터 보존·재처리 필요 시 |

> **트러블 로그** — 시계열 분석 잡에서 partition-level 워터마크에 `MIN` 을 잘못 적용하면 stuck-in-the-past 가 발생.
> 예: 5개 시·도 지역별 partition 으로 들어오는 IoT 센서 데이터에서 1개 지역이 네트워크 문제로 30분 단위로
> 늦게 들어오는데 `MIN(event_time)` 으로 워터마크를 잡으면 전체 잡이 그 지역 시간에 끌려가 영원히 30분 뒤에서 전진,
> 윈도우는 절대 닫히지 않고 state store 가 시간당 수 GB씩 자람.
> 운영 중에 갑자기 윈도우가 emit 되지 않고 state 만 부풀 때 가장 먼저 의심할 것.
> 파티션 단위는 반드시 `MAX`, 전체 통합 단계에서만 trade-off 에 따라 `MIN`/`MAX` 를 고를 것.

---

### 3-2. 패턴 #12: 정적 지연 데이터 통합기 (Static Late Data Integrator)

지연 데이터가 "고정된 기간 내" 에서만 의미 있을 때 쓰는 단순한 통합 패턴.

#### 상황 (Problem)

**책의 use case:**
- 일일 배치 잡이 블로그를 참조하는 외부 사이트의 통계를 생성.
- **15일까지의 데이터는 approximate 로 간주** — 15일 안에 들어온 지연 데이터까지 통합 허용.
- 15일을 넘긴 데이터는 무시.
- 현재 잡은 "오늘자" 데이터만 처리하므로 지연 데이터가 누락됨.
- **결정적 제약**: 매일 15개의 별도 잡을 돌리는 대신, **단일 파이프라인에서** 지난 14일 + 오늘을 함께 처리해야 함.

> **사이드바 — Processing time partition 의 함정**
> 가장 쉬운 우회는 processing time 기반 파티션이지만, downstream 이 event time 파티션을 쓴다면
> "내 파이프라인에는 지연이 없어도, downstream 에 지연을 떠넘기는" 결과가 됨. 문제를 다른 곳으로 이동시킬 뿐.

#### 해결 (Solution)

1. **Static lookback window** 정의 — "얼마나 과거까지 다시 처리할지" 의 고정 기간 (예: 14일).
   - 실행일이 2024-12-31, 윈도우 14일이면 → 2024-12-17 ~ 2024-12-30 + 오늘.
   - 2024-12-15 에 지연 데이터가 들어와도 그건 무시됨.
2. **메인 파이프라인 내 통합 단계 배치**.
   - 책의 Figure 3-6 에 따라 3가지 전략:
     - **Sequential (지연 데이터 먼저 → 현재 데이터)**: stateful 파이프라인용. 이전 결과에 의존하므로 과거 상태가 정확해야 함.
     - **Parallel (지연 + 현재 동시 처리)**: stateless 한정.
     - **Sequential 반대 (현재 먼저 → 지연 나중)**: stateless 한정, 최신 데이터를 빠르게 노출하고 싶을 때.

```
3가지 통합 전략 비교
--------------------------------------------------------------
(1) Sequential — late 먼저 (stateful 권장)
    [late T-14..T-1] ──► [current T]
       이전 누적 결과가 다음 결과에 영향 → 순서 보장 필요

(2) Parallel
    [late T-14..T-1] ┐
                     ├──► [merge / write]
    [current T]      ┘
       stateless 한정. 빠르지만 결과 의존성 없어야 안전

(3) Sequential — current 먼저
    [current T] ──► [late T-14..T-1]
       사용자에게 "오늘 결과" 부터 노출하고 싶을 때 유용
--------------------------------------------------------------
```

#### 고려사항 (Consequences)

데이터 정확성을 일부 회복하지만, 복잡도 비용을 동반.

**Snowball backfilling effect — 컨슈머도 다 같이 백필**
- 당신이 14일치를 백필하면 정합성 신경 쓰는 다운스트림도 14일치 백필해야 함 → 그들의 컨슈머도 마찬가지 → 시스템 전체 compute 폭증.
- 막을 방법은 없음. **백필 액션을 다운스트림에 통지** 하는 것이 최선.

**Overlapping executions and backfilling — 중복 백필**
- lookback window 가 4일이고 잡이 10/10, 10/11, 10/12 에 실행됐다고 가정.
- 세 실행을 모두 replay 하면:
  - 10/10 → 10/06~10/09 처리
  - 10/11 → 10/07~10/10 처리
  - 10/12 → 10/08~10/11 처리
- 결과적으로 같은 날짜가 여러 번 처리됨 → **백필 시 마지막 실행(10/12) 만 재실행** 하면 충분.

**Pipeline trigger — 백필은 메인 파이프라인 내부에서**
- 별도 백필 파이프라인을 따로 띄우면 위와 같은 overlap 문제 재발 → 통합 단계는 반드시 메인 파이프라인 안에 둘 것.

**Waste of resources — 빈 lookback 도 매번 실행됨**
- "그 기간에 새 지연 데이터가 없어도" 매일 동일 기간을 다시 훑음 → 자원 낭비.
- 완화: 사전 검사 task 를 두거나 → 패턴 #13 (Dynamic) 으로 전환.

**Time requirement — 시간 파티션 필수**
- 데이터셋에 시간 개념이 없으면 이 패턴 자체가 적용 불가.

#### 구현 예시 (Examples)

**예시 — Apache Airflow `Dynamic Task Mapping`**

`expand()` 로 lookback 기간만큼 동적으로 task 생성:
```python
# 1) 백필 대상 날짜 목록 생성 — lookback = 2일
@task
def generate_backfilling_runs():
    dr: DagRun = get_current_context()['dag_run']
    backfilling_dates = []
    days_to_backfill = 2
    start_date_to_backfill = (dr.execution_date
                              - datetime.timedelta(days=days_to_backfill))
    for days_to_add in range(0, days_to_backfill):
        date_to_backfill = start_date_to_backfill + datetime.timedelta(days=days_to_add)
        backfilling_dates.append(date_to_backfill.date().strftime('%Y-%m-%d'))
    return backfilling_dates

# 2) 각 날짜마다 별도 task 인스턴스를 동적으로 매핑
@task
def integrate_late_data(late_date: str):
    copy_file(late_date)

# 3) 워크플로우: 현재 일자 적재 → 백필 날짜 목록 생성 → 각 날짜별 통합
backfilling_runs_generator = generate_backfilling_runs()
(file_to_load_sensor >> load_current_file() >> backfilling_runs_generator >>
 integrate_late_data.expand(late_date=backfilling_runs_generator))
```

> **트러블 로그** — Static Late Data Integrator 를 처음 도입할 때 백필을 잘못 돌리면 같은 파티션이 N배 처리됨.
> 예: lookback 7일짜리 잡을 운영 중인데 코드 버그 수정 후 "지난 7일 전부 백필" 한답시고
> 7/14, 7/15, 7/16... 7개 실행을 모두 재실행하면 각 실행이 7일치를 처리하므로
> 같은 날짜가 평균 7번 덮어써짐 → BigQuery 비용 7배, idempotent 하지 않은 sink 라면 중복 INSERT.
> **백필 시에는 가장 마지막 실행 하나만 재실행** 하면 lookback 덕분에 전 기간이 자연스럽게 커버됨.

---

### 3-3. 패턴 #13: 동적 지연 데이터 통합기 (Dynamic Late Data Integrator)

고정 lookback 의 한계 (자원 낭비, 윈도우 밖 데이터 손실) 를 극복하기 위한 동적 변형.

#### 상황 (Problem)

**책의 use case:**
- 기존 Static 패턴으로 15일 lookback 을 운영 중.
- 프로덕트 오너가 "15일을 넘긴 모든 지연 데이터까지 반영하라" 고 요구.
- 매일 15일 전체를 무지성 백필하는 방식은 비효율적이고 윈도우를 늘려도 한계가 있음.
- **결정적 제약**: "지연이 실제로 들어온 파티션만" 정확히 골라서 다시 처리해야 함.

#### 해결 (Solution)

핵심 아이디어 — **state table** 을 도입해 파티션별 마지막 업데이트 시각을 추적:

| Partition | Last processed time | Last update time |
|-----------|--------------------|--------------------|
| 2024-12-17 | 2024-12-17T10:20 | 2024-12-17T03:00 |
| 2024-12-18 | 2024-12-18T09:55 | 2024-12-20T10:12 ← 늦게 업데이트됨 |

→ `Last update time > Last processed time` 인 파티션만 백필 대상.

```sql
-- 백필 대상 조회 쿼리
SELECT partition FROM state_table
WHERE `Last update time` > `Last processed time`
  AND `Partition` < `Processed partition`
```

`Last update time` 을 어떻게 얻는가는 저장소에 따라 다름:
- **BigQuery**: `INFORMATION_SCHEMA.PARTITIONS.last_modified_timestamp` 제공.
- **Apache Iceberg**: 파티션 메타데이터 테이블에 `last_updated_at` 컬럼.
- **Delta Lake**: 네이티브 컬럼은 없고 `DeltaLog.getChanges` 로 직접 추출 (Scala API).
- 위에 해당하지 않는 저장소 → 별도 업데이트 추적 테이블을 직접 유지해야 함.

> **사이드바 — Not Whole Partitions**
> 영향받은 entity 만 식별 가능하면 전체 파티션을 덮어쓰지 않고 해당 entity 만 갱신 가능 → 자원 최적화.
> 단, 구현 복잡도 증가.

```
Static vs Dynamic 비교
--------------------------------------------------------------
[Static]
  매일 무조건 N일치 백필
  └─► 지연 없는 파티션도 재실행 → 자원 낭비
       (단순함이 장점)

[Dynamic]
  state table 조회 → 진짜 지연된 파티션만 골라 백필
  └─► 자원 효율 ↑, but 다음 함정 발생:
       - 동시 실행 시 같은 파티션 중복 백필 가능
       - state table 운영 부담
       - 매우 오래된 지연 데이터가 들어오면 stateful 잡 백필 폭발
--------------------------------------------------------------
```

#### 고려사항 (Consequences)

자원 낭비를 줄이는 대신 새로운 복잡도가 따라옴.

**Concurrency — 동시 실행 시 중복 백필**
- 파이프라인이 병렬 실행을 허용하면 같은 파티션이 두 번 잡힐 수 있음.
- 해결: state table 에 `Is processed` 컬럼 추가 + 쿼리에 조건 추가:

```sql
SELECT partition FROM state_table
WHERE `Last update time` > `Last processed time`
  AND `Partition` < `Processed partition`
  AND `Is processed` = false
```

- 추가로 파이프라인 측 보강:
  - 각 잡 시작 시 현재 파티션의 `Is processed = true` 설정 (이전 run 성공 시에만).
  - 백필 대상 조회 task 도 retrieved 파티션을 `Is processed = true` 로 업데이트.
  - 정상 처리 완료 시 `Is processed = false` 로 되돌려, 추후 새 지연 데이터 도착 시 재시도 가능.
- 부작용: 백필 생성 task 가 실패하면 future run 이 dependency 로 멈춰 long in-progress 상태 → 수동 개입 필요.

**Stateful pipelines and very late data — 매우 오래된 지연**
- Incremental Sessionizer 같은 stateful 파이프라인이 마지막 성공 run = 2024-10-20 인데, 2024-09-21 의 지연 데이터가 도착한다면?
- → 9/21 ~ 10/20 전체를 재실행해야 정합성 유지됨.
- 즉, 동적 lookback 이라고 해서 무제한일 수는 없음. **dynamic 에도 상한 lookback 을 정의** 할 필요가 있음.

**Scheduling complexity — 운영 복잡도**
- 동적 백필 파이프라인의 동적 생성 자체가 어려움.
- 저장소가 last-modification time 을 안 주면 추적 테이블을 직접 유지 (개발·운영 비용).

#### 구현 예시 (Examples)

**예시 1 — Delta Lake (Scala): 마지막 변경 버전 추출**

Delta Lake 는 native `last_updated_at` 이 없으므로 `DeltaLog.getChanges` 로 추적:
```scala
val deltaLog = DeltaLog.forTable(sparkSession, jobArguments.tableFullPath)
val partitionsChangeVersions: Iterator[(String, Long)] = deltaLog.getChanges(0)
  .flatMap {
    case (version, actions) => {
      val changedPartitionsInVersion: Set[String] = actions.map {
        case addFile: AddFile if addFile.dataChange =>
          Some(addFile.partitionValues.map(e => s"${e._1}=${e._2}").mkString("/"))
        case removeFile: RemoveFile if removeFile.dataChange =>
          Some(removeFile.partitionValues.map(e => s"${e._1}=${e._2}").mkString("/"))
        case _ => None
      }.filter(_.isDefined).map(_.get).toSet

      changedPartitionsInVersion.map(partition => (partition, version))
    }
  }

// 파티션별 최신 버전
val lastVersionForEachPartition: Map[String, Long] = partitionsChangeVersions
  .toSeq.groupBy(pair => pair._1).mapValues(pairs => pairs.map(_._2).max)
```

**예시 2 — 백필 대상 산출 (Delta Lake 기준 버전 비교)**

```scala
val partitionsToBackfill = sparkSession.read.format("delta").load(TablePath)
  .select("partition", "isProcessed", "lastProcessedVersion").as[PartitionState]
  .filter(state => state.lastProcessedVersion.isDefined)
  .filter(state => {
    // (1) 최신 쓰기 버전이 마지막 처리 버전보다 크고
    (!lastVersionPerPartition.contains(state.partition) ||
     lastVersionPerPartition(state.partition) > state.lastProcessedVersion.get) &&
    // (2) 아직 처리 중이 아니며
    !state.isProcessed &&
    // (3) 현재 파티션보다 과거인 파티션만
    state.partition < currentPartition
  })

partitionsToBackfill.map(_.partition).collect()
```

**예시 3 — Airflow 에서 동시성 제어 + 동적 task 매핑**

`depends_on_past=True` 로 핵심 task 만 순차 실행을 강제:
```python
@task
def generate_backfilling_arguments():
    context = get_current_context()
    current_partition_date = context['ds']
    dag_run: DagRun = context['dag_run']
    dag_run_start_time: str = dag_run.start_date.isoformat()

    # 동시 실행 race condition 회피 — 실행 시각을 포함한 run_id
    def _run_id_for_event_time(event_time: str) -> str:
        return f'backfill_{event_time}_from_{current_partition_date}_{dag_run_start_time}'

    configuration = read_backfilling_configuration(current_partition_date)
    return list(map(lambda partition: {
        'execution_date': partition['event_time'],
        'trigger_run_id': _run_id_for_event_time(partition['event_time'])
    }, configuration['partitions']))


with DAG('devices_loader', max_active_runs=5,
         default_args={'depends_on_past': False, ...}) as dag:

    # 동시성 5 허용하되, 두 핵심 task 는 직전 실행 성공 시에만 동작
    processing_marker = SparkKubernetesOperator(
        task_id='mark_partition_as_being_processed',
        depends_on_past=True
    )
    backfill_creation_job = SparkKubernetesOperator(
        task_id='get_late_partitions_and_mark_them_as_being_processed',
        depends_on_past=True
    )
```

| 항목 | Static (#12) | Dynamic (#13) |
|------|--------------|---------------|
| Lookback 기간 | 고정 (예: 14일) | 가변 (state table 기반) |
| 자원 사용 | 매번 전체 lookback 실행 | 실제 지연된 파티션만 |
| 운영 복잡도 | 낮음 | 높음 (state table + 동시성 제어) |
| 매우 오래된 지연 | 윈도우 밖이면 손실 | 처리 가능, 단 stateful 잡은 백필 폭발 위험 |
| 적합 상황 | 지연 패턴이 안정적 + 단순 SLA | 지연이 산발적이거나 윈도우 길어야 할 때 |

> **트러블 로그** — Dynamic 통합기에서 `depends_on_past` 를 잘못 걸면 운영 사고가 됨.
> 예: state table 갱신 task 에 `depends_on_past=True` 만 걸고 알람을 안 두면, 새벽 1회 실패 시
> 이후 모든 일자의 run 이 "이전 run 미완료" 로 long in-progress 에 빠져 표면적으로는 "DAG 가 도는 듯한" 상태가 되지만
> 실제로는 한 줄도 처리되지 않음 → 며칠 후 다운스트림 대시보드 0건으로 발견.
> `depends_on_past=True` 를 거는 모든 task 에는 **반드시 stuck-state 알람** (예: 12시간 이상 in-progress) 을 함께 설정할 것.

---

## 4. 요약

오류는 피할 수 없음 — 버그, 낮은 품질의 입력 데이터, 일시적 하드웨어 이슈 등 원인은 다양함.
이 챕터의 5개 패턴은 **"데이터 자체의 결함"** 을 운영적으로 다루는 도구상자.

### 4-1. 5개 패턴 한눈 비교

| # | 패턴 | 카테고리 | 책의 use case 시나리오 | 핵심 트레이드오프 |
|---|------|---------|----------------------|----------------|
| 09 | Dead-Letter | 처리 불능 | Kafka → object store, poison pill로 잡 중단 반복 | 운영 안정성 ↔ 오류 은폐 + 백필 눈덩이 |
| 10 | Windowed Deduplicator | 중복 | 스트리밍 → 배치 동기화 시 producer 재시도 중복 | 정확성 ↔ state store 크기 + 자원 |
| 11 | Late Data Detector | 지연 탐지 | 네트워크 단절 사용자가 한꺼번에 flush 한 방문 이벤트 | 단조성(MAX) ↔ skew 시 데이터 손실 |
| 12 | Static Late Data Integrator | 지연 통합 | 외부 사이트 일일 통계 (15일 approximate) | 단순함 ↔ 자원 낭비 + 윈도우 밖 손실 |
| 13 | Dynamic Late Data Integrator | 지연 통합 | "15일 넘은 지연도 반영" 비즈니스 요구 변경 | 자원 효율 ↔ state table + 동시성 복잡도 |

### 4-2. 패턴 선택 의사결정 가이드

```
"파이프라인에서 오류/이상 데이터가 관측됐다"
              │
              ▼
[Q1] 어떤 종류의 문제인가?
  ├─ 페이로드 파싱 실패 / 스키마 위반 ──► #09 Dead-Letter
  ├─ 같은 이벤트가 여러 번 도착             ──► #10 Windowed Deduplicator
  └─ event_time 이 현재 워터마크보다 과거   ──► [Q2] 로
              │
              ▼
[Q2] 지연 데이터를 어떻게 다룰까?
  ├─ 그냥 무시해도 비즈니스적으로 OK         ──► #11 Late Data Detector 만 (drop)
  ├─ 보존만 하고 별도 분석                  ──► #11 + DLQ-like side output (Flink)
  └─ 실제로 다시 통합해야 함                 ──► [Q3] 로
              │
              ▼
[Q3] 통합 방식은?
  ├─ 허용 기간이 고정(예: "최대 14일")        ──► #12 Static Late Data Integrator
  └─ 허용 기간이 가변 + 자원 낭비 줄여야 함    ──► #13 Dynamic Late Data Integrator
              │
              ▼
[Q4] 스트리밍 잡이라면?
  └─ withWatermark + dropDuplicates 결합 필수
     (#10 + #11 은 실질적으로 한 파이프라인에서 같이 동작)
```

### 4-3. 실무 조합 예시 (책 use case 기반)

**시나리오 1: 블로그 방문 이벤트 스트리밍 처리 (Bronze → Silver)**
- 챕터 2의 #03 CDC (Kafka 적재) 와 결합
- #09 Dead-Letter (JSON 파싱 실패용 DLQ topic) + #10 Windowed Deduplicator (`visit_id, visit_time` 기준) + #11 Late Data Detector (`withWatermark('event_time', '1 hour')`)
- 셋이 한 스트리밍 잡 안에서 동시 적용

**시나리오 2: 외부 참조 사이트 일일 통계 (Gold)**
- 데이터 지연이 최대 15일까지 발생, 그 이후는 비즈니스적으로 무의미
- #12 Static Late Data Integrator + Airflow Dynamic Task Mapping
- 백필 시에는 lookback 을 고려해 "마지막 실행 하나" 만 재실행

**시나리오 3: 전자상거래 주문 지연 보정**
- 지연이 산발적이지만 어쩌다 30일 전 주문이 들어옴 → 누락 시 매출 직접 손실
- #13 Dynamic Late Data Integrator + Delta Lake `DeltaLog.getChanges` + Airflow `depends_on_past`
- state table 의 `Is processed` 컬럼으로 동시 실행 방어

**시나리오 4: 신규 잡 배포 직후 모니터링**
- 모든 변환 함수를 `try-catch` 래핑 + #09 Dead-Letter sink 구성
- 드롭률 알람을 baseline 대비 비율로 설정 (절대 건수 X)
- 임계치 초과 시 잡 자동 정지 → downstream 오염 차단

### 4-4. 패턴 전반을 관통하는 운영 원칙

이 챕터를 관통하는 4가지 운영 교훈:

- **오류를 숨기는 만큼 알람으로 보상하라** — Dead-Letter, Late Data Detector 모두 fail-fast 를 포기하는 대가로 "조용히 잘못된 데이터가 흐르는" 위험을 떠안음. 드롭률·지연율 알람은 패턴의 일부로 봐야 하며, 알람 없는 데드 레터는 사고를 며칠 뒤로 미룰 뿐.
- **Exactly-once 처리 ≠ Exactly-once 전달** — Windowed Deduplicator 와 워터마크는 처리 측의 보장만 줄 뿐, 진짜 한 번의 전달은 Chapter 4 의 idempotency 패턴이 별도로 필요. 두 개념을 혼동하면 "중복 제거를 했는데 왜 중복이 나오지" 라는 디버깅에 시간을 낭비함.
- **워터마크의 단조성(monotonicity)이 stateful 잡의 전제** — 워터마크가 뒤로 가면 윈도우 정합성과 state store 가 동시에 망가짐. partition-level 은 `MAX` 가 기본, `MIN` 은 stuck-in-the-past 가 발생할 수 있어 신중히 적용.
- **백필은 lookback 윈도우를 의식해서 돌려라** — Static 패턴에서 잘못 백필하면 같은 데이터가 lookback 배수만큼 중복 처리되어 비용이 폭증함. Dynamic 패턴에서는 `depends_on_past` 와 함께 stuck-state 알람을 반드시 짝지어야 long in-progress 사고를 막을 수 있음.

### 4-5. 다음 챕터와의 연결

Error management 패턴은 exactly-once 의 "느낌" 만 줄 뿐, 진짜 한 번의 전달을 보장하지는 않음.
이를 완성하는 것이 다음 챕터의 **Idempotency 패턴** — 재시도와 백필이 시스템 정합성을 깨지 않도록 만드는 도구들.
