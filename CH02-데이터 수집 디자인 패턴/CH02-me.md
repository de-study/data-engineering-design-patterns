# 2. 데이터 수집 디자인 패턴

# 2.1 전체 적재

- 전체 적재(full load) 디자인 패턴은 매번 데이터셋 전체를 로드하는 데이터 수집 시나리오를 의미
- 데이터베이스 부트스트랩, 참조 데이터셋(reference dataset) 생성 등에 유용

## 패턴1: 전체 로더

### 2.1.1 문제

- 데이터셋이 잘 변경되지 않고 전체 레코드가 백만 개를 넘지 않는 천천히 변화하는 엔티티
- 해당 엔티티는 마지막 데이터 수집 이후 변경된 레코드를 감지하는 데 도움이 되는 속성을 정의하지 않음

### 2.1.2 해결책

- 전체 로더: 추출과 적재(EL) 사용 - 별도의 데이터 변환이 필요하지 않으므로 동종의 데이터 스토어에 이상적
    - 이종의 데이터베이스 간에 데이터를 적재할 경우 추출 단계와 적재 단계 사이에 변환 계층 필요(ETL)
        - 형식 변환, 단위 변환, 구조 변환 등

### 2.1.3 결과

1. 데이터 크기
    - 데이터셋의 크기가 천천히 증가하면 문제 없음
    - 동적으로 진화하는 데이터셋은 정적 컴퓨팅 자원을 사용하여 처리할 경우 문제 발생 → 데이터 처리 계층의 오토스케일링 기능 활용 필요
2. 데이터 일관성
    - 데이터를 완전히 덮어쓰거나 삭제 및 삽입 작업으로 완전히 교체하는 경우
        - 데이터 수집 프로세스와 데이터셋 조회 파이프라인이 동시 실행되는 경우 컨슈머는 데이터를 부분적으로만 처리하거나 전혀 보지 못할 수 있음
        - 트랜잭션을 사용 하거나 뷰와 같은 단일 데이터 노출 추상화에 의존 필요
    - 예상치 못한 문제가 발생하여 이전 버전의 데이터셋을 사용해야 하는 경우
        - 델타 레이크, 아파치 아이스버그 또는 빅쿼리 등과 같이 타임 트래블 기능을 지원하는 형식 사용
        - 단일 데이터 노출 추상화 개념을 사용하여 직접 해당 기능 구현 가능

### 2.1.4 예제

- 동일하거나 호환 가능한 두 데이터 스토어 간에 데이터셋 수집

```scala
# 원본에 없지만 목적지에는 있는 객체 모두를 제거(--delete 인수)
$ aws s3 sync s3://input-bucket s3://output-
```

- 적재 작업에 병렬 처리나 분산 처리 추가 같은 일부 튜닝 작업이 필요한 경우
    - Delta Lake는 데이터 저장 중 에러가 나면 작업 전 상태로 되돌리는 기능을 제공

```scala
# 아파치 스파크 및 델타 레이크를 사용한 추출 적재 구현
input_data = spark.read.schema(input_data_schema).json("s3://devices/list")
input_data.write.format("delta").save("s3://master/devices")
```

- 네이티브 버전 관리 기능이 없는 데이터베이스에서 구현
    - 새로운 데이터를 새 테이블에 받고, 마지막에 "이름표(View)"만 바꿔 다는 방식

```scala
# airflow + PostgreSQL

# 데이터 수집 - 데이터셋을 명시적으로 버전이 관리되는 테이블에 씀
COPY devices_${version} FROM '/data_to_load/dataset.csv' CSV DELIMITER ';' HEADER;

# 노출 작업 - 최종 사용자에게 노출되는 뷰의 참조를 변경(뷰 갱신 작업)
CREATE OR REPLACE VIEW devices AS SELECT * FROM devices_${version}
```

# 2.2 증분 적재

## 패턴2: 증분 로더

### 2.2.1 문제

- 데이터 크기가 계속 증가하는 레거시 방문 데이터를 마지막 실행 이후 추가된 방문만 통합
- 각 방문 이벤트는 불변

### 2.2.2 해결책

- **델타 컬럼** 사용하여 마지막 실행 이후 추가된 레코드 식별
    - 일반적으로 불변의 방문 데이터와 같은 이벤트 기반 데이터의 경우 “수집 시간”을 컬럼으로 사용
    - 실시간 데이터에서는 데이터 프로듀서가 이미 처리한 이벤트 시간에 대한 지연된 데이터를 전송할 경우 수집 프로세스 상에서 해당 레코드 누락 가능성 있음
- 시간 분할 데이터셋(**time-partitioned datasets**): 수집 잡은 시간 기반 분할을 사용해 수집할 새로운 레코드 집합을 감지 - 파티셔닝 사용
    - 수집을 위한 데이터가 이미 필터링되고 스토리지 계층에서 논리적으로 조직되어서 프로세스가 크게 단순화되고 최적화됌
    - 새로운 파티션 수집이 보장되도록 준비 마커 패턴 사용

### 2.2.3 결과

수집된 데이터 크기를 줄이는데 좋지만 아래 사항에 대한 고려 필요

- 물리삭제
    - 증분 로더를 갱신과 삭제가 가능한 이벤트 처리에 사용하는 것은 까다로울 수 있음
    - 물리 삭제 대신 논리 삭제(soft delete)를 사용하기도 함
    - 논리 삭제: 프로듀서가 데이터를 물리적으로 제거하지 않는 대신 해당 데이터를 삭제된 것으로 표시(DELETE 대신 UPDATE 명령어 사용)
    - 삽입 전용(insert-only) 테이블을 사용하기도 함(이벤트를 시간 순서대로 기록)
        - INSERT 작업을 통해서만 새 레코드 허용
        - 데이터 재구성에 대한 책임은 컨슈머에게 전가
- 백필링
    - 예를 들어 2달치 데이터 처리 후, 백필 필요 → 전체 적재 수행에 따른 더 많은 자원 필요
    - 수집 윈도를 제한하여 문제 완화 가능 - 수집 잡이 매시간 실행된다면 수집 프로세스를 한 시간으로 제한 가능
        - 데이터 크기를 더 잘 제어 가능, 백필링 시에도 컴퓨팅 요규 사항에 놀라지 않게 됌
        - 동시 수집. 입력 데이터 스토어에서 지원하는 한 여러 개의 백필링 잡을 동시에 수행 가능

### 2.2.4 예제

- 증분 로드 파티셔닝 DAG 예제

```python
from airflow import DAG
from airflow.providers.amazon.aws.sensors.s3 import S3KeySensor
from airflow.providers.cncf.kubernetes.operators.spark_kubernetes import SparkKubernetesOperator
from datetime import datetime, timedelta

# 1. DAG 기본 설정
default_args = {
    'owner': 'data_eng',
    'depends_on_past': True, # 이전 날짜 작업이 성공해야 오늘 작업 시작 (데이터 일관성)
    'start_date': datetime(2024, 1, 1)
}

with DAG(
    dag_id='incremental_load_workflow',
    default_args=default_args,
    schedule_interval='@daily', # 매일 실행
    catchup=False
) as dag:

    # 2. 센서 단계: 데이터가 들어왔는지 확인 (이미지 예제 2-6의 FileSensor 역할)
    # {{ ds }}는 Airflow에서 제공하는 '실행 날짜' 매크로입니다.
    wait_for_data = S3KeySensor(
        task_id='check_s3_partition_exists',
        bucket_name='raw-data-bucket',
        bucket_key='input/date={{ ds }}/_SUCCESS', # 해당 날짜 폴더에 완료 파일이 있는지 확인
        timeout=1800,
        poke_interval=60
    )

    # 3. 트리거 단계: 스파크 잡 실행 (이미지 예제 2-8의 SparkKubernetesOperator)
    # 실제 증분 로직(BETWEEN 구문 등)이 담긴 파일을 실행합니다.
    submit_spark_job = SparkKubernetesOperator(
        task_id='run_incremental_spark_job',
        application_file='load_job_spec.yaml', # 이 파일 안에서 {{ ds }}를 받아 특정 날짜만 필터링함
        namespace='spark-jobs'
    )

    # 4. 의존성 정의 (이미지의 >> 기호와 동일)
    # 데이터 확인이 완료되면 >> 증분 수집 잡을 실행한다.
    wait_for_data >> submit_spark_job
```

- 델타 컬럼과 시간 경계를 포함한 데이터 수집 작업

```python
in_date = (sparkk_session.read.text(input_path).select('value', functions.from_json(functions.col('value'), 'ingestion_time TIMESTAMP')))

input_to_write = in_date.filter(
	f'ingestion_time BETWEEN "{date_from}" AND "{date_to}"'
)

input_to_write.mode('append').select('value').write.text(output_path)
```

## 패턴3: 변경 데이터 캡처

데이터 수집 시 지연 시간이 낮아야 하거나 물리 삭제를 지원할 자체 기능이 필요할 때는 변경 데이터 캡처 패턴이 더 맞는 선택지 일 수 있음

### 2.3.1 문제

- 데이터 수집 속도가 매우 느리고, 다운스트림 컨슈머들이 데이터 대기 시간이 너무 길다고 불평

### 2.3.2 해결책

- CDC 사용: 내부 데이터베이스 커밋 로그에서 수정된 모든 레코드를 지속적으로 직접 수집하는 방식으로 구성
- 커밋로그는 추가만 가능한 구조 - 기존 레코드에 대한 모든 작업을 로그파일 끝에 기록
- CDC는 지연 감소 보장뿐만 아니라, 물리 삭제를 포함한 모든 유형의 데이터 작업을 가로채기(데이터베이스의 트랜잭션 로그를 실시간으로 읽어서 복제본에 반영) 때문에 데이터 제거를 위해 데이터 프로듀서에게 논리 삭제 명령할 필요 없음
- 위 특징 떄문에 CDC를 도입하면 데이터 파이프라인 설계가 단순해짐
    - **자율성:** 데이터 프로듀서(원본 DB 운영자)에게 "우리 데이터 분석해야 하니까 데이터를 직접 지우지 말고 삭제 플래그(논리 삭제)만 세워주세요"라고 부탁하거나 강제할 필요가 없음
    - **완전성:** 프로듀서가 어떤 방식으로 데이터를 처리하든(진짜 지우든, 수정하든), CDC가 로그 레벨에서 모든 변화를 가로채기 때문에 분석 시스템은 원본 DB의 상태를 완벽하게 복제 가능

### 2.3.3 결과

아래 사항에 대한 고려 필요

- 복잡성: 기술 역량 필요 + 서버에서 커밋 로그를 활성화해야 하는 경우처럼 운영 팀의 도움이 필요할 수 있음
- 데이터 범위: 클라이언트의 구현에 따라 클라이언트 프로세스를 시작한 후에만 변경 사항 적용 가능
- 페이로드: CDC는 레코드와 함꼐 메타데이터를 제공하기 때문에 관련 없는 속성을 무시하도록 transform이 필요할 수 있음
- 데이터 시맨틱:
