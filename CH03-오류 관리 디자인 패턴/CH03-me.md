# 3. 오류 관리 디자인 패턴

# 3.1 처리할 수 없는 레코드

## 패턴 #09: 데드 레터

### 3.1.1 문제

스트리밍 잡이 아파치 카프카 토픽에 기록된 방문 이벤트를 객체 스토어에 쓰는 상황에서 데이터 프로듀서가 처리할 수 없는 레코드를 생성하기 시작하여 잡이 실패 

- 일시적 오류: 일시적으로 발생하지만 결국 자동으로 복구
    - 데이터베이스 연결 문제 등
- 비일시적 오류: 처리할 수 없는 레코드가 대표적인 예, “독약 메시지”라고도 부름

### 3.1.2 해결책

#### 1. 데드레터(Dead-Letter) 패턴

- 스트림 처리, 배치 워크로드에서도 널리 지원
- 스트리밍은 한번에 하나의 레코드로 작동, 개별 레코드를 데드 레터 스토리지에 저장 가능
- 배치 작업은 많은 데이터를 처리하고 오류가 자주 발생하는 레코드의 하위집합을 한 번에 데드 레터 스토리지에 기록

1. 잡이 실패할 수 있는 코드 위치 식별
    - 사용자 정의 매핑 함수
    - 데이터 처리 프레임워크에 의해 완전히 관리되는 오류-안전 변환
2. 식별된 실패 가능 지점에 안전 제어 추가
    - 매핑 함수 사용: try-catch
    - 오류-안전 변환 사용: if-else
- 사후 분석 단계에서 실패를 더 잘 이해할 수 있또록 실패한 메시지를 메타데이터에 포함시켜야 할 수도 있는데 “메타데이터 데코레이터 패턴” 활용 가능
1. 목적지 결정: 성공적으로 처리된 레코드와 유형이 동일 할 수도, 다를 수도 있기 때문에 아래 사항 고려 필요
- 복원성: 데드 레터 스토리지에 대한 데드 레터 전략을 고민하지 않아도 됌
- 모니터링의 용이성: 오류가 있는 레코드를 잡이 언제 처리하는지, 특히 최근에 기록된 오류가 얼마나 많은지 궁음한데 이 두가지 지표를 분석하면 오류가 일시적인지 비일시적인지 판별 가능
- 쓰기 성능: 처리되지 않은 레코드를 별도의 위치에 쓰는 행위는 전체 잡 실행 시간에 비용발생시켜 쓰기 성능에 영향을 줄 수 있음

→ 데드레터 저장에는 가용성이 높고, 빠르며, 모니터링하기 쉬운 클라우드의 객체 스토어나 스트리밍 브로커가 적합

1. 실패한 레코드를 메인 데이터 흐름으로 수집하는 리플레이 파이프라인

### 3.1.3 결과

#### 1. 스노우볼 백필링 효과

- 리플레이 파이프라인을 실행했을 떄 수집된 레코드가 다운스트림 컨슈머에 의해 이미 처리된 파티션에 들어갈 수 있음 → 백필링 작업 필요 → 다운스트림 컨슈머도 데이터를 다시 처리해야 함
- 실패한 레코드를 리플레이 하지 않으면 백필링 실행을 트리거하지 않을 수 있지만 데드 레터 데이터를 잃을 수 있음
- 리플레이 파이프라인을 실행하는 경우 데이터셋이 완전해지지만, 다운스트림 컨슈머가 유사한 백필링 프로세스를 실행하지 않거나 실행할 수 없는 경우 데이터셋이 일관성을 잃을 수 있음

#### 2. 데드 레터 레코드 식별

- 데드 레터 레코드를 정상적인 데이터 수집 파이프라인에서 추가된 레코드와 구별을 원하는 경우 다운스트림 컨슈머에서 리플레이된 레코드를 건너뛰는 필터 조건을 구현하거나 각 레코드의 출처를 간단히 추적하는 것이 유용
    - `was_deead_lettered`라는 속성이나 `boolean` 컬럼 추가 가능
    - 더 완전한 메타데이터를 사용하여 잡 이름, 버전, 리플ㄹ에ㅣ 시간을 레코드에 주석으로 달 수 있음 - 데이터 데코레이션 패턴과 완벽히 일치하는 방법

#### 3. 순서와 일관성

- 데이터 일관성이 깨질 수 있음
- 윈도가 5분간 비활성화되면 종료되는 규칙에서 이벤트가 5분 연속으로 데드 레터 스토리지에 쌓이면, 컨슈머가 세션 종료 → 세션이 부분적이 되어 데이터 일관성 깨짐
- 순서 지정된 데이터 전달 요구 상황에서도 실패한 전달을 리플레이하면 순서의 일관성이 깨짐
    - 10:01, 10:02, 10:03 데이터에서 10:02 데이터가 DL가 되고 리플레이 파이프라인을 타게된다면 10:01, 10:03, 10;02 순으로 데이터 적재

#### 4. 오류-안전 함수

- 오류-안전 함수는 오류가 발생하는 경우 런타임 예외를 던지지 않고 NULL 값을 반환하기 때문에 코드상의 런타임 문제 위험을 크게 줄이지만 데드 레터 패턴 구현은 어려움
    - 함수마다 다를 수 있는 오류-안전 시맨틱 이해가 필요하여 데드 레터 패턴 구현은 가능하지만 어려울 수 있음

#### 5. 오류 또는 실패

- 데드 레터 패턴이 처리 잡을 유지해준다고 해도 오류를 숨기고, 결국 파이프라인을 중다나시켜야 할 치명적인 실패를 은폐할 수 있음

### 3.1.4 예제

#### 1. 스트림 처리 컨텍스트

- 아파치 플링크 예제: 처리된 레코드를 기록할 수 있는 추가 목적지인 사이드 출력(side outputs)라는 기능에 의존

```scala
# 흐름에서 벗어난 예외 데이터를 모으기 위해 invalid_visits라는 태그를 가진 OutputTag를 선언

invalid_data_output: OutputTag = OutputTag('invalid_visits', Types.STRING())
visits: DataStream = data_source.map(MapJsonToReducedVisit(invalid_data_output), Types.STRING())

# try-catch 블록 내에서 JSON 파싱 에러 등의 예외가 발생했을 때, 해당 데이터를 사이드 출력(invalid_data_output)으로 보냄
def map_rows(self, json_payload: str) -> str:
    try:
        evt = json.loads(json_payload)
        evt_time = int(datetime.datetime.fromisoformat(evt['event_time']))
        yield json.dumps({'visit_id': evt['visit_id'], 'event_time': evt_time, 'page': evt['page']})
        
    except Exception as e:
        yield self.invalid_data_output, _wrap_input_with_error(json_payload, e)

kafka_sink_valid_data: KafkaSink = ...
kafka_sink_invalid_data: KafkaSink = ...

# 최종적으로는 정상 데이터 스트림과 사이드 출력 스트림을 각각 서로 다른 카프카 토픽(KafkaSink)으로 저장하는 구조
visits.get_side_output(invalid_data_output).sink_to(kafka_sink_invalid_data)
visits.sink_to(kafka_sink_valid_data)
```

- 아파치 스파크 SQL과 델타 레이크: 오류-안전 CONCAT 데이터 변환

```python
# COCNCAT 함수를 사용할 때 값이 누락되어 null이 발생하는 상황을 감지하기 위해 데이터의 유효성 여부를 판단하는 is_valid 플래그 생성
spark_session.sql('''
    SELECT type, full_name, version, name_with_version,
        WHEN (full_name IS NOT NULL OR version IS NOT NULL)
            AND name_with_version IS NOT NULL THEN false ELSE true
        END AS is_valid
    FROM (SELECT type, full_name, version, CONCAT(full_name, version) AS name_w_version
          FROM devices_to_load)''')
          
# 캐시된 데이터셋 위에 필터를 적용해 올바르게 변환된 결과와 잘못 변환된 결과를 서로 다른 위치에 기록
devices_to_load_with_validity_flag.persist()
```

# 3.2 중복된 레코드

## 패턴 #10: 윈도 중복 제거

- 배치와 스트리밍 파이프라인에서 모두 중복제거를 적용하는 방법은 “데이터를 제한”하는 것
- 스트리밍 잡: 시간 기반 윈도로 제한 범위 설정
- 배치 잡: 현재 처리 중인 데이터셋으로 그 범위를 설정
- “자동 재시도”에 대한 주의 필요
    - 정확히 한 번 처리하기 방식은 런타임 오류가 발생하지 않을 경우에만 작동
    - 재시작된 잡 실행이 중복 제거 로직과 관계없이 이미 처리된 레코드를 재처리할 수 있기 때문

### 3.2.1 문제

- 정확히 한 번 처리 보장이 필요한 경우 데이터 프로듀서의 자동 재시도 떄문에 스트리밍 계층에 중복된 이벤트가 자주 발생

### 3.2.2. 해결책

#### 1. 윈도 중복 제거 패턴

1. 각 레코드의 고유성을 보장하는 중복 제거 속성 식별
2. 중복 제거 범위 정의 
    - 배치 잡의 경우 현재 처리 중인 데이터셋이 그 범위가 됌
    - 스트리밍 잡의 경우 정의상 무제한의 레코드 집합에서 작업 - 잡이 중복 제거 속성으로 구성되어 이미 처리된 키를 유지하는 시간 기반 윈도들을 생성해 완료된 데이터셋을 시뮬레이션
3. 코드 구현
    - 배치: `row_number()` 조건과 함께 `DISTINCT` 또는 `WINDOW` 함수 사용
    - 스트리밍: 상태 스토어를 통해 레코드가 이미 조회되었는지 확인하고 저장되는 로직이 필요할 수 있음

상태 스토어의 세가지 유형

1. 로컬
    - 상태가 오직 메모리에 존재
2. 내결함성을 갖춘 로컬
    - 상태가 주로 메모리에 존재하지만 잡은 내결함성을 이유로 원격 스토리지에 저장
    - 지속성 작업은 시간이나 일관성 측면에서 비용이 발생
    - 윈도나 마이크로배치 내에서 값을 갱신 한 후 반복마다 발생하면, 일관성은 괜찮지만 잡이 느려짐
3. 원격
    - 상태는 원격 데이터 스토어에만 존재
    - 내결함성을 기본적으로 제공해도 데이터 지연이나 파이프라인의 전체적인 비용에 부정적인 영향을 미칠 수 있음

### 3.2.3 결과

#### 1. 공간 vs 시간 트레이드오프

- 스트리밍 파이프라인에 유효
- 짧은 윈도는 중복된 데이터 일부 유실 가능, 자원에 미치는 영향은 작음
- 윈도 지속 시간을 연장하면 더 많은 고유 키를 상태 스토어에 저장 및 관리해야 하므로 더 많은 자원 필요

#### 2. 멱등성 프로듀서

- 정확하게 중복 데이터를 제거했더라도 처리된 레코드가 정확히 한 번만 전달된다는 보장은 없음(4장 참고)

### 3.2.4 예제

#### 1. 배치 파이프라인

- 아파치 스파크 `dropDuplicates` 함수로 자동 중복 처리 가능(스트리밍 데이터 원천에서도 이용 가능)

```python
dataset = (session.read.schema('...').format('json').load(f'{base_dir}/input'))
deduplicated = dataset.dropDuplicates(['type', 'full_name', 'version'])
```

- 네이티브 중복 제거가 제공되지 않는 경우 그룹화 활용: SQL의 `WINDOW` 함수 사용

```python
SELECT type, full_name, version FROM (
	SELECT type, full_name, version, 
		ROW_NUMBER() OVER (PARTITION By type, full_name, version ORDER BY 1) AS position
	FROM duplicated_devices
) WHERE posision = 1
		
```

#### 2. 스트리밍 파이프라인

- 아파치 스파크 `dropDuplicates` 함수로 자동 중복 처리 가능 - 워터마크 사용

```python
from pyspark.sql import functions as F
from pyspark.sql.types import StructType, StructField, StringType, TimestampType

# 예제 3-7: 스트리밍 데이터 준비에서 dropDuplicates를 사용한 중복 제거
# 1. 입력 데이터의 JSON 스키마 정의
event_schema = StructType([
    StructField("visit_id", StringType(), True),
    StructField("visit_time", TimestampType(), True)
])

# 2. JSON 문자열 파싱 및 구조화된 컬럼 추출
deduplicated_visits = (
    input
    .select(F.from_json("value", event_schema).alias("value_struct"), "value")
    .select("value_struct.visit_time", "value_struct.visit_id", "value")
)

# 예제 3-8: 스트리밍 데이터 만료 시 dropDuplicates를 사용한 중복 제거
# 3. 워터마크 설정 및 중복 데이터 제거 프로세스 적용
final_deduplicated_visits = (
    deduplicated_visits
    .withWatermark("visit_time", "10 minutes")
    .dropDuplicates(["visit_id", "visit_time"])
    .drop("visit_time", "visit_id")
)
```

#### 워터마크

- 늦게 도착한 데이터의 경계를 정의: 현재 워터마크 값보다 오래된 레코드가 파이프라인에 통합되지 않도록 함
- 중복 제거 컨텍스트에서 잡이 주어진 키를 기억하는 기간도 제어
- 워터마크보다 오래된 모든 기억된 엔트리는 자동으로 제거

# 3.3 지연 데이터

## 패턴 #11: 지연 데이터 탐지기

### 3.3.1 문제

실시간 이벤트 생성에서 이벤트 데이터가 지연되어 들어오는 경우

### 3.3.2 해결책

#### 1. 지연 데이터 탐지기 패턴

1. 문제가 시간과 관련되므로, 패턴은 지연 데이터를 추적하기 위한 시간 기반의 속성 정의 필요: 해당 속성은 특정 이벤트가 발생한 시점을 설명할 수 있어야 함

이벤트 시간과 처리시간

- 이벤트 시간: 특정 작업이 언제 발생했는지 (이벤트가 발생한 시간)
- 처리 시간: 데이터 파이프라인이 해당 작업과 상호 작용한 시점
1. 입력 데이터 스토어의 각 파티션에 개별적으로 적용할 데이터 지연 집계 전략 정의 필요
    - 지연 집계 전략이 단조롭게 증가해야함 즉, 추적된 이벤트 시간은 앞으로만 이동하며 되돌아 갈 수 없음 → `MAX` 함수 사용
2. 전체 진행 상황을 나타내기 위해 모든 파티션에 대한 단일 이벤트 시간을 계산할 추가 집계 전략 결정 필요
    - 가장 느린 업스트림 의존성을 따라야 할 경우 `MIN` 함수 사용: 더 많은 데이터가 제시간에 처리된다고 간주할 수 있지만 처리 로직이 이벤트 시간을 기반으로 일부 버퍼링을 수행하면, 가장 느린 파티션의 이벤트 시간을 따르기 때문에 버퍼링이 많아짐
    - 가장 빠른 업스트림 의존성을 따를 경우 `MAX` 함수 사용: 가장 느린 의존성에서 오는 레코드를 건너뛸 위험이 있지만 버퍼 크기를 줄여 스토리지 부담 완화 가능
    - `MIN` + `MAX`: 여러 파티션된 데이터 원천과 상호 작용할 때만 가능
        - 각 데이터 원천 위에 첫 번째 집계가 있고, 모든 데이터 원천을 대상으로 한 추가 집계가 존재하며 각 원천에 `MIN` 함수를 적용하고 전체에 `MAX` 함수를 적용하거나 그 반대로도 가능
3. 예상 밖의 지연을 허용하기 위해 허용된 지연 시간 속성 필요
    - `MAX(event_time) - allowed_lateness` 로 표현됌 - 이벤트를 제시간에 도착한 것으로 간주하는 최소 이벤트 시간(워터마크)를 의미
