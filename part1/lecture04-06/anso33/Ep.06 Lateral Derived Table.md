### 06. Lateral Derived Table

Derived Table은 쿼리의 from절에서 서브 쿼리를 통해서 생성되는 임시 테이블을 말합니다. 일반적인 **Derived Table은 이전 테이블을 참조할 수 없지만**, LATERAL을 사용하면 앞서 참조된 테이블의 컬럼을 활용할 수 있어요. 

- **LATERAL JOIN의 특징**
    - **앞서 정의된 테이블의 값을 서브쿼리에서 활용 가능**
    - **각 행마다 다른 서브쿼리 결과를 반환할 수 있음**
    - **특정 조건을 만족하는 상위 N개의 데이터를 조회하는 데 유용**
    - **일반 JOIN과 비교해 더 유연한 데이터 필터링 가능**

- derived table
    - **기존 방식** → 각 유저의 최신 주문을 조회하고 싶다면 일반적으로 서브쿼리를 사용해야 함.
    
    ```java
    SELECT *
    FROM users u
    JOIN (
        SELECT user_id, order_date, total_amount
        FROM orders
        WHERE order_date = (SELECT MAX(order_date) 
    								    FROM orders WHERE user_id = u.id)) o 
    								    ON u.id = o.user_id;
    ```
    
    - 서브쿼리가 실행될 때, 전체 orders 테이블에서 각 user_id별로 MAX(order_date) 값을 찾음. →
    
    만약 orders 데이터가 많아지면, **모든 유저의 전체 주문 데이터를 먼저 가져오고** → 그 중 최신 데이터를 골라내야 함. 따라서 **불필요한 데이터 조회로 인해 성능 저하** 발생.
    
    - **문제점**
        - orders 테이블을 여러 번 조회해야 해서 비효율적.
        - 서브쿼리가 users 테이블을 참조할 수 없어서 비직관적.

- lateral derived table
    
    ```java
    SELECT u.id, u.name, o.*
    FROM users u
    JOIN LATERAL (
        SELECT * 
        FROM orders o 
        WHERE o.user_id = u.id 
        ORDER BY o.order_date DESC 
        LIMIT 1
    ) o ON true;
    ```
    
    - 즉, **users 테이블의 한 행을 가져온 후 해당 user_id에 대해 가장 최근의 order를 조회**하는 방식.
    - 서브쿼리가 **각 행마다 다르게 실행**되므로 성능 최적화 가능.
        - MySQL에서 **LATERAL JOIN**을 사용하면, **메인 테이블(users)에서 각 행을 하나씩 읽고, 그 행에 대한 특정 조건을 만족하는 서브쿼리를 실행**할 수 있습니다. 즉, 기존의 일반 JOIN이나 Derived Table방식에서는 **한 번에 전체 데이터를 처리**하려고 하지만, **LATERAL JOIN은 메인 테이블의 각 행에 대해서 개별적으로 다른 조건을 적용할 수 있도록 설계**되어 있습니다.
    - ORDER BY와 LIMIT을 활용하여 **각 사용자의 가장 최근 주문을 효율적으로 조회할 수 있습니다.**
    - LATERAL을 사용하면 users의 id를 서브쿼리 내에서 직접 참조 가능하여 가독성이 좋아짐.

- **LATERAL JOIN 활용 사례**
    1. 유저별 최근 3개의 주문 조회
        
        ```java
        SELECT u.id, u.name, o.*
        FROM users u
        JOIN LATERAL (
            SELECT * 
            FROM orders o 
            WHERE o.user_id = u.id 
            ORDER BY o.order_date DESC 
            LIMIT 3
        ) o ON true;
        ```
        
    2. 각 제품의 최신 리뷰 가져오기 
        
        ```java
        SELECT p.id, p.name, r.*
        FROM products p
        JOIN LATERAL (
            SELECT * 
            FROM reviews r 
            WHERE r.product_id = p.id 
            ORDER BY r.review_date DESC 
            LIMIT 1
        ) r ON true;
        ```