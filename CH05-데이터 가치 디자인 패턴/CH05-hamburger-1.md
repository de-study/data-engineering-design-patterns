## 2\. 데이터 강화 (Data Enrichment)

원시 데이터는 기술적 제약으로 인해 품질이나 확장된 컨텍스트가 부족한 경우가 많다. 데이터 강화 패턴은 이러한 제한을 극복하고 데이터를 다양한 이해관계자에게 훨씬 유용하게 만들어 준다.

#### 정적 조이너 (Static Joiner) 패턴

만약 강화하려는 데이터셋의 본질이 정적(Static)이라면, 가장 직관적이고 쉬운 **정적 조이너 패턴**을 선택하면 된다.

-   **문제 상황:** 새로 진행되는 프로젝트에서 사용자의 등록 일자와 일상적인 활동 간의 의존성을 분석해 달라는 비즈니스 요청을 받았다. 하지만 안타깝게도 활동 원시 데이터셋에는 사용자 컨텍스트가 포함되어 있지 않고, 오직 정적 사용자 참조 데이터셋에서만 해당 정보를 찾을 수 있는 상황이다. 비즈니스 요구에 대응하기 위해 이 참조 데이터를 사용자 활동에 가져와 결합해야 한다.
-   **해결책:** 두 데이터셋을 식별자(예: user\_id)를 기준으로 결합(Join)한다. 이 패턴은 배치 파이프라인뿐만 아니라 스트리밍 파이프라인에도 완벽하게 적용된다.

**1\. 시간이 경과함에 따른 변화 관리: 천천히 변화하는 차원 (SCD)**

정적 조이너 패턴을 사용할 때 결합할 참조 데이터셋이 완전히 고정되어 있지 않고 아주 드물게 변한다면, 시간에 민감한 데이터 강화를 위해 **천천히 변화하는 차원(SCD, Slowly Changing Dimensions)** 전략을 사용해야 한다.

**SCD 유형 2 (SCD Type 2)**

-   **메커니즘:** 데이터의 변경 이력을 새로운 레코드로 추가하되, From\_date, To\_date, 그리고 현재 최신 데이터 여부를 나타내는 플래그(is\_current) 컬럼을 두어 유효 기간을 관리한다.
-   **특징:** 각각의 현재 값에는 종료 일자(To\_date)가 NULL로 비어 있으며, 유효 일자로 기록의 추적을 관리한다.

**코드 예제: SCD 유형 2 조인 및 파이스파크 구현**

PostgreSQL 환경에서의 SCD 유형 2 기반 조인

SCD 유형 2 구조에서는 단순히 u.id = v.user\_id 조건만으로 조인하면 안 되며, 이벤트 발생 시점(event\_time)이 해당 마스터 데이터의 유효 기간 내에 있었는지 확인하는 시간 조건(BETWEEN start\_date AND end\_date)이 반드시 포함되어야 한다.

```
CREATE TABLE dedp.users (
    id TEXT NOT NULL,
    login VARCHAR(45) NOT NULL,
    start_date TIMESTAMP NOT NULL DEFAULT NOW(),
    end_date TIMESTAMP NOT NULL DEFAULT '9999-12-31'::timestamp,
    PRIMARY KEY(id, start_date)
);

CREATE TABLE dedp.visits (
    visit_id CHAR(36) NOT NULL,
    event_time TIMESTAMP NOT NULL,
    PRIMARY KEY(visit_id, event_time)
);
```

```
SELECT v.visit_id, v.event_time, v.page, u.id, u.login, u.email
FROM dedp.visits v 
JOIN dedp.users u ON u.id = v.user_id
-- 데이터셋의 현재 상태를 얻기 위해 NOW()를 사용하거나 오케스트레이터의 실행 시간을 동적으로 바인딩하여 멱등성을 강제한다.
AND NOW() BETWEEN start_date AND end_date;
```

PySpark에서의 스트림-배치 조인 (Delta Lake + Kafka)  
아파치 스파크(Apache Spark)에서는 배치 데이터셋과 스트리밍 데이터셋을 결합할 때도 일반 배치 조인과 동일한 구조의 API를 사용할 수 있어 복잡하지 않다.

```
# 예제 5-3: 파이스파크에서 스트림-배치 조인
# 1. 배치 참조 데이터셋 로드 (Delta Lake 형식)
devices = spark.read.format('delta').load(...)

# 2. 스트리밍 방문 데이터셋 로드 (Kafka 소스)
visits = spark.readStream.format('kafka').load(...)

# 3. 레프트 아우터 조인 실행 (일치하는 레코드가 없어도 스트림 유지를 위해 Left Outer 사용)
enriched_stream = visits.join(
    devices,
    on=(visits.device_type == devices.type) & (visits.device_version == devices.version),
    how='left_out'
)
```

PySpark 작성자(Writer) 내 버퍼링을 통한 API 강화 최적화  
네트워크 I/O 비용을 줄이기 위해 행(Row) 단위로 실시간 API를 호출하지 않고, 지정된 임계치(BUFFER\_THRESHOLD)만큼 데이터를 모았다가 배치로 외부 IP 매핑 서비스를 벌크 호출하는 구현 방식이다.

```
# 예제 5-4: 파이스파크 작성자와 데이터 강화
class KafkaWriterWithEnricher:
    BUFFER_THRESHOLD = 100
    
    def __init__(self):
        self.buffered_to_enrich = []

    def process(self, self_or_row, row=None):
        # 파이스파크 가동 환경에 따라 인자 구조 처리 유의
        current_row = row if row is not None else self_or_row
        
        if len(self.buffered_to_enrich) == self.BUFFER_THRESHOLD:
            self._enrich_ips()     # 버퍼에 쌓인 IP들을 한 번에 대량 요청으로 강화
            self._flush_records()  # 출력 테이블에 적재 후 버퍼 초기화
        else:
            self.buffered_to_enrich.append(current_row)

    def _enrich_ips(self):
        # 고비용 네트워크 처리량을 최적화하기 위해 set()을 통해 중복을 제거한 뒤 벌크 요청 생성
        ips = ','.join(set(visit.ip for visit in self.buffered_to_enrich))
        # 외부 IP 매핑 서비스 API 호출 및 결과 매핑 로직...
```

**SCD 유형 4 (SCD Type 4)**

-   **메커니즘:** 두 개의 테이블에 의존하는 방식이다.
-   **특징:** 첫 번째 테이블에는 각 엔티티의 **현재 값만 저장**하고, 두 번째 테이블에는 현재 값을 포함한 **과거의 모든 변경 내역(이력)을 분리하여 저장**한다.

**2\. API 기반 강화 vs 구체화된 정적 데이터 강화**

SQL 조인과 같은 선언적 방식 외에도 프로그래밍 API를 활용해 데이터셋을 강화할 수 있다. 이때 아래와 같이 크게 두 가지 접근 방식이 존재한다.

```
[실시간 API 강화]
데이터셋 처리 ──> 외부 API 호출 (실시간 통신) ──> 처리된 데이터셋을 출력 테이블에 적재

[구체화된 API 정적 데이터 강화]
외부 API ──> API 동기화 ──> 내부 데이터 스토어(DB) ──> 데이터셋 처리(JOIN) ──> 출력 테이블 적재
```

1) 실시간 API 강화 (HTTP 라이브러리 활용)

-   **장점:** 가장 손쉽게 구현할 수 있으며 외부 API의 최신 상태를 즉시 반영한다.
-   **단점 및 최적화:** 네트워크 처리량으로 인해 고비용 작업이 될 수 있다. 이를 최적화하기 위해 한 번에 여러 항목의 정보를 요청할 수 있는 **대량 작업(Bulk/Batch Request 또는 버퍼링)을 활용**하는 것이 권장된다.

2) 구체화된 API 정적 데이터 강화 (테이블 구체화 방식)

-   **장점:** 외부 API 데이터를 전처리 단계에서 내부 테이블로 미리 구체화(Materialize)한 뒤 SQL JOIN 문으로 처리한다. API 서버의 장애에 영향을 받지 않고 데이터 스토리지 계층 내에서 효율적으로 접근할 수 있다.
-   **멱등성 보장:** 파이프라인을 재플레이(Replay)할 때마다 항상 동일한 데이터셋으로 처리되도록 보장하려면 이처럼 API를 구체화하고 SCD 형식을 함께 활용하는 해결책을 선택해야 한다.

**3\. 구현 시 주의해야 할 고려사항 (결과 및 한계)**

정적 조이너 패턴과 데이터 강화는 간단해 보이지만, 실제 운영 환경에서는 몇 가지 주의할 점이 있다.

**지연 데이터와 일관성 (Latency & Consistency)**

-   **이상과 현실:** 이상적인 상황에서는 방문 데이터셋과 사용자 데이터셋이 동일한 속도로 생성 및 변화하겠지만, 실제로는 결합 대상인 사용자 프로필 변경 등의 이벤트 처리가 지연될 수 있다.
-   **스트리밍 파이프라인의 한계:** 스트리밍 잡과 정적 데이터셋은 수명 주기가 다르다. 정적 데이터셋이 먼저 갱신되기를 기다리지 않고 스트리밍 데이터가 먼저 유입되면 조인 실패를 초래하거나 빈 참조 테이블과 결합하는 문제가 발생할 수 있다.
-   **해결책:** 스트리밍 파이프라인에서 지연 문제를 완화하려면 강화 데이터셋을 동적인 것으로 간주하고 결합하는 **동적 조이너(Dynamic Joiner) 패턴**을 사용해야 한다. 반면 배치 파이프라인에서는 오케스트레이션을 통해 강화 데이터셋이 준비될 때까지 기다리는 **준비 마커(Ready Marker) 패턴** 등을 활용하면 되므로 상대적으로 해결이 간단하다.

**원시 파일 형식과 일관성 보장**

-   정적 데이터셋이 JSON이나 CSV 같은 원시 파일 형식으로 쓰이면 동시성 문제로 빈 테이블을 처리하게 될 위험이 있다.
-   이를 방지하기 위해 원자성(Atomicity)과 일관성(Consistency) 보장을 제공하는 **델타 레이크(Delta Lake)나 아파치 아이스버그** 같은 현대적인 테이블 파일 형식을 사용하는 것이 안전하다. 컨수머가 커밋되지 않은 파일을 조회하지 않으므로 데이터 안정성이 확보된다.

#### 동적 조이너 (Dynamic Joiner) 패턴

정적 조이너 패턴은 정적인 특징 때문에 두 개의 스트리밍 데이터셋을 결합할 때 최적의 선택지가 되지 못한다. 스트리밍은 데이터셋이 가능한 한 빠르게 처리되고 지속적으로 움직이는 반면, 정적 작업은 더 천천히 변화하는 데이터에서 동작하기 때문이다.

-   **문제 상황 (Problem):** 사용자(Users)와 방문(Visits) 사용 사례를 위해 정적 조이너를 구현했으나, 매주 수천 명의 신규 사용자가 유입되고 프로필 변경이 잦아지면서 실시간 데이터 동기화가 깨지기 시작한다. 결과적으로 강화된 데이터셋의 효과가 떨어지고 다운스트림 컨수머에 문제를 일으키므로, 스트리밍 브로커에 등록된 양쪽 이벤트를 실시간으로 결합할 더 나은 방법이 필요하다.
-   **해결책 (Solution):** 결합하려는 두 데이터셋이 모두 동적 상태일 때는 식별자 키뿐만 아니라 '시간 경계(Time Boundaries)'를 조인 메소드에 필수로 정의하는 **동적 조이너 패턴**을 고려해야 한다.

**동적 조인의 핵심 메커니즘: 버퍼와 가비지 컬렉션 (GC)**

두 스트리밍 데이터셋은 지연 시간이 서로 다르다. 강화 데이터셋이 원시 데이터셋보다 늦게 도착하거나 그 반대일 수 있으므로, 동적 조이너는 주로 **시간 조건이 추가된 상태**로 완료된다.

-   **시간적 시맨틱 맞추기:** 시간 조건을 정의하면 양쪽 스트림에서 조인될 레코드의 버퍼에 시간 경계가 설정된다. 버퍼는 조인이 일어날 수 있도록 추가 시간을 제공하며, 두 데이터 원천 간에 허용되는 지연 차이를 메워준다.
-   **GC 워터마크 (Watermark):** 이벤트를 영원히 보반할 수는 없으므로, 매우 오래된 이벤트를 각 버퍼에서 언제 삭제할지 정의해야 한다. 레코드가 너무 늦게 도착하면 조인을 잃게 되지만, 버퍼를 관리 가능한 크기로 유지하기 위한 필수적인 트레이드오프다.

```
[스트림 A (빠름)] ──> [버퍼에 Key 유지 및 일정 시간 대기] ──┐
                                                           ├─> [시간 내 일치 시 조인 성공]
[스트림 B (느림)] ──> [뒤늦게 따라잡아 일치 확인] ──────────┘
*지연 시간 동안 매칭 실패 시 GC 워터마크에 의해 버퍼에서 제거됨*
```

**고려사항 및 한계**

-   **공간 vs 정확성 트레이드오프:** 버퍼 공간을 늘리면 지연 데이터와의 일치 가능성(정확성)은 높아지지만 하드웨어 비용이 증가한다. 반대로 공간을 줄이면 스토리지는 최적화되지만 지연 차이가 클 때 조인을 놓칠 위험이 커진다.
-   **지연 데이터 통합의 한계:** 스트림 처리는 네트워크 장애 등으로 인한 하위 집합의 지연에 내결함성이 약하다. 일시적 연결 문제로 GC 워터마크가 이동해 버려 버퍼 유효성이 끝나면 지연 이벤트는 무시될 수 있으므로, 조인 결과를 100% 보장하려면 3장의 지연 데이터 추적 패턴을 병행해야 한다.

동적 조인 구현 코드 예제

Spark Structured Streaming에서의 시간 기반 조인  
아파치 스파크 구조적 스트리밍에서는 양쪽 스트림에 withWatermark를 정의하고, 조인 조건문 내에 BETWEEN을 활용한 시간 범위(예: 2분 이내)를 명시하여 구현한다.

```
# 예제 5-5: 스파크 구조적 스트리밍 시간 기반 조인
visits_from_kafka = (visits_data_stream
                     .withWatermark('event_time', '10 minutes'))

ads_from_kafka = (ads_data_stream
                  .withWatermark('display_time', '10 minutes'))

# 사용자가 페이지 방문 후 최대 2분 이내에 광고를 표시할 수 있다는 비즈니스 규칙 적용
visits_with_ads = visits_from_kafka.join(
    ads_from_kafka,
    expr("""
        page = visit_page AND
        display_time BETWEEN event_time AND event_time + INTERVAL 2 minutes
    """),
    'left_outer'
)
```

Apache Flink에서의 템포럴 테이블 조인 (Temporal Table Join)  
아파치 플링크는 스트리밍 데이터 원천을 결합하는 보다 관리된 방법으로 템포럴 테이블 조인을 제공한다. 파이썬 API에서는 지원되지 않아 자바(Java) 코드로 작성된다.

```
// 예제 5-6: 플링크 템포럴 테이블 조인 (주요 흐름 요약)
tableEnv.createTemporaryTable("visits_tmp_table", TableDescriptor.forConnector("kafka")
    .schema(Schema.newBuilder().fromColumns(SchemaBuilders.forVisits())
    .watermark("event_time", "event_time - INTERVAL '5' MINUTES").build()).option("topic", "visits"));

tableEnv.createTemporaryTable("ads_tmp_table", TableDescriptor.forConnector("kafka")
    .schema(Schema.newBuilder().fromColumns(SchemaBuilders.forAds())
    .watermark("update_time", "update_time - INTERVAL '5' MINUTES").build()).option("topic", "ads"));

// TemporalTableFunction 초기화 및 시스템 함수 등록 (각 페이지에 대한 최신의 광고를 매핑)
TemporalTableFunction adsLookupFunction = adsTable.createTemporalTableFunction($("update_time"), $("ad_page"));
tableEnv.createTemporarySystemFunction("adsLookupFunction", adsLookupFunction);

// joinLateral 구조를 사용하여 최신 상태 테이블과 결합
Table joinResult = visitsTable.joinLateral(call("adsLookupFunction", $("event_time")), $("ad_page").isEqual($("page"))).select($("*"));
```

## 3\. 데이터 데코레이션 (Data Decoration)

데이터셋이 강화 패턴을 통해 비즈니스 가치를 높였다면, 다음 질문은 '이것으로 충분한가'이다. 최종 형태를 완전히 갖추지 못한 원시 상태이거나 가공 필드가 뒤섞인 비정형 상태라면 데이터 레이어를 격상시킬 추가 준비 작업이 필요하다.

#### 래퍼 (Wrapper) 패턴

래핑(Wrapping)은 소프트웨어 공학에서 객체에 별도의 동작이나 속성을 추가하는 것을 뜻하며, 데이터 영역에서는 **레코드의 원본 부분과 변환된(계산된) 부분을 별개로 분리**하는 데 쓰인다.

-   **문제 상황 (Problem):** 다양한 데이터 제공업체에서 발재(생성)되어 출력 스키마가 서로 다른 방문 데이터를 처리해야 한다. 단일 위치에 통합 스키마를 채워 다운스트림 컨수머가 쉽게 쓰도록 만들어야 하지만, 동시에 디버깅 요구를 위해 **초기 원본 구조와 원래 값은 유실 없이 그대로 유지**되어야 한다.
-   **해결책 (Solution):** 원본 레코드를 훼손하지 않기 위해 고수준의 **envelope(봉투)** 구조체로 원래 값을 감싸고, 실행 컨텍스트(처리 시간, 잡 버전 등)나 계산된 속성들을 별도 필드로 참조하는 **래퍼 패턴**을 사용한다.

정형 데이터를 위한 래퍼 패턴 구현 유형 4가지

정형 데이터 컨텍스트에서는 동일 테이블 내의 추가 컬럼이나 중첩 구조를 통해 4가지 형태의 래퍼를 구현할 수 있다.

| 구현 유형 | 구조적 특징 | 비고 / 스키마 관리 관점 |
| --- | --- | --- |
| 구현 1 | 원래 레코드는 평면(Flat) 구조로 저장하고, 계산된 컬럼은 **중합(Nested) 속성**으로 분리 | 조회 시 더 빠른 비정규화 접근 방식 |
| 구현 2 | 원본 필드들을 평면 구조로 두고, **계산된 레코드들만 중첩 struct(...)** 구조체로 래핑 | 구현 1의 반대 작업 |
| 구현 3 | 원본과 계산된 컬럼 모두를 동일한 레벨의 **단일 평면 구조**로 결합 | 조회가 느려질 수 있으나 기존 구조 변경이 불가능할 때 유용 |
| 구현 4 | 원본 데이터와 계산된 데이터를 고유 키로 조인 가능한 **두 개의 별개 테이블로 분리**하여 저장 | 데이터셋을 논리적으로 완전히 독립시켜야 할 때 선택 |

**결과 및 한계 (도메인 분할과 크기)**

-   **도메인 분할:** 원본과 계산된 값이 명확히 구분되어 디버깅에 유리하지만, 데이터 조회가 복잡해져 두 위치에 나뉘어 있음을 컨수머가 인지해야 한다. 이 혼란을 막기 위해 래핑 데이터는 실버 계층(전처리 단계)까지만 노출하고, 최종 사용자용 골드 데이터셋에는 노출하지 않는 것이 좋다.
-   **스토리지 크기와 트래픽:** 계산된 필드가 추가되므로 레코드 전체 크기와 네트워크 트래픽에 영향을 준다. 따라서 필요한 컬럼만 물리적으로 접근할 수 있는 프로젝션을 지원하는 컬럼형 데이터 스토리지(AWS Redshift, GCP BigQuery 등)를 인프라로 활용하여 제한 사항을 완화하는 관행이 일반적이다.

데이터 데코레이션 구현 코드 예제

PySpark에서의 메타데이터 및 실행 컨텍스트 래핑  
잡 버전, 배치 번호 같은 실행 컨텍스트를 구조체(F.struct)로 묶고 원본 데이터와 함께 최종적으로 하나의 JSON 도큐먼트로 래핑하여 출력 DB에 적재하는 전형적인 구현 예시다.

```
# 예제 5-7: 파이스파크로 메타데이터 래핑하기
# 1. 실행 컨텍스트 정보를 구조체 컬럼으로 추가
visits_w_processing_context = visits.withColumn(
    'processing_context', 
    F.struct(F.lit(job_version).alias('job_version'), F.lit(batch_number).alias('batch_version'))
)

# 2. 초기 값(raw_data)과 기술적 컨텍스트를 품은 고수준 envelope을 생성하고 JSON으로 직렬화
visits_to_save = visits_w_processing_context.withColumn(
    'value', 
    F.to_json(F.struct(
        F.col('value').cast('string').alias('raw_data'),
        F.col('processing_context')
    ))
)
```

Spark SQL 환경에서의 구조체(Struct) 데코레이션  
스파크 SQL에서는 NAMED\_STRUCT나 STRUCT 함수를 사용해 계산된 메타데이터 컬럼을 격상시키거나 원시 값을 하위 구조체로 격하시켜 결합 분리할 수 있다.

```
-- 예제 5-8: NAMED_STRUCT를 이용해 계산된 컬럼들을 'decorated' 구조체로 모으기
SELECT *, 
       NAMED_STRUCT(
           'is_connected', CASE WHEN context.user.connected_since IS NULL THEN false ELSE true END,
           'page_referral_key', CONCAT_WS('-', page, context.referral)
       ) AS decorated 
FROM input_visits;
```

```
-- 예제 5-9: 계산된 값은 상위 컬럼으로 올리고, 원시 값 전체를 'raw' 구조체 안으로 격하하기
SELECT 
    CASE WHEN context.user.connected_since IS NULL THEN false ELSE true END AS is_connected,
    CONCAT_WS('-', page, context.referral) AS page_referral_key,
    STRUCT(visit_id, event_time, user_id, page, context) AS raw
FROM input_visits;
```

주의: 예제 5-9처럼 래핑할 때 원시 값과 계산된 값을 완벽히 구조적으로 구분해주지 않으면 최종 사용자가 두 값의 출처를 오인할 수 있으므로 설계 시 유의해야 한다.
