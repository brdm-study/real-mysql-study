# Generated Column과 함수 기반 인덱스(Function Based Index)

## Generated Column이란?
Generated Column은 테이블의 컬럼 값이 사용자 입력이 아니라 미리 정의된 표현식에 의해 자동으로 생성되는 컬럼입니다. 표현식에는 고정된 값, 함수 호출, 또는 다른 컬럼에 대한 연산 등이 들어갈 수 있습니다. 사용자는 이 컬럼에 직접 값을 입력하거나 수정할 수 없습니다.

Generated Column은 두 가지 타입으로 나뉘는데요

1. **가상 컬럼 (Virtual Generated Column)**
2. **스토어드 컬럼 (Stored Generated Column)**

## Virtual Generated Column (가상 컬럼)
- 컬럼 값이 **디스크에 저장되지 않고**, 조회 시점에 자동으로 계산됩니다.
- 값 계산 시점: 데이터를 읽기 전 또는 before 트리거 실행 직후
- 인덱스 생성 가능 (인덱스 값만 디스크에 저장됨)

> 기본적으로 생성되는 Generated Column 타입이며, NULL 값을 허용합니다

## Stored Generated Column (스토어드 컬럼)
- 컬럼 값이 계산되어 실제로 **디스크에 저장**됩니다.
- 값 계산 시점: 데이터가 Insert 또는 Update 될 때
- 인덱스 생성 가능
- **Primary Key** 로 설정 가능 (가상 컬럼은 불가능)

## DDL 작업 시 주의사항

**가능한 작업**
- `ALTER TABLE` 명령으로 `Generated Column` 추가, 변경, 삭제, 이름 변경 가능
- 일반 컬럼 ↔ `Stored Generated Column` 간의 전환 가능
 
**불가능한 작업**
- 가상 컬럼 → 일반 컬럼 전환 불가능
- 가상 컬럼 ↔ 스토어드 컬럼 간 직접 전환 불가능 (삭제 후 재추가 방식으로 가능)
 
**DDL 알고리즘 권장 순서**
- 빠른 작업을 위해 `INSTANT` 방식 시도
- 불가능하면 `INPLACE` 방식에 `LOCK=NONE` 옵션으로 시도
 
## 유효성 검사(Validation) 옵션

Generated Column을 추가하거나 변경할 때 데이터의 무결성을 검사하는 두 가지 옵션이 존재합니다.

첫번째 옵션은 `WITHOUT VALIDATION` 옵션 입니다. 기본값으로 동작하는 옵션으로 기존 데이터를 검사하지 않으므로 오류 발생 가능성이 있습니다. 운영 환경에서 권장되는 옵션이죠.

두번째 옵션은 `WITH VALIDATION` 옵션입니다. 데이터 복사를 수행하여 데이터 무결성 검사를 진행합니다. 만약 무결성 위반 사항이 발견될 경우 작업 실패로 이어지며, DML 발생 시 메타데이터 락이 발생할 가능성도 있습니다.
따라서, 운영 환경에선 비추천 되는 옵션이죠.

## 함수 기반 인덱스 (Function Based Index)

함수 기반 인덱스는 일반적인 컬럼이나 컬럼 Prefix 대신 표현식의 결과를 직접 인덱싱합니다.

```sql
CREATE INDEX f_index ON tab ((col1 + col2), (col1 * col2));
CREATE INDEX f_index ON tab (DATE(col));
```

위와 같은 방식으로 만들어진 인덱스는 아래처럼 활용이 가능해요.

```sql
SELECT * FROM tab WHERE (col1 + col2) > 10;
```

이렇게 일반 컬럼과 혼합해서 복합 인덱스 구성이 가능합니다. (단 표현식은 반드시 괄호로 묶어 표현해야 해요!)

## 인덱스 사용 시 주의사항

위와 같은 함수 기반 인덱스는 현식이 정확히 일치해야만 인덱스를 사용합니다.

예를 들어, `(col1 + 1)` 로 정의된 인덱스가 있다면, `(1 + col1)` 을 사용하는 쿼리에선 해당 인덱스가 실행계획에 반영되지 않아요.

같은 이유로, 데이터 타입까지 정확하게 일치해야만 해요. double 타입 컬럼을 연산한 인덱스는 값이 같더라도, int 타입 조건과 매칭되지 않죠.

인덱스를 사용 가능한 연산자도 정해져 있어요 -> `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN` 등이에요.

## Generated Column & 함수 기반 인덱스 제한사항

**Generated Column 제한**
- 표현식에서 비결정적 함수, 변수, 서브쿼리 사용 불가
- INSERT / UPDATE 시 직접 값 입력 불가 (DEFAULT만 가능)
- Generated 컬럼은 NEW.col_name, OLD.col_name 으로 트리거에서 참조 가능
- Primary Key에 표현식 사용 불가능

**함수 기반 인덱스 제한**
- 비결정적 함수 표현식 인덱스 불가능
- 공간(Spatial) 또는 전문 검색(Full-text) 인덱스 지원하지 않음
- 컬럼의 prefix로는 표현식 인덱스 사용 불가 (substring, cast 등 함수 사용해야 함)

예를 들어볼까요? 특정 문자열 부분을 자주 조회하는 경우 효율적으로 인덱스를 사용할 수 있어요.

이메일 도메인으로 검색해야하는 경우를 예로 들어볼게요

```sql
-- 인덱스 생성
CREATE INDEX ix_email_domain ON users((SUBSTRING_INDEX(email, '@', -1)));

-- 조회 쿼리
SELECT *
FROM users
WHERE SUBSTRING_INDEX(email, '@', -1) = 'gmail.com';
```

이런식으로 함수 기반 인덱스는 실무에서 유용하게 사용될 여지가 있어보여요.

함수 기반 인덱스나 Generated 컬럼은 매우 강력한 기능이지만, 올바른 표현식과 데이터 타입을 반드시 확인하고, 실제 실행 계획을 검증한 후 사용하는 것이 중요할 것 같습니다!
