데이터셋을 다른 파이프라인과 공유하면 그 가치를 크게 향상할 수 있다. 하지만 데이터셋에 개인 식별 정보(PII) 등의 민감 정보가 포함되어 있고 사용자가 제3자 공유에 동의하지 않았다면, 데이터 공유 전 특별한 준비 단계를 수행해야 한다.

## 패턴 #46: 익명화기 (Anonymizer)

### 1\. 문제 정의

-   조직에서 고객 행동 분석 및 커뮤니케이션 전략 최적화를 위해 외부 데이터 분석 회사와 계약을 체결했다.
-   공유 데이터셋에 다수의 개인 식별 정보(PII) 속성이 포함되어 있으나, 일부 사용자가 제3자 공유에 동의하지 않은 상태이다.
-   데이터 엔지니어링 팀은 공유 데이터셋이 개인정보 보호 규정을 완전히 준수하도록 처리 파이프라인을 구축해야 하는 임무를 맡았다.

> **NOTE: 개인 식별 정보(PII)만이 아니다** PII가 가장 흔하게 논의되는 사례이지만, 보호해야 할 대상은 이외에도 보호 건강 정보(PHI), 지적 재산(IP) 데이터 등이 존재한다. (본 문서에서는 간결함을 위해 PII로 통일하여 기술한다.)

### 2\. 해결책 및 구현 방식

익명화기 패턴의 목표는 데이터셋에서 민감한 데이터를 제거하거나 변환하여 각 레코드를 익명 정보로 바꾸는 것이다. 이를 통해 데이터 컨슈머가 특정 사용자를 식별할 수 없도록 만든다.

익명화기 패턴은 크게 3가지 구현 방식을 지원한다:

1.  **데이터 제거 (Data Removal)**
    -   선택한 민감 컬럼을 입력 데이터셋에서 완전히 삭제하는 방식으로, 구현이 가장 간단하다.
2.  **데이터 변조 (Data Perturbation)**
    -   입력 값에 노이즈를 추가하여 원본과 다른 의미를 갖도록 만든다.
    -   _예시_: IP 주소 컬럼의 무작위 위치에 속성을 더해 123.456.789.012를 1823.456.7809.012로 변형한다.
3.  **합성 데이터 대체 (Synthetic Data Replacement)**
    -   원래 값을 머신러닝 모델 등 스마트 모델 기반의 합성 데이터 생성기에서 나오는 값으로 치환한다.
    -   대체 값은 원본과 동일한 속성을 가진 것처럼 보이지만 실제 값은 다르다.
    -   _예시_: 국가 컬럼에서 Portugal을 합성적으로 Croatia로 대체한다.

데이터 제거와 변조는 매핑 함수나 컬럼 변환으로 비교적 쉽게 구현할 수 있다. 반면 합성 데이터 대체 방식은 데이터 과학 팀의 모델 구축 작업이 필요할 수 있으며, 덜 이상적인 형태로는 무작위 값 생성 대체 함수를 만들어 구현할 수도 있다.

### 3\. 패턴의 결과 및 트레이드오프

-   **보안성**: 실제로 민감한 데이터를 완벽하게 보호한다.
-   **정보 유실 (Information Loss)**: 데이터의 사용성에 큰 영향을 미친다. 민감 컬럼이 제거되거나 대체됨에 따라 데이터 분석가나 데이터 과학자 등 최종 사용자가 분석에 이 컬럼들을 활용할 수 없다.이는 잘못된 데이터 예측 모델이나 부정확한 비즈니스 인사이트로 이어질 위험이 있다.

### 4\. 예제 (PySpark & Faker)

생일 컬럼을 제거하고, 이메일 주소를 Faker 라이브러리를 통해 낮은 품질의 합성 데이터로 대체하는 예제이다.

-   **컬럼 제거**: 읽을 컬럼이 적으면 SELECT 문에서 생략하고, 그렇지 않다면 drop 함수를 사용한다.
-   **값 대체**: 컬럼 기반 함수(withColumn) 또는 레코드 기반 함수(mapInPandas)를 사용한다.

```
# 예제 7-22: 파이스파크에서 컬럼 삭제 및 익명화 예제 대체
@pandas_udf(StringType())
def replace_email(emails: pandas.Series) -> pandas.Series:
    faker_generator = Faker()
    return emails.apply(lambda email: faker_generator.email())

# birthday 컬럼 삭제 후 email 컬럼을 무작위 생성 이메일로 대체
users.drop('birthday').withColumn('email', replace_email(users.email))
```

-   **결과**: 출력 데이터셋에서 birthday 컬럼은 삭제되며, email 값은 무작위 생성 이메일 주소로 완전히 대체된다.

## 패턴 #47: 의사 익명화기 (Pseudo-Anonymizer)

### 1\. 문제 정의

-   익명화기 패턴은 강력한 보호를 제공하지만, 데이터 누락/변경으로 인해 데이터 분석 및 과학 파이프라인에 미치는 부정적 영향이 매우 크다.
-   외부 분석 회사에 공유된 익명화 데이터셋에서 주요 컬럼들이 제거되어, 팀이 비즈니스 쿼리에 답할 수 없는 문제가 발생했다.
-   이에 따라 실제 PII 값은 숨기되, 분석 및 비즈니스 용도로 더 유용하게 활용할 수 있는 형태로 대체된 데이터셋이 필요해졌다.

### 2\. 해결책 및 구현 방식

의사 익명화기 패턴은 원본과 어느 정도 관련이 있는 다른 값으로 초기 데이터를 대체하여 데이터의 비즈니스 의미를 유지한다.

상황에 따라 다음 4가지 방식을 사용한다:

1.  **데이터 마스킹 (Data Masking)**
    -   민감 데이터를 의미 없는 문자나 현실적인 대체 값으로 가린다.
    -   _예시_: 사회보장번호(SSN) 999-55-1040을 XXX-XX-1040 또는 9XX-5X-1XXX로 마스킹한다. (단, 이 경우 서로 다른 사용자가 동일한 마스킹 SSN을 가질 수 있다.)
2.  **데이터 토큰화 (Data Tokenization)**
    -   초기 값을 가상 값으로 대체하고, 원본과 대체 값 간의 매핑 정보를 보안이 강화된 '토큰 볼트(Token Vault)'에 저장한다.
    -   볼트에 대한 접근 권한이 손상되면 권한 없는 자가 토큰화된 값을 되돌릴 수 있으므로 볼트 보안 관리가 핵심이다.
3.  **해싱 (Hashing)**
    -   민감 값을 복원할 수 없는 단방향 값으로 완전히 대체한다.
    -   _예시_: contact@waitingforcode.com을 SHA-256 알고리즘 및 Base64 인코딩을 거쳐 gd0B+pUpXYVZ9nqhgLRuban0CiLZRKVp4dcmvmocsYE=로 변환한다.
4.  **암호화 (Encryption)**
    -   암호화 키에 의존하여 컬럼/레코드 단위로 적용하며, 암호화 키 접근 권한이 있는 사용자만 원본을 복원할 수 있다.

> **NOTE: 익명화 vs 의사 익명화**
> 
> 의사 익명화된 PII 데이터는 **다른 데이터셋과 결합될 경우 사용자가 다시 식별될 가능성**이 존재한다. 반면 완전한 익명화 데이터셋은 다른 데이터와 결합하더라도 식별 불가능한 상태가 유지된다.

### 3\. 결과 및 한계점

-   **잘못된 보안 인식 (데이터 결합에 의한 식별 위험)**
    -   의사 익명화는 보장 정도가 상대적으로 약하다.
    -   의사 익명화된 테이블들을 서로 결합(Join)할 때 특정 인물을 식별해내는 결합 식별 문제가 발생할 수 있다.

### 4\. 예제 

-   country (지리적 영역으로 일반화)와 ssn (데이터 마스킹)은 mapInPandas를 통해 변환한다.
-   salary는 정수형에서 문자열 범주형(0-50000 등)으로 변환되므로 타입 호환성을 위해 withColumn 기반 컬럼 매핑을 별도로 적용한다.

```
# 예제 7-24: 일반화 및 데이터 마스킹을 통한 가명 처리
def pseudo_anonymize_users(input_pandas: pandas.DataFrame) -> pandas.DataFrame:
    def pseudo_anonymize_country(country: str) -> str:
        countries_area_mapping = {
            'Poland': 'eu', 'France': 'eu', 'Spain': 'eu', 'the USA': 'na'
        }
        return countries_area_mapping[country]

    def pseudo_anonymize_ssn(ssn: str) -> str:
        return f'{ssn[0]}***-{ssn[5]}***-{ssn[10]}***'

    for rows in input_pandas:
        rows['country'] = rows['country'].apply(lambda c: pseudo_anonymize_country(c))
        rows['ssn'] = rows['ssn'].apply(lambda ssn: pseudo_anonymize_ssn(ssn))
        yield rows

# 예제 7-25: 유형 변환을 사용한 컬럼 기반 의사 익명화
pseud_anonymized_users = (
    users.mapInPandas(pseudo_anonymize_users, users.schema)
    .withColumn('salary', functions.expr('''
        CASE WHEN salary BETWEEN 0 AND 50000 THEN "0-50000"
             WHEN salary BETWEEN 50000 AND 60000 THEN "50000-60000"
             ELSE "60000+" END'''))
)
```

데이터를 보호하는 것만으로는 충분하지 않다. 데이터는 동일한 시스템 내 또는 서로 다른 시스템 간에 지속적으로 흐르며, 시스템은 이 데이터에 지속적으로 접근해야 한다. 따라서 안전한 연결을 보장하기 위한 보안 접근 전략이 필수적이다.

## 패턴 #48: 비밀 포인터 (Secrets Pointer)

### 1\. 문제 정의

-   **사용 사례**: 방문 실시간 처리 파이프라인에서 각 이벤트에 지리적 위치 정보를 추가(풍부화)하기 위해 외부 API를 활용한다. 해당 API의 유일한 인증 방식은 로그인/패스워드 쌍이다.
-   **발생했던 문제**: 과거 팀에서 실수로 다른 API에 사용되는 로그인/패스워드를 공유하는 사고가 발생했다. API가 요청별로 과금되는 방식이었기 때문에 유출로 인한 청구 금액이 크게 증가했다.
-   **목표**: 새로운 데이터 풍부화 API와 상호작용하는 코드에 로그인/패스워드를 직접 저장하는 위험을 피하고자 한다.

### 2\. 해결책 및 작동 방식

자격 증명(Credentials)은 민감한 매개변수이다. 가장 좋은 보안 방법은 자격 증명을 코드나 파일 어디에도 직접 저장하지 않고 참조(포인터)를 사용하는 것이다.

-   **비밀 관리자(Secrets Manager) 서비스 활용**: 구글 클라우드 Secret Manager나 AWS Secrets Manager와 같은 중앙 관리 서비스를 활용하여 로그인, 패스워드, API 키 등의 민감한 값을 저장한다.
-   **장점**:
    1.  **접근 모니터링 용이**: 중앙화된 장소에서 민감 데이터를 관리하므로 접근 제어 및 모니터링이 쉬워진다.
    2.  **구성 요소 관리 용이**: 컨슈머를 일일이 수정할 필요 없이 관리자 서비스에서 새로운 자격 증명 셋을 쉽게 설정할 수 있다.
    3.  **환경별 관리 간소화**: 테라폼(Terraform)과 같은 IaC(Code로서의 인프라) 스택 이용 시 모든 환경(개발, 검증, 운영 등)에서 동일한 비밀 이름을 유지할 수 있어 환경별 매개변수를 다루는 복잡함이 줄어든다.
-   **접근 보호의 2가지 수준**:
    -   **1단계**: 컨슈머가 비밀 관리자 서비스 자체에 접근할 수 있는지 권한을 검증한다.
    -   **2단계**: 검색된 자격 증명 자체를 통해 대상 API나 데이터베이스 접근 가능 여부를 보장한다.

### 3\. 결과 및 고려사항 (트레이드오프)

-   **캐싱 무효화(Cache Invalidation) 문제**:
    -   통신 비용을 줄이고 실행 시간을 최적화하기 위해 자격 증명을 로컬에 캐싱할 수 있다.
    -   다만 자격 증명이 새로 고쳐졌을 때 캐시가 무효화되지 않으면 연결 오류가 발생한다.
    -   가장 간단한 해결책은 자격 증명 변경 시 작업이 실패하도록 두고, 재시작 시 새 자격 증명을 비밀 관리자에서 다시 로드하게 만드는 것이다. (단, 실패가 잦아질 수 있으므로 재시도 시에도 데이터 올바름을 유지하는 **멱등성 디자인 패턴**을 함께 고려해야 한다.)
    -   비동기 새로 고침 프로세스를 사용하는 경우에도 매개변수를 바꾸기 전에 데이터를 쓰기 시작하면 출력 데이터 스토어 쓰기 오류가 발생할 수 있다.
-   **잘못된 보안 인식 및 로그 유출 위험**:
    -   비밀 관리자에 저장했다고 해서 안전하다는 착각을 하면 안 된다. 실수로 애플리케이션 로그에 자격 증명을 출력하도록 작성하면 여전히 비밀이 유출될 수 있다.
-   **비밀 생성 주체의 필요성**:
    -   컨슈머가 비밀을 직접 다루지 않더라도, 비밀 저장소에 비밀 값을 최초로 안전하게 생성해 넣는 주체(사람 관리자 또는 IaC 스택)가 반드시 필요하다.

### 4\. 예제 (PySpark)

코드 내에 평문 자격 증명을 넣지 않고, boto3를 통해 AWS Secrets Manager에서 자격 증명을 불러와 PostgreSQL 데이터베이스에 연결하는 예제이다.

```
# 예제 7-27: 평문 자격 증명 없이 아파치 스파크에서 PostgreSQL로의 데이터베이스 연결
secretsmanager_client = boto3.client('secretsmanager')

# 비밀 관리자에서 user와 pwd 값을 검색
db_user = secretsmanager_client.get_secret_value(SecretId='user')['SecretString']
db_password = secretsmanager_client.get_secret_value(SecretId='pwd')['SecretString']

# 가져온 참조 값을 이용해 JDBC 연결
spark_session.read.option('driver', 'org.postgresql.Driver').jdbc(
    url='jdbc:postgresql:dedp', 
    table='dedp.devices',
    properties={'user': db_user, 'password': db_password}
)
```

이 방식은 여전히 코드상에서 db\_user와 db\_password를 참조하지만, 실제 값은 코드베이스에 포함되지 않고 별도의 자산으로 비밀 관리자에서 안전하게 관리된다.

## 패턴 #49: 비밀 없는 커넥터 (Secretless Connector)

### 1\. 문제 정의

-   **사용 사례**: 새로운 데이터 처리 서비스를 통합하려는 팀이 클라우드 관리 자원과 상호작용하기 위해 API 키를 사용하는 코드 예제들을 확인했다.
-   **목표**: 팀은 이러한 API 키조차도 관리하고 싶지 않으며, 코드에서 어떠한 종류의 자격 증명도 참조하지 않고 클라우드 자원에 접근하기를 원한다.

### 2\. 해결책 및 구현 방식

비밀 없는 커넥터 패턴은 애플리케이션 코드에 관리해야 할 자격 증명을 전혀 두지 않는 방식이다. 대표적으로 2가지 주요 접근 방식이 있다.

1.  **IAM 기반 정책 접근 (IAM-based Policy Access)**
    -   클라우드 제공업체의 IAM(Identity and Access Management) 서비스를 사용한다.
    -   사용자, 그룹 또는 역할(Role)에 문서 형태로 읽기/쓰기 접근 정책을 할당하여 제어한다.
    -   물리적 사용자뿐만 아니라 자동화되어 실행되는 데이터 처리 작업(애플리케이션 사용자)에도 동일하게 적용된다.
2.  **인증서 기반 인증 (Certificate-based Authentication)**
    -   IAM 서비스 대신 인증 기관(CA, Certificate Authority)을 활용한다.
    -   연결 프로세스에서 사용되는 인증서를 검증하여 작업에 필요한 권한을 부여한다.

### 3\. IAM 기반 자격 증명 없는 접근 워크플로 (4단계)

1.  **요청 발행**: 애플리케이션 사용자(데이터 처리 잡)가 클라우드 서비스(예: 객체 스토어)에 읽기 등의 요청을 발행한다.
2.  **권한 검증 요청**: 서비스는 객체를 즉시 반환하지 않고, IAM 서비스에 연결하여 해당 사용자가 요청을 충족할 권한이 있는지 검증한다.
3.  **권한 전달**: IAM 서비스가 해당 클라우드 서비스에 권한 범위 목록을 전달한다.
4.  **응답 반환**: 필요한 권한이 모두 충족되면 클라우드 서비스가 사용자에게 요청한 데이터를 반환하며, 권한이 없으면 오류를 반환한다.

### 4\. 결과 및 고려사항 (트레이드오프)

-   **초기 구성 수고 발생**: 'Secretless'라고 해서 아무런 작업이 필요 없는 것은 아니다. AWS의 경우 다른 서비스와 상호작용할 수 있도록 STS(보안 토큰 서비스)에서 반환하는 임시 자격 증명을 이용하기 위한 역할 가정(Role Assumption) 권한 설정 등의 작업을 거쳐야 한다.
-   **인증서 교체(Rotation) 오버헤드**:
    -   보안을 위해 인증서나 접근 키를 정기적으로 교체할 때 추가적인 관리 비용이 발생한다.
    -   기존 컨슈머의 서비스 중단을 방지하려면 **새 자격 증명 생성 $\\rightarrow$ 공유 $\\rightarrow$ 구/신 자격 증명 동시 지원 $\\rightarrow$ 컨슈머 이동 확인 $\\rightarrow$ 기존 자격 증명 삭제** 순서의 체계적인 교체 프로세스를 거쳐야 한다.

### 5\. 예제

#### 예제 1: PySpark에서 PostgreSQL 인증서 기반 연결

패스워드를 매개변수로 넘기지 않고, SSL 인증서를 검증하는 로직을 통해 데이터베이스에 연결한다.

```
# 예제 7-28: 아파치 스파크에서 PostgreSQL로의 인증서 기반 연결
input_data = spark.read.option('driver', 'org.postgresql.Driver').jdbc(
    url='jdbc:postgresql:dedp',
    table='dedp.devices',
    properties={
        'ssl': 'true',
        'sslmode': 'verify-full', # 서버 호스트 이름과 인증서 저장 이름의 일치 여부 검증
        'user': 'dedp_test',
        'sslrootcert': 'dataset/certs/ssl-cert-snakeoil.pem'
    }
)
```

#### 예제 2: GCP Dataflow & GCS IAM 연동 (Terraform HCL)

GCP 환경에서 Dataflow 작업에 서비스 계정(Service Account)을 생성하고 IAM 권한을 부여하여, 자격 증명 없이 GCS 버킷에 접근하도록 설정하는 예제이다.

```
# 예제 7-29: GCP에서 서비스 계정(Service Account) 생성
resource "google_service_account" "visits_job_sa" {
  account_id   = "dedp"
  display_name = "Dataflow SA for processing visits from GCS"
}

# 예제 7-30: Visits 버킷에 읽기 권한 할당 및 Dataflow 잡에 연결
resource "google_storage_bucket_iam_binding" "visits_access" {
  bucket  = "visits"
  role    = "roles/storage.objectViewer"
  members = [
    "serviceAccount:${google_service_account.visits_job_sa.email}"
  ]
}

resource "google_dataflow_job" "visits_aggregator" {
  # ... (기타 설정)
  service_account_email = google_service_account.visits_job_sa.email
}
```

이렇게 서비스 계정을 통해 visits\_aggregator 잡에 아이덴티티를 부여하면, 런타임에 별도의 자격 증명(패스워드나 키)을 코드에 보관할 필요 없이 visits 버킷을 조회할 수 있다.
