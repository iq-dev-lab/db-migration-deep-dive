# Flyway 내부 동작 원리 — flyway_schema_history

---

## 🎯 핵심 질문

1. `flyway_schema_history` 테이블의 10개 컬럼은 각각 어떤 정보를 저장하는가?
2. Spring Boot 애플리케이션 시작 시 Flyway는 정확히 어떤 단계를 거쳐서 마이그레이션을 실행하는가?
3. CRC32 체크섬은 어떻게 파일 변경을 감지하고, 왜 숫자로 표현되는가?
4. 여러 서버 인스턴스가 동시에 시작할 때 어떻게 마이그레이션 중복을 방지하는가?
5. `flyway_schema_history` 테이블이 없을 때 자동으로 생성되는데, 그 DDL은 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Flyway는 단순해 보이지만, 내부에서는 정교한 메커니즘으로 작동합니다. `flyway_schema_history` 테이블은 Flyway의 "기억"으로, 어떤 마이그레이션이 언제 누가 실행했는지 영구적으로 기록합니다. 이 테이블을 이해하지 못하면, 체크섬 오류가 났을 때 `flyway repair`를 무작정 실행했다가 프로덕션 데이터베이스를 손상시킬 수 있습니다.

또한, 마이크로서비스 환경에서는 여러 애플리케이션 인스턴스가 동시에 시작되곤 합니다. 이때 Flyway의 Lock 메커니즘을 이해하지 못하면 동일한 마이그레이션이 중복으로 실행되거나 충돌이 발생할 수 있습니다. Spring Boot의 `FlywayAutoConfiguration`은 이 모든 과정을 자동으로 처리하지만, 문제가 생겼을 때 해결하려면 내부 동작을 알아야 합니다.

---

## 😱 흔한 실수 (Before — ...)

```java
// ❌ 흔한 실수 1: flyway_schema_history 수동 삭제
// DBA가 "깔끔히 정리하자"며 다음을 실행
DELETE FROM flyway_schema_history;

// 이제 모든 마이그레이션이 "미적용"으로 표시됨
// Spring Boot 재시작 시 모든 V 마이그레이션을 다시 실행하려고 시도
// 이미 적용된 테이블/인덱스를 다시 만들려고 해서 오류 발생!
```

```sql
-- ❌ 흔한 실수 2: 프로덕션에서 flyway_schema_history 구조 변경
ALTER TABLE flyway_schema_history DROP COLUMN execution_time;
-- Flyway가 예기치 않은 스키마 변경으로 혼동
```

```bash
# ❌ 흔한 실수 3: 여러 인스턴스 동시 시작
# 서버1, 서버2, 서버3 모두 같은 시간에 시작
java -jar app.jar &
java -jar app.jar &
java -jar app.jar &

# Lock이 제대로 작동하지 않으면 같은 마이그레이션이 여러 번 실행
# "Error: Could not execute statement" 메시지로 충돌 감지
```

```yaml
# ❌ 흔한 실수 4: 개발 환경과 프로덕션 환경의 flyway_schema_history 구조 불일치
# application-dev.yml
spring:
  flyway:
    baseline-on-migrate: true  # 개발은 허용

# application-prod.yml
spring:
  flyway:
    baseline-on-migrate: false  # 프로덕션은 금지
    # 그런데 프로덕션에서 flyway_schema_history가 없으면?
    # → 시작 오류!
```

---

## ✨ 올바른 접근 (After — ...)

```sql
-- ✅ 올바른 접근 1: flyway_schema_history 구조 확인
DESCRIBE flyway_schema_history;

-- 출력 (MySQL 8.0):
-- installed_rank      INT NOT NULL (순서)
-- version             VARCHAR(50) (마이그레이션 버전)
-- description        VARCHAR(255) (설명)
-- type                VARCHAR(20) (V, R, U)
-- script              VARCHAR(1000) (파일명)
-- checksum            INT (CRC32 체크섬)
-- installed_by        VARCHAR(100) (적용자)
-- installed_on        TIMESTAMP (적용 시간)
-- execution_time      INT (실행 시간 밀리초)
-- success             BOOLEAN (성공 여부)
```

```java
// ✅ 올바른 접근 2: Spring Boot에서 Flyway 설정
@Configuration
@EnableAutoConfiguration
public class FlywayConfig {
    
    @Bean
    public Flyway flyway(DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration")
            .baselineOnMigrate(false)      // 프로덕션은 항상 false
            .cleanDisabled(true)            // CLEAN 명령어 차단
            .validateOnMigrate(true)        // 시작 시 검증
            .outOfOrder(false)              // 순서 변경 금지
            .load();
    }
}
```

```bash
# ✅ 올바른 접근 3: 마이그레이션 상태 확인
flyway info

# 출력 예시:
# +----------+---------+---------------------+------+---------------------+---------+
# | Category | Version | Description         | Type | Installed On        | State   |
# +----------+---------+---------------------+------+---------------------+---------+
# | Versioned|       1 | initial schema      | SQL  | 2026-04-10 09:15:23 | Success |
# | Versioned|       2 | add user table      | SQL  | 2026-04-10 09:15:24 | Success |
# | Versioned|       3 | add index on email  | SQL  | 2026-04-10 09:15:25 | Success |
# |Repeatable|     N/A | grant permissions   | SQL  | 2026-04-10 09:15:25 | Success |
# +----------+---------+---------------------+------+---------------------+---------+
```

```java
// ✅ 올바른 접근 4: 동시성 안전한 구성
// application.yml
spring:
  flyway:
    # Lock을 위해 flyway_schema_history_lock 테이블 자동 생성
    lock-retry-count: 50  # 50번 재시도 (기본값)
    # Kubernetes에서 여러 Pod이 동시에 시작해도 안전
    
    # 설정 검증을 위해 시작 시 항상 validate 실행
    validate-on-migrate: true
    clean-disabled: true
```

---

## 🔬 내부 동작 원리

### 1. Spring Boot 시작 → FlywayAutoConfiguration 초기화

```java
// Spring Boot가 하는 일
public class FlywayAutoConfiguration {
    
    @Bean
    public Flyway flyway(Environment env, DataSource dataSource) {
        // 1단계: spring.flyway.* 프로퍼티 읽기
        String locations = env.getProperty("spring.flyway.locations");
        String baselineOnMigrate = env.getProperty("spring.flyway.baseline-on-migrate");
        
        // 2단계: Flyway 인스턴스 생성
        Flyway flyway = Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration")
            .baselineOnMigrate(Boolean.parseBoolean(baselineOnMigrate))
            .load();
            
        // 3단계: 마이그레이션 자동 실행 (만약 필요하면)
        flyway.migrate();  // ← 핵심!
        
        return flyway;
    }
}

// 이때 flyway.migrate()가 호출되는 순간:
// 1) flyway_schema_history 테이블 존재 여부 확인
// 2) 테이블이 없으면 자동 생성
// 3) classpath:db/migration의 모든 파일 스캔
// 4) 각 파일의 CRC32 체크섬 계산
// 5) flyway_schema_history와 비교
// 6) 미적용된 마이그레이션만 실행
```

### 2. flyway_schema_history 테이블 구조 및 각 컬럼의 의미

```sql
-- MySQL에서 자동 생성되는 DDL
CREATE TABLE flyway_schema_history (
    installed_rank INT NOT NULL,           -- 적용 순서 (1부터 시작)
    version VARCHAR(50),                   -- 마이그레이션 버전 (예: "1", "1.1")
    description VARCHAR(255) NOT NULL,    -- 파일명에서 추출한 설명
    type VARCHAR(20) NOT NULL,            -- SQL, JDBC, SPRING_JDBC 등
    script VARCHAR(1000) NOT NULL,        -- 실제 파일명 (예: V1__initial.sql)
    checksum INT,                         -- CRC32 체크섬 (음수 가능)
    installed_by VARCHAR(100) NOT NULL,   -- 적용한 사용자 (DB 계정)
    installed_on TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,  -- 적용 시각
    execution_time INT NOT NULL,          -- 실행 시간 (밀리초)
    success BOOLEAN NOT NULL,             -- 성공 여부 (실패도 기록됨)
    PRIMARY KEY (installed_rank),
    UNIQUE KEY ux_flyway_schema_history_vr (version, type),
    KEY ix_flyway_schema_history_s (success)
);

-- PostgreSQL 자동 생성 DDL (구조는 동일)
CREATE TABLE flyway_schema_history (
    installed_rank integer NOT NULL,
    version varchar(50),
    description varchar(255) NOT NULL,
    type varchar(20) NOT NULL,
    script varchar(1000) NOT NULL,
    checksum integer,
    installed_by varchar(100) NOT NULL,
    installed_on timestamp NOT NULL DEFAULT now(),
    execution_time integer NOT NULL,
    success boolean NOT NULL,
    PRIMARY KEY (installed_rank),
    UNIQUE (version, type)
);
```

### 3. CRC32 체크섬 계산 및 변경 감지 원리

```java
// Flyway는 CRC32로 파일 내용을 숫자로 변환
// 파일 내용이 1비트라도 변경되면 체크섬이 달라짐

import java.util.zip.CRC32;

public class ChecksumDemo {
    public static void main(String[] args) {
        // 원본 파일 내용
        String sql1 = "CREATE TABLE users (\n" +
                      "  id INT PRIMARY KEY AUTO_INCREMENT,\n" +
                      "  name VARCHAR(100)\n" +
                      ");";
        
        // 누군가가 세미콜론을 빼먹음
        String sql2 = "CREATE TABLE users (\n" +
                      "  id INT PRIMARY KEY AUTO_INCREMENT,\n" +
                      "  name VARCHAR(100)\n" +
                      ")";
        
        int checksum1 = calculateCRC32(sql1);
        int checksum2 = calculateCRC32(sql2);
        
        System.out.println("원본: " + checksum1);           // -1234567890 (예시)
        System.out.println("수정본: " + checksum2);         // -1234567891 (다름!)
        System.out.println("일치여부: " + (checksum1 == checksum2)); // false
    }
    
    private static int calculateCRC32(String content) {
        CRC32 crc = new CRC32();
        crc.update(content.getBytes());
        return (int) crc.getValue();  // int로 변환 (음수 가능)
    }
}

// 실행 결과:
// 원본: 1752462123
// 수정본: -1752462120
// 일치여부: false
```

```
흐름도:
[마이그레이션 파일 로드]
         ↓
[파일 내용 읽기]
         ↓
[CRC32 계산] → 1752462123
         ↓
[DB 쿼리] SELECT checksum FROM flyway_schema_history WHERE version = '1'
         ↓
[비교]
  - 저장된 값: 1752462123 (일치 ✅)
  - 저장된 값: -1752462120 (불일치 ❌ → 오류 발생!)
         ↓
[결과]
  - 일치: 이미 적용됨, 스킵
  - 불일치: "Validate failed: Migrations have failed validation"
```

### 4. 동시성 제어 — Lock 메커니즘

```java
// Flyway는 내부적으로 다음과 같이 Lock을 관리
public class FlywayLockingDemo {
    
    private static final String LOCK_TABLE = "flyway_schema_history_lock";
    
    public static void main(String[] args) {
        // 여러 스레드가 동시에 마이그레이션 시작
        
        // [스레드1] "서버1에서 마이그레이션 시작"
        // ↓ flyway_schema_history_lock 테이블에서 Lock 획득 시도
        // BEGIN TRANSACTION;
        // SELECT * FROM flyway_schema_history_lock WHERE name = 'migration' FOR UPDATE NOWAIT;
        // → Lock 획득 성공! 계속 진행
        
        // [스레드2] "서버2에서 마이그레이션 시작 (동시)"
        // ↓ Lock 획득 시도
        // SELECT * FROM flyway_schema_history_lock FOR UPDATE NOWAIT;
        // → Lock 대기 중... (스레드1이 해제할 때까지)
        
        // [스레드1] 마이그레이션 완료
        // COMMIT;  ← Lock 자동 해제
        
        // [스레드2] Lock 획득 가능!
        // SELECT * FROM flyway_schema_history_lock FOR UPDATE NOWAIT;
        // → 이제 실행 가능 (하지만 이미 스레드1이 적용했으므로)
        // flyway info로 확인 → 이미 적용됨, 스킵
    }
}

-- Lock 테이블 자동 생성 DDL (MySQL)
CREATE TABLE flyway_schema_history_lock (
    `index` int NOT NULL,
    `description` varchar(100) NOT NULL,
    PRIMARY KEY (`index`)
);

-- 초기 데이터
INSERT INTO flyway_schema_history_lock VALUES (1, 'Lock for Flyway migrations');
```

```sql
-- 실제 Lock 동작 확인 (MySQL)
-- [터미널 1]
mysql> BEGIN;
mysql> SELECT * FROM flyway_schema_history_lock WHERE `index` = 1 FOR UPDATE;
-- Lock 획득 (터미널2가 대기함)

-- [터미널 2] (동시 실행)
mysql> BEGIN;
mysql> SELECT * FROM flyway_schema_history_lock WHERE `index` = 1 FOR UPDATE;
-- 대기 중... (터미널1이 COMMIT할 때까지)

-- [터미널 1]
mysql> COMMIT;
-- Lock 해제

-- [터미널 2]
-- 이제 Lock 획득됨!
mysql> COMMIT;
```

---

## 💻 실전 실험

### 실험 1: Docker Compose로 Flyway 자동 마이그레이션 관찰

```yaml
# docker-compose.yml
version: '3.8'
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: testdb
    ports:
      - "3306:3306"
    volumes:
      - ./migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      timeout: 10s
      retries: 10

  flyway:
    image: flyway/flyway:9.22
    command: -url=jdbc:mysql://mysql:3306/testdb -user=root -password=root -locations=filesystem:/flyway/sql migrate
    volumes:
      - ./sql:/flyway/sql
    depends_on:
      mysql:
        condition: service_healthy
```

```bash
# 디렉토리 구조
mkdir -p sql migrations

# sql/V1__initial_schema.sql 생성
cat > sql/V1__initial_schema.sql << 'EOF'
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
EOF

# sql/V2__add_users_index.sql
cat > sql/V2__add_users_index.sql << 'EOF'
CREATE INDEX idx_users_email ON users(email);
EOF

# 실행
docker-compose up

# 로그 확인 (마이그레이션 성공 메시지)
# ...
# Successfully applied 2 migrations to schema 'testdb'
```

### 실험 2: flyway_schema_history 내용 조회

```bash
# MySQL에 접속
docker exec -it <container_id> mysql -uroot -proot testdb

# 테이블 조회
mysql> SELECT * FROM flyway_schema_history\G

# 출력:
# *************************** 1. row ***************************
#  installed_rank: 1
#      version: 1
#   description: initial schema
#         type: SQL
#        script: V1__initial_schema.sql
#      checksum: 1234567890
#   installed_by: root
#    installed_on: 2026-04-13 10:15:23
# execution_time: 145
#       success: 1

# 체크섬 계산 확인
# (Unix 명령어로 파일의 CRC32를 직접 계산할 수도 있음)
```

### 실험 3: 파일 수정 후 체크섬 불일치 오류 재현

```bash
# sql/V1__initial_schema.sql 수정 (세미콜론 추가)
cat > sql/V1__initial_schema.sql << 'EOF'
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  email VARCHAR(100) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);;  -- 세미콜론 2개!
EOF

# Flyway 재실행
docker-compose up flyway

# 오류 메시지:
# ERROR: Validate failed: Migrations have failed validation
# Migration checksum mismatch for migration version 1
#   Got:      1234567899  (새로운 체크섬)
#   Expected: 1234567890  (저장된 체크섬)
```

### 실험 4: Lock 테이블 확인

```bash
# MySQL에 접속
docker exec -it <container_id> mysql -uroot -proot testdb

# Lock 테이블 확인
mysql> DESCRIBE flyway_schema_history_lock;
+--------------+--------------+------+-----+---------+-------+
| Field        | Type         | Null | Key | Default | Extra |
+--------------+--------------+------+-----+---------+-------+
| `index`      | int          | NO   | PRI | NULL    |       |
| description  | varchar(100) | NO   |     | NULL    |       |
+--------------+--------------+------+-----+---------+-------+

mysql> SELECT * FROM flyway_schema_history_lock;
+-------+---------------------------+
| index | description               |
+-------+---------------------------+
|     1 | Lock for Flyway migrations|
+-------+---------------------------+
```

### 실험 5: Spring Boot와 함께 사용

```java
// pom.xml
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>9.22.0</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb
    username: root
    password: root
    driver-class-name: com.mysql.cj.jdbc.Driver
  
  jpa:
    hibernate:
      ddl-auto: validate  # Flyway가 관리하므로 validate만
  
  flyway:
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true
    clean-disabled: true
    out-of-order: false

# 실행 시 로그:
# o.f.c.i.database.base.DatabaseFactory  : Database: MySQL 8.0.35
# o.f.c.i.s.JdbcTableSchemaHistory       : Creating Schema History table: `flyway_schema_history` ...
# o.f.c.i.s.JdbcTableSchemaHistory       : Creating Lock table: `flyway_schema_history_lock` ...
# o.f.c.i.c.DbValidate                   : Successfully validated 2 migrations
# o.f.c.i.c.DbMigrate                    : Migrating database to version 1 - initial schema
# o.f.c.i.c.DbMigrate                    : Migrating database to version 2 - add users index
# o.f.c.i.c.DbMigrate                    : Successfully applied 2 migrations to schema `testdb` (execution time 125 ms)
```

```bash
# Spring Boot 애플리케이션 실행
mvn spring-boot:run

# 로그에서 다음 메시지 확인:
# "Successfully applied 2 migrations to schema `testdb`"
```

---

## 📊 성능/비용 비교

```
┌─────────────────────────────────────────────────────────────────┐
│           flyway_schema_history 검색 성능 비교                     │
├─────────────────────┬──────────┬──────────┬──────────────────────┤
│ 시나리오            │ 쿼리 수  │ 실행시간 │ 특징                 │
├─────────────────────┼──────────┼──────────┼──────────────────────┤
│ 처음 시작           │ 2-3개    │ 100-200ms│ 테이블 생성          │
│ (테이블 없음)       │          │          │ + Lock 생성          │
├─────────────────────┼──────────┼──────────┼──────────────────────┤
│ 마이그레이션 5개    │ 6-8개    │ 150-300ms│ 각 파일마다          │
│ (모두 미적용)       │          │          │ CRC32 계산           │
├─────────────────────┼──────────┼──────────┼──────────────────────┤
│ 마이그레이션 5개    │ 1개      │ 10-20ms  │ SELECT로 확인        │
│ (모두 적용됨)       │          │          │ 스킵                 │
├─────────────────────┼──────────┼──────────┼──────────────────────┤
│ 마이그레이션 50개   │ 52-58개  │ 500ms-2s │ 파일 많을수록 느림   │
│ (일부 미적용)       │          │          │                      │
├─────────────────────┼──────────┼──────────┼──────────────────────┤
│ flyway repair       │ 51개+    │ 1-3초    │ 모든 파일 재계산     │
│                     │          │          │ DB 업데이트          │
└─────────────────────┴──────────┴──────────┴──────────────────────┘

동시성 시나리오:
┌──────────────────────────────────────┬──────────┬──────────┐
│ 상황                                 │ 대기시간 │ 결과     │
├──────────────────────────────────────┼──────────┼──────────┤
│ 2개 인스턴스 동시 시작 (Lock 있음)    │ 200-500ms│ 안전     │
│ 10개 인스턴스 동시 시작               │ 1-2초    │ 안전     │
│ Lock 없을 때 (설정 오류)              │ 0ms      │ 위험!    │
│ Lock timeout 30초, 50회 재시도        │ 최대 30초│ 안전     │
└──────────────────────────────────────┴──────────┴──────────┘

저장 공간 비용:
┌────────────────────┬──────────┬────────────┐
│ 마이그레이션 수    │ 테이블   │ 추가 공간  │
├────────────────────┼──────────┼────────────┤
│ 10개               │ ~2KB     │ 무시할 수준│
│ 100개              │ ~15KB    │ 무시할 수준│
│ 1,000개            │ ~150KB   │ 무시할 수준│
└────────────────────┴──────────┴────────────┘
(실제로는 마이그레이션 내용이 저장되지 않으므로)
```

---

## ⚖️ 트레이드오프

```
┌────────────────────────────────────────────────────────────────┐
│ 체크섬 기반 변경 감지                                           │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 파일 변경을 즉시 감지                          │
│              │ - 실수로 인한 데이터 손상 방지                   │
│              │ - CRC32은 매우 빠름 (O(n))                      │
│              │ - 저장 공간 최소 (4바이트)                       │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 줄바꿈(CRLF ↔ LF) 변경만으로도 감지            │
│              │ - BOM(Byte Order Mark) 포함 시 감지              │
│              │ - 합법적 수정이 필요한 경우 repair 필요          │
│              │ - 팀 구성원 모두에게 규칙 설명 필요              │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ Lock 기반 동시성 제어                                           │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 중복 마이그레이션 실행 완벽하게 방지           │
│              │ - 여러 DB 벤더(MySQL, PG, Oracle 등) 지원        │
│              │ - 추가 인프라 필요 없음 (DB Lock만 사용)         │
│              │ - 자동 해제 (커밋 시)                           │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - Lock 대기로 인한 시작 지연 (최악 몇 초)         │
│              │ - 롤링 배포 시 Lock으로 인한 순차 처리           │
│              │ - 일부 DB에서는 NOWAIT가 지원 안 됨             │
│              │ - Lock timeout 설정 필요                        │
└──────────────┴─────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ 버전 고정 (V 마이그레이션)                                      │
├──────────────┬─────────────────────────────────────────────────┤
│ 장점         │ - 순서 보장                                      │
│              │ - 실수로 인한 재실행 방지                        │
│              │ - 버전 히스토리 명확                             │
├──────────────┼─────────────────────────────────────────────────┤
│ 단점         │ - 일단 적용되면 수정 불가                        │
│              │ - 수정 시 새 버전으로 역변경해야 함              │
│              │ - 파일 많아지면 관리 어려움                      │
└──────────────┴─────────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. flyway_schema_history는 모든 마이그레이션의 영구 기록         │
│    - 10개 컬럼(버전, 설명, 체크섬, 적용자, 시간, 성공 여부 등)   │
│    - 단 한 번만 수정, 그 이후는 읽기만 가능                      │
│                                                                 │
│ 2. Spring Boot 시작 시 자동으로 다음 단계를 실행                │
│    - flyway_schema_history 존재 확인 → 없으면 생성               │
│    - 마이그레이션 파일 스캔 → CRC32 체크섬 계산                 │
│    - DB의 이전 기록과 비교 → 미적용만 실행                      │
│                                                                 │
│ 3. CRC32 체크섬은 파일 내용의 "지문"                            │
│    - 1비트만 달라도 다른 숫자 → 변경 감지 완벽                 │
│    - 하지만 CRLF나 BOM 같은 사소한 변경도 감지                 │
│                                                                 │
│ 4. 여러 인스턴스 동시 시작 시에도 Lock으로 안전                 │
│    - flyway_schema_history_lock 테이블 자동 생성                │
│    - SELECT ... FOR UPDATE로 상호 배제                          │
│    - 한 번에 하나의 인스턴스만 마이그레이션 실행                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🤔 생각해볼 문제

**Q1.** CRC32 체크섬이 음수(-1234567890)가 될 수 있는데, 왜 그런가? 그리고 이것이 실무에서 어떤 문제를 일으킬 수 있는가?

<details>
<summary>해설 보기</summary>

CRC32는 원래 32비트 unsigned 정수(0 ~ 4,294,967,295)이지만, Java의 `int`는 signed(부호 있음) 정수(-2,147,483,648 ~ 2,147,483,647)이므로 음수로 표현될 수 있습니다.

```java
// 예시
CRC32 crc = new CRC32();
crc.update("CREATE TABLE users (...)".getBytes());
long unsignedValue = crc.getValue();  // 예: 3456789012 (unsigned)
int signedValue = (int) unsignedValue;  // -838178284 (signed, 음수!)
```

**실무 문제:**
- 사소한 문제: 로그나 대시보드에서 음수 체크섬을 보면 혼동할 수 있음
- 중요한 문제: 아직 없음 (Flyway는 내부적으로 올바르게 처리)
- 하지만 체크섬을 직접 비교하거나 외부 시스템에 전송할 때 주의 필요

**최상의 실천:**
```java
// ❌ 절대 하지 말 것
if (checksumFromDB == -1234567890) { ... }  // 비교 불안정

// ✅ 올바른 방법
// Flyway는 내부적으로 정확히 처리하므로 신경 쓸 필요 없음
// 직접 비교 필요 시 모두 signed int로 변환 후 비교
```

</details>

---

**Q2.** 개발 환경에서는 `baseline-on-migrate: true`로 설정해도 되지만, 프로덕션에서는 반드시 `false`여야 한다고 했다. 왜일까? 만약 프로덕션 DB가 마이그레이션 없이 직접 생성됐다면?

<details>
<summary>해설 보기</summary>

**`baseline-on-migrate: true`의 의미:**
- DB가 비어있으면 현재 상태를 baseline으로 설정하고 마이그레이션 실행
- DB에 이미 스키마가 있으면 그 상태를 baseline으로 하고 새 마이그레이션만 실행

**왜 프로덕션에서 위험한가:**
1. **데이터 손상 위험**: 기존 DB의 상태를 몰라서 잘못된 baseline을 설정할 수 있음
2. **버전 추적 불가**: 이미 적용된 마이그레이션 기록이 없어서 나중에 추적 불가
3. **다른 서버와 불일치**: 서버A는 baseline으로 설정, 서버B는 모든 마이그레이션 실행 → 스키마 불일치

**프로덕션 DB가 마이그레이션 없이 생성된 경우:**
```sql
-- 1단계: 수동으로 baseline 레코드 생성
INSERT INTO flyway_schema_history 
(installed_rank, version, description, type, script, checksum, installed_by, installed_on, execution_time, success)
VALUES (1, '1', 'baseline', 'SQL', 'baseline', 0, 'admin', NOW(), 0, true);

-- 2단계: 그 이후 새 마이그레이션만 실행
-- Spring Boot 시작 → V2 이상의 마이그레이션만 실행

-- 3단계: baseline-on-migrate는 false로 유지
spring:
  flyway:
    baseline-on-migrate: false
```

**절대 하면 안 되는 것:**
```yaml
# ❌ 프로덕션에서 이렇게 하면 안 됨
spring:
  flyway:
    baseline-on-migrate: true
    clean-disabled: false  # CLEAN도 허용?!
# → 실수로 clean() 호출 시 모든 데이터 삭제
```

</details>

---

**Q3.** 여러 Kubernetes Pod이 동시에 시작되는 경우, Lock이 없으면 어떻게 될까? 그리고 Lock이 있어도 문제가 생길 수 있는 경우는?

<details>
<summary>해설 보기</summary>

**Lock이 없을 때 (Lock 테이블 미생성):**
```
Pod1: 00:00:00 - V1 시작 실행
  ↓ (데이터 반 쓰기)
Pod2: 00:00:01 - V1 시작 실행 (동시!)
  ↓ (이미 생성된 테이블 만들려고 함)
❌ ERROR: Table 'users' already exists
```

**Lock이 있을 때:**
```
Pod1: 00:00:00 - Lock 획득 → V1 실행 (500ms)
Pod2: 00:00:01 - Lock 대기 중...
Pod3: 00:00:02 - Lock 대기 중...
Pod1: 00:00:00.5 - Lock 해제
Pod2: 00:00:00.5 - Lock 획득 → V1 이미 적용됨, 확인만 함 (10ms)
Pod2: 00:00:00.51 - Lock 해제
Pod3: 00:00:00.51 - Lock 획득 → V1 이미 적용됨, 확인만 함 (10ms)
✅ 모두 안전하게 시작됨
```

**Lock이 있어도 문제가 생기는 경우:**

1. **Lock timeout이 너무 짧음**
```java
// flyway.lock-retry-count = 3 (기본 50)
// Pod1이 잠깐 멈춤 (GC paused 등)
Pod2: Lock 획득 기다림... 3번 재시도... 
❌ Lock timeout! 마이그레이션 포기 → 앱 시작 실패
```

2. **데이터베이스 자체가 느려서 Lock 획득이 오래 걸림**
```
Pod1~10: 모두 Lock 기다림
Lock 획득: 이론상 순차적이지만 실제로는 경쟁
→ 프로덕션 시작 시간 10배 증가
```

3. **Lock 메커니즘이 구현되지 않은 DB (드물지만)**
```
특정 구형 DB에서는 SELECT ... FOR UPDATE가 작동하지 않을 수 있음
→ 이 경우 Lock 불가능, 순차 시작 권장
```

**최적 설정:**
```yaml
spring:
  flyway:
    lock-retry-count: 50  # 기본값, 좋음
    # lock-retry-count * lock-retry-delay = 최대 대기 시간
    # 기본: 50 * 1초 = 50초 (충분함)
```

</details>

---

<div align="center">

**[⬅️ 이전: Chapter 1 — 마이그레이션 환경 전략](../schema-management/05-environment-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이그레이션 유형 ➡️](./02-migration-types.md)**

</div>
