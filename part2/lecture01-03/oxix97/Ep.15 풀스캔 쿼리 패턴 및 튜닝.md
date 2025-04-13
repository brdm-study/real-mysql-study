
#### 테이블 풀스캔
테이블 풀스캔은 데이터베이스에서 테이블의 모든 행을 순차적으로 읽는 방식으로, 조건에 맞는 데이터를 찾기 위해 전체 테이블을 스캔합니다. 이는 인덱스를 사용하지 않거나, 인덱스가 비효율적일 때 발생합니다.

이번에 공부할 내용은 인덱스가 있음에도 불구하고 인덱스를 사용하지 못하고 테이블 풀스캔으로 처리되는 대표적인 경우들은 다음과 같습니다.
#### 1. 컬럼의 산술 연산 수행
```mysql
SELECT * FROM tb1 WHERE count + 10 < 2000
```

`count + 10` 은  `count`와 값이 다를 수 있기 때문에 인덱스를 사용하지 않습니다.
#### 2. 함수 사용
```mysql
SELECT * FROM users WHERE DATE(joined_at) = '2022-07-21'
```

DATE 함수를 사용함에 따라 값이 가공되어서 인덱스를 사용하지 못하는 것입니다. 이러한 경우에는 인덱스를 사용할 수 있도록, 조건 값을 범위 조건과 같은 형태로 변경해 주는 것이 좋습니다.

```mysql
SELECT * FROM users WHERE joined_at >='2022-07-21 00:00:00' AND joined_at < '2022-07-22 00:00:00';
```
#### 3. 형 변환
```mysql
SELECT * FROM users WHERE account_type(문자열 타입) = 3;
```

문자열 타입인 컬럼에 숫자 값을 조건으로 주어 account 타입에 대해 내부적으로 형변환이 수행되어 컬럼이 가공되서 인덱스를 사용하지 못하고 풀스캔이 발생한 것입니다.

실제 컬럼의 타입과 일치하는 타입으로 조건 값을 주었을 때는 정상적으로 인덱스를 사용할 수 있습니다.

하지만 예외적으로,  MySQL 내부적으로 형변환 우선순위가 있어 숫자를 문자로 변경하는 것보다 문자를 숫자로 변경하는 것에 더 우선순위를 두기 때문에 형변환 처리가 아닌 문자열 값을 숫자로 형변환하게 되어 정상적으로 인덱스를 사용할 수 있습니다.

### 2. 인덱싱 되지 않은 컬럼을 조건절에 OR 연산과 함께 사용
```mysql
SELECT * FROM users WHERE account_type = '7' OR joined_at >= '2022-07-24 00:00:00';
```
만일 인덱스가 account_type에 대한 컬럼만 존재하는 경우 account_type은 인덱스를 활용할 수도 있지만 joined_at와 OR로 연결되어 있는 경우 해당 컬럼에 대한 조건은 테이블 전체 데이터를 일일이 스캔할 수 밖에 없습니다.

결국, 조건을 만족하는 데이터를 찾기 위해서는 풀스캔이 필수적이기 때문에 풀스캔으로 동작하게 됩니다.

이때, 테이블의 joined_at에 대한 인덱스를 새로 추가한 경우에는 풀스캔을 수행하지 않게 됩니다. 즉, 조건으로 주어진 컬럼들 모두 개별 인덱스가 존재해야 쿼리가 테이블 풀스캔을 수행하지 않게 됩니다.

#### 3. 복합 인덱스의 컬럼들 중 선행 컬럼을 조건에서 누락
```mysql
CREATE TABLE users(
	id int NOT NULL AUTO_INCREMENT,
	account_type varchar(10) NOT NULL,
	joined_at datetime NOT NULL,
	...,
	PRIMARY KEY (id),
	KEY ix_accounttype_joinedat (account_type, joined_at)
)

# 선행 컬럼을 조건에서 누락
SELECT * FROM users WHERE joined_at >= '2022-07-24 00:00:00';

# 개선된 쿼리
SELECT * FROM users WHERE account_type = '3' AND joined_at >= '2022-07-24 00:00:00';
```

복합인덱스로 구성되어 있는 경우 선행 컬럼을 제외하고 나머지 컬럼들로만 쿼리의 조건으로 주어진 경우 쿼리는 인덱스를 사용하지 못합니다.

인덱스는 구성하는 컬럼 순서대로 정렬된 인덱스 데이터가 만들어지므로, 인덱스의 선행 컬럼이 조건으로 주어지지 않으면 쿼리에서 해당 인덱스를 사용할 수 없습니다.

즉, 인덱스를 구성하는 컬럼들의 순서가 쿼리에서 인덱스 사용 여부를 결정하는데 중요하다는 것을 알 수 있습니다.

#### 4. LIKE 연산에서 시작 문자열로 와일드 카드를 사용
```mysql
# LIKE 연산자 사용 시 앞뒤로 '%' 있으면 풀스캔
SELECT * FROM users WHERE first_name LIKE '%Esther%';

# 인덱스 사용
SELECT * FROM users WHERE first_name LIKE 'Esther%';
```

#### 5. 정규식 연산 사용
```mysql
SELECT * FROM users WHERE first_name REGEXP '^[abc]...';

SELECT * FROM users WHERE first_name REGEXP '^Esther';
```

정규 표현식이 주어진 경우 단순하게 값의 일치 여부를 확인하는 것을 넘어 실제로 데이터가 조건으로 주어진 표현식을 만족하는지 확인을 해야 되기 때문에 인덱스를 사용하지 못하는 것입니다.

#### 6. 테이블 풀스캔이 인덱스 사용보다 더 효율적인 경우
테이블 풀스캔이 인덱스 사용하는 것보다 효율적인 경우에는 인덱스를 의도적으로 사용하지 않습니다.

쿼리가 실행될 때 쿼리 최적화를 진행하는 단계에서 옵티마이저가 이 부분을 판단하여 실행 계획을 수립하게 됩니다.