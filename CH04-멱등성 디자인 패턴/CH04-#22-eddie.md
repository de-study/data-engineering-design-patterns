# 4.4 불변의 데이터셋(Immutable Dataset)

## 패턴#21 프록시(Proxy)

### (1) 문제상황

매일 Spark 배치 잡이 Kafka에서 결제 이벤트를 읽어서 Delta Lake `payments` 테이블에 적재한다고 가정해.

- 1월 1일 실행 → `payments` 테이블에 100건 적재
- 1월 2일 실행 → TRUNCATE 후 INSERT, `payments` 테이블에 120건만 남음
- 1월 3일 실행 → 또 덮어씀

멱등성은 지켜졌어. 재실행해도 항상 같은 결과가 나오니까.

근데 감사팀이 "1월 1일 당시 결제 데이터 보여줘"라고 하면? 이미 덮어썼으니까 복원 불가야.

금융·의료처럼 **"이 시점에 이 데이터가 이 상태였다"를 반드시 보장해야 하는 도메인**에서는 매번 덮어쓰는 구조 자체가 문제가 된다.



### (2) 솔루션

주요 컨셉: 매 실행마다 새 테이블에 쓰고, 소비자에게 노출되는 이름(뷰)만 갱신한다.

매일 실행할 때마다 새 테이블을 만들어서 거기에 쓴다. 기존 테이블은 절대 건드리지 않는다.
- 1월 1일 실행 → `payments_20250101` 테이블 생성 후 적재
- 1월 2일 실행 → `payments_20250102` 테이블 생성 후 적재
- 1월 3일 실행 → `payments_20250103` 테이블 생성 후 적재

근데 소비자는 매일 테이블 이름이 바뀌면 쿼리를 매일 수정해야 해. 그래서 뷰(View: 실제 데이터 없이 특정 테이블을 가리키는 포인터)를 하나 만든다.
- 1월 1일 실행 후 → `payments` 뷰가 `payments_20250101`을 가리킴
- 1월 2일 실행 후 → `payments` 뷰가 `payments_20250102`를 가리킴

소비자는 항상 `payments`만 조회하면 된다. 과거 테이블은 그대로 남아있으니까 
1월 1일 데이터도 언제든 조회 가능하다.
재실행이 일어나도 이전 테이블은 건드리지 않고, 새 테이블에 새로 쓰고 뷰만 다시 연결함.



### (3) 결과

장점:
- 이전 실행 데이터가 절대 훼손되지 않음 → 감사/규제 대응 가능
- 소비자는 뷰 이름 하나만 알면 됨, 내부 구조 변경에 영향 없음
- 실행 실패 후 재실행해도 멱등성 보장

단점 및 주의사항:
- 모든 DB가 뷰(View)를 지원하지는 않음. 지원 안 하면 매니페스트 파일로 대체해야 하는데, 소비자가 매니페스트를 직접 읽어야 해서 번거로워짐
- 내부 테이블이 실행마다 쌓이므로 주기적인 정리 정책(retention) 필요
- S3 같은 오브젝트 스토어에선 Object Lock으로 물리적 불변성을 추가로 걸어야 진짜 보장이 됨 (Airflow 레벨만으론 부족)
- 쓰기 권한은 테이블 생성만 허용해야 함. 삭제/수정 권한이 있으면 실수로 이전 테이블을 지울 수 있음




### (4) 예시

엔지니어 독백:
> 우리 팀은 매주 IoT 디바이스 상태 데이터를 PostgreSQL에 적재하고 있었어. 처음엔 그냥 같은 테이블에 TRUNCATE 후 INSERT로 운영했는데, 어느 날 감사팀에서 3주 전 데이터 기준 보고서를 다시 만들어달라는 요청이 왔어. 근데 데이터가 이미 다 덮여있어서 복원이 불가능했지. 그때부터 Proxy 패턴으로 바꿨어.

Airflow DAG는 두 태스크로 구성된다.
- `load_data_to_internal_table`: 매 실행 시 고유 이름의 내부 테이블에 데이터 적재. COPY 커맨드로 단순 삽입
- `refresh_view`: `dedp.payments` 뷰를 새 내부 테이블로 교체

내부 테이블 이름 생성 로직이 핵심이다. Airflow 실행 컨텍스트에서 DAG 시작 시간을 가져와 suffix로 붙인다. 같은 DAG를 재실행해도 항상 같은 suffix가 나오기 때문에 멱등성이 보장된다.

코드내용: "오늘 DAG가 몇 시에 시작됐는지"를 테이블 이름에 붙여주는 것."
```
# ★ 핵심: DAG 시작 시간으로 고유한 테이블 이름 생성
def get_payments_table_name() -> str:
    context = get_current_context()
    dag_run: DagRun = context['dag_run']
    table_suffix = dag_run.start_date.strftime('%Y%m%d_%H%M%S')
    return f'dedp.payments_internal_{table_suffix}'
```
한 줄씩 보면:
`context = get_current_context()`:  Airflow에서 현재 실행 중인 DAG의 정보를 가져옴

`dag_run: DagRun = context['dag_run']` :
- `context['dag_run']`에서 꺼낸 값을 `dag_run`이라는 변수에 담는데, 그 변수의 타입이 `DagRun`이라는 걸 명시해주는 거야.
- 그 정보 중에서 "이번 실행(dag_run)" 객체를 꺼내. dag_run 안에 시작시간, 실행ID 같은 정보가 들어있어.
```
context = {
    'dag_run': DagRun(
        run_id='scheduled__2025-01-01T14:00:00',
        start_date=datetime(2025, 1, 1, 14, 0, 0),
        dag_id='payments_pipeline'
    ),
    'ds': '2025-01-01',
    ...
}
```

`table_suffix = dag_run.start_date.strftime('%Y%m%d_%H%M%S')`:
- dag_run의 시작 시간을 `20250101_143022` 형태의 문자열로 변환해. 
- 이게 테이블 이름에 붙을 suffix야.

`return f'dedp.payments_internal_{table_suffix}`:
- 최종적으로 `dedp.payments_internal_20250101_143022` 같은 테이블 이름을 반환해.
- 뷰 갱신 SQL의 핵심은 `CREATE OR REPLACE VIEW`다. 
- 이 SQL이 원자적으로 뷰가 가리키는 테이블을 교체하기 때문에 소비자 입장에선 중단 없이 최신 데이터로 전환된다.

```
# ★ 핵심: 뷰가 가리키는 테이블만 교체. 소비자 쿼리는 변경 없음
{% set payments_internal_table = get_payments_table_name() %}
CREATE OR REPLACE VIEW dedp.payments AS
    SELECT * FROM {{ payments_internal_table }};
```

엔지니어 팁:
> 신입 때 이 패턴 처음 보면 "왜 이렇게 복잡하게 하지?"라고 생각할 수 있어. 근데 실무에서 데이터 롤백 요청 한 번 받아보면 바로 이해돼. 그리고 한 가지 꼭 기억해: 내부 테이블 생성 권한만 줘야 해. 삭제 권한까지 주면 누군가 실수로 지워버리는 순간 불변성 보장이 전부 날아가.



### (5) 최신트렌드

요즘 실무에서 Proxy 패턴을 구현하는 방식은 크게 세 가지다.

- Delta Lake / Apache Iceberg (가장 선호)
    - 테이블 자체가 버전 관리를 내장하고 있어 덮어써도 이전 스냅샷이 남음
    - `AS OF VERSION` 또는 `TIMESTAMP AS OF` 쿼리로 과거 시점 조회 가능
    - 별도 내부 테이블 생성 없이 Time Travel이 Proxy 역할을 대신함
    - 코드가 가장 단순하고 운영 부담이 적어 신규 구축 시 1순위
- Unity Catalog (Databricks) / AWS Glue Data Catalog
    - 카탈로그 레벨에서 테이블 버전과 접근 권한을 함께 관리
    - 뷰 갱신 없이 카탈로그 alias 또는 tag로 "현재 버전" 지정 가능
- S3 Object Lock + Airflow 뷰 갱신 (전통적 방식)
    - Delta/Iceberg를 쓸 수 없는 레거시 환경에서 사용
    - Object Lock으로 물리적 불변성 보장 + 뷰 교체로 접근점 고정
    - 운영 복잡도가 높아 신규 구축엔 비추, 레거시 유지보수 상황에서 만나게 됨
