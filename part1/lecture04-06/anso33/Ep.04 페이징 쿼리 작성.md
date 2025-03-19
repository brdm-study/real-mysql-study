### 04. 페이징 쿼리

페이징 쿼리는 원하는 전체 데이터에 대해 부분적으로 나눠서 데이터를 조회 및 처리하는 방법을 말합니다. 

보통은 limit & offset 을 사용해서 페이징 쿼리를 사용하는데요. 

```java
SELECT 
    cc1_0.id      
FROM 
    coffee_chat cc1_0      
WHERE 
    cc1_0.is_deleted = FALSE
ORDER BY 
    cc1_0.created_date DESC      
LIMIT 20 OFFSET 200;
```

이렇게 offset base 페이지네이션일 때 **OFFSET 200** → 앞의 200건을 건너뛰고 **LIMIT 20** → 그다음 20건을 가져오게 되는거죠.

그런데 **OFFSET은 앞의 N개를 먼저 읽고 버린 후, 그다음 데이터를 가져오는 방식**이기 때문에 OFFSET 값이 커질수록 불필요한 읽기(Scanning)가 증가하게 됩니다. 

그래서 limit & offset 을 사용하지 않고 데이터를 원하는 만큼만 조회해서 가져올 수 있는 방법을 알아야합니다. 

1. 범위 기반 방식 → 날짜 기간이나 숫자 범위로 나눠서 데이터를 조회하는 방식
    
    ```java
    SELECT * 
    FROM coffee_chat 
    WHERE created_date BETWEEN '2024-03-01' AND '2024-03-31';
    ```
    
    주로 배치 작업에서 테이블의 전체 데이터를 일정한 날짜나 숫자 범위로 나눠서 조회할 때 사용할 수 있습니다. 
    
2. 데이터 개수 기반 방식
    
    지정된 데이터 건수만큼 결과 데이터를 반환하는 형태로 구현된 방식
    
    cursor base pagination (?)
    
    ```java
    SELECT * 
    FROM coffee_chat 
    WHERE id > 1000 
    ORDER BY id ASC
    LIMIT 20;
    ```
    
    배치보다는 주로 서비스단에서 많이 사용됩니다. 
    
    - 이 방식의 주의할 점 → 정렬을 누구 기준으로 하느냐에 따라서 데이터 누락이 발생할 수 있습니다.
        
        (범위 조건 컬럼과 식별자 컬럼의 값 순서가 일치하지 않을 때) 
        
        강의에서는 이런 경우를 해결하기 위해서 쿼리의 where 절 조건문을 상세하게 걸었었는데요. 
        
    
    enddate + id →  정렬 기준 1번 종료날짜 2번 id /
    
    1번 20250103        20250103 01          
    
    4번 20250103        20250103 04
    
    3번 20250102        20250102 03
    
    2번 20250101        20250101 02