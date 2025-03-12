## MySQL Function
- Built-in Function
- User Defined Function(UDF)
- Stored Function
## DETERMINISTIC vs NOT DETERMINISTIC
### DETERMINISTIC
- 동일 상태와 동일 입력으로 호출 -> 동일한 결과 반환
- 그렇지 않은 경우 -> NOT DETERMINISTIC

시작시점의 스냅샷을 보도록 구현되어 있습니다. 즉, MySQL 서버에서 실행되는 Query 문장 하나는 동일한 데이터 상태를 보게 됩니다.

하나의 문장에서 Stored Function이 여러번 호출되어도 전부 동일한 상태를 확인할 수 있습니다.


```mysql
# func 1
CREATE
	DEFINER=root@'lcoalhost'
	FUNCTION func1() RETURNS INTERGER
	DETERMINISTIC SQL SECURITY INVOKER
BEGIN
	SET @func1_called = IFNULL(@func1_called,0)+1;
	RETURN 1;
END ;;

# func 2
CREATE
	DEFINER=root@'lcoalhost'
	FUNCTION func1() RETURNS INTERGER
	DETERMINISTIC SQL SECURITY INVOKER
BEGIN
	SET @func2_called = IFNULL(@func2_called,0)+1;
	RETURN 1;
END ;;

# 실행 결과
func1 : 3
func2 : 12
```

![[Pasted image 20250312121517.png]]

`NOT DETERMINISTIC` 함수의 호출 횟수가 높은 것은 쿼리의 실행 계획이 인덱스를 활용하지 못하는 것을 확인할 수 있습니다.

즉, 풀 테이블 스캔이 사용된다는 것이 핵심적인 문제였던 것을 확인할 수 있습니다.
### NOT DETERMINISTIC 최적화 이슈

`NOT DETERMINISTIC` 함수의 결과는 비확정적
- 매번 호출 시점마다 결과가 달라질 수있습니다.
	- 비교 기준 값이 상수가 아닌 변수
	- 매번 레코드를 읽은 후, `WHERE` 절을 평가할 때마다 결과가 달라질 수 있음
	- 인덱스에서 특정 값을 검색할 수 없음
	- 인덱스 최적화 불가능
#### NOT DETERMINISTIC 효과
NOT DETERMINISTIC Built-in Function
- RAND()
- UUID()
- SYSDATE()
- NOW()

`NOT DETERMINISTIC Function 표현식`
```sql
WHERE col=(RAND()*1000)
```
#### NOT DETERMINISTIC 예외
NOW() vs SYSDATE()
- 동일하게 현재 일자와 시간 반환하는 함수
- 둘 모두 NOT DETERMINISTIC 함수
- 하지만
	- NOW() 함수는, DETERMINISTIC 처럼 동작한다.
	- NOT() 함수는, 하나의 `Statement`내에서는 , `Statement`의 시작 시점 반환
	- SYSDATE() 함수는, NOT DETERMINISTIC
	- SYSDATE() 함수는, 매번 함수 호출 시점 반환

### Stored Function 주의 사항
MySQL 서버에서`Stored Function`을 호출하는 경우 기본 값이 `NOT DETERMINISTIC`으로 인식되기 때문에 생성시 기본 옵션을 꼭 명시적으로 설정하는 것이 좋습니다.

```mysql
CREATE
	DEFINER = ...
	FUNCTION function_name(...)
	RETURNS ...
	DETERMINISTIC
	SQL SECURITY INVOKER
BEGIN
...
```


진짜 1도 모르겠다 큰일이네