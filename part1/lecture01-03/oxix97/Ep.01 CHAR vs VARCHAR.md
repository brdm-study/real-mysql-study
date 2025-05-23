
## 모델링
### 논리 모델링 (업무)
- 업무 분석
- 엔티티, 속성, 관계 도출
- 정규화
- 서비스에 대한 이해 > 기술
### 물리 모델링 (DBMS)
- DBMS 벤더별 최적 컬럼 타입 선정
- 접근 패턴 분석
- 반 정규화
- 인덱스 전략 수립
- 서비스에 대한 이해 < 기술

## CHAR vs VARCHAR
### 공통점
- 문자열 저장용 컬럼
- 최대 저장 가능 문자 길이 명시 (바이트 수가 아닌 길이), 
### 차이점
- 값의 실제 크기에 관게없이 고정된 공간 할당 여부
- 최대 저장 길이 : CHAR(255), VARCHAR(16383)
- 저장된 값의 길이 관리 여부
	- CHAR(10) -> 남는 공간에 상관없이 할당, 다만 UTF8MB4의 경우 저장된 값 길이 관리
		- 가변 길이 문자셋을 사용하더라도 VARCHAR와 동작 방식이 유사함.
		- ex) ABCD, 문자 4,  공백 6
	- VARCHAR(10) -> 사용되는 공간만큼 할당, 저장된 값 길이 저장
		- 문자 셋 관계없이, 꼭 필요한 만큼만 공간 사용
		- ex) 4ABCD -> 길이 1, 문자 4

### CHAR 타입의 공간 낭비
- 고정된 길이의 값 저장은 CHAR, 그 외의 경우 VARCHAR
- 저장되는 문자열의 최소 최대 길이 가변 폭이 큰 경우
- 저장되는 값의 길이 변동이 크지 않다면 낭비는 크지 않다. -> 서비스의 영향이 거의 없다.

## 컬럼 값의 길이 변경시의 동작

### VARCHAR
- 초기에 레코드에 저장된 공간은 delete marking 한 이후 새로운 빈 공간에 레코드를 저장
- 수정을 할 수록 새로운 레코드를 찾기가 어려워지며, 찾지 못하는 경우 컴팩션 이후 다시 찾게 됨 -> 옮겨쓰기
### CHAR
- 초기에 충분한 공간을 레코드에 예약해 두었기 때문에 새로운 빈 공간을 찾지 않아도 됨 -> VARCHAR보다 새로운 레코드 찾는 일이 적음

### 문자열 타입 선정
- CHAR
	- 값의 가변 길이 범위 폭이 좁고 자주 변경되는 경우 (특히 인덱스된 컬럼의 경우)
	- 20자 이하의 고정 길이 데이터
	- 대부분의 값이 비슷한 길이일 경우 CHAR가 유리
- VARCHAR
	- 8,000자 이하의 문자열
	- 20자 이상이거나 길이가 가변적인 경우
## 질문
- 조각모음을 하게 되면 비효율적으로 공간을 차지하는 것이 성능에 영향을 미치는지? 조각모음 할 때만 가끔 영향이 가는건가? -> 주기적으로 Delete Marking?된 거 배치 돌듯이 정리할 수는 없는건가?
  
- UTF8MB4 문자셋을 사용할 때 CHAR 타입의 동작 방식이 VARCHAR와 유사하다고 했는데,  대부분 UTF8MB4를 사용하는 것으로 알고있는데 VARCHAR만 사용해도 큰 문제가 없다는거 아닌가?





