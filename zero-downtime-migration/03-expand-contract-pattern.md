# Expand-Contract 패턴

---

## 🎯 핵심 질문

- Expand-Contract는 왜 필요한가? 왜 한 번에 스키마를 변경하지 않는가?
- 배포 순서가 왜 중요한가? (DB 변경 먼저 vs 앱 배포 먼저)
- Blue-Green 배포 환경에서 Expand-Contract를 어떻게 적용하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

많은 팀이 개발 환경에서는 DB와 앱을 동시에 배포하므로 문제가 없습니다. 하지만 프로덕션에서는:

1. **구버전 앱과 신버전 DB가 공존**: 블루-그린 배포 중 구버전 인스턴스가 여전히 요청 처리
2. **배포 중단 없음**: Kubernetes 롤링 업데이트, AWS ALB 무중단 배포
3. **스키마 변경과 배포 분리**: DB 변경은 모든 앱 인스턴스에 영향

Expand-Contract를 모르면:
- 구버전 앱이 없는 컬럼에 접근하려고 시도 → 오류
- 신버전 앱이 있어야 할 컬럼을 찾을 수 없음 → NullPointerException

---

## 😱 흔한 실수 (Before — 동시 배포 시도)

```sql
-- Before: 한 번에 스키마 변경
-- 프로덕션 DB:
ALTER TABLE users DROP COLUMN username;
ALTER TABLE users ADD COLUMN full_name VARCHAR(100);

-- 동시에 앱 배포:
-- 배포 중 상황:
-- ├─ Pod 1 (구버전): SELECT username FROM users ← 컬럼 없음!
-- ├─ Pod 2 (구버전): SELECT username FROM users ← 컬럼 없음!
-- ├─ Pod 3 (신버전): SELECT full_name FROM users ← 아직 없음!
-- └─ Pod 4 (신버전): SELECT full_name FROM users ← 아직 없음!

-- 결과:
-- Column 'username' doesn't exist
-- Column 'full_name' doesn't exist
-- → SLA 위반, 사용자 요청 실패 급증
```

**문제점**:
1. DDL 실행 중 구버전 앱이 컬럼 없음 오류
2. DDL 완료 후 신버전 배포 전, 신버전은 새 컬럼을 찾을 수 없음
3. 배포 순서를 잘못 이해하면 문제 발생

---

## ✨ 올바른 접근 (After — 3단계 Expand-Contract)

```sql
-- 1단계: EXPAND - 새 컬럼 추가 (구버전 앱도 무시할 수 있도록)
-- DB 배포 (모든 앱 인스턴스에 영향 없음)
ALTER TABLE users 
ADD COLUMN full_name VARCHAR(100) NULL;  -- NULL로 시작, 기본값 없음

-- 확인: 구버전 앱은 여전히 username 사용
SELECT username FROM users;  -- OK (full_name은 무시)
SELECT full_name FROM users;  -- NULL 반환 (앱이 처리 필요)

-- 2단계: 마이그레이션 - 기존 데이터를 새 컬럼으로 복사
-- (백필, 구버전과 신버전 앱이 공존하는 동안 수행)
-- 배치 작업으로 점진적 진행
UPDATE users 
SET full_name = CONCAT(first_name, ' ', last_name)
WHERE full_name IS NULL
LIMIT 10000;

-- 배시 스크립트로 반복:
-- while true; do
--   mysql -e "UPDATE users SET full_name = ... WHERE full_name IS NULL LIMIT 10000;"
--   if [ $(mysql -e "SELECT COUNT(*) FROM users WHERE full_name IS NULL;") -eq 0 ]; then
--     break
--   fi
--   sleep 5
-- done

-- 3단계: CONTRACT - 앱 배포 + 구 컬럼 제거
-- 3-1. 앱 배포 (신버전: full_name만 사용)
-- 블루 → 그린 배포 완료
-- 모든 Pod가 신버전

-- 3-2. 구 컬럼 제거 (이제 안전함, 앱이 사용하지 않으므로)
ALTER TABLE users DROP COLUMN username;
```

**결과**: 무중단 배포, 스키마 변경 완료

---

## 🔬 내부 동작 원리

### 1. Expand-Contract 3단계 상세

```
┌──────────────────────────────────────────────────────────────┐
│ 초기 상태: 구버전 앱, 기존 스키마                            │
├──────────────────────────────────────────────────────────────┤
│ DB Schema:                                                   │
│ ├─ id INT                                                    │
│ ├─ username VARCHAR(100)                                     │
│ ├─ first_name VARCHAR(50)                                    │
│ └─ last_name VARCHAR(50)                                     │
│                                                              │
│ App Code:                                                    │
│ public User getUser(Long id) {                               │
│   return template.queryForObject(                            │
│     "SELECT id, username FROM users WHERE id = ?",          │
│     id                                                       │
│   );                                                         │
│ }                                                            │
└──────────────────────────────────────────────────────────────┘

                            EXPAND 단계

┌──────────────────────────────────────────────────────────────┐
│ DB Schema (변경됨):                                          │
│ ├─ id INT                                                    │
│ ├─ username VARCHAR(100)  ← 아직 사용 중                     │
│ ├─ first_name VARCHAR(50)                                    │
│ ├─ last_name VARCHAR(50)                                     │
│ └─ full_name VARCHAR(100) NULL  ← 신규 추가 (NULL)          │
│                                                              │
│ App Code: (여전히 구버전)                                     │
│ SELECT id, username FROM users;  ← full_name 무시, OK       │
│                                                              │
│ 동시에 백필 진행:                                            │
│ UPDATE users SET full_name = CONCAT(...)                     │
│   WHERE full_name IS NULL LIMIT 10000;                       │
└──────────────────────────────────────────────────────────────┘

                      MIGRATE 단계 (진행 중)

┌──────────────────────────────────────────────────────────────┐
│ DB Schema (변경 진행):                                       │
│ ├─ id INT                                                    │
│ ├─ username VARCHAR(100)                                     │
│ ├─ first_name VARCHAR(50)                                    │
│ ├─ last_name VARCHAR(50)                                     │
│ └─ full_name VARCHAR(100)  ← 점진적으로 데이터 채워짐        │
│                                                              │
│ App Code: (구버전과 신버전 공존)                             │
│ 구버전: SELECT username FROM users;  ← OK                   │
│ 신버전: SELECT full_name FROM users;  ← 일부만 채워짐, OK  │
│                                                              │
│ 백필 상태: 70% 완료                                          │
└──────────────────────────────────────────────────────────────┘

                      MIGRATE 단계 (완료)

┌──────────────────────────────────────────────────────────────┐
│ DB Schema:                                                   │
│ ├─ id INT                                                    │
│ ├─ username VARCHAR(100)  ← 사용 종료됨                      │
│ ├─ first_name VARCHAR(50)                                    │
│ ├─ last_name VARCHAR(50)                                     │
│ └─ full_name VARCHAR(100)  ← 모두 채워짐 (0% NULL)          │
│                                                              │
│ App Code: (신버전으로 완전 전환)                             │
│ SELECT full_name FROM users;  ← 모두 값 있음, OK            │
│                                                              │
│ 백필 상태: 100% 완료, 구 컬럼 제거 준비 완료               │
└──────────────────────────────────────────────────────────────┘

                       CONTRACT 단계

┌──────────────────────────────────────────────────────────────┐
│ 최종 DB Schema:                                              │
│ ├─ id INT                                                    │
│ ├─ first_name VARCHAR(50)  ← 더 이상 필요 없음              │
│ ├─ last_name VARCHAR(50)   ← 더 이상 필요 없음              │
│ └─ full_name VARCHAR(100)  ← 이제 유일한 이름 컬럼          │
│                                                              │
│ App Code: (신버전)                                           │
│ public User getUser(Long id) {                               │
│   return template.queryForObject(                            │
│     "SELECT id, full_name FROM users WHERE id = ?",         │
│     id                                                       │
│   );                                                         │
│ }                                                            │
└──────────────────────────────────────────────────────────────┘
```

### 2. 배포 순서의 중요성 (Expand-Contract에서)

#### 시나리오 A: 잘못된 순서 (DB 변경 → 앱 배포)

```
T1: DB 변경 (EXPAND)
    ALTER TABLE users ADD COLUMN full_name VARCHAR(100) NULL;
    
T2~T5: 블루-그린 배포 진행 (Pod 하나씩)
    └─ 문제 발생!
    
Timeline:
┌─────────────────────────────────────┐
│ T1: DB EXPAND                       │
│     Full_name 컬럼 추가             │
├─────────────────────────────────────┤
│ T2-T4: 구버전 Pod 아직 실행 중      │
│ 이때 새 INSERT/UPDATE 발생          │
│ ├─ 구버전: username 설정, full_name 무시
│ └─ DB: full_name은 NULL             │
├─────────────────────────────────────┤
│ T5: 신버전 Pod 시작                 │
│ 신버전: full_name 사용 시작         │
│ └─ 일부 행의 full_name은 NULL      │
│    (구버전이 설정하지 않았으므로)    │
├─────────────────────────────────────┤
│ T6-T10: 백필 시작                   │
│ UPDATE users SET full_name = ...    │
│ WHERE full_name IS NULL LIMIT 10000;│
│                                     │
│ 이 사이에 구버전이 INSERT 하면:     │
│ full_name = NULL (again!)           │
└─────────────────────────────────────┘

최악의 경우: 백필이 완료되지 않은 상태에서 컬럼 제거
ALTER TABLE users DROP COLUMN username;  # 너무 서둘렀음!
→ 신버전은 full_name 사용 중, full_name이 모두 채워지지 않음
→ 데이터 손실 가능성
```

#### 시나리오 B: 올바른 순서 (앱 배포 먼저 → DB 변경)

```
T1~T4: 앱 배포 먼저 (신버전, 양쪽 컬럼 모두 읽고 씀)
    ├─ Pod 1: 신버전 (username, full_name 모두 사용)
    ├─ Pod 2: 신버전
    ├─ Pod 3: 신버전
    └─ Pod 4: 신버전 (완전 전환)
    
T5: DB EXPAND (모든 Pod가 신버전이므로 안전)
    ALTER TABLE users ADD COLUMN full_name VARCHAR(100) NULL;
    
T6~T10: 백필 (신버전만 실행 중이므로 안전)
    UPDATE users SET full_name = username WHERE full_name IS NULL;
    
T11: 확인 후 구 컬럼 제거 (모든 데이터 full_name에 있음)
     ALTER TABLE users DROP COLUMN username;
     
T12: 선택적 배포 (신버전이 full_name만 사용하도록 정리)
     (구버전도 full_name을 지원하므로 이미 가능)
```

**올바른 순서**:
```
1. 앱 배포 (신버전: 새 컬럼 읽기 + 쓰기 모두 지원, 양쪽 모두 사용)
2. DB EXPAND (새 컬럼 추가)
3. 백필 (데이터 복사)
4. 앱 재배포 (신버전: 새 컬럼만 사용, 구 컬럼 무시)
5. DB CONTRACT (구 컬럼 제거)
```

### 3. JPA Entity 코드 패턴

```java
// EXPAND 단계: 새 필드 추가, 양쪽 모두 읽고 씀
@Entity
@Table(name = "users")
public class User {
    @Id
    private Long id;
    
    // 구 필드 (계속 매핑)
    @Column(name = "username")
    private String username;
    
    // 신규 필드 (EXPAND 단계에 추가)
    @Column(name = "full_name", nullable = true)  // NULL 허용
    private String fullName;
    
    // 비즈니스 로직: 양쪽 모두 사용
    public void setName(String username, String fullName) {
        this.username = username;
        this.fullName = fullName;
    }
    
    // getter는 fullName을 우선 사용
    public String getName() {
        return fullName != null ? fullName : username;
    }
}

// DB: INSERT/UPDATE 시 양쪽 컬럼에 쓰기
// username과 fullName을 동시에 설정
User user = new User();
user.setName("john", "John Doe");  // 양쪽에 쓰기
repository.save(user);

// 백필 이후 (MIGRATE 단계):
// 구버전 앱이 fullName이 없는 행을 INSERT할 수 없음 (이미 모두 배포됨)
// INSERT 시:
INSERT INTO users (id, username, full_name) 
VALUES (1, 'john', 'John Doe');  -- 양쪽 모두 설정됨

// CONTRACT 단계 이후: 
@Entity
@Table(name = "users")
public class User {
    @Id
    private Long id;
    
    // 구 필드 제거 (유지 시간 감소)
    // @Column(name = "username")  ← 제거됨
    // private String username;     ← 제거됨
    
    // 신규 필드만 매핑
    @Column(name = "full_name", nullable = false)  // NOT NULL
    private String fullName;
    
    public String getName() {
        return fullName;  // 이제는 하나만 사용
    }
}
```

---

## 💻 실전 실험

### 실험 1: 3단계 SQL 파일 작성 및 실행

```sql
-- phase1-expand.sql
-- 실행 시점: DB 배포 (앱 배포 전)
ALTER TABLE users 
ADD COLUMN full_name VARCHAR(100) NULL,
ALGORITHM=INSTANT;

-- 확인: 구버전 앱 호환성
SELECT id, username FROM users;  -- OK
SELECT id, full_name FROM users;  -- NULL 반환
```

```sql
-- phase2-migrate.sql
-- 실행 시점: 앱 배포 후, 백필 작업
-- 배치 크기: 10000행씩 (한 번에 몇 초 정도 소요)

-- 배치 1회 실행
UPDATE users 
SET full_name = CONCAT(
  COALESCE(first_name, ''), 
  ' ', 
  COALESCE(last_name, '')
)
WHERE full_name IS NULL
LIMIT 10000;

-- 결과 확인
SELECT COUNT(*) as remaining_nulls FROM users WHERE full_name IS NULL;

-- 이 쿼리를 5분마다 반복 실행 (확인 쿼리가 0을 반환할 때까지)
-- while [ $(mysql -e "SELECT COUNT(*) FROM users WHERE full_name IS NULL;") -gt 0 ]; do
--   mysql < phase2-migrate.sql
--   sleep 300  # 5분 대기
-- done
```

```sql
-- phase3-contract.sql
-- 실행 시점: 백필 완료 후, 모든 Pod가 신버전 배포 완료

-- 사전 확인: NULL이 더 이상 없는지 확인
SELECT COUNT(*) as null_count FROM users WHERE full_name IS NULL;
-- 결과: 0

-- 구 컬럼 제거
ALTER TABLE users 
DROP COLUMN username,
ALGORITHM=INPLACE;

-- 새 컬럼을 NOT NULL로 변경 (이제 안전함)
ALTER TABLE users 
MODIFY COLUMN full_name VARCHAR(100) NOT NULL,
ALGORITHM=INSTANT;
```

### 실험 2: 배포 순서 시뮬레이션 (Bash + MySQL)

```bash
#!/bin/bash
# migration-workflow.sh

DB_HOST="127.0.0.1"
DB_USER="root"
DB_PASS="password"
DB_NAME="testdb"

# 색상 정의
BLUE='\033[0;34m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m'  # No Color

# 1단계: EXPAND
echo -e "${BLUE}=== 1. EXPAND 단계: 새 컬럼 추가 ===${NC}"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
ALTER TABLE users 
ADD COLUMN full_name VARCHAR(100) NULL,
ALGORITHM=INSTANT;
SHOW COLUMNS FROM users;
EOF

echo -e "${YELLOW}구버전 앱이 username 사용 가능한지 확인:${NC}"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
SELECT COUNT(*) as username_count FROM users;
EOF

# 2단계: MIGRATE (배치 시뮬레이션)
echo -e "${BLUE}=== 2. MIGRATE 단계: 데이터 백필 ===${NC}"

# 먼저 테스트 데이터 삽입 (구버전 앱이 username만 설정한 것으로 가정)
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
INSERT INTO users (username, first_name, last_name, full_name)
VALUES 
('john', 'John', 'Doe', NULL),
('jane', 'Jane', 'Smith', NULL),
('bob', 'Bob', 'Johnson', NULL);
EOF

echo -e "${YELLOW}백필 전: full_name이 NULL인 행의 수${NC}"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
SELECT COUNT(*) as null_count FROM users WHERE full_name IS NULL;
EOF

# 배치 백필 시뮬레이션 (10행씩)
for i in {1..5}; do
  echo -e "${YELLOW}배치 $i 실행 중...${NC}"
  mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
UPDATE users 
SET full_name = CONCAT(first_name, ' ', last_name)
WHERE full_name IS NULL
LIMIT 10;
EOF
  
  remaining=$(mysql -h $DB_HOST -u $DB_USER -p$DB_PASS -se \
    "SELECT COUNT(*) FROM $DB_NAME.users WHERE full_name IS NULL;")
  echo -e "${YELLOW}남은 NULL: $remaining${NC}"
done

echo -e "${GREEN}백필 완료${NC}"

# 3단계: CONTRACT
echo -e "${BLUE}=== 3. CONTRACT 단계: 구 컬럼 제거 ===${NC}"
mysql -h $DB_HOST -u $DB_USER -p$DB_PASS $DB_NAME << EOF
-- 최종 확인
SELECT COUNT(*) as null_count FROM users WHERE full_name IS NULL;

-- 구 컬럼 제거
ALTER TABLE users 
DROP COLUMN username,
ALGORITHM=INPLACE;

-- 최종 스키마 확인
SHOW COLUMNS FROM users;
EOF

echo -e "${GREEN}=== 마이그레이션 완료 ===${NC}"
```

### 실험 3: Blue-Green 배포 시뮬레이션

```java
// 구버전 앱 코드
// BlueVersion.java
public class UserService {
    public void createUser(String username, String firstName, String lastName) {
        User user = new User();
        user.setUsername(username);  // 구 컬럼에만 쓰기
        userRepository.save(user);
    }
    
    public User getUser(Long id) {
        User user = userRepository.findById(id).orElse(null);
        // 구 컬럼만 읽음 (full_name이 있어도 무시)
        System.out.println(user.getUsername());
        return user;
    }
}

// 신버전 앱 코드
// GreenVersion.java
public class UserService {
    public void createUser(String username, String firstName, String lastName) {
        User user = new User();
        user.setUsername(username);  // 구 컬럼 (호환성)
        user.setFullName(firstName + " " + lastName);  // 신규 컬럼도 쓰기
        userRepository.save(user);
    }
    
    public User getUser(Long id) {
        User user = userRepository.findById(id).orElse(null);
        // 신규 컬럼 우선 (EXPAND-CONTRACT 패턴)
        System.out.println(
            user.getFullName() != null 
            ? user.getFullName() 
            : user.getUsername()
        );
        return user;
    }
}

// 배포 순서:
// T1: Pod 1 → Blue (구버전)
// T2: Pod 2 → Green (신버전)
// T3: Pod 3 → Green
// T4: Pod 4 → Green (완전 전환)
// T5: DB EXPAND (이제 안전)
// T6: 백필 시작
// ...
```

---

## 📊 성능/비용 비교

| 전략 | 다운타임 | 배포 횟수 | 복잡도 | 위험도 |
|------|--------|----------|--------|--------|
| **Expand-Contract** | 0분 | 2~3회 | 높음 | 낮음 |
| **동시 배포** | 5~10분 | 1회 | 낮음 | 높음 |
| **야간 배포 + 동기화 대기** | 0분 | 1회 | 중간 | 중간 |

**시간 계산** (Expand-Contract):
```
EXPAND:    1분 (INSTANT, Lock 무시할 수준)
MIGRATE:   10분 (배치 처리, 동시 DML 허용)
CONTRACT:  1분 (INSTANT, Lock 무시할 수준)
──────────────────
총 소요:   12분

vs 동시 배포: 30분 (Lock 발생, DML 차단)
```

---

## ⚖️ 트레이드오프

| 선택지 | 장점 | 단점 |
|--------|------|------|
| **Expand-Contract** | 무중단, 명확한 단계 | 배포 횟수 증가, 코드 복잡도 |
| **동시 배포** | 간단, 1회 배포 | 다운타임 발생, 롤백 어려움 |
| **Dual Write** (구+신 컬럼 동시 쓰기) | 중간 버전, 마이그레이션 안전 | 응용 로직 복잡, 코드 가독성 저하 |

---

## 📌 핵심 정리

1. **Expand-Contract**: 구버전과 신버전 앱이 공존하는 블루-그린 배포 필수 패턴
2. **3단계**:
   - EXPAND: 새 컬럼 추가 (NULL)
   - MIGRATE: 기존 데이터 복사 (배치)
   - CONTRACT: 구 컬럼 제거
3. **배포 순서**: 앱 배포 먼저 (신버전이 양쪽 컬럼 지원) → DB 변경 → 백필
4. **무중단**: 각 단계 Lock 없음, 다운타임 0분
5. **JPA 패턴**: @Column 양쪽 매핑, 비즈니스 로직에서 우선순위 설정

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: EXPAND 단계에서 새 컬럼을 NULL이 아닌 DEFAULT 값으로 추가하면 어떻게 되는가?</strong></summary>

**답변**:

DEFAULT 값이 있으면 기존 행에도 자동으로 값이 할당되므로, 백필이 필요 없어질 수 있습니다.

```sql
-- Case 1: NULL (권장)
ALTER TABLE users 
ADD COLUMN full_name VARCHAR(100) NULL;
-- 기존 1000만 행: full_name = NULL
-- 신규 INSERT: full_name은 앱이 설정하거나 NULL
-- 필요: 백필 (UPDATE로 기존 행에 값 할당)

-- Case 2: DEFAULT 값
ALTER TABLE users 
ADD COLUMN full_name VARCHAR(100) DEFAULT 'Unknown';
-- 기존 1000만 행: full_name = 'Unknown' (자동 할당!)
-- 신규 INSERT: full_name = 'Unknown' (앱이 overwrite하지 않으면)
-- 문제: 
-- - 1. 모든 기존 행의 full_name이 'Unknown' → 데이터 정확도 낮음
-- - 2. 신규 INSERT도 'Unknown'이면 구버전 앱이 INSERT한 것인지
--      신버전 앱이 INSERT한 것인지 구분 불가
-- 해결: 여전히 UPDATE로 real value로 변경 필요

-- Case 3: DEFAULT 없고 NOT NULL (위험)
ALTER TABLE users 
ADD COLUMN full_name VARCHAR(100) NOT NULL;
-- Error: All rows must have a value for full_name
-- 기존 행을 처리할 방법이 없으므로 ADD COLUMN 자체 실패

결론:
NULL DEFAULT를 선택하고, 백필로 진짜 값을 채우는 것이 올바른 방식
DEFAULT 값이 있으면 데이터 품질 저하 및 마이그레이션 복잡도 증가
```

</details>

<details>
<summary><strong>Q2: 백필 중에 구버전 앱이 새 컬럼을 건드리면(UPDATE username)어떻게 되는가?</strong></summary>

**답변**:

구버전 앱은 새 컬럼 full_name을 모르므로, 구 컬럼 username만 업데이트합니다.

```
Timeline:

T1: 백필 배치 1 실행
    UPDATE users 
    SET full_name = CONCAT(first_name, ' ', last_name)
    WHERE full_name IS NULL
    LIMIT 10000;
    
    처리된 행: ID 1~10000, full_name = 'John Doe', 'Jane Smith' 등

T2: 구버전 앱이 ID 5000의 사용자를 업데이트
    UPDATE users SET username = 'john_new' WHERE id = 5000;
    
    결과:
    ├─ username: 'john_new' (변경됨)
    └─ full_name: 'John Doe' (이미 채워짐, 변경 안 됨)
    
    문제: username과 full_name이 일치하지 않음!
    - username: 'john_new'
    - full_name: 'John Doe' (old value)

T3: 신버전 앱이 이 행을 읽으면:
    SELECT full_name FROM users WHERE id = 5000;
    → 'John Doe' (잘못된 값!)

해결책 1: dual-write (신버전 배포 후)
신버전 배포 후 username 변경 금지 (이미 모든 Pod가 신버전)
→ 문제 해결

해결책 2: 추가 로직
신버전이 username도 읽고 비교
if (full_name != null && !fullName.contains(username.split('_')[0])) {
    // 불일치 감지, 로그 기록, 알림
}

해결책 3: 강제 양쪽 업데이트 (신버전 배포 전)
신버전 배포 시점에 앱이 username 변경 시 full_name도 동시 업데이트
User user = repository.findById(5000);
user.setUsername('john_new');
user.setFullName('John New');  // 동시 업데이트
repository.save(user);
```

**권장**: 배포 순서를 엄격히 지키고, 신버전 배포 완료 후에 구 컬럼 DROP

</details>

<details>
<summary><strong>Q3: 어떤 상황에서는 Expand-Contract 없이 한 번에 배포해도 되는가?</strong></summary>

**답변**:

특정 조건이 모두 만족되면 Expand-Contract 없이 동시 배포 가능합니다.

```sql
조건 1: Single-instance 배포
- 마이크로서비스 아키텍처에서 1개 Pod만 실행
- 또는 서버리스 (Lambda, Cloud Functions)
- 가능한 이유: 구버전 앱이 없으므로 호환성 걱정 불필요

조건 2: Scheduled downtime 허용
- 배포 중 서비스 중단 가능
- 야간 배포 + 30분 유지보수 시간
- 가능한 이유: 구버전과 신버전 공존 기간 없음

조건 3: 스키마 변경이 Additive만 (필드 제거 없음)
- 새 컬럼 추가 (기본값 포함)
- 새 테이블 생성
- 인덱스 추가
- 불가능한 이유: 필드 제거는 구버전이 참조하면 오류

조건 4: API 버전 관리
- /api/v1 (구버전)과 /api/v2 (신버전) 동시 운영
- 구버전은 구 스키마, 신버전은 신 스키마 사용
- 가능한 이유: 각 API가 독립적인 데이터 접근

현실적으로:
- 대부분의 프로덕션 환경: Expand-Contract 필수
- 조건 1~4가 모두 만족: Expand-Contract 선택사항
- 의심스러우면: Expand-Contract 사용 (안전)
```

</details>

---

<div align="center">

**[⬅️ 이전: Online DDL과 gh-ost](./02-online-ddl-gh-ost.md)** | **[홈으로 🏠](../README.md)** | **[다음: 안전한 컬럼 추가 ➡️](./04-add-column-safely.md)**

</div>
