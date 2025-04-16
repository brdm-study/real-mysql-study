
SELECT * 과 SELECT COUNT(*), 성능 상 큰 차이가 없으며, ORM의 경우
SELECT COUNT(DISTINCT(id))를 자동으로 쿼리를 실행하는데 이는 성능이 매우 떨어질 수 있다.

### COUNT *
```sql
SELECT COUNT(*)
	WHERE ix_fd = 'A' AND non_ix_fd='B';

SELECT *
	WHERE ix_fd = 'A' AND non_ix_fd='B';

# SELECT *은 LIMIT와 동시에 사용되지만
# SELECT COUNT(*)은 LIMIT 없이 사용된다.
```

- 거의 동일한 성능을 내게 된다. (SELECT ALL 이 반환하는 양이 많기 때문에 네트워크 비용은 더 크게 되지만 MySQL 서버내부에서 처리 하는 것은 거의 동일)

- 만일 페이징을 적용하며 데이터가 많은 경우 COUNT 쿼리가 훨씬 오래 걸리게 된다.

## COUNT * 사용 가이드
- Convering Index는 꼭 필요한 경우에만 사용 하도록 한다.

### COUNT * vs COUNT(DISTINCT expr)
- COUNT * 는 레코드 건수만 확인
- COUNT (DISTINCT expr)는 임시테이블로 중복 제거 후 건수 확인 -> 테이블 레코드를 모두 임시테이블로 복사하게 되어 메모리와 CPU 성능이 많이 필요하게 되어 쿼리의 성능이 느려지게 됨

### COUNT * 튜닝
- 최고의 튜닝은 쿼리 자체를 제거하는 것
	- 전체 결과 건수 확인 쿼리 제거
	- 페이저 번호 없이, **이전*, **이후** 페이지 이동
- 쿼리를 제거할 수 없다면, 대략적 건수 활용
	- 부분 레코드 건수 조회
		- 표시할 페이지 번호만큼의 레코드만 건수 확인
	- 임의의 페이지 번호는 표기
		- 첫 페이지에서 10개 페이지 표시 후, 실제 해당 페이지로 이동하면서 페이지 번호 보정
	- 통계 정보 활용

## 질문
- Convering Index가 뭐죠?