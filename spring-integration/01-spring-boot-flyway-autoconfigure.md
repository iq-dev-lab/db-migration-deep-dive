# Spring Boot + Flyway 자동 설정

---

## 🎯 핵심 질문

Spring Boot는 어떤 조건에서 Flyway를 자동으로 초기화하고, 그 과정을 커스터마이징하려면 어떻게 해야 할까? 멀티 DataSource 환경에서는 각 DataSource마다 독립적인 Flyway 설정이 필요한데, 이를 어떻게 구현할까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Boot의 자동 설정은 "관례에 의한 설정(Convention over Configuration)" 철학을 따릅니다. Flyway의 자동 설정을 이해하지 않으면:

1. **예상 밖의 마이그레이션 실행**: 개발 환경에서만 마이그레이션을 원하는데 프로덕션에서도 실행되거나 반대의 경우
2. **멀티 DataSource 환경 복잡성**: 주 DB와 읽기 전용 DB, 메타데이터 DB 등 여러 DB를 사용할 때 각각을 독립적으로 관리해야 함
3. **테스트 환경 격리 부족**: 테스트마다 DB 상태가 초기화되지 않아 테스트 간 의존성 발생
4. **배포 후 문제**: 자동 설정의 기본값이 프로덕션에 맞지 않아 시작 실패 또는 예상 밖의 DDL 실행

---

## 😱 흔한 실수 (Before)

### 문제 1: 모든 환경에서 자동 마이그레이션 실행

```yaml
# application.yml (문제: 프로덕션에서도 자동 마이그레이션)
spring:
  jpa:
    hibernate:
      ddl-auto: validate
  datasource:
    url: jdbc:mysql://prod-db:3306/myapp
```

**문제점**: 의존성에 `flyway-core`만 있으면 프로덕션 배포 시에도 자동으로 마이그레이션이 실행됩니다. 만약 마이그레이션에 버그가 있다면 배포 직후 서비스 중단이 발생합니다.

### 문제 2: 멀티 DataSource에서 Flyway 설정 누락

```java
// 문제: 두 번째 DataSource에 대한 Flyway 설정이 없음
@Configuration
public class DataSourceConfig {
    @Bean(name = "primaryDataSource")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create()
            .driverClassName("com.mysql.cj.jdbc.Driver")
            .url("jdbc:mysql://primary-db:3306/main")
            .username("user")
            .password("pass")
            .build();
    }
    
    @Bean(name = "replicaDataSource")
    public DataSource replicaDataSource() {
        // Replica DB는 읽기 전용 → 마이그레이션 불필요
        return DataSourceBuilder.create()
            .driverClassName("com.mysql.cj.jdbc.Driver")
            .url("jdbc:mysql://replica-db:3306/main")
            .username("user")
            .password("pass")
            .build();
    }
}
```

**문제점**: Replica DB에 Flyway가 자동 설정되어 마이그레이션을 시도하므로 쓰기 권한이 없어 실패합니다.

### 문제 3: 테스트 시 실제 DB가 초기화됨

```java
@SpringBootTest
@Transactional
public class UserServiceTest {
    // 각 테스트마다 전체 마이그레이션이 재실행
    // 다른 테스트가 남긴 데이터가 섞임
    
    @Test
    public void testCreateUser() {
        // ...
    }
}
```

**문제점**: `@FlywayTest`를 사용하지 않으면 테스트마다 깨끗한 스키마를 보장하지 못합니다.

---

## ✨ 올바른 접근 (After)

### 올바른 Approach 1: 환경별 자동 설정 제어

```yaml
# application.yml (공통)
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baselineOnMigrate: true
    
---
# application-dev.yml (개발: 자동 마이그레이션 활성화)
spring:
  profiles: dev
  flyway:
    enabled: true
  jpa:
    hibernate:
      ddl-auto: validate

---
# application-prod.yml (프로덕션: 자동 마이그레이션 비활성화)
spring:
  profiles: prod
  flyway:
    enabled: false  # 수동 배포 프로세스에서만 실행
  jpa:
    hibernate:
      ddl-auto: validate
```

그 후 프로덕션에서는 배포 파이프라인에서 명시적으로 실행:

```bash
# 프로덕션 배포 파이프라인
java -jar myapp.jar migrate  # 커스텀 커맨드
# 또는
mvn flyway:migrate -Dflyway.configFiles=prod-config.conf
```

### 올바른 Approach 2: FlywayMigrationStrategy로 동작 커스터마이징

```java
@Configuration
public class FlywayConfig {
    
    /**
     * 마이그레이션 전 repair() 실행
     * (이전 실패한 마이그레이션으로 인한 상태 정상화)
     */
    @Bean
    public FlywayMigrationStrategy flywayMigrationStrategy() {
        return flyway -> {
            // 1. 이전 실패한 마이그레이션 정상화
            flyway.repair();
            
            // 2. 마이그레이션 실행
            flyway.migrate();
        };
    }
}
```

**동작 원리**:
- `repair()`: flyway_schema_history에서 failed=TRUE인 행을 success=TRUE로 변경
- 이를 통해 다음 마이그레이션이 정상적으로 시작 가능

### 올바른 Approach 3: 멀티 DataSource + 독립적 Flyway 설정

```java
@Configuration
public class MultiDataSourceFlywayConfig {
    
    // === 주 DataSource (쓰기 가능) ===
    @Bean
    @Primary
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create()
            .driverClassName("com.mysql.cj.jdbc.Driver")
            .url("jdbc:mysql://primary-db:3306/main")
            .username("admin")
            .password("password")
            .build();
    }
    
    @Bean
    @Primary
    public Flyway primaryFlyway(
            @Qualifier("primaryDataSource") DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/primary")
            .baselineOnMigrate(true)
            .load();
    }
    
    @Bean
    public FlywayMigrationInitializer primaryFlywayInitializer(
            Flyway primaryFlyway) {
        return new FlywayMigrationInitializer(primaryFlyway);
    }
    
    // === 복제 DataSource (읽기 전용) ===
    @Bean
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create()
            .driverClassName("com.mysql.cj.jdbc.Driver")
            .url("jdbc:mysql://replica-db:3306/main")
            .username("readonly")
            .password("password")
            .build();
    }
    
    // Replica는 마이그레이션 불필요 (읽기 전용)
    // PrimaryFlyway가 실행 후 Replication으로 자동 동기화
    
    // === 메타데이터 DataSource ===
    @Bean
    public DataSource metadataDataSource() {
        return DataSourceBuilder.create()
            .driverClassName("com.mysql.cj.jdbc.Driver")
            .url("jdbc:mysql://metadata-db:3306/metadata")
            .username("admin")
            .password("password")
            .build();
    }
    
    @Bean
    public Flyway metadataFlyway(
            @Qualifier("metadataDataSource") DataSource dataSource) {
        return Flyway.configure()
            .dataSource(dataSource)
            .locations("classpath:db/migration/metadata")
            .schemas("metadata")
            .table("flyway_schema_history_metadata")
            .baselineOnMigrate(true)
            .load();
    }
    
    @Bean
    public FlywayMigrationInitializer metadataFlywayInitializer(
            Flyway metadataFlyway) {
        return new FlywayMigrationInitializer(metadataFlyway);
    }
    
    // === JPA EntityManager 설정 ===
    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            @Qualifier("primaryDataSource") DataSource dataSource) {
        LocalContainerEntityManagerFactoryBean em = 
            new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource);
        em.setPackagesToScan("com.example.domain");
        
        JpaVendorAdapter vendorAdapter = new HibernateJpaVendorAdapter();
        em.setJpaVendorAdapter(vendorAdapter);
        
        Properties props = new Properties();
        props.setProperty("hibernate.dialect", 
            "org.hibernate.dialect.MySQL8Dialect");
        props.setProperty("hibernate.ddl-auto", "validate");
        em.setJpaProperties(props);
        
        return em;
    }
    
    @Bean
    public PlatformTransactionManager transactionManager(
            EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

**핵심 포인트**:
1. `@Primary`: 주 DataSource를 기본값으로 설정
2. `@Qualifier`: 특정 DataSource 명시적 선택
3. `FlywayMigrationInitializer`: Spring Boot가 Flyway 초기화를 담당하도록 함
4. `schemas()`: 각 Flyway가 사용할 스키마 명시 (선택사항)
5. `table()`: 각 Flyway가 사용할 히스토리 테이블명 분리

### 올바른 Approach 4: 테스트 격리를 위한 @FlywayTest

```java
// pom.xml 또는 build.gradle에 추가
// <dependency>
//     <groupId>org.flywaydb.flyway-test-extensions</groupId>
//     <artifactId>flyway-spring-test</artifactId>
//     <version>9.0.0</version>
//     <scope>test</scope>
// </dependency>

@SpringBootTest
@FlywayTest  // 각 테스트 전에 마이그레이션 재실행
public class UserRepositoryTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @BeforeEach
    public void setUp() {
        // @FlywayTest가 이미 마이그레이션 완료
        // 여기서 추가 데이터 셋업 가능
    }
    
    @Test
    @FlywayTest(locationsForMigrate = "classpath:db/migration/test")
    public void testUserCreation() {
        User user = new User();
        user.setName("Test User");
        user.setEmail("test@example.com");
        
        userRepository.save(user);
        
        User found = userRepository.findByEmail("test@example.com").orElse(null);
        assertNotNull(found);
        assertEquals("Test User", found.getName());
    }
    
    @Test
    @FlywayTest(locationsForMigrate = {
        "classpath:db/migration",
        "classpath:db/migration/test"
    })
    public void testWithMultipleMigrationLocations() {
        // 주 마이그레이션 + 테스트 마이그레이션 모두 실행
        // 예: R__insert_test_data.sql이 매번 재실행됨
        
        int count = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM users", Integer.class);
        assertEquals(10, count);  // R__insert_test_data.sql이 10건 삽입
    }
}
```

---

## 🔬 내부 동작 원리

### 1. FlywayAutoConfiguration 활성화 조건

```
Spring Boot 시작
  ↓
@ConditionalOnClass(Flyway.class) 확인
  ├─ flyway-core 의존성 확인
  ├─ jdbc 또는 r2dbc 드라이버 확인
  └─ DataSource 빈 확인
  ↓
FlywayAutoConfiguration 활성화
  ↓
FlywayMigrationInitializer 생성
  ↓
ApplicationContext.publishEvent(MigrationStartedEvent)
  ↓
Flyway.migrate() 실행
  ↓
ApplicationContext.publishEvent(MigrationFinishedEvent)
```

**조건 상세**:
- `@ConditionalOnBean(DataSource.class)`: DataSource 빈 필수
- `@ConditionalOnProperty("spring.flyway.enabled", matchIfMissing = true)`: 기본값은 true

### 2. FlywayMigrationInitializer 역할

```java
// Spring Boot 내부 코드 (참고)
public class FlywayMigrationInitializer 
        implements InitializingBean, Ordered {
    
    private final Flyway flyway;
    
    @Override
    public void afterPropertiesSet() throws Exception {
        // 모든 빈 초기화 완료 후 실행
        // InitializingBean.afterPropertiesSet()은 
        // BeanPostProcessor보다 뒤에 실행
        
        if (shouldMigrate()) {
            this.flyway.migrate();  // 마이그레이션 실행
        }
    }
    
    @Override
    public int getOrder() {
        return Ordered.HIGHEST_PRECEDENCE;  // 우선순위 최고
    }
}
```

**실행 순서**:
1. DataSource 초기화
2. JPA EntityManagerFactory 초기화
3. FlywayMigrationInitializer.afterPropertiesSet() 호출
4. Flyway.migrate() 실행
5. Other beans initialization

### 3. 마이그레이션 히스토리 관리

```sql
-- flyway_schema_history 테이블 구조
CREATE TABLE flyway_schema_history (
    installed_rank INT NOT NULL PRIMARY KEY,
    version VARCHAR(50),
    description VARCHAR(255) NOT NULL,
    type VARCHAR(20) NOT NULL,  -- SQL, JDBC, UNDO
    script VARCHAR(1000) NOT NULL,
    checksum INT,
    installed_by VARCHAR(100) NOT NULL,
    installed_on TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    execution_time INT NOT NULL,  -- ms
    success TINYINT(1) NOT NULL,
    INDEX idx_success (success)
);
```

**success 필드 의미**:
- `1`: 성공
- `0`: 실패 (repair() 전)
- `1`: 실패 후 repair()로 정상화

---

## 💻 실전 실험

### 실험 1: FlywayMigrationStrategy로 repair 후 migrate 실행

**파일 구조**:
```
src/main/resources/db/migration/
├── V1__create_users_table.sql
├── V2__add_email_column.sql
└── V3__create_orders_table.sql
```

**V1__create_users_table.sql**:
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**V2__add_email_column.sql**:
```sql
ALTER TABLE users ADD COLUMN email VARCHAR(100) UNIQUE;
```

**V3__create_orders_table.sql**:
```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    total DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

**FlywayConfig.java**:
```java
package com.example.config;

import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.FlywayException;
import org.springframework.boot.autoconfigure.flyway.FlywayMigrationStrategy;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Profile;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Configuration
public class FlywayConfig {
    
    @Bean
    @Profile("!test")
    public FlywayMigrationStrategy flywayMigrationStrategy() {
        return flyway -> {
            log.info("Starting Flyway migration with repair strategy...");
            
            try {
                // 1단계: 이전 실패한 마이그레이션 복구
                int repairedCount = flyway.repair();
                log.info("Repaired {} failed migration(s)", repairedCount);
                
                // 2단계: 마이그레이션 실행
                org.flywaydb.core.api.MigrationInfo[] migrations = 
                    flyway.migrate().migrations;
                log.info("Successfully migrated {} migration(s)", 
                    migrations.length);
                
                for (org.flywaydb.core.api.MigrationInfo migration : migrations) {
                    log.debug("Applied: {} - {}", 
                        migration.getVersion(), 
                        migration.getDescription());
                }
            } catch (FlywayException e) {
                log.error("Flyway migration failed: {}", e.getMessage());
                throw e;
            }
        };
    }
}
```

**테스트 코드**:
```java
package com.example;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.TestPropertySource;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
@TestPropertySource(properties = {
    "spring.flyway.enabled=true",
    "spring.jpa.hibernate.ddl-auto=validate"
})
public class FlywayMigrationTest {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Test
    public void testMigrationSucceeded() {
        // users 테이블 존재 확인
        Integer userTableCount = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM information_schema.TABLES " +
            "WHERE TABLE_SCHEMA=DATABASE() AND TABLE_NAME='users'",
            Integer.class);
        assertEquals(1, userTableCount, "users 테이블이 없음");
        
        // orders 테이블 존재 확인
        Integer orderTableCount = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM information_schema.TABLES " +
            "WHERE TABLE_SCHEMA=DATABASE() AND TABLE_NAME='orders'",
            Integer.class);
        assertEquals(1, orderTableCount, "orders 테이블이 없음");
        
        // email 컬럼 존재 확인
        Integer emailColumnCount = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM information_schema.COLUMNS " +
            "WHERE TABLE_SCHEMA=DATABASE() AND TABLE_NAME='users' " +
            "AND COLUMN_NAME='email'",
            Integer.class);
        assertEquals(1, emailColumnCount, "email 컬럼이 없음");
    }
}
```

**실행 결과**:
```
2026-04-13 10:15:23 INFO FlywayConfig - Starting Flyway migration with repair strategy...
2026-04-13 10:15:23 INFO FlywayConfig - Repaired 0 failed migration(s)
2026-04-13 10:15:23 INFO FlywayConfig - Successfully migrated 3 migration(s)
2026-04-13 10:15:23 DEBUG FlywayConfig - Applied: 1 - create users table
2026-04-13 10:15:23 DEBUG FlywayConfig - Applied: 2 - add email column
2026-04-13 10:15:23 DEBUG FlywayConfig - Applied: 3 - create orders table
```

---

### 실험 2: 멀티 DataSource 독립적 Flyway 구성

**application.yml**:
```yaml
spring:
  datasource:
    primary:
      url: jdbc:mysql://localhost:3306/primary_db
      username: root
      password: password
      driver-class-name: com.mysql.cj.jdbc.Driver
    replica:
      url: jdbc:mysql://localhost:3307/replica_db
      username: root
      password: password
      driver-class-name: com.mysql.cj.jdbc.Driver
```

**MultiDataSourceConfig.java**:
```java
package com.example.config;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import org.flywaydb.core.Flyway;
import org.springframework.boot.autoconfigure.flyway.FlywayMigrationInitializer;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.orm.jpa.JpaTransactionManager;
import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
import org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter;
import org.springframework.transaction.PlatformTransactionManager;

import javax.sql.DataSource;
import java.util.Properties;

@Configuration
@EnableJpaRepositories(
    basePackages = "com.example.repository",
    entityManagerFactoryRef = "entityManagerFactory",
    transactionManagerRef = "transactionManager"
)
public class MultiDataSourceConfig {
    
    // ===== Primary DataSource (쓰기 가능) =====
    
    @Bean(name = "primaryDataSource")
    @Primary
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource primaryDataSource() {
        return new HikariDataSource();
    }
    
    @Bean(name = "primaryFlyway")
    @Primary
    public Flyway primaryFlyway(
            javax.sql.DataSource primaryDataSource) {
        return Flyway.configure()
            .dataSource(primaryDataSource)
            .locations("classpath:db/migration/primary")
            .baselineOnMigrate(true)
            .baselineVersion("0")
            .load();
    }
    
    @Bean(name = "primaryFlywayInitializer")
    @Primary
    public FlywayMigrationInitializer primaryFlywayInitializer(
            Flyway primaryFlyway) {
        return new FlywayMigrationInitializer(primaryFlyway);
    }
    
    @Bean(name = "entityManagerFactory")
    @Primary
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            DataSource primaryDataSource) {
        LocalContainerEntityManagerFactoryBean em = 
            new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(primaryDataSource);
        em.setPackagesToScan("com.example.domain");
        em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        
        Properties props = new Properties();
        props.setProperty("hibernate.dialect", 
            "org.hibernate.dialect.MySQL8Dialect");
        props.setProperty("hibernate.ddl-auto", "validate");
        em.setJpaProperties(props);
        
        return em;
    }
    
    @Bean(name = "transactionManager")
    @Primary
    public PlatformTransactionManager transactionManager(
            LocalContainerEntityManagerFactoryBean entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory.getObject());
    }
    
    // ===== Replica DataSource (읽기 전용) =====
    
    @Bean(name = "replicaDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.replica")
    public DataSource replicaDataSource() {
        return new HikariDataSource();
    }
    
    // Replica는 마이그레이션 불필요
    // Primary DB의 마이그레이션 후 Replication으로 자동 동기화
}
```

**마이그레이션 파일**:
```
src/main/resources/db/migration/primary/
└── V1__initial_schema.sql
```

**V1__initial_schema.sql**:
```sql
-- Primary DB에만 적용될 마이그레이션
CREATE TABLE IF NOT EXISTS users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(100) NOT NULL UNIQUE,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS posts (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    title VARCHAR(255) NOT NULL,
    content LONGTEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_posts_user_id ON posts(user_id);
```

**통합 테스트**:
```java
package com.example.integration;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;

import javax.sql.DataSource;

import static org.junit.jupiter.api.Assertions.assertEquals;

@SpringBootTest
public class MultiDataSourceFlywayTest {
    
    @Autowired
    @Qualifier("primaryDataSource")
    private DataSource primaryDataSource;
    
    @Autowired
    @Qualifier("replicaDataSource")
    private DataSource replicaDataSource;
    
    @Test
    public void testPrimaryDataSourceMigration() {
        JdbcTemplate primaryJdbc = new JdbcTemplate(primaryDataSource);
        
        Integer usersTableCount = primaryJdbc.queryForObject(
            "SELECT COUNT(*) FROM information_schema.TABLES " +
            "WHERE TABLE_SCHEMA=DATABASE() AND TABLE_NAME='users'",
            Integer.class);
        assertEquals(1, usersTableCount, "Primary DB에서 users 테이블 없음");
        
        Integer postsTableCount = primaryJdbc.queryForObject(
            "SELECT COUNT(*) FROM information_schema.TABLES " +
            "WHERE TABLE_SCHEMA=DATABASE() AND TABLE_NAME='posts'",
            Integer.class);
        assertEquals(1, postsTableCount, "Primary DB에서 posts 테이블 없음");
    }
    
    @Test
    public void testReplicaDataSourceIndependence() {
        // Replica는 마이그레이션 대상이 아님
        // 별도 Replication으로 primary 테이블 동기화
        
        JdbcTemplate replicaJdbc = new JdbcTemplate(replicaDataSource);
        
        // 이 테스트는 primary 마이그레이션이 replica로 복제되었음을 확인
        // (실제 환경에서는 MySQL Replication 설정 필요)
    }
}
```

---

### 실험 3: @FlywayTest를 사용한 테스트 격리

**build.gradle**:
```gradle
dependencies {
    testImplementation 'org.flywaydb.flyway-test-extensions:flyway-spring-test:9.0.0'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    testImplementation 'com.h2database:h2'
}
```

**UserRepository.java**:
```java
package com.example.repository;

import com.example.domain.User;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByEmail(String email);
    Optional<User> findByUsername(String username);
}
```

**User.java**:
```java
package com.example.domain;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false, unique = true)
    private String username;
    
    @Column(unique = true)
    private String email;
    
    @Column(nullable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

**UserRepositoryFlywayTest.java**:
```java
package com.example.repository;

import com.example.domain.User;
import org.flywaydb.test.annotation.FlywayTest;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.TestPropertySource;

import static org.junit.jupiter.api.Assertions.*;

@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.ANY)
@TestPropertySource(properties = {
    "spring.flyway.enabled=true",
    "spring.flyway.locations=classpath:db/migration",
    "spring.jpa.hibernate.ddl-auto=validate"
})
public class UserRepositoryFlywayTest {
    
    @Autowired
    private UserRepository userRepository;
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    @Test
    @FlywayTest  // 각 테스트 전에 마이그레이션 재실행
    public void testSaveAndFindUser() {
        // 테스트 시작: 완벽하게 깨끗한 스키마
        
        User user = new User();
        user.setUsername("john_doe");
        user.setEmail("john@example.com");
        
        User saved = userRepository.save(user);
        assertNotNull(saved.getId());
        
        User found = userRepository.findByEmail("john@example.com")
            .orElse(null);
        assertNotNull(found);
        assertEquals("john_doe", found.getUsername());
    }
    
    @Test
    @FlywayTest
    public void testFindByUsername() {
        User user1 = new User();
        user1.setUsername("alice");
        user1.setEmail("alice@example.com");
        userRepository.save(user1);
        
        User user2 = new User();
        user2.setUsername("bob");
        user2.setEmail("bob@example.com");
        userRepository.save(user2);
        
        User found = userRepository.findByUsername("alice")
            .orElse(null);
        assertNotNull(found);
        assertEquals("alice@example.com", found.getEmail());
        
        // 다른 테스트의 영향 없음 (각 테스트마다 격리됨)
    }
    
    @Test
    @FlywayTest
    public void testUniqueConstraint() {
        User user1 = new User();
        user1.setUsername("charlie");
        user1.setEmail("charlie@example.com");
        userRepository.save(user1);
        userRepository.flush();
        
        // 같은 이메일로 저장 시도
        User user2 = new User();
        user2.setUsername("david");
        user2.setEmail("charlie@example.com");  // 중복
        
        assertThrows(Exception.class, () -> {
            userRepository.save(user2);
            userRepository.flush();
        });
    }
}
```

**테스트 실행**:
```bash
mvn test -Dtest=UserRepositoryFlywayTest

# 출력:
# [INFO] -------------------------------------------------------
# [INFO]  T E S T S
# [INFO] -------------------------------------------------------
# [INFO] Running com.example.repository.UserRepositoryFlywayTest
# 
# [FlywayTest] Cleaning database for test: testSaveAndFindUser
# [FlywayTest] Running Flyway migration...
# [FlywayTest] Migration completed: 1 applied
# [FlywayTest] Test passed: testSaveAndFindUser
# 
# [FlywayTest] Cleaning database for test: testFindByUsername
# [FlywayTest] Running Flyway migration...
# [FlywayTest] Migration completed: 1 applied
# [FlywayTest] Test passed: testFindByUsername
# 
# [INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
```

---

## 📊 성능/비용 비교

| 항목 | 자동 설정 | FlywayMigrationStrategy | 멀티 DataSource |
|------|---------|------------------------|-----------------|
| **초기화 시간** | 빠름 (100ms~500ms) | 약간 느림 (repair 오버헤드) | 각 DS별로 누적 (500ms~2s) |
| **프로덕션 적합성** | 낮음 (예상 못한 마이그레이션) | 중간 (제어 가능) | 높음 (명시적 관리) |
| **개발 편의성** | 높음 (설정 최소) | 중간 | 낮음 (많은 설정) |
| **실패 복구력** | 낮음 | 높음 (repair 자동) | 높음 (각각 독립) |
| **메모리 사용** | 적음 | 적음 | 많음 (N개 DataSource) |
| **DB 연결 수** | 1개 | 1개 | N개 (각 DS마다) |

---

## ⚖️ 트레이드오프

### 1. 자동 설정의 편의성 vs 명시성
- **편의성**: `spring-boot-starter-data-jpa` 추가만으로 자동 마이그레이션
- **위험**: 어떤 조건에서 마이그레이션이 실행되는지 불명확
- **해결**: 환경별 `application-{profile}.yml`로 명시적 제어

### 2. FlywayMigrationStrategy의 오버헤드
- **장점**: repair() 자동화로 실패 복구 자동화
- **단점**: 매번 repair() 호출로 약간의 성능 저하
- **권장사항**: 개발/테스트 환경에서만 사용

### 3. 멀티 DataSource의 복잡성
- **장점**: 각 DB를 독립적으로 관리 가능
- **단점**: 빈 설정이 2배 이상 증가
- **권장사항**: 필요할 때만 (주 DB + replica 정도는 자동 동기화로)

---

## 📌 핵심 정리

1. **Spring Boot의 FlywayAutoConfiguration은 DataSource + flyway-core 존재하면 자동 활성화**
   - `spring.flyway.enabled=true` (기본값)
   - FlywayMigrationInitializer가 애플리케이션 시작 후 마이그레이션 실행

2. **프로덕션에서는 자동 마이그레이션 비활성화 필수**
   - `application-prod.yml`에서 `spring.flyway.enabled=false`
   - 배포 파이프라인에서 명시적으로 실행

3. **FlywayMigrationStrategy로 커스터마이징 가능**
   - repair() + migrate() 순서로 자동 복구
   - 로깅, 조건부 실행 등 추가 로직 가능

4. **멀티 DataSource 환경에서는 각 DataSource마다 Flyway 설정 필요**
   - @Primary로 기본값 지정
   - @Qualifier로 특정 DataSource 명시
   - FlywayMigrationInitializer로 초기화 순서 제어

5. **테스트 환경에서는 @FlywayTest로 격리**
   - 각 테스트마다 깨끗한 마이그레이션 상태
   - 테스트 간 데이터 독립성 보장

---

## 🤔 생각해볼 문제

<details>
<summary><strong>Q1: 프로덕션과 개발 환경에서 같은 마이그레이션을 적용해야 할까, 다르게 적용해야 할까?</strong></summary>

**A**: 기본적으로 같은 마이그레이션을 적용해야 합니다. 대신 **실행 방식**만 다릅니다.

- **개발**: `spring.flyway.enabled=true` → 애플리케이션 시작 시 자동 마이그레이션
- **프로덕션**: `spring.flyway.enabled=false` → 배포 전 별도 단계에서 수동/CLI 실행

이유:
1. 스키마는 모든 환경에서 동일해야 함 (테스트, 스테이징, 프로덕션)
2. 마이그레이션 파일도 동일해야 함 (`src/main/resources/db/migration`)
3. 차이는 실행 시점/방식일 뿐 (auto vs manual)

예외:
- 테스트 전용 마이그레이션: `src/test/resources/db/migration` (별도 위치)
- 환경 특화 데이터: `classpath:db/migration/prod` vs `classpath:db/migration/dev` (선택사항)
</details>

<details>
<summary><strong>Q2: @FlywayTest를 사용하면 테스트 속도가 느려질까?</strong></summary>

**A**: 네, 느려질 수 있지만 정확성이 더 중요합니다.

**성능 비교**:
- 마이그레이션 없음: 100ms (테스트 시작)
- 마이그레이션 1개: 150~300ms (SQL 파일 실행)
- 마이그레이션 10개: 500ms~2s (누적)

**최적화 방법**:
1. 테스트 전용 마이그레이션 최소화
   - 필요한 스키마만 포함: `src/test/resources/db/migration`
   - 불필요한 마이그레이션은 건너뛰기

2. 병렬 테스트 실행
   ```gradle
   test {
       maxParallelForks = Runtime.getRuntime().availableProcessors()
   }
   ```

3. Testcontainers 캐싱
   - 첫 테스트: 컨테이너 생성 (3~5초)
   - 이후 테스트: 캐시 재사용 (100ms)

**권장사항**: 정확성 > 속도. 느리더라도 @FlywayTest 사용하고, 속도 개선은 나중에.
</details>

<details>
<summary><strong>Q3: 멀티 DataSource 환경에서 두 번째 DataSource에도 Flyway를 설정해야 할까?</strong></summary>

**A**: 상황에 따라 다릅니다.

**Flyway 설정 필요** (둘 다 쓰기):
```
Primary DB ←→ Secondary DB (양쪽 모두 쓰기)
├─ 각각 독립적인 마이그레이션 필요
├─ 다른 스키마 버전 가능
└─ 각 Flyway는 독립적인 flyway_schema_history 테이블
```

**Flyway 설정 불필요** (읽기 전용 Replica):
```
Primary DB → Replica DB (한쪽 쓰기, 한쪽 읽기)
├─ Primary DB만 마이그레이션
├─ Replication으로 자동 동기화
└─ Replica는 flyway_schema_history 불필요 (읽기 전용이므로)
```

**결정 기준**:
| 상황 | Flyway 설정 |
|-----|-----------|
| Replica (읽기 전용) | X |
| 캐시 DB (읽기 전용) | X |
| Secondary (쓰기 가능) | O |
| Sharded DB 각각 | O |

**구현 예시** (Replica는 제외):
```java
// 위의 "멀티 DataSource" 예제 참조
// replicaFlyway 빈을 정의하지 않음 → Replica는 마이그레이션 안 함
```
</details>

---

<div align="center">

**[⬅️ 이전: Chapter 6 — 마이그레이션 감사(Audit)와 컴플라이언스](../cicd-integration/05-migration-audit.md)** | **[홈으로 🏠](../README.md)** | **[다음: 테스트 데이터 관리 ➡️](./02-test-data-management.md)**

</div>
