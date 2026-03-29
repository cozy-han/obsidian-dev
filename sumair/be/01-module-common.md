# sumair-be-common

공통 라이브러리 모듈. 모든 서비스 모듈이 의존한다.

## 기본 정보
- GroupId: `com.sumair`
- ArtifactId: `sumair-common`
- Version: `0.0.1`
- Java 파일: 526개

## 패키지 구조

```
com.sumair.common
├── core
│   ├── annotaion/       # 커스텀 어노테이션
│   ├── exception/       # 예외 처리 (BaseException, ErrorCode, ErrorHandler)
│   ├── http/webclient/  # WebClientService, InnerWebClientService
│   ├── message/         # 메시지 로컬라이제이션 (i18n)
│   ├── properties/      # 설정 프로퍼티 관리
│   ├── rest/            # REST 클라이언트 설정
│   ├── security/
│   │   ├── crypto/      # AES 암복호화
│   │   ├── jasypt/      # Jasypt 프로퍼티 암호화
│   │   ├── masking/     # 데이터 마스킹 (PII)
│   │   └── secur/       # 보안 핵심 구현
│   └── seq/             # 시퀀스 ID 생성
└── (mapper 98개)        # 공통 엔티티 매퍼
```

## 주요 의존성

| 라이브러리                             | 용도           |
| --------------------------------- | ------------ |
| spring-boot-starter-web           | 웹            |
| spring-boot-starter-aop           | AOP          |
| spring-boot-starter-webflux       | WebClient    |
| spring-boot-starter-validation    | 유효성 검증       |
| spring-boot-starter-jdbc          | JDBC         |
| jasypt-spring-boot-starter 3.0.5  | 프로퍼티 암호화     |
| mybatis 3.5.0                     | ORM          |
| springfox 3.0.0                   | Swagger      |
| commons-lang3 / commons-beanutils | Apache 유틸    |
| httpclient / commons-httpclient   | HTTP 클라이언트   |
| jsch 0.1.53                       | SSH/SFTP     |
| xstream 1.4.9                     | XML 직렬화      |
| itext 2.1.7                       | PDF 생성       |
| poi 3.15                          | Excel 처리     |
| zxing 3.1.0                       | QR코드 생성      |
| slack-api-client 1.30.0           | Slack 알림     |
| jbcrypt 0.4                       | 비밀번호 해싱      |
| jaxb-api / jaxws-api              | XML/SOAP 바인딩 |
| modelmapper 3.1.1                 | 객체 매핑        |

## 설정 파일
- `application-common.yml` - 공통 설정 (crypto, jasypt, jwt, 서비스 URL)
- `application-databases.yml` - DB 설정
- `application-ibe.yml` - IBE 전용 설정
- `application-init.yml` - 초기화 설정

## 주요 매퍼 (98개)
공통 엔티티에 대한 MyBatis 매퍼 제공:
- `BaseOrdPymDtlTxnMapper` - 주문 결제 상세
- `BasePtyCrtfcTxnMapper` - 인증 트랜잭션
- `BaseCnsNoticeBasMapper` - 공지사항
- `BaseOrdBookerBasMapper` - 예약자 정보
- `PrdFareClassCdMapper` - 운임 클래스 코드
- 등 93개 추가 매퍼

## 핵심 패턴
- MyBatis XML 기반 매퍼 (Java 코드 내 XML 리소스 포함)
- AOP 어노테이션 기반 암복호화 미들웨어
- PII 데이터 마스킹 처리
- 중앙 예외 처리 (에러 코드 + 레벨 체계)
- Jasypt로 dev/prod 민감 설정 암호화
