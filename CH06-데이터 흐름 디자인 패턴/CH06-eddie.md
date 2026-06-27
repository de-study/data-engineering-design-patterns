## CH 06 데이터 흐름(Data Flow) 디자인 패턴

**CH05에서 CH06으로 넘어오는 맥락**

- CH05는 **데이터 변환** 
	- 원시 이벤트를 받아서 → 집계, 세션, 정렬된 결과물을 만드는 것
	- "무엇을 만드는가"

- CH06은 **파이프라인 실행 제어**
	- 태스크 A가 끝나야 태스크 B를 실행
	- 팀A 파이프라인이 끝나야 팀B 파이프라인 시작
	- 여러 태스크를 동시에 실행할지, 순서대로 실행할지
	- "어떤 순서로, 언제, 누가 실행하는가"

**CH06의 핵심 질문**
- 여러 태스크를 어떤 순서로 실행할 것인가
- 여러 파이프라인이 서로 어떻게 의존할 것인가
- 병렬/순차 실행을 어떻게 제어할 것인가

**CH06 패턴 전체 구조**

|카테고리|패턴|핵심|
|---|---|---|
|시퀀스(Sequence)|#32 Local Sequencer|같은 파이프라인 내 태스크 순차 실행|
||#33 Isolated Sequencer|팀 간 분리된 파이프라인 연결|
|팬인(Fan-In)|Aligned Fan-In|모든 상위 태스크 성공 후 실행|
||Unaligned Fan-In|상위 태스크 성공 여부 무관하게 실행|
|팬아웃(Fan-Out)|Parallel Split|하나의 태스크에서 여러 브랜치 병렬 실행|
||Exclusive Choice|조건에 따라 하나의 브랜치만 실행|
|오케스트레이션|Single Runner|파이프라인 동시 실행 1개로 제한|
||Concurrent Runner|파이프라인 동시 실행 허용|




## 패턴#32 로컬 시퀀서(Local Sequencer)

### (1) 문제상황

**배경 : 단일 거대 잡의 한계**

- 수년간 누적된 잡 → 코드 수백 줄, 변환 로직 3배 증가
- 잡이 중간에 실패하면 처음부터 전체 재실행 → 디버깅 시간 폭증
- 실패 원인 파악이 어려움 → 어느 단계에서 터진 건지 알 수 없음
- 비즈니스 로직은 제거 불가

**문제 핵심**

|상황|문제|
|---|---|
|하나의 큰 잡으로 묶인 경우|실패 시 전체 재실행, 재시도 비용 증가|
|단계 간 의존성 불명확|어떤 단계가 어떤 데이터에 의존하는지 파악 어려움|
|유지보수|코드 수정 시 영향 범위 파악 어려움|

---

### (2) 솔루션

**주요 컨셉 : 하나의 큰 잡을 데이터 의존성 기준으로 순차적인 작은 태스크로 분리한다**

**분리 기준 3가지**

- 관심사 분리 : 태스크 이름 짓기가 어려우면 → 너무 많은 역할이 하나에 묶인 신호
- 유지보수성 : 재시도 시 이미 성공한 단계까지 재실행되면 → 분리 필요
- 구현 노력 : 오케스트레이터가 제공하는 추상화(SQL 실행, API 호출 등)를 못 쓰고 직접 구현하면 → 분리 필요

**동작 방식**

- 오케스트레이션 레이어(Airflow 등) : 태스크 간 `>>` 의존성으로 순차 실행
- 데이터 처리 레이어(PySpark 등) : 변수를 다음 단계의 입력으로 연결

---

### (3) 결과

**긍정적 효과**

- 실패 시 해당 태스크부터 재시작 → 재시도 비용 감소
- 단계별 책임 명확 → 디버깅 용이
- 오케스트레이터 내장 기능 활용 가능 (센서, 조건부 실행 등)

**트레이드오프**

|항목|내용|
|---|---|
|경계 식별 어려움|어디서 나눌지 정답이 없음, 판단 필요|
|과분리 위험|너무 잘게 쪼개면 오케스트레이터 오버헤드 증가|

---

### (4) 예시

> "레거시 Spark 잡 리팩토링할 때 Local Sequencer 많이 씀. 300줄짜리 잡이 센서→로드→변환→노출 4단계로 쪼개지면, 변환 단계에서 터졌을 때 센서/로드 단계 건너뛰고 변환부터 재실행할 수 있어. 비용이랑 시간 확 줄어들어."

**도메인** : 블로그 방문 데이터 수집 → 정제 → 노출 파이프라인  
**컴포넌트**

- Airflow DAG : 태스크 순서 오케스트레이션
- Spark 잡 : 데이터 처리 레이어

**Airflow 오케스트레이션 레이어**

```
# ★ 핵심: >> 연산자 — 왼쪽 태스크 성공 후 오른쪽 태스크 실행
input_data_sensor >> load_data_to_table >> expose_new_table
```

- `input_data_sensor` : 데이터 도착 감지
- `load_data_to_table` : 내부 테이블에 적재
- `expose_new_table` : 최종 사용자에게 노출

**PySpark 데이터 처리 레이어**

```
# 1단계: 원시 데이터 읽기
input_dataset = spark_session.read...
# input_dataset = S3에서 읽어온 원시 방문 이벤트 DataFrame

# ★ 핵심: 2단계 결과를 변수에 담아 3단계의 입력으로 연결
valid_and_enriched = (
    input_dataset
    .filter(...)   # 2단계: 유효성 검사 — 잘못된 이벤트 제거
    .join(...)     # 3단계: 데이터 보강 — 유저 정보 등 추가
)
# valid_and_enriched가 없으면 write 단계 실행 불가 → 암묵적 순서 보장

# 4단계: 정제된 데이터 저장
valid_and_enriched.write...
```
- Airflow : 태스크 단위로 실행 순서 선언, 실패 시 해당 태스크부터 재시작 가능
- PySpark : 코드 흐름이 순서 결정, 중간 실패 시 잡 전체 재실행 필요

AWS EMR 오케스트레이션 레이어
- EMR은 Airflow `>>`처럼 자동으로 상위 태스크 성공 여부를 체크하지 않음.  
- `ActionOnFailure` 설정으로 실패 시 동작을 **직접 명시**해야 함.
~~~
# 1단계: 데이터 적재 스텝 추가
aws emr add-steps \
  --cluster-id j-CLUSTER_ID \
  --steps Type=Spark,\
          Name="DataLoader",\
          ActionOnFailure=TERMINATE_CLUSTER,\  # ★ 핵심: 실패 시 클러스터 종료
          Args=[--class com.waitingforcode.DataLoader]

# 2단계: 데이터 노출 스텝 추가
aws emr add-steps \
  --cluster-id j-CLUSTER_ID \
  --steps Type=Spark,\
          Name="DataPublisher",\
          ActionOnFailure=TERMINATE_CLUSTER,\  # ★ 핵심: 실패 시 클러스터 종료
          Args=[--class com.waitingforcode.DataPublisher]
~~~

> "신입 때 주의할 점. Airflow에서 `>>`로 연결하면 기본적으로 상위 태스크 성공 시에만 다음 태스크 실행돼. 근데 AWS EMR 스텝 방식은 `ActionOnFailure` 직접 설정 안 하면 실패해도 다음 스텝 실행될 수 있어. EMR 쓸 때 `TERMINATE_CLUSTER` 꼭 확인해."

---

### (5) 최신트렌드

|툴|특징|선택 기준|
|---|---|---|
|Apache Airflow|태스크 의존성 시각화, 광범위한 오퍼레이터|배치 파이프라인 표준|
|Databricks Workflows|Spark 잡 네이티브 통합, 태스크 단위 재실행|Databricks 환경|
|Prefect / Dagster|코드 중심 오케스트레이션, 테스트 용이|최신 데이터 스택, Python 친화적|
|dbt|SQL 변환 단계 자동 의존성 관리|분석 엔지니어링, ELT 파이프라인|

**실무 트렌드**

- Airflow가 여전히 표준이지만 Dagster/Prefect로 전환하는 팀 증가
- 이유 : 코드 기반 의존성 정의 → 테스트 작성 용이, 로컬 개발 편리
- Databricks 환경이면 Workflows가 Airflow 대비 Spark 잡 관리 통합도가 높음


### (6)추가 - Orchestration 비교

**Airflow vs Dagster/Prefect 차이**

Airflow는 DAG 정의와 비즈니스 로직이 분리되어 있음.

```
# Airflow: DAG 파일 따로, 실제 로직 따로
@dag
def my_pipeline():
    task_a = PythonOperator(task_id='a', python_callable=run_a)
    task_b = PythonOperator(task_id='b', python_callable=run_b)
    task_a >> task_b

# 테스트하려면 Airflow 환경 전체가 필요
# 로컬에서 단위 테스트 작성 어려움
```

Dagster는 데이터 에셋 자체가 의존성을 선언함.

```
# Dagster: 함수 자체가 파이프라인
@asset
def cleaned_visits(raw_visits):      # raw_visits가 있어야 실행 → 자동 의존성
    return raw_visits.filter(...)

@asset
def session_table(cleaned_visits):   # cleaned_visits가 있어야 실행 → 자동 의존성
    return cleaned_visits.groupBy(...)

# 테스트: 그냥 Python 함수 테스트와 동일
def test_cleaned_visits():
    result = cleaned_visits(mock_raw_visits)
    assert result.count() > 0
```

**실무에서 편한 포인트**

|항목|Airflow|Dagster/Prefect|
|---|---|---|
|로컬 테스트|Airflow 서버 띄워야 함|그냥 pytest로 실행|
|의존성 선언|`>>` 연산자로 수동 연결|함수 인자가 곧 의존성|
|에셋 계보|별도 lineage 툴 필요|UI에서 자동으로 시각화|
|디버깅|로그 보려면 UI 접속|로컬 터미널에서 바로 확인|

**Databricks Workflows가 편한 포인트**

- Spark 잡을 Workflows에 등록하면 클러스터 생성/종료 자동 관리
- Airflow에서 Spark 잡 실행하려면 `SparkSubmitOperator` + 클러스터 설정 직접 관리
- 태스크 단위 재실행 시 동일 클러스터 재사용 가능 → 클러스터 시작 대기 시간 없음
- Delta Lake, MLflow, Unity Catalog와 네이티브 통합 → 별도 연결 설정 불필요

**선택 기준 요약**

|상황|선택|
|---|---|
|이미 Airflow 운영 중, 팀 규모 큼|Airflow 유지|
|새 프로젝트, Python 친화적 팀|Dagster|
|Databricks 올인 환경|Databricks Workflows|
|경량 파이프라인, 빠른 프로토타이핑|Prefect|
