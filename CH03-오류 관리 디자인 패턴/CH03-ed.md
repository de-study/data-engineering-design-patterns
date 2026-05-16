# 3.1 처리할수 없는 레코드
## (1)패턴#09: 데드레터
## 1. 처리 불가 레코드 (Unprocessable Records)

> 데이터 품질 문제는 데이터 프로젝트에서 반복적으로 발생. 처리 불가 레코드는 종종 데이터 처리 job 전체를 중단시키는 치명적 실패를 유발. 그러나 특히 long-running 스트리밍 job에서는 fail-fast 접근을 항상 유지하는 것이 불가능.


## 1-1. 패턴 #01: Dead-Letter

처리 실패한 레코드를 버리지 않고 별도 저장소에 보관해, 나머지 레코드 처리를 계속하는 패턴.

### 배경 — 왜 나왔나

기존 파이프라인은 두 가지 방식 중 하나였음:

- 레코드 하나 실패 → job 전체 중단 (fail-fast)
- 실패 레코드 그냥 무시 → 데이터 영구 손실

둘 다 문제. "실패한 레코드만 따로 빼고, 나머지는 계속 처리하자"는 요구에서 나온 패턴.

---

### 에러 종류 2가지

**Transient Error (일시적 오류)**

- 일시적이며 자동 복구됨
- 예: DB 일시 장애 → 자동 connection retry로 해결

**Nontransient Error (영구적 오류)**

- 영구적이며 스스로 절대 복구 안 됨
- 일명 "poison pill 메시지"
- 예: 처리 불가 레코드 — job을 멈추고 수동 개입 필요
- Dead-Letter 패턴이 다루는 대상

---

### 상황 (Problem)

문제상황:

- 스트리밍 job이 Apache Kafka 토픽에서 visit 이벤트를 object store에 씀
- 데이터 producer들이 갑자기 처리 불가 레코드를 생성하기 시작
- job이 계속 실패함
- 3일 연속으로 수동 대응: job 재시작 + checkpoint 파일의 processed offset 직접 수정
- 이 수동 작업 없이 파이프라인이 실패 레코드가 있어도 계속 실행되고, 나중에 에러를 조사할 수 있는 해결책이 필요

핵심 문제: Kafka offset 블로킹

- Kafka는 offset 순서대로 레코드를 처리
- bad record 하나가 예외를 던지면 job 중단
- 재시작해도 checkpoint에서 동일한 bad record를 다시 만남
- → 무한 반복. 엔지니어가 매번 offset을 수동으로 건너뜀

---

### 해결 (Solution)

패턴 구현 4단계:

```
[1단계] 실패 가능 지점 식별
   커스텀 매핑 함수, 타입 변환 등 예외가 발생할 수 있는 코드 구간 파악

[2단계] 안전 장치 추가
   try-catch (코드 레벨) 또는 error-safe transformation (프레임워크 레벨)

[3단계] 실패 레코드 → Dead-Letter 저장소로 라우팅
   실패 레코드 + 실패 메타데이터(에러 메시지, 타임스탬프 등) 함께 저장

[4단계] Replay 파이프라인 (선택)
   저장된 실패 레코드를 수정 후 메인 흐름에 재투입
```

**안전 장치 2가지**

try-catch (코드 레벨):

- 개발자가 직접 실패 가능 구간을 감싸는 방식
- 어떤 예외든 처리 가능 — 완전한 제어권
- 단점: 직접 짜야 하므로 누락 가능성 있음
- 적합한 상황: 비즈니스 로직이 복잡하거나 프레임워크가 지원 안 하는 커스텀 변환

error-safe transformation (프레임워크 레벨):

- 프레임워크가 실패를 자동으로 처리해주는 내장 기능
- Spark 예시: `from_json()`, `CONCAT()` — 파싱 실패 시 예외 대신 NULL 반환
- if-else 조건으로 NULL인 것(실패한 것)을 Dead-Letter로 분기
- 적합한 상황: 타입 변환, JSON 파싱, 스키마 검증처럼 프레임워크가 이미 안전하게 처리하는 경우

실무 선호: 둘을 혼용. 프레임워크 지원 변환은 error-safe transformation 먼저, 그 위 커스텀 로직은 try-catch.

**Dead-Letter 저장소 선택 기준**

- Resiliency — 저장소 자체가 죽으면 안 됨. Dead-Letter의 Dead-Letter는 없어야 함
- Monitoring 용이성 — "최근 얼마나 많이 실패했나"를 빠르게 파악 가능해야 함. 이게 핵심 성공 지표
- 쓰기 성능 — 메인 처리에 지연을 주면 안 됨

실무 선호 저장소:

- S3 (object store): 배치 파이프라인, 비용 저렴, 모니터링은 CloudWatch/Athena 조합
- Kafka Dead-Letter 토픽: 스트리밍 파이프라인, 빠름, 실시간 모니터링 용이, Replay가 자연스러움

**전체 아키텍처**

```
[Input] → [Processing + try-catch / error-safe]
                │
        ┌───────┴───────┐
   성공 레코드        실패 레코드
        │                  │
   [Main Output]    [Dead-Letter Store]
                           │
                    [Monitoring Layer]
                           │
                    [Replay Pipeline] → [Main Output]
```

---

### 배치 vs 스트리밍 비교

**배치에서 실패 발생 방식**

- 전체 데이터를 한번에 읽고 처리하다가 특정 레코드에서 예외 발생
- job 전체가 실패로 표시되고 중단
- 재실행하면 처음부터 다시 (이미 처리한 것도 포함)
- Dead-Letter 처리: 실패 레코드 subset을 한번에 별도 파일로 저장 → 나머지 정상 처리

**스트리밍에서 실패 발생 방식**

- Kafka offset 순서대로 처리하다 bad record에서 예외
- job 죽으면 checkpoint의 마지막 offset부터 재시작
- 문제: 재시작해도 같은 bad record를 다시 만남 → 무한 반복
- Dead-Letter 처리: bad record를 Dead-Letter 토픽/S3에 저장 → offset 블로킹 없이 계속 진행

|구분|실패 영향|재실행 비용|Dead-Letter 저장소|
|---|---|---|---|
|배치|job 전체 중단|처음부터 재실행|S3, 파일|
|스트리밍|offset 블로킹, 무한 재실패|offset 수동 조작 필요|Kafka Dead-Letter 토픽, S3|

스트리밍이 배치보다 훨씬 치명적 — 막힌 offset이 뒤따라오는 모든 레코드를 블로킹하기 때문

---

### Replay 파이프라인 — 재처리로 살릴 수 있는 경우

실패 원인이 "데이터 문제"냐 "환경 문제"냐로 판단:

**케이스 1 — 스키마 버전 불일치 (데이터 문제)**

- 이커머스 주문 이벤트: `{"amount": 9900}` → `{"total_amount": 9900}` 으로 필드명 변경
- 파이프라인이 `amount` 필드를 못 찾아 실패 → Dead-Letter 저장
- Replay: `total_amount` → `amount` 로 변환 후 재투입 → 정상 처리

**케이스 2 — 타임스탬프 포맷 불일치 (데이터 문제)**

- 배달 앱 위치 이벤트: ISO 8601 → `"15/01/2024 10:30:00"` 으로 포맷 변경
- Spark `to_timestamp()` 파싱 실패 → Dead-Letter 저장
- Replay: 포맷 감지 후 ISO 8601 변환 → 재투입 → 정상 처리

**케이스 3 — 외부 API 일시 장애 (환경 문제)**

- 결제 이벤트 처리 중 환율 API 5분 장애 → 그 시간대 이벤트 전부 실패
- Replay: API 복구 후 동일 로직 재실행 → 정상 처리

**살릴 수 없는 경우**

- 필수 필드 자체가 누락 (`order_id` 없음)
- 완전히 깨진 바이너리
- → Dead-Letter에 보존만 하고 알람 발송 후 수동 조사

---

### 고려사항 (Consequences)

**1. Snowball backfilling effect — 눈덩이 역방향 채우기 효과**

Replay 파이프라인 실행 시 재투입 레코드가 이미 다운스트림 컨슈머가 처리한 파티션에 속할 수 있음. → 다운스트림이 해당 파티션을 다시 처리해야 함 → 그 다운스트림의 다운스트림도 재처리 필요 → 연쇄 backfilling 발생

```
Pipeline 1 (Replay) → Pipeline 2 재처리 → Pipeline 3 재처리 → ...
```

미티게이션 옵션:

- Replay 안 함 → dead-lettered 데이터 손실 (단순하지만 데이터 불완전)
- Replay 함 → 데이터 완전하지만 다운스트림이 backfilling 못 하면 불일관성 발생
- 완벽한 해결책 없음. trade-off 선택 필요
```
(1)의미
미티게이션(Mitigation)은 "완전히 해결은 못 하지만, 피해를 줄이는 대응 방법
여기서 근본 문제는 Replay를 하든 안 하든 어떤 식으로든 손해가 생긴다는 것.
- Replay 안 하면 → 데이터 손실
- Replay 하면 → 다운스트림 불일관성
둘 다 완벽하지 않음. 그래서 "어느 쪽 손해를 감수할 것인가"를 선택하는 게 미티게이션.
---

(2)실무 예시
이커머스 주문 이벤트 5분치가 Dead-Letter에 쌓인 상황:
- Replay 안 함 → 그 5분치 주문 데이터 영구 손실. 일일 매출 집계가 틀림. 단순하고 운영 부담 없음
- Replay 함 → 주문 데이터는 살아남. 그런데 이미 정산팀이 그 시간대 데이터를 "없음"으로 처리해버렸다면 → 정산 테이블과 주문 테이블이 불일치. 정산팀도 backfilling 해줘야 하는데 못 하는 상황이면 더 큰 문제
---

(3)결론
미티게이션은 "둘 다 나쁜데 덜 나쁜 쪽을 고르는 것"이고, 
이 선택은 비즈니스 도메인에 따라 달라진다.

- 금융/정산 → 데이터 손실이 더 치명적 → Replay 선택
- 실시간 추천/로그 → 약간의 불일관성 감수 가능 → Replay 안 함
```


**2. Dead-lettered 레코드 식별**

Replay로 재투입한 레코드와 정상 레코드를 구분해야 함.

방법:
- boolean 컬럼 `was_dead_lettered` 추가
- 더 완전한 메타데이터: job 이름, 버전, replay 시각 기록 (Metadata Decorator 패턴)

**3. 순서/일관성 파괴**

예시: IoT 센서가 매분 이벤트 전송, 5분 비활동 시 세션 종료 로직
- 5분간 이벤트가 Dead-Letter에 쌓임 → 컨슈머가 세션 닫아버림
- Replay로 이벤트 재투입해도 세션이 이미 닫힌 상태 → 데이터 불일치

순서 예시:
- 정상 처리: 10:00, 10:02 저장
- Replay: 10:01 재투입
- 최종 결과: 10:00, 10:02, 10:01 (순서 깨짐)

**4. Error-safe function의 함정**

error-safe function은 실패 시 예외 대신 NULL 반환 → 예외를 잡을 수 없음. Dead-Letter 구현이 더 복잡해짐:
- 예외 캡처 대신 입력값과 출력값 비교
- 입력은 있는데 출력이 NULL → 처리 실패로 간주
- 함수마다 error-safety 의미가 다름 → 주의 필요

**5. 에러 은폐 위험**
Dead-Letter 패턴은 에러를 숨김 → 시스템 전체 장애 징후를 놓칠 수 있음. → 일정 임계치 이상 Dead-Letter 쌓이면 알람 발송 + job 중단하는 alerting layer 필수

---

### 구현 예시 (Examples)

**예시 1 — Apache Flink (스트리밍): side output 방식**

Flink의 side output: 실패 레코드를 별도 출력 채널로 분기하는 내장 기능

```
# OutputTag 선언 — side output 채널 정의 (채널 이름과 타입 지정)
invalid_data_output: OutputTag = OutputTag('invalid_visits', Types.STRING())

# 매핑 함수에서 실패 레코드를 side output으로 분기
def map_rows(self, json_payload: str) -> str:
    try:
        evt = json.loads(json_payload)
        evt_time = int(datetime.datetime.fromisoformat(evt['event_time']))
        # 핵심: 정상 처리된 레코드는 yield로 메인 출력에 전달
        yield json.dumps({'visit_id': evt['visit_id'], 'event_time': evt_time, 'page': evt['page']})
    except Exception as e:
        # 핵심: 실패 레코드는 side output 채널로 전달 (에러 메타데이터 포함)
        yield self.invalid_data_output, _wrap_input_with_error(json_payload, e)

# 정상 레코드 → valid Kafka 토픽
# 실패 레코드 → invalid Kafka 토픽 (Dead-Letter 저장소)
visits.get_side_output(invalid_data_output).sink_to(kafka_sink_invalid_data)
visits.sink_to(kafka_sink_valid_data)
```

→ `yield self.invalid_data_output` 이 핵심 — 실패 레코드를 Dead-Letter 채널로 라우팅


**예시 2 — Apache Spark SQL (배치): error-safe transformation 방식**
error-safe `CONCAT` 함수 사용 — 실패 시 예외 대신 NULL 반환

```
# 1단계: CONCAT으로 변환 시도 + is_valid 플래그 생성
# CONCAT은 error-safe — 어느 컬럼이 NULL이면 예외 없이 NULL 반환
spark_session.sql('''
    SELECT type, full_name, version, name_with_version,
        CASE
            WHEN (full_name IS NOT NULL OR version IS NOT NULL)
            AND name_with_version IS NULL THEN false ELSE true
        END AS is_valid
    FROM (
        SELECT type, full_name, version,
        CONCAT(full_name, version) AS name_with_version  -- 핵심: error-safe 변환
        FROM devices_to_load
    )
''')

# 2단계: persist() — 쿼리를 두 번 실행하지 않도록 캐싱 (핵심)
devices_to_load_with_validity_flag.persist()

# 정상 레코드 → 메인 Delta 테이블
(devices_to_load_with_validity_flag.filter('is_valid IS TRUE')
    .drop('is_valid').write.mode('overwrite')
    .format('delta').save(f'{base_dir}/output/devices-table'))

# 실패 레코드 → Dead-Letter Delta 테이블
(devices_to_load_with_validity_flag.filter('is_valid IS FALSE')
    .drop('is_valid').write.mode('overwrite')
    .format('delta').save(f'{base_dir}/output/devices-dead-letter-table'))
```

→ `.persist()` 가 핵심 — 없으면 filter가 두 번 실행되어 쿼리 비용 2배 → SQL 선언형 언어에서 Dead-Letter 구현은 verbose해지는 단점 있음


**`.persist()`는 계산 결과를 메모리에 저장해두는 것이고, 없으면 같은 계산을 두 번 한다는 뜻이다.**


**문제 상황 이해**
코드 흐름이 이렇다:
```
1. 쿼리 실행 → is_valid 컬럼 포함한 DataFrame 생성
2. filter('is_valid IS TRUE')  → 정상 레코드만 골라서 저장
3. filter('is_valid IS FALSE') → 실패 레코드만 골라서 저장
```

---

**persist() 없을 때**
Spark는 lazy evaluation이라 실제로 저장할 때 계산을 시작함.

- 2번 filter 저장할 때 → 1번 쿼리 처음부터 실행
- 3번 filter 저장할 때 → 1번 쿼리 또 처음부터 실행

→ 같은 쿼리를 두 번 실행. 비용 2배.

---

**persist() 있을 때**
```
1. 쿼리 실행 → 결과를 메모리에 저장 (여기서 한 번만 실행)
2. filter('is_valid IS TRUE')  → 메모리에서 꺼내서 씀
3. filter('is_valid IS FALSE') → 메모리에서 꺼내서 씀
```
→ 쿼리 한 번만 실행. 메모리에서 두 번 읽기만 함.

---

**verbose 얘기는**
try-catch 방식이었으면 코드가 단순했을 것:
```
try:
    정상 처리 → 저장
except:
    Dead-Letter → 저장
```

그런데 SQL error-safe 방식은 예외를 안 던지니까 직접 `is_valid` 플래그를 만들고, persist하고, filter 두 번 해야 함. 코드가 길어지고 복잡해진다는 뜻.


#### +추가설명
**전체 목표: devices 테이블에서 `full_name + version` 합치기. 실패한 놈은 Dead-Letter에 저장.**

---

**1단계 — SQL 실행**

```
spark_session.sql('''
    SELECT type, full_name, version, name_with_version,
        CASE
            WHEN (full_name IS NOT NULL OR version IS NOT NULL)
            AND name_with_version IS NULL THEN false ELSE true
        END AS is_valid
    FROM (
        SELECT type, full_name, version,
        CONCAT(full_name, version) AS name_with_version
        FROM devices_to_load
    )
''')
```

안쪽 서브쿼리부터 읽어야 함:

```
-- 안쪽: CONCAT으로 합치기 시도
SELECT type, full_name, version,
CONCAT(full_name, version) AS name_with_version
FROM devices_to_load
```

결과:

```
type   | full_name | version | name_with_version
-------|-----------|---------|------------------
phone  | Galaxy    | S23     | GalaxyS23
tablet | NULL      | Pro11   | NULL
laptop | MacBook   | NULL    | NULL
```

`CONCAT`은 error-safe라 NULL이 있으면 예외 없이 그냥 NULL 반환.

```
-- 바깥쪽: is_valid 플래그 판단
CASE
    WHEN (full_name IS NOT NULL OR version IS NOT NULL)
    AND name_with_version IS NULL THEN false
    ELSE true
END AS is_valid
```

조건 해석: "full_name이나 version 둘 중 하나는 값이 있는데, 합친 결과가 NULL이면 → 변환 실패"

최종 결과:

```
type   | full_name | version | name_with_version | is_valid
-------|-----------|---------|-------------------|--------
phone  | Galaxy    | S23     | GalaxyS23         | true
tablet | NULL      | Pro11   | NULL              | false
laptop | MacBook   | NULL    | NULL              | false
```

---

**2단계 — persist()**

```
devices_to_load_with_validity_flag.persist()
```

여기가 핵심.

Spark는 lazy evaluation — `spark.sql(...)` 을 써도 실제로 계산을 안 함. "이렇게 계산해라"는 설계도만 들고 있음.

실제 계산은 `.write.save()` 처럼 결과가 필요한 순간에 시작됨.

persist() 없으면:

```
정상 레코드 저장할 때 → SQL 처음부터 실행
실패 레코드 저장할 때 → SQL 또 처음부터 실행
→ 같은 SQL 두 번 실행. 비용 2배.
```

persist() 있으면:

```
persist() 호출 시점 → SQL 딱 한 번 실행, 결과를 메모리에 보관
정상 레코드 저장할 때 → 메모리에서 꺼냄
실패 레코드 저장할 때 → 메모리에서 꺼냄
→ SQL 한 번만 실행.
```

---

**3단계 — 정상/실패 분리 저장**

```
# is_valid = true인 것만 → 정상 테이블
devices_to_load_with_validity_flag.filter('is_valid IS TRUE')
    .drop('is_valid')   # is_valid 컬럼은 내부 판단용이라 저장할 때 제거
    .write.mode('overwrite').format('delta').save('output/devices-table')

# is_valid = false인 것만 → Dead-Letter 테이블
devices_to_load_with_validity_flag.filter('is_valid IS FALSE')
    .drop('is_valid')
    .write.mode('overwrite').format('delta').save('output/devices-dead-letter-table')
```

---

**verbose가 뭔 말이냐**

try-catch 방식이었으면:

```
try:
    정상 처리 → 저장
except:
    Dead-Letter → 저장
```

2줄로 끝남.

근데 error-safe 방식은 예외를 안 던지니까:

- is_valid 플래그 직접 만들고
- persist() 하고
- filter() 두 번 하고
- drop() 두 번 하고
- save() 두 번 해야 함

코드가 훨씬 길고 복잡해진다는 뜻.

### 엔지니어 독백
Dead-Letter에 쌓인 레코드 수가 갑자기 급증했는데 알람이 없어 3시간 후에 발견한 사례. 
그 시간 동안 다운스트림 대시보드는 정상처럼 보였지만 실제 데이터의 40%가 Dead-Letter에 가 있던 상황. 
Dead-Letter 패턴을 쓴다고 끝이 아님 — 반드시 "Dead-Letter에 N건 이상 쌓이면 알람"을 같이 구현해야 함. 
패턴은 운영 편의를 위한 것이지, 에러를 조용히 묻어두라는 게 아님. CloudWatch 메트릭 또는 Kafka 컨슈머 lag 모니터링을 Dead-Letter 저장소에도 걸어둘 것.


# 3.2 중복된 레코드
## (1)패턴#10:윈도 중복제거
## 2. 중복 레코드 (Duplicated Records)

처리 불가 레코드 관리가 첫 번째 단계라면, 두 번째는 전달 의미론(delivery semantics) 문제. 
분산 시스템에서 exactly-once 전달은 매우 달성하기 어려움. 
현실에서는 at-least-once 환경이 일반적 — 레코드가 한 번 이상 도착할 수 있음. 
비즈니스 로직이 각 레코드를 정확히 한 번만 처리해야 한다면 중복 제거가 필요.

---

## 2-1. 패턴 #02: Windowed Deduplicator

데이터를 "범위가 있는 것"으로 간주해서 그 범위 안에서만 중복을 제거하는 패턴. 
배치는 현재 처리 중인 데이터셋이 범위, 
스트리밍은 시간 기반 윈도우가 범위.

### 배경 — 왜 나왔나

분산 시스템에서 네트워크 장애, 재시도 로직 등으로 같은 레코드가 여러 번 전달되는 것은 피하기 어려움. 비즈니스 입장에서는 주문 이벤트가 두 번 처리되면 매출 집계가 2배로 잡히는 치명적 문제 발생. 중복을 제거해서 각 레코드를 정확히 한 번만 처리하기 위한 패턴.

---

### 상황 (Problem)

문제상황:
- 배치 job이 스트리밍 레이어에서 object store로 동기화된 visit 이벤트를 처리
- 비즈니스 사용자에게 데이터를 직접 노출하므로 각 레코드에 대해 exactly-once 처리 보장 필요
- 문제: 스트리밍 레이어가 데이터 producer의 자동 재시도로 인해 중복 이벤트를 자주 생성

---

### 해결 (Solution)

구현 2단계:

```
[1단계] 중복 제거 기준 속성 식별
   레코드의 유일성을 보장하는 컬럼 조합 파악
   예: visit_id + visit_time

[2단계] 중복 제거 범위 정의
   배치: 현재 처리 중인 데이터셋 전체 (과거 데이터셋까지 포함 가능하나 컴퓨트 비용 증가)
   스트리밍: 시간 기반 윈도우 — 윈도우 안에서 이미 처리한 키를 state store에 보관
            (그래서 워터마크지정해서 처리하는듯)
```

**배치 vs 스트리밍 중복 제거 방식 차이**

배치:

- 한번에 전체 데이터를 갖고 있음
- `DISTINCT` 또는 `WINDOW` 함수로 간단히 처리
- 전체 데이터셋이 곧 묵시적 윈도우

스트리밍:

- 데이터가 끝없이 계속 들어옴 — 전체 데이터를 한번에 볼 수 없음
- 이미 처리한 레코드 키를 state store에 저장해서 중복 여부 확인
- 윈도우 시간이 지나면 state store에서 자동 삭제 (메모리 무한 증가 방지)

**State Store 3종류**

| 종류                      | 저장 위치        | 속도    | 장애 시     | 사용 상황                     |
| ----------------------- | ------------ | ----- | -------- | ------------------------- |
| Local                   | 메모리만         | 가장 빠름 | state 소실 | 테스트, state 손실 허용 시        |
| Local + fault-tolerance | 메모리 + 원격 저장소 | 빠름    | 복구 가능    | 실무 일반적 선택 (Spark default) |
| Remote                  | 원격 저장소만      | 느림    | 복구 가능    | latency 감수하고 일관성 최우선 시    |

### 실무에서 어떻게 상태스토어 구축해서 사용하나?

**실무에서는 "Local + fault-tolerance" 를 기본으로 사용한다.**

---

**EMR + Spark Structured Streaming 환경 기준**

State Store 기본값은 HDFS/S3 기반 RocksDB 또는 기본 내장 state store.

구체적으로:

- 메모리: Executor 메모리에 현재 윈도우의 키 보관
- 원격 저장소: S3 또는 HDFS에 checkpoint와 함께 state 주기적으로 저장
- 장애 시: S3의 마지막 checkpoint에서 state 복구 후 재시작

---

**Spark에서 state store 선택지**

기본 내장 (HDFS-based):

- 별도 설정 없이 checkpoint 경로만 잡으면 자동으로 S3에 state 저장
- 대부분의 경우 이걸로 충분

RocksDB state store (Spark 3.2+):

- 메모리 부족할 때 디스크로 overflow 가능
- state가 매우 큰 경우 (수억 개 키) 에 선택
- 설정: `spark.sql.streaming.stateStore.providerClass` 를 RocksDB로 변경

---

**Remote (Redis 등) 를 쓰는 경우**

- 여러 스트리밍 job이 같은 state를 공유해야 할 때
- 예: 사용자 세션 state를 여러 파이프라인이 동시에 읽고 써야 하는 경우
- 단점: 네트워크 왕복 비용 때문에 처리 속도 느려짐. latency 민감한 파이프라인에는 부적합

---

**실무 결정 기준 요약**

- 일반적인 deduplication, aggregation → 기본 내장 state store + S3 checkpoint
- state 크기가 매우 커서 메모리 부족 → RocksDB state store
- 여러 job 간 state 공유 필요 → Redis 등 Remote
- 테스트/로컬 개발 → Local (메모리만, checkpoint 없음)

---

### 고려사항 (Consequences)

**1. Space vs Time trade-off — 공간 대 시간 트레이드오프**

스트리밍에만 해당. 윈도우 크기 설정이 핵심 결정:

- 윈도우 짧게 → 메모리/state store 적게 사용. 단, 윈도우 밖 중복은 못 잡음
- 윈도우 길게 → 더 많은 중복 잡음. 단, state store에 보관할 키가 많아져 메모리/비용 증가

완벽한 중복 제거는 불가능. 얼마나 많은 중복을 허용할 것인가의 trade-off.

**2. Idempotent producer — 중복 제거가 exactly-once 전달을 보장하지 않음**

중복 레코드를 제거해도 transient 에러 + 자동 재시도가 발생하면 동일 레코드가 두 번 처리될 수 있음. 완전한 exactly-once 전달이 필요하면 Chapter 4의 idempotency 패턴을 함께 사용해야 함.

---

### 구현 예시 (Examples)

**예시 1 — Spark 배치: dropDuplicates**

```
# 가장 단순한 방법 — type, full_name, version 조합이 같으면 중복으로 판단
dataset = session.read.schema('...').format('json').load(f'{base_dir}/input')
deduplicated = dataset.dropDuplicates(['type', 'full_name', 'version'])
# 파라미터 없으면 전체 컬럼 기준으로 중복 제거
```

**예시 2 — SQL: WINDOW 함수 기반 중복 제거**

dropDuplicates 같은 native 함수가 없을 때 SQL로 직접 구현:

```
-- ROW_NUMBER()로 같은 키 조합 중 첫 번째만 남김
SELECT type, full_name, version FROM (
    SELECT type, full_name, version,
        ROW_NUMBER() OVER (
            PARTITION BY type, full_name, version  -- 중복 판단 기준 컬럼
            ORDER BY 1
        ) AS position
    FROM duplicated_devices
) WHERE position = 1  -- 핵심: 각 그룹에서 첫 번째 레코드만 통과
```

→ `PARTITION BY` 안의 컬럼 조합이 중복 판단 기준. 같은 조합이면 한 개만 남김.

**예시 3 — Spark 스트리밍: watermark + dropDuplicates**

스트리밍에서도 dropDuplicates 사용 가능. 단, watermark 설정이 필수:

```
# 1단계: 입력 데이터 파싱 — 시간 기반 컬럼 추출
event_schema = StructType([
    StructField("visit_id", StringType()),
    StructField("visit_time", TimestampType())
])

deduplicated_visits = (input
    .select(F.from_json("value", event_schema).alias("value_struct"), "value")
    .select("value_struct.visit_time", "value_struct.visit_id", "value")

    # 2단계: watermark 설정 — 핵심
    # visit_time 기준으로 10분 이상 늦게 도착한 레코드는 버림
    # 동시에 state store에서 10분 지난 키도 자동 삭제 (메모리 무한 증가 방지)
    .withWatermark("visit_time", "10 minutes")

    # 3단계: visit_id + visit_time 조합으로 중복 제거
    # watermark 범위 안에서만 중복 체크
    .dropDuplicates(["visit_id", "visit_time"])

    # 중복 제거용으로 쓴 컬럼은 최종 출력에서 제거
    .drop("visit_time", "visit_id"))
```

→ `withWatermark`가 핵심 — 없으면 state store가 무한히 커져 결국 OOM으로 job 죽음 → watermark 시간 = state store 보관 기간 = 중복 감지 가능 범위

---
### 자세한 예시
**State Store는 "이미 처리한 키 목록"을 메모리에 들고 있고, S3에 주기적으로 백업한다.**

---

**전체 그림**

```
Kafka → Spark Executor (메모리: State Store) → Output
                    ↕ 주기적 백업
                   S3 (checkpoint/state/)
```

---

**1. State Store가 뭘 저장하나**

deduplication 기준으로:

```
State Store 내용 (Executor 메모리):
┌──────────────────────────────────┐
│ visit_id  │ visit_time           │
│-----------|----------------------│
│ A001      │ 2024-01-15 10:00:01  │
│ B002      │ 2024-01-15 10:00:02  │
│ C003      │ 2024-01-15 10:00:04  │
└──────────────────────────────────┘

새 레코드 들어올 때마다:
1. State Store에서 visit_id 조회
2. 있으면 → 중복. 버림
3. 없으면 → 정상. 처리 + State Store에 추가
```

---

**2. S3에 어떤 형태로 저장되나**

Spark가 자동으로 checkpoint 경로 아래에 저장. 엔지니어가 신경 쓸 건 경로 설정뿐.

```
# 코드에서 설정하는 부분 — 여기가 전부
query = (deduplicated_visits
    .writeStream
    .option("checkpointLocation", "s3://my-bucket/checkpoints/visit-dedup/")
    .format("delta")
    .start("s3://my-bucket/output/visits/"))
```

S3에 실제로 쌓이는 구조:

```
s3://my-bucket/checkpoints/visit-dedup/
├── commits/          ← 어느 microbatch까지 처리했는지 기록
│   ├── 0
│   ├── 1
│   └── 2
├── offsets/          ← Kafka의 어느 offset까지 읽었는지 기록
│   ├── 0
│   ├── 1
│   └── 2
└── state/            ← State Store 내용 (메모리 백업본)
    └── 0/
        └── 0/
            ├── 1.delta   ← microbatch 1 시점의 state
            └── 2.delta   ← microbatch 2 시점의 state
```

---

**3. watermark와 state 자동 삭제**

watermark 10분 설정 시:

```
현재 시각 기준 10분 이전 키는 State Store에서 자동 삭제

10:00 → A001 추가
10:05 → B002 추가
10:11 → watermark가 10:01로 이동
        → A001 (10:00) 삭제됨 ← 10분 지났으므로
        → B002 (10:05) 아직 유지
```

watermark 없으면:

```
State Store에 키가 계속 쌓임
→ Executor 메모리 부족
→ OOM으로 job 죽음
```

---

**4. 장애 발생 시 복구 흐름**

```
[정상 운영]
Kafka → Executor (State Store: A001, B002, C003) → S3 output
                        ↓ microbatch마다 S3에 백업

[Executor 죽음]
S3 checkpoint 확인
→ "microbatch 2까지 처리했고, 그 시점 State Store는 이것"
→ Executor 재시작
→ S3에서 state 복구 (A001, B002, C003 다시 메모리에 로드)
→ Kafka offset 2 이후부터 재시작
→ 중복 없이 이어서 처리
```

checkpoint 없으면:

```
Executor 죽음 → 재시작 → State Store 비어있음
→ 이미 처리한 A001이 또 들어와도 "처음 보는 것"으로 처리
→ 중복 발생
```

---

**5. 실무 EMR 환경 전체 구성**

```
[Kafka]
    ↓ 이벤트 스트림
[EMR Spark Executor × N개]
    ├── 각 Executor가 자기 파티션의 State Store를 메모리에 보유
    ├── 기본: 128MB ~ 512MB per Executor 메모리 중 state 할당
    └── microbatch마다 (보통 10초~1분) S3에 state 스냅샷 저장
    ↓ 처리 완료 레코드
[S3 output / Delta Lake]

[S3 checkpoint]
    ├── state/ : State Store 백업
    ├── offsets/ : Kafka offset 기록
    └── commits/ : 완료된 microbatch 기록
```

---

**6. RocksDB는 언제 쓰나**

기본 state store는 state 전체를 메모리에 올림.

state가 너무 크면 (키가 수천만 개 이상):

```
기본 state store → OOM
RocksDB state store → 메모리 부족분을 Executor 로컬 디스크로 overflow
```

설정 한 줄 추가:

```
spark.conf.set(
    "spark.sql.streaming.stateStore.providerClass",
    "org.apache.spark.sql.execution.streaming.state.RocksDBStateStoreProvider"
)
```

---

**엔지니어 독백**
신입 때 제일 많이 하는 실수가 checkpointLocation 경로를 job마다 같은 걸로 쓰는 거야. 예를 들어 `s3://bucket/checkpoint/` 이렇게 하나로 고정해두면, job A의 state와 job B의 state가 뒤섞여서 재시작할 때 전혀 엉뚱한 state를 들고 일어남. 반드시 job마다 고유한 경로를 써야 해. `s3://bucket/checkpoint/visit-dedup-v1/` 이런 식으로. 그리고 로직을 바꿨을 때도 새 경로로 바꿔야 해. 기존 state가 새 로직이랑 호환이 안 되는 경우가 많거든. 이거 모르고 같은 경로 재사용했다가 이상한 state 들고 올라와서 중복이 갑자기 폭발하는 사고를 꽤 많이 봤어.

---
### 엔지니어 독백
스트리밍 deduplication에서 watermark를 너무 짧게 (1분) 설정했다가 producer 재시도 간격이 2분이라 중복이 전혀 안 잡힌 사례. 결과: 결제 이벤트가 2배로 집계되어 당일 매출 리포트가 실제의 2배로 잡혔고 경영진이 "역대 최고 매출"이라고 착각한 일이 있었음. watermark는 "producer의 최대 재시도 간격 + 네트워크 지연 + 충분한 버퍼"를 합산해서 설정해야 함. 짧을수록 메모리는 아끼지만 중복을 못 잡고, 길수록 중복은 잘 잡지만 state store 메모리 비용이 올라감. 도메인 특성에 맞게 결정할 것.
# 3.3 지연데이터
## (1)패턴#11: 지연데이터 탐직기
## (2)패턴#12: 정적 지연 데이터 통합기
## (3)패턴#13: 동적 지연 데이터 통합기

## 3. 지연 데이터 (Late Data)

처리 불가 레코드, 중복 레코드 다음 세 번째 문제. 겉으로 무해해 보이지만 파이프라인에 심각한 영향을 줌. 늦은 데이터를 다루는 첫 번째 단계는 탐지(detection). 탐지가 되어야 "버릴지, 따로 저장할지, 재처리할지" 전략을 적용할 수 있음.

---

## 3-1. 패턴 #11: Late Data Detector

늦게 도착한 데이터를 감지하는 패턴. 감지 자체가 목적이고, 감지 후 처리는 다음 패턴들(Late Data Integrator)이 담당.

### 배경 — 왜 나왔나

이벤트가 생성된 시각(event time)과 파이프라인이 처리하는 시각(processing time)은 다름. 네트워크 장애, 모바일 오프라인 상황 등으로 이벤트가 생성 후 한참 뒤에 도착하는 경우가 생김. 이걸 구분 없이 처리하면 이미 닫힌 집계 윈도우에 데이터가 끼어들어 결과가 틀어짐.

**Event Time vs Processing Time**

- Event Time: 이벤트가 실제로 발생한 시각 (사용자가 버튼 누른 시각)
- Processing Time: 파이프라인이 그 이벤트를 처리한 시각
- Processing Time은 절대 늦을 수 없음. Late Data는 항상 Event Time 기준

---

### 상황 (Problem)

문제상황:

- 블로깅 플랫폼 방문자가 visit 이벤트를 거의 실시간으로 생성 — 보통 15초 이내 도착
- 그러나 사용자가 네트워크 연결을 잃으면 이벤트를 로컬에 버퍼링
- 연결 복구 후 한꺼번에 flush → 수십 분~수 시간 늦게 도착
- 이 늦은 이벤트를 탐지해서 각 use case에 맞는 전략 적용 필요 (무시, 재처리 등)

1. 원칙: 
	- watermark는 현재 실제 시각을 전혀 고려하지 않는다. 
	- 오직 데이터 안에 담긴 event_time만 본다.
2. 파티션마다 이벤트시간이 다른데 어떻게 처리할 것 인가:
	- Kafka 파티션마다 이벤트 시각이 제각각일 때, watermark 기준점을 "가장 빠른 파티션"에 맞출지 "가장 느린 파티션"에 맞출지 선택하는 문제다.


---

### 해결 (Solution)

**구현 3단계**

```
[1단계] Event Time 기준 속성 정의
   언제 이벤트가 발생했는지 나타내는 컬럼 선택 (예: event_time)
   Processing Time이 아닌 Event Time이어야 함

[2단계] 파티션별 latency 집계 전략 결정
   각 파티션에서 현재까지 본 가장 최신 event time 추적
   → MAX 함수 사용 (단조 증가 보장)
   → MIN 함수 사용 금지 (stuck-in-the-past 문제 발생)

[3단계] Watermark 계산
   watermark = MAX(event_time) - allowed_lateness
   watermark보다 오래된 이벤트 = 늦은 데이터로 판단
```

**전역 집계 전략 3가지**

파티션이 여러 개일 때 전체를 대표하는 단일 event time 계산 방법:

|전략|의미|장점|단점|적합한 상황|
|---|---|---|---|---|
|MIN|가장 느린 파티션 기준|더 많은 데이터를 on-time으로 수용|버퍼 크기 커짐, 느린 파티션에 발목 잡힘|데이터 손실 허용 불가 시|
|MAX|가장 빠른 파티션 기준|버퍼 작음, 처리 빠름|느린 파티션 데이터 late 처리 위험|실시간성이 중요할 때|
|MIN + MAX 조합|데이터 소스별 다른 전략 적용|유연함|복잡도 높음|여러 데이터 소스를 다룰 때|

**Watermark 계산 예시 (allowed lateness = 30분)**

```
1번 microbatch 도착: 10:00, 10:05, 10:06
  watermark 후보 = MAX(10:00, 10:05, 10:06) - 30분 = 09:36
  output watermark = 09:36

2번 microbatch 도착: 9:20, 9:31, 10:07
  watermark 후보 = MAX(10:07) - 30분 = 09:37
  09:37 > 현재 watermark 09:36 → watermark 업데이트 = 09:37
  9:20, 9:31 → watermark(09:37)보다 오래됨 → 늦은 데이터로 판단 → 무시
```

watermark는 단조 증가 — 후보가 현재 watermark보다 낮으면 무시. 절대 뒤로 안 감.


### <자세한 예시 설명>

1.원칙: watermark는 현재 실제 시각(벽시계)을 전혀 보지 않는다. 오직 데이터 안에 담긴 event_time만 본다.
2.파티션마다 이벤트시간이 다른데 어떻게 처리할 것 인가:
- Kafka 파티션마다 이벤트 시각이 제각각일 때, watermark 기준점을 "가장 빠른 파티션"에 맞출지 "가장 느린 파티션"에 맞출지 선택하는 문제다.
3.워터마크
- 워터마크 : 이벤트타임 - 허용시간(30분)
- 워터마크 정의 : 윈도 단위에서 최소 이벤트 시간
풀어서 설명하면:
watermark는 "이 시각보다 오래된 이벤트는 늦은 데이터로 버린다"는 기준선이다.
이 기준선을 계산하려면 먼저 "지금 전체 파티션 중 어느 파티션의 시각을 대표값으로 쓸 것인가"를 결정해야 한다.
```
파티션 0 (서울): 10:50
파티션 1 (부산): 10:48
파티션 2 (제주): 09:30  ← 장애로 90분 밀림

MAX 선택 → 가장 빠른 서울(10:50) 기준
         → watermark = 10:20
         → 제주 데이터(09:30) 버려짐

MIN 선택 → 가장 느린 제주(09:30) 기준
         → watermark = 09:00
         → 제주 데이터(09:30) 살아남
         → 단, 제주 장애 지속 시 watermark 전체가 멈춤
```
---

**상황 설정**

```
현재 실제 시각: 11:00
watermark 허용 지연: 30분
Kafka 파티션: 3개
```

이커머스 주문 이벤트를 Kafka로 받는 스트리밍 job.

```
파티션 0: 서울 리전 주문
파티션 1: 부산 리전 주문
파티션 2: 제주 리전 주문
```

현재 각 파티션에서 지금까지 본 가장 최신 event_time:

```
파티션 0 (서울): 10:50  ← 정상. 현재 11:00 기준 10분 지연
파티션 1 (부산): 10:48  ← 거의 정상. 12분 지연
파티션 2 (제주): 09:30  ← 네트워크 장애. 90분 밀림
```

---

**핵심 질문: "지금 전체 기준 시간(watermark)을 어디로 잡을 것인가?"**

---

**MAX를 쓰면**

```
현재 실제 시각: 11:00
각 파티션 최신 event_time: 10:50, 10:48, 09:30

MAX(10:50, 10:48, 09:30) = 10:50
watermark = 10:50 - 30분 = 10:20

→ "10:20 이전 이벤트는 늦은 데이터"
```

제주(파티션 2)에서 09:30짜리 이벤트가 계속 들어오고 있음:

```
09:30 < watermark 10:20 → 늦은 데이터 → 버려짐
```

결과:

```
서울 주문 (10:50) → 정상 처리
부산 주문 (10:48) → 정상 처리
제주 주문 (09:30) → 전부 버려짐 ← 90분치 주문 데이터 손실
```

---

**MIN을 쓰면**

```
현재 실제 시각: 11:00
각 파티션 최신 event_time: 10:50, 10:48, 09:30

MIN(10:50, 10:48, 09:30) = 09:30
watermark = 09:30 - 30분 = 09:00

→ "09:00 이전 이벤트만 늦은 데이터"
```

제주(파티션 2)에서 09:30짜리 이벤트가 들어옴:

```
09:30 > watermark 09:00 → 정상 데이터 → 처리됨
```

결과:

```
서울 주문 (10:50) → 정상 처리
부산 주문 (10:48) → 정상 처리
제주 주문 (09:30) → 정상 처리 ← 데이터 손실 없음
```

---

**그러면 MIN이 좋은 거 아닌가? 문제가 뭔데?**

제주 파티션 장애가 계속되는 상황:

```
현재 실제 시각: 11:00
watermark 허용 지연: 30분

11:00 시점: 파티션 2 = 09:30 → MIN = 09:30 → watermark = 09:00
11:01 시점: 파티션 2 = 09:31 → MIN = 09:31 → watermark = 09:01
11:02 시점: 파티션 2 = 09:32 → MIN = 09:32 → watermark = 09:02
...
12:00 시점: 파티션 2 = 10:30 → MIN = 10:30 → watermark = 10:00
```

실제 시각은 11:00 → 12:00으로 1시간 흘렀는데 watermark는 09:00 → 10:00으로 1시간 뒤처진 채 진행 중.

이게 왜 문제냐:

```
현재 실제 시각: 11:00
스트리밍 job: 10분 윈도우로 주문 건수 집계

10:00~10:10 윈도우를 닫으려면
→ watermark가 10:10을 넘어야 함
→ 그런데 제주 장애로 watermark = 09:00에 머물고 있음
→ 10:00~10:10 윈도우를 닫지 못함
→ 결과가 안 나옴
→ 대시보드가 1시간째 업데이트 안 됨
```

제주 파티션이 완전히 죽어버리면:

```
현재 실제 시각: 계속 흐름 (11:00 → 12:00 → 13:00...)
파티션 2 event_time: 09:30에서 영원히 멈춤

MIN이 09:30에서 영원히 안 올라감
→ watermark = 09:00에서 영원히 안 올라감
→ 10:00~10:10 윈도우가 영원히 닫히지 않음
→ 결과가 영원히 안 나옴  ← stuck-in-the-past
```

---

**파티션 내부에서는 왜 무조건 MAX냐**

```
현재 실제 시각: 11:00
watermark 허용 지연: 30분

파티션 0 안에서 이벤트가 순서 뒤섞여 도착:
10:50 도착 → 10:48 도착 → 10:51 도착
(네트워크 지연으로 10:50이 10:48보다 먼저 도착한 상황)
```

MIN을 쓰면:

```
10:50 처리 → MIN = 10:50 → watermark = 10:20
10:48 처리 → MIN = 10:48 → watermark = 10:18  ← 뒤로 감!
10:51 처리 → MIN = 10:48 → watermark = 10:18  ← 여전히 뒤로 간 상태
```

watermark가 10:20 → 10:18로 뒤로 가버림:

```
이미 watermark 10:20 기준으로 닫은 윈도우가 있었다면
→ 10:18로 watermark가 후퇴
→ 닫았던 윈도우를 다시 열어야 함
→ 또 닫음 → 또 열림 → open-close-open 무한 루프
```

MAX를 쓰면:

```
10:50 처리 → MAX = 10:50 → watermark = 10:20
10:48 처리 → MAX = 10:50 → watermark = 10:20  ← 그대로 유지
10:51 처리 → MAX = 10:51 → watermark = 10:21  ← 앞으로만 감
```

watermark는 절대 뒤로 안 감. 단조 증가 보장.

---

**결론: 언제 무엇을 쓰나**

```
파티션 내부 집계
→ 무조건 MAX
→ 이유: 이벤트 순서 뒤섞임 발생 시 watermark가 뒤로 가는 것 방지

전체 파티션 통합 집계
→ 데이터 손실 절대 안 되는 도메인 (금융, 결제)
   → MIN 사용
   → 단, 느린 파티션 장애 모니터링 필수
   → 장애 파티션 하나가 전체를 멈출 수 있음을 인지

→ 실시간성이 중요한 도메인 (광고 클릭, 실시간 대시보드)
   → MAX 사용
   → 느린 파티션 데이터는 Late Data Integrator로 나중에 재처리

→ 여러 데이터 소스를 동시에 처리하는 경우
   → 소스 내부: MAX (단조 증가 보장)
   → 소스 간 통합: MIN (소스 하나 장애 시 다른 소스 데이터 버리지 않음)
```

---

### 고려사항 (Consequences)

**1. Late Data Capture 지원 여부**

프레임워크마다 늦은 데이터 탐지 + 캡처 지원 수준이 다름:

|프레임워크|늦은 데이터 탐지|늦은 데이터 캡처|
|---|---|---|
|Spark Structured Streaming|지원 (withWatermark)|별도 API 없음 — 구현 복잡|
|Apache Flink|지원|지원 (side output으로 쉽게 캡처)|
```
Spark:
이벤트 B (10:15) 감지 → "늦은 데이터네" → 그냥 버림. 끝.
버려진다는 사실을 엔지니어가 알 방법이 없음

Flink:
이벤트 B (10:15) 감지 → "늦은 데이터네" → side output으로 별도 저장
→ 엔지니어가 나중에 꺼내서 재처리 가능
```

**2. MIN 전략의 stuck-in-the-past 문제**

파티션별 event time 추적에 MIN을 쓰면:

```
시나리오: watermark가 10:30까지 진행됨
→ 10:30 이전 state를 모두 완료로 처리하고 emit

그런데 늦은 데이터가 계속 들어와 MIN 기준 watermark가 9:00으로 후퇴
→ 이미 emit한 state를 다시 열어야 함 (open-close-open 무한 루프)
→ 늦은 데이터가 계속 오면 watermark 자체가 진행 못 함 (stuck-in-the-past)
→ state store가 무한히 커짐
```

→ 파티션별 집계는 반드시 MAX 사용

**3. MAX 전략의 event skew 문제**

데이터 소스 5개 중 4개가 네트워크 장애로 40분 늦게 도착하는 상황:

```
정상 소스 1개: MAX event time = 10:40
장애 소스 4개: MAX event time = 10:00

전체 MAX = 10:40 → watermark = 10:10
장애 소스 데이터(10:00)는 watermark(10:10)보다 오래됨 → 전부 late 처리
```

완벽한 해결책 없음. 모니터링 + Late Data Integrator 패턴으로 재처리하는 게 최선.

---

### 구현 예시 (Examples)

**예시 1 — Spark Structured Streaming: withWatermark**

```
# Kafka value는 바이트라 먼저 문자열로 변환
# {"visit_id": 1, "event_time": "2024-01-15 10:00:00", "page": "/home"} 형태
# from_json으로 JSON 파싱 후 visit_id, event_time, page 컬럼으로 풀어냄

visits_events = (input_data
    .selectExpr('CAST(value AS STRING)')
    .select(F.from_json('value', 'visit_id INT, event_time TIMESTAMP, page STRING')
        .alias('visit'))
    .selectExpr('visit.*'))



# watermark = MAX(event_time) - 1시간
# 1시간 이상 늦게 도착한 이벤트는 늦은 데이터로 판단하고 버림
# 10분 윈도우마다 방문 건수 집계
# 예: [10:00-10:10] → 152건, [10:10-10:20] → 87건

session_window = (visits_events
    .withWatermark('event_time', '1 hour')
    .groupBy(F.window(F.col('event_time'), '10 minutes'))
    .count())
```

**withWatermark 동작 흐름 예시 (윈도우 10분, watermark 1시간)**

```
event_time 03:15 도착 → watermark = 02:15, [03:10-03:20] 윈도우 오픈
event_time 03:00 도착 → watermark = 02:15, [03:00-03:10] 윈도우 추가
event_time 01:50 도착 → watermark(02:15)보다 오래됨 → 무시
event_time 04:31 도착 → watermark = 03:31
                      → [03:00-03:10], [03:10-03:20] 윈도우 emit (watermark 초과)
                      → [04:30-04:40] 윈도우 오픈
```

→ Spark는 자동으로 늦은 데이터를 무시하지만, 무시된 데이터를 따로 캡처하는 API는 없음

**예시 2 — Apache Flink: 늦은 데이터 캡처 (side output)**

Flink는 watermark 값에 직접 접근 가능 → 늦은 데이터를 별도 저장소로 캡처 가능

```
# 1단계: event_time 추출기 정의
class VisitTimestampAssigner(TimestampAssigner):
    def extract_timestamp(self, value: Any, record_timestamp: int) -> int:
        event = json.loads(value)
        event_time = datetime.datetime.fromisoformat(event['event_time'])
        return int(event_time.timestamp())

# 2단계: 늦은 데이터 판단 + 분기 처리
class VisitLateDataProcessor(ProcessFunction):
    def __init__(self, late_data_output: OutputTag):
        self.late_data_output = late_data_output

    def process_element(self, value: Visit, ctx: 'ProcessFunction.Context'):
        current_watermark = ctx.timer_service().current_watermark()
        if current_watermark > value.event_time:
            # 핵심: event_time이 watermark보다 오래됨 → 늦은 데이터 → side output으로 분기
            yield (self.late_data_output, json.dumps({'visit': value, 'is_late': True}))
        else:
            # 정상 데이터 → 메인 출력
            yield json.dumps({'visit': value, 'is_late': False})

# 3단계: watermark 전략 설정 + 파이프라인 연결
watermark_strategy = (WatermarkStrategy
    # 5초까지 순서 뒤바뀜 허용
    .for_bounded_out_of_orderness(Duration.of_seconds(5))
    .with_timestamp_assigner(VisitTimestampAssigner()))

late_data_output = OutputTag('late_events', Types.STRING())

visits = (data_source
    .map(map_json_to_visit)
    .process(VisitLateDataProcessor(late_data_output), Types.STRING()))

# 정상 데이터 → 메인 Kafka 토픽
visits.sink_to(kafka_sink_valid_data)
# 늦은 데이터 → 별도 Kafka 토픽 (재처리 또는 분석용)
visits.get_side_output(late_data_output).sink_to(kafka_sink_late_visits)
```

→ `ctx.timer_service().current_watermark()` 가 핵심 — 현재 watermark 값에 직접 접근해서 늦은 데이터 판단 → Spark와 달리 Flink는 늦은 데이터를 side output으로 캡처해 별도 저장 가능

---

### 엔지니어 독백

allowed lateness를 너무 넉넉하게 (24시간) 잡았다가 state store가 하루치 이벤트를 전부 메모리에 들고 있어서 Executor OOM이 발생한 사례. 하루 이벤트가 수억 건이었는데 watermark가 24시간이니까 그 안에 들어온 모든 visit_id를 state store가 기억하고 있어야 했음. 결국 Executor 메모리가 터지면서 job 전체가 죽었고, 재시작해도 같은 문제 반복. allowed lateness는 실제 데이터 특성을 분석해서 설정해야 함 — "늦게 오는 데이터가 실제로 얼마나 늦게 오는가"를 로그로 먼저 측정하고, 95th percentile 기준으로 설정하는 게 실무 관행.




## 3-2. 패턴 #12: Static Late Data Integrator

늦은 데이터를 무시하지 않고, 고정된 기간만큼 과거를 되돌아보며 늦게 도착한 데이터를 메인 파이프라인에 통합하는 패턴.

### 배경 — 왜 나왔나

Late Data Detector는 늦은 데이터를 감지하고 버리는 것까지만 함. 그런데 늦은 데이터가 비즈니스적으로 중요한 경우 (예: 이커머스 주문의 절반이 늦게 도착) 버리는 것은 선택지가 안 됨. 늦은 데이터를 살려서 이미 처리한 파티션에 통합해야 하는 요구에서 나온 패턴.

---

### Processing Time 기반 파티션으로 도망가면 안 되는 이유

늦은 데이터 문제를 피하려고 event_time 대신 processing_time(데이터가 도착한 시각) 기준으로 파티션을 나누는 팀들이 있음. 이러면 내 파이프라인은 늦은 데이터 문제가 없어 보임.

그런데 이건 문제를 다운스트림으로 떠넘기는 것:

```
내 파이프라인: processing_time 기준 파티션
→ 09:00 파티션 안에 이런 데이터가 섞여 있음:
   - 09:00 이벤트 80%
   - 08:00 이벤트 10%  ← 1시간 늦게 도착
   - 07:00 이벤트 10%  ← 2시간 늦게 도착

다운스트림이 event_time 기준으로 처리하면:
→ 08:00, 07:00 이벤트가 늦은 데이터로 인식됨
→ 결국 다운스트림이 Late Data 문제를 처리해야 함
```

문제가 사라진 게 아니라 다운스트림으로 이동한 것. 근본 해결이 아님.

---

### 상황 (Problem)

문제상황:

- 매일 배치 job이 블로그 포스트를 참조한 외부 사이트 통계를 생성
- 늦은 데이터 허용 기간: 최대 15일 (15일 이상 된 데이터는 버림)
- 현재 문제: 배치 job이 당일 데이터만 처리 → 15일 이내 늦은 데이터는 통합 안 됨
- 목표: 매일 실행할 때 당일 + 과거 15일치 늦은 데이터를 함께 처리

---

### 해결 (Solution)

**Static Lookback Window 정의**

lookback window: "현재 실행 기준으로 얼마나 과거까지 되돌아볼 것인가"를 고정값으로 설정.

```
오늘: 2024-12-31
lookback window: 14일

→ 이번 실행이 처리하는 날짜:
   2024-12-31 (오늘)
   2024-12-30 (1일 전)
   2024-12-29 (2일 전)
   ...
   2024-12-17 (14일 전)

→ 2024-12-15짜리 늦은 데이터가 오늘 도착해도
   lookback window(14일) 밖이라 무시
```

**늦은 데이터 통합 위치 — 3가지 전략**

```
전략 1: 늦은 데이터 먼저 (Sequential - Late First)
   [과거 파티션 재처리] → [오늘 데이터 처리]

전략 2: 동시 처리 (Parallel)
   [과거 파티션 재처리]
          +               → [완료]
   [오늘 데이터 처리]

전략 3: 오늘 먼저 (Sequential - Current First)
   [오늘 데이터 처리] → [과거 파티션 재처리]
```

선택 기준:

|파이프라인 종류|권장 전략|이유|
|---|---|---|
|Stateful (이전 결과에 의존)|전략 1 (늦은 데이터 먼저)|오늘 결과를 만들려면 과거 데이터가 먼저 정확해야 함|
|Stateless (독립적 처리)|전략 2 또는 3|순서 무관. 오늘 데이터 먼저 내보내고 싶으면 전략 3|

**Stateful 예시:**

```
오늘 사용자 누적 구매액을 계산하는 파이프라인
→ 어제 누적액이 틀려 있으면 오늘 계산도 틀림
→ 반드시 과거 늦은 데이터 먼저 통합 후 오늘 실행
```

**Stateless 예시:**

```
오늘 페이지별 방문수를 계산하는 파이프라인
→ 어제 방문수와 무관하게 독립적으로 계산 가능
→ 오늘 먼저 처리하고 과거 늦은 데이터는 나중에 통합해도 됨
```

---

### 고려사항 (Consequences)

**1. Snowball backfilling effect**

내가 과거 파티션을 재처리하면 다운스트림도 그 파티션을 재처리해야 함. 다운스트림의 다운스트림도 재처리 필요 → 연쇄 backfilling 발생.

대응: 재처리한 파티션 목록을 다운스트림에 즉시 통보. 다운스트림이 어떻게 할지는 그쪽이 결정.

**2. Overlapping executions — 겹치는 backfill 문제**

lookback window가 4일인 파이프라인을 3일치 backfill하는 상황:

```
2024-10-10 실행: 10-09, 10-08, 10-07, 10-06 재처리
2024-10-11 실행: 10-10, 10-09, 10-08, 10-07 재처리  ← 10-07~09 중복
2024-10-12 실행: 10-11, 10-10, 10-09, 10-08 재처리  ← 10-08~10 중복
```

10-07, 10-08, 10-09가 여러 번 중복 재처리됨 → 컴퓨트 낭비 + 데이터 중복 위험.

해결: 가장 최신 실행(10-12)만 재실행하면 됨. lookback window가 알아서 이전 날짜들을 커버.

**3. Pipeline trigger — backfill은 반드시 메인 파이프라인 안에서**

lookback window 기반 backfill을 별도 파이프라인으로 분리하면 안 됨. → 위의 overlapping 문제가 그대로 발생. → 반드시 메인 파이프라인 안의 task로 포함시켜야 함.

**4. 리소스 낭비**

lookback window 기간에 늦은 데이터가 없어도 매번 재처리 실행. → 데이터 없는 날도 컴퓨트 비용 발생. → 해결: 재처리 전 "늦은 데이터가 있는지" 체크하는 control task 추가. 또는 Dynamic Late Data Integrator 패턴 사용 (다음 패턴).

**5. 시간 파티션 없으면 사용 불가**

데이터셋에 시간 개념(event_time 기반 파티션)이 없으면 이 패턴 적용 불가.

---

### 구현 예시 (Examples)

**Airflow Dynamic Task Mapping으로 구현**

Dynamic Task Mapping: 실행 시점에 데이터를 보고 task 수를 동적으로 생성하는 Airflow 기능.

```
# 1단계: backfill할 날짜 목록 생성
# lookback window = 2일 → 오늘 기준 2일 전까지 날짜 리스트 반환
@task
def generate_backfilling_runs():
    dr: DagRun = get_current_context()['dag_run']
    backfilling_dates = []
    days_to_backfill = 2

    # 시작일: 오늘 - 2일
    start_date_to_backfill = dr.execution_date - datetime.timedelta(days=days_to_backfill)

    # 2일치 날짜 리스트 생성
    for days_to_add in range(0, days_to_backfill):
        date_to_backfill = start_date_to_backfill + datetime.timedelta(days=days_to_add)
        backfilling_dates.append(date_to_backfill.date().strftime('%Y-%m-%d'))

    return backfilling_dates
# 반환값 예시 (오늘이 2024-12-31이면):
# ['2024-12-29', '2024-12-30']
```

```
# 2단계: 날짜별 늦은 데이터 통합 task
# expand()가 핵심 — backfilling_dates 리스트 원소 수만큼 task를 자동 생성
@task
def integrate_late_data(late_date: str):
    copy_file(late_date)  # 해당 날짜 파티션 재처리

integrate_late_data.expand(late_date=generate_backfilling_runs())
# 결과: integrate_late_data('2024-12-29'), integrate_late_data('2024-12-30') 2개 task 자동 생성
```

```
# 3단계: 전체 DAG 흐름 (전략 3: 오늘 먼저, 늦은 데이터 나중)
backfilling_runs_generator = generate_backfilling_runs()

(file_to_load_sensor          # 오늘 파일 도착 대기
    >> load_current_file()    # 오늘 데이터 처리
    >> backfilling_runs_generator                              # backfill 날짜 목록 생성
    >> integrate_late_data.expand(late_date=backfilling_runs_generator))  # 날짜별 재처리
```

실행 시 Airflow DAG 모습:

```
[file_to_load_sensor]
        ↓
[load_current_file]        ← 오늘(2024-12-31) 처리
        ↓
[generate_backfilling_runs]
        ↓
[integrate_late_data.2024-12-29]  ← 동적 생성
[integrate_late_data.2024-12-30]  ← 동적 생성
```

---

### 엔지니어 독백

lookback window를 30일로 잡고 매일 실행했더니 매 실행마다 30개 파티션을 재처리하는 비용이 쌓여 EMR 클러스터 비용이 월말에 3배로 튀어오른 사례. 늦은 데이터가 실제로 들어오는 날은 한 달에 5일도 안 됐는데 30일치를 매일 돌린 것. 책 권장대로 재처리 전에 "해당 파티션에 실제로 늦은 데이터가 있는지" 체크하는 control task를 앞에 달아서 데이터 없으면 skip하도록 수정했더니 비용이 정상화됨. lookback window 크기와 실제 늦은 데이터 빈도를 먼저 측정하고 설정할 것.


## 3-3. 패턴 #05: Dynamic Late Data Integrator

Static Late Data Integrator가 고정 기간만큼 무조건 과거를 재처리했다면, Dynamic은 실제로 늦은 데이터가 있는 파티션만 골라서 재처리하는 패턴.

### 배경 — 왜 나왔나

Static의 문제:

- lookback window가 고정 (예: 15일) → 15일 이상 늦은 데이터는 무조건 버림
- 늦은 데이터가 없는 날도 매번 과거 파티션을 재처리 → 컴퓨트 낭비

비즈니스 요구가 "15일 이상 늦은 데이터도 살려야 한다"로 바뀌는 순간 Static은 한계에 부딪힘. Dynamic은 고정 기간 없이 실제로 변경된 파티션만 찾아서 재처리.

---

### 상황 (Problem)

문제상황:

- 기존에 Static Late Data Integrator로 15일치 늦은 데이터를 처리하고 있었음
- 제품 오너가 "15일 초과한 늦은 데이터도 전부 통합하라"고 요구
- 고정 lookback window로는 불가능 → 동적으로 변경된 파티션만 찾아서 재처리 필요

---

### 해결 (Solution)

**핵심 아이디어: State Table**

"어느 파티션이 마지막으로 언제 처리됐고, 마지막으로 언제 업데이트됐는지"를 기록하는 별도 테이블을 유지.

```
State Table 구조:
┌──────────────┬──────────────────────┬──────────────────────┐
│ partition    │ last_processed_time  │ last_update_time     │
├──────────────┼──────────────────────┼──────────────────────┤
│ 2024-12-17   │ 2024-12-17T10:20     │ 2024-12-17T03:00     │ ← 정상. 처리 후 업데이트 없음
│ 2024-12-18   │ 2024-12-18T09:55     │ 2024-12-20T10:12     │ ← 늦은 데이터 있음!
└──────────────┴──────────────────────┴──────────────────────┘

2024-12-18 파티션:
- 내가 마지막으로 처리한 시각: 2024-12-18T09:55
- 파티션이 마지막으로 업데이트된 시각: 2024-12-20T10:12
- 처리 후에 업데이트가 생겼다 → 늦은 데이터가 들어온 것
```

**재처리 대상 파티션 쿼리**

```
-- 조건 3가지를 모두 만족하는 파티션만 backfill 대상
SELECT partition FROM state_table WHERE
    last_update_time > last_processed_time   -- 처리 후에 새 데이터가 들어옴
    AND partition < current_partition        -- 현재 처리 중인 파티션보다 과거
    AND is_processed = false                 -- 아직 backfill 진행 중이 아님
```

**파이프라인 전체 흐름**

```
[데이터 처리 완료]
        ↓
[State Table: last_processed_time 업데이트]
        ↓
[State Table 쿼리: last_update_time > last_processed_time 인 파티션 조회]
        ↓
[조회된 파티션만 동적으로 재처리]
```

**last_update_time을 어떻게 얻나**

데이터 저장소마다 다름:

|저장소|방법|비고|
|---|---|---|
|BigQuery|`INFORMATION_SCHEMA.PARTITIONS` 뷰의 `last_modified_timestamp`|기본 제공|
|Apache Iceberg|파티션 메타데이터 테이블의 `last_updated_at`|기본 제공|
|Delta Lake|`DeltaLog.getChanges()` 로 버전별 변경 파티션 추출|코드 필요|
|기타|직접 update tracking 테이블 구현|복잡|

---

### 고려사항 (Consequences)

**1. 동시 실행 시 중복 backfill 문제**

파이프라인이 동시에 4개 실행 중인 상황:

```
실행 A: 2024-12-13 처리 중 → state table 조회 → 12-10, 12-11 backfill 대상 발견
실행 B: 2024-12-14 처리 중 → state table 조회 → 12-10, 12-11 backfill 대상 발견
실행 C: 2024-12-15 처리 중 → state table 조회 → 12-10, 12-11 backfill 대상 발견

→ 12-10, 12-11이 3번씩 중복 재처리됨
```

해결: state table에 `is_processed` 컬럼 추가

```
State Table (is_processed 추가):
┌──────────────┬──────────────────────┬──────────────────────┬──────────────┐
│ partition    │ last_processed_time  │ last_update_time     │ is_processed │
├──────────────┼──────────────────────┼──────────────────────┼──────────────┤
│ 2024-12-10   │ 2024-12-10T10:00     │ 2024-12-12T08:00     │ true         │ ← 이미 backfill 중
│ 2024-12-11   │ 2024-12-11T10:00     │ 2024-12-13T09:00     │ true         │ ← 이미 backfill 중
└──────────────┴──────────────────────┴──────────────────────┴──────────────┘

실행 B, C가 조회하면 is_processed = true → 건너뜀 → 중복 없음
```

동시 실행 보호를 위한 파이프라인 조정 3가지:

- 파이프라인 시작 시 현재 파티션의 `is_processed = true` 로 먼저 업데이트
- backfill 대상 파티션 조회 후 해당 파티션들도 `is_processed = true` 로 업데이트
- 처리 완료 후 `last_processed_time` 업데이트 + `is_processed = false` 로 리셋 (다음 늦은 데이터 대비)

**2. Stateful 파이프라인 + 매우 늦은 데이터**

Stateful 파이프라인(이전 실행 결과에 의존)에서 아주 오래된 늦은 데이터가 오면:

```
마지막 성공 실행: 2024-10-20
늦은 데이터 발견: 2024-09-21 파티션에 새 데이터

→ 2024-09-21부터 2024-10-20까지 전체 재처리 필요 (약 30일치)
→ 그 사이 모든 파티션이 연쇄적으로 재처리됨
→ 컴퓨트 비용 폭발
```

대응: Dynamic이라도 최대 허용 lookback 기간을 별도로 설정. 그 이상 오래된 데이터는 버림.

**3. 스케줄링 복잡도**

- 저장소마다 last_update_time 추출 방법이 다름
- 일부 저장소는 직접 구현 필요
- 동시 실행 보호 로직까지 추가하면 파이프라인 복잡도가 상당히 올라감

---

### 구현 예시 (Examples)

**Airflow + Delta Lake 기반 구현**

Delta Lake는 last_update_time을 기본 제공하지 않아서 버전 기반으로 변경 파티션을 추출:

```
# Delta Lake 버전 기반으로 변경된 파티션 추출 (Scala)
# DeltaLog.getChanges(0): 버전 0부터 모든 변경 이력 조회
val deltaLog = DeltaLog.forTable(sparkSession, tableFullPath)
val partitionsChangeVersions = deltaLog.getChanges(0)
    .flatMap { case (version, actions) =>
        # 각 버전에서 추가/삭제된 파일의 파티션 추출
        val changedPartitions = actions.map {
            case addFile: AddFile if addFile.dataChange =>
                Some(addFile.partitionValues.map(e => s"${e._1}=${e._2}").mkString("/"))
            case removeFile: RemoveFile if removeFile.dataChange =>
                Some(removeFile.partitionValues.map(e => s"${e._1}=${e._2}").mkString("/"))
            case _ => None
        }.filter(_.isDefined).map(_.get).toSet
        changedPartitions.map(partition => (partition, version))
    }

# 파티션별 가장 최신 버전만 남김
# 예: 2024-12-18 파티션이 버전 3, 5, 7에서 변경됐으면 → 7만 남김
val lastVersionForEachPartition = partitionsChangeVersions
    .toSeq.groupBy(_._1).mapValues(pairs => pairs.map(_._2).max)
```

```
# State Table과 비교해서 backfill 대상 파티션 추출
# lastProcessedVersion: 내가 마지막으로 처리한 Delta 버전
# lastVersionPerPartition: Delta Lake에서 실제 최신 버전
val partitionsToBackfill = sparkSession.read.format("delta").load(TablePath)
    .select("partition", "isProcessed", "lastProcessedVersion")
    .as[PartitionState]
    .filter(state => state.lastProcessedVersion.isDefined)
    .filter(state =>
        # 조건 1: Delta Lake에 새 버전이 있음 (내가 처리한 버전보다 최신)
        (!lastVersionPerPartition.contains(state.partition) ||
         lastVersionPerPartition(state.partition) > state.lastProcessedVersion.get)
        # 조건 2: 아직 backfill 진행 중이 아님
        && !state.isProcessed
        # 조건 3: 현재 처리 중인 파티션보다 과거
        && state.partition < currentPartition
    )
# 결과: ["2024-12-18", "2024-12-15"] 같은 backfill 대상 날짜 목록
```

```
# Airflow DAG — 동시 실행 보호 + Dynamic Task Mapping
# depends_on_past=True: 이전 실행이 성공해야 다음 실행 가능
# → 동시에 여러 실행이 같은 파티션을 중복 backfill하는 것 방지
with DAG('devices_loader', max_active_runs=5,
         default_args={'depends_on_past': False}) as dag:

    # 현재 파티션 is_processed = true 마킹 (다음 실행이 중복 처리 못 하게)
    # depends_on_past=True: 이전 실행의 이 task가 성공해야 실행
    processing_marker = SparkKubernetesOperator(
        task_id='mark_partition_as_being_processed',
        depends_on_past=True
    )

    # backfill 대상 파티션 조회 + is_processed = true 마킹
    # depends_on_past=True: race condition 방지
    backfill_creation_job = SparkKubernetesOperator(
        task_id='get_late_partitions_and_mark_them_as_being_processed',
        depends_on_past=True
    )
```

전체 DAG 흐름:

```
[mark_partition_as_being_processed]   ← 현재 파티션 잠금 (depends_on_past=True)
        ↓
[메인 데이터 처리]
        ↓
[get_late_partitions_and_mark_them]   ← backfill 대상 조회 + 잠금 (depends_on_past=True)
        ↓
[integrate_late_data.2024-12-18]      ← 동적 생성
[integrate_late_data.2024-12-15]      ← 동적 생성
        ↓
[update_last_processed_time + is_processed = false]  ← 처리 완료 + 잠금 해제
```

---

### Static vs Dynamic 비교

|구분|Static|Dynamic|
|---|---|---|
|lookback 방식|고정 기간 (예: 15일)|실제 변경된 파티션만|
|리소스|늦은 데이터 없어도 매번 재처리|변경된 파티션만 재처리|
|구현 복잡도|단순|복잡 (State Table, 동시 실행 보호)|
|15일 초과 데이터|버림|처리 가능|
|적합한 상황|허용 지연이 명확히 정해진 경우|지연 기간이 불규칙하거나 제한 없는 경우|

---

### 트러블 로그

Dynamic Late Data Integrator에서 `depends_on_past=True` 설정한 task가 하나 실패했는데 그걸 모르고 지나쳤다가 이후 모든 실행이 그 task에서 영원히 블로킹된 사례. Airflow UI에서 보면 해당 task가 계속 "upstream_failed" 상태로 쌓이는데, 원인 파악 없이 그냥 clear만 반복하다가 결국 state table이 꼬여버림. `depends_on_past=True` task가 실패하면 수동 개입 전까지 파이프라인 전체가 멈춘다는 것을 팀 전체가 인지하고 있어야 함. 반드시 해당 task 실패 시 즉시 알람을 달아두고, 실패 원인 해결 후 Airflow에서 해당 task만 targeted clear 하는 절차를 문서화해둘 것.
