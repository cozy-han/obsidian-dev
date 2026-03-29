# MySQL to Aurora 마이그레이션 검토

## 전제 조건
- 현재: AWS RDS MySQL 8.0
- 목표: AWS Aurora MySQL (MySQL 호환 에디션)
- Aurora MySQL은 MySQL 5.7/8.0과 높은 호환성을 가지지만 100%는 아님

---

## 1. 설정/인프라 변경 사항

### 1-1. JDBC URL 변경 (필수)

**파일:** `sumair-be-common/src/main/resources/application-databases.yml`

모든 환경(common/local/dev/prod)의 JDBC URL을 Aurora 엔드포인트로 변경해야 한다.

```yaml
# AS-IS
jdbc-url: jdbc:log4jdbc:mysql://palnet-dev.cexpliz30rwl.ap-northeast-2.rds.amazonaws.com:3306/sumair?...

# TO-BE (Aurora 클러스터 엔드포인트)
jdbc-url: jdbc:log4jdbc:mysql://sumair-aurora-cluster.cluster-xxxxx.ap-northeast-2.rds.amazonaws.com:3306/sumair?...
```

변경 대상 데이터소스:
| 데이터소스 | 환경 |
|-----------|------|
| main | common, local, dev, prod |
| message | common, local, dev, prod |

> Aurora는 **Writer 엔드포인트**와 **Reader 엔드포인트**가 분리됨. 읽기 부하 분산이 필요하면 Reader 엔드포인트 활용 검토.

### 1-2. JDBC 연결 파라미터 검토

현재 사용 중인 MySQL 전용 파라미터:

| 파라미터 | 현재 값 | Aurora 호환 | 비고 |
|---------|--------|------------|------|
| `useSSL=false` | false | O | Aurora는 SSL 기본 지원. 프로덕션에서는 `useSSL=true` 권장 |
| `characterEncoding=UTF-8` | UTF-8 | O | 그대로 사용 |
| `autoReconnect=true` | true | △ | Aurora 페일오버 시 동작 불확실. `failoverClusterInstanceEndpoint` 사용 권장 |
| `allowMultiQueries=true` | true | O | 호환 |
| `serverTimezone=Asia/Seoul` | Asia/Seoul | O | Aurora 파라미터 그룹에서도 설정 필요 |
| `allowPublicKeyRetrieval=true` | true | O | 호환 |

**주의: `autoReconnect=true`**
- Aurora 페일오버 시 커넥션이 끊어지면 autoReconnect만으로는 불충분
- **MariaDB Connector/J** 또는 **AWS JDBC Driver** 사용 시 페일오버 자동 감지 가능
- 대안: `jdbc:mysql:aws://` (AWS JDBC Driver for MySQL)

### 1-3. MySQL JDBC 드라이버

현재 `mysql-connector-java` 사용 중. Aurora에서도 동작하지만 AWS 전용 드라이버가 더 적합하다.

| 모듈 | 현재 버전 |
|------|----------|
| home, ibe, pay, message | 8.0.32 |
| batch | 8.0.29 |

**권장:** `software.amazon.jdbc:aws-advanced-jdbc-wrapper` (AWS JDBC 드라이버)
- Aurora 페일오버 자동 감지
- 클러스터 토폴로지 인식
- Read/Write 분리 지원

### 1-4. log4jdbc 래퍼

```properties
# log4jdbc.log4j2.properties
log4jdbc.drivers=com.mysql.cj.jdbc.Driver
```
- Aurora MySQL 사용 시 그대로 호환
- AWS JDBC 드라이버 전환 시 드라이버 클래스명 변경 필요

### 1-5. Java 코드 내 MySQL 직접 참조

```
sumair-be-home/.../AirlineCommonSclpstServiceImpl.java
  → import com.mysql.cj.Session;
```
- MySQL 드라이버 전용 클래스 직접 사용. AWS JDBC 드라이버로 전환 시 **컴파일 에러 발생**.

---

## 2. Mapper XML - MySQL 전용 SQL 함수 검토

### 2-1. NOW() - 167건 (호환되지만 주의)

Aurora MySQL에서 `NOW()`는 호환되지만 **타임존 처리**가 다를 수 있다.

| 모듈 | 주요 파일 | 건수 |
|------|----------|------|
| home | MypageReservationMapper.xml | 35+ |
| home | BaseUserMapper.xml | 8+ |
| batch | CustomerMapper.xml | 15+ |
| 기타 | 다수 파일 | 109+ |

**조치:** Aurora 파라미터 그룹에서 `time_zone = Asia/Seoul` 설정 확인 필수

### 2-2. DATE_FORMAT() - 40건+ (호환)

Aurora MySQL에서 완전 호환. 추가 조치 불필요.

주요 사용 파일:
- `MypageReservationMapper.xml` (14건)
- `CustomerMapper.xml` (5건)
- `BaseComSeqMapper.xml` (3건)
- `WebcheckinMapper.xml` (4건)

### 2-3. IFNULL() - 28건 (호환)

Aurora MySQL에서 `IFNULL()` 완전 호환. 그대로 사용 가능.

| 모듈 | 파일 | 건수 |
|------|------|------|
| home | MypageReservationMapper.xml | 14 |
| batch | WeatherMapper.xml | 9 |
| pay | PointMapper.xml, BookingPaymentMapper.xml, CouponMapper.xml | 4 |
| ibe | OrdBasMapper.xml, OrdBookBasMapper.xml | 2 |

> 참고: 표준 SQL인 `COALESCE()`로 대체하면 DB 독립성이 높아지지만 필수는 아님

### 2-4. GROUP_CONCAT() - 27건 (주의 필요)

Aurora MySQL에서 호환되지만 **`group_concat_max_len` 기본값(1024)**에 의해 결과가 잘릴 수 있다.

주요 사용:
| 파일 | 건수 | 용도 |
|------|------|------|
| CustomerCommonMapper.xml | 10 | 고객 정보 집계 |
| AuthUserMapper.xml | 8 | 사용자 권한 집계 |
| BaseUserMapper.xml | 2 | 사용자 정보 |
| MypageReservationMapper.xml | 3 | 예약 정보 |
| AirlineScheduleMapper.xml | 1 | 스케줄 |
| BasePrdRevenueReportTxnMapper.xml | 2 | 매출 리포트 |
| MainHashTagMapper.xml | 1 | 해시태그 |

**조치:** Aurora 파라미터 그룹에서 `group_concat_max_len` 값을 기존 MySQL과 동일하게 설정

### 2-5. ON DUPLICATE KEY UPDATE - 1건 (주의 필요)

```
sumair-be-batch/.../WeatherMapper.xml:57
```

Aurora MySQL에서 호환되지만, Aurora의 스토리지 엔진 차이로 **데드락 빈도가 다를 수 있음**. 배치 작업에서 대량 UPSERT 시 성능 테스트 필요.

### 2-6. @@TIME_ZONE 시스템 변수 - 1건 (확인 필요)

```
sumair-be-batch/.../AdminMapper.xml:116
```

Quartz 크론 트리거 생성 시 `@@TIME_ZONE` 사용. Aurora에서도 호환되지만 파라미터 그룹의 타임존 설정에 따라 값이 달라질 수 있음.

**조치:** Aurora 파라미터 그룹에서 `time_zone` 값이 기존 MySQL과 동일한지 확인

### 2-7. UNIX_TIMESTAMP() - 1건 (호환)

```
sumair-be-batch/.../AdminMapper.xml:98
```

Aurora MySQL에서 호환. 타임존 설정만 맞으면 문제 없음.

### 2-8. STR_TO_DATE() - 5건 (호환)

| 파일 | 건수 |
|------|------|
| MainWeatherMapper.xml | 1 |
| CustomerUpdateMapper.xml | 1 |
| AirlineCheckinSeatMapServiceMapper.xml | 2 |
| ReservationMapper.xml | 1 |

Aurora MySQL에서 완전 호환.

### 2-9. SUBSTRING_INDEX() - 1건 (호환)

```
MypageReservationMapper.xml:869
```

Aurora MySQL에서 호환.

---

## 3. Aurora 고유 이슈

### 3-1. 페일오버 처리

Aurora는 Multi-AZ 구성 시 자동 페일오버를 수행하는데, 이때:
- **기존 DB 커넥션이 모두 끊어짐**
- HikariCP가 재연결하지만 **진행 중인 트랜잭션은 실패**
- `autoReconnect=true`만으로는 불충분

**권장 대응:**
1. AWS JDBC Driver 사용 (페일오버 감지 내장)
2. 또는 애플리케이션 레벨 재시도 로직 추가
3. HikariCP `connectionTestQuery` 설정

### 3-2. Read/Write 엔드포인트 분리

Aurora는 Writer/Reader 엔드포인트가 분리됨:
- **Writer:** 쓰기 전용 (INSERT/UPDATE/DELETE)
- **Reader:** 읽기 전용 (SELECT), 최대 15개 리플리카

현재 단일 DataSource 구조이므로 즉시 분리할 필요는 없지만, 읽기 부하가 많은 경우(home 모듈의 항공편 조회 등) Reader 엔드포인트 활용을 고려할 수 있음.

### 3-3. 커넥션 풀 설정 조정

Aurora는 MySQL보다 커넥션 처리 비용이 다름:

| 설정 | 현재 값 | Aurora 권장 |
|------|--------|------------|
| maximum-pool-size (prod) | 20 | 동일 또는 증가 가능 (Aurora는 더 많은 커넥션 처리 가능) |
| max-lifetime | 300000 (5분) | 유지 |
| idle-timeout | 60000 (1분) | 유지 |

### 3-4. Quartz + c3p0 (batch 모듈)

batch 모듈에서 Quartz가 c3p0 커넥션 풀을 별도로 사용. Aurora 엔드포인트로 변경 시 c3p0 설정도 함께 변경 필요.

**파일:** `application-quartz.yml`

---

## 4. 변경 작업 체크리스트

### 필수 (마이그레이션 전)
- [ ] Aurora 파라미터 그룹에서 `time_zone = Asia/Seoul` 설정
- [ ] Aurora 파라미터 그룹에서 `group_concat_max_len` 기존 값과 동일하게 설정
- [ ] `application-databases.yml` - 모든 환경의 JDBC URL을 Aurora 엔드포인트로 변경
- [ ] `application-quartz.yml` - Quartz DB URL도 Aurora로 변경
- [ ] message 데이터소스 URL도 Aurora로 변경 (별도 인스턴스인 경우 확인)

### 권장 (안정성 향상)
- [ ] `autoReconnect=true` 대신 AWS JDBC Driver 도입 검토
- [ ] `com.mysql.cj.Session` 직접 import 제거 (AirlineCommonSclpstServiceImpl.java)
- [ ] mysql-connector-java 버전 통일 (batch만 8.0.29, 나머지 8.0.32)
- [ ] SSL 활성화 검토 (`useSSL=true` + Aurora CA 인증서)

### 선택 (성능 최적화)
- [ ] Reader 엔드포인트 분리 (읽기 전용 쿼리 라우팅)
- [ ] `IFNULL()` → `COALESCE()` 전환 (DB 독립성)
- [ ] ON DUPLICATE KEY UPDATE 사용 부분 성능 테스트 (WeatherMapper.xml)

---

## 5. 리스크 평가 요약

| 항목 | 리스크 | 사유 |
|------|--------|------|
| Mapper XML SQL 호환성 | **낮음** | Aurora MySQL은 MySQL 함수 대부분 호환 |
| 타임존 | **중간** | NOW() 167건, @@TIME_ZONE 1건 - 파라미터 그룹 설정으로 해결 |
| GROUP_CONCAT 잘림 | **중간** | 27건 - 파라미터 그룹 설정으로 해결 |
| 페일오버 처리 | **높음** | 현재 autoReconnect만 의존, 트랜잭션 유실 가능 |
| 커넥션 드라이버 | **낮음** | mysql-connector-java 그대로 사용 가능 |
| Java 코드 호환성 | **낮음** | com.mysql.cj.Session import 1건만 확인 |

**결론:** Mapper XML 기준으로는 MySQL 전용 함수가 모두 Aurora MySQL 호환이므로 **SQL 수정 없이 마이그레이션 가능**. 주요 작업은 **JDBC URL 변경 + 파라미터 그룹 설정 + 페일오버 대비**이다.
