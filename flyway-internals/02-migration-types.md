# 마이그레이션 유형 — Versioned, Repeatable, Undo

---

## 🎯 핵심 질문

1. V(Versioned) 마이그레이션과 R(Repeatable) 마이그레이션의 근본적인 차이는 무엇이고, 각각 언제 사용해야 하는가?
2. Repeatable 마이그레이션이 "항상 Versioned 이후에 실행"된다는 보장은 어떻게 이루어지는가?
3. 같은 버전의 Repeatable 마이그레이션이 여러 개 있으면 실행 순서는 어떻게 결정되는가?
4. Undo 마이그레이션(U)은 왜 Flyway Teams 전용이고, 커뮤니티 버전에서 롤백을 하려면 어떻게 해야 하는가?
5. Repeatable 마이그레이션을 수정했을 때, 언제 다시 실행되고 몇 번 실행될 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션 유형을 잘못 선택하면 같은 SQL이 여러 번 실행되거나 실행되지 않는 예측 불가능한 상황이 발생합니다. 예를 들어, 권한을 다시 부여하는 SQL을 한 번만 실행해야 하는데 Versioned으로 만들면 나중에 수정할 수 없고, Repeatable으로 만들면 매번 재실행됩니다. 또한 뷰나 저장 프로시저는 "생성" 구문이 아니라 "생성 또는 대체" 구문을 사용해야 하는데, 이를 모르면 마이그레이션이 계속 실패합니다.

Undo 마이그레이션을 이해하려면 Flyway의 철학을 알아야 합니다. Flyway는 "정방향만 가능"이라는 원칙을 가지고 있는데, 이는 한 번 적용된 마이그레이션은 되돌릴 수 없다는 뜻입니다. Teams 버전의 Undo는 예외이지만, 커뮤니티 버전을 사용해야 한다면 대안을 알아야 합니다.

---

## 😱 흔한 실수 (Before — ...)

```java
// ❌ 흔한 실수 1: 중복되는 이름으로 Repeatable 마이그레이션 생성
// R__grant_user_permissions.sql (version 1)
// R__grant_user_permissions.sql (version 2) ← 같은 이름!
// 
// 결과: 파일명이 같으면 같은 마이그레이션으로 간주
// Flyway는 어느 것을 실행할지 알 수 없음 → 에러 또는 예측 불가능한 동작
```

```sql
-- ❌ 흔한 실수 2: Repeatable 마이그레이션에서 CREATE 사용
-- R__create_views.sql (잘못된 방법)
CREATE VIEW user_summary AS
SELECT id, name, email FROM users;

-- 같은 파일을 두 번째로 실행하면:
-- ERROR: View 'user_summary' already exists
-- Repeatable은 "재실행 가능"해야 하는데, 이 구문은 불가능!
```

```sql
-- ❌ 흔한 실수 3: 버전 순서를 무시한 Repeatable
-- V1__initial.sql
CREATE TABLE users (id INT PRIMARY KEY);

-- R__grant_permissions.sql
GRANT SELECT ON users TO 'app_user';

-- 하지만 V2__add_admin_table.sql이 나중에 추가되면?
-- V1 → R → V2 순서로 실행 (R이 끼어있음!)
-- V2에서 새로 만든 테이블에는 권한이 없음 ← 버그!
```

```java
// ❌ 흔한 실수 4: Undo 마이그레이션을 역으로 사용
// U1__rollback_users_table.sql (커뮤니티 버전에서 작동 안 함)
DROP TABLE users;

// 커뮤니티 버전에서는 U 타입 마이그레이션을 무시함
// → 파일이 있어도 실행되지 않음
```

```yaml
# ❌ 흔한 실수 5: Repeatable의 실행 순서 오해
# classpath:db/migration/
# R__a_setup.sql
# R__b_grant.sql
# R__c_refresh_views.sql
#
# 알파벳 순서가 맞는데, 파일 수정 시 다시 실행되려면?
# R__a_setup.sql 수정 → a_setup만 재실행, b, c는 안 함!
# 모두 재실행되는 게 아님을 주의!
```

---

## ✨ 올바른 접근 (After — ...)

```sql
-- ✅ 올바른 접근 1: V와 R의 명확한 분리

-- V1__create_users_table.sql (Versioned - 한 번만)
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL,
  email VARCHAR(100) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- R__grant_user_permissions.sql (Repeatable - 매번 확인)
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO 'app_user'@'%';
GRANT SELECT ON users TO 'read_only'@'%';

-- V2__add_admin_user.sql (Versioned - 한 번만)
INSERT INTO users (name, email) VALUES ('Admin', 'admin@company.com');

-- R__create_user_views.sql (Repeatable - 매번 재생성)
CREATE OR REPLACE VIEW active_users AS
SELECT id, name, email FROM users WHERE created_at > DATE_SUB(NOW(), INTERVAL 30 DAY);
```

```sql
-- ✅ 올바른 접근 2: Repeatable에서 CREATE OR REPLACE 사용
-- R__create_views.sql
CREATE OR REPLACE VIEW user_summary AS
SELECT 
  COUNT(*) as total_users,
  MAX(created_at) as latest_join
FROM users;

-- 실행 결과:
-- 첫 실행: 뷰 생성
-- 체크섬 저장
-- 파일 수정 → 체크섬 변경
-- 두 번째 실행: 뷰 삭제 후 재생성 (OR REPLACE)
-- 또 다른 실행: 동일하게 재생성 (멱등성)
```

```sql
-- ✅ 올바른 접근 3: Repeatable 마이그레이션의 구체적 명시
-- R__01_grant_permissions.sql
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO 'app_user'@'%';

-- R__02_grant_audit_permissions.sql
GRANT SELECT ON audit_log TO 'app_user'@'%';

-- R__03_refresh_materialized_views.sql
REFRESH MATERIALIZED VIEW mv_user_stats;

-- R__04_update_search_index.sql
-- (외부 시스템 호출, Java 마이그레이션으로 구현)
```

```java
// ✅ 올바른 접근 4: 커뮤니티 버전에서 롤백 전략
// Undo 마이그레이션 대신 "역방향 마이그레이션" 버전 생성

// V1__create_users_table.sql
// CREATE TABLE users (...);

// V2__drop_users_table.sql (나중에 필요하면 사용)
// DROP TABLE users;
// 하지만 이건 "롤백"이 아니라 "정방향 마이그레이션"일 뿐

// 올바른 철학:
// - 마이그레이션은 항상 정방향만
// - 실수로 적용한 마이그레이션을 제거하려면 새 버전으로 역처리
// - "롤백"이라는 개념 자체를 피함
```

```yaml
# ✅ 올바른 접근 5: Flyway 설정
spring:
  flyway:
    locations: classpath:db/migration
    # V 마이그레이션: 버전 숫자 기준으로 정렬
    # R 마이그레이션: 파일명 알파벳 순서로 정렬
    # R은 항상 모든 V 이후에 실행
```

---

## 🔬 내부 동작 원리

### 1. Versioned(V) 마이그레이션 — 버전 고정, 한 번만

```java
// Flyway의 내부 처리 로직 (의사 코드)
public void migrate() {
    // 1단계: 모든 마이그레이션 파일 스캔
    List<Migration> migrations = scanMigrations();
    
    // V 마이그레이션: 버전 번호가 있음
    // V1__initial.sql → version = "1"
    // V2__add_index.sql → version = "2"
    // V3_1__add_column.sql → version = "3.1"
    
    // 2단계: 버전별로 정렬
    Collections.sort(migrations, (a, b) -> {
        // "1" < "2" < "3.1" < "10" (버전 비교, 문자 비교 아님)
        return compareVersions(a.getVersion(), b.getVersion());
    });
    
    // 3단계: 각 V 마이그레이션 실행
    for (Migration v : migrations.getVersionedMigrations()) {
        // 이미 적용됐나?
        MigrationRecord record = findInHistory(v.getVersion());
        if (record != null) {
            // 체크섬 확인
            if (record.checksum == v.calculateChecksum()) {
                // 동일 → 스킵
                continue;
            } else {
                // 체크섬 불일치!
                throw new ValidationFailedException(
                    "Checksum mismatch for version " + v.getVersion());
            }
        } else {
            // 미적용 → 실행
            v.execute();
            insertHistoryRecord(v);
        }
    }
}

// V 마이그레이션의 특징:
// 1) 한 번 적용되면 다시 실행 안 됨
// 2) 체크섬 불일치 시 오류 (수정 불가)
// 3) 버전이 고정되므로 버전 순서대로 실행 보장
// 4) 데이터 마이그레이션에 주로 사용
```

```
실행 흐름:
V1__initial.sql (버전 1.0)
   ↓
V2__add_column.sql (버전 2.0)
   ↓
V3__add_index.sql (버전 3.0)
   ↓
V10__add_user_role.sql (버전 10.0, 버전 비교!)
   ↓
... (모든 V 실행 완료)

이미 적용된 V는 스킵
```

### 2. Repeatable(R) 마이그레이션 — 버전 없음, 필요할 때마다

```java
// Repeatable 마이그레이션의 처리
public void migrateRepeatable() {
    // 1단계: R 마이그레이션만 필터링
    List<Migration> repeatables = migrations.stream()
        .filter(m -> m.type == MigrationType.REPEATABLE)
        .collect(toList());
    
    // R__grant_permissions.sql
    // R__create_views.sql
    // R__update_functions.sql
    
    // 2단계: 파일명 알파벳 순서로 정렬
    Collections.sort(repeatables, (a, b) -> {
        // "grant_permissions" < "create_views"
        return a.getDescription().compareTo(b.getDescription());
    });
    
    // 3단계: 각 R 마이그레이션 처리
    for (Migration r : repeatables) {
        // R 마이그레이션은 버전이 없음 (대신 description 사용)
        // key = "description_" + description
        MigrationRecord record = findInHistory("R", r.getDescription());
        
        if (record == null) {
            // 처음 실행 → 실행
            r.execute();
            insertHistoryRecord(r);
        } else {
            // 이미 적용됨 → 체크섬 확인
            if (record.checksum == r.calculateChecksum()) {
                // 체크섬 동일 → 파일 변경 없음, 스킵
                continue;
            } else {
                // 체크섬 다름 → 파일이 변경됨!
                // V와 달리 에러 아님, 다시 실행!
                r.execute();
                updateHistoryRecord(r);  // 체크섬만 업데이트
            }
        }
    }
}

// R 마이그레이션의 특징:
// 1) 한 번만 실행되지 않음
// 2) 파일 수정 → 체크섬 변경 → 재실행
// 3) 버전이 없으므로 V 이후 항상 실행
// 4) 멱등성이 보장되어야 함 (여러 번 실행 가능)
// 5) DDL(뷰, 함수) 또는 권한 부여에 주로 사용
```

```
실행 흐름:
V1__initial.sql (버전 1.0)
   ↓
V2__add_column.sql (버전 2.0)
   ↓
V3__add_index.sql (버전 3.0)
   ↓ (모든 V 실행 완료)
   ↓
R__01_grant_permissions.sql (알파벳 순)
   ↓
R__02_create_views.sql (알파벳 순)
   ↓
R__03_update_functions.sql (알파벳 순)
   ↓ (모든 R 실행 완료)

R 마이그레이션 수정:
R__01_grant_permissions.sql (체크섬 변경)
   ↓ (다시 실행됨)
```

### 3. Undo(U) 마이그레이션 — Teams 전용, 롤백

```java
// Undo는 Flyway Teams에서만 지원
// 커뮤니티 버전에서는 U 파일이 있어도 무시됨

// U1__rollback_create_users_table.sql
// (이 파일은 스캔되지만 실행되지 않음)

// Undo의 개념:
// V1__create_users_table.sql → CREATE
// U1__rollback_create_users_table.sql → DROP (역방향)

// Flyway Teams의 동작:
// - Command: flyway undo
// - 가장 최근의 V 마이그레이션을 롤백
// - 해당하는 U 마이그레이션 실행
// - flyway_schema_history에서 기록 삭제

// 커뮤니티 버전의 대안:
// - Undo 개념을 포기하고 정방향만 사용
// - 실수한 마이그레이션을 되돌리려면 새 V 마이그레이션 작성
public class CommunityVersionRollbackStrategy {
    // V1__create_users_table.sql
    // CREATE TABLE users (...);
    
    // 실수: 잘못된 컬럼 추가했음
    // 이제 해결하려면?
    
    // Option 1: 새 V 마이그레이션으로 수정
    // V2__fix_users_table.sql
    // ALTER TABLE users DROP COLUMN wrong_column;
    
    // Option 2: 심각한 실수면 복잡한 역변경
    // - 데이터 내보내기
    // - 테이블 삭제
    // - 새로운 버전의 전체 스키마 만들기
    // - 데이터 복원
    
    // → 정방향만 사용 권장!
}
```

---

## 💻 실전 실험

### 실험 1: V와 R 마이그레이션 함께 실행

```bash
# 디렉토리 구조 준비
mkdir -p db/migration

# V1__create_users_table.sql 생성
cat > db/migration/V1__create_users_table.sql << 'EOF'
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100) UNIQUE NOT NULL,
  name VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
EOF

# V2__add_roles_table.sql
cat > db/migration/V2__add_roles_table.sql << 'EOF'
CREATE TABLE roles (
  id INT PRIMARY KEY AUTO_INCREMENT,
  role_name VARCHAR(50) UNIQUE NOT NULL
);

ALTER TABLE users ADD COLUMN role_id INT;
ALTER TABLE users ADD CONSTRAINT fk_role FOREIGN KEY (role_id) REFERENCES roles(id);
EOF

# R__grant_permissions.sql
cat > db/migration/R__grant_permissions.sql << 'EOF'
-- Repeatable: 파일 수정 시마다 재실행
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO 'app_user'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON roles TO 'app_user'@'%';
EOF

# R__create_views.sql
cat > db/migration/R__create_views.sql << 'EOF'
CREATE OR REPLACE VIEW user_list AS
SELECT id, email, name FROM users;

CREATE OR REPLACE VIEW role_list AS
SELECT id, role_name FROM roles;
EOF

# Flyway 실행
flyway -url=jdbc:mysql://localhost:3306/testdb \
       -user=root -password=root \
       -locations=filesystem:./db/migration \
       migrate

# 출력:
# Database: MySQL 8.0
# Successfully validated 2 versioned migrations and 2 repeatable migrations
# Migrating to version 1
# Migrating to version 2
# Migrating repeatable migration: grant_permissions
# Migrating repeatable migration: create_views
# Successfully applied 4 migrations
```

### 실험 2: Repeatable 마이그레이션 수정 후 재실행

```bash
# R__grant_permissions.sql 수정 (읽기 권한 추가)
cat > db/migration/R__grant_permissions.sql << 'EOF'
-- Repeatable: 수정됨 → 재실행됨
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO 'app_user'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON roles TO 'app_user'@'%';
GRANT SELECT ON users TO 'read_only'@'%';  -- 새 라인!
GRANT SELECT ON roles TO 'read_only'@'%';  -- 새 라인!
EOF

# 다시 실행
flyway migrate

# 출력:
# Repeatable migration: grant_permissions has changed and should be re-applied
# Re-migrating repeatable migration: grant_permissions
# Successfully applied 1 migration
```

### 실험 3: flyway info로 현재 상태 확인

```bash
# 마이그레이션 상태 조회
flyway info

# 출력:
# +---------+---------+------------------------+------+---------------------+---------+
# | Category| Version | Description            | Type | Installed On        | State   |
# +---------+---------+------------------------+------+---------------------+---------+
# | Versioned |   1   | create users table     | SQL  | 2026-04-13 10:15:00 | Success |
# | Versioned |   2   | add roles table        | SQL  | 2026-04-13 10:15:01 | Success |
# |Repeatable| N/A    | grant_permissions      | SQL  | 2026-04-13 10:15:02 | Success |
# |Repeatable| N/A    | create_views           | SQL  | 2026-04-13 10:15:03 | Success |
# +---------+---------+------------------------+------+---------------------+---------+
```

### 실험 4: Repeatable 실행 순서 확인

```bash
# 파일 여러 개 생성
cat > db/migration/R__01_setup.sql << 'EOF'
-- 실행 1
SELECT 1;
EOF

cat > db/migration/R__02_grant.sql << 'EOF'
-- 실행 2
GRANT SELECT ON users TO 'app_user'@'%';
EOF

cat > db/migration/R__03_views.sql << 'EOF'
-- 실행 3
CREATE OR REPLACE VIEW user_summary AS SELECT COUNT(*) as count FROM users;
EOF

# SQL 로그를 확인할 수 있게 설정
flyway -url=jdbc:mysql://localhost:3306/testdb \
       -user=root -password=root \
       -locations=filesystem:./db/migration \
       info

# flyway_schema_history 확인
mysql -uroot -proot testdb -e "SELECT version, description, type FROM flyway_schema_history ORDER BY installed_rank;"

# 출력:
# version | description | type
# --------|-------------|-----
# 1       | create users table | SQL
# 2       | add roles table | SQL
# NULL    | 01_setup | SQL (R은 version이 NULL)
# NULL    | 02_grant | SQL
# NULL    | 03_views | SQL
```

### 실험 5: 파일명 순서 영향 확인

```bash
# R 마이그레이션의 파일명을 변경
# 이전: R__01_setup.sql, R__02_grant.sql
# 변경: R__grant.sql, R__setup.sql (역순!)

mv db/migration/R__01_setup.sql db/migration/R__setup_delayed.sql
mv db/migration/R__02_grant.sql db/migration/R__grant_first.sql

# 다시 실행
flyway migrate

# flyway_schema_history 다시 확인
mysql -uroot -proot testdb -e "SELECT description, installed_on FROM flyway_schema_history WHERE type='SQL' AND version IS NULL ORDER BY installed_rank DESC LIMIT 3;"

# 출력: (순서가 알파벳순으로 다시 정렬됨)
# description | installed_on
# grant_first | 2026-04-13 10:20:01
# setup_delayed | 2026-04-13 10:20:02
```

---

## 📊 성능/비용 비교

```
┌────────────────────────────────────────────────────────────────┐
│ 마이그레이션 유형별 실행 특성                                   │
├──────────┬──────────┬──────────┬──────────┬───────────────────┤
│ 유형     │ 버전관리 │ 재실행   │ 체크섬   │ 사용 시기         │
├──────────┼──────────┼──────────┼──────────┼───────────────────┤
│ V        │ 고정     │ 불가능   │ 필수     │ 스키마 변경       │
│(Versioned)         │ (오류)   │ (불일치) │ 데이터 초기 로드  │
├──────────┼──────────┼──────────┼──────────┼───────────────────┤
│ R        │ 없음     │ 필요시   │ 선택적   │ 뷰, 함수, 권한    │
│(Repeatable)        │ (파일    │ (파일    │ 시드 데이터       │
│          │          │ 수정시)  │ 비교시)  │                   │
├──────────┼──────────┼──────────┼──────────┼───────────────────┤
│ U        │ 고정     │ 불가능   │ 필수     │ 롤백 (Teams only) │
│ (Undo)   │ (V+순서) │ (한번)   │ (필수)   │ 커뮤니티 미지원   │
└──────────┴──────────┴──────────┴──────────┴───────────────────┘

실행 횟수 비교 (같은 파일 반복 실행 시):
┌─────────┬────────┬─────────────────────────┐
│ 유형    │ 1회차  │ 2회차 (파일 수정 안 함) │
├─────────┼────────┼─────────────────────────┤
│ V       │ 실행   │ 스킵 (적용됨)           │
│ R       │ 실행   │ 스킵 (체크섬 동일)      │
│ U(Teams)│ 실행   │ 불가능 (이미 롤백됨)   │
└─────────┴────────┴─────────────────────────┘

실행 횟수 비교 (파일 수정 후):
┌─────────┬────────┬─────────────────────────┐
│ 유형    │ 1회차  │ 2회차 (파일 수정함)     │
├─────────┼────────┼─────────────────────────┤
│ V       │ 실행   │ 오류! (체크섬 불일치)  │
│ R       │ 실행   │ 재실행 (체크섬 변경)    │
│ U(Teams)│ 실행   │ 불가능                 │
└─────────┴────────┴─────────────────────────┘

파일명 지정 규칙:
┌──────────────────────────────────────────────────────────┐
│ V1__create_users_table.sql                               │
│ ↓                                                         │
│ V (타입) + 1 (버전) + __ (구분자) + (설명).sql (파일명)  │
│                                                          │
│ R__grant_permissions.sql                                │
│ ↓                                                         │
│ R (타입) + __ (구분자) + (설명).sql (파일명, 버전 없음) │
│                                                          │
│ U1__rollback_users_table.sql (Teams only)               │
│ ↓                                                         │
│ U (타입) + 1 (버전) + __ + (설명).sql                   │
└──────────────────────────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
┌────────────────────────────────────────────────────────────────┐
│ Versioned(V) 마이그레이션                                      │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 버전이 고정되므로 순서 보장                    │
│              │ - 한 번 적용되면 다시 실행 안 됨               │
│              │ - 실수로 재적용될 위험 없음                    │
│              │ - 스키마 변경 추적 명확                        │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 일단 적용되면 수정 불가능                    │
│              │ - 수정하려면 새 V 마이그레이션 필요            │
│              │ - 파일 많아질 수 있음                          │
│              │ - 뷰/함수 변경이 어려움 (DROP+CREATE 필요)    │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Repeatable(R) 마이그레이션                                     │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 파일 수정 후 자동 재실행 가능                 │
│              │ - 뷰/함수 변경 간단                            │
│              │ - 권한 부여를 한 곳에서 관리                   │
│              │ - 멱등성 보장하면 반복 실행 안전               │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 멱등성을 개발자가 보장해야 함                 │
│              │ - 파일 수정 시 의도치 않은 재실행 가능         │
│              │ - 버전 순서 없어서 V와의 순서 관계 혼동        │
│              │ - 파일명 순서에 의존 (a < b < c)             │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Undo(U) 마이그레이션 (Teams only)                             │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 명시적 롤백 가능                              │
│              │ - Flyway가 자동으로 관리                       │
│              │ - flyway_schema_history 자동 정리              │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - Teams 라이선스 필요 (비용)                    │
│              │ - 커뮤니티 버전 사용자는 불가능                │
│              │ - 복잡한 데이터 롤백은 여전히 어려움           │
│              │ - "정방향만 진행"이 아니라 운영 복잡도 증가    │
└──────────────┴─────────────────────────────────────────────────┘

실무 권장:
┌────────────────────────────────────────────────────────────────┐
│ 1. V 마이그레이션: 데이터베이스 스키마 변경 (테이블, 인덱스)    │
│ 2. R 마이그레이션: 뷰, 함수, 권한, 시드 데이터                 │
│ 3. Undo: 사용하지 말 것 (정방향만 진행)                       │
│ 4. 실수한 마이그레이션: 새 V로 역처리 (DROP, DELETE 등)       │
└────────────────────────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. V(Versioned) - 버전 고정, 한 번만 실행                      │
│    - V1, V2, V3... (버전 번호 필수)                            │
│    - 체크섬 불일치 시 오류 발생                                 │
│    - 데이터 마이그레이션, 스키마 변경에 사용                     │
│                                                                 │
│ 2. R(Repeatable) - 버전 없음, 파일 수정 시 재실행              │
│    - R__, R__01, R__grant_permissions 등 (버전 불필요)        │
│    - 파일명 알파벳 순서로 정렬 (01 < 02 < grant < update)     │
│    - 뷰, 함수, 권한, 시드 데이터에 사용                        │
│    - 멱등성 필수 (CREATE OR REPLACE, DELETE + INSERT 등)      │
│                                                                 │
│ 3. R은 항상 모든 V 이후에 실행                                 │
│    - 실행 순서: V1 → V2 → V3 → R__a → R__b → R__c             │
│    - V가 추가되면: V1 → V2 → V3 → V4 → R__a → R__b → R__c    │
│                                                                 │
│ 4. Undo는 정방향만의 철학을 버림 (Teams 전용, 권장 안 함)      │
│    - 커뮤니티: 실수한 마이그레이션은 새 V로 역처리             │
│    - V1__add_column → V2__drop_column (정방향)                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🤔 생각해볼 문제

**Q1.** R__grant_permissions.sql을 수정해서 파일을 저장했다. 그러면 Flyway는 이 파일을 몇 번 실행하는가? 그리고 체크섬 업데이트는 몇 번 발생하는가?

<details>
<summary>해설 보기</summary>

**답: 1번 실행, 1번 체크섬 업데이트**

```
흐름:
1. R__grant_permissions.sql 파일 스캔
2. 현재 CRC32 체크섬 계산 (예: 12345)
3. DB 쿼리: SELECT checksum FROM flyway_schema_history 
           WHERE type='R' AND description='grant_permissions'
4. 저장된 체크섬 조회 (예: 11111, 이전 값)
5. 비교: 12345 ≠ 11111 → 파일 변경됨!
6. SQL 실행 (1번)
7. flyway_schema_history UPDATE (체크섬만 11111 → 12345)
8. 이후 다시 실행 시까지 대기
```

**주의사항:**
- Repeatable은 **파일마다 1회씩만** 재실행됨
- 여러 번 수정해도 매번 1회씩만 재실행 (누적 실행 아님)
- 하지만 그 과정에서 권한이 중복될 수 있으므로 주의

```sql
-- ❌ 위험한 구현
-- R__grant_permissions.sql (첫 실행)
GRANT SELECT ON users TO 'app_user'@'%';

-- 수정 후 (재실행)
GRANT SELECT ON users TO 'app_user'@'%';
GRANT INSERT ON users TO 'app_user'@'%';  -- 새 권한

-- 실행 결과: 중복 GRANT는 에러 아님 (MySQL은 무시)

-- ✅ 더 안전한 구현
-- R__grant_permissions.sql
REVOKE ALL PRIVILEGES ON users FROM 'app_user'@'%';  -- 기존 권한 제거
GRANT SELECT, INSERT ON users TO 'app_user'@'%';      -- 새로 부여
```

</details>

---

**Q2.** 다음 파일들의 실행 순서는 무엇인가? V1, R__create, R__01, V2, R__grant, V3

<details>
<summary>해설 보기</summary>

**답: V1 → V2 → V3 → R__01 → R__create → R__grant**

```
분류:
- V 마이그레이션: V1, V2, V3 (버전 번호 기준으로 정렬)
- R 마이그레이션: R__create, R__01, R__grant (파일명 알파벳 순서)

정렬:
- V: 1 < 2 < 3 (버전 비교)
- R: "01" < "create" < "grant" (알파벳 비교, "0" < "c" < "g")

따라서 최종 순서:
V1 (버전 1)
   ↓
V2 (버전 2)
   ↓
V3 (버전 3)
   ↓ (모든 V 완료)
   ↓
R__01 (알파벳 "01")
   ↓
R__create (알파벳 "create")
   ↓
R__grant (알파벳 "grant")
```

**주의: Repeatable 파일명의 함정**
```
❌ 틀린 명명
R__1_setup.sql
R__2_grant.sql
R__10_views.sql

정렬 결과: "1_setup" < "10_views" < "2_grant" (알파벳!)
→ 예상과 다른 순서!

✅ 올바른 명명
R__01_setup.sql
R__02_grant.sql
R__10_views.sql

정렬 결과: "01_setup" < "02_grant" < "10_views"
→ 예상대로 순서 유지
```

</details>

---

**Q3.** V1__create_users_table.sql을 이미 프로덕션에 적용했는데, 나중에 테이블명을 users에서 accounts로 변경해야 한다. 올바른 방법은?

<details>
<summary>해설 보기</summary>

**답: 새로운 V2 마이그레이션을 만들어서 정방향 처리**

```
❌ 절대 하면 안 될 것:
// V1__create_users_table.sql 수정
CREATE TABLE accounts (  -- 테이블명 변경
  id INT PRIMARY KEY AUTO_INCREMENT,
  ...
);

// Flyway 재실행 시:
// ERROR: Validate failed: Migrations have failed validation
// Migration checksum mismatch for migration version 1
→ 프로덕션 DB는 이미 'users' 테이블이 있음 (accounts 아님)
→ 복구 불가능한 상태
```

```
✅ 올바른 방법: 새로운 V2 작성
// V1__create_users_table.sql (기존, 건드리지 않음)
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100) UNIQUE NOT NULL
);

// V2__rename_users_to_accounts.sql (새로 생성)
ALTER TABLE users RENAME TO accounts;

// 또는 더 세밀한 제어:
// V2__create_accounts_and_migrate.sql
CREATE TABLE accounts LIKE users;           -- 동일 구조 복제
INSERT INTO accounts SELECT * FROM users;   -- 데이터 이동
-- 필요하면 foreign key 수정, view 수정 등
-- DROP TABLE users; (나중에 필요하면)
```

**왜 정방향만 가능한가?**
- V1이 이미 프로덕션 DB에 적용됨 (users 테이블 생성됨)
- V1을 "롤백"할 수 없음 (데이터 손상 위험)
- 따라서 V2로 "정방향" 수정 (users → accounts 변경)
- 이것이 Flyway의 철학

**롤백이 꼭 필요하다면?**
- 프로덕션이 아닌 개발 환경: flyway clean (모두 삭제) → 재시작
- 프로덕션: 불가능 (데이터 손상)
- Flyway Teams Undo: 비용 + 복잡도 증가

</details>

---

<div align="center">

**[⬅️ 이전: Flyway 내부 동작 원리](./01-flyway-schema-history.md)** | **[홈으로 🏠](../README.md)** | **[다음: Flyway 설정 옵션 완전 분석 ➡️](./03-flyway-config-options.md)**

</div>
