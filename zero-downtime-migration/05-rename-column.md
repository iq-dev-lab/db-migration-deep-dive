# 컬럼 이름 변경 — 직접 RENAME의 위험

---

## 🎯 핵심 질문

- 왜 `RENAME COLUMN old TO new`는 Breaking Change인가?
- 스키마 변경과 앱 배포 순서가 중요한 이유는 무엇인가?
- 언제 직접 RENAME이 안전한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

컬럼 이름 변경은 단순 메타데이터 작업으로 보이지만, ORM(JPA, SQLAlchemy)과 쿼리 매핑 때문에 Breaking Change입니다.

다양한 데이터 접근 계층이 컬럼명을 하드코딩하므로:
- JPA `@Column(name="old_name")`
- MyBatis `<result column="old_name" />`
- Raw JDBC `SELECT old_name FROM ...`
- 뷰, 저장 프로시저
- ETL/BI 도구

한 곳이라도 구컬럼명을 사용하면 오류 발생합니다.

---

## 😱 흔한 실수 (Before — 직접 RENAME)

```sql
-- Before: 한 번에 RENAME (Breaking Change!)
ALTER TABLE users RENAME COLUMN username TO user_handle;

-- 결과:
-- ├─ JPA 오류:
--   @Column(name="username")  -- 이제 없음!
--   java.sql.SQLException: Unknown column 'username' in field list
--
-- ├─ MyBatis 오류:
--   <result column="username" />  -- 이제 없음!
--   java.sql.SQLException: Unknown column 'username'
--
-- ├─ 저장 프로시저 오류:
--   SELECT username FROM users;  -- 컬럼 없음!
--   Error 1054: Unknown column 'username'
--
-- └─ 뷰 오류:
--   CREATE VIEW user_public AS SELECT username, email FROM users;
--   -- 컬럼 없음! (뷰가 자동 수정되지 않음)

-- 배포 순서:
-- DB 변경 → 앱 배포 대기 중
-- 모든 SELECT: Unknown column 'username'
```

**문제점**:
1. 즉시 오류 발생
2. 모든 SELECT/INSERT/UPDATE 실패
3. 롤백 필요

---

## ✨ 올바른 접근 (After — Expand-Contract 적용)

```sql
-- After: 5단계로 안전한 이름 변경 (Expand-Contract)

-- 1단계: 새 컬럼 추가 (기존 컬럼 유지)
-- DB 배포 (모든 앱에 영향 없음)
ALTER TABLE users 
ADD COLUMN user_handle VARCHAR(100) NULL,
ALGORITHM=INSTANT;

-- 2단계: 앱 배포 (v2, 양쪽 컬럼 모두 지원)
-- 앱 코드: username과 user_handle을 동시에 쓰기
User user = new User();
user.setUsername("john");      // 구 컬럼 (호환성)
user.setUserHandle("john");     // 신 컬럼 (새로운 이름)
userRepository.save(user);      // 양쪽 모두 저장

// 읽기: 새 컬럼 우선
String handle = user.getUserHandle() != null 
    ? user.getUserHandle() 
    : user.getUsername();

-- 3단계: 데이터 복사 (백필)
UPDATE users 
SET user_handle = username 
WHERE user_handle IS NULL 
LIMIT 10000;

-- (반복, 모든 행이 user_handle을 가질 때까지)

-- 4단계: 앱 배포 (v3, 새 컬럼만 사용)
// 이제 user_handle만 읽고 씀
User user = userRepository.findById(id);
String handle = user.getUserHandle();

// 또는 JPA의 경우 @Column(name="user_handle")로 변경

-- 5단계: 구 컬럼 제거 (모든 앱이 새 이름 사용)
ALTER TABLE users 
DROP COLUMN username,
ALGORITHM=INPLACE;
```

**결과**: 무중단 이름 변경, 명확한 단계

---

## 🔬 내부 동작 원리

### 1. RENAME COLUMN 메커니즘 (MySQL 8.0.14+)

```sql
-- RENAME COLUMN 직접 사용 (MySQL 8.0.14+)
ALTER TABLE users RENAME COLUMN username TO user_handle;

내부 동작:
1. 메타데이터 Lock 획득 (EXCLUSIVE, 짧음)
2. 메타데이터 변경:
   - 컬럼명: username → user_handle
   - 모든 참조 업데이트:
     ├─ 테이블의 컬럼 정의
     ├─ 인덱스 (사용 중이면 무효화)
     ├─ 저장 프로시저 (파손, 문법 오류)
     ├─ 뷰 (파손, 문법 오류)
     └─ Generated Column (의존성 파손)
3. 메타데이터 Lock 해제

물리 저장소: 변경 없음
인덱스: 자동 재구성 필요할 수도

시간: 밀리초 (INSTANT)
Lock: NONE

하지만:
모든 어플리케이션에서 즉시 오류 발생!
```

### 2. 컬럼명 변경이 Breaking Change인 이유

```
데이터 접근 계층별 영향:

┌──────────────────────────────────────┐
│ ALTER TABLE users                    │
│ RENAME COLUMN username TO user_handle│
│                                      │
│ 메타데이터 변경 완료 ✓                │
└──────────────────────────────────────┘
                  │
                  └─→ ❌ JPA 오류
                      @Column(name="username")
                      
                  └─→ ❌ MyBatis 오류
                      <result column="username" />
                      
                  └─→ ❌ Raw JDBC 오류
                      SELECT username FROM users;
                      
                  └─→ ❌ 저장 프로시저 오류
                      CREATE PROCEDURE ... AS
                      SELECT username FROM users;
                      
                  └─→ ❌ 뷰 오류
                      CREATE VIEW v AS
                      SELECT username FROM users;
                      
                  └─→ ⚠️ 쿼리 로그 분석 도구 오류
                      (예전 쿼리가 username을 참조)

모든 오류 원인:
"username" 컬럼이 존재하지 않음
```

### 3. Expand-Contract를 통한 안전한 변경

```
Timeline:

┌─────────────────────────────────────────┐
│ 1단계: ADD COLUMN user_handle            │
│ 메타데이터 추가 (NULL, 기본값 없음)      │
└─────────────────────────────────────────┘
                  │
                  └─→ ✓ JPA 호환
                      @Column(name="username") 아직 유효
                      
                  └─→ ✓ MyBatis 호환
                      <result column="username" /> 아직 유효
                      
                  └─→ ✓ 저장 프로시저 호환
                      모든 SELECT username 정상 작동
                      
                  └─→ ✓ 뷰 호환
                      모든 뷰가 정상 작동

┌─────────────────────────────────────────┐
│ 2단계: 앱 배포 (v2)                      │
│ 양쪽 컬럼 지원:                          │
│ - username에서 읽기                     │
│ - username과 user_handle에 쓰기          │
└─────────────────────────────────────────┘
                  │
                  └─→ ✓ 구버전(v1) 실행 중인 경우
                      SELECT username 정상, v1은 계속 작동
                      
                  └─→ ✓ 신버전(v2)이 INSERT
                      INSERT (username, user_handle)
                      VALUES ('john', 'john')
                      양쪽 모두 값 있음

┌─────────────────────────────────────────┐
│ 3단계: 데이터 백필                       │
│ UPDATE users SET user_handle = username  │
│ WHERE user_handle IS NULL                │
│ (배치 처리)                              │
└─────────────────────────────────────────┘
                  │
                  └─→ ✓ 앱 v1 계속 실행
                      SELECT username 정상

                  └─→ ✓ 앱 v2 계속 실행
                      SELECT user_handle = 'john'

┌─────────────────────────────────────────┐
│ 4단계: 앱 배포 (v3)                      │
│ 오직 user_handle만 사용                  │
│ (username 완전히 무시)                   │
└─────────────────────────────────────────┘
                  │
                  └─→ ✓ 모든 Pod가 v3
                      모두 user_handle 사용

┌─────────────────────────────────────────┐
│ 5단계: DROP COLUMN username               │
│ (이제 구 컬럼 필요 없음)                 │
└─────────────────────────────────────────┘
                  │
                  └─→ ✓ 안전한 삭제
                      모든 앱이 새 컬럼 사용 중
```

### 4. JPA Entity 코드 패턴

```java
// 1단계~3단계: 양쪽 컬럼 지원
@Entity
@Table(name = "users")
public class User {
    @Column(name = "username")
    private String username;
    
    @Column(name = "user_handle", nullable = true)
    private String userHandle;
    
    // getter/setter
}

// 4단계: 새 컬럼만 사용하도록 변경
@Entity
@Table(name = "users")
public class User {
    // @Column(name = "username")
    // private String username;  // 제거됨
    
    @Column(name = "user_handle", nullable = false)
    private String userHandle;  // 이제 필수
    
    // getter/setter
}

// 또는 마이그레이션:
// 1단계~3단계: 별칭 메서드로 호환성 제공
@Entity
@Table(name = "users")
public class User {
    @Column(name = "username")
    private String username;
    
    @Column(name = "user_handle", nullable = true)
    private String userHandle;
    
    // 호환성 유지 (구버전 코드가 username을 호출해도)
    public String getUsername() {
        return username;
    }
    
    public void setUsername(String username) {
        this.username = username;
        this.userHandle = username;  // 양쪽 동기화
    }
    
    // 신버전 코드 (user_handle 직접 사용)
    public String getUserHandle() {
        return userHandle;
    }
    
    public void setUserHandle(String userHandle) {
        this.userHandle = userHandle;
    }
}

// 4단계: 동기화 제거
@Entity
@Table(name = "users")
public class User {
    @Column(name = "user_handle")
    private String userHandle;
    
    public String getUserHandle() {
        return userHandle;
    }
    
    public void setUserHandle(String userHandle) {
        this.userHandle = userHandle;
    }
}
```

---

## 💻 실전 실험

### 실험 1: 직접 RENAME의 위험성 재현

```bash
docker run -d \
  --name mysql80_rename \
  -e MYSQL_ROOT_PASSWORD=root \
  -p 3306:3306 \
  mysql:8.0

mysql -h 127.0.0.1 -u root -proot -e "CREATE DATABASE testdb;"
```

```sql
USE testdb;

-- 테이블 생성
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(100),
  email VARCHAR(100)
);

INSERT INTO users (username, email) 
VALUES ('john', 'john@example.com');

-- 뷰 생성 (구컬럼명 사용)
CREATE VIEW user_names AS 
SELECT id, username FROM users;

-- 저장 프로시저 생성 (구컬럼명 사용)
DELIMITER //
CREATE PROCEDURE get_user_name(IN user_id INT)
BEGIN
  SELECT username FROM users WHERE id = user_id;
END //
DELIMITER ;

-- 사전: 모든 것이 정상
SELECT * FROM users;  -- OK
SELECT * FROM user_names;  -- OK
CALL get_user_name(1);  -- OK

-- 위험한 RENAME 실행
ALTER TABLE users RENAME COLUMN username TO user_handle;

-- 이제 오류!
-- 쿼리 1: 직접 SELECT
SELECT username FROM users;
-- Error 1054: Unknown column 'username'

-- 쿼리 2: 뷰 사용
SELECT * FROM user_names;
-- Error 1356: View 'testdb.user_names' references invalid table(s)
-- or column(s) or function(s) or definer/invoker of view lack appropriate access rights

-- 쿼리 3: 저장 프로시저
CALL get_user_name(1);
-- Error 1054: Unknown column 'username'

-- 뷰 재정의 필요
DROP VIEW user_names;
CREATE VIEW user_names AS 
SELECT id, user_handle FROM users;

-- 저장 프로시저 재정의 필요
DROP PROCEDURE get_user_name;
DELIMITER //
CREATE PROCEDURE get_user_name(IN user_id INT)
BEGIN
  SELECT user_handle FROM users WHERE id = user_id;
END //
DELIMITER ;

-- 이제 다시 정상
SELECT * FROM user_names;  -- OK
CALL get_user_name(1);  -- OK
```

### 실험 2: Expand-Contract 안전한 변경

```sql
USE testdb;

-- 사전 상태 복원
DROP PROCEDURE IF EXISTS get_user_name;
DROP VIEW IF EXISTS user_names;

CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(100),
  email VARCHAR(100)
);

INSERT INTO users (username, email) 
VALUES ('john', 'john@example.com'),
       ('jane', 'jane@example.com');

-- 뷰, 저장 프로시저 재생성 (구컬럼명)
CREATE VIEW user_names AS 
SELECT id, username FROM users;

DELIMITER //
CREATE PROCEDURE get_user_name(IN user_id INT)
BEGIN
  SELECT username FROM users WHERE id = user_id;
END //
DELIMITER ;

-- 1단계: 새 컬럼 추가
ALTER TABLE users 
ADD COLUMN user_handle VARCHAR(100) NULL,
ALGORITHM=INSTANT;

-- 확인: 구컬럼은 여전히 작동
SELECT * FROM user_names;  -- OK (username 여전히 있음)
CALL get_user_name(1);  -- OK

-- 2단계: 앱 배포 (양쪽 컬럼 지원)
-- JPA에서:
-- @Column(name="username") AND @Column(name="user_handle")
-- INSERT/UPDATE 시 양쪽 모두 설정

-- 2.5단계: 데이터 복사 (백필)
UPDATE users 
SET user_handle = username 
WHERE user_handle IS NULL;

-- 확인
SELECT id, username, user_handle FROM users;
-- id | username | user_handle
--  1 | john     | john
--  2 | jane     | jane

-- 3단계: 앱 배포 (신 컬럼만 사용)
-- JPA에서:
-- @Column(name="user_handle")만 사용
-- username 무시

-- 4단계: 뷰와 저장 프로시저 선택적으로 재정의
-- (또는 아래 단계에서 컬럼 삭제 시 함께)

-- 5단계: 구 컬럼 제거
ALTER TABLE users DROP COLUMN username;

-- 확인: 새 컬럼만 존재
SELECT * FROM users;  -- id, email, user_handle

-- 6단계: 뷰와 저장 프로시저 재정의 (선택)
-- (이미 파손된 상태이므로, 재생성)
DROP VIEW user_names;
CREATE VIEW user_names AS 
SELECT id, user_handle FROM users;

DROP PROCEDURE get_user_name;
DELIMITER //
CREATE PROCEDURE get_user_name(IN user_id INT)
BEGIN
  SELECT user_handle FROM users WHERE id = user_id;
END //
DELIMITER ;

-- 최종 확인
SELECT * FROM user_names;  -- OK
CALL get_user_name(1);  -- OK
```

### 실험 3: MyBatis 매핑 확인

```java
// MyBatis Mapper (XML 또는 Annotation)

// 1단계~3단계:
@Select("SELECT id, username, user_handle FROM users WHERE id = #{id}")
@Results({
  @Result(column="id", property="id"),
  @Result(column="username", property="username"),  // 아직 필요
  @Result(column="user_handle", property="userHandle")
})
User getUserById(int id);

// 4단계 (앱 배포):
@Select("SELECT id, user_handle FROM users WHERE id = #{id}")
@Results({
  @Result(column="id", property="id"),
  @Result(column="user_handle", property="userHandle")  // username 제거
})
User getUserById(int id);

// 또는 간단히:
@Select("SELECT id, email, user_handle FROM users WHERE id = #{id}")
User getUserById(int id);
// 자동 매핑 (컬럼명 = 필드명)
```

---

## 📊 성능/비용 비교

| 전략 | 다운타임 | 배포 횟수 | 수정 범위 | 위험도 |
|------|--------|----------|---------|--------|
| **직접 RENAME** | 0분 | 1회 | 모든 코드 | 높음 |
| **Expand-Contract** | 0분 | 2~3회 | 단계적 | 낮음 |
| **컬럼 별칭** (추가 VIEW) | 0분 | 1회 | 제한적 | 중간 |

---

## ⚖️ 트레이드오프

| 선택지 | 장점 | 단점 |
|--------|------|------|
| **직접 RENAME** | 간단, 1회 배포, 빠름 | Breaking Change, 모든 코드 변경 필수, 위험 |
| **Expand-Contract** | 무중단, 단계적, 안전 | 배포 횟수 증가, 과정 복잡, 임시 컬럼 유지 |
| **VIEW 별칭** | 코드 변경 최소 | VIEW 관리 복잡, 성능 오버헤드 |

---

## 📌 핵심 정리

1. **RENAME COLUMN**: MySQL 메타데이터는 즉시 변경되지만, 모든 애플리케이션에서 오류 발생
2. **Breaking Change**: JPA, MyBatis, 저장 프로시저, 뷰 모두 영향
3. **Expand-Contract 5단계**:
   - ADD COLUMN (새 컬럼)
   - 앱 배포 (양쪽 지원)
   - 데이터 백필
   - 앱 재배포 (새 컬럼만)
   - DROP COLUMN (구 컬럼)
4. **무중단**: 각 단계마다 Lock 없음
5. **언제 직접 RENAME**: DB 전용 컬럼(ORM/코드 참조 없음)인 경우만 안전

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 컬럼이 여러 뷰에서 참조된다면 Expand-Contract를 해야 할까?</strong></summary>

**답변**:

여러 뷰에서 참조되는 경우, Expand-Contract는 선택사항입니다.

```sql
상황:
CREATE VIEW view1 AS SELECT username FROM users;
CREATE VIEW view2 AS SELECT id, username, email FROM users;
CREATE VIEW view3 AS SELECT username FROM users WHERE active = 1;

선택지 1: Expand-Contract (완벽한 안전)
- 3단계 모두 진행
- 모든 VIEW를 새 컬럼으로 재정의
- 비용: 관리 복잡도 높음

선택지 2: 직접 RENAME (모든 VIEW 파손 감수)
- 1회 배포
- 모든 VIEW를 즉시 재정의
ALTER TABLE users RENAME COLUMN username TO user_handle;

-- 파손된 뷰 확인
SELECT TABLE_NAME FROM INFORMATION_SCHEMA.VIEWS 
WHERE TABLE_SCHEMA = 'testdb' AND TABLE_NAME IN ('view1', 'view2', 'view3');

-- 뷰 상태 확인
SELECT TABLE_NAME, VIEW_DEFINITION FROM INFORMATION_SCHEMA.VIEWS 
WHERE TABLE_SCHEMA = 'testdb';
-- 오류: View definition seems problematic

-- 모든 뷰 재정의
DROP VIEW view1, view2, view3;
CREATE VIEW view1 AS SELECT user_handle FROM users;
CREATE VIEW view2 AS SELECT id, user_handle, email FROM users;
CREATE VIEW view3 AS SELECT user_handle FROM users WHERE active = 1;

선택지 3: 절충 (VIEW 별칭)
-- 컬럼 이름은 RENAME, 기존 쿼리와의 호환성은 VIEW로
ALTER TABLE users RENAME COLUMN username TO user_handle;

-- VIEW를 별칭으로 사용
CREATE VIEW username AS 
SELECT user_handle FROM users;
-- 기존 SELECT username FROM users를 SELECT * FROM username로 변경
-- (이 방식은 성능 저하 가능)

추천:
- VIEW 개수 < 5개: 직접 RENAME 후 모두 재정의
- VIEW 개수 >= 5개: Expand-Contract 고려
- VIEW가 외부 도구(BI, ETL)에서 참조: Expand-Contract 필수
```

</details>

<details>
<summary><strong>Q2: 백필 중에 구버전 앱이 username을 UPDATE하면 user_handle과 일치하지 않을까?</strong></summary>

**답변**:

네, 동기화 문제가 발생할 수 있습니다.

```
Timeline:

T1: 백필 배치 1
    UPDATE users SET user_handle = username
    WHERE user_handle IS NULL LIMIT 10000;
    
    처리된 행: ID 1~10000
    username = 'john', user_handle = 'john'

T2: 구버전 앱 UPDATE
    UPDATE users SET username = 'john_new' WHERE id = 5000;
    
    결과:
    username = 'john_new'
    user_handle = 'john' (변경 안 됨!)

T3: 신버전 앱 READ
    SELECT user_handle FROM users WHERE id = 5000;
    결과: 'john' (오래된 값)

문제: 신버전이 낡은 이름을 사용

해결책 1: 신버전이 양쪽을 모두 UPDATE (2단계)
// 2단계: 신버전 배포 (양쪽 UPDATE)
Update user = ...;
user.setUsername("john_new");      // 구 컬럼
user.setUserHandle("john_new");     // 신 컬럼 (동기화)

해결책 2: 백필 때 트리거 사용 (권장하지 않음)
-- 트리거: username 변경 시 user_handle도 자동 변경
CREATE TRIGGER sync_user_handle 
AFTER UPDATE ON users FOR EACH ROW
BEGIN
  IF NEW.username != OLD.username THEN
    UPDATE users SET user_handle = NEW.username WHERE id = NEW.id;
  END IF;
END;

해결책 3: 강제 앱 배포 순서
-- 1단계: ADD COLUMN
-- 2단계: 앱 배포 (양쪽 지원, 동기화 코드 포함)
-- 3단계: 백필 (이제 신버전이 모두 처리)
-- 4단계: 앱 재배포
```

**권장**: 배포 순서를 엄격히 지키고, 신버전이 양쪽 컬럼을 동기화하도록 코드 작성

</details>

<details>
<summary><strong>Q3: 컬럼 이름 변경이 안전한 경우는 어떤 경우인가?</strong></summary>

**답변**:

특정 조건이 모두 만족되면 직접 RENAME이 안전합니다.

```sql
안전한 조건 (모두 만족해야 함):

1. ORM 매핑 없음
   ├─ JPA @Column(name="...") 사용 안 함
   ├─ SQLAlchemy Column 사용 안 함
   └─ MyBatis <result column="..."> 사용 안 함
   └─ 결론: Raw JDBC 또는 Stored Procedure만 사용

2. 직접 쿼리 코드 없음
   ├─ SELECT username FROM ... 하드코딩 없음
   ├─ 동적 쿼리 빌더에서 컬럼명 하드코딩 없음
   └─ 결론: 모든 SELECT가 *를 사용하거나 명시적 별칭

3. 뷰/저장 프로시저 미참조
   └─ 또는 자동 RENAME 가능 (MySQL에서 기능 미제공)

4. 외부 도구 미참조
   ├─ BI 도구 (Tableau, Power BI)
   ├─ ETL 도구 (Apache Airflow)
   ├─ 데이터 파이프라인
   └─ 결론: 모두 없어야 함

5. Single-instance 또는 동시 배포 불가
   └─ 결론: Expand-Contract 필수 아님

실제 예:
-- 안전한 경우:
-- DB 전용 내부 컬럼 (앱이 직접 접근 안 함)
ALTER TABLE logs RENAME COLUMN created_ts TO created_at;
-- app이 logs 테이블을 직접 SELECT하지 않으면 안전

-- 위험한 경우:
-- ORM 매핑되는 컬럼
@Entity
@Table(name="users")
public class User {
  @Column(name="username")  // ← 이 경우 RENAME 위험!
}
ALTER TABLE users RENAME COLUMN username TO user_handle;  // ❌ Breaking
```

</details>

---

<div align="center">

**[⬅️ 이전: 안전한 컬럼 추가](./04-add-column-safely.md)** | **[홈으로 🏠](../README.md)** | **[다음: 인덱스 추가 ➡️](./06-add-index-safely.md)**

</div>
