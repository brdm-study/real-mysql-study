# Lateral Derived Table

## Lateral Derived Table이란?

파생 테이블(Derived Table)은 쿼리의 FROM 절에서 서브쿼리를 통해 생성되는 임시 테이블입니다.

이 테이블은 쿼리 실행 시점에 임시 테이블 처럼 잠깐 생겼다가 사라지는 특성을 같습니다. 따라서 인덱스도 생성되지 않기 때문에 성능 최적화를 위해 가급적 작은 데이터셋을 생성하는 것이 좋습니다.

```sql
SELECT product_id, total_sales
FROM (
    SELECT product_id, SUM(price * quantity) AS total_sales
    FROM sales
    GROUP BY product_id
) AS sales_summary
WHERE total_sales >= 1000;
```

위 쿼리에서는 sales 테이블을 조회해서 SUM 함수를 통해 sales_summary 라는 Derived Table을 생성하는 예시입니다.

위와 같이 파생 테이블을 사용하면 복잡한 계산을 미리 수행한 후 그 결과를 다시 쿼리에 활용할 수 있다는 장점이 있습니다.

그렇다면 Lateral Derived Table은 무엇일까요? MySQL 8버전 이상에서 지원되는 Lateral Derived Table은 각 행(row)에 대해 다른 서브쿼리를 실행할 수 있는 파생 테이블입니다.

```sql
SELECT c.customer_id, c.name, recent_orders.order_id, recent_orders.order_date
FROM customers c
JOIN LATERAL (
    SELECT o.order_id, o.order_date
    FROM orders o
    WHERE o.customer_id = c.customer_id
    ORDER BY o.order_date DESC
    LIMIT 1
) AS recent_orders ON TRUE;
```

위와 같은 쿼리에서 JOIN 문 뒤, 파생 테이블 앞에 명시적으로 LATERAL 키워드를 작성한 것을 확인할 수 있습니다.

이처럼 작성된 쿼리는 다음과 같이 동작합니다.

1. `customers` 테이블에서 고객 정보를 조회합니다.
2. `LATERAL` 서브 쿼리에서 `orders` 테이블을 검색하는데, 각 고객의 최신 주문 1건만 조회합니다.
3. 외부 쿼리의 `customer_id` 를 파생 테이블 내에서 고객별 개별적인 서브쿼리 실행이 가능합니다.

이렇게 효율적으로 동작하는 것이죠.

만약 이를 LATERAL 없이 작성했다면 어땠을까요?

```sql
SELECT c.customer_id, c.name, ro.order_id, ro.order_date
FROM customers c
JOIN (
    SELECT o.customer_id, o.order_id, o.order_date
    FROM orders o
    WHERE o.order_date = (
        SELECT MAX(order_date)
        FROM orders
        WHERE customer_id = o.customer_id
    )
) AS ro ON c.customer_id = ro.customer_id;
```

위와 같이 `orders` 테이블에서 각 고객의 가장 최근 주문 날짜를 `MAX()` 를 통해 구하고
다시 `orders` 테이블에 조인하여 가장 최근 주문날짜에 해당하는 주문을 찾아 `customers` 와 조인하여 고객정보를 조회합니다.

이렇게 두 방식을 비교해볼게요.

파생 테이블은 먼저 전체 테이블을 조회한 후 조인을 합니다.

하지만 LATERAL을 사용하면 고객별로 주문 테이블을 개별 조회하는 방식으로 동작하죠.

필터링 방식도 일반 파생 테이블에선 MAX()함수를 통해 부하가 큰 조건 검색이 있었다면, 

LATERAL에서는 `ORDER BY` 와 `LIMIT` 으로 별도 함수 없이 선언적이고 명시적으로 검색이 가능했죠.

마지막으로 MAX() 함수 떄문에 전체 테이블 스캔이 발생하고 인덱스 사용이 어려웠다면, LATERAL의 경우에는 인덱스를 사용한 최적화가 가능한 형태가 됩니다.

