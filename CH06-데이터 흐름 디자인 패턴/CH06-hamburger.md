실시간 데이터 처리를 하다 보면 **'세션(Session)'** 단위로 데이터를 묶어서 분석해야 할 때가 많다.

예를 들어, 한 사용자가 웹사이트에 들어와서 이탈하기 전까지 어떤 행동을 순서대로 했는지 분석하는 '유저 세션 분석'이 대표적이다.

이를 효율적으로 해결하기 위한 디자인 패턴인 **상태 저장 세션화 처리기(Stateful Sessionizer)** 패턴에 대해 알아보자!

## 1\. 상태 저장 세션화 처리기(Stateful Sessionizer)

**1-2. 배경: 기존 배치(Batch) 방식의 한계와 문제점**

기존의 배치 파이프라인이나 단순한 증분 세션화 방식은 데이터를 시간 기반(예: 1시간 단위 파티션)으로 나누어 처리한다. 하지만 이 방식은 다음과 같은 치명적인 문제가 존재한다.

-   **지연 시간(Latency) 문제:** 비즈니스 요구사항은 "몇 초 내로 유저의 방문 정보를 실시간 확인"하는 것인데, 배치 방식은 구조적으로 몇 십 분에서 몇 시간의 지연이 발생할 수밖에 없다.
-   **데이터 최신성 저하:** 데이터의 최신성이 중요한 상황에서 기존의 증분 세션화 처리기 방식은 유용하지 않다.

이를 해결하기 위해서는 **스트림 처리 계층 위에 '상태 저장 세션화 처리기(Stateful Sessionizer)' 패턴**을 도입해야 한다.

**1-3. 해결책: 상태 저장 세션화 처리기**

이 패턴의 핵심은 단순한 실시간 스트리밍을 넘어, **'상태 스토어(State Store)'라는 추가 컴포넌트를 활용해 진행 중인 모든 세션을 메모리 및 내결함성 스토리지에 유지하며 처리**한다는 점이다.

작동 워크플로우 (3단계)

1.  **세션 확인 및 재개:** 새로운 레코드가 들어오면, 기존에 대기 중인 세션이 상태 스토어에 있는지 확인하고 있다면 이를 재개(Resume)한다.
2.  **비즈니스 로직 결합:** 새로 들어온 레코드를 기존 세션 상태와 결합한다. (예: 유저가 방문한 페이지 목록을 시간 순서대로 누적 저장)
3.  **세션 완료 및 출력:** 세션이 완료되었거나(타임아웃 등), 부분 세션을 컨슈머에게 즉시 제공해야 하는 경우 대기 중인 세션 기록을 최종 출력으로 변환해 내보낸다. 완료되지 않은 새로운 상태는 다시 상태 스토어에 기록된다.

**1-4. 세션을 정의하는 두 가지 추상화 방식**

실시간 스트리밍 프레임워크(Apache Spark, Apache Flink 등)에서 세션을 관리할 때 크게 두 가지 방식을 사용할 수 있다.

1\. 세션 윈도 (Session Window)

-   **개념:** 각 세션 키(유저 ID 등)에 대해 동적으로 생성되는 윈도이다.
-   **간격 지속 시간(Gap Duration):** 동일한 세션 키를 가진 두 이벤트 사이의 최대 비활성 기간(예: 20분)을 정의한다. 데이터 간의 간격이 이 값보다 크면 새로운 세션 윈도가 생성된다.

2\. 임의의 상태 저장 처리 (Arbitrary Stateful Processing)

-   세션 윈도보다 훨씬 높은 유연성을 제공한다. 사용자가 간격 지속 시간 로직을 직접 정의하거나 세션마다 다른 동적 작업을 수행할 수 있다. 스파크 구조적 스트리밍의 applyInPandasWithState나 플링크의 Keyed Process Function 등이 이에 해당한다.

**1-5. 구현 시 주의해야 할 결과 및 트레이드오프 (Trade-off)**

상태 저장 세션화는 강력하지만, 아키텍처 관점에서 아래 사항들을 반드시 고려해야 한다.

-   **최소 한 번의 처리 (At-least-once)와 체크포인팅:** 상태 스토어 저장이 매번 발생하지 않고 체크포인팅(Checkpointing)이라는 쓰기 과정을 통해 불규칙하게 발생한다. 잡(Job)이 중단되었다가 재시작되면 마지막 성공 체크포인트부터 다시 시작하므로 중복 처리가 발생할 수 있다. 따라서 세션 키 생성 시 멱등성(Idempotency)을 고려해야 하며, 실시간 시간(Runtime Clock)처럼 재시작 시 변할 수 있는 부작용(Side-effect) 연산은 피하는 것이 좋다.
-   **클러스터 확장(Scaling) 비용:** 컴퓨팅 용량을 변경하면 상태를 재조정(State Rescaling)해야 하므로 상태 비저장 잡에 비해 비용이 많이 든다.
-   **비활성 기간 길이와 만료 전략:** 완료된 세션을 정리하기 위해 만료 시간을 관리해야 한다. 이때 '처리 시간(Processing Time)' 기준 만료는 지연이 발생할 경우 세션이 너무 일찍 만료될 위험이 있으므로, 가급적 '이벤트 시간(Event Time)'과 워터마크(Watermark)를 조합해 만료 전략을 추론하는 것이 안전하다.

실시간 데이터 파이프라인을 구축할 때 데이터의 최신성만큼이나 중요한 속성이 바로 '순서(Order)'다.

예를 들어, 금융 거래 데이터나 차량 위치 추적 시스템 등에서는 이벤트가 발생한 순서대로 다운스트림 컨슈머에게 전달되는 것이 매우 중요하다. 순서가 뒤바뀌면 건물을 뛰어넘는 자동차 추적 결과가 나오거나 데이터 정합성이 깨지는 대참사가 발생할 수 있기 때문이다.

하지만 분산 환경에서 대규모 데이터를 실시간으로 전송할 때 순서를 보장하기란 쉽지 않다. 오늘은 실시간 데이터 정렬을 보장하기 위한 디자인 패턴인 '**빈 팩 정렬기(Bin Pack Sorter)'**와 **'선입 선출 정렬기(FIFO Orderer)'** 패턴에 대해 알아보자!

## 2\. 빈 팩 정렬기 (Bin Pack Sorter) 패턴

대규모 데이터 전송 시 겪는 가장 큰 악몽 중 하나는 바로 '부분 커밋(Partial Commits)'이다.

**2-1. 문제점: 부분 커밋이 순서를 깨뜨리는 이유**

네트워크 통신을 최적화하기 위해 많은 대량 쓰기(Bulk/Batch) API(예: AWS DynamoDB의 BatchWriteItem, Amazon Kinesis의 PutRecords)를 사용한다. 하지만 이 API들은 요청 전체가 성공하거나 실패하는 것이 아니라, 일부 레코드만 성공하는 '부분 커밋 시맨틱'을 가진다.

예를 들어 10:00, 10:10, 10:20 타임스탬프를 가진 레코드를 한 번에 보냈는데 중간에 있는 10:10 데이터만 전송에 실패했다고 가정해 보자. 시스템은 실패한 10:10 데이터를 재시도 메커니즘을 통해 나중에 다시 보낼 것이다. 결과적으로 컨슈머는 10:00 -> 10:20 -> 10:10 순서로 데이터를 받게 되어 순서가 완전히 뒤섞이게 된다.

**2-2. 해결책: 빈 팩 정렬기의 작동 방식**

이 패턴은 데이터를 무작정 보내는 대신, **그룹핑 키와 이벤트 시간을 기준으로 데이터를 먼저 정렬한 뒤 '독립된 하위 집합(전달 빈, Delivery Bin)'으로 패킹하여 전송**하는 방식이다.

1.  **로컬 정렬:** 아파치 스파크 등에서 파티션 내 로컬 정렬 메커니즘(sortWithinPartitions)을 활용해 그룹핑 키와 시간 순으로 레코드를 정렬한다.
2.  **전달 빈(Bin) 생성:** 정렬된 레코드를 순회하며, 동일한 그룹핑 키를 공유하는 엔티티의 이벤트들을 하나의 '빈(Bin)'으로 그룹화한다. 빈 하나에는 단 하나의 그룹핑 키만 존재하도록 제한한다.
3.  **순차적 플러싱:** 생성된 빈을 다운스트림(예: Amazon Kinesis)으로 순차적으로 방출한다. 특정 빈이 최종 출력으로 완전히 기록되기 전까지는 다음 빈을 전송하지 않는다.

이 방식을 사용하면 부분 커밋이 발생하더라도 동일한 키 내에서의 순서는 엄격하게 보장할 수 있으며, 네트워크 오버헤드와 비용을 최적화할 수 있다.

예시

```
# 1. 파티션 내에서 visit_id와 event_time 기준으로 로컬 정렬 후 개별 파티션 처리
events.sortWithinPartitions([F.col('visit_id'), F.col('event_time')]) \
      .foreachPartition(lambda rows: write_records_to_kinesis(rows))

# 2. 파티션 내에서 동일 키별로 전용 빈(Bin)에 나누어 담아 순서 보장
def write_records_to_kinesis(visits_rows):
    producer = boto3.client('kinesis')
    delivery_groups = []
    last_visit_id = None
    groups_index = 0
    
    for visit in visits_rows:
        if visit.visit_id != last_visit_id:
            last_visit_id = visit.visit_id
            groups_index = 0  # 키가 바뀌면 빈 위치를 0으로 재설정
            
        if len(delivery_groups) <= groups_index:
            delivery_groups.append([])
        delivery_groups[groups_index].append(visit)
        groups_index += 1
    # 이후 정렬된 그룹별로 Kinesis 전송 로직이 이어짐
```

**2-3. 트레이드오프 (Trade-off)**

-   **장점:** 네트워크 대량 요청(Bulk API)의 이점을 누리면서도 부분 커밋으로 인한 순서 뒤바뀜 문제를 완벽히 방어할 수 있다.
-   **단점:** 사용자 정의 정렬 및 빈 생성 로직이 필요하므로 일반적인 정렬보다 구현 복잡도가 높다. 또한, 파이프라인 전체가 실패하여 재시도될 경우 전체적인 순서가 깨질 수 있는 위험이 여전히 존재한다.

## 3\. 선입 선출 정렬기 (FIFO Orderer) 패턴

만약 처리량 최적화(버퍼링 및 대량 요청)에 대한 제약이 조금 더 완화된 환경이라면, 훨씬 더 직관적이고 단순한 대안인 **선입 선출 정렬기(FIFO Orderer)** 패턴을 고려할 수 있다.

**3-1. 개념 및 작동 방식**

이 패턴의 핵심은 "한 번에 하나의 레코드만 전달하고, 해당 레코드에 대한 전송 확인 응답(ACK)을 성공적으로 받은 후에만 다음 레코드를 진행하는 것"이다. 동시성 수준을 1로 제한하는 엄격한 동기식(Synchronous) 전송 방식이다.

가장 오래된 레코드부터 차례대로 밀어내기 때문에 복잡한 정렬 알고리즘이 필요 없다. 아파치 카프카의 프로듀서 전송 후 flush()를 동기적으로 호출하는 형태가 이에 해당한다.

**3-2. 주의: '진행 중인 요청(In-flight Requests)'의 함정**

처리를 더 빠르게 하려고 응답을 기다리지 않고 다음 요청을 연속해서 보내는(In-flight) 대량 API를 구현하면, 두 번째 요청은 성공하고 첫 번째 요청이 실패해 재시도되는 순간 FIFO 순서 시맨틱이 파괴된다.

이를 해결하기 위해 각 메시지 브로커 및 스트리밍 플랫폼은 다음과 같은 튜닝 설정을 제공한다.

-   **Apache Kafka:** 성능 저하를 방지하면서 순서를 보장하기 위해 max.in.flight.requests.per.connection=1로 설정하거나, 최대 5개의 동시 요청을 수용하면서 순서를 보장하는 **멱등성 프로듀서(enable.idempotency=True)** 기능을 활성화해야 한다.
-   **Amazon Kinesis:** 대량 API를 사용하기 어려운 구조이므로, 올바른 순서 보장을 위해 SequenceNumberForOrdering 속성을 사용하여 이전 레코드의 시퀀스 번호를 다음 레코드 전송 시 매핑해 주어야 한다.
-   **GCP Pub/Sub:** 순서 보장 메커니즘이 내장되어 있지만, 단일 리전에 작성하는 경우에만 작동하며 프로듀서 측에서 ordering\_key 속성을 명시적으로 설정해 주어야 그룹핑 키별 FIFO가 보장된다.

카프카 멱등성 프로듀서 설정 예시

```
# 최대 5개의 동시 요청을 허용하면서도 순서를 보장하는 안전한 FIFO 설정
producer = Producer({
    'max.in.flight.requests.per.connection': 5,
    'enable.idempotency': True,
    'queue.buffering.max.ms': 2000
})
producer.produce(...)
```

**3-3. 트레이드오프 (Trade-off)**

-   **장점:** 개념과 구현이 매우 단순하며 직관적이다.
-   **단점:** 레코드마다 확인 응답을 기다리거나 동시성을 극도로 제한해야 하므로 **I/O 오버헤드와 지연 시간(Latency)이 크게 증가**한다. 대규모 트래픽을 처리해야 하는 대시보드나 파이프라인에서는 성능 병목의 원인이 된다. 또한, FIFO가 '정확히 한 번 처리(Exactly-once)'를 의미하는 것은 아니므로 재시작 시 중복이 발생할 수 있어 다운스트림의 멱등성 패턴 결합이 필수적이다.

## 4\. 로컬 시퀀서

앞서 살펴본 선입 선출 정렬기(FIFO Orderer)는 구현이 간단하지만, 메시지마다 건건이 ACK를 기다려야 하므로 처리량(Throughput)이 심각하게 떨어지는 고질적인 문제가 있었다.

이를 해결하기 위해 분산 메시지 브로커(Kafka 등)의 **키 기반 파티셔닝**을 활용해 비동기로 데이터를 대량 푸시하면서도, 컨슈머 단에서 미세하게 뒤틀릴 수 있는 순서와 중복을 최종적으로 완벽하게 잡아주는 것이 로컬 시퀀서(Local Sequencer)다.

**4-1. 왜 파티셔닝만으로는 부족할까? (로컬 시퀀서의 필요성)**

카프카나 키네시스 같은 브로커가 특정 키를 가진 데이터를 언제나 동일한 파티션에 할당하고, 단일 컨슈머 스레드가 이를 순서대로 읽어온다 하더라도 다음의 분산 환경 이슈들로 인해 순서가 깨지거나 데이터가 중복될 수 있다.

-   **네트워크 재시도(Retry):** 업스트림 프로듀서나 브로커 간의 일시적인 통신 장애로 특정 메시지가 재전송되면서 순서가 뒤집힐 수 있다.
-   **소비 지연 및 복구:** 컨슈머 잡이 일시적으로 중단되었다가 체크포인트나 마지막 오프셋부터 데이터를 다시 읽어올 때, 이미 처리된 데이터가 중복으로 대량 유입될 수 있다.

따라서 컨슈머 내부 메모리에 **현재 처리해야 할 올바른 순서 번호(Sequence Number)를 관리하는 '로컬 시퀀서'를 두고, 들어오는 데이터를 검증 및 정렬하는 과정**이 필수적이다.

**4-2. 로컬 시퀀서의 핵심 작동 메커니즘**

로컬 시퀀서는 각 그룹핑 키(예: visit\_id)별로 "다음에 처리해야 할 예상 시퀀스 번호(Next Expected Sequence Number)"를 추적한다.

1.  **정상 순서 도착:** 들어온 메시지의 시퀀스 번호가 Next Expected와 일치하면 즉시 최종 스토리지에 기록하고, Next Expected를 1 증가시킨다.
2.  **중복 데이터 도착 (과거 데이터):** 시퀀스 번호가 Next Expected보다 작다면, 이미 과거에 처리된 중복 데이터이므로 가볍게 무시(Drop)한다.
3.  **순서 뒤바뀜 도착 (미래 데이터):** 시퀀스 번호가 Next Expected보다 크다면(예: 10번을 기다리는데 11번이 먼저 옴), 중간에 데이터가 누락되었거나 뒤바뀐 것이다. 이 경우 11번을 즉시 저장하지 않고 내부 힙(Heap)이나 버퍼 메모리에 임시로 보관(Buffering)한 뒤 대기한다. 이후 10번이 도착하면 10번을 처리하고, 버퍼에 있던 11번을 꺼내어 연속적으로 처리한다.

**4-3. 시퀀스 상태를 관리하는 로컬 시퀀서 구조체 (LocalSequencer)**

각 키(세션)별로 다음에 기대하는 번호와 버퍼를 관리하는 핵심 클래스

```
import heapq

class LocalSequencer:
    def __init__(self, start_sequence: int = 0):
        # 다음에 처리되어야 할 올바른 시퀀스 번호
        self.next_expected_sequence = start_sequence
        # 순서가 뒤바뀌어 먼저 도착한 미래의 데이터를 정렬된 상태로 보관할 미니 힙(Min-Heap)
        self.buffered_records = []

    def add_to_buffer(self, sequence_number: int, record_value: dict):
        """ 순서가 어긋난 미래 데이터를 버퍼(우선순위 큐)에 담아둔다 """
        heapq.heappush(self.buffered_records, (sequence_number, record_value))

    def pop_next_buffered(self):
        """ 버퍼의 맨 앞에 있는 데이터가 내가 기다리던 번호인지 확인하고 꺼낸다 """
        if self.buffered_records and self.buffered_records[0][0] == self.next_expected_sequence:
            return heapq.heappop(self.buffered_records)
        return None
```

**4-4. 파이프라인 매니저에서의 데이터 조율 (OrderingPipelineManager)**

브로커에서 가져온 레코드를 로컬 시퀀서의 규칙에 따라 필터링하고 정렬하여 싱크에 기록하는 실질적인 아키텍처 비즈니스 로직

```
class OrderingPipelineManager:
    def __init__(self, consumer, storage_client):
        self.consumer = consumer
        self.storage_client = storage_client
        # 각 visit_id(키)별로 독립적인 로컬 시퀀서 상태를 유지 관리함 (메모리 상태 스토어)
        self.sequencers = {} 

    def process_record_in_order(self, record):
        visit_data = record.value
        visit_id = visit_data['visit_id']
        current_seq = visit_data['sequence_number']

        # 해당 visit_id에 대한 시퀀서가 없다면 새로 초기화 (여기서는 시작 번호를 데이터 기반으로 가정)
        if visit_id not in self.sequencers:
            self.sequencers[visit_id] = LocalSequencer(start_sequence=current_seq)

        sequencer = self.sequencers[visit_id]

        # Case 1: 중복 데이터 처리 (Drop 과거 데이터)
        if current_seq < sequencer.next_expected_sequence:
            print(f"중복 데이터 무시: {visit_id} (기대값: {sequencer.next_expected_sequence}, 들어온값: {current_seq})")
            return

        # Case 2: 순서 뒤바뀜 처리 (Buffer 미래 데이터)
        if current_seq > sequencer.next_expected_sequence:
            print(f"순서 뒤바뀜 감지, 버퍼 적재: {visit_id} - Seq {current_seq}")
            sequencer.add_to_buffer(current_seq, visit_data)
            return

        # Case 3: 올바른 순서 처리 (Process 정상 데이터)
        if current_seq == sequencer.next_expected_sequence:
            self._write_and_advance(visit_id, current_seq, visit_data, sequencer)

    def _write_and_advance(self, visit_id, seq, data, sequencer):
        """ 저장소에 순서대로 기록하고, 버퍼에 연쇄적으로 처리할 수 있는 데이터가 있는지 확인 """
        # 1. 현재 정상 데이터 싱크 저장소 기록
        self.storage_client.write(key=visit_id, data=data)
        sequencer.next_expected_sequence += 1

        # 2. 로컬 버퍼(우선순위 큐)를 확인하여 순서가 이어지는 미래 데이터가 있다면 연쇄 처리
        next_buffered = sequencer.pop_next_buffered()
        while next_buffered:
            buffered_seq, buffered_data = next_buffered
            print(f"버퍼에서 순서 복구 및 처리: {visit_id} - Seq {buffered_seq}")
            self.storage_client.write(key=visit_id, data=buffered_data)
            sequencer.next_expected_sequence += 1
            
            # 다음 번호도 연속해서 버퍼에 있는지 재확인
            next_buffered = sequencer.pop_next_buffered()
```

**4-5. 로컬 시퀀서 패턴의 결과 및 트레이드오프 (Trade-off)**

-   **장점: 완벽한 신뢰성과 극대화된 성능**
    -   프로듀서는 네트워크 대량 요청(Bulk 비동기) 전송을 적극 활용해 높은 Throughput을 뽑아낼 수 있다.
    -   프로듀서와 브로커 단에서 발생하는 미세한 순서 뒤바뀜 현상을 컨슈머의 로컬 메모리(우선순위 큐) 단에서 비용 효율적으로 교정한다.
-   **단점 1: 메모리 압박 (OOM 위험)**
    -   특정 패킷(예: seq 10번)이 영원히 유실되거나 심각하게 지연(Straggler)되는 경우, 그 이후의 데이터(seq 11~10000번)가 처리되지 못하고 계속 컨슈머 로컬 버퍼 메모리에 쌓이게 된다. 이는 곧 메모리 오버플로우(OOM)로 이어질 수 있다.
-   **단점 2: 타임아웃 및 데드락 해제 로직의 필요성**
    -   유실된 메시지로 인해 파이프라인이 무한정 대기하는 것을 방지하기 위해, 버퍼에 데이터가 머무를 수 있는 최대 시간(Hold Time)을 정의하고 타임아웃 발생 시 강제로 Next Expected를 건너뛰거나 에러 큐(DLQ)로 보내는 방어 로직이 추가로 요구된다.
