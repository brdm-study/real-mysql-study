## Paging?
- 원하는 전체 데이터에 대해 `부분적으로 나눠서 데이터를 조회 및 처리`하는 방법
- DB 및 애플리케이션 서버의 `리소스(CPU/메모리/네트워크) 사용 효율 증가`
- 대용량 데이터를 다루는 웹페이지나 앱 개발 시 사용자에게 빠르고 정확한 정보를 제공하므로 페이징 쿼리의 이해와 적용은 필수적이다!!
## Paging Query 작성 가이드

DB 서버에서 제공하는 LIMIT & OFFSET 구문을 사용하는 경우가 많지만 `오히려 DBMS 서버에 더 많은 부하를 발생시킬 수 있습니다.`

```sql
Query - 1) SELECT * FROM tab LIMIT 500 OFFSET 0;

Query - 2) SELECT * FROM tab LIMIT 500 OFFSET N;
```

DBMS에서 순차적으로 레코드를 읽지 않고 `지정된 OFFSET 이후 데이터만 바로 가져올 수는 없습니다.`
다시 말해 순차적으로 읽은 후에야 원하는 데이터를 반환할 수 있으며, LIMIT & OFFSET 구문을 사용하는 경우 `쿼리 실행 횟수가 늘어날 수록 점점 읽는 데이터가 많아짐 -> 응답 시간이 길어짐`

<span style="background:#fff88f">결론적으로 LIMIT & OFFSET 구문을 사용하지 않으면서 데이터를 원하는 만큼만 조회해서 가져갈 수 있도록 쿼리를 작성해야합니다.</span>

대표적인 2가지 방식으로 구현 가능
- 범위 기반 방식
- 데이터 개수 기반 방식
## 범위 기반 방식
- 날짜 기간이나 숫자 범위로 나눠서 데이터를 조회하는 방식
- WHERE절에서 조회 범위를 직접 지정하는 형태이며 LIMIT절이 사용되지 않음
- 주로 배치 작업 등에서 테이블의 전체 데이터를 일정한 날짜/숫자 범위로 나눠서 조회할 때 사용

### 장점
- 쿼리에서 사용되는 조회 조건이 단순하며, 여러 번 쿼리를 나누어 실행하더라도 사용하는 쿼리 형태는 동일합니다.

### 예시
- 숫자인 id값을 바탕으로 범위를 나눠 데이터를 조회
```sql
SELECT *
FROM users
WHERE id > 0 AND id <= 5000
```

- 날짜 기준으로 나눠 조회
```sql
SELECT *
FROM payments
WHERE finished_at >= '2025-03-09' AND finished_at < '2025-03-12'
```

특정 컬럼에 대해 범위를 나누어 조회하는 경우에 쿼리 성능 향상을 위하여 해당 컬럼의 인덱스를 생성해 두는 것이 좋습니다.

데이터의 양이 많고 특정 컬럼에 대한 세부적인 조건을 부여할 때 사용하는 방식

## 데이터 개수 기반 방식
- 대용량 데이터 방식보다 일반 서비스에서 주로 많이 사용되는 방식입니다.
- `지정된 데이터 건 수만큼 결과 데이터를 반환`하는 형태로 구현 된 방식입니다
- 처음 쿼리를 실행할 때와 그 이후 쿼리를 실행할 때 쿼리 형태가 달라집니다.
- 쿼리에서 `ORDER BY`, `LIMIT` 절이 사용됩니다.

### 데이터 개수 기반 방식 (동등 조건 사용 시)
```mysql
CREATE TABLE payments(
	id int NOT NULL AUTO INCREMENT,
	user_id int NOT NULL,
	...,
	PRIMARY KEY (id),
	KEY ix_userid_id (user_id, id)
)

# 페이징 적용 대상 쿼리
SELECT *
FROM payments
WHERE user_id = ?
```

<u>ORDER BY 절에는 각각의 데이터를 식별할 수 있는 컬럼(PK 와 같은)이 반드시 포함되야합니다.</u>

#### 1회차
```sql
SELECT *
FROM user_id
WHERE user_id = ?
ORDER BY id
LIMIT 30
```
#### N회차
```sql
SELECT *
FROM user_id
WHERE user_id = ?
    AND id > {이전 데이터의 마지막 id 값}
ORDER BY id
LIMIT 30
```

ORDER BY 절에는 각각의 데이터를 식별할 수 있는 식별자 컬럼이 반드시 포함되어야 합니다.
N회차 쿼리에서 이전에 가져온 데이터들 중 마지막 id 값의 다음 순서에 해당되는 데이터셋을 순차적으로 반환받을 수 있게 됩니다.

또한 지정된 건 수만큼만 읽어서 반환할 수 있게 하는 인덱스가  존재하므로, 내부적으로 정렬이 발생하지 않고 명시 된 개수만큼만 데이터를 읽게 됩니다.

### 데이터 개수 기반 방식 (범위 조건 사용 시 )
```sql
CREATE TABLE payments(
	id int NOT NULL AUTO_INCREMENT,
	user_id int NOT NULL,
	finished_at datetime NOT NULL,
	...,
	PRIMARY KEY (id),
	KEY ix_finishedat_id (finished_at, id)
)

# 페이징 대상 쿼리
SELECT *
FROM payments
WHERE finished_at >= ? AND finished_at < ?

# 1회차
SELECT *
FROM payments
WHERE finished_at >= '2025-03-11' AND finished_at < '2025-03-12'
ORDER BY finished_at, id
LIMIT 5

# N회차 : WHERE절에 식별자 컬럼 조건 추가
SELECT *
FROM payments
WHERE finished_at >= '2025-03-11' AND finished_at < '2025-03-12'
	AND id > ?  
ORDER BY finished_at, id
LIMIT 5
->이후에 반환되는 데이터 중 id가 항상 크다는 보장이 없으며, 있는 경우에 누락 발생 !!


# N회차 : 올바른 쿼리
SELECT *
FROM payments
WHERE ((finished_at = '2025-03-11' AND id > ?)
	OR (finished_at > '2025-03-11' AND finished_at < '2025-03-12'))
ORDER BY finished_at, id
LIMIT 5
-> 실제로 필요한 만큼만 데이터 읽어서 반환
```
#### 왜 ORDER BY 절에 finished_at 이 포함되어야 할까요?
id컬럼만 명시하는 경우, 조건을 만족하는 데이터들을 모두 읽어들인 후 id로 정렬한 다음 지정된 건수만큼 반환하게 됩니다.

`finished_at` 컬럼을 선두에 명시하면, (finished_at, id) 인덱스를 사용해서 정렬 작업 없이 원하는 건수만큼 순차적으로 데이터를 읽을 수 있으므로 처리 효율이 향상됩니다.

<u>단, pk와 같이 테이블에 저장된 순서대로 값이 증가하므로 각 컬럼에 대해 데이터 간 순서가 동일한 경우 WHERE절에 식별자 컬럼 조건만 추가해도 문제가 없습니다.</u>

![[Pasted image 20250312003431.png]]
