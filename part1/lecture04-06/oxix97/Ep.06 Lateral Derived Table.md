
## Lateral Derived Table이란?
- `Derived Table(파생 테이블)`은 쿼리의<u> FROM 절에서 서브쿼리를 통해 생성되는 임시테이블</u>을 의미합니다.
- 일반적으로 `Derived Table(파생 테이블)`은 선행테이블의 컬럼을 참조할 수 없으나, `Lateral Derived Table`은 참조가 가능합니다.
- 정의된 `Derived Table(파생 테이블)` 앞부분에 `LATERAL` 키워드를 추가해서 사용할 수 었습니다.
- 참조한 값을 바탕으로 동적으로 결과를 생성합니다.
### 동작 방식
```mysql
SELECT e.emp_no,s.sales_count, s.total_sales
FROM employees e
LEFT JOIN LATERAL (
	SELECT COUNT(*) AS sales_count,
		IFNULL(SUM(total_price),0) AS total_sales
	FROM sales
	WHERE emp_no=e.emp_no
) s on TRUE;
```

1. `employees` 테이블을 순회 후 레터럴 키워드 생성
2. `LATERAL` 레터럴 키워드를 가지고 조건 조회

-> 결과 데이터가 선행 쿼리의 의존적이므로 `select_type`이 `DEPENDENT DERIVED`로 나오게 된다.

### 활용 예제 - 종속 서브 쿼리의 다중 값 반환
```mysql
SELECT 
	d.dept_name,
	x.earliest_hire_date,
	x.full_name
FROM
	departments d
INNER JOIN LATERAL (
	SELECT 
		e.hire_date as earliest_hire_date,
			CONCAT(e.first_name, ' ',e.last_name) as full_name
	FROM
		dept_emp de
	INNER JOIN employees e ON e.emp_no = de.emp_no
	WHERE de.dept_no = d.dept_no
	ORDER BY e.hire_date LIMIT 1) x
```

`LATERAL` 기능을 사용하면 하나의 서브 쿼리로 원하는 값들을 모두 조회할 수 있으므로 쿼리가 좀 더 간결해지고 쿼리 처리 효율이 향상됩니다.

---
### 활용 예제 - SELECT 절 내 연산 결과 반복 참조
```mysql
# 일별 매출 데이터 조회
SELECT 
	profit,
	avg_profit,
	expected_profit,
	sales_achievement_rate
FROM daily_revenue,
	LATERAL (SELECT (total_sales * margin_rate) as profit) p,
	LATERAL (SELECT (profit / total_sales_number) as avg_profit) ap,
	LATERAL (SELECT (expected_sales * margin_rate) as expected_profit) ep,
	LATERAL (SELECT (profit / expected_profit * 100) as sales_achievement_rate) sar
WHERE sales_date = '2025-03-12'
```

각각의 서브쿼리에서 먼저 계산된 값에 대해 그 다음 순서에 위치한 `Lateral Derived` 테이블에서 해당 값을 참조하여 중복 연산을 제거하고 가독성을 높일 수 있습니다.

---
### 활용 예제 - 선행 데이터를 기반으로 한 데이터 분석
```mysql
# 유저 이벤트 테이블
CREATE TABLE user_events(
	id int NOT NULL AUTO_INCREMENT,
	user_id int NOT NULL,
	event_type varchar(50) NOT NULL,
	...,
	created_at datetime NOT NULL,
	PRIMARY KEY (id),
	KEY ix_eventtype_userid_created_at(event_type, user_id, created_at)
);
```
#### 요구사항
- 2024년 1월에 가입한 사용자들 중 일주일내로 결제까지 완료한 사용자의 비율

```mysql
# LATERAL TABLE 미적용
SELECT 
	sum(sign_up) as signed_up,
	sum(complete_purchase) as completed_purchase,
	(sum(complete_purchase) / sum(sign_up) * 100) as conversion_rate
FROM(
	SELECT
		user_id, 1 as sign_up, min(created_at) as sign_up_time
	FROM
		user_events
	WHERE
		event_type = 'SIGN_UP'
		AND created_at >= '2024-01-01'
		AND created_at < '2024-02-01'
	GROUP BY
		user_id
) e1
LEFT JOIN (
	SELECT 
		1 AS complete_purchase,
		min(created_at) as complete_purchase_time,
	FROM 
		user_events
	WHERE
		event_type = 'COMPLETE_PURCHASE'
	GROUP BY
		user_id
) e2 ON e2.user_id = e1.user_id
	AND e2.complete_purchase_time >= e1.sign_up_time
	AND e2.complete_purchase_time < DATE_ADD(e1.sign_up_time, INTERVAL 7 DAY);
```

```mysql
# LATERAL TABLE 적용
SELECT 
	sum(sign_up) as signed_up,
	sum(complete_purchase) as completed_purchase,
	(sum(complete_purchase) / sum(sign_up) * 100) as conversion_rate
FROM(
	SELECT
		user_id, 1 as sign_up, min(created_at) as sign_up_time
	FROM
		user_events
	WHERE
		event_type = 'SIGN_UP'
		AND created_at >= '2024-01-01'
		AND created_at < '2024-02-01'
	GROUP BY
		user_id
) e1 LEFT JOIN LATERAL(
	SELECT 
		1 as complete_purchase
	FROM
		user_events
	WHERE
		user_id = e1.user_id
		AND event_type = 'COMPLETE_PURCHASE'
		AND created_at >= e1.sign_up_time
		AND created_at < DATE_ADD(e1.sign_up_time, INTERVAL 7 DAY)
	ORDER BY
		event_type, user_id, created_at
	LIMIT 1
) e2 ON TRUE
```

---
### 활용 예제 - Top N 데이터 조회
```mysql
# 카테고리
CREATE TABLE categories(
	id int NOT NULL AUTO_INCREMENT,
	name varchar(50) NOT NULL,
	...,
	PRIMARY KEY (id)
)

# 기사
CREATE TABLE articles(
	id int NOT NULL AUTO_INCREMENT,
	category_id int NOT NULL,
	title varchar(255) NOT NULL,
	views int NOT NULL
	...,
	PRIMARY KEY (id),
	KEY ix_categoryid_views (category_id, views)
)
```
#### 요구 사항
- 카테고리별 조회수가 가장 높은 3개 기사 추출

```mysql
# LATERAL테이블 미적용

SELECT x.name, x.title, x.views
FROM (
	SELECT	c.name, a.title, a.views,
		ROW_NUMBER() OVER
			(PARTITION BY a.category_id ORDER BY a.views DESC) as article_rank
	FROM categories c
	INNER JOIN articles a ON a.category_id = c.id
) x
WHERE x.article_rank <= 3

-> (2.88 sec)
```

```mysql
# LATERAL 테이블 적용

SELECT c.name, a.title, a.views
FROM categories c
INNER JOIN LATERAL(
	SELECT category_id, title, views
	FROM articles
	WHERE category_id = c.id
	ORDER BY category_id DESC, views DESC
	LIMIT 3
) a

-> (0.00 sec)
```
## MySQL LATERAL 테이블: 장단점 및 사용 시기

아래는 MySQL의 `LATERAL` 테이블 사용에 대한 **장단점**과 **사용하면 좋은 경우/나쁜 경우**를 표로 정리한 내용입니다.

| **구분**     | **내용**                                                |
| ---------- | ----------------------------------------------------- |
| **장점**     |                                                       |
| 행별 데이터 처리  | 각 행에 대해 동적으로 데이터를 처리할 수 있어 복잡한 로직 구현이 가능함.            |
| 쿼리 구조 단순화  | 외부 테이블 데이터를 참조하여 중복 코드를 줄이고 쿼리를 간결하게 작성 가능.           |
| 다양한 데이터 처리 | 다중 행 반환, 조건별 데이터 필터링, 정렬 등 복잡한 데이터 요구사항을 쉽게 처리 가능.    |
| 특정 데이터 필터링 | 예: 각 주문에 대해 가장 최근의 상품이나 특정 조건의 데이터를 가져오는 작업에서 유용.     |
| **단점**     |                                                       |
| 성능 저하      | 각 행마다 서브쿼리가 실행되므로 대규모 데이터셋에서 성능 저하 가능성이 높음.           |
| 캐시 불가능     | 서브쿼리가 반복 실행되므로 캐싱되지 않아 쿼리 비용 증가.                      |
| 최적화 필요     | 적절한 인덱스와 쿼리 최적화가 없으면 성능 문제가 심화될 수 있음.                 |
| 제한된 사용 사례  | 모든 작업에 적합하지 않으며, 과도하게 사용 시 유지보수와 코드 가독성 문제를 초래할 수 있음. |

---
### **사용하면 좋은 경우 vs 나쁜 경우**

| **구분**        | **사용하면 좋은 경우**                                              | **사용하면 나쁜 경우**                                                   |
| ------------- | ----------------------------------------------------------- | ---------------------------------------------------------------- |
| **데이터 크기**    | 데이터셋이 작거나 중간 규모로, 각 행에 대해 동적 처리가 필요한 경우.                    | 대규모 데이터셋에서 각 행마다 서브쿼리를 실행해야 하는 경우(성능 저하 우려).                     |
| **쿼리 복잡도**    | 기존 서브쿼리나 `JOIN`으로는 처리하기 어려운 복잡한 로직을 간결하게 구현해야 할 때.          | 단순한 `JOIN`이나 서브쿼리로도 해결 가능한 작업을 `LATERAL`로 구현하려는 경우(불필요한 복잡성 증가). |
| **필터링 및 정렬**  | 예: 각 그룹에서 상위 1개의 데이터를 가져오거나, 특정 조건에 따라 동적으로 데이터를 필터링해야 할 때. | 정렬이나 필터링이 필요하지 않거나, 정적 쿼리로 충분히 처리 가능한 경우.                        |
| **성능 요구사항**   | 성능이 크게 중요하지 않고, 정확성과 간결함이 더 중요한 작업(예: 분석 작업).               | 실시간 응답 속도가 중요한 애플리케이션(예: 사용자-facing 서비스).                        |
| **인덱스 활용 여부** | 적절한 인덱스를 설정하여 서브쿼리 성능을 최적화할 수 있는 경우.                        | 서브쿼리에 사용하는 컬럼에 인덱스가 없거나 추가가 어려운 경우(성능 저하 발생 가능).                 |
