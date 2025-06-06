## UUID?
UUID는 `Universally Unique Identifier`의 약자로, 전 세계적으로 고유한 식별자를 생성하기 위한 표준 규약입니다. 

UUID는 128비트 길이의 숫자로 구성되며, 충돌 가능성이 극히 낮아 다양한 시스템 간에 안전하게 식별자를 공유할 수 있습니다.

UUID는 32자리의 16진수 문자열로 표현되며, 하이픈(-)으로 구분된 8-4-4-4-12 형태를 가집니다. 
ex: `550e8400-e29b-41d4-a716-446655440000`

**Version- 1 & 2**
- Timestamp 기반의 UUID 생성
- 별도의 Unique한 값 입력없이 생성 가능

**Version- 3 & 5**
- name과 namespace의 MD5 또는 SHA-1 해시 기반의 UUID 생성
- 생성시 Unique한 입력을 필요로 함

**Version-4**
- 완전 랜덤한 UUID 생성
### Version-1
Timestamp의 비트 순서가 바뀌어서 UUID에 배치가 되며 생성 시점이 동일해도, 정렬 순서가 일치하지는 않는 특징이 있습니다.

`UUID version-1`의 타임스탬프는 100 나노초 단위로 1씩 증가하며, 7분 10여초 단위로 첫번째 파트가 초기화 됩니다.
#### UUID vs B-Tree
- UUID
	- 시점과는 별개로 랜덤한 값 생성
	- (상대적으로) 긴 문자열 CHAR(32) or VARCHAR(32)
- B-Tree 인덱스의 성능 저해 요소
	- 정렬되지 않은 키 값 생성,  INSERT
	- 길이가 긴 키 값
	- 일반적으로 UUID 컬럼은 `unique` 제약 필요

#### WorkingSet
MySQL 인덱스의 용량이 매우 크다 하더라도 꼭 필요한 부분만 메모리로 읽어서 쿼리를 처리할 수 있습니다. 이는 데이터베이스 성능에 중요한 영향을 미치며, 특히 물리 메모리보다 데이터 크기가 클 경우 효율적인 메모리 관리가 필요합니다.

하지만 인덱스 전체 데이터가 WorkingSet인 경우, Disk IO가 많이 늘어날 수록, 쿼리의 성능이 저하되지만 UUID는 값을 매번 생성합니다. 즉, 전체 인덱스 크기만큼의 메모리가 있어야 합니다.

### UUID_TO_BIN() & BIN_TO_UUID()

MySQL에서 `UUID_TO_BIN()`과 `BIN_TO_UUID()` 함수는 UUID를 효율적으로 저장하고 조회하기 위해 사용됩니다. 이 두 함수는 서로 반대되는 역할을 수행하며, MySQL 8.0 이상에서 지원됩니다.

```mysql
# 시간 무관하게 랜덤한 값
SELECT HEX(UUID_TO_BIN(@uuid,0));

# 시간 순서대로 정렬된 값
SELECT HEX(UUID_TO_BIN(@uuid,1));
```

시간 순서대로 정렬된 값은 최근 데이터일수록 UUID가 큰 값을 가지게 되는 특징이 있습니다. 그래서 최근의 데이터를 주로 필요로 하는 서비스에서 UUID 컬럼의 인덱스로 생성해도 최근 몇 개월의 UUID prefix를 가진 데이터만 메모리에 적재가 되어 있으면 쿼리 처리를 빠르게 할 수 있다.

![Pasted image 20250409213048](https://github.com/user-attachments/assets/a08fd4b0-6dd9-4652-88b9-3aa56edd5baf)

