# Java 기반 마이그레이션

---

## 🎯 핵심 질문

1. `BaseJavaMigration`을 상속받아 마이그레이션을 구현할 때, 반드시 구현해야 하는 3가지 메서드는 무엇인가?
2. SQL로는 표현할 수 없는 마이그레이션 예시 3가지와 Java로 해결하는 방법은?
3. Java 마이그레이션의 버전과 실행 순서는 SQL 마이그레이션과 어떻게 섞이는가?
4. `isTransactional()` 메서드가 반환하는 값에 따라 마이그레이션 동작이 어떻게 달라지는가?
5. Spring Bean을 Java 마이그레이션에서 사용할 수 없는 이유는 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대부분의 마이그레이션은 SQL로 충분하지만, 실무에서는 SQL만으로 불가능한 경우가 많습니다. 예를 들어, JSON 데이터를 파싱해서 여러 컬럼으로 분리해야 한다면? 또는 외부 API를 호출해서 데이터를 보강해야 한다면? 또는 복잡한 비즈니스 로직으로 기존 데이터를 변환해야 한다면? 이런 경우 Java 마이그레이션이 필수입니다.

또한 Java 마이그레이션은 Spring Boot의 자동 설정 메커니즘과 다르게 동작하므로, 잘못 이해하면 Spring Bean을 주입하려다가 `NullPointerException`을 받게 됩니다. 트랜잭션 처리도 SQL 마이그레이션과 다르며, 버전 관리도 복잡합니다. 이 장에서는 Java 마이그레이션의 실제 동작 원리와 함정을 배웁니다.

---

## 😱 흔한 실수 (Before — ...)

```java
// ❌ 흔한 실수 1: Spring의 자동 설정 가정
public class V2__ProcessJsonData extends BaseJavaMigration {
    
    @Autowired  // ← Spring Bean 주입을 시도
    private UserRepository userRepository;
    
    @Override
    public void migrate(Context context) throws Exception {
        // NullPointerException 발생!
        // userRepository는 null
        // 왜냐하면 Java 마이그레이션은 Spring Context 외부에서 실행되기 때문
    }
}
```

```java
// ❌ 흔한 실수 2: 마이그레이션 순서 오해
// V1__create_table.sql
CREATE TABLE users (id INT PRIMARY KEY, raw_json TEXT);

// V2__process_json.java (버전 2)
// SQL: V1 → V2
// Java: V2 실행
// 
// 하지만 만약 다음 파일이 추가되면?
// R__migrate_json_data.java (Repeatable)
// 
// 실행 순서:
// V1 (SQL, 버전 1)
// V2 (Java, 버전 2)
// V3 (SQL, 버전 3) ← 새로 추가
// R__migrate (Repeatable)
//
// V2가 생성한 컬럼을 V3이 삭제하면?
// → V2 (Java 마이그레이션)이 V3보다 먼저 실행되므로 문제!

// ❌ 흔한 실수 3: 트랜잭션 미처리
public class V2__ProcessJsonData extends BaseJavaMigration {
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        
        // isTransactional() 기본값: true
        // 따라서 자동 트랜잭션
        
        Statement stmt = conn.createStatement();
        stmt.execute("UPDATE users SET processed = true");
        
        // 만약 다음 줄에서 오류 발생
        throw new Exception("Something went wrong");
        
        // 자동 ROLLBACK되지 않음!
        // 기본적으로 Flyway는 자동 ROLLBACK을 관리
        // 하지만 isTransactional()을 무시하고 자체 트랜잭션 관리하면?
        // → 혼동 가능
    }
}

// ❌ 흔한 실수 4: SQL 공격 (쿼리 포함)
public class V2__ProcessJsonData extends BaseJavaMigration {
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        Statement stmt = conn.createStatement();
        
        String userId = "1'; DROP TABLE users; --";  // SQL injection!
        stmt.execute("SELECT * FROM users WHERE id = " + userId);
        // → 테이블 삭제!
    }
}

// ❌ 흔한 실수 5: 복잡한 로직을 마이그레이션에
public class V5__ReferralBonusCalculation extends BaseJavaMigration {
    @Override
    public void migrate(Context context) throws Exception {
        // 수백 줄의 복잡한 비즈니스 로직
        // 추천인별 보너스 계산
        // 월별 수수료 정산
        // 환율 변환
        // ... (너무 복잡)
        
        // 문제:
        // - 단위 테스트 불가능
        // - 수정 불가능 (이미 프로덕션 적용)
        // - 디버깅 어려움
    }
}
```

---

## ✨ 올바른 접근 (After — ...)

```java
// ✅ 올바른 접근 1: BaseJavaMigration 상속 및 필수 메서드 구현
public class V2__ProcessJsonData extends BaseJavaMigration {
    
    // 필수 메서드 1: 버전 반환
    @Override
    public Integer getChecksum() {
        // 이 메서드는 checksum 계산
        // 파일 기반이 아니라 메서드 기반
        // 보통 null 반환 (Flyway가 자동 계산)
        return null;
    }
    
    // 필수 메서드 2: 실행 로직
    @Override
    public void migrate(Context context) throws Exception {
        // Context에서 직접 Connection 획득
        Connection conn = context.getConnection();
        
        try (Statement stmt = conn.createStatement()) {
            // SQL 실행
            stmt.execute("UPDATE users SET processed = true WHERE raw_json IS NOT NULL");
        } catch (SQLException e) {
            // 예외 처리
            throw new RuntimeException("Failed to process JSON data", e);
        }
    }
    
    // 선택 메서드 3: 트랜잭션 여부 (기본값: true)
    @Override
    public boolean isTransactional() {
        // true: 자동 트랜잭션 (권장)
        // false: 자동 트랜잭션 미사용 (DDL이 필요한 경우)
        return true;
    }
    
    // 선택 메서드 4: 버전 반환
    @Override
    public String getVersion() {
        // 버전을 명시적으로 지정할 수 있음
        // 파일명에서 추출하지 않으므로 수동 지정
        return "2";
    }
    
    // 선택 메서드 5: 설명 반환
    @Override
    public String getDescription() {
        // 마이그레이션 설명
        return "Process JSON data in users table";
    }
}
```

```java
// ✅ 올바른 접근 2: Prepared Statement로 SQL Injection 방지
public class V3__LoadUserData extends BaseJavaMigration {
    
    @Override
    public String getVersion() { return "3"; }
    
    @Override
    public String getDescription() { return "Load user data from CSV"; }
    
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        
        // ❌ 절대 하지 말 것
        // String query = "INSERT INTO users VALUES ('" + userId + "')";
        
        // ✅ 올바른 방법: PreparedStatement 사용
        String sql = "INSERT INTO users (id, name, email) VALUES (?, ?, ?)";
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setInt(1, 1);
            pstmt.setString(2, "Alice");
            pstmt.setString(3, "alice@example.com");
            pstmt.executeUpdate();
        }
    }
}
```

```java
// ✅ 올바른 접근 3: 트랜잭션 관리
public class V4__MigrateJsonToColumns extends BaseJavaMigration {
    
    @Override
    public String getVersion() { return "4"; }
    
    @Override
    public String getDescription() { return "Migrate JSON to separate columns"; }
    
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        
        // isTransactional() = true (기본값)
        // Flyway가 자동으로 다음을 관리:
        // - BEGIN TRANSACTION
        // - 마이그레이션 실행
        // - 성공 시: COMMIT
        // - 실패 시: ROLLBACK
        
        try (Statement stmt = conn.createStatement()) {
            // Step 1: 임시 컬럼 추가
            stmt.execute("ALTER TABLE users ADD COLUMN first_name VARCHAR(100)");
            
            // Step 2: 데이터 처리
            stmt.execute("UPDATE users SET first_name = JSON_EXTRACT(json_data, '$.firstName')");
            
            // Step 3: 검증
            try (ResultSet rs = stmt.executeQuery("SELECT COUNT(*) as cnt FROM users WHERE first_name IS NULL")) {
                rs.next();
                int nullCount = rs.getInt("cnt");
                if (nullCount > 0) {
                    throw new RuntimeException("JSON migration failed: " + nullCount + " records without first_name");
                }
            }
        }
        // 모든 단계 성공 → 자동 COMMIT
        // 어느 단계 실패 → 자동 ROLLBACK
    }
    
    @Override
    public boolean isTransactional() {
        return true;  // 트랜잭션 자동 관리
    }
}
```

```java
// ✅ 올바른 접근 4: DDL 마이그레이션 (isTransactional = false)
public class V5__CreateStoredProcedure extends BaseJavaMigration {
    
    @Override
    public String getVersion() { return "5"; }
    
    @Override
    public String getDescription() { return "Create stored procedure"; }
    
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        
        // DDL은 일부 DB에서 트랜잭션 불가능
        // MySQL: ALTER TABLE, CREATE VIEW 등은 implicit commit
        // PostgreSQL: 트랜잭션 가능하지만 복잡함
        
        try (Statement stmt = conn.createStatement()) {
            String sql = "CREATE PROCEDURE get_user_summary() " +
                        "BEGIN " +
                        "  SELECT COUNT(*) as user_count FROM users; " +
                        "END";
            stmt.execute(sql);
        }
    }
    
    @Override
    public boolean isTransactional() {
        return false;  // DDL 때문에 트랜잭션 비활성화
    }
}
```

---

## 🔬 내부 동작 원리

### 1. BaseJavaMigration의 메서드 구조

```java
public abstract class BaseJavaMigration {
    
    // 필수 구현
    public abstract void migrate(Context context) throws Exception;
    
    // 선택 구현 (기본값)
    public String getVersion() {
        return null;  // 파일명에서 추출 (V2__description → 2)
    }
    
    public String getDescription() {
        return null;  // 파일명에서 추출 (V2__description → description)
    }
    
    public Integer getChecksum() {
        return null;  // 클래스명으로 자동 계산
    }
    
    public boolean isTransactional() {
        return true;  // 트랜잭션 자동 관리
    }
}

// Context 인터페이스
public interface Context {
    // DB 커넥션 획득 (Flyway가 관리)
    Connection getConnection();
    
    // 다른 메서드들
    int getDataSourceInfo();
}
```

### 2. Java 마이그레이션 버전과 실행 순서

```java
// 파일 구조:
// src/main/resources/db/migration/
// ├── V1__create_table.sql
// ├── V2__ProcessJsonData.java
// ├── V3__add_column.sql
// └── V4__DataMigration.java

// 버전 추출:
// V1__create_table.sql → version = "1"
// V2__ProcessJsonData.java → getVersion() 반환값 (null이면 클래스명으로 추출 시도)
// V3__add_column.sql → version = "3"
// V4__DataMigration.java → getVersion() 반환값

// 실행 순서:
// 1. V1__create_table.sql (SQL, 버전 1)
// 2. V2__ProcessJsonData.java (Java, 버전 2)
// 3. V3__add_column.sql (SQL, 버전 3)
// 4. V4__DataMigration.java (Java, 버전 4)
// → 버전 번호로 통합 정렬! (타입 무관)

// Repeatable도 포함:
// 1. V1 (SQL)
// 2. V2 (Java)
// 3. V3 (SQL)
// 4. V4 (Java)
// 5. R__views.sql (Repeatable)
// 6. R__permissions.java (Repeatable Java)
```

```java
// Flyway의 내부 버전 비교 로직
public class VersionComparator {
    
    public int compare(String v1, String v2) {
        // "1" vs "2"
        // "1.0" vs "2.0"
        // "2" vs "2.1"
        // "2.1" vs "3"
        
        // 숫자 단위로 비교 (텍스트 아님)
        String[] parts1 = v1.split("\\.");
        String[] parts2 = v2.split("\\.");
        
        for (int i = 0; i < Math.max(parts1.length, parts2.length); i++) {
            int p1 = i < parts1.length ? Integer.parseInt(parts1[i]) : 0;
            int p2 = i < parts2.length ? Integer.parseInt(parts2[i]) : 0;
            
            if (p1 != p2) {
                return Integer.compare(p1, p2);
            }
        }
        return 0;
    }
}

// 예시:
// "1" < "2" < "3" < "10" < "11"
// (텍스트: "10" < "2"! 하지만 Flyway는 숫자 비교)
```

### 3. Spring Context와의 분리

```java
// Spring Boot 시작 시 Flyway 실행 흐름
public class FlywayAutoConfiguration {
    
    @Bean
    public Flyway flyway(DataSource dataSource) {
        // 1단계: Spring ApplicationContext 생성 중
        // → 아직 @Bean들이 초기화되지 않음
        
        Flyway flyway = Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration")
            .load();
        
        // 2단계: 마이그레이션 실행 (이 시점)
        flyway.migrate();
        // → Java 마이그레이션 클래스 로드
        // → new V2__ProcessJsonData() 생성
        // → @Autowired 필드들? 아직 주입 안 됨!
        
        // 3단계: Flyway 완료 후
        // → Spring이 @Bean 초기화 계속
        
        return flyway;
    }
}

// ❌ 결과적으로:
@Component
public class V2__ProcessJsonData extends BaseJavaMigration {
    @Autowired
    private UserRepository repo;  // null! (아직 주입 안 됨)
}

// ✅ 해결책:
// DataSource, Connection만 사용 가능
public class V2__ProcessJsonData extends BaseJavaMigration {
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        // JDBC로 직접 DB 접근
    }
}
```

### 4. 트랜잭션 처리 메커니즘

```java
// isTransactional() = true (기본값)
public void migrateWithTransaction() {
    try {
        conn.setAutoCommit(false);  // Flyway이 관리
        conn.beginTransaction();     // 자동 시작
        
        migration.migrate(context);  // 마이그레이션 실행
        
        conn.commit();               // 자동 커밋
    } catch (Exception e) {
        conn.rollback();             // 자동 롤백
        throw e;
    } finally {
        conn.setAutoCommit(true);    // 원상 복구
    }
}

// isTransactional() = false
public void migrateWithoutTransaction() {
    // 트랜잭션 관리 안 함
    // 각 SQL 문이 개별 트랜잭션
    // (or 명시적으로 BEGIN/COMMIT 필요)
    
    migration.migrate(context);  // 마이그레이션 실행
    // 성공/실패 여부 상관없이 롤백 안 함
}
```

---

## 💻 실전 실험

### 실험 1: BaseJavaMigration 구현

```java
// 파일: src/main/java/db/migration/V2__ProcessJsonData.java

package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import java.sql.*;

public class V2__ProcessJsonData extends BaseJavaMigration {
    
    @Override
    public String getVersion() {
        return "2";
    }
    
    @Override
    public String getDescription() {
        return "Process JSON data in user_profiles";
    }
    
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        
        // JSON 데이터 파싱 및 컬럼 분리
        try (Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery(
                 "SELECT id, json_data FROM user_profiles WHERE json_data IS NOT NULL")) {
            
            while (rs.next()) {
                int id = rs.getInt("id");
                String jsonData = rs.getString("json_data");
                
                // 간단한 JSON 파싱 (실무에서는 Jackson/Gson 사용)
                String firstName = extractField(jsonData, "firstName");
                String lastName = extractField(jsonData, "lastName");
                String email = extractField(jsonData, "email");
                
                // 데이터 업데이트
                updateUserProfile(conn, id, firstName, lastName, email);
            }
        }
        
        System.out.println("Successfully processed JSON data");
    }
    
    private String extractField(String json, String field) {
        // 간단한 JSON 파싱
        String search = "\"" + field + "\":\"";
        int start = json.indexOf(search);
        if (start < 0) return null;
        start += search.length();
        int end = json.indexOf("\"", start);
        return json.substring(start, end);
    }
    
    private void updateUserProfile(Connection conn, int id, 
                                   String firstName, String lastName, String email) 
            throws SQLException {
        String sql = "UPDATE user_profiles SET first_name = ?, last_name = ?, email = ? WHERE id = ?";
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, firstName);
            pstmt.setString(2, lastName);
            pstmt.setString(3, email);
            pstmt.setInt(4, id);
            pstmt.executeUpdate();
        }
    }
    
    @Override
    public boolean isTransactional() {
        return true;  // 트랜잭션 자동 관리
    }
}
```

### 실험 2: Spring Boot에서 실행

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>9.22.0</version>
</dependency>

<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-mysql</artifactId>
    <version>9.22.0</version>
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/myapp
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  
  flyway:
    locations: classpath:db/migration
    validate-on-migrate: true
```

```bash
# 마이그레이션 파일 준비
mkdir -p src/main/resources/db/migration
mkdir -p src/main/java/db/migration

# V1__initial.sql
cat > src/main/resources/db/migration/V1__initial.sql << 'EOF'
CREATE TABLE user_profiles (
  id INT PRIMARY KEY AUTO_INCREMENT,
  json_data TEXT,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  email VARCHAR(100)
);

INSERT INTO user_profiles (json_data) VALUES 
('{"firstName":"Alice","lastName":"Smith","email":"alice@example.com"}'),
('{"firstName":"Bob","lastName":"Jones","email":"bob@example.com"}');
EOF

# mvn spring-boot:run 실행
# 로그 출력:
# o.f.c.i.c.DbMigrate : Migrating database to version 1 - initial
# o.f.c.i.c.DbMigrate : Migrating database to version 2 - Process JSON data in user_profiles
# Successfully processed JSON data
# o.f.c.i.c.DbMigrate : Successfully applied 2 migrations
```

### 실험 3: Java 마이그레이션 검증

```bash
# Java 마이그레이션 메타데이터 확인
mysql -uroot -proot myapp << 'EOF'
SELECT * FROM flyway_schema_history WHERE version = '2';

-- 출력:
-- installed_rank | version | description | type | script | checksum | installed_by | installed_on | execution_time | success
-- 2 | 2 | Process JSON data in user_profiles | JDBC | db/migration/V2__ProcessJsonData.class | 1234567890 | root | 2026-04-13 10:20:00 | 234 | 1

-- SQL 마이그레이션과의 차이:
-- SQL: script = "V1__initial.sql"
-- Java: script = "db/migration/V2__ProcessJsonData.class"
-- Java는 클래스 경로로 기록
EOF

# 데이터 확인
mysql -uroot -proot myapp << 'EOF'
SELECT * FROM user_profiles;

-- 출력:
-- id | json_data | first_name | last_name | email
-- 1 | {"firstName":"Alice"...} | Alice | Smith | alice@example.com
-- 2 | {"firstName":"Bob"...} | Bob | Jones | bob@example.com
EOF
```

### 실험 4: 트랜잭션 동작 확인

```java
// 트랜잭션 롤백 테스트
public class V3__TransactionTest extends BaseJavaMigration {
    
    @Override
    public String getVersion() { return "3"; }
    
    @Override
    public String getDescription() { return "Transaction test"; }
    
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        
        try (Statement stmt = conn.createStatement()) {
            // Step 1: 컬럼 추가 (성공)
            stmt.execute("ALTER TABLE user_profiles ADD COLUMN processed BOOLEAN DEFAULT false");
            System.out.println("Column added");
            
            // Step 2: 데이터 업데이트
            stmt.execute("UPDATE user_profiles SET processed = true");
            System.out.println("Data updated");
            
            // Step 3: 의도적 오류 (롤백 테스트)
            throw new RuntimeException("Simulated error for rollback test");
        }
    }
    
    @Override
    public boolean isTransactional() {
        return true;  // 트랜잭션 활성화
    }
}

// 실행 결과:
// Column added
// Data updated
// RuntimeException 발생!
// → 자동 ROLLBACK
// → processed 컬럼이 없는 상태로 롤백됨
// → 마이그레이션 실패 기록
// 
// Spring Boot 재시작
// → V3 다시 실행 시도
// → 같은 오류 반복
// → 앱 시작 실패
```

### 실험 5: DDL 마이그레이션 (isTransactional = false)

```java
public class V4__CreateView extends BaseJavaMigration {
    
    @Override
    public String getVersion() { return "4"; }
    
    @Override
    public String getDescription() { return "Create user summary view"; }
    
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        
        try (Statement stmt = conn.createStatement()) {
            String viewSql = "CREATE OR REPLACE VIEW user_summary AS " +
                            "SELECT " +
                            "  COUNT(*) as total_users, " +
                            "  COUNT(DISTINCT DATE(installed_on)) as active_days " +
                            "FROM flyway_schema_history";
            
            stmt.execute(viewSql);
            System.out.println("View created");
        }
    }
    
    @Override
    public boolean isTransactional() {
        return false;  // DDL이므로 트랜잭션 비활성화
    }
}
```

---

## 📊 성능/비용 비교

```
┌────────────────────────────────────────────────────────────────┐
│ SQL vs Java 마이그레이션 성능 비교                              │
├──────────────────┬──────────┬─────────────────────────────────┤
│ 작업              │ SQL      │ Java                            │
├──────────────────┼──────────┼─────────────────────────────────┤
│ 테이블 생성       │ 5ms      │ 30ms (JVM 오버헤드)             │
│ 데이터 업데이트   │ 100ms    │ 150ms (코드 실행 오버헤드)      │
│ JSON 파싱         │ 불가능   │ 200ms (로직 처리)               │
│ 외부 API 호출     │ 불가능   │ 1000ms+ (네트워크)              │
└──────────────────┴──────────┴─────────────────────────────────┘

복잡도 비교:
┌────────────────────────────────────────────────────────────────┐
│ 작업                    │ SQL 복잡도 │ Java 복잡도 │ 권장      │
├─────────────────────────┼────────────┼─────────────┼──────────┤
│ 테이블 생성             │ 낮음       │ 높음        │ SQL      │
│ 인덱스 생성             │ 낮음       │ 높음        │ SQL      │
│ 간단한 데이터 이동      │ 중간       │ 중간        │ SQL      │
│ JSON 파싱 및 분리       │ 매우 높음  │ 낮음        │ Java     │
│ 조건부 데이터 변환      │ 높음       │ 낮음        │ Java     │
│ 외부 API 호출           │ 불가능     │ 가능        │ Java     │
│ 비즈니스 로직 적용      │ 불가능     │ 가능        │ Java     │
└─────────────────────────┴────────────┴─────────────┴──────────┘

빌드 시간 영향:
┌────────────────────────────────────────────────────────────────┐
│ Java 마이그레이션 파일 개수 │ 컴파일 시간 │ 실행 오버헤드  │
├──────────────────────────────┼─────────────┼──────────────┤
│ 0개 (SQL만)                  │ 100ms       │ 0ms          │
│ 1~5개                        │ 500ms       │ 50ms         │
│ 10개 이상                    │ 1000ms+     │ 100ms+       │
└──────────────────────────────┴─────────────┴──────────────┘
```

---

## ⚖️ 트레이드오프

```
┌────────────────────────────────────────────────────────────────┐
│ SQL 마이그레이션                                                │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 빠른 실행 속도                                 │
│              │ - DB 네이티브 최적화                             │
│              │ - 간단한 문법                                    │
│              │ - 데이터베이스 독립 실행                         │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 복잡한 로직 표현 불가                         │
│              │ - JSON 파싱 어려움                               │
│              │ - 외부 시스템 연동 불가                         │
│              │ - DB별 문법 차이 (MySQL vs PostgreSQL)          │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Java 마이그레이션                                               │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 복잡한 로직 표현 가능                          │
│              │ - JSON/XML 파싱 쉬움                             │
│              │ - 외부 API 호출 가능                             │
│              │ - Java 생태계 라이브러리 활용                    │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 느린 실행 속도 (JVM 오버헤드)                  │
│              │ - Spring Bean 주입 불가능                       │
│              │ - 단위 테스트 어려움                             │
│              │ - 수정 불가능 (한 번 적용되면)                  │
│              │ - 복잡한 문법                                    │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ 선택 기준                                                       │
├────────────────────────────────────────────────────────────────┤
│ SQL 사용:                                                      │
│ - 테이블, 인덱스, 뷰, 함수 생성/삭제                            │
│ - 간단한 데이터 INSERT/UPDATE                                   │
│ - 권한 부여                                                      │
│                                                                │
│ Java 사용:                                                     │
│ - JSON/XML 데이터 파싱 및 분리                                  │
│ - 복잡한 조건부 변환                                            │
│ - 외부 API 호출로 데이터 보강                                    │
│ - 레거시 데이터 정규화                                          │
└────────────────────────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. BaseJavaMigration은 3가지 필수 원칙                         │
│    - getVersion(): 버전 반환 (null이면 자동 추출)               │
│    - getDescription(): 설명 반환                                │
│    - migrate(Context): 마이그레이션 로직 (필수)                 │
│                                                                 │
│ 2. Java 마이그레이션은 Spring Context 외부에서 실행             │
│    - @Autowired로 Bean 주입 불가능                             │
│    - DataSource와 Connection만 사용 가능                       │
│    - JDBC로 직접 DB 접근                                       │
│                                                                 │
│ 3. 버전 및 실행 순서는 SQL과 통합                               │
│    - V1__sql.sql, V2__java.java는 버전 숫자로 정렬             │
│    - Repeatable도 동일하게 처리                                │
│    - 모든 V 이후에 R이 실행                                     │
│                                                                 │
│ 4. 트랜잭션 처리: isTransactional() 메서드                      │
│    - true (기본값): 자동 트랜잭션 관리                          │
│    - false: 자동 트랜잭션 미사용 (DDL용)                       │
│    - 성공 시 COMMIT, 실패 시 ROLLBACK                          │
│                                                                 │
│ 5. SQL Injection 방지: PreparedStatement 필수                 │
│    - 절대: String 연결로 쿼리 작성                              │
│    - 필수: ? 플레이스홀더와 setString/setInt                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🤔 생각해볼 문제

**Q1.** Java 마이그레이션에서 Spring Bean(@Autowired로 주입)을 사용할 수 없는 이유는 정확히 무엇이고, 만약 꼭 필요하다면 어떤 우회 방법이 있을까?

<details>
<summary>해설 보기</summary>

**원인: Flyway 마이그레이션이 Spring ApplicationContext 초기화 전에 실행**

```
Spring Boot 시작 순서:
1. Spring ApplicationContext 생성 시작
2. 기본 Bean들 로드 (datasource 등)
3. FlywayAutoConfiguration 실행
   3-1. Flyway 빈 생성
   3-2. flyway.migrate() 호출 ← 여기서 Java 마이그레이션 실행
       - V2__MyJavaMigration.class 로드
       - new V2__MyJavaMigration() 인스턴스 생성
       - migrate() 메서드 호출
   3-3. Flyway 빈 반환
4. 나머지 Bean 초기화 (@Autowired 주입 발생)
5. Spring Boot 시작 완료

결과:
- 3-2 시점에 UserRepository @Autowired 필드?
- 아직 초기화 안 됨! (4번에서 초기화)
- NullPointerException 발생
```

**우회 방법:**

```java
// ❌ 방법 1: 불가능 (절대)
@Component
public class V2__MyMigration extends BaseJavaMigration {
    @Autowired UserRepository repo;  // null!
}

// ✅ 방법 2: ApplicationContext 주입 받기
// (가능하지만 복잡함)
@Component
public class V2__MyMigration extends BaseJavaMigration {
    
    private static ApplicationContext context;
    
    public V2__MyMigration(ApplicationContext ctx) {
        V2__MyMigration.context = ctx;
    }
    
    @Override
    public void migrate(Context flywayContext) throws Exception {
        // Spring Bean 획득 (복잡!)
        UserRepository repo = context.getBean(UserRepository.class);
        // 이제 사용 가능
    }
}

// 문제: Flyway는 기본 생성자로 인스턴스 생성
// 따라서 위 방법도 실패!

// ✅ 방법 3: Connection 직접 사용 (권장)
public class V2__MyMigration extends BaseJavaMigration {
    
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        
        // JDBC로 직접 DB 접근
        try (PreparedStatement pstmt = 
             conn.prepareStatement("SELECT * FROM users WHERE id = ?")) {
            pstmt.setInt(1, 1);
            try (ResultSet rs = pstmt.executeQuery()) {
                // 데이터 처리
            }
        }
    }
}

// ✅ 방법 4: 마이그레이션 후 별도 Bean으로 처리
// 1. Java 마이그레이션: JDBC로 간단한 DDL/DML만 처리
// 2. Spring Bean: 마이그레이션 완료 후 복잡한 로직 처리

@Component
public class DataInitializer {
    
    @Autowired private UserRepository repo;
    
    @EventListener(ApplicationReadyEvent.class)
    public void afterStartup() {
        // Spring Boot 완전 시작 후 실행
        // 이때 @Autowired가 정상 주입됨
        repo.findAll();  // 가능!
    }
}
```

**결론:**
- Flyway Java 마이그레이션: Spring 무관, JDBC 직접 사용
- 복잡한 비즈니스 로직: ApplicationReadyEvent 이용
- 둘을 혼재하지 말 것

</details>

---

**Q2.** V1__create.sql이 정상 작동하는데, V2__ProcessData.java가 실패하면, 전체 마이그레이션의 상태는 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**답: V1은 완료, V2는 실패 (부분 적용)**

```
초기 상황:
flyway_schema_history: (비어있음)
DB: (비어있음)

Step 1: V1__create.sql 실행
┌─────────────────────────────┐
│ CREATE TABLE users (         │
│   id INT PRIMARY KEY,        │
│   name VARCHAR(100)          │
│ );                           │
└─────────────────────────────┘
결과:
- users 테이블 생성됨
- flyway_schema_history에 V1 레코드 삽입
- success = true

Step 2: V2__ProcessData.java 실행
┌─────────────────────────────┐
│ @Override                    │
│ public void migrate(...) {   │
│   Connection conn = ...;     │
│   stmt.execute(...);         │
│   throw new Exception();  ← 여기서 실패!
│ }                            │
└─────────────────────────────┘

문제: 이미 실행된 SQL이 있을 수 있음
- isTransactional() = true (기본값)
- 자동 ROLLBACK 발생
- 하지만 V1은 이미 committed!

결과:
- users 테이블: 존재 (V1에서 생성)
- flyway_schema_history:
  - V1: installed_rank=1, success=true
  - V2: installed_rank=2, success=false (!) ← 기록됨

Step 3: Spring Boot 재시작
flyway.migrate() 다시 실행:
- V1: 이미 적용됨 (스킵)
- V2: 이전에 실패함
  - success=false인 레코드 있음
  - 다시 실행? 아니다!
  - Flyway는 실패한 마이그레이션을 재시도하지 않음!
  - → 더 이상 마이그레이션 진행 불가!
  - → 앱 시작 실패!
```

**복구 방법:**

```sql
-- 방법 1: flyway_schema_history에서 실패 레코드 삭제
DELETE FROM flyway_schema_history WHERE version = '2' AND success = false;

-- 이제 flyway migrate 실행하면:
-- - V2 레코드가 없으므로 다시 실행 가능
-- - V2를 수정한 후 재실행

-- 방법 2: 실패 원인 파악 및 수정
-- V2__ProcessData.java의 오류 수정
// throw new Exception(); → 제거 또는 로직 수정

-- 방법 3: 수정된 V2 재배포
git commit -am "Fix V2__ProcessData"
git push

-- 프로덕션 환경:
DELETE FROM flyway_schema_history WHERE version = '2';
-- Spring Boot 재시작
-- flyway migrate 실행
```

**예방:**

```java
// isTransactional() = true일 때 안전성
public class V2__ProcessData extends BaseJavaMigration {
    
    @Override
    public void migrate(Context context) throws Exception {
        // 트랜잭션 자동 관리
        // - 시작: BEGIN
        // - 끝: COMMIT (성공) / ROLLBACK (실패)
        
        Connection conn = context.getConnection();
        
        try {
            // Step 1
            conn.createStatement().execute("UPDATE users SET ...");
            
            // Step 2
            // 여기서 실패하면?
            conn.createStatement().execute("UPDATE products SET ...");  // 잘못된 쿼리
            
            // Step 1과 Step 2 모두 ROLLBACK
            // flyway_schema_history에는 success=false로 기록
        } catch (SQLException e) {
            // Flyway가 자동 ROLLBACK
            throw e;
        }
    }
    
    @Override
    public boolean isTransactional() {
        return true;  // 권장!
    }
}
```

</details>

---

**Q3.** Java 마이그레이션에서 PreparedStatement를 쓰지 않고 String 연결로 쿼리를 작성하면 어떤 보안 위험이 있고, 실제로 데이터를 어떻게 손상시킬 수 있는가?

<details>
<summary>해설 보기</summary>

**답: SQL Injection 공격으로 데이터 삭제, 변조, 유출 가능**

```java
// ❌ 위험한 코드
public class V2__LoadUserData extends BaseJavaMigration {
    
    @Override
    public void migrate(Context context) throws Exception {
        // 외부 데이터 (예: CSV 파일 또는 API 응답)
        String userId = "1'; DROP TABLE users; --";  // 악의적 입력
        String name = "Robert' OR '1'='1";
        
        Connection conn = context.getConnection();
        
        // String 연결로 쿼리 작성
        String sql = "INSERT INTO users (id, name) VALUES ('" + userId + "', '" + name + "')";
        // 결과: INSERT INTO users (id, name) VALUES ('1'; DROP TABLE users; --, 'Robert' OR '1'='1')
        
        conn.createStatement().execute(sql);
        
        // 실행되는 SQL:
        // 1. INSERT INTO users (id, name) VALUES ('1', '')
        // 2. DROP TABLE users (users 테이블 삭제!)
        // 3. -- (나머지 주석 처리)
        
        // 결과:
        // - users 테이블이 완전 삭제됨
        // - 모든 데이터 손실
        // - 서비스 다운
    }
}

// ✅ 안전한 코드
public class V2__LoadUserData extends BaseJavaMigration {
    
    @Override
    public void migrate(Context context) throws Exception {
        String userId = "1'; DROP TABLE users; --";  // 같은 입력
        String name = "Robert' OR '1'='1";
        
        Connection conn = context.getConnection();
        
        // PreparedStatement 사용
        String sql = "INSERT INTO users (id, name) VALUES (?, ?)";
        try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
            pstmt.setString(1, userId);  // SQL이 아니라 데이터로 취급
            pstmt.setString(2, name);
            pstmt.executeUpdate();
        }
        
        // 실행되는 쿼리:
        // INSERT INTO users (id, name) VALUES (?, ?)
        // Parameter 1: "1'; DROP TABLE users; --" (문자열 데이터)
        // Parameter 2: "Robert' OR '1'='1" (문자열 데이터)
        
        // 결과:
        // - users 테이블에 정상 INSERT
        // - 테이블 삭제 안 됨
        // - 안전함!
    }
}
```

**SQL Injection의 다양한 공격:**

```
1. 데이터 삭제
   Input: "1'; DELETE FROM users; --"
   위험: 모든 사용자 데이터 삭제

2. 데이터 변조
   Input: "' OR '1'='1"
   쿼리: SELECT * FROM users WHERE id = '' OR '1'='1'
   위험: 모든 사용자 데이터 노출

3. 인증 우회
   Input: "admin' --"
   쿼리: SELECT * FROM users WHERE username = 'admin' --' AND password = '...'
   위험: 비밀번호 검증 스킵

4. 테이블 구조 파악
   Input: "1' UNION SELECT table_name FROM information_schema.tables --"
   위험: DB 구조 파악 후 다른 테이블 공격

5. 명령 실행 (DB 권한에 따라)
   Input: "'; EXECUTE xp_cmdshell('dir'); --"  (SQL Server)
   위험: 서버 운영 체제 명령 실행
```

**마이그레이션에서의 특별한 위험:**

```
마이그레이션은 프로덕션 DB에서 실행되는 가장 높은 권한의 스크립트
- flyway_user는 보통 DDL, DML, DCL 모두 가능
- 한 번 실행되면 수정 불가능
- 전체 팀에 영향

SQL Injection 시나리오:
1. 개발자가 의도 모르게 CSV 파일을 로드
2. CSV가 악의적인 데이터 포함
3. String 연결로 쿼리 작성
4. DROP TABLE 또는 DELETE FROM 실행
5. 프로덕션 데이터 삭제!
6. 백업에서 복구... (시간, 비용, 신뢰도 손상)
```

**방어:**

```java
// 1. 항상 PreparedStatement 사용
String sql = "INSERT INTO users (id, name) VALUES (?, ?)";
try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
    pstmt.setString(1, userId);
    pstmt.setString(2, name);
    pstmt.executeUpdate();
}

// 2. 입력값 검증
if (!userId.matches("^[0-9]+$")) {
    throw new IllegalArgumentException("Invalid user ID");
}

// 3. 코드 리뷰 (String + 감지)
// IDE에서 경고: "String concatenation in SQL"

// 4. 정적 분석 도구 (SpotBugs, Checkmarx)
// SQL Injection 패턴 자동 감지

// 5. 마이그레이션 권한 제한
// flyway_user는 필요한 권한만 부여
GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.* TO 'flyway_user'@'%';
// (DROP 권한 제거)
```

</details>

---

<div align="center">

**[⬅️ 이전: Flyway 설정 옵션 완전 분석](./03-flyway-config-options.md)** | **[홈으로 🏠](../README.md)** | **[다음: Flyway Callbacks ➡️](./05-flyway-callbacks.md)**

</div>
