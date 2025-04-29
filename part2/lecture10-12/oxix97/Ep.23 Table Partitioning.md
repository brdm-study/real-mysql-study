## Table Partitioning?
하나의 테이블을 물리적으로 여러 테이블로 분할해서 데이터를 저장하는 기법

테이블 파티셔닝이 되면 내부적으로 여러 개의 테이블로 분할되고 그에 따라 실제 데이터와 인덱스 데이터도 나눠서 저장됩니다.

내부적으로는 분할되어 있지만 사용자 입장에서는 여전히 하나의 테이블로 접근해서 사용할 수 있습니다. 때문에 사용자는 사용성 측면에서는 거의 차이를 느낄 수 없습니다.

---
## Table Partitioning 필요한 이유

### 1. 삭제 가능한 이력 데이터들을 효율적으로 관리

`로그성 데이터들이 저장되는 테이블`이 이에 해당됩니다. 파티셔닝을 통해 데이터를 `날짜별, 범위별 혹은 다른 기준에 따라 분할`함으로써 필요하지 않게 된 로그성 `데이터를 쉽게 식별하고 처리`할 수 있습니다.

때문에 불필요한 데이터를 삭제하는 작업이 간소화 됩니다. 데이터 삭제 시 DELETE 문으로 삭제하는 것이 아닌 대상 `데이터가 담긴 파티션을 DROP` 하는 명령문을 실행함으로 사용했던 `디스크 공간을 완전히 반납`하게 되므로 서버의 자원도 `효율적으로 재활용` 할 수 있습니다.

### 2. 자원 사용 효율 증가 및 쿼리 성능 향상

DB 서버의 자우너을 좀 더 효율적으로 사용할 수 있으며, 쿼리 성능 또한 향상될 수 있습니다.

이러한 이점은 데이터 접근이 특정 범위에 집중되는 상황에서 더욱 명확하게 드러납니다.

예를 들어, 테이블을 날짜 범위로 파티셔닝한다고 가정하였을 때, 각 파티션은 특정 기간의 데이터만을 보유하도록 설정됩니다.

특정 날짜 범위에 해당하는 데이터를 조회하는 쿼리가 실행되면 MySQL은 조건 범위에 해당하지 않는 파티션을 쿼리 처리 과정에서 자동적으로 제외하게 되며 이러한 과정을 `Partition Pruning` 이라고 합니다.

`Partition Pruning`은 자원 사용 효율과 쿼리 성능을 향상시키는 핵심 기술이며, 예시는 다음과 같습니다.

```mysql
CREATE TABLE orders (
    order_id INT NOT NULL,
    order_date DATE NOT NULL,
    customer_name VARCHAR(255),
    amount DECIMAL(10,2),
    PRIMARY KEY (order_id, order_date)
)
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p0 VALUES LESS THAN (2020),
    PARTITION p1 VALUES LESS THAN (2021),
    PARTITION p2 VALUES LESS THAN (2022),
    PARTITION p3 VALUES LESS THAN (MAXVALUE)
);
```

>orders 테이블
 ├── p0 ( ~ 2019년 데이터)
 ├── p1 (2020년 데이터)
 ├── p2 (2021년 데이터)
 └── p3 (2022년 ~ 현재 데이터)

>+-------------+
|   orders    |
+------+------+------+------+
|  p0  |  p1  |  p2  |  p3  |
|(~2019)|2020|2021|2022~|
+------+------+------+------+

하나의 논리적 테이블이지만 내부적으로 `연도별로 쪼개서 저장`하여 해당 되는 파티션만 스캔하기 때문에 빠르게 처리할 수 있습니다.

---
## Partitioning Type
`Range`, `List`, `Hash`, `Key` 타입이 있으며, 각각의 대해 확장된 타입으로 `Range Columns`, `List Columns`, `Linear Hash`, `Linear Key` 타입이 있습니다.
### Range
컬럼 값의 `범위(Range)`에 따라 파티션을 나누는 방식을 의미합니다. 제일 많이 사용하는 타입중의 하나이며 일정한 범위로 파팃녀을 분할해서 사용하는 경우가 대다수 입니다.

지정된 범위 값을 바탕으로 데이터를 분할하게 되고, 범위는 날짜나 숫자 값을 기반으로 지정됩니다.

```mysql
PARTITION BY RANGE (YEAR(order_date)) (
    PARTITION p0 VALUES LESS THAN (2020),
    PARTITION p1 VALUES LESS THAN (2021),
    PARTITION p2 VALUES LESS THAN (2022),
    PARTITION p3 VALUES LESS THAN MAXVALUE
);
```
#### Range Columns
**RANGE Partitioning을 여러 컬럼 조합**이나 **비산술 타입**(예: DATE, DATETIME, VARCHAR 등)에서도 쉽게 쓰기 위한 확장형
##### 기존 Range의 한계
- 단일 컬럼만 가능
- 숫자/함수 결과로만 분할 가능
##### Range Columns 특징
- 다중 컬럼 가능
- 직접 컬럼 값을 기반으로 비교 (함수 필요 없음)

```mysql
CREATE TABLE sales (
    sale_id INT,
    sale_date DATE,
    region VARCHAR(20)
)
PARTITION BY RANGE COLUMNS (sale_date, region) (
    PARTITION p0 VALUES LESS THAN ('2020-01-01', 'Seoul'),
    PARTITION p1 VALUES LESS THAN ('2021-01-01', 'Busan'),
    PARTITION p2 VALUES LESS THAN (MAXVALUE, MAXVALUE)
);
```

날짜 + 지역 같이 복합적인 범위 분할이 필요하거나 `YEAR()`, `MONTH()` 와 같은 함수를 쓰기 싫을때? 사용합니다.

### List
`컬럼 값이 특정 집합(LIST)에 속하는지`에 따라 파티션을 나누는 방식이며, 명시적인 값 목록을 지정하거나, 값들이 불연속적일 때 유리합니다.

```mysql
PARTITION BY LIST (region) (
    PARTITION p_north VALUES IN ('Seoul', 'Incheon'),
    PARTITION p_south VALUES IN ('Busan', 'Daegu'),
    PARTITION p_etc VALUES IN ('Others')
);
```
#### List Columns
LIST Partitioning을 다중 컬럼 조합이나 **복합 타입** 지원하는 확장형
##### 기존 List의 한계
- 단일 컬럼만 가능
##### List Columns 특징
- 복수 컬럼 가능
- 명시적으로 (값1, 값2) 형태로 지정

```mysql
CREATE TABLE employees (
    emp_id INT,
    department VARCHAR(50),
    region VARCHAR(50)
)
PARTITION BY LIST COLUMNS (department, region) (
    PARTITION p0 VALUES IN (('HR', 'Seoul'), ('HR', 'Busan')),
    PARTITION p1 VALUES IN (('IT', 'Seoul'), ('IT', 'Busan')),
    PARTITION p2 VALUES IN (('Finance', 'Seoul'), ('Finance', 'Busan'))
);
```

부서 + 지역 같이 고정된 조합 리스트로 관리할 때 사용합니다.

### Hash
`해시 함수를 이용해 파티션을 나누는 방식`이며, 특정 값에 대한 규칙성이 없으며 자동 분산을 통해 균등 분포를 목표로 합니다.

주로 데이터 양이 많고, 특정 컬럼 기준으로 고르게 나누고 싶을 때 사용합니다.
### Key
MySQL 내부 해시 알고리즘(KEY)을 이용해 파티션을 나누는 방식이며, Hash와 비슷하지만 `Column`만 입력받는다는 점에서 차이가 있습니다.

## Partition Table 사용
### Partition 추가
#### 1. 마지막 Partition 이후 범위의 신규 Partition 추가
`MAXVALUE`로 값을 지정한 `Partition`이 존재 하지 않는 경우 `ADD PARTITION` 명령으로 다음 범위에 해당하는 `Partition`을 추가하면 됩니다.
#### 2. MAXVALUE 파티션이 존재하는 경우
`MAXVALUE` 파티션이 존재하는 경우 `ADD PARTITION` 명령을 사용할 수 없으며, `REORGANIZE PARTITION` 명령으로 `MAXVALUE` 파티션을 재구성하는 형태로 신규 파티션을 추가해야합니다.

또한, 새로운 파티션을 마지막 파티션 이후가 아닌 기존 파티션들 사이에 추가하는 경우도 동일합니다.

### Partition 제거
파티션 자체를 제거하며, 필요 시 별도 스토리지(콜드 데이터 저장용) 백업 후 제거

```mysql
ALTER TABLE user_log DROP PARTITION p202401;
```
### Partition 비우기
파티션 자체는 남겨두고 내부의 저장된 데이터만 제거

```mysql
ALTER TABLE user_log TRUNCATE PARTITION p202401;
```

### Partition Table의 Index 사용
- 고유 키가 아닌 일반 보조 인덱스의 경우 자유롭게 구성해서 생성 가능합니다.
- 파티션 별로 동일한 구조의 인덱스가 생성되며, 파트션마다 별도로 인덱스를 생성하거나, 인덱스 구성을 다르게 가져가는 것은 불가능합니다.
- `Partition Pruning`을 통해 접근 대상 파티션 선정 후 인덱스 스캔
- `Partition Pruning`은 쿼리의 `WHERE`절에 파티셔닝 기준 컬럼에 대한 조건이 있으야 가능합니다.
	- `UNIX_TIMESTAMP()`, `YEAR()` 함수를 사용해도 꼭 표현식과 동일한 형태로 `WHERE`절에 명시해야 되는 것은 아님

```mysql
CREATE TABLE logs (
  id INT,
  created_at DATETIME
)
PARTITION BY RANGE (UNIX_TIMESTAMP(created_at)) (
  PARTITION p0 VALUES LESS THAN (UNIX_TIMESTAMP('2024-01-01')),
  PARTITION p1 VALUES LESS THAN (UNIX_TIMESTAMP('2025-01-01')),
  PARTITION pmax VALUES LESS THAN MAXVALUE
);

# 1. 파티션 키와 동일한 표현식
SELECT * FROM logs 
WHERE UNIX_TIMESTAMP(created_at) < UNIX_TIMESTAMP('2024-06-01');

# 2. 다른 표현식
SELECT * FROM logs 
WHERE created_at < '2024-06-01';
```

즉, **표현식이 일치하지 않아도**, 내부적으로 **결과가 정적으로 계산 가능**하면 pruning은 작동한다는 것을 알 수 있습니다.
#### 쿼리 처리 과정 다이어그램
![[Pasted image 20250429111421.png]]

---
### Partition Table 사용 가이드
#### 요구사항에 알맞은 범위 선정
주로 접근하는 범위 또는 삭제 대상 범위를 바탕으로 일/주/월/년 등으로 필요에 맞게 파티셔닝 하는 것이 효율적입니다. 

범위를 너무 잘게 혹은 너무 크게 잡은 경우 자원 사용이나 관리 측면에서 비효율적일 수 있으니 요구사항에 맞춰 적절한 범위를 선정하는 것이 중요합니다.

#### MAXVALUE 파티션 사용
데이터가 저장될 때 저장될 파티션이 존재하지 않는 경우 에러가 발생하며 치명적인 서비스 장애로 이어질 수 있습니다.

장애 발생 방지 차원에서 가능하면 `MAXVALUE PARTITION`을 미리 만들어두는 것이 좋습니다.

#### 파티션 추가 / 삭제 시 트래픽이 적은 시점에 수행
파티션 추가/삭제와 같은 작업은 `ALTER`명령과 동일하게 `METADTA LOCK`이 발생하므로, 테이블로 유입되는 트래픽이 적은 시점에 수행하는 것이 안전합니다.

추가적으로, DB에 오래 열려있는 트랜잭션이 있는지도 확인한 후 작업하는 것이 좋습니다.

#### 필요 시 파티션 관리 프로그램 (자동화 스크립트) 개발
관리해야하는 파티션 테이블 수가 많다면, 자동으로 파티션을 관리해주는 프로그램을 개발해서 사용하는 것이 운영 코스트를 줄일 수 있는 하나의 방안이 될 수 있습니다.

실제 실무에서도 운영 효율을 위해 파티션관리를 위한 자동화 스크립트를 만들어서 사용하는 경우가 많습니다.

---
## Partition Table 사용 제약 및 주의사항
1. 외래키 및 공간 데이터 타입, 전문검색 인덱스 사용 불가
2. 파티션 표현식에 MySQL 내장 함수들을 사용할 수 있으나, 모두 `Partition Pruning` 기능을 지원하는 것은 아닙니다.
	- `TO_DAYS()`, `TO_SECONDS()`, `YEAR()`, `UNIX_TIMESTAMP()`만 지원
3. 테이블의 모든 고유키에 파티셔닝 기준 컬럼이 반드시 포함되어야 합니다.
4. `WHERE`절에 파티셔닝 기준 컬럼에 대한 조건이 포함되어야 파트션들에만 접근하는 최적화인 `Partition Pruning`이 적용됩니다.
5. 값이 자주 변경되는 컬럼을 파티션 기준 컬럼으로 선정해서는 안됩니다.
	- 파티셔닝 기준 값이 변경되는 경우 파티션간 데이터 이동이 발생하기 때문입니다.


> Partitioning이 존재하는 이유는 크게 3가지입니다.
>
> **(1) 데이터 분산**: 한 테이블에 너무 많은 데이터를 몰아넣지 않고, 물리적으로 나누기
>   
>**(2) 쿼리 최적화**: "필요한 파티션만 읽는다(=Partition Pruning)" → 검색 성능 극대화
>
>**(3) 관리 편의성**: 오래된 데이터 삭제(=DROP PARTITION)나 백업 등을 빠르게 수행
>
  이 세 가지는 전부 "**파티션에 따른 데이터 분리**"가 전제되어야 성립합니다.  
>
  즉, **데이터는 한 번 파티션에 배치되면 변하지 않는다**는게 대원칙
