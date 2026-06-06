# 5.1 데이터 강화

## 패턴 #23: 정적 조이너

### 5.1.1 문제

- 사용자의 등록 일자와 일상적인 활동 간의 의존성을 이해하기 쉽도록 데이터셋을 생성해 달라는 요청
- 참조 데이터를 사용자 활동에 가져와야 함

### 5.1.2 해결책

정적 조이너 패턴

- 내가 가진 중심 데이터(로그, 주문 내역 등)에 부가적인 정보(회원 정보, 상품 카테고리 등)을 결합하여 데이터를 더 풍부하게 만드는 패턴
- 배치 뿐만 아니라 스트리밍 파이프라인에도 적용 가능
- 두 데이터셋을 결합하는 데 사용할 속성 목록 필요
    - 키 조건 외에 특히 강화 데이터셋이 천천히 변화하는 차원의 형태를 구현할 때는 결합 시 시간 제약 발생 가능
1. 코드 구현 시 강화는 종종 SQL JOIN 문으로 표현되는데 최신의 데이터 처리 프레임워크와 더불어 고전적인 데이터 웨어하우스도 지원하므로 범용적
2. API 활용하여 데이터셋 강화 - HTTP 라이브러리를 사용하여 데이터 처리 계층과 외부 API간의 통신 활성화 
    
    2-1. 데이터가 파이프라인을 흘러갈 때 매번 외부 API를 직접 호출하여 최신 정보를 가져와 결합하는 방식으로 부가정보가 실시간으로 계속 변해서 무조건 가장 최신의 상태를 조회해야 할 때 사용 - 환율 등 
    
    - 실시간 API 강화: API ← 데이터셋 처리 → 처리된 데이터셋을 출력 테이블에 적재
    
    2-2API로는 테이블과 같은 데이터 스토리지 계층에서 데이터셋에 접근 가능하기 때문에 필요하다면 API로 노출된 데이터셋을 전처리 단계에서 테이블로 구체화하여 JOIN문의 일부로 사용 가능 
    
    - API가 제공하는 데이터가 자주 바뀌지 않는 경우 API를 매번 호출하면 비효율적이므로 새벽에 API 데이터를 통째로 긁어와서 DB에 테이블로 저장(구체화, Materialize), 실제 데이터 처리할때는 빠르게 SQL JOIN 사용
    - 구체화된 API 정적 데이터 강화: API ← API 동기화 → DB → 데이터셋 처리 → 처리된 데이터셋을 출력 테이블에 적재

천천히 변화하는 차원

- 시간에 민감한 데이터 강화를 사용해야 한다면 천천히 변화하는 차원 (SCD)의 형태로 구현가능
- 해결책에서 강화 데이터셋은 SCD 유형2나 유형4를 구현해야 함
    - 유형2: 일자로 추적을 관리하고 각각의 현재 값에는 종료 일자가 비어있음
    - 유형4: 두 개의 테이블에 의존하며 첫 번째 테이블은 각 엔티티의 현재 값을 저장하고, 두번째 테이블은 현재 값을 포함한 과거 값을 저장

### 5.1.3 결과

#### 1. 지연 데이터와 일관성

- 이상적인 상황에서는 이벤트가 생성되는 속도와 같은 속도로 사용자가 변화 - 실제와는 다름
- 스트리밍 파이프라인에서 지연 문제 완화할면 동적 조이너 패턴 사용 필요
- 배치 파이프라인에서는 오케스트레이션을 통해 강화 데이터셋이 존재할 때까지 기다림
    - 파일/테이블 존재 여부
    - 데이터 크기 및 용량 기준 트리거
    - 업스트림 태스크 완료(의존성 체인) 트리거

#### 2. 멱등성

- 배치 파이프라인을 백필할 때 멱등성 고려해야 함
- 시간 기반 쿼리를 수행할 수 없다면 조인하기 전에 시간 측면을 제어할 수 있도록 강화 데이터셋을 데이터 계층으로 가져와야함
    - 시간 기반 쿼리를 수행할 수 없다 = 오직 “현재 상태”만 제공 가능
    - 이 상태에서 과거 데이터를 백필하면 과거 로그에 현재의 부가 정보가 결합되어 데이터 일관성이 꺠지고 멱등성이 무너짐
    - 외부 데이터를 스냅샷 저장 등의 형태로 데이터 계층으로 가져와서 과거의 특정 시점 데이터를 직접 제어할 수 있도록 만들어야 함

### 5.1.4 예제

- SCD 유형2 예제

```sql
# 테이블 1
CREATE TABLE dedp.users ( # ...
    id TEXT NOT NULL,
    login VARCHAR(45) NOT NULL,
    start_date TIMESTAMP NOT NULL DEFAULT NOW(),
    end_date TIMESTAMP NOT NULL DEFAULT '9999-12-31'::timestamp
    PRIMARY KEY(id, start_date)
);

# 테이블 2
CREATE TABLE dedp.visits ( # ...
    visit_id CHAR(36) NOT NULL,
    event_time TIMESTAMP NOT NULL,
    PRIMARY KEY(visit_id, event_time)
);

# 유형2 조인 예제
SELECT v.visit_id, v.event_time, v.page, u.id, u.login, u.email
FROM dedp.visits v JOIN dedp.users u ON u.id = v.user_id
    AND NOW() BETWEEN start_date AND end_date;

```

- 데이터셋의 현재 상태를 얻기 위해 NOW() 사용
- 데이터 오케스트레이터가 젝ㅇ하는 실행 시간 사용 가능 - 파이프라인 실행과 관련된 불변의 속성으로 멱등성을 강제하는데 도움

- 배치 데이터셋과 스트리밍 데이터셋 조인

```python
devices: DataFrame = spark.read.format('delta').load(...)
visits: DataFrame = (spark.readStream.format('kafka').load()...

(visits.join(devices_table, [visits.device_type == devices.type,
    visits.device_version == devices.version], 'left_outer'))
```

- 정적 데이터셋과 스트리밍 잡은 수명 주기가 다르기 때문에 조인 실패나 전체 다시 쓰는 경우 빈 참조 테이블과 조인할 수 있음
- JSON, CSV 같은 원시 형식을 사용하는 경우 발생할 수 있지만 원자성과 일관성 보장을 제공하는 테이블 파일 형식에 의존한다면 방지 가능

- 원시 데이터셋을 API를 통해 노출된 데이터셋과 결합

```python
class KafkaWriterWithEnricher:
    BUFFER_THRESHOLD = 100
    # ...
    def process(self, row):
        if len(self.buffered_to_enrich) == self.BUFFER_THRESHOLD:
            self._enrich_ips()
            self._flush_records()
        else:
            self.buffered_to_enrich.append(row)

    def _enrich_ips(self):
        ips = (','.join(set(visit.ip for visit in self.buffered_to_enrich)))
        if visit.ip not in self.enriched_ips)))
        fetched_ips = requests.get(f'http://localhost:8080/geolocation/fetch?ips={ips}',
            headers={'Content-Type': 'application/json', 'Charset': 'UTF-8'})
        if fetched_ips.status_code == 200:
            mapped_ips = json.loads(fetched_ips.content)['mapped']
            self.enriched_ips.update(mapped_ips)
```

- 대량 작업의 활용 권장 - 네트워크 처리량 최적화 가능

## 패턴 #24: 동적 조이너

### 5.1.5 문제

- 각 사용자 변경 사항이 변경 데이터 캡처(CDC) 패턴에서 스트리밍 브로커에 등록 - 데이터셋이 둘 다 동적 상태

### 5.1.6 해결책

- 정적 조이너 패턴과 키의 식별, 조인 메소드의 정의를 한다는 것은 같지만 “시간 경계”에 대한 설정이 추가로 필요
    - 전용 시간 관리 전략이 없으면 두 데이터셋의 지연 시간이 서로 다르기 때문에 많은 조인이 빈 값일 위험이 있음
1. 시간 조건 정의
    - 양족 스트림에서 조인된 레코드의 버퍼에 시간 경계가 설정되면서 빠른 데이터 원천이 느린 데이터 원천과 시간적인 시맨틱을 맞출 수 있음
2. GC 워터마크 
    - 두 스트림의 이벤트를 보관하는 대신 오래된 이벤트가 각 버퍼에서 언제 사라져야 하는지를 정의
    - 레코드 하나가 매우 늦게 도착하면 조인을 잃을 수 있지만 버퍼를 관리 가능한 크기로 유지하기 위한 트레이드오프

### 5.1.7 결과

#### 1. 공간 vs 정확성 트레이드오프

- 버퍼 공간을 늘리면 효율성을 최적화할 수 있지만 자원 비용 발생
- 공간을 줄이면 스토리지 최적화도지만 지연 차이가 클 경우 일치 가능성이 낮아짐

#### 2. 지연 데이터

- 스트림 처리는 본질적으로 낮은 지연 처리 시맨틱으로 인해 파이프라인에서 지연 데이터 통합에 대한 내결함성이 약함
- 이 제한을 극복하려면 지연 데이터를 추적하고 통합해야 함

### 5.1.8 예제

# 5.2 데이터 데코레이션

## 패턴 #25: 래퍼

- 래핑은 소프트웨어 공학에서 객체에 별도의 동작이나 속성을 추가한다는 의미

### 5.2.1 문제

- 출력 스키마가 다른 다양한 데이터 수집 상황
- 다양한 필드를 추출하여 단일 위치에 넣는 잡을 작성하고 이를 통해 다운스트림 컨슈머가 쉽게 처리할 수 있게 해야함
- 처리 로직을 단순화하기 위해 계산된 값을 원래 값과 명확히 분리하되 디버깅 요구를 위해 원래 구조 유지 필요
- 문제 상황 예시

```html
웹 로그: {"user_id": 123, "ip": "1.1.1.1", "device": "PC"}

앱 로그: {"uid": 123, "ip_address": "1.1.1.1", "os": "iOS"}

오프라인 매장 태블릿 로그: {"cust_no": 123, "wifi_ip": "1.1.1.1", "store_id": "Gangnam"}
```

- 필드 이름이 (`user_id`, `uid`, `cust_no`)로 전부 다름
- 다운스트림(데이터 분석가나 다른 시스템)에서 쓰기 편하게 하려면 이를 **하나의 표준 테이블**로 합쳐야 하고, 분석을 편하게 하기 위해 IP 주소를 기반으로 대략적인 `국가(Country)` 정보도 미리 계산해서 넣어주고 싶음
- 고려 사항
1. 서로 다른 스키마를 어떻게 하나의 테이블에 집어넣을 것인가?
2. 나중에 IP 변환 로직이 잘못되었을 때 디버깅을 하려면 원본 IP 필드가 그대로 살아있어야 하는데, 형태가 다 다른 원본을 어떻게 유지할 것인가?

### 5.2.2 해결책

- 레코드 수준에 또 다른 추상화 추가: 예를 들어, 입력 방문 이벤트는 두 개의 절에 각각 해당하는 원시필드와 계산된 필드로 구성된 이벤트로 변환 가능
- 아파치 아브로, protobuf, JSON등의 형식 외에 테이블 같은 구조화된 형식에서도 지원
- 이러한 컨텍스트에서 필요한 경우 원본 테이블과 조인할 수 있는 별도의 테이블로 구현하거나 동일한 테이블 내의 추가 컬럼으로 구현 할 수 있음
    - 동일한 테이블 내 추가 컬럼(nested 구조 또는 접두사 활용)
    
    | **computed (STRUCT 타입)** | **raw (JSON/VARIANT 타입)** |
    | --- | --- |
    | `{"standard_user_id": "123", "country": "KR"}` | `{"user_id": 123, "ip": "1.1.1.1", "device": "PC"}` |
    | `{"standard_user_id": "123", "country": "KR"}` | `{"uid": 123, "ip_address": "1.1.1.1", "os": "iOS"}` |
    - 별도의 테이블로 구현(1:1 조인 관계)
    
    ```html
    # 두 테이블은 event_id로 묶여 있어 필요할 때만 SQL JOIN 문으로 디버깅
    standard_events 테이블: 공통적이고 계산된 값들만 명확한 스키마로 저장 
    (event_id, standard_user_id, country)
    
    raw_events 테이블: 각 소스별 원본 데이터를 통째로 문자열이나 텍스트로 저장 
    (event_id, raw_text_data)
    ```
    

예시 

1. 원래 레코드를 평면 구조로 저장하고 계산된 컬럼은 중첩된 속성으로 저장

| **user_id (원래)** | **ip (원래)** | **computed (중첩 컬럼)** |
| --- | --- | --- |
| 123 | 1.1.1.1 | `{"country": "KR"}` |
1. 계산된 레코드를 단일 평면 구조로 저장

| **country (계산됨)** | **raw (중첩 컬럼)** |
| --- | --- |
| KR | `{"user_id": 123, "ip": "1.1.1.1"}` |
1. 모든 컬럼을 동일한 수준의 평면 구조로 저장

| **user_id** | **ip** | **country** |
| --- | --- | --- |
| 123 | 1.1.1.1 | KR |
1. 데이터를 나중에 고유 키로 조인할 수 있는 두 개의 별개 테이블에 저장
- 1,2번 방식은 조회 시 더 빠른 비정규화 접근 방식 사용
- 정규화된 접근 방식을 사용한 3번은 조회가 느릴 수 있지만 데이터셋을 논리적으로 독립시켜야 하거나 기존 구조를 변경할 수 없을 때 더 나은 선택지 일 수 있음

### 5.2.3 결과

#### 1. 도메인 분할

- 이 패턴이 주어진 도메인의 속성을 분리하기 때문에 도메인 분할은 자연스러운 논리적 결과
- 변환된 값과 변환되지 않은 값을 명확히 구분하기 때문에 데이터 조회가 더 복잡해지며, 컨슈머는 사용자 데이터가 두 위치에 나뉘어 있다는 점을 인지해야 함

#### 2. 크기

- decorated 값은 처리된 레코드의 본질적인 부분을 형성하므로 전체 크기와 네트워크 트래픽에 영향을 미침
- 데이터 스토리지 형식이 데이터 원천 프로젝션을 지원한다면 제한 사항 완화 가능 - 해당 기능을 통해 관심있는 컬럼을 선택하고 데이터 원천에 해당 컬럼만 물리적으로 접근하도록 요청 가능 (AWS Redshift, GCP BigQuery)

### 5.2.4 예제

```python
# 파이스파크롤 메타데이터 랩핑하기
visits_w_processing_context = (visits.withColumn('processing_context', F.struct(
    F.lit(job_version).alias('job_version'), F.lit(batch_number).alias('batch_version')
)))

visits_to_save = (visits_w_processing_context.withColumn('value', F.to_json(
    F.struct(F.col('value').cast('string').alias('raw_data'),
    F.col('processing_context')))))

# SQL에서 추가적인 구조체로 랩피하기
SELECT *, NAMED_STRUCT(
    'is_connected',
    CASE WHEN context.user.connected_since IS NULL
         THEN false ELSE true END,
    'page_referral_key', CONCAT_WS('-', page, context.referral)
) AS decorated FROM input_visits

# SQL에서 원시 값 구조체로 랩핑하기 
SELECT
    CASE WHEN context.user.connected_since IS NULL
         THEN false ELSE true END AS is_connected,
    CONCAT_WS('-', page, context.referral) AS page_referral_key,
    STRUCT(visit_id, event_time, user_id, page, context) AS raw
FROM input_visits

```
