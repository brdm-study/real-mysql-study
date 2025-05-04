
`Update Query`와 `Delete Query`에서 `Join`을 사용하는 경우가 자주 발생하지는 않지만 주의점은 무엇인지 명확하게 알고 넘어가도록 하겠습니다.

**Use Case**
- 다른 테이블의 컬럼 값을 참조해서 `Update`, `Delete`하고 싶은 경우
- 한 번에 여러 테이블에 대해 `Update / Delete` 하고 싶은 경우

## JOIN UPDATE
##### 예시 1
```mysql
# 수수료 금액 update
UPDATE product p
INNER JOIN fee_info f ON f.company_id = p.company_id
SET p.fee_amount = (p.price * f.fee_rate)
WHERE p.company_id = 1;

# 상품명 정보 update
UPDATE product p
INNER JOIN order o ON o.product_id = p.id
SET p.name = 'Carrot Juice',
	o.product_name = 'Carrot Juice'
WHERE p.id = 1234;
```

각각 개별적으로 업데이트를 수행하는 것이 아니라 `JOIN`을 사용하여 하나의 쿼리로 한 번에 처리하는 것을 확인할 수 있습니다.

한 번에 여러 테이블의 데이터를 업데이트 하는 경우, `UPDATE SET`에 대상 컬럼들을 모두 명시해줘야 합니다. (변경이 필요한 컬럼들만) 또한, `LEFT JOIN`등과 같은 다른 `JOIN` 형태로 사용이 가능합니다.

---
##### 예시 2
```mysql
# 쿠폰의 만료일자 변경이 필요하고 각 쿠폰마다 만료일자는 모두 다르다고 가정
CREATE TABLE user_coupon(
	user_id int NOT NULL,
	coupon_id varchar(8) NOT NULL,
	...,
	expired_at datetime DEFAULT NULL,
	KEY ix_couponid (coupon_id)
);

# 1.단건 UPDATE
UPDATE user_coupon
SET expired_at = '2025-04-28'
WHERE coupon_id = 'b062e232';

UPDATE user_coupon
SET expired_at = '2025-04-29'
WHERE coupon_id = 'b062e074';

UPDATE user_coupon
SET expired_at = '2025-04-30'
WHERE coupon_id = 'b062ed24';
...

# 2.JOIN UPDATE
UPDATE user_coupon uc
INNER JOIN (
	VALUES ROW ('b062e232', '2025-04-28'),
	VALUES ROW ('b062e074', '2025-04-29'),
	VALUES ROW ('b062ed24', '2025-04-30')
	...
) change_coupon (coupon_id, expired_at)
ON change_coupon.coupon_id = uc.coupon_id
SET uc.expired_at = change_coupon.expired_at;
```

1번 방법과 같이 각각의 업데이트를 수행할 수도 있지만, 2번 방식과 같이 `JOIN UPDATE`를 사용하면, 여러 건ㅇ늬 데이터들을 한 번에 업데이트 할 수 있습니다.

`VALUES`라는 `TABLE CONSTRUCTOR`와 `ROW CONSTRUCTOR`를 사용하면 업데이트해야 되는 쿠폰 아이디와 만료일자 값을 하나의 세트로 묶어 테이블 형태의 데이터를 만들 수 있습니다.

만들어진 가상의 테이블을 `user_coupon.coupon_id`와 `JOIN`하면 쿠폰 각각에 대해 맞는 만료일자 값으로 업데이트 할 수 있게 됩니다.

---
## JOIN DELETE

```mysql
# 6개월 동안 활동하지 않은 유저들에 대한 기록 삭제
DELETE ul
FROM user_log ul
INNER JOIN user u
ON u.userId = ul.user_id
WHERE u.last_active < DATE_SUB(CURDATE(), INTERVAL 6 MONTH);

# 'Fruits'와 관련된 카테고리 전부 삭제
DELETE p, c
FROM product p
INNER JOIN category c ON c.id = p.category_id
WHERE c.name = 'Fruits'
```

`DELTE ... FROM` 절 사이에 데이터 삭제 대상 테이블 목록을 명시하여, 쿼리에서 참조하고 있는 테이블들 중 `전체` or `일부`에 대해 삭제할 수있습니다.

또한, `LEFT JOIN` 등 다른 유형의 `JOIN`들도 사용할 수 있습니다.

---
## Using Optimizer Hint

`JOIN UPDATE`, `JOIN DELETE`는 쿼리 처리 시 여러 테이블들을 접근하므로 일반 `SELECT` 문과 유사하게 쿼리 실행 시 효율적인 처리를 위해 테이블을 접근하는 순서를 고정하거나 사용하는 인덱스를 강제하는 등의 쿼리 최적화를 위한 `HINT`인 `Optimizer Hint`가 필요할 수 있습니다.

##### 예시 1
```mysql
# Optimizer Hint가 명시되어 있지 않은 경우
DELETE p, c
FROM product p
INNER JOIN category c
ON c.id = p.category_id
WHERE c.name = 'Fruits'
```

| id  | select_type | table | type | key           | key_len | ref       | rows | Extra |
| --- | ----------- | ----- | ---- | ------------- | ------- | --------- | ---- | ----- |
| 1   | DELETE      | c     | ref  | ix_name       | 43      | const     | 1    | NULL  |
| 1   | DELETE      | p     | ref  | ix_categoryid | 4       | test.c.id | 16   | NULL  |

`Optimizer Hint`가 명시되어 있지 않은 실행 계획을 보면 `Category` 테이블을 `Name` 컬럼에 대한 인덱스를 사용해 먼저 접근한 후, `Product` 테이블을 JOIN 하는 것을 알 수 있습니다.

```mysql
# Optimizer Hint 사용
# JOIN_FIXED_ORDER = 'STRAIGHT_JOIN'
DELETE /** JOIN_FIXED_ORDER() */ p, c
FROM product p
INNER JOIN category c
ON c.id = p.category_id
WHERE c.name = 'Fruits'
```

| id  | select_type | table | type  | key     | key_len | ref  | rows | Extra       |
| --- | ----------- | ----- | ----- | ------- | ------- | ---- | ---- | ----------- |
| 1   | DELETE      | c     | ALL   | NULL    | NULL    | NULL | 80   | NULL        |
| 1   | DELETE      | p     | range | ix_name | 43      | NULL | 1    | Using where |

`JOIN_FIXED_ORDER`라는 `Hint`가 명시되어 있는 경우의 실행 계획을 보면 `Product` 테이블을 먼저 접근하는 것을 알 수 있습니다. 

필요한 경우 쿼리 실행 시 항상 최적화된 형태로 실행될 수 있도록 `Hint`를 명시해 실행 계획을 고정해서 사용할 수 있습니다.

### 주의 사항
- 참조하는 테이블들의 데이터에는 `읽기 잠금(Shared Lock)`이 발생하므로, 잠금 경합이 발생할 수 있습니다.
- `Join Update`의 경우 `Join`되는 테이블들의 관계가 `1:N`일 때, N 테이블의 컬럼 값을 1 테이블에 업데이트하는 경우 예상과는 다르게 처리될 수 있습니다. (N : M도 마찬가지)
- `Join Update & Join Delete` 쿼리는 단일 `UPDATE / DELETE`보다 쿼리 형태가 복잡하므로, 반드시 사전에 쿼리 실행계획 확인이 필요합니다.

