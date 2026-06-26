## 패턴#31 빈 팩 정렬기(Bin Pack Orderer)

### (1) 문제상황
**배경 : 부분 커밋(Partial Commit)이 왜 문제인가**

- 일반 커밋 : 성공 또는 실패, 2가지 상태만 존재
- 부분 커밋 : 성공 / 부분 실패 / 전체 실패, 3가지 상태 존재
- 부분 실패 발생 시 → 일부 레코드만 저장됨 → 재시도 필요
- 재시도한 레코드가 이미 저장된 레코드보다 늦게 도착 → 순서 깨짐

**부분 커밋을 지원하는 실제 API**
- Amazon Kinesis `PutRecords`
- AWS DynamoDB `BatchWriteItem`
- Elasticsearch bulk API

**실제 문제 상황**
블로그 플랫폼, 외부 파트너 웹사이트에 방문 이벤트를 API로 동기화하는 잡이 있다.

- 10분 처리 타임 윈도우로 이벤트 버퍼링
- 파트너별, 분 단위로 이벤트를 이벤트 타임 순서대로 전달해야 함
- 타겟 스토리지 : Kinesis (`PutRecords` → 부분 커밋 발생 가능)

```
전달해야 할 이벤트:
[10:00] partner_A, visit_id=V001
[10:00] partner_A, visit_id=V002
[10:10] partner_A, visit_id=V001
[10:10] partner_A, visit_id=V002
```

bulk API로 한 번에 전달 시:
```
PutRecords 요청 → [10:00 V001, 10:00 V002, 10:10 V001, 10:10 V002]
결과 → 부분 실패: [10:10 V001, 10:10 V002]만 저장됨
재시도 → [10:00 V001, 10:00 V002] 재전송
최종 저장 순서 → 10:10 → 10:00  ← 순서 역전
```

**문제 핵심**
- 건별 전송 : 순서 보장 가능하지만 네트워크 요청 수 폭증
- bulk 전송 : 네트워크 효율적이지만 부분 커밋으로 순서 깨짐
- 필요한 것 : bulk 전송의 효율성 + 순서 보장




### (2) 솔루션
**주요 컨셉 : 레코드를 그루핑 키 + 이벤트 타임으로 정렬 후, 같은 위치의 서로 다른 엔티티끼리 bin으로 묶어 순차 전송한다**

**bin이란** : 서로 다른 `visit_id`의 같은 순서 이벤트를 하나로 묶은 배열. 한 bin 안에 같은 `visit_id`는 절대 2개 이상 존재하지 않음

**처리 흐름 3단계**
1. 그루핑 키(`visit_id`) + 이벤트 타임 기준으로 정렬
2. 정렬된 레코드를 bin으로 분배 → bin 하나에 같은 `visit_id` 1개만
3. bin을 순서대로 순차 전송 → 현재 bin 전송 완료 확인 후 다음 bin 전송

**bin 구성 예시**
```
정렬 후:
[10:00 V001], [10:00 V002], [10:10 V001], [10:10 V002]

bin 분배:
bin[0] → [10:00 V001, 10:00 V002]   ← V001, V002 각 1개씩
bin[1] → [10:10 V001, 10:10 V002]   ← V001, V002 각 1개씩

전송:
bin[0] 전송 → 완료 확인 → bin[1] 전송
```

**부분 커밋 발생 시 재시도가 안전한 이유**
- bin 하나 안에 같은 `visit_id`가 1개뿐
- bin[0]에서 V002 실패 → V002만 재시도 → V001 순서에 영향 없음
- bin[0] 완료 전까지 bin[1] 전송 안 함 → 10:00이 항상 10:10보다 먼저

---

### (3) 결과

**긍정적 효과**
- 부분 커밋 환경에서도 이벤트 타임 순서 보장
- bulk API 사용으로 건별 전송 대비 네트워크 요청 수 감소

**트레이드오프**

|항목|내용|
|---|---|
|파이프라인 전체 재시도|이미 전송된 bin이 재전송됨 → 순서 보장 깨질 수 있음|
|구현 복잡도|단순 ORDER BY보다 bin 생성 로직 추가 필요|

---

### (4) 예시

> "Kinesis PutRecords 쓰다가 부분 커밋으로 이벤트 순서가 꼬인 적 있어. PutRecords는 성공/실패가 레코드별로 따로 떨어지거든. 그때부터 Bin Pack Orderer 써서 bin 단위로 순차 전송하도록 바꿨어."

**도메인** : 블로그 플랫폼 방문 이벤트 → 파트너 Kinesis로 동기화  
**컴포넌트**

- Spark 잡 : 이벤트 정렬 + bin 생성
- Amazon Kinesis : 파트너 전달 타겟 (부분 커밋 발생)

**코드 1 : 파티션 내 정렬 (Spark 잡)**

파티션 내부에서만 정렬 → 네트워크 셔플 없이 로컬 정렬

```
# ★ 핵심: sortWithinPartitions — 셔플 없이 파티션 내부에서만 정렬
(events
  .sortWithinPartitions([F.col('visit_id'), F.col('event_time')])
  .foreachPartition(lambda rows: write_records_to_kinesis(rows)))
```

**코드 2 : bin 생성 + Kinesis 전송**

정렬된 레코드를 bin으로 분배하고 순차 전송

```
def write_records_to_kinesis(visits_rows):
    delivery_groups = []
    groups_index = 0
    last_visit_id = None

    # ★ 핵심: visit_id가 바뀌면 groups_index 리셋 → bin 위치 0부터 다시
    for visit in visits_rows:
        if visit.visit_id != last_visit_id:
            last_visit_id = visit.visit_id
            groups_index = 0                        # bin 위치 초기화

        if len(delivery_groups) <= groups_index:
            delivery_groups.append([])              # 새 bin 생성
        delivery_groups[groups_index].append(visit) # bin에 레코드 추가
        groups_index += 1

    # bin 순서대로 순차 전송
    for group in delivery_groups:
        kinesis.put_records(group)                  # 현재 bin 전송 완료 후 다음 bin
```

**bin 생성 과정 추적**

```
입력 (정렬 완료):
V001/10:00 → V002/10:00 → V001/10:10 → V002/10:10

처리:
V001/10:00 : last_visit_id=None→V001, index=0, bin[0]=[V001/10:00], index=1
V002/10:00 : last_visit_id=V001→V002, index리셋=0, bin[0]=[V001/10:00, V002/10:00], index=1
V001/10:10 : last_visit_id=V002→V001, index리셋=0, bin[0] 이미 있음 → bin[1]=[V001/10:10], index=1
V002/10:10 : last_visit_id=V001→V002, index리셋=0, bin[1]에 추가 → bin[1]=[V001/10:10, V002/10:10]

결과:
bin[0] → [V001/10:00, V002/10:00]
bin[1] → [V001/10:10, V002/10:10]
```

> "신입 때 주의할 점 하나. `sortWithinPartitions`는 파티션 내부만 정렬해. 파티션 간 순서는 안 맞춰줘. 그래서 같은 `visit_id` 이벤트가 여러 파티션에 흩어지면 bin 로직이 의미 없어져. `visit_id` 기준으로 파티셔닝을 먼저 맞춰놓거나, repartition 먼저 하고 써야 해."

---

### (5) 최신트렌드

**실무에서 Bin Pack Orderer를 어떤 방식으로 처리하는가**

|툴/방식|특징|선택 기준|
|---|---|---|
|Kafka 멱등 프로듀서|`enable.idempotence=true`로 순서 + 중복 방지 자동 보장|Kafka가 타겟인 경우|
|Kafka `max.in.flight.requests.per.connection=1`|in-flight 요청 1개로 제한 → 순서 보장|단순 FIFO 순서 보장 필요 시|
|Flink + Kafka exactly-once|트랜잭션 기반 순서 + 정확히 한 번 보장|순서 + 중복 제거 모두 필요 시|
|AWS Kinesis Enhanced Fan-Out|샤드 키 기반 순서 보장 내장|Kinesis 환경에서 순서 필요 시|

**실무 트렌드**

- Kafka가 타겟이면 Bin Pack Orderer 직접 구현보다 **Kafka 멱등 프로듀서**로 대체하는 게 일반적
- 이유 : Kafka 멱등 프로듀서가 파티션 내 순서 + at-least-once → exactly-once 자동 처리
- Kinesis처럼 부분 커밋이 구조적으로 발생하는 타겟에서만 Bin Pack Orderer 패턴 직접 구현





## 패턴#32 FIFO 정렬기(FIFO Orderer)

### (1) 문제상황

**Bin Pack Orderer(#30)에서 넘어온 맥락**

- Bin Pack Orderer : bulk 전송 + 순서 보장, but 구현 복잡 + 버퍼링으로 레이턴시 증가
- 데이터량이 적거나 레이턴시보다 단순성이 중요한 경우엔 과한 구현

**실제 문제 상황**

- 스트리밍 잡이 방문 이벤트에서 특정 이벤트만 감지해서 다른 스트림으로 전달
- 요건 : 레코드를 **즉시** 전달 → 버퍼링 불가
- 요건 : 처리 순서(FIFO) 보장 필요
- 데이터량 : 소규모, 저지연 최우선

**문제 핵심**

|방식|문제|
|---|---|
|bulk 전송 (Bin Pack Orderer)|버퍼 채울 때까지 대기 → 즉시 전달 불가|
|일반 건별 전송|순서 보장 없음, ack 실패 시 중복 전송 위험|

---

### (2) 솔루션

**주요 컨셉 : 레코드를 하나씩 전송하고, 각 전송의 ack를 확인한 후 다음 레코드를 전송한다**

**구현 방식 3가지**

|방식|동작|선택 기준|
|---|---|---|
|건별 전송 + flush|`send()` → `flush()` → 다음 레코드|가장 단순, 타겟이 부분 커밋 지원 시|
|bulk + `max.in.flight=1`|1초 버퍼링, 동시 요청 1개로 제한|약간의 처리량 개선 필요 시|
|멱등 프로듀서 (`enable.idempotence=True`)|동시 요청 최대 5개, 순서 자동 보장|Kafka 타겟, 처리량 + 순서 모두 필요 시|

**실무 선호 : 멱등 프로듀서**

- 이유 : 동시 요청 5개까지 허용하면서도 Kafka가 순서 자동 보장
- `max.in.flight=1`은 처리량이 너무 낮음

**FIFO ≠ exactly-once 주의**

- `send()` 성공 후 `ack()` 실패 가능
- 재시작 시 이미 전달된 레코드 재전송 → 중복 발생
- 해결 : CH04 멱등성 패턴 (#Keyed Idempotency 등) 조합 필요

---

### (3) 결과

**긍정적 효과**

- 구현 단순 (Bin Pack Orderer 대비)
- 버퍼링 없이 즉시 전달 가능
- 처리 순서 보장

**트레이드오프**

|항목|내용|
|---|---|
|I/O 오버헤드|건별 전송 시 레코드 수만큼 네트워크 요청 발생|
|레이턴시 증가|데이터 많을수록 건별 전송 부담 커짐|
|exactly-once 미보장|FIFO는 순서만 보장, 중복 제거는 별도 처리 필요|

---

### (4) 예시

> "이벤트 필터링해서 특정 이벤트만 다른 Kafka 토픽으로 포워딩하는 잡 짤 때 FIFO Orderer 씀. 데이터 많지 않고 즉시 전달이 중요한 경우엔 멱등 프로듀서 설정만으로 충분해. Bin Pack Orderer처럼 bin 로직 짤 필요 없어."

**도메인** : 블로그 방문 이벤트 중 특정 이벤트만 감지 → Kafka 토픽으로 포워딩  
**컴포넌트**

- Spark Structured Streaming 잡 : 이벤트 필터링
- Kafka 프로듀서 : FIFO 순서 보장 전송

**방식 1 : 건별 전송 (가장 단순)**

```
# ★ 핵심: flush() — send() 직후 즉시 전송 강제, 버퍼 대기 없음
producer.produce(topic, value=record)
producer.flush()   # 이 레코드 ack 확인 후 다음으로
```

**방식 2 : bulk + 동시 요청 1개 제한**

```
# ★ 핵심: max.in.flight.requests.per.connection=1 — 동시 요청 1개로 순서 보장
producer = Producer({
    'max.in.flight.requests.per.connection': 1,  # 동시 요청 1개만
    'queue.buffering.max.ms': 1000               # 최대 1초 버퍼링
})
producer.produce(topic, value=record)
# flush() 없음 — 프로듀서가 알아서 배치 전송
```

**방식 3 : 멱등 프로듀서 (실무 권장)**

```
# ★ 핵심: enable.idempotence=True — 동시 요청 5개 허용하면서 Kafka가 순서 자동 보장
producer = Producer({
    'enable.idempotence': True,                  # 멱등 프로듀서 활성화
    'max.in.flight.requests.per.connection': 5,  # 동시 요청 최대 5개
    'queue.buffering.max.ms': 2000               # 2초 버퍼링으로 처리량 개선
})
producer.produce(topic, value=record)
```

> "신입 때 `flush()` 빠뜨리고 건별 전송 구현했다가 이벤트가 뒤섞여서 나온 적 있어. `produce()` 자체는 버퍼에 넣는 거지 즉시 전송이 아니거든. `flush()` 없으면 버퍼가 다 찰 때까지 기다렸다가 한꺼번에 나가는데, 그 사이에 순서 꼬여. 그리고 FIFO가 exactly-once라고 착각하지 마. 순서만 보장하는 거야. 중복 제거는 따로 멱등성 패턴 붙여야 해."

---

### (5) 최신트렌드

**실무에서 FIFO Orderer를 어떤 방식으로 처리하는가**

|툴/방식|특징|선택 기준|
|---|---|---|
|Kafka 멱등 프로듀서|순서 + 중복 방지 자동, 동시 요청 5개|Kafka 타겟, 가장 일반적|
|Kafka transactions|exactly-once + 순서 보장|중복 제거까지 필요 시|
|Kinesis PutRecord + SequenceNumberForOrdering|샤드 내 순서 보장|Kinesis 타겟|
|Flink exactly-once sink|체크포인트 기반 순서 + exactly-once|대규모 스트리밍 파이프라인|

**실무 트렌드**

- Kafka 환경 : 멱등 프로듀서가 사실상 표준 → 별도 FIFO 로직 구현 불필요
- exactly-once까지 필요하면 Kafka transactions 또는 Flink exactly-once sink 조합
- Kinesis 환경 : 부분 커밋 구조상 FIFO 직접 구현 불가피 → `PutRecord` + `SequenceNumberForOrdering` 조합





