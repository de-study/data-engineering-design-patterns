**CH06의 핵심 질문**
- 여러 태스크를 어떤 순서로 실행할 것인가
- 여러 파이프라인이 서로 어떻게 의존할 것인가
- 병렬/순차 실행을 어떻게 제어할 것인가

**CH06 패턴 전체 구조**

| 카테고리          | 패턴                     | 핵심                     |
| ------------- | ---------------------- | ---------------------- |
| 시퀀스(Sequence) | #32 Local Sequencer    | 같은 파이프라인 내 태스크 순차 실행   |
|               | #33 Isolated Sequencer | 팀 간 분리된 파이프라인 연결       |
| 팬인(Fan-In)    | Aligned Fan-In         | 모든 상위 태스크 성공 후 실행      |
|               | Unaligned Fan-In       | 상위 태스크 성공 여부 무관하게 실행   |
| 팬아웃(Fan-Out)  | Parallel Split         | 하나의 태스크에서 여러 브랜치 병렬 실행 |
|               | Exclusive Choice       | 조건에 따라 하나의 브랜치만 실행     |
| 오케스트레이션       | Single Runner          | 파이프라인 동시 실행 1개로 제한     |
|               | Concurrent Runner      | 파이프라인 동시 실행 허용         |

## 6.2 팬인(Fan-In)

- 패턴#35 정렬된 팬인(Aligned Fan-In)
- 패턴#36 비정렬 팬인(Unaligned Fan-In)
- 직전 서브챕터(시퀀스)는 태스크를 순서대로 잇는 패턴이었다.
- 팬인은 반대로 여러 갈래(브랜치)를 하나로 **합치는** 지점을 다룬다.



## 패턴#34 정렬된 팬인(Aligned Fan-In)

### (1)문제상황
- 결론: 하루치 방문 로그를 한 잡으로 돌렸더니, 한 시간만 깨져도 24시간을 전부 재처리해야 하는 고통이 있다.

- 기존 방식과 실패 장면:
    - `visits_raw`(원천 방문 이벤트 테이블, 시간 단위 파티셔닝)를 하루치 단일 잡으로 집계했다.
    - 새벽에 23시 파티션 하나가 밀려 잡이 터졌다.
    - 재실행하면 00시부터 다시 계산한다. 멀쩡한 23시간을 또 처리해 복구가 몇 시간씩 걸린다.
- 개선 방향:
    - 시간 파티셔닝은 이미 있으니, 시간별로 쪼개 병렬로 돌리고 싶다.
    - 그러면 깨진 그 한 시간만 다시 돌리면 된다.
- 쪼개니 생긴 새 문제 (이 패턴이 푸는 지점):
    - 시간별 태스크 24개로 나눴다. 각자 부분 집계를 만든다.
    - 그런데 소비자는 하루 전체 뷰만 원한다. 24개 결과를 하나로 합칠 병합 태스크가 필요하다.
    - 병합 태스크는 언제 돌아야 하나? → 24개가 **전부 성공한 뒤**에만 돌아야 한다. 하나라도 실패하면 불완전한 일별 뷰가 나온다.
    - 이 "전부 성공해야 시작" 조건이 곧 정렬된(aligned) 팬인이다.

엔지니어 독백:
> 이거 진짜 내가 데인 거라 말해준다. 첫 회사에서 방문 로그 일별 집계를 하루치 한 방에 돌렸거든. 
> 근데 꼭 새벽 4시에 23시 파티션이 밀려서 잡이 터진다. 문제는 그다음이야. 
> 재실행 누르면 00시부터 다시 처리해. 
> 앞 23시간 멀쩡한 걸 또 계산하느라 아침 9시까지 못 끝나고, 
> 그동안 대시보드는 어제 데이터로 멈춰 있어. 
> 상사는 계속 물어보지, 데이터는 안 나오지. 그때 선배가 "왜 하루를 한 덩어리로 돌려? 시간별로 쪼개"라고 하더라. 쪼개고 나니까 깨진 그 한 시간만 다시 돌리면 돼. 5분이면 복구된다. 
>신입이면 이거 하나만 기억해라. **실패는 무조건 난다. 중요한 건 터졌을 때 얼마나 작게 고칠 수 있냐다.**


### (2)솔루션
- 주요컨셉: "언제 합치나"는 오케스트레이션이, "어떻게 합치나"는 SQL/Spark가 푼다.
- 두 계층으로 갈리는 배경:
    - 파이프라인 관심사가 둘로 나뉜다.
    - 오케스트레이션 계층 : 태스크 순서·조건 조율 (Airflow)
    - 처리 계층 : 실제 데이터 변환·결합 (SQL 또는 Spark)
- 오케스트레이션 숙제 — "언제":
    - 24개 브랜치 전부 성공 후 병합 실행.
    - Airflow 기본 트리거 규칙이 "부모 전부 성공" → 연결만 하면 자동 정렬.
- 처리 숙제 — "어떻게":
    - 24개 부분 집계를 **SQL/Spark로** 하나로 합친다.
    - 합치는 연산은 두 개뿐 → UNION, JOIN.
- UNION : 세로로 쌓기 (같은 스키마를 행으로 이어붙임)

```
-- 0시 집계와 1시 집계를 세로로 쌓아 하루치로
SELECT hour, visits FROM agg_hour_00
UNION ALL
SELECT hour, visits FROM agg_hour_01
```

- 설명: 두 결과의 컬럼 구조가 같아야 한다. 행 수만 늘어난다(100+100=200행).
- `UNION ALL` : 중복 제거 없이 그대로 쌓음. `UNION`(중복 제거)보다 빠름 → 파티션이 안 겹치는 시간별 집계엔 `ALL`이 정답.
- JOIN : 가로로 붙이기 (다른 지표를 키로 결합)

```
-- 시간별 방문수 + 시간별 매출을 hour 키로 옆에 붙임
SELECT v.hour, v.visits, s.revenue
FROM hourly_visits v
JOIN hourly_sales s ON v.hour = s.hour
```

- 설명: 공통 키(`hour`)로 묶어 컬럼을 옆에 붙인다. 컬럼 수가 는다.
- 실무 선호:
    - 이 시나리오(24개 동일 스키마 집계) → **UNION ALL**. 같은 모양을 세로로 쌓는 거라.
    - 다른 지표 결합(방문수+매출) → JOIN.
    - 판단 기준: 같은 모양 쌓기 = UNION, 다른 축 붙이기 = JOIN.

- 엔지니어 독백:    
    > UNION 쓸 때 컬럼 순서 믿지 마라. 기본 UNION은 위치 기반이라 컬럼 순서만 어긋나도 엉뚱한 값이 섞여도 에러 없이 넘어간다. 24개 시간별 결과처럼 같은 쿼리 복붙이면 괜찮은데, 손으로 짠 브랜치를 합칠 땐 나는 무조건 이름 기반으로 맞춘다. 이거 프로덕션에서 한 번 데이면 데이터 틀어진 걸 며칠 뒤에야 안다.
    

### (3)결과

- 결론: 24개 브랜치를 병렬로 얻는 대신, 3가지를 감수해야 한다.
- 인프라 스파이크(Infrastructure spike):
    - 배경: 단일 잡은 자원을 순차로 쓴다. 24개로 쪼개면 동시에 24잡이 뜬다.
    - 문제: 순간 자원 요구가 24배로 튄다.
    - 대응: 탄력적 프로비저닝(elastic provisioning) 있으면 무시 가능. 없으면 동시 실행 수를 제한해 균형.
- 스케줄 스큐/오버헤드(Scheduling skew):
    - 배경: 브랜치마다 시작·완료 시점이 다르다.
    - 문제: 병합 태스크는 가장 느린 브랜치를 기다린다. 대기 오버헤드 발생.
- 브랜치 다수 시 복잡도:
    - 배경: 정렬된 팬인은 브랜치를 병합 태스크에 전부 연결한다.
    - 문제: 입력 브랜치가 수십 개면 그래프 관리·디버깅 부담 증가.

- 엔지니어 독백:
    > 신입한테. 24개 동시 실행이 멋져 보이지만, 자원 한도 안 보고 브랜치 늘리면 클러스터가 그 순간 다 잡아먹혀서 옆 팀 잡까지 밀린다. 나는 브랜치 수 늘릴 때 항상 max active tasks부터 확인한다. 병렬은 공짜가 아니다.
    

### (4)예시

- 결론: 오케스트레이션 계층(Airflow)과 처리 계층(PySpark) 각각의 구현을 책 예시로 본다.
- 도메인: 블로그 방문 로그. 시간별 CSV 파티션(`date=.../hour=00/dataset.csv`)을 읽어 일별 집계.


- 예시 1) 오케스트레이션 구현 (Example 6-6) — for 루프로 24개 브랜치 동적 생성
- 한 줄 요약: `for` 루프가 시간별 태스크 24개를 만들고, 전부 공통 병합 태스크에 연결한다.

```
clear_context = PostgresOperator(...)
generate_trends = PostgresOperator(...)

for hour_to_load in [f"{hour:02d}" for hour in range(24)]:
    file_sensor = FileSensor(
        task_id=f'wait_for_{hour_to_load}',
        filepath=input_dir + '/date={{ ds_nodash }}/hour=' + hour_to_load + '/dataset.csv',
    )
    visits_loader = PostgresOperator(
        task_id=f'load_hourly_visits_{hour_to_load}',
        params={'hour': hour_to_load},
    )
    # ★ 핵심: 24개 브랜치가 모두 generate_trends로 수렴
    clear_context >> file_sensor >> visits_loader >> generate_trends
```

- 실행 순서:
    - ① `clear_context` : 집계 전 대상 테이블 초기화.
    - ② 시간별로 `file_sensor`(해당 시간 CSV 대기) → `visits_loader`(그 시간 적재).
    - ③ 24개 `visits_loader` 전부 성공 시 → `generate_trends`(일별 병합) 실행.
- ★ 핵심: 마지막 줄. 24개 브랜치가 전부 `generate_trends`에 연결돼, Airflow 기본 규칙("부모 전부 성공")이 정렬을 자동 보장.
- 각주:
    - `range(24)` : 0~23시. `f"{hour:02d}"`는 `00`~`23` 2자리 포맷.
    - 정적 리스트면 `[load_1, load_2] >> merge` 형태로도 가능. 더 장황하지만 연결이 명시적이라 가독성 좋음.



- 예시 2) 처리 구현 (Example 6-7) — PySpark UNION
- 한 줄 요약: 두 시간별 결과 DataFrame을 이름 기준으로 세로로 합친다.

```
input_dataset_1: DataFrame = ...   # 예: 0시 집계
input_dataset_2: DataFrame = ...   # 예: 1시 집계
# ★ 핵심: unionByName = 컬럼 '이름' 기준 결합
output_dataset = input_dataset_1.unionByName(input_dataset_2)
```

- ★ 핵심: `unionByName`.
    - 배경: 기본 `union`은 컬럼 '위치' 기준이라, 순서만 어긋나도 엉뚱한 컬럼끼리 합쳐진다(에러 없이).
    - 해결: `unionByName`은 컬럼 이름을 맞춰 결합 → 위치 실수 방지.



- 예시 3) 위치 기반 함정 (Example 6-8) — 기본 UNION의 위험

```
SELECT a, b, c FROM abc
UNION
SELECT c, b, a FROM cba
```

- 설명: 위치 기반이라 `a+c`, `b+b`, `c+a`로 잘못 결합됨. 에러 없이 오염된 데이터 생성.
- 그래서 Spark는 `unionByName`을 별도 제공. 단 덜 알려져 기본 `UNION` 습관을 주의.

> 뭔말?

 - 결론: 기본 UNION은 **컬럼 이름이 아니라 위치(순서)로 합쳐서**, 순서만 어긋나면 엉뚱한 값이 조용히 섞인다.
- 왜 위험한가 (배경):
    - 초기 SQL UNION은 성능 때문에 단순 설계됐다. 컬럼 이름을 대조하지 않고 왼쪽부터 순서대로 짝지음.
    - 그래서 두 쿼리의 컬럼 순서만 달라도 에러 없이 잘못 합쳐진다.
- 예시로 보자:
    - 테이블 A(`user_id, age`) : 값 `(1, 30)`
    - 테이블 B(`age, user_id`) : 값 `(25, 2)` ← 컬럼 순서가 반대
- 의도: `user_id`끼리, `age`끼리 합치기.
- 기본 UNION 결과 (위치 기준, 1번째끼리·2번째끼리):

```
user_id, age
1,  30      ← A: 정상
25, 2       ← B: age(25)가 user_id 자리로, user_id(2)가 age 자리로 뒤바뀜
```

- 문제의 본질:
    - `user_id=25, age=2`라는 말도 안 되는 행이 생겼는데 **에러가 안 난다.**
    - 나이 2세, 유저ID 25... 다운스트림에서 며칠 뒤에야 발견된다.
- 해결:
    - Spark `unionByName` : 위치 무시, **컬럼 이름**으로 맞춰 합침. → 순서 반대여도 `user_id`끼리 정상 결합.
- 엔지니어 독백:
    
    > 신입한테. 같은 `SELECT` 복붙해서 UNION하면 순서가 같으니 안 터진다. 근데 브랜치마다 손으로 컬럼 짠 걸 합칠 땐 무조건 `unionByName`이다. 위치 기반은 틀려도 조용해서, 사고 나면 원인 찾는 데만 하루 날린다.
---

- 엔지니어 독백:
    > 신입한테. `for` 루프로 태스크 찍어내는 패턴은 Airflow에서 제일 흔한 관용구다. 근데 루프 안에서 `task_id`를 고유하게 안 만들면(예: `wait_for_{hour}` 없이) 태스크가 덮어써져 24개가 1개로 뭉개진다. 나는 루프 태스크 짤 때 `task_id`에 반복 변수 박혔는지부터 확인한다. 그리고 `unionByName`은 무조건 손에 익혀둬라. 위치 기반 union으로 데이터 틀어지면 며칠 뒤에나 발견되고, 그땐 이미 다운스트림 다 오염돼 있다.




### (5)최신트렌드
- 결론: 정적 `for` 루프 브랜치의 한계를 3가지 도구가 각각 보완한다.
- Airflow Dynamic Task Mapping (2.3+):
    - 배경: 기존 `for` 루프는 브랜치 수가 코드에 고정된다. 파티션 수가 런타임에 바뀌면(예: 어떤 날 20시간치만 도착) 대응 불가.
    - 해결: `.expand()`가 런타임 입력 개수만큼 태스크를 동적 생성. 팬인 브랜치 수가 유동적일 때 적합.
- Spark AQE(Adaptive Query Execution, 3.0+):
    - 배경: UNION/JOIN 후 파티션 수가 고정돼, 데이터 쏠림(skew)·과다 파티션이 잦았다.
    - 해결: 런타임 통계로 파티션을 자동 병합·스큐 조정. 팬인 병합 성능을 자동 최적화.
- dbt:
    - 배경: SQL 팬인을 수동 오케스트레이션하면 의존 관계가 코드에 안 드러난다.
    - 해결: 모델 간 `ref()`로 DAG 자동 구성. 부분합 → 최종 집계 의존을 선언적으로 표현.
- 실무 선호:
    - 이미 Airflow 사용, 브랜치 수 유동적 → Dynamic Task Mapping.
    - Spark 집계 성능 병목 → AQE 먼저 켬. 설정 한 줄이라 도입 비용 최저.
    - SQL 중심 팀 → dbt로 의존 선언.
- 엔지니어 독백:
    
    > 신입한테. AQE는 Spark 3 쓰면 그냥 켜라. `spark.sql.adaptive.enabled` 하나 켜는 걸로 스큐 잡히는 경우가 많다. 나는 새 클러스터 세팅할 때 이거 기본으로 넣는다. 반대로 Dynamic Task Mapping은 좋아 보여도 브랜치 수가 진짜 매번 바뀔 때만 써라. 고정 24시간이면 그냥 `for` 루프가 읽기 쉽다. 도구는 문제 생겼을 때 넣는 거다.
    





## 패턴#35 비정렬 팬인(Unaligned Fan-In)

### (1)문제상황

- 결론: 시간 하나 실패하면 그날 집계가 통째로 안 나와서, 불완전하더라도 먼저 내보내고 싶은 상황이다.
- 기존 방식과 실패 장면:
    - 정렬된 팬인으로 24개 시간별 브랜치를 돌려왔다.
    - 규칙상 24개 전부 성공해야 병합이 돈다.
    - 몇 번, 한 시간이 처리 실패했다. 그 결과 그날 집계 뷰가 아예 안 나갔다.
    - 다운스트림 소비자는 23시간치 멀쩡한 데이터도 못 받았다.
- 개선 욕구:
    - 회의 결과: 부분 데이터라도 먼저 내보내고, 빈 시간은 나중에 채우자로 합의.
- 완화하니 생긴 새 문제 (이 패턴이 푸는 지점):
    - 실패를 허용하고 병합을 돌리면 → 소비자는 이게 완전한 데이터인지 부분인지 모른다.
    - "23/24시간만 반영됨" 같은 완성도 표시가 없으면 소비자가 완전한 뷰로 오해한다.
    - 즉 "전부 성공" 조건을 푸는 대신, 완성도를 알리는 장치가 필요하다.
- 엔지니어 독백:
    
    > 신입한테. 부분 데이터 허용은 편해 보이지만, 소비자한테 "이거 부분이야"를 안 알리면 그게 더 큰 사고다. 완전한 줄 알고 매출 리포트 냈다가 나중에 3시간 빠진 거 들통나면, 차라리 데이터가 아예 없던 게 나았다는 소리 듣는다. 완화할 거면 완성도 플래그는 무조건 세트로 간다.
    

### (2)솔루션

- 주요컨셉: orchestrator는 trigger rule을 `ALL_SUCCESS`에서 `ALL_DONE`으로 완화해 partial 데이터를 만들고, 처리 계층은 그 partial 여부를 `is_partial` 플래그로 소비자에게 알린다.
- 1단계 — trigger rule 완화:
    - 배경: `ALL_SUCCESS`(Airflow 기본값)는 부모 전부 성공을 요구해, 한 브랜치 fail이 그날 집계 전체를 막았다.
    - `ALL_DONE` 정의: 부모가 성공이든 fail이든 **끝나기만** 하면 자식 실행. success만 세지 않음.
    - 결과: 23개 success + 1개 fail이어도 병합 진행.
- 2단계 — 왜 완성도 플래그가 필수인가:
    - `ALL_DONE`의 부작용: 소비자는 도착한 데이터가 complete인지 partial인지 구분 불가.
    - 소비자는 데이터가 오면 complete로 가정 → partial 데이터로 잘못된 결론.
    - 따라서 partial 여부를 알리는 장치 필수.
- 예시 (완성도 표시):
    - `visits_raw`에서 처리된 시간 수를 센다.
    - 24개 다 처리 → `is_partial = false`
    - 20개만 처리 → `is_partial = true` (4시간 누락)
    - 소비자는 `WHERE is_partial = false`로 partial 데이터 배제.
- 3단계 — 완성도 알림 방식 (선택지):
    - flag 컬럼(`is_partial`) : 결과 테이블에 컬럼 추가.
    - metadata tag : object storage 객체 태그로 부착. 데이터 본문 미변경.
    - downstream 알림 : 이메일 등으로 partial 상태 통보.
    - 비공개 : complete 아니면 외부 미공개, 내부 테이블만 기록.
- 실무 선호:
    - 소비자가 SQL 필터링 → `is_partial` flag 컬럼. `WHERE is_partial = false`로 즉시 배제.
    - 파일 기반 lake → metadata tag. 데이터 재작성 없이 상태만 갱신.


### (3)결과

- 결론: fail을 허용한 대가로 단점 2가지가 생긴다.
- 가독성 저하:
    - 배경: `ALL_SUCCESS` 기본값일 땐 그래프만 봐도 "전부 성공해야 다음"이 읽혔다.
    - 문제: `ALL_DONE` 완화는 코드 속 옵션에만 있고 그래프엔 안 보인다. success 태스크 + fail 태스크를 둘 다 붙이면 흐름이 더 꼬인다.
    - 대응: fail 전용 훅 사용. Airflow `on_failure_callback`, AWS Step Functions `Catch`
- partial 데이터 완성도 누락:
    - 배경: 소비자는 데이터가 오면 complete로 가정한다.
    - 문제: partial임을 안 알리면 소비자가 불완전 데이터로 잘못된 결론.
    - 대응: `is_partial` / `is_approximate` 플래그 필수 부착.
- 엔지니어 독백:
    
    > 이거 내가 실제로 당한 썰 하나 푼다. 예전에 `ALL_DONE`으로 완화해놓은 DAG를 딴 사람이 만들어놨는데, 나는 그것도 모르고 "왜 3시간 fail 났는데 집계가 나가지?" 하고 반나절을 팠다. 그래프엔 아무 표시도 없거든. 결국 코드 열어서 trigger rule 한 줄 보고 알았다. 그날 이후로 나는 `ALL_DONE` 쓰면 태스크 이름을 `merge_allow_partial`처럼 대놓고 바꾸거나 doc에 박아둔다. 완화 규칙은 숨으면 무조건 사고 난다. 그리고 `is_partial` 플래그 없이 partial 내보내는 건 진짜 하지 마라 — 완전한 줄 알고 리포트 나간 다음에 "사실 4시간 빠졌어요" 하는 순간 신뢰가 끝난다.



### (4)예시

- 결론: 트리거 규칙 한 줄 바꾸고, 완성도 플래그를 SQL로 계산한다.
- 예시 1) 트리거 완화 (Example 6-9, Airflow)

```
generate_cube = PostgresOperator(
    trigger_rule=TriggerRule.ALL_DONE   # ★ 핵심
)
for hour_to_load in [f"{hour:02d}" for hour in range(24)]:
    file_sensor = FileSensor(task_id=f'wait_for_{hour_to_load}', ...)
    visits_loader = PostgresOperator(task_id=f'load_hourly_visits_{hour_to_load}', ...)
    clear_context >> file_sensor >> visits_loader >> generate_cube
```

- ★ 핵심: `trigger_rule=TriggerRule.ALL_DONE`.
    - 배경: Airflow 기본값은 `ALL_SUCCESS`(부모 전부 성공해야 시작).
    - 동작: `ALL_DONE`은 성패 무관, 부모가 전부 '끝나기만' 하면 자식 실행.
    - 함정: 규칙이 코드에만 있어 그래프론 안 보임.




- 예시 2) 완성도 플래그 계산 (Example 6-10, SQL)
결론: 이 SQL은 **집계 결과를 저장하면서, "24시간 다 처리됐는지"를 세어 `is_approximate` 값으로 같이 박는 코드**다.

한 줄 요약:
- 집계(cube)를 만들면서, 처리된 시간 수를 세서 24 미만이면 `is_approximate = true`를 붙인다.
```
INSERT INTO dedp.visits_cube (..., is_approximate)
SELECT ...,
  (SELECT CASE WHEN cnt.all_hours = 24 THEN false ELSE true END
   FROM (SELECT COUNT(DISTINCT execution_time_hour_id) AS all_hours
         FROM dedp.visits_raw
         WHERE execution_time_id = '{{ ds }}') AS cnt) -- ★ 핵심
FROM dedp.visits_raw GROUP BY CUBE(...);
```

- 실행 순서:
    - ① 안쪽 서브쿼리 : 그날(`ds`) `visits_raw`에 실제 들어온 시간 수를 센다. `COUNT(DISTINCT execution_time_hour_id)`.
    - ② `CASE` : 그 수가 24면 `false`(complete), 아니면 `true`(partial).
    - ③ 바깥 `SELECT` : 집계 결과를 만들면서 각 행에 ②의 값을 `is_approximate` 컬럼으로 붙인다.
    - ④ `INSERT` : 완성된 결과를 `visits_cube`에 저장.
- ★ 핵심: 안쪽 서브쿼리.
    - 여기서 "몇 시간 처리됐나"를 세는 게 partial 판정의 전부다. 이 한 덩어리가 완성도를 결정한다.
- 각주 (용어):
    - `execution_time_hour_id` : 시간 파티션 식별자. 24개면 0~23시 전부 도착.
    - `{{ ds }}` : Airflow 매크로. 실행일자(`YYYY-MM-DD`)로 자동 치환. 내가 정하는 변수 아님.
    - `GROUP BY CUBE(...)` : 여러 컬럼 조합의 소계를 한 번에 만드는 집계 방식. 배경 — 조합별로 쿼리 여러 번 돌리던 걸 한 번에 처리하려고 나옴.
```
execution_time_id | execution_time_hour_id | visits
2025-07-04        | 00                     | 50
2025-07-04        | 01                     | 60
2025-07-03        | 23                     | 40   
```


- 예시 데이터 (24시간 다 처리된 날 = complete):
```
execution_time_id | page   | country | visits | is_approximate
2025-07-03        | /home  | KR      | 1200   | false
2025-07-03        | /blog  | US      | 340    | false
```

- 예시 데이터 (20시간만 처리된 날 = partial):
```
execution_time_id | page   | country | visits | is_approximate
2025-07-04        | /home  | KR      | 980    | true    ← 4시간 누락, 값 불완전
2025-07-04        | /blog  | US      | 260    | true
```

- consumer 활용:
    - `WHERE is_approximate = false` → complete 데이터만 조회.
    - 위 예시면 7/4 행은 자동 배제됨.




- 예시 3) 덜 선언적인 구현 (Example 6-11, AWS Step Functions + Lambda)
결론: **Airflow의 `ALL_DONE`을 못 쓰는 환경(AWS Step Functions)에서,
partial 여부를 코드로 직접 판단**하는 예시
```
# lambda-table-creator
def lambda_handler(event, context):
    table_metadata = {}
    if False in event['ProcessorResults']:   # ★ 핵심: 실패 하나라도 있으면
        table_metadata['is_partial'] = True  # 부분 플래그 부착
    return True
```
<1단계>
- 배경:
    - Airflow는 `ALL_DONE` 한 줄로 "fail 나도 진행"을 선언적으로 처리했다.
    - AWS Step Functions에는 그런 trigger rule 옵션이 없다.
    - 그래서 "일부 실패해도 진행 + partial 표시"를 **코드(Lambda)로 직접** 구현해야 한다.
- 등장 컴포넌트 (Lambda 3개, 각 역할):
    - Lambda ①(partition detector) : 처리할 시간 파티션들을 찾는다.
    - Lambda ②(processor) : 파티션별로 처리하고, 성공/실패를 `true`/`false`로 반환한다.
    - Lambda ③(table-creator) : ②의 결과들을 모아, 하나라도 `false`면 `is_partial=true`를 붙인다.
- 위 코드(`lambda_handler`)의 정체:
    - 세 번째 Lambda ③(table-creator)다.
    - `event['ProcessorResults']` : ②가 반환한 성패 리스트(예: `[true, true, false, ...]`).
    - `if False in ...` : 그 리스트에 `false`가 하나라도 있으면 partial로 판정.

<2단계>
- 입력 (`event`):

```
event = {
    "ProcessorResults": [true, true, false, true]   -- Lambda ②가 반환한 파티션별 성패
}
```

- 실행 순서:
    - ① `table_metadata = {}` : 빈 메타데이터 딕셔너리 생성.
    - ② `if False in event['ProcessorResults']` : 성패 리스트에 `false`가 있는지 검사. 위 예시는 3번째가 `false` → 참.
    - ③ `table_metadata['is_partial'] = True` : partial로 판정, 플래그 부착.
    - ④ `return True` : 처리 완료 신호 반환.
- ★ 핵심: `if False in event['ProcessorResults']`
    - 이 한 줄이 partial 판정의 전부.
    - Airflow `ALL_DONE`은 orchestrator가 자동 판정했지만, Step Functions엔 그 기능이 없어 **리스트에 fail이 있나를 코드로 직접 검사**한다.
- 두 방식 비교:
    - Airflow : `trigger_rule=ALL_DONE` 선언 → orchestrator가 판정 (선언적)
    - Step Functions : `if False in ...` → 코드가 판정 (명령적, 덜 선언적)
- 엔지니어 독백:

    > 신입한테. Step Functions 같은 환경엔 Airflow의 trigger rule 같은 편의 기능이 없다. 그래서 "실패 허용" 로직을 매번 Lambda에 손으로 짠다. 이런 코드 볼 때 나는 "이게 orchestrator가 해줄 일을 대신하는구나"로 읽는다. 도구가 안 해주면 사람이 짜는 거다.

- 엔지니어 독백:
    > 신입한테. `ALL_DONE`은 그래프 봐선 절대 안 보인다. 나는 이런 완화 규칙 쓸 때 태스크 이름이나 doc_md에 "부분 허용" 흔적을 꼭 남긴다. 안 그러면 6개월 뒤 내가 봐도 "얘는 왜 실패했는데 다음이 돌지?" 하고 헤맨다.
    





### (5)최신트렌드
엔지니어 독백:

> 신입한테 요즘 현업 분위기 말해준다.
> 
> 예전엔 partial 여부를 `is_partial` 컬럼으로 매 row에 박았다. 이게 진짜 별로였다. consumer가 `WHERE` 한 줄 까먹으면 그냥 partial 데이터로 리포트 나간다. 사람 실수에 의존하는 구조라 늘 불안했다.
> 
> 요즘은 Delta Lake나 Iceberg를 많이 쓴다. 왜 좋냐면, 완성도를 row가 아니라 **write(commit/snapshot) 단위**로 붙일 수 있다. "이 write 자체가 partial이다"가 데이터에 딱 박히니까, consumer가 컬럼 필터 안 걸어도 놓칠 일이 없다. 기존 컬럼 방식의 "놓치기 쉬움"을 구조적으로 없앤 거라 체감이 크다.
> 
> 그리고 요즘 트렌드는 partial을 **표시**하는 걸 넘어 아예 **막는** 쪽이다. Great Expectations나 dbt test를 파이프라인에 꽂아두면 "24시간 안 채워지면 run을 fail시켜라"를 걸 수 있다. partial이 consumer한테 도달조차 안 한다. 나는 매출 같은 민감 데이터는 무조건 이걸로 막는다. 반쪽 데이터가 대시보드에 뜨는 순간 그걸로 의사결정 나버리거든.
> 
> 정리하면 — 표시만 필요하면 Delta/Iceberg metadata, 아예 막아야 하면 test gate. 요즘은 "믿을 만한 데이터만 내보낸다"는 쪽으로 무게가 실린다.



- 결론: 요즘은 completeness를 row 플래그가 아니라 **table 메타데이터** 또는 **test gate**로 관리한다.
- Delta Lake / Iceberg 메타데이터:
    - 이전 한계: `is_partial` per-row column은 consumer가 `WHERE` 필터를 까먹으면 partial 데이터가 그대로 새어나갔다.
    - 왜 요즘 쓰나: completeness를 write(commit/snapshot) 단위로 붙여, row 필터 없이도 write 자체가 partial인지 판별된다.
- Great Expectations / dbt tests:
    - 이전 한계: partial 여부를 consumer가 매번 수동 확인했다.
    - 왜 요즘 쓰나: "24 hours 완비" 같은 조건을 test로 걸어, 미충족 시 run을 fail시켜 partial을 아예 차단한다. 표시(flag)가 아니라 차단(gate).
- 실무 선호 (상황별):
    - consumer가 partial도 보되 구분만 필요 → Delta/Iceberg 메타데이터. write 단위 상태가 native라 도입 비용 최저.
    - partial을 아예 내보내면 안 됨(매출 등 민감 데이터) → GE/dbt test gate.

> 저게 뭐지?


Great Expectations:
- 정체: Python 기반 data validation 프레임워크.
- 이전 한계: 데이터 검증을 사람이 SQL로 매번 눈으로 확인했다. 누락·중복을 놓치기 쉬웠다.
- 하는 일: "이 컬럼은 null 없어야 함", "row 수 24개여야 함" 같은 expectation을 코드로 선언 → 자동 검사.

dbt tests:
- 정체: SQL 변환 도구 dbt(data build tool)에 내장된 test 기능.
- 이전 한계: SQL 변환 결과의 정합성을 사람이 사후 확인했다.
- 하는 일: 모델에 `unique`, `not_null`, `row_count` 같은 test를 붙여 → 빌드 시 자동 검사, 실패 시 run 중단.

이 패턴에서의 역할:
- "24 hours 완비" 조건을 test로 건다.
- 미충족이면 run을 fail → partial 데이터가 consumer로 못 나감.
- 즉 flag(표시)가 아니라 gate(차단) 방식.


- 엔지니어 독백:
    
    > 신입한테. flag는 "보이게", gate는 "못 나가게"다. 매출처럼 잘못 나가면 끝나는 데이터는 무조건 gate로 막아라. partial이 대시보드에 뜨는 순간 그걸로 의사결정 나버린다. 반대로 로그 분석처럼 partial도 쓸모 있으면 metadata로 열어두고 consumer가 판단하게 둔다. 요즘 분위기는 "믿을 데이터만 내보낸다"로 기운다.
    
