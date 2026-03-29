# Spring Modulith Kotlin 이슈 조사

## 조사 배경

Bitcoin 자동매매 프로젝트를 Java에서 Kotlin으로 전환하는 과정에서 Spring Modulith의 `@ApplicationModule(type = Type.OPEN)` 설정이 제대로 인식되지 않는 문제 발생.

현재 상황:
- `core` 모듈: `package-info.java`에 `Type.OPEN` 설정
- 멀티모듈 Gradle 프로젝트
- Kotlin으로 전환 중

## 핵심 문제

### 1. Kotlin의 package-info.java 지원 제한

**문제**: Kotlin은 Java의 `package-info.java` 파일을 제대로 지원하지 않음. Kotlin 코드에서는 이 파일이 완전히 무시됨.

**출처**:
- [Spring Modulith Issue #522](https://github.com/spring-projects/spring-modulith/issues/522)
- [Kotlin Discussions - Package-info.java not compiled](https://discuss.kotlinlang.org/t/solved-package-info-java-not-compiled-for-kotlin-packages/20335)

> "Kotlin lacks proper support for Java packages and enabling support for those is pretty cumbersome for Kotlin projects. Package-info.java files are completely ignored for Kotlin code, while they work for Java code."

### 2. Spring Modulith와 Gradle 멀티모듈

**중요 발견**: Spring Modulith는 **패키지 기반 모듈화**를 위해 설계되었으며, Gradle 멀티모듈 프로젝트와는 다른 접근 방식임.

**출처**: [Spring Modulith Discussion #151](https://github.com/spring-projects/spring-modulith/discussions/151)

> "Spring Modulith is primarily designed for package-based module organization within a single application rather than Gradle multi-module setups. Spring Modulith defines the domain structure only by package structure."

## 해결 방법

### Kotlin 프로젝트에서 @ApplicationModule 사용하기

`package-info.java` 대신 **마커 클래스**를 생성하고 `@PackageInfo` 어노테이션 사용:

```kotlin
package com.autocoin.core

import org.springframework.modulith.ApplicationModule
import org.springframework.modulith.ApplicationModule.Type
import org.springframework.modulith.PackageInfo

@ApplicationModule(type = Type.OPEN)
@PackageInfo
class ModuleMetadata
```

**핵심 포인트**:
- `@PackageInfo`: Kotlin에서 필수. Spring Modulith가 이 파일을 패키지 선언 파일로 인식하도록 함
- Java에서는 `package-info.java` 파일 타입 자체가 암묵적으로 이 역할을 하지만, Kotlin에서는 명시적으로 어노테이션 필요
- 클래스명은 관례적으로 `ModuleMetadata`, `PackageInfo`, `Module` 등을 사용
- 모듈의 **루트 패키지**에 위치해야 함

**출처**:
- [Spring Modulith with Kotlin - codecentric](https://www.codecentric.de/en/knowledge-hub/blog/modularization-the-easy-way-spring-modulith-with-kotlin-and-hexagonal-architecture)
- [Spring Modulith Issue #522](https://github.com/spring-projects/spring-modulith/issues/522)

### Type.OPEN의 의미

```kotlin
@ApplicationModule(type = Type.OPEN)
```

**효과**:
1. 다른 모듈에서 이 모듈의 internal 타입에 접근 가능 (일반적으로는 금지됨)
2. 서브패키지의 모든 타입이 명시적으로 named interface에 할당되지 않는 한, unnamed named interface에 추가됨

**사용 시나리오**:
- 레거시 코드베이스를 점진적으로 Spring Modulith 구조로 전환할 때
- 기존 의존성을 유지하면서 단계적으로 모듈화를 진행할 때

**주의사항**:
> "In a fully-modularized application, using open application modules usually hints at sub-optimal modularization and packaging structures."

완전히 모듈화된 애플리케이션에서는 `Type.OPEN` 사용이 최적이 아닌 구조를 암시함.

**출처**: [Spring Modulith fundamentals.adoc](https://github.com/spring-projects/spring-modulith/blob/main/src/docs/antora/modules/ROOT/pages/fundamentals.adoc)

### Named Interface 정의 (Kotlin)

공개 API를 명시적으로 정의하려면:

```kotlin
package com.autocoin.core.domain

import org.springframework.modulith.NamedInterface
import org.springframework.modulith.PackageInfo

@NamedInterface("domain")
@PackageInfo
class ModuleMetadata
```

**출처**: [Spring Modulith with Kotlin - codecentric](https://www.codecentric.de/en/knowledge-hub/blog/modularization-the-easy-way-spring-modulith-with-kotlin-and-hexagonal-architecture)

## 관련 GitHub Issues

1. **[Issue #522](https://github.com/spring-projects/spring-modulith/issues/522)**: Support for package info types
   - `@PackageInfo` 어노테이션 도입 배경
   - Kotlin 프로젝트 지원 개선

2. **[Issue #284](https://github.com/spring-projects/spring-modulith/issues/284)**: Support for open application modules
   - `Type.OPEN` 기능 논의
   - 점진적 모듈화를 위한 opt-in 기능

3. **[Issue #291](https://github.com/spring-projects/spring-modulith/issues/291)**: Decorators are not recognized when using Kotlin
   - Kotlin 프로젝트에서 발생하는 인식 문제

4. **[Discussion #238](https://github.com/spring-projects/spring-modulith/discussions/238)**: Any issues using it with Kotlin and Gradle?
   - Kotlin + Gradle 사용 시 일반적인 이슈 논의

5. **[Discussion #151](https://github.com/spring-projects/spring-modulith/discussions/151)**: Spring Modulith and Gradle Modules
   - Spring Modulith와 Gradle 멀티모듈의 차이점 논의

## Bitcoin 프로젝트 최종 적용 결과

### 적용한 방식: Kotlin 마커 클래스 (`@PackageInfo`)

모든 `package-info.java`를 삭제하고 각 모듈 루트 패키지에 `ModuleMetadata.kt`를 생성했다.

#### core 모듈 (공유 도메인)
```kotlin
// core/src/main/kotlin/com/autocoin/core/ModuleMetadata.kt
@ApplicationModule(type = ApplicationModule.Type.OPEN)
@PackageInfo
class ModuleMetadata
```
- `Type.OPEN`으로 모든 서브패키지(`domain`, `event`, `exception`, `port`) 노출
- `spring-modulith-api`를 `implementation`으로 `core/build.gradle.kts`에 추가 필요

#### 소비 모듈 (api-client, execution 등)
```kotlin
// api-client/src/main/kotlin/com/autocoin/apiclient/ModuleMetadata.kt
@ApplicationModule(allowedDependencies = ["core"])  // "core :: *" 아님!
@PackageInfo
class ModuleMetadata
```

### 핵심 주의사항

#### `allowedDependencies`에서 named interface 문법 주의
- `"core :: *"` — OPEN 모듈과 함께 사용하면 **실패함**. named interface가 없으므로 `*`가 아무것도 매치하지 않음
- `"core"` — OPEN 모듈은 이렇게 참조해야 정상 동작

#### `@NamedInterface`는 서브패키지를 포함하지 않음 (Spring Modulith 1.3.4 기준)
- `@NamedInterface("domain")` on `core.domain` → `core.domain.market.MarketCode`는 노출 안 됨
- 직접 패키지의 타입만 노출됨
- 서브패키지까지 전부 노출하려면 `Type.OPEN` 사용 필요

#### `core/build.gradle.kts` 의존성 추가
```kotlin
dependencyManagement {
    imports {
        mavenBom(libs.spring.modulith.bom.get().toString())
    }
}
dependencies {
    implementation(libs.spring.modulith.api)
}
```
`gradle/libs.versions.toml`에 추가:
```toml
spring-modulith-api = { module = "org.springframework.modulith:spring-modulith-api" }
```

## 참고 자료

- [Spring Modulith with Kotlin - codecentric](https://www.codecentric.de/en/knowledge-hub/blog/modularization-the-easy-way-spring-modulith-with-kotlin-and-hexagonal-architecture)
- [Spring Modulith Issue #522](https://github.com/spring-projects/spring-modulith/issues/522)
- [Spring Modulith Issue #284](https://github.com/spring-projects/spring-modulith/issues/284)
- [Spring Modulith Discussion #238](https://github.com/spring-projects/spring-modulith/discussions/238)
- [Spring Modulith Discussion #151](https://github.com/spring-projects/spring-modulith/discussions/151)
- [Introduction to Spring Modulith | Baeldung](https://www.baeldung.com/spring-modulith)
- [Spring Modulith vs Multi-Module projects - bootify.io](https://bootify.io/multi-module/spring-modulith-vs-multi-module.html)
