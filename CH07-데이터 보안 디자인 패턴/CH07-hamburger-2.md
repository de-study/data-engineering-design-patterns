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
