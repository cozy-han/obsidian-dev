# AWS Advanced JDBC Wrapper 적용 가이드

> AWS Advanced JDBC Wrapper는 기존 MySQL JDBC 드라이버를 감싸서 Aurora 페일오버 자동 감지, Read/Write 분리 등의 기능을 추가하는 래퍼 드라이버다.
> Java 8+ 호환. 이 프로젝트(Java 1.8)에서 사용 가능.

## 현재 구조

```
애플리케이션
  ↓
log4jdbc (SQL 로깅 래퍼)
  ↓
mysql-connector-java (MySQL JDBC 드라이버)
  ↓
Aurora MySQL
```

## 목표 구조

```
애플리케이션
  ↓
AWS Advanced JDBC Wrapper (페일오버/클러스터 인식)
  ↓
mysql-connector-java (MySQL JDBC 드라이버)
  ↓
Aurora MySQL
```

> **중요:** log4jdbc와 AWS JDBC Wrapper는 둘 다 JDBC 래퍼여서 중첩이 어렵다.
> AWS Wrapper의 `logQuery` 플러그인으로 SQL 로깅을 대체하거나, log4jdbc를 제거해야 한다.

---

## Step 1. Maven 의존성 추가

### 1-1. 모든 모듈의 pom.xml에 추가

```xml
<!-- AWS Advanced JDBC Wrapper -->
<dependency>
    <groupId>software.amazon.jdbc</groupId>
    <artifactId>aws-advanced-jdbc-wrapper</artifactId>
    <version>3.2.0</version>
</dependency>
```

대상 모듈 (5개):
- `sumair-be-home/pom.xml`
- `sumair-be-ibe/pom.xml`
- `sumair-be-pay/pom.xml`
- `sumair-be-batch/pom.xml`
- `sumair-be-message/pom.xml`

> 최신 버전은 [Maven Central](https://mvnrepository.com/artifact/software.amazon.jdbc/aws-advanced-jdbc-wrapper)에서 확인

### 1-2. mysql-connector-java 유지

AWS Wrapper는 내부적으로 MySQL 드라이버를 사용하므로 기존 의존성을 그대로 유지한다.
단, **버전 통일 권장** (batch만 8.0.29 → 8.0.32로 맞추기).

---

## Step 2. application-databases.yml 변경

### 2-1. JDBC URL 형식 변경

```yaml
# AS-IS (log4jdbc + mysql)
jdbc-url: jdbc:log4jdbc:mysql://호스트:3306/sumair?params...
driver-class-name: net.sf.log4jdbc.sql.jdbcapi.DriverSpy

# TO-BE (aws-wrapper + mysql)
jdbc-url: jdbc:aws-wrapper:mysql://호스트:3306/sumair?params...
driver-class-name: software.amazon.jdbc.Driver
```

### 2-2. HikariCP에 Wrapper 설정 추가

```yaml
spring:
  datasource:
    main:
      jdbc-url: jdbc:aws-wrapper:mysql://sumair-aurora-cluster.cluster-xxxxx.ap-northeast-2.rds.amazonaws.com:3306/sumair?characterEncoding=UTF-8&useSSL=false&allowMultiQueries=true
      username: admin
      password: ENC(...)
      driver-class-name: software.amazon.jdbc.Driver
      hikari:
        data-source-properties:
          wrapperPlugins: failover,efm2
          wrapperDialect: aurora-mysql
        exception-override-class-name: software.amazon.jdbc.util.HikariCPSQLException
        maximum-pool-size: 20
        minimum-idle: 5
        max-lifetime: 840000
        idle-timeout: 60000
        connection-timeout: 30000
        validation-timeout: 5000
```

### 2-3. 전체 환경별 변경 예시

```yaml
### common (기본값)
spring:
  datasource:
    main:
      jdbc-url: jdbc:aws-wrapper:mysql://aurora-host:3306/sumair?characterEncoding=UTF-8&useSSL=false&allowMultiQueries=true
      username: sumair
      password: palnet!234
      driver-class-name: software.amazon.jdbc.Driver
      hikari:
        data-source-properties:
          wrapperPlugins: failover,efm2
          wrapperDialect: aurora-mysql
        exception-override-class-name: software.amazon.jdbc.util.HikariCPSQLException
        maximum-pool-size: 10
        minimum-idle: 10
        max-lifetime: 840000
        idle-timeout: 60000
        connection-timeout: 30000
        validation-timeout: 5000
    message:
      jdbc-url: jdbc:aws-wrapper:mysql://message-aurora-host:3306/sumair?characterEncoding=UTF-8&useSSL=false&serverTimezone=Asia/Seoul
      username: sumair
      password: xxx
      driver-class-name: software.amazon.jdbc.Driver
      hikari:
        data-source-properties:
          wrapperPlugins: failover,efm2
          wrapperDialect: aurora-mysql
        exception-override-class-name: software.amazon.jdbc.util.HikariCPSQLException
        maximum-pool-size: 3
        max-lifetime: 840000

---
### local
spring:
  config:
    activate:
      on-profile: local
  datasource:
    main:
      jdbc-url: jdbc:aws-wrapper:mysql://sumair-dev-aurora-writer.chh3noeqbmvd.ap-northeast-2.rds.amazonaws.com:3306/sumair?characterEncoding=UTF-8&useSSL=false&allowMultiQueries=true
      username: admin
      password: ENC(pZ8SuzYVK/TxFUX3kS2FfJUOFO2hau85THuXIanbbZo=)

---
### dev
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    main:
      jdbc-url: jdbc:aws-wrapper:mysql://sumair-dev-aurora-writer.chh3noeqbmvd.ap-northeast-2.rds.amazonaws.com:3306/sumair?characterEncoding=UTF-8&useSSL=false&allowMultiQueries=true

---
### prod
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    main:
      jdbc-url: jdbc:aws-wrapper:mysql://sumair-prod-aurora.cluster-xxxxx.ap-northeast-2.rds.amazonaws.com:3306/sumair?characterEncoding=UTF-8&useSSL=false&allowMultiQueries=true
```

### 2-4. 주요 HikariCP 설정 설명

| 설정 | 값 | 설명 |
|------|---|------|
| `wrapperPlugins` | `failover,efm2` | 페일오버 + 향상된 장애 모니터링 |
| `wrapperDialect` | `aurora-mysql` | Aurora MySQL 전용 dialect |
| `exception-override-class-name` | `software.amazon.jdbc.util.HikariCPSQLException` | HikariCP가 페일오버 예외를 올바르게 처리하도록 함 |
| `max-lifetime` | `840000` (14분) | Aurora 권장. 기존 5분보다 넉넉하게 |

---

## Step 3. log4jdbc 제거 + logQuery 플러그인 전환

AWS Wrapper와 log4jdbc는 둘 다 JDBC 래퍼여서 중첩이 불가능하다.
**log4jdbc를 제거하고, AWS Wrapper의 `logQuery` 플러그인으로 SQL 로깅을 대체**하는 것이 가장 깔끔한 방법이다.

### 3-1. 현재 log4jdbc 사용 현황

각 모듈의 `logback-spring.xml`에서 log4jdbc 로거를 사용 중:

| 모듈 | local 프로파일 | dev/prod 프로파일 |
|------|--------------|-----------------|
| home | `jdbc.sqlonly` DEBUG | off |
| ibe | `jdbc.sqlonly` + `jdbc.resultsettable` DEBUG | `jdbc.sqlonly` DEBUG |
| batch | `jdbc.sqlonly` + `jdbc.resultsettable` DEBUG | off |
| message | `jdbc.sqlonly` + `jdbc.resultsettable` DEBUG | off |
| pay | `jdbc.resultsettable` INFO, `jdbc.sqlonly` OFF | off |

→ 주로 **local 개발 시 SQL 확인** 용도로 사용. prod에서는 거의 꺼져있음.

### 3-2. log4jdbc.log4j2.properties 파일 삭제 (5개 모듈)

```
sumair-be-home/src/main/resources/log4jdbc.log4j2.properties
sumair-be-ibe/src/main/resources/log4jdbc.log4j2.properties
sumair-be-common/src/main/resources/log4jdbc.log4j2.properties
sumair-be-message/src/main/resources/log4jdbc.log4j2.properties
sumair-be-pay/src/main/resources/log4jdbc.log4j2.properties
```

### 3-3. pom.xml에서 log4jdbc 의존성 제거

각 모듈(6개)의 pom.xml에서 삭제:
```xml
<!-- 삭제 대상 -->
<dependency>
    <groupId>org.bgee.log4jdbc-log4j2</groupId>
    <artifactId>log4jdbc-log4j2-jdbc4</artifactId>
    <version>1.16</version>
</dependency>
```

### 3-4. logQuery 플러그인 활성화

`application-databases.yml`의 wrapperPlugins에 `logQuery` 추가:

```yaml
spring:
  datasource:
    main:
      jdbc-url: jdbc:aws-wrapper:mysql://aurora-host:3306/sumair?characterEncoding=UTF-8&useSSL=false&allowMultiQueries=true
      driver-class-name: software.amazon.jdbc.Driver
      hikari:
        data-source-properties:
          wrapperPlugins: failover,efm2,logQuery
          wrapperDialect: aurora-mysql
          # executeBatch 등 파라미터로 전달되지 않는 SQL도 캡처 (리플렉션 사용, 약간의 성능 부하)
          enhancedLogQueryEnabled: true
        exception-override-class-name: software.amazon.jdbc.util.HikariCPSQLException
```

#### logQuery 플러그인 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|---------|------|-------|------|
| `enhancedLogQueryEnabled` | boolean | `false` | `true`로 설정 시 Java Reflection으로 모든 SQL 캡처. `executeBatch()` 등도 포함. 약간의 성능 부하 있음 |

#### 로그 레벨 제어 (선택)

Wrapper 전체 로그 레벨 조정:
```yaml
hikari:
  data-source-properties:
    wrapperLoggerLevel: INFO   # OFF, SEVERE, WARNING, INFO, CONFIG, FINE, FINER, FINEST, ALL
```

### 3-5. logback-spring.xml 변경

log4jdbc의 `jdbc.sqlonly` 로거를 AWS Wrapper의 `software.amazon.jdbc` 로거로 교체한다.

#### AS-IS (현재 - log4jdbc)
```xml
<!-- log4jdbc 로거 (제거 대상) -->
<logger level="off" name="jdbc"/>
<logger level="debug" name="jdbc.sqlonly"/>
<logger level="debug" name="jdbc.resultsettable"/>
```

#### TO-BE (logQuery 플러그인)
```xml
<!-- AWS JDBC Wrapper SQL 로깅 -->
<logger level="DEBUG" name="software.amazon.jdbc"/>

<!-- Wrapper 내부 상세 로그가 너무 많으면 plugin만 제한 -->
<logger level="DEBUG" name="software.amazon.jdbc.plugin.LogQueryConnectionPlugin"/>

<!-- 기존 log4jdbc 로거 제거 또는 off -->
<logger level="off" name="jdbc"/>
```

#### 모듈별 변경 예시 (home 기준)

```xml
<!-- sumair-be-home/src/main/resources/config/log/logback-spring.xml -->
<configuration>
    <!-- ... 기존 appender 유지 ... -->

    <!-- ============ 삭제 ============ -->
    <!--
    <logger level="off" name="jdbc"/>
    -->

    <logger level="info" name="com.zaxxer.hikari"/>
    <logger level="off" name="javax.sql.DataSource"/>

    <springProfile name="local">
        <root level="info">
            <appender-ref ref="Console"/>
            <appender-ref ref="RollingFile"/>
        </root>

        <logger level="debug" name="com.sumair"/>

        <!-- ============ 변경: log4jdbc → logQuery ============ -->
        <!-- AS-IS: <logger level="debug" name="jdbc.sqlonly"/> -->
        <!-- TO-BE: -->
        <logger level="DEBUG" name="software.amazon.jdbc.plugin.LogQueryConnectionPlugin"/>
    </springProfile>

    <springProfile name="!local">
        <root level="info">
            <appender-ref ref="Console"/>
            <appender-ref ref="RollingFile"/>
        </root>

        <logger level="info" name="com.sumair"/>

        <!-- prod에서는 SQL 로깅 끄기 -->
        <logger level="OFF" name="software.amazon.jdbc.plugin.LogQueryConnectionPlugin"/>

        <springProfile name="dev">
            <logger level="debug" name="com.zaxxer.hikari.HikariConfig"/>
            <logger level="trace" name="com.zaxxer.hikari"/>
            <!-- dev에서 SQL 보고 싶으면 아래 활성화 -->
            <!-- <logger level="DEBUG" name="software.amazon.jdbc.plugin.LogQueryConnectionPlugin"/> -->
        </springProfile>
    </springProfile>
</configuration>
```

### 3-6. 전체 모듈 logback-spring.xml 변경 요약

| 모듈 | 파일 경로 | 변경 내용 |
|------|----------|----------|
| home | `config/log/logback-spring.xml` | `jdbc.sqlonly` → `software.amazon.jdbc.plugin.LogQueryConnectionPlugin` |
| ibe | `config/log/logback-spring.xml` | `jdbc.sqlonly` + `jdbc.resultsettable` → `software.amazon.jdbc.plugin.LogQueryConnectionPlugin` |
| batch | `config/log/logback-spring.xml` | `jdbc.sqlonly` + `jdbc.resultsettable` → `software.amazon.jdbc.plugin.LogQueryConnectionPlugin` |
| message | `config/log/logback-spring.xml` | `jdbc.sqlonly` + `jdbc.resultsettable` → `software.amazon.jdbc.plugin.LogQueryConnectionPlugin` |
| pay | `config/log/logback-spring.xml` | `jdbc.resultsettable` + `jdbc.sqlonly` → `software.amazon.jdbc.plugin.LogQueryConnectionPlugin` |

### 3-7. log4jdbc vs logQuery 비교

| 항목 | log4jdbc | logQuery 플러그인 |
|------|---------|-----------------|
| SQL 출력 | PreparedStatement 파라미터가 치환된 완성 SQL | 실행되는 SQL 문 로깅 |
| ResultSet 테이블 | `jdbc.resultsettable`로 테이블 형태 출력 | 미지원 (필요 시 MyBatis 로깅 보완) |
| 실행 시간 | `jdbc.sqltiming`으로 수행 시간 출력 | 미지원 (필요 시 `executionTime` 플러그인 추가) |
| 성능 부하 | 중간 | 낮음 (`enhancedLogQueryEnabled=false` 시) |
| 페일오버 호환 | 불가 (래퍼 충돌) | 완벽 호환 (같은 래퍼 내 플러그인) |

#### ResultSet 테이블 출력이 필요한 경우 (보완)

log4jdbc의 `jdbc.resultsettable` 기능이 필요하면 MyBatis 로깅으로 보완:
```xml
<!-- MyBatis가 SQL + 파라미터 + 결과 건수를 로깅 -->
<logger level="TRACE" name="com.sumair.home.airline"/>
<logger level="TRACE" name="org.apache.ibatis"/>
```

#### SQL 실행 시간이 필요한 경우 (보완)

AWS Wrapper의 `executionTime` 플러그인 추가:
```yaml
wrapperPlugins: failover,efm2,logQuery,executionTime
```

---

## Step 4. MybatisConfig.java 변경 (변경 없음)

현재 `DataSourceBuilder.create().build()` 방식으로 DataSource를 생성하고 있어서
`application.yml`의 설정만 변경하면 자동으로 HikariDataSource가 AWS Wrapper 설정을 적용한다.

**Java 코드 변경 불필요:**
- `sumair-be-home/config/MybatisConfig.java` - 변경 없음
- `sumair-be-ibe/config/MybatisConfig.java` - 변경 없음
- `sumair-be-pay/config/MybatisConfig.java` - 변경 없음
- `sumair-be-batch/config/MybatisConfiguration.java` - 변경 없음
- `sumair-be-message/config/MybatisConfig.java` - 변경 없음

---

## Step 5. Quartz 설정 변경 (batch 모듈)

Quartz는 별도 DataSource를 사용하므로 따로 변경 필요.

### 5-1. application-quartz.yml 변경

```yaml
# AS-IS
org:
  quartz:
    dataSource:
      quartzDs:
        driver: com.mysql.cj.jdbc.Driver
        URL: jdbc:mysql://호스트:3306/sumair

# TO-BE
org:
  quartz:
    dataSource:
      quartzDs:
        driver: software.amazon.jdbc.Driver
        URL: jdbc:aws-wrapper:mysql://aurora-호스트:3306/sumair
```

> Quartz는 자체 커넥션 풀(c3p0)을 사용하므로 HikariCP의 `exception-override-class-name` 같은 설정은 해당 없음.
> Quartz의 `StdJDBCDelegate`는 Aurora MySQL과 호환됨.

---

## Step 6. Java 코드 수정 (1건)

### 6-1. MySQL 전용 import 제거

```
sumair-be-home/.../AirlineCommonSclpstServiceImpl.java:3
  import com.mysql.cj.Session;  ← 제거 필요
```

이 import가 실제 사용되는지 확인 후:
- 사용하지 않으면: import 삭제
- 사용한다면: 표준 javax.sql 또는 Spring 세션으로 대체

---

## Step 7. autoReconnect 파라미터 제거

AWS Wrapper의 failover 플러그인이 자동으로 페일오버를 처리하므로
JDBC URL에서 `autoReconnect=true`를 제거한다.

```
# AS-IS
?characterEncoding=UTF-8&autoReconnect=true&useSSL=false&allowMultiQueries=true

# TO-BE
?characterEncoding=UTF-8&useSSL=false&allowMultiQueries=true
```

---

## 변경 파일 요약 체크리스트

### pom.xml (6개)
- [ ] `sumair-be-common/pom.xml` - log4jdbc 제거
- [ ] `sumair-be-home/pom.xml` - aws-wrapper 추가, log4jdbc 제거
- [ ] `sumair-be-ibe/pom.xml` - aws-wrapper 추가, log4jdbc 제거
- [ ] `sumair-be-pay/pom.xml` - aws-wrapper 추가, log4jdbc 제거
- [ ] `sumair-be-batch/pom.xml` - aws-wrapper 추가, log4jdbc 제거, mysql 버전 통일
- [ ] `sumair-be-message/pom.xml` - aws-wrapper 추가, log4jdbc 제거

### 설정 파일 (2개)
- [ ] `sumair-be-common/src/main/resources/application-databases.yml` - URL/드라이버 전환 + logQuery 플러그인 추가
- [ ] `sumair-be-batch/src/main/resources/application-quartz.yml` - URL/드라이버 전환

### logback-spring.xml (5개) - log4jdbc → logQuery 전환
- [ ] `sumair-be-home/src/main/resources/config/log/logback-spring.xml`
- [ ] `sumair-be-ibe/src/main/resources/config/log/logback-spring.xml`
- [ ] `sumair-be-batch/src/main/resources/config/log/logback-spring.xml`
- [ ] `sumair-be-message/src/main/resources/config/log/logback-spring.xml`
- [ ] `sumair-be-pay/src/main/resources/config/log/logback-spring.xml`

### 삭제 파일 (5개)
- [ ] `sumair-be-home/src/main/resources/log4jdbc.log4j2.properties`
- [ ] `sumair-be-ibe/src/main/resources/log4jdbc.log4j2.properties`
- [ ] `sumair-be-common/src/main/resources/log4jdbc.log4j2.properties`
- [ ] `sumair-be-message/src/main/resources/log4jdbc.log4j2.properties`
- [ ] `sumair-be-pay/src/main/resources/log4jdbc.log4j2.properties`

### Java 파일 (1개)
- [ ] `AirlineCommonSclpstServiceImpl.java` - `com.mysql.cj.Session` import 제거

---

## 플러그인 옵션 참고

| 플러그인                        | 코드                        | 설명                                  |
| --------------------------- | ------------------------- | ----------------------------------- |
| Failover                    | `failover`                | Aurora 페일오버 시 자동 재연결                |
| Enhanced Failure Monitoring | `efm2`                    | 빠른 장애 감지 (소켓 레벨 모니터링)               |
| Read/Write Splitting        | `readWriteSplitting`      | SELECT는 Reader, DML은 Writer로 자동 라우팅 |
| Log Query                   | `logQuery`                | SQL 쿼리 로깅 (log4jdbc 대체)             |
| Aurora Connection Tracker   | `auroraConnectionTracker` | 커넥션 추적                              |
| IAM Authentication          | `iam`                     | IAM DB 인증                           |

**기본 권장 구성:** `failover,efm2`
**SQL 로깅 포함:** `failover,efm2,logQuery`
**R/W 분리 포함:** `failover,efm2,readWriteSplitting`

---

## 참고 자료
- [AWS Advanced JDBC Wrapper GitHub](https://github.com/aws/aws-advanced-jdbc-wrapper)
- [Spring Boot HikariCP 예제](https://github.com/aws/aws-advanced-jdbc-wrapper/blob/main/examples/SpringBootHikariExample/README.md)
- [Spring Transaction Failover 예제](https://github.com/aws/aws-advanced-jdbc-wrapper/blob/main/examples/SpringTxFailoverExample/README.md)
- [AWS 블로그 - Wrapper 소개](https://aws.amazon.com/blogs/database/introducing-the-advanced-jdbc-wrapper-driver-for-amazon-aurora/)
- [Maven Central 최신 버전](https://mvnrepository.com/artifact/software.amazon.jdbc/aws-advanced-jdbc-wrapper)
