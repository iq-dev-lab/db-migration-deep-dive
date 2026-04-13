# Flyway Callbacks — 마이그레이션 전후 자동화

---

## 🎯 핵심 질문

1. Flyway Callback의 7가지 실행 시점(beforeMigrate, afterMigrate, beforeEachMigrate, afterEachMigrate, beforeClean, afterClean, onMigrateError)은 정확히 언제 실행되는가?
2. SQL Callback과 Java Callback의 차이점은 무엇이고, 각각 어떤 경우에 사용해야 하는가?
3. afterMigrate 콜백에서 materialized view를 새로고침하려면 어떻게 작성해야 하고, 어떤 에러가 발생할 수 있는가?
4. onMigrateError 콜백을 사용해서 실패 알림을 외부 시스템(Slack, 이메일)으로 보내려면?
5. 여러 개의 afterMigrate 콜백이 있을 때, 실행 순서는 어떻게 결정되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이그레이션 자체는 스키마 변경일 뿐이지만, 그 이후의 작업이 중요합니다. 새 테이블을 만들었으면 권한을 부여해야 하고, 컬럼을 추가했으면 기존 데이터를 채워야 하며, 뷰를 변경했으면 materialized view를 새로고침해야 합니다. 이런 작업들을 수동으로 하거나 별도 스크립트로 관리하면 실수가 발생하기 쉽습니다.

Flyway Callback을 사용하면 이 모든 작업을 자동화할 수 있습니다. 또한 마이그레이션이 실패했을 때 자동으로 알림을 보내거나, 캐시를 초기화하거나, 모니터링 시스템에 이벤트를 보낼 수 있습니다. 실무에서는 이런 자동화가 운영 비용과 휴먼 에러를 크게 줄입니다.

---

## 😱 흔한 실수 (Before — ...)

```sql
-- ❌ 흔한 실수 1: Callback 없이 권한을 별도로 부여
-- V1__create_table.sql
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100)
);

-- 그 다음 개발자가 수동으로 실행해야 함:
-- $ mysql -uroot -proot mydb < grant_permissions.sql
-- GRANT SELECT, INSERT, UPDATE, DELETE ON mydb.users TO 'app_user'@'%';

-- 문제:
-- 1. 수동 작업 (자동화 안 됨)
-- 2. 마이그레이션 적용 후 권한 설정 전 권한 없이 앱 시작 가능
-- 3. 스테이징에서는 권한 적용했지만 프로덕션에서는 빼먹을 수 있음
```

```sql
-- ❌ 흔한 실수 2: afterMigrate 콜백 없이 materialized view 관리
-- V1__initial.sql
CREATE TABLE sales (
  id INT PRIMARY KEY,
  amount DECIMAL(10,2),
  created_at TIMESTAMP
);

CREATE MATERIALIZED VIEW mv_sales_summary AS
SELECT
  DATE(created_at) as sale_date,
  SUM(amount) as total_amount
FROM sales;

-- 문제:
-- 1. 마이그레이션 후 새 데이터가 추가되면 mv_sales_summary는 stale
-- 2. REFRESH MATERIALIZED VIEW를 수동으로 실행해야 함
-- 3. 개발자가 깜빡하면 레포트가 잘못됨
```

```java
// ❌ 흔한 실수 3: onMigrateError 콜백 없이 실패를 알 수 없음
public class Application {
    public static void main(String[] args) {
        // Spring Boot 시작
        SpringApplication.run(Application.class, args);
        
        // 마이그레이션 중 오류 발생
        // → 앱 시작 실패
        // → 운영팀이 로그를 봐야 알 수 있음
        // → 얼마나 심각한지, 뭘 해야 하는지 모름
        // → 수동으로 조사해야 함
    }
}

// ✅ 올바른 방법: onMigrateError 콜백으로 자동 알림
public class MigrationFailureCallback implements FlywayCallback {
    
    @Override
    public void onMigrateError(...) {
        // Slack에 메시지 전송
        // 이메일 발송
        // 모니터링 시스템에 alert 생성
        // → 운영팀이 즉시 인지
    }
}
```

```java
// ❌ 흔한 실수 4: 여러 Callback의 순서를 모르고 작성
// beforeMigrate.sql (파일 1)
-- 로그 기록

// beforeMigrate_2.sql (파일 2)
-- 캐시 초기화

// 실행 순서가 1 → 2일까, 2 → 1일까?
// 파일명에 따라 알파벳 순서로 정렬!
// "beforeMigrate.sql" < "beforeMigrate_2.sql"
// → 1 → 2 순서로 실행
// → 하지만 캐시를 초기화한 후 로그를 기록해야 하는데 반대로 실행될 수 있음
```

```java
// ❌ 흔한 실수 5: Callback에서 Spring Bean 주입 시도
@Component
public class MigrationCallback implements FlywayCallback {
    
    @Autowired  // ← 주입 안 됨!
    private SlackService slackService;
    
    @Override
    public void afterMigrate(Context context) {
        slackService.send("Migration completed");  // NullPointerException!
    }
}
```

---

## ✨ 올바른 접근 (After — ...)

```sql
-- ✅ 올바른 접근 1: SQL Callback으로 권한 자동 부여
-- V1__create_table.sql
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- afterMigrate.sql (자동 실행)
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO 'app_user'@'%';
GRANT SELECT ON users TO 'read_only'@'%';

-- 동작:
-- 1. V1__create_table.sql 실행 (테이블 생성)
-- 2. afterMigrate.sql 자동 실행 (권한 부여)
-- 3. 모든 다른 마이그레이션 완료 후
-- 4. afterMigrate__.sql (한 번 더) 자동 실행
```

```sql
-- ✅ 올바른 접근 2: Repeatable으로 권한 관리 (권장)
-- R__grant_permissions.sql (Repeatable)
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO 'app_user'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON products TO 'app_user'@'%';
GRANT SELECT ON users TO 'read_only'@'%';
GRANT SELECT ON products TO 'read_only'@'%';

-- 동작:
-- - 매 마이그레이션마다 실행
-- - 파일 수정 시 재실행
-- - 권한이 중복되어도 MySQL은 무시
-- - 새 테이블을 만든 후 권한 누락 방지
```

```sql
-- ✅ 올바른 접근 3: afterMigrate로 view 새로고침
-- V1__initial.sql
CREATE TABLE orders (
  id INT PRIMARY KEY,
  user_id INT,
  amount DECIMAL(10,2),
  created_at TIMESTAMP
);

-- V2__add_product_table.sql
CREATE TABLE products (
  id INT PRIMARY KEY,
  name VARCHAR(100)
);

-- R__refresh_views.sql (Repeatable)
CREATE OR REPLACE VIEW order_summary AS
SELECT
  o.id,
  o.amount,
  p.name
FROM orders o
LEFT JOIN products p ON o.product_id = p.id;

-- afterMigrate.sql
-- PostgreSQL의 materialized view를 새로고침
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_sales_summary;

-- MySQL/MariaDB는 REFRESH 지원 안 함
-- → 대신 DROP + CREATE로 구현
DROP VIEW IF EXISTS sales_summary;
CREATE VIEW sales_summary AS SELECT ...;
```

```java
// ✅ 올바른 접근 4: Java Callback으로 외부 시스템 연동
@Component
public class MigrationNotificationCallback implements FlywayCallback {
    
    private final SlackClient slackClient;
    
    public MigrationNotificationCallback(SlackClient slackClient) {
        this.slackClient = slackClient;  // 생성자 주입
    }
    
    @Override
    public void onMigrateError(Context context, int state) {
        // 마이그레이션 실패 시 Slack 알림
        String message = String.format(
            "Database migration failed on %s (version %d)",
            context.getConnection().getMetaData().getURL(),
            state
        );
        
        slackClient.sendMessage("#ops", message);
    }
    
    @Override
    public void afterMigrate(Context context) {
        // 마이그레이션 성공 시 알림
        slackClient.sendMessage("#ops", "Database migration completed successfully");
    }
}
```

```java
// ✅ 올바른 접근 5: Callback 실행 순서 제어
// 파일명으로 순서 지정:
// 1. beforeMigrate__01_log.sql (01 < 02 < 03)
// 2. beforeMigrate__02_clear_cache.sql
// 3. beforeMigrate__03_set_settings.sql

// Java Callback:
@Component
public class OrderedCallbacks implements FlywayCallback {
    
    @Override
    public void beforeMigrate(Context context) {
        // Java Callback은 모든 SQL Callback 후 실행
        System.out.println("Java callback after all SQL callbacks");
    }
}

// 최종 순서:
// 1. beforeMigrate__01_log.sql
// 2. beforeMigrate__02_clear_cache.sql
// 3. beforeMigrate__03_set_settings.sql
// 4. Java beforeMigrate 콜백
// 5. 실제 V1, V2, V3 마이그레이션 실행
```

---

## 🔬 내부 동작 원리

### 1. 7가지 Callback 실행 시점

```
Spring Boot 시작 시 마이그레이션 흐름:

┌─────────────────────────────────────────────────────────┐
│ 1. beforeMigrate (모든 마이그레이션 전)                  │
│    - SQL: beforeMigrate.sql, beforeMigrate__*.sql        │
│    - Java: FlywayCallback.beforeMigrate()                │
│    - 용도: 캐시 초기화, 권한 확인, 로그 기록            │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 2. beforeEachMigrate (각 마이그레이션 전, 반복)           │
│    - SQL: beforeEachMigrate.sql                          │
│    - Java: FlywayCallback.beforeEachMigrate()            │
│    - V1 전 → beforeEachMigrate → V1 실행                │
│    - V2 전 → beforeEachMigrate → V2 실행                │
│    - V3 전 → beforeEachMigrate → V3 실행                │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 3. V1__initial.sql 실행                                 │
│ 4. V2__add_column.sql 실행                              │
│ 5. V3__add_index.sql 실행                               │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 6. afterEachMigrate (각 마이그레이션 후, 반복)           │
│    - SQL: afterEachMigrate.sql                           │
│    - Java: FlywayCallback.afterEachMigrate()             │
│    - V1 후 → afterEachMigrate 실행                      │
│    - V2 후 → afterEachMigrate 실행                      │
│    - V3 후 → afterEachMigrate 실행                      │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 7. afterMigrate (모든 마이그레이션 후)                   │
│    - SQL: afterMigrate.sql, afterMigrate__*.sql          │
│    - Java: FlywayCallback.afterMigrate()                 │
│    - 용도: 권한 부여, 뷰 새로고침, 인덱스 분석          │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 8. onMigrateError (마이그레이션 실패 시)                │
│    - V1, V2, V3 중 어느 것이든 실패하면 호출             │
│    - SQL: (지원 안 함)                                   │
│    - Java: FlywayCallback.onMigrateError()               │
│    - 자동 ROLLBACK 후 호출됨                             │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ clean 명령어 사용 시 (flyway clean):                     │
│ beforeClean → DROP ALL → afterClean                      │
│ - 프로덕션에서는 clean-disabled: true로 차단!             │
└─────────────────────────────────────────────────────────┘
```

### 2. SQL Callback vs Java Callback

```java
// SQL Callback 메커니즘
public class SqlCallbackScanner {
    public List<Callback> scanCallbacks() {
        // db/migration/ 디렉토리 스캔
        // 파일명 패턴:
        // - beforeMigrate.sql
        // - beforeMigrate__*.sql (여러 개)
        // - afterMigrate.sql
        // - afterMigrate__grant_permissions.sql
        // - beforeEachMigrate.sql
        // - afterEachMigrate.sql
        // - beforeClean.sql
        // - afterClean.sql
        
        // 파일명을 "이름__순서" 형식으로 정렬
        // beforeMigrate.sql (순서 없음)
        // beforeMigrate__01_setup.sql (01)
        // beforeMigrate__02_clear_cache.sql (02)
        // → 1 → 2 순서로 실행
    }
}

// Java Callback 메커니즘
public interface FlywayCallback {
    void beforeMigrate(Context context);
    void afterMigrate(Context context);
    void beforeEachMigrate(Context context);
    void afterEachMigrate(Context context);
    void beforeClean(Context context);
    void afterClean(Context context);
    void onMigrateError(Context context, int state);
}

// Spring은 classpath에서 FlywayCallback 구현체 자동 발견
@Component
public class MyCallback implements FlywayCallback {
    @Override
    public void afterMigrate(Context context) {
        System.out.println("After migration!");
    }
    // 나머지는 기본 구현 (do nothing)
}
```

### 3. 실행 순서 및 동작

```
┌─────────────────────────────────────────────────────────┐
│ Callback 실행 순서                                       │
├─────────────────────────────────────────────────────────┤
│ 1. SQL Callback: beforeMigrate.sql (파일명 알파벳순)    │
│    - beforeMigrate.sql                                  │
│    - beforeMigrate__01_*.sql                            │
│    - beforeMigrate__02_*.sql                            │
│                                                         │
│ 2. Java Callback: beforeMigrate() (여러 Bean)           │
│    - @Component A beforeMigrate()                       │
│    - @Component B beforeMigrate()                       │
│    (순서: @Order 또는 등록 순서)                        │
│                                                         │
│ 3. V1, V2, V3 마이그레이션 실행                         │
│                                                         │
│ 4. SQL Callback: afterMigrate.sql                       │
│    - afterMigrate.sql                                   │
│    - afterMigrate__01_*.sql                             │
│    - afterMigrate__02_*.sql                             │
│                                                         │
│ 5. Java Callback: afterMigrate()                        │
│    - @Component A afterMigrate()                        │
│    - @Component B afterMigrate()                        │
└─────────────────────────────────────────────────────────┘
```

---

## 💻 실전 실험

### 실험 1: SQL Callback으로 권한 자동 부여

```bash
# 디렉토리 구조
mkdir -p db/migration

# V1__create_users.sql
cat > db/migration/V1__create_users.sql << 'EOF'
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(100) NOT NULL
);
EOF

# R__grant_permissions.sql (Repeatable - 항상 권한 동기화)
cat > db/migration/R__grant_permissions.sql << 'EOF'
-- 모든 권한 제거 후 재부여 (멱등성)
-- MySQL은 REVOKE 후 GRANT 필요
-- 하지만 권한 없으면 REVOKE가 실패할 수 있으므로 조건부
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO 'app_user'@'%';
GRANT SELECT, INSERT, UPDATE, DELETE ON products TO 'app_user'@'%';
GRANT SELECT ON users TO 'read_only'@'%';
GRANT SELECT ON products TO 'read_only'@'%';
EOF

# afterMigrate.sql (모든 마이그레이션 후 실행)
cat > db/migration/afterMigrate.sql << 'EOF'
-- 인덱스 분석 (InnoDB에서 최적화)
ANALYZE TABLE users;
ANALYZE TABLE products;

-- 통계 업데이트
-- (MySQL 8.0+)
-- 자동 실행되지만 명시적으로 호출 가능
EOF

# Spring Boot 실행
mvn spring-boot:run

# 결과 확인
mysql -uroot -proot myapp << 'EOF'
SELECT * FROM flyway_schema_history;

-- 실행 순서:
-- 1. V1__create_users.sql (V 마이그레이션)
-- 2. R__grant_permissions.sql (R 마이그레이션)
-- 3. afterMigrate.sql (Callback)

-- 권한 확인
SHOW GRANTS FOR 'app_user'@'%';
SHOW GRANTS FOR 'read_only'@'%';
EOF
```

### 실험 2: Java Callback으로 Slack 알림

```java
// pom.xml에 HTTP 클라이언트 추가
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>4.10.0</version>
</dependency>

// SlackNotificationCallback.java
package db.migration;

import org.flywaydb.core.api.callback.Callback;
import org.flywaydb.core.api.callback.Context;
import org.flywaydb.core.api.callback.Event;
import org.springframework.stereotype.Component;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.MediaType;

@Component
public class SlackNotificationCallback implements Callback {
    
    private static final String SLACK_WEBHOOK = "https://hooks.slack.com/services/YOUR/WEBHOOK/URL";
    private static final OkHttpClient client = new OkHttpClient();
    
    @Override
    public void handle(Event event, Context context) {
        String message = null;
        
        switch (event) {
            case AFTER_MIGRATE:
                message = buildSuccessMessage(context);
                break;
            case MIGRATE_ERROR:
                message = buildErrorMessage(context);
                break;
        }
        
        if (message != null) {
            sendToSlack(message);
        }
    }
    
    private String buildSuccessMessage(Context context) {
        return String.format(
            "✅ Database migration successful\n" +
            "Database: %s\n" +
            "Timestamp: %s",
            context.getConnection().getMetaData().getURL(),
            System.currentTimeMillis()
        );
    }
    
    private String buildErrorMessage(Context context) {
        return String.format(
            "❌ Database migration FAILED\n" +
            "Database: %s\n" +
            "Error: Check logs for details",
            context.getConnection().getMetaData().getURL()
        );
    }
    
    private void sendToSlack(String message) {
        String json = String.format(
            "{\"text\": \"%s\"}",
            message.replace("\"", "\\\"").replace("\n", "\\n")
        );
        
        RequestBody body = RequestBody.create(json, MediaType.get("application/json"));
        Request request = new Request.Builder()
            .url(SLACK_WEBHOOK)
            .post(body)
            .build();
        
        try {
            client.newCall(request).execute();
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
    
    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.AFTER_MIGRATE || event == Event.MIGRATE_ERROR;
    }
    
    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return false;  // HTTP 호출은 트랜잭션 밖에서
    }
}
```

### 실험 3: beforeEachMigrate로 로깅

```sql
-- beforeEachMigrate.sql
-- 각 마이그레이션 전 로그 기록
INSERT INTO migration_log (event_type, message, timestamp)
VALUES ('BEFORE_EACH', 'Starting migration', NOW());

-- afterEachMigrate.sql
-- 각 마이그레이션 후 로그 기록
INSERT INTO migration_log (event_type, message, timestamp)
VALUES ('AFTER_EACH', 'Completed migration', NOW());

-- 마이그레이션 확인
mysql -uroot -proot myapp << 'EOF'
SELECT * FROM migration_log ORDER BY timestamp;

-- 출력:
-- event_type | message | timestamp
-- BEFORE_EACH | Starting migration | 2026-04-13 10:00:00
-- (V1 실행)
-- AFTER_EACH | Completed migration | 2026-04-13 10:00:01
-- BEFORE_EACH | Starting migration | 2026-04-13 10:00:02
-- (V2 실행)
-- AFTER_EACH | Completed migration | 2026-04-13 10:00:03
EOF
```

### 실험 4: onMigrateError로 실패 감지

```java
public class V2__FailingMigration extends BaseJavaMigration {
    
    @Override
    public String getVersion() { return "2"; }
    
    @Override
    public String getDescription() { return "Intentional failure"; }
    
    @Override
    public void migrate(Context context) throws Exception {
        // 의도적 실패
        throw new RuntimeException("Simulated migration error");
    }
}

// ErrorHandlingCallback.java
@Component
public class ErrorHandlingCallback implements FlywayCallback {
    
    @Override
    public void onMigrateError(Context context, int state) {
        // state: 마이그레이션 상태
        System.out.println("Migration failed at state: " + state);
        
        // 알림 전송
        // sendEmailAlert("DBA Team", "Migration failed");
    }
}

// 실행 결과:
// ❌ Migration failed at state: 2
// → errorHandlingCallback.onMigrateError() 호출됨
```

---

## 📊 성능/비용 비교

```
┌────────────────────────────────────────────────────────────────┐
│ Callback 실행 시간 오버헤드                                    │
├──────────────────┬──────────┬─────────────────────────────────┤
│ Callback 종류    │ 시간     │ 특징                            │
├──────────────────┼──────────┼─────────────────────────────────┤
│ SQL (간단함)     │ 5-10ms   │ GRANT, INDEX 분석 등            │
│ SQL (복잡함)     │ 100-500ms│ REFRESH VIEW, 데이터 처리 등    │
│ Java (HTTP)      │ 1000ms+  │ 외부 시스템 호출 (네트워크)     │
│ Java (로컬)      │ 50-100ms │ 캐시 초기화, 메트릭 기록 등     │
└──────────────────┴──────────┴─────────────────────────────────┘

마이그레이션별 총 시간 (3개 마이그레이션 + Callback):
┌──────────────────────────────────────────────────────────────┐
│ 구성                          │ 예상 시간 │ 주요 시간 소비처 │
├───────────────────────────────┼──────────┼──────────────┤
│ 기본 (Callback 없음)          │ 100ms    │ 마이그레이션 │
│ + 간단한 SQL Callback         │ 120ms    │ GRANT        │
│ + 복잡한 SQL Callback         │ 300ms    │ VIEW 처리    │
│ + Java Callback (HTTP)        │ 1500ms   │ 네트워크     │
│ + 여러 Callback (모두)        │ 2000ms+  │ 네트워크     │
└──────────────────────────────────────────────────────────────┘

권장: 외부 호출은 비동기로!
```

---

## ⚖️ 트레이드오프

```
┌────────────────────────────────────────────────────────────────┐
│ SQL Callback                                                    │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - DB 네이티브로 실행 (빠름)                     │
│              │ - 파일 기반 관리 (버전 관리 쉬움)               │
│              │ - 복잡한 로직도 SQL로 구현 가능                 │
│              │ - Spring 의존성 없음                            │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 외부 시스템 연동 불가능                      │
│              │ - HTTP 호출 불가                               │
│              │ - 에러 처리 제한적                              │
│              │ - 파일명으로 순서 지정해야 함                   │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Java Callback                                                   │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 복잡한 로직 구현 가능                         │
│              │ - HTTP, 이메일 등 외부 시스템 연동              │
│              │ - @Order로 순서 제어 가능                       │
│              │ - Spring 통합 가능                              │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 느린 실행 (특히 네트워크 호출)                │
│              │ - Spring 의존성 필요                            │
│ (상황따라)   │ - onMigrateError만 메서드명 다름                │
│              │ - Bean 주입이 복잡 (생성자 주입만 가능)         │
└──────────────┴─────────────────────────────────────────────────┘

선택 가이드:
- 권한, 인덱스, 뷰: SQL Callback (빠름)
- 외부 알림, 캐시: Java Callback (필요)
- 간단한 로직: SQL (먼저 시도)
- 복잡한 로직: Java (필요시)
```

---

## 📌 핵심 정리

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. 7가지 Callback 시점                                         │
│    - beforeMigrate: 모든 마이그레이션 전                        │
│    - beforeEachMigrate: 각 마이그레이션 전 (반복)               │
│    - afterEachMigrate: 각 마이그레이션 후 (반복)                │
│    - afterMigrate: 모든 마이그레이션 후                         │
│    - onMigrateError: 마이그레이션 실패 시                      │
│    - beforeClean, afterClean: clean 명령어 시                   │
│                                                                 │
│ 2. SQL Callback vs Java Callback                               │
│    - SQL: 파일명 패턴으로 자동 발견, DB 네이티브                │
│    - Java: @Component + FlywayCallback 구현, 외부 연동          │
│                                                                 │
│ 3. 실행 순서                                                    │
│    - SQL Callback 파일명 알파벳순 정렬                          │
│    - Java Callback @Order 또는 등록 순서                        │
│    - beforeMigrate → 마이그레이션 → afterMigrate              │
│                                                                 │
│ 4. 일반적인 사용 사례                                           │
│    - beforeMigrate: 캐시 초기화, 백업                          │
│    - afterMigrate: 권한 부여, 뷰 새로고침                       │
│    - onMigrateError: 알림 (Slack, 이메일)                     │
│                                                                 │
│ 5. 주의사항                                                    │
│    - SQL Callback은 개별 트랜잭션                               │
│    - Java Callback에서 Spring Bean 주입 불가능               │
│    - 외부 호출은 비동기로 (시작 시간 증가 방지)                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🤔 생각해볼 문제

**Q1.** afterMigrate 콜백에서 여러 권한 부여 SQL을 실행할 때, 하나가 실패하면 롤백되는가?

<details>
<summary>해설 보기</summary>

**답: Callback의 트랜잭션 처리는 복잡**

```
SQL Callback의 트랜잭션 처리:
- SQL 콜백은 개별 SQL 문 단위로 실행
- 각 SQL이 자신의 트랜잭션에서 실행
- 한 SQL 실패 → 롤백 (그 SQL만)
- 다른 SQL은 계속 실행

예시:
afterMigrate.sql:
┌─────────────────────────────────────────┐
│ GRANT ... (성공)                        │
│ GRANT ... (성공)                        │
│ GRANT ... (실패, 권한 없음)              │ ← 실패해도 이어서 실행?
│ GRANT ... (성공 또는 스킵)                │
└─────────────────────────────────────────┘

실제 동작:
- Statement 1: GRANT (성공)
- Statement 2: GRANT (성공)
- Statement 3: GRANT (실패, SQLException 발생)
- → Flyway가 예외를 로깅하고 계속 진행할 수도, 중단할 수도
- (Flyway 버전 및 설정에 따라 다름)
```

**안전한 작성 방법:**

```sql
-- ✅ 올바른 방법: 조건부 권한 부여
-- R__grant_permissions.sql (Repeatable)

-- 1. 기존 권한 확인 및 삭제 (조건부)
GRANT SELECT, INSERT, UPDATE, DELETE ON users TO 'app_user'@'%';

-- 2. 여러 권한은 한 문장에
GRANT SELECT, INSERT, UPDATE, DELETE ON products TO 'app_user'@'%';

-- 3. 에러가 발생해도 계속 진행하려면?
-- 이건 마이그레이션이 아니라 별도 스크립트에서 처리
```

**권장:**
```sql
-- ❌ 위험: 권한 부여 실패 시 전체 마이그레이션 실패
GRANT ... ON users TO ...;
GRANT ... ON products TO ...;  -- 실패하면?

-- ✅ 안전: Repeatable로 멱등성 보장
-- R__grant_permissions.sql
-- 파일 수정 시마다 재실행
-- 권한이 중복되어도 MySQL은 무시
-- 새로운 테이블 추가 시 권한 자동 부여
```

</details>

---

**Q2.** beforeMigrate와 beforeEachMigrate가 모두 있으면, 각각 몇 번 실행되는가?

<details>
<summary>해설 보기</summary>

**답: beforeMigrate는 1회, beforeEachMigrate는 N회**

```
시나리오: V1, V2, V3 마이그레이션 있음

실행 흐름:
┌─────────────────────────────────────┐
│ 1. beforeMigrate 실행 (1회)          │
│    - 캐시 초기화                      │
│    - 권한 확인                        │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ 2. beforeEachMigrate 실행 (V1 전)    │
│    - 로그 기록: "V1 시작"             │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ 3. V1 마이그레이션 실행              │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ 4. afterEachMigrate 실행 (V1 후)     │
│    - 로그 기록: "V1 완료"             │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ 5. beforeEachMigrate 실행 (V2 전)    │
│    - 로그 기록: "V2 시작"             │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ 6. V2 마이그레이션 실행              │
└─────────────────────────────────────┘
        ↓ (V3도 동일하게 반복)
        ↓
┌─────────────────────────────────────┐
│ N. afterMigrate 실행 (1회, 마지막)   │
│    - 권한 부여                        │
│    - 통계 업데이트                    │
└─────────────────────────────────────┘

실행 횟수:
- beforeMigrate: 1회
- beforeEachMigrate: 3회 (V1, V2, V3 각각)
- afterEachMigrate: 3회 (V1, V2, V3 각각)
- afterMigrate: 1회

총 Callback 호출: 1 + 3 + 3 + 1 = 8회
실제 마이그레이션: 3회 (V1, V2, V3)
```

**실무 활용:**

```sql
-- beforeMigrate.sql (1회)
-- 마이그레이션 전 한 번만 필요한 작업
BACKUP TABLE users TO BACKUP_TABLE_BEFORE_MIGRATION;

-- beforeEachMigrate.sql (N회)
-- 각 마이그레이션 전 매번 필요한 작업
INSERT INTO migration_log VALUES (NOW(), 'Migration starting');

-- afterEachMigrate.sql (N회)
-- 각 마이그레이션 후 매번 필요한 작업
INSERT INTO migration_log VALUES (NOW(), 'Migration completed');

-- afterMigrate.sql (1회)
-- 마이그레이션 후 한 번만 필요한 작업
GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'app_user'@'%';
ANALYZE TABLE users;
ANALYZE TABLE products;
```

</details>

---

**Q3.** onMigrateError 콜백에서 HTTP로 Slack에 알림을 보냈는데, 모든 마이그레이션이 롤백되는가?

<details>
<summary>해설 보기</summary>

**답: 마이그레이션은 롤백, Slack 메시지는 전송됨**

```
트랜잭션 격리:
┌─────────────────────────────────────┐
│ V1, V2, V3 마이그레이션 실행 (트랜잭션 내) │
│ ...                                  │
│ V2에서 실패! (SQL 오류)               │
│ ← 자동 ROLLBACK (V1, V2 모두)         │
└─────────────────────────────────────┘
        ↓
┌─────────────────────────────────────┐
│ onMigrateError 콜백 실행              │
│ (트랜잭션 외부)                       │
│                                      │
│ slackService.send("Failed!")         │
│ → HTTP POST 요청                     │
│ → Slack에 메시지 도착                 │
│ → 이미 ROLLBACK된 후!                │
└─────────────────────────────────────┘
```

**핵심:**
- 마이그레이션: ROLLBACK (데이터 일관성 유지)
- Callback: 트랜잭션 외부 실행 (HTTP 호출 등)
- 알림은 전송됨 (ROLLBACK 후)

**실무 예시:**

```java
@Component
public class MigrationFailureCallback implements FlywayCallback {
    
    @Override
    public void onMigrateError(Context context, int state) {
        // 이 시점:
        // 1. 마이그레이션 실패
        // 2. 자동 ROLLBACK 완료
        // 3. DB는 이전 상태로 복구됨
        
        String message = String.format(
            "🚨 Database migration FAILED\n" +
            "State: %d\n" +
            "Database will be rolled back\n" +
            "Please investigate and fix the issue",
            state
        );
        
        slackService.send("#ops", message);
        emailService.send("dba@company.com", message);
        metricsService.increment("migration.failures");
    }
}
```

**주의사항:**

```
onMigrateError에서 하면 안 되는 것:
- 추가 DB 쿼리 실행 (이미 트랜잭션 끝남)
- 데이터 수정 (ROLLBACK된 후)
- 기존 데이터 복구 시도 (이미 자동 복구됨)

onMigrateError에서 해야 할 것:
- 외부 알림 (Slack, Email)
- 로그 기록
- 모니터링 메트릭 전송
- 관리자 호출 (Pagerduty 등)
```

</details>

---

<div align="center">

**[⬅️ 이전: Java 기반 마이그레이션](./04-java-migration.md)** | **[홈으로 🏠](../README.md)** | **[다음: 체크섬 불일치 오류 해결 ➡️](./06-checksum-mismatch.md)**

</div>
