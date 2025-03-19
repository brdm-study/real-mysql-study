### 05. Stored Function

MySQL의 Stored Function(저장 함수)는 SQL 쿼리 내부에서 사용할 수 있는 사용자 정의 함수입니다.
반복적인 SQL 로직을 캡슐화하여 재사용 가능하며, 쿼리에서 직접 호출할 수 있습니다.

```java
DELIMITER $$

CREATE FUNCTION get_total_orders(customer_id INT) 
RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE total_orders INT;
    SELECT COUNT(*) INTO total_orders FROM orders WHERE customer_id = customer_id;
    RETURN total_orders;
END $$

DELIMITER ;
```

- DETERMINISTIC vs NOT DETERMINISTIC
    
    MySQL의 STORED FUNCTION은 함수의 결정성(determinism)에 따라 DETERMINISTIC 또는 NOT DETERMINISTIC으로 설정됩니다. 
    
    **DETERMINISTIC은 같은 입력에 대해서 항상 같은 결과를 반환하는 것을 말하고, NOT DETERMINISTIC 은 같은 입력이어도 실행할 때마다 결과가 다를 수 있음을 말합니다.** 
    
    NOT DETERMINISTIC 함수는 비교 기준 값이 변수이고, 매번 레코드에서 읽은 후에 WHERE 절을 평가할 때마다 결과가 달라질 수 있습니다. 따라서 **인덱스 최적화에 사용할 수 없게 됩니다.** 
    
    MySQL은 인덱스를 활용할 때 DETERMINISTIC한 함수를 선호합니다.
    
    반대로 NOT DETERMINISTIC 함수는 **쿼리 최적화에 불리**하며, 특히 WHERE 절에서 사용하면 **인덱스가 무효화**됩니다.
    
    따라서 저장함수를 선언할 때 DETERMINISTIC 을 꼭 명시해서 성능을 지킬 수 있도록 합시다. 
    
    그런데 그렇다면 NOT DETERMINISTIC 함수는 항상 안좋은걸까요? 성능 최적화 측면에서는 불리할 수 있지만, 특정 상황에서는 **반드시 필요**하고 **유용**한 경우가 있습니다. RAND(), UUID(), NOW() 같은 값들은 실행할 때마다 변하는 것이 **필연적으로 필요한 경우(랜덤 할인, 추천)나,** CURRENT_TIMESTAMP() 등을 활용하여 **시간에 따라 다른 값을 반환해야 하는 경우가 있답니다.**