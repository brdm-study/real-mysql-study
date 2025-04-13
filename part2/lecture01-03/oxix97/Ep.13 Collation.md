## Collation ?
- 문자를 비교하거나 정렬할 때 사용되는 규칙
- 문자집합 `Character Set (무자와 코드값의 조합이 정의되어있는 것)` 에 종속적
- MySQL에서 모든 문자열 타입 컬럼은 독립적인 문자집합과 콜레이션을 가질 수 있다.
- 사용자가 특별히 지정하지 않는 경우, 서버에 설정된 문자집합의 기본 콜레이션으로 자동 설정된다.
- `SHOW COLLLATION` 으로 종류를 확인할 수 있다.

## Collation의 종류

`Collation`은 다음과 같은 네이밍 컨벤션을 가집니다.

**문자집합_언어종속_UCA버전_민감도**
##### 문자집합
- utf8mb4
- utf8mb3
- latin1
- euckr
##### 언어종속
특정 언어에 대한 해당 언어에서 정의한 정렬 순서에 의해 정렬 및 비교를 수행합니다.(다른 언어들에는 적용 X)

locale code `tr`, `turkish`로 표시됩니다. => ex) utf8mb4_tr_...
##### UCA버전
`Unicode Collation Algorithm Version` 콜레이션들에서 사용하는 문자열 비교 표준 알고리즘을 의미합니다.

UCA 기반으로 동작하는 콜레이션들은 정보에 이름이 표시가 되며 다음과 같이 나타납니다. ex) `utf8mb4_0900_...`, `utf8mb4_general_...`

`general`의 경우는 UCA기반이 아닌 MySQL에서 비교, 성능 향상등을 위해 자체적으로 커스텀한 규칙이 적용된 콜레이션입니다.
##### 민감도
콜레이션의 문자열 비교 시 악센트 구분이나 대소문자를 구분, binary 값 기반 비교를 수행하는지에 대한 특성들이 명시되는 부분입니다.

| _ai  | Accent-insensitive |
| ---- | ------------------ |
| _as  | Accent-sensitive   |
| _ci  | Case-insensitive   |
| _cs  | Case-sensitive     |
| _ks  | Kana-sensitive     |
| _bin | Binary             |
ai, as는 악센트 구분 여부, ci,cs는 대소문자 구분 여부, ks는 일본어, bin은 Binary 값을 비교하고 정렬하는 것을 의미합니다.

## 동작 방식
### 유니코드 기준
데이터 비교
- `DUCET (Default Unicode Collation Element Table)`에 정의된 가중치 값을 바탕으로 비교하며 가중치 값은 `Primary`.`Secondary`.`Tertiary`와 같이 단계적으로 구성됩니다.
- 콜레이션에 따라 사용되는 가중치 값이 달라지며 `ai_ci`는 `Primary Weight`까지, `as_ci`는 `Secondary Weight`까지, `as_cs`
는 `Tertiary Weight`까지 사용합니다.
- 한글의 경우 분해하여 가중치 값을 조합한 후 비교하며, `WEIGHT_STRING()` 함수를 통해 문자열의 가중치 값을 확인할 수 있습니다.

## Collation 설정
기본적으로 MySQL 서버의 설정된 Character Set의 디폴트 콜레이션으로 글로벌하게 설정되어 있습니다.

데이터베이스, 테이블, 컬럼 별로 독립적으로 지정할 수 있습니다.

Character Set, Collation 모두 특별히 값을 설정하지 않은 경우, 기본적으로 서버는 `Character Set : utf8mb4 `, `Collation : utf8mb4_0900_ai_ci`로 설정되어 있으며, `character_set_server & collection_server` 설정 변수를 통해 원하는 값으로 설정할  수 있습니다.

### 주의사항
#### 1. 서로 다른 콜레이션을 가진 컬럼들 값 비교 시 에러가 발생
동일한 하나의 콜레이션을 사용하도록 변경해야 하며, 이를 위해 필요한 경우 실제로 컬럼들의 콜레이션 설정을 변경하거나 쿼리에서 `collate` 키워드를 사용하여 비교 연산을 수행하는 컬럼에 대해 원하는 콜레이션을 명시적으로 지정할 수 있습니다.

#### 2. WHERE절에서 콜레이션 변경 시 일반 인덱스 사용 불가
WHERE절에서 `collate` 키워드를 사용하여 변경하는 경우 해당 컬럼의 인덱스가 존재하더라도 사용할 수 없습니다.

![Pasted image 20250409124912](https://github.com/user-attachments/assets/6d1a16fc-15ec-41d6-b7fd-94af74c846e1)

일반적으로 인덱스 데이터는 인덱싱되는 문자열 컬럼에 지정된 콜레이션 규칙에 따라 정렬되어 저장되고, 인덱스를 통한 데이터 검색이나 비교 연산등도 컬럼에 설정된 콜레이션을 바탕으로 수행됩니다.

즉, 쿼리에서 명시적으로 `collate`를 사용해 변경하는 경우 실제 콜레이션과 다르기 때문에 인덱스를 사용할 수 없습니다.

![Pasted image 20250409125230](https://github.com/user-attachments/assets/d42ac8d3-5575-430f-94d5-591aee366265)

표현식을 사용하는 함수 기반 인덱스로 하여 쿼리에서 사용하는 형태 그대로 표현식에 지정하여 인덱스를 생성하면 제대로 사용할 수 있습니다.

#### 3. PK도 Collation의 영향을 받는다.
만일 대소문자를 구분하지 않는 콜레이션을 사용한다면, 'esther'와 'ESTHER'를 구분할 수 없습니다.

해당 컬럼이 unique 제약이 설정되어 있는 경우 데이터의 고유성을 판단하는데 영향을 미치게 됩니다.

컬럼에 unique 제약 설정의 경우 해당 부분을 고려하여 설정하는 것이 중요합니다.
#### 4. 기본 콜레이션(utf8mb4_0900_ai_ci)에서의 한글 비교 문제

![Pasted image 20250409201407](https://github.com/user-attachments/assets/477ecdf3-a54c-407f-98f6-d663ffbeaffd)
![Pasted image 20250409201537](https://github.com/user-attachments/assets/523e770e-a6bf-4c73-a275-9d82fefe0fc2)

#### 5. 대소문자 구문을 위한 콜레이션 선정
기본 콜레이션은 대소문자를 구분하지 않으며, utf8mb4 문자집합에 속한 콜레이션 중 대소문자를 구분할 수 있는 콜레이션을 선정해야합니다.
