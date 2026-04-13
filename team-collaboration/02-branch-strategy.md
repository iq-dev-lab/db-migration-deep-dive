# 브랜치 전략과 마이그레이션

---

## 🎯 핵심 질문

Feature 브랜치에서 마이그레이션을 작성할 때 어떤 순서로 진행해야 안전할까요? 장기 브랜치에서 마이그레이션하면 왜 문제가 생기고, Squash Merge 시 주의해야 할 점은 무엇일까요?

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션은 Git 커밋과 다르게, 일단 데이터베이스에 적용되면 **되돌리기 어렵습니다**. 따라서 브랜치 전략이 매우 중요합니다. 잘못된 브랜칭으로 인해 스키마 불일치, 데이터 손실, 또는 배포 실패가 발생할 수 있습니다. 특히 마이그레이션과 비즈니스 로직이 뒤섞여 있으면 배포 순서가 꼬일 수 있습니다.

---

## 😱 흔한 실수 (Before)

### 시나리오 1: 장기 브랜치에서의 마이그레이션

**상황:** 개발자가 3개월간 feature 브랜치에서 작업 중

```bash
# 2024-01-15 시작
$ git checkout -b feature/user-management main
# 해당 시점 마이그레이션: V5까지 적용됨

# 3개월간 열심히 작업...
# (main에서는 V5 → V15까지 변경됨)

# 2024-04-15 PR 준비
$ git log --oneline -5
c3f4a5 feat: add user roles table
d2e3b4 feat: add user permissions table
c1f2a3 feat: add user preferences column
b0e1f2 feat: add user authentication
a9d0e1 migration: V6__Add_users_table.sql  ← 오래전 마이그레이션!
```

**문제:**

1. **스키마 진화가 놓침**
   ```sql
   -- main의 현재 users 테이블 (V15까지 적용 후)
   SELECT * FROM users;
   -- 컬럼: id, email, phone, address, status, role, permission_id, ...
   
   -- Feature 브랜치의 V6 마이그레이션
   CREATE TABLE users (
       id BIGINT PRIMARY KEY,
       email VARCHAR(255)
   );
   -- ❌ V6 이후 main에서 추가된 컬럼들이 없음!
   ```

2. **의존성 문제**
   ```sql
   -- main의 V10: user_roles 테이블 생성
   -- Feature의 V6: users 테이블 생성 (V10 없음)
   -- Feature의 V7: user_permissions 생성 (user_roles 외래키 참조)
   -- ❌ 실행 순서: V6 → V7 → V10 (외래키 오류!)
   ```

3. **데이터 마이그레이션 불일치**
   ```
   main에서 users.status 처리:
   V8: users.status VARCHAR(50) 추가
   V12: users.status → ENUM으로 변경
   
   Feature 브랜치:
   V6: users 테이블 생성 (status 없음)
   V7: 별도로 status 추가 (VARCHAR)
   ❌ main의 V12가 적용 안 됨 → ENUM 변환 누락!
   ```

### 시나리오 2: Squash Merge로 커밋 합치기

**상황:** Feature 브랜치의 여러 마이그레이션을 하나로 squash

```bash
# Feature 브랜치
$ git log --oneline
c3f4a5 feat: add user roles
c2f3a4 migration: V7__Add_roles_table.sql
c1f2a3 feat: add permissions
c0f1a2 migration: V6__Add_permissions_table.sql

# Main 브랜치로 돌아가서
$ git checkout main
$ git merge --squash feature/user-management
$ git commit -m "feat: add user roles and permissions"
# ❌ 마이그레이션 파일들도 하나의 커밋으로 합쳐짐!
```

**DB 영향:**

```sql
-- Squash 전 마이그레이션 히스토리
flyway_schema_history:
V6__Add_permissions_table.sql
V7__Add_roles_table.sql

-- Squash 후?
git log --oneline -1
a9d0e1 feat: add user roles and permissions

-- Git에는 파일 변경이 기록되지만,
-- 별도의 마이그레이션 커밋이 아님
-- → CI/CD에서 마이그레이션을 놓칠 수 있음!
```

### 시나리오 3: 버전 번호 꼬임

**상황:** Squash merge 후 마이그레이션 번호가 겹침

```bash
# Feature A (먼저 머지됨)
V5__20240415_143000__Add_users.sql
V5__20240415_144530__Add_orders.sql  ← V5 두 개

# Feature B (나중에 머지됨)
V5__20240415_145000__Add_products.sql  ← 또 V5!

# 정렬 결과
V5__20240415_143000
V5__20240415_144530
V5__20240415_145000 (or 중복 감지)
```

---

## ✨ 올바른 접근 (After)

### 1. Feature 브랜치 마이그레이션 최적 워크플로우

**단계별 가이드:**

```bash
# ============================================
# 1단계: main 브랜치에서 최신 상태 동기화
# ============================================
$ git checkout main
$ git pull origin main
$ git log --oneline -3  # 최신 커밋 확인

# ============================================
# 2단계: 현재 마이그레이션 상태 확인
# ============================================
$ ls -la db/migration/ | tail -10

# 현재 최신 마이그레이션이 V15라고 하자
# V15__20240415_100000__Add_payment_methods.sql
```

```bash
# ============================================
# 3단계: Feature 브랜치 생성
# ============================================
# ❌ 잘못된 순서: 브랜치 먼저 생성
# git checkout -b feature/user-roles
# 마이그레이션 작성
# → 다른 개발자가 먼저 main에 머지되면 버전 밀림!

# ✅ 올바른 순서: 마이그레이션부터 파악
$ git checkout -b feature/user-roles origin/main

# ============================================
# 4단계: 현재 시간으로 마이그레이션 파일명 결정
# ============================================
# 현재 시간: 2024-04-15 14:35:42
# 버전 계획: V16

$ cat > db/migration/V16__20240415_143542__Add_user_roles_table.sql << 'EOF'
CREATE TABLE user_roles (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    role_name VARCHAR(50) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_user_roles_name ON user_roles(role_name);
EOF

# ============================================
# 5단계: 기능 구현과 마이그레이션은 분리해서 커밋
# ============================================
$ git add db/migration/V16__20240415_143542__Add_user_roles_table.sql
$ git commit -m "migration: add user roles table

- Create user_roles table with role_name, description
- Add index on role_name
- Version: V16"

# 기능 구현 (별도 커밋)
$ cat > src/main/java/com/example/UserRole.java << 'EOF'
@Entity
@Table(name = "user_roles")
public class UserRole {
    @Id
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String roleName;
    
    private String description;
}
EOF

$ git add src/
$ git commit -m "feat: add UserRole entity with repository"
```

```bash
# ============================================
# 6단계: PR 생성 및 로컬 검증
# ============================================
$ git push origin feature/user-roles
# GitHub에서 PR 생성

# 로컬에서 main과의 차이 확인
$ git fetch origin
$ git diff origin/main...HEAD

# ============================================
# 7단계: PR 머지 전 최종 검증 (중요!)
# ============================================
# PR이 오래 열려 있으면 main이 변경될 수 있음
$ git fetch origin
$ git rebase origin/main  # 또는 merge

# ❌ 충돌 발생?
# git status로 확인

# ============================================
# 8단계: 로컬에서 마이그레이션 재실행으로 검증
# ============================================
$ mvn flyway:validate
$ mvn flyway:info  # 현재 상태 확인

$ mvn clean test  # 통합 테스트 실행
```

### 2. Squash Merge 시 마이그레이션 관리

**마이그레이션이 포함된 PR은 Squash하지 않기:**

```yaml
# .github/merge-strategy.yml (팀 규칙)
rules:
  - pattern: "db/migration/"
    strategy: "NO_SQUASH"
    reason: "마이그레이션은 개별 커밋으로 유지"
  
  - pattern: "src/"
    strategy: "SQUASH"
    reason: "기능 구현은 하나로 합침"
```

**만약 Squash가 필요하면:**

```bash
# ❌ 마이그레이션을 포함해서 Squash
git merge --squash feature/user-roles

# ✅ 마이그레이션만 별도로 처리
# 1. 마이그레이션 커밋만 cherry-pick
git cherry-pick <migration-commit-hash>

# 2. 기능 구현만 squash merge
git merge --squash feature/user-roles

# 또는 대화형 rebase
git rebase -i origin/main
# p (pick)    migration: V16
# s (squash)  feat: add UserRole
# s (squash)  feat: add repository
```

### 3. Trunk-Based Development에서의 마이그레이션

장기 브랜치를 피하고 매일 main에 머지:

```bash
# 매일 아침
$ git pull origin main

# 작은 단위의 기능과 마이그레이션 (1-2일 작업)
$ git checkout -b feature/add-user-status
# 마이그레이션 + 기능 구현
$ git push origin feature/add-user-status
$ # PR → Review → Merge (당일)

# 다음 날
$ git checkout main
$ git pull origin main  # 기어 업데이트
$ git checkout -b feature/add-user-role  # 새 기능
```

**장점:**
- 마이그레이션 충돌 최소화
- 스키마 드리프트 방지
- 빠른 피드백

---

## 🔬 내부 동작 원리

### 1. Git 브랜치와 마이그레이션 파일 추적

```
main 브랜치 (시간 흐름)
│
├─ Commit A: V1__Initial.sql (2024-01-01)
├─ Commit B: V2__Add_users.sql (2024-01-15)
│
└─ Commit C: V3__Add_orders.sql (2024-02-01)
    │
    └─ feature-old 브랜치 (2024-02-01에서 생성)
       │
       ├─ Commit C': V4__Custom_feature.sql (2024-02-05)
       ├─ Commit D': Code implementation (2024-02-20)
       ├─ Commit E': More code (2024-03-15)
       └─ Commit F': Final code (2024-04-15)  ← 3개월 후!

# 이 시점에 main:
# ├─ Commit C: V3__Add_orders.sql
# ├─ Commit D: V4__Add_products.sql (다른 팀원)
# ├─ Commit E: V5__Add_reviews.sql
# ├─ Commit F: V6__Modify_users.sql (users 스키마 변경!)
# ...
# └─ Commit Z: V15__Add_payment_methods.sql

# feature-old를 main에 머지하면?
# → V3 이후 V4 (feature)가 아니라
#   V3 → V4~V15 (main) → V4 (feature)로 실행됨!
```

### 2. 마이그레이션 파일의 체크섬 검증

Flyway는 마이그레이션 파일을 **MD5 체크섬**으로 추적:

```sql
-- flyway_schema_history
+----------+---------+---------+--------------------+----------+
| version  | script  | status  | checksum           | installed|
+----------+---------+---------+--------------------+----------+
| 6        | V6__... | SUCCESS | 2934f8a934...     | 20240415 |
| 7        | V7__... | SUCCESS | a8c9d7f2b1...     | 20240415 |
+----------+---------+---------+--------------------+----------+

# Feature 브랜치의 V6이 main의 V6과 다르면?
# checksum mismatch → 오류!
```

**Squash merge 후:**

```bash
# 커밋 기록은 합쳐졌지만
$ git log --oneline
a9d0e1 feat: add user management (squashed)

# 마이그레이션 파일은 실제로 존재함
$ ls db/migration/V6* db/migration/V7*
V6__Add_permissions.sql
V7__Add_roles.sql

# Flyway는 파일 시스템을 스캔하므로 정상 작동
$ mvn flyway:info
V6 ✓
V7 ✓
```

### 3. 마이그레이션 순서 결정 알고리즘

Flyway의 **정렬 규칙:**

```java
// 마이그레이션 버전 비교
class MigrationVersion implements Comparable<MigrationVersion> {
    public int compareTo(MigrationVersion other) {
        // 1. 주 버전 비교
        if (this.major != other.major) {
            return Integer.compare(this.major, other.major);
        }
        
        // 2. 부 버전 비교
        if (this.minor != other.minor) {
            return Integer.compare(this.minor, other.minor);
        }
        
        // 3. 마이크로 버전 비교 (타임스탐프 등)
        return this.micro.compareTo(other.micro);
    }
}

// 정렬 결과
V1__Initial.sql
V2__20240101_100000__Add_users.sql      ← 타임스탐프 있음
V2__20240101_120000__Add_orders.sql
V3__Add_products.sql
V3__20240415_143000__Fix_products.sql
```

---

## 💻 실전 실험

### 실험 1: 장기 브랜치 시뮬레이션 및 문제 재현

**초기 설정:**

```bash
# Git 저장소 초기화
mkdir migration-demo && cd migration-demo
git init

# main 브랜치에서 초기 마이그레이션 커밋
mkdir db/migration

cat > db/migration/V1__Initial_schema.sql << 'EOF'
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
EOF

git add db/migration/
git commit -m "migration: initial schema (V1)"
```

**main 브랜치에서의 진화 (3개월):

```bash
# 2024-01-15
git log --oneline | head -1
# a1b2c3d migration: initial schema (V1)

# 이 때 feature-old 브랜치 생성
git checkout -b feature-old

# 3개월 동안 feature에서 작업...
cat > db/migration/V2__20240215_100000__Add_user_roles.sql << 'EOF'
CREATE TABLE user_roles (
    id BIGINT PRIMARY KEY,
    role_name VARCHAR(50)
);
ALTER TABLE users ADD COLUMN role_id BIGINT;
EOF

git add db/migration/
git commit -m "migration: add user roles (V2)"

# 구현
git commit --allow-empty -m "feat: add UserRole entity"
git commit --allow-empty -m "feat: add role repository"
git commit --allow-empty -m "feat: add role service"
```

**main에서의 동시 진화:**

```bash
# feature-old에서 벗어남
git checkout main

# 다른 팀원들이 계속 커밋
cat > db/migration/V2__20240120_100000__Add_products_table.sql << 'EOF'
CREATE TABLE products (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(255),
    price DECIMAL(10, 2)
);
EOF

git add db/migration/
git commit -m "migration: add products table (V2)"

cat > db/migration/V3__20240201_100000__Add_orders_table.sql << 'EOF'
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    total_amount DECIMAL(10, 2),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
EOF

git add db/migration/
git commit -m "migration: add orders table (V3)"

# ... 더 많은 마이그레이션과 기능
for i in {4..10}; do
    cat > db/migration/V$i"__2024"0$((i+10))"_100000__Schema_update_$i.sql" << EOF
-- Schema update V$i
ALTER TABLE users ADD COLUMN field_$i VARCHAR(100);
EOF
    git add db/migration/
    git commit -m "migration: schema update V$i"
done
```

**충돌 상황:**

```bash
# feature-old를 main에 머지 시도
git merge feature-old

# Git 수준에서는 파일 충돌 없음 (파일명이 다름)
# 하지만...

$ ls db/migration/ | sort
V1__Initial_schema.sql
V2__20240120_100000__Add_products_table.sql    ← main
V2__20240215_100000__Add_user_roles.sql        ← feature (다른 파일!)
V3__20240201_100000__Add_orders_table.sql
V4__20240204_100000__Schema_update_4.sql
...
V10__20240212_100000__Schema_update_10.sql
```

**문제:**

```sql
-- users 테이블 (main의 V10 적용 후)
DESC users;
+--------------------+
| Field              |
+--------------------+
| id                 |
| email              |
| created_at         |
| field_4            |
| field_5            |
| ...                |
| field_10           |
+--------------------+

-- Feature의 V2에서 예상하는 users 테이블
ALTER TABLE users ADD COLUMN role_id BIGINT;
-- field_4 ~ field_10이 없음!
-- Foreign key constraint: role_id 참조 테이블이 없을 수 있음!
```

### 실험 2: 올바른 브랜칭 워크플로우 검증

```bash
# ============================================
# 상황: feature-hotfix를 main에 동기화하며 작업
# ============================================

git checkout main
git pull origin main

# 최신 마이그레이션 확인
$ ls db/migration/ | sort | tail -3
V8__Add_payment_methods.sql
V9__Add_invoices.sql
V10__Add_audit_log.sql

# Feature 생성 (이 시점의 스키마 상태 포함)
git checkout -b feature/hotfix/critical-bug

# 마이그레이션 필요 없음 (버그 수정만)
# 하지만 필요한 경우:
cat > db/migration/V11__20240415_150000__Add_bug_fix_field.sql << 'EOF'
ALTER TABLE audit_log ADD COLUMN bug_category VARCHAR(50);
EOF

git add db/migration/
git commit -m "migration: add bug_category to audit_log"

# 버그 수정
echo "bug fix" > src/fix.java
git add src/
git commit -m "fix: critical bug in payment processing"

# PR 준비: main과 재동기화
git fetch origin
git rebase origin/main

# 충돌 없음! (feature 브랜치가 짧음)
$ git log --oneline origin/main..HEAD
a1b2c3d migration: add bug_category to audit_log
d4e5f6g fix: critical bug in payment processing
```

### 실험 3: Squash Merge vs Regular Merge 비교

```bash
# ============================================
# 마이그레이션 포함 Feature
# ============================================

git checkout main
git pull origin main

git checkout -b feature/analytics

# 마이그레이션 1
cat > db/migration/V11__20240415_100000__Add_analytics_tables.sql << 'EOF'
CREATE TABLE page_views (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    page_url VARCHAR(500),
    viewed_at TIMESTAMP
);

CREATE TABLE user_events (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT,
    event_type VARCHAR(100),
    event_data JSON,
    created_at TIMESTAMP
);
EOF

git add db/migration/
git commit -m "migration: add analytics tables (V11)"

# 마이그레이션 2
cat > db/migration/V12__20240415_110000__Add_analytics_indexes.sql << 'EOF'
CREATE INDEX idx_page_views_user_id ON page_views(user_id);
CREATE INDEX idx_page_views_created ON page_views(viewed_at);
CREATE INDEX idx_user_events_user_id ON user_events(user_id);
EOF

git add db/migration/
git commit -m "migration: add analytics indexes (V12)"

# 기능 구현
cat > src/Analytics.java << 'EOF'
public class AnalyticsService {
    public void trackPageView(String userId, String url) {}
    public void trackEvent(String userId, String event) {}
}
EOF

git add src/
git commit -m "feat: add AnalyticsService"

git log --oneline
c3f4a5 feat: add AnalyticsService
c2f3a4 migration: add analytics indexes (V12)
c1f2a3 migration: add analytics tables (V11)
```

**Regular Merge (권장):**

```bash
git checkout main
git merge feature/analytics --no-ff

git log --oneline
c3f4a5 Merge branch 'feature/analytics'
c2f3a4 feat: add AnalyticsService
c1f2a3 migration: add analytics indexes (V12)
c0f1a2 migration: add analytics tables (V11)

# ✅ 마이그레이션이 모두 유지됨
$ mvn flyway:info | grep V1[12]
| V11 | SUCCESS |
| V12 | SUCCESS |
```

**Squash Merge (위험):**

```bash
git checkout main
git merge --squash feature/analytics
git commit -m "feat: add analytics feature"

git log --oneline
a9d0e1 feat: add analytics feature (squashed)

# 파일 시스템에는 마이그레이션 파일이 있지만
ls db/migration/V1[12]*
V11__Add_analytics_tables.sql
V12__Add_analytics_indexes.sql

# CI/CD 스크립트가 마이그레이션 커밋만 추적하면 놓칠 수 있음!
git log --grep="migration:" | grep V1[12]
# (empty) ← 마이그레이션 커밋이 없음!
```

---

## 📊 성능/비용 비교

| 브랜칭 전략 | 마이그레이션 충돌 | 스키마 드리프트 | 배포 복잡성 | 권장도 |
|-----------|----------------|--------------|----------|------|
| **장기 브랜치** (3개월+) | 매우 높음 | 매우 높음 | 매우 복잡 | ❌❌❌ |
| **중기 브랜치** (2-4주) | 중간 | 중간 | 중간 | ⚠️ |
| **단기 브랜치** (1-3일) | 낮음 | 낮음 | 간단함 | ✅✅✅ |
| **Trunk-Based** (당일) | zero | zero | 최소 | ✅✅✅ |

**권장:** Trunk-Based Development 또는 최대 1주일 브랜치

---

## ⚖️ 트레이드오프

### 1. 빠른 개발 vs 안전한 마이그레이션

```
장기 브랜치 (3개월)
├─ 장점: 기능이 완성될 때까지 독립적 개발
└─ 단점: 마이그레이션 충돌, 스키마 불일치
        → 배포 시간 30배 증가 가능

Trunk-Based (매일 머지)
├─ 장점: 충돌 최소화, 빠른 피드백
└─ 단점: 기능이 미완성된 상태로도 머지 필요
        → Feature flag 또는 API versioning 필요
```

**권장:** Trunk-Based + Feature Flag

### 2. 코드 품질 vs 배포 속도

```
Regular Merge
├─ 각 커밋이 보존됨
├─ History 추적 용이
└─ 마이그레이션 추적 명확

Squash Merge
├─ 커밋 수 감소
├─ History 간결
└─ 마이그레이션 추적 어려움 ❌
```

**마이그레이션 포함 시:** Regular Merge 필수

---

## 📌 핵심 정리

1. **Feature 브랜치 수명**: 최대 1주일 유지, 매일 main과 동기화
2. **마이그레이션 먼저**: main pull → feature 생성 → 마이그레이션 작성 → 기능 구현
3. **PR 전 재검증**: 머지 전에 `git rebase origin/main`으로 충돌 확인
4. **Squash 금지**: 마이그레이션 포함 PR은 regular merge 사용
5. **Trunk-Based 권장**: 여러 팀의 경우 매일 main에 머지
6. **CI 자동화**: 마이그레이션 커밋 감지 및 자동 검증

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: Release 브랜치에서 긴급 마이그레이션이 필요하면 어떻게 할까요?</strong></summary>

**시나리오:**
```
main (V10까지 적용)
├─ release/1.0 (V5까지 적용, 배포 준비 중)
└─ hotfix/critical (V11 긴급 마이그레이션 필요)
```

**문제:** V6~V10이 release 브랜치에 없음

**해결책:**

```bash
# 1. Release 브랜치에 cherry-pick
git checkout release/1.0

# 필요한 마이그레이션만 선택
git cherry-pick <V6-commit> <V7-commit> ... <V11-commit>

# 또는 2. Main에서 백포트
git merge main --no-ff

# 또는 3. 새 마이그레이션으로 통합
cat > db/migration/V6__20240415_160000__Backport_fixes.sql << 'EOF'
-- main의 V6-V11을 하나로 통합
-- (주의: 유연성 떨어짐)
EOF
```

**교훈:** Release 브랜치는 최소화, main에서 병합.

</details>

<details>
<summary><strong>Q2: Rebase vs Merge - 마이그레이션에서는 어느 것이 안전할까요?</strong></summary>

**Rebase 사용 시:**

```bash
git checkout main
git pull origin main

git checkout feature/users
git rebase origin/main

# 결과: feature의 커밋이 main 최신 위에 재정렬
# 마이그레이션 파일명은 변경 없음 ✓
# 하지만 history가 변경됨 (force push 필요)
git push origin feature/users --force-with-lease

# 다른 개발자의 feature-orders 브랜치는?
# feature-orders는 여전히 구 main 기반
# → 충돌 가능성 ⚠️
```

**Merge 사용 시:**

```bash
git checkout main
git pull origin main

git checkout feature/users
git merge origin/main

# 결과: 병합 커밋 생성
# 모든 개발자가 같은 main 기반 사용
# → 충돌 최소화 ✓

git push origin feature/users
```

**권장:** Squash merge 금지 + Regular merge 사용

</details>

<details>
<summary><strong>Q3: 여러 마이그레이션을 포함한 대규모 PR을 머지할 때 주의할 점은?</strong></summary>

**시나리오:**
```
feature/major-refactor (V5 → V20, 15개 마이그레이션)
├─ V5__Add_table1.sql
├─ V6__Add_table2.sql
├─ V7__Alter_table1.sql
...
└─ V20__Final_schema_update.sql
```

**위험:**
1. **하나의 마이그레이션이 실패하면** 나머지가 실행 안 됨
2. **Rollback 계획 없음** - 어디부터 되돌릴지 불명확
3. **배포 중 데이터 손실** 가능

**해결책:**

```yaml
# 1. 마이그레이션 검증 (PR 단계)
validation:
  - flyway:validate
  - sqlfluff lint  # SQL 스타일 검사
  - custom-checks: no-drop-table, no-truncate

# 2. 원자성(Atomicity) 검증
checks:
  - each-migration-can-rollback: true
  - foreign-keys-preserved: true

# 3. 순서 검증
  - versions-sequential: true
```

```bash
# 4. 배포 전 테스트 DB에서 리드리 (dry run)
mvn flyway:repair  # 실패한 마이그레이션 정리
mvn flyway:validate
mvn flyway:migrate
mvn test

# 5. 문제 발견 시 롤백 계획
# V5-V10은 성공했는데 V11에서 실패?
# → 롤백은 권장하지 않음, 대신 V11을 수정해서 V21로 생성
```

**교훈:** 대규모 마이그레이션은 작은 단위로 분할, 각각 테스트.

</details>

---

<div align="center">

**[⬅️ 이전: 마이그레이션 버전 충돌](./01-version-conflict.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이그레이션 코드 리뷰 체크리스트 ➡️](./03-code-review-checklist.md)**

</div>
