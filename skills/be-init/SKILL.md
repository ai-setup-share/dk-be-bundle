---
name: be-init
description: Spring Boot 3.2.1 + Java 21 + Hexagonal Architecture 8-모듈 백엔드 인프라를 스캐폴드한다. 도메인 코드는 만들지 않고(모든 도메인은 be-planner / add-* 스킬이 담당) 빌드·프로파일·감사·예외·테스트 기반만 세팅한다. 생성 직후 `./gradlew :modules:applications:api-application:compileJava :modules:applications:api-application:compileTestJava :modules:repository-jdbc:compileTestJava --no-daemon` 로 검증하고, 실패 시 원인 분류표 기반으로 최대 3회 자가 수정한다. 재실행(settings.gradle.kts 존재) 시 즉시 abort.
---

# be-init — 백엔드 인프라 스캐폴더

## 1. 스킬 경계 (만드는 것 / 만들지 않는 것)

만드는 것:
- 8개 Gradle 모듈 (api-application, api, service, repository-jdbc, infrastructure, model, schema, exception) + settings.gradle.kts + 루트 build.gradle.kts (모두 Kotlin DSL)
- Gradle wrapper 8.5
- `Application.java` + `@EnableJdbcAuditing`
- Auditing 컨트랙트: `AuditFields` 인터페이스 (abstract BaseEntity 아님 — 이유는 §4.6)
- 프로파일: main yml(active=local), application-local.yml(H2 file), application-test.yml(H2 mem, api-application · repository-jdbc 양쪽에 배치)
- 예외 계층: `DomainException`(abstract) + `HttpException`(interface) + 구체 4종 (`EntityNotFoundException` abstract / `ConflictException` / `ForbiddenException` / `InvalidInputException`), `GlobalExceptionHandler`, `ErrorResponse`
- `IntegrationTestBase` (repository-jdbc, JDBC 슬라이스 보존)
- schema.sql (주석만)
- springdoc UI
- 각 모듈 패키지 디렉터리 + `.gitkeep`
- .gitignore, README.md(최소)

만들지 않는 것:
- 도메인 코드, 엔티티, 리포지토리, 유스케이스, 컨트롤러 → be-planner / add-* 담당
- Auth, OAuth, 세션
- Elasticsearch, Redis, Kafka
- Dockerfile, CI 설정
- 샘플 DDL (schema.sql은 주석 1줄만)

## 2. 입력 파라미터

`${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차를 따른다. `{projectRoot}/dk-be-bundle.config.json`이 있으면 그것을 읽고, 없으면 사용자에게 묻는다.

| key | 예시 | 용도 |
| --- | --- | --- |
| `projectName` | `share-ai-setup-be` | rootProject.name, H2 파일명 |
| `group` | `share.ai.setup` | Gradle group |
| `packageRoot` | `share.ai.setup` | Java 루트 패키지 (group과 달라도 됨) |
| `projectRoot` | `/Users/.../share-ai-setup-be` | 생성 위치 |

파생값: `packagePath = packageRoot.replace('.', '/')`.
**사전 불변식 검사**: `packagePath == packageRoot.replace('.','/')` 가 성립해야 파일 생성 시작. 깨지면 abort.

**재실행 가드**: `{projectRoot}/settings.gradle.kts` 가 이미 존재하면 즉시 abort (도메인 덮어쓰기 방지).

## 3. 모듈 구조 & 의존성 그래프

```
api-application → api, service, repository-jdbc, infrastructure, schema, exception
api             → model, service, exception
service         → model, exception, infrastructure
repository-jdbc → model, exception, infrastructure
infrastructure  → model, exception
model           → (pure)
exception       → (pure)
schema          → (resource-only)
```

핵심 판단:
- **infrastructure 는 port 모듈** — Repository 인터페이스가 여기에 산다.
- `service → infrastructure`: Reader/Writer 가 `*Repository` port 에 의존. 구현체는 모름.
- `repository-jdbc → infrastructure`: JdbcRepository 가 `*Repository` port 를 구현.
- `api-application` 이 service(포트 사용자) + repository-jdbc(어댑터) 를 함께 로드해 Spring DI 로 연결. 전형적 port-adapter 패턴.
- `api-application → schema`: schema.sql 을 런타임·테스트 classpath 에 보장.
- HTTP 클라이언트·외부 IO 어댑터는 별도 모듈(예: `http-adapter`)을 추가할 수 있으나 be-init 범위 밖.

## 4. 파일 스펙 (모두 구체적 내용 포함)

### 4.1 `settings.gradle.kts`

```kotlin
rootProject.name = "{projectName}"

include(
    ":modules:applications:api-application",
    ":modules:api",
    ":modules:service",
    ":modules:repository-jdbc",
    ":modules:infrastructure",
    ":modules:model",
    ":modules:schema",
    ":modules:exception",
)
```

### 4.2 루트 `build.gradle.kts`

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.2.1" apply false
    id("io.spring.dependency-management") version "1.1.4" apply false
}

allprojects {
    group = "{group}"
    version = "0.0.1-SNAPSHOT"
    repositories { mavenCentral() }
}

subprojects {
    apply(plugin = "java")
    apply(plugin = "io.spring.dependency-management")

    java {
        toolchain { languageVersion.set(JavaLanguageVersion.of(21)) }
    }

    the<io.spring.gradle.dependencymanagement.dsl.DependencyManagementExtension>().apply {
        imports {
            mavenBom("org.springframework.boot:spring-boot-dependencies:3.2.1")
            mavenBom("org.junit:junit-bom:5.14.0")
            mavenBom("org.mockito:mockito-bom:5.20.0")
        }
    }

    dependencies {
        "compileOnly"("org.projectlombok:lombok")
        "annotationProcessor"("org.projectlombok:lombok")
        "testCompileOnly"("org.projectlombok:lombok")
        "testAnnotationProcessor"("org.projectlombok:lombok")

        "testImplementation"("org.springframework.boot:spring-boot-starter-test") {
            exclude(group = "org.junit.vintage", module = "junit-vintage-engine")
        }
        "testImplementation"("org.assertj:assertj-core:3.26.3")
    }

    tasks.withType<Test>().configureEach { useJUnitPlatform() }

    tasks.withType<JavaCompile>().configureEach {
        options.encoding = "UTF-8"
        options.compilerArgs.add("-parameters")
    }
}
```

설계 근거:
- `apply false`로 루트에는 BootPlugin과 dependency-management를 걸지 않는다 → 루트에서 `bootJar`가 잘못 활성화되지 않고, dep-management가 전 서브프로젝트에 자동 전파되어 통제 상실되는 위험을 피한다.
- Lombok은 전 모듈 공통 → 루트 `subprojects` 블록에서 일괄 주입. 버전은 BOM이 관리.
- JUnit/Mockito BOM은 test 구성 전용이지만 BOM import는 compile 스코프에 영향을 주지 않으므로 루트에서 한 번만 선언.
- `-parameters` 필수: Spring Data JDBC 3.x 는 Entity 생성자 파라미터명을 리플렉션으로 매핑한다. Lombok `@AllArgsConstructor` 가 만든 ctor 의 파라미터명은 `-parameters` 없으면 `arg0, arg1...` 로 strip 되어 런타임 매핑이 실패한다.

### 4.3 `modules/applications/api-application/build.gradle.kts`

`BootJar` 타입 import를 피하기 위해 `application` 플러그인 + `mainClass` 로 메인 클래스를 지정한다 (import 누락 시 빌드 스크립트 자체가 실패해 verify 루프가 무력화되는 문제 회피).

```kotlin
plugins {
    id("org.springframework.boot")
    application
}

dependencies {
    implementation(project(":modules:api"))
    implementation(project(":modules:service"))
    implementation(project(":modules:repository-jdbc"))
    implementation(project(":modules:infrastructure"))
    implementation(project(":modules:schema"))
    implementation(project(":modules:exception"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jdbc")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")

    runtimeOnly("com.h2database:h2")
}

application {
    mainClass.set("{packageRoot}.Application")
}

tasks.named<Jar>("jar") { enabled = false }
// bootJar 은 boot 플러그인 기본 구성으로 활성화됨 — 명시적 타입 참조 불필요
```

### 4.4 나머지 모듈 build.gradle.kts

**`api(project(...))` vs `implementation(project(...))`**: 모듈의 **public API 에 노출되는 타입** (인터페이스 인자/반환, 예외 타입) 을 포함한 모듈은 `api(...)` 로 선언해 consumer 에 전이 노출한다. consumer 가 직접 declare 하지 않아도 타입 import 가능. 내부 구현에만 쓰이면 `implementation(...)`.

`modules/api`:
```kotlin
dependencies {
    api(project(":modules:model"))
    api(project(":modules:service"))
    api(project(":modules:exception"))

    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.3.0")
}
```

`modules/service`:
```kotlin
dependencies {
    api(project(":modules:model"))
    api(project(":modules:exception"))
    api(project(":modules:infrastructure"))

    implementation("org.springframework.boot:spring-boot-starter")
    implementation("org.springframework:spring-tx")

    // IntegrationConfig 가 EnableJdbcAuditing/EnableJdbcRepositories 를 import,
    // 통합 테스트에서 JDBC 어댑터·H2 가 실제로 필요하므로 선제 포함.
    testImplementation("org.springframework.boot:spring-boot-starter-data-jdbc")
    testImplementation("com.h2database:h2")
    testImplementation(project(":modules:repository-jdbc"))
    testImplementation(project(":modules:schema"))
}
```

`modules/repository-jdbc`:
```kotlin
dependencies {
    api(project(":modules:model"))
    api(project(":modules:exception"))
    api(project(":modules:infrastructure"))

    implementation("org.springframework.boot:spring-boot-starter-data-jdbc")

    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.boot:spring-boot-starter-data-jdbc")
    testImplementation(project(":modules:schema"))
    testRuntimeOnly("com.h2database:h2")
}
```

`modules/infrastructure`:
```kotlin
dependencies {
    api(project(":modules:model"))
    api(project(":modules:exception"))
    implementation("org.springframework.boot:spring-boot-starter")
}
```

`modules/model`, `modules/exception`, `modules/schema`: 빈 `dependencies { }` (Lombok 등은 루트 subprojects 블록에서 주입).

### 4.5 `Application.java`

경로: `modules/applications/api-application/src/main/java/{packagePath}/Application.java`

```java
package {packageRoot};

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jdbc.repository.config.EnableJdbcAuditing;

@SpringBootApplication
@EnableJdbcAuditing
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### 4.6 Auditing — `AuditFields` 인터페이스 (BaseEntity abstract class 아님)

**기술적 근거**: Spring Data JDBC 는 JPA 의 `@MappedSuperclass` 를 지원하지 않는다. abstract BaseEntity 에 `@CreatedDate / @LastModifiedDate` 를 얹어도 자식 엔티티에 필드가 상속되지 않거나 매핑이 실패해 timestamp 가 조용히 null 로 남는다. 따라서 공통 상속 대신 **인터페이스 컨트랙트**를 제공하고, 각 엔티티는 본인 필드에 직접 `@CreatedDate / @LastModifiedDate` 를 선언한다.

경로: `modules/model/src/main/java/{packagePath}/model/audit/AuditFields.java`

```java
package {packageRoot}.model.audit;

import java.time.Instant;

public interface AuditFields {
    Instant getCreatedAt();
    Instant getUpdatedAt();
}
```

다운스트림 사용 예 (예시만, 생성하지 않음):
```java
public class Article implements AuditFields {
    @CreatedDate      private Instant createdAt;
    @LastModifiedDate private Instant updatedAt;
    // ...
}
```

### 4.7 예외 계층 (2단 구조) + 핸들러

**설계**: `DomainException` (abstract, detail 보유) + `HttpException` (interface, 상태/메시지 노출) 2층 구조. 구체 예외는 둘 다 상속/구현.

`modules/exception/src/main/java/{packagePath}/exception/DomainException.java`
```java
package {packageRoot}.exception;

public abstract class DomainException extends RuntimeException {

    private final String detail;

    protected DomainException(String detail) {
        super(detail);
        this.detail = detail;
    }

    public String getDetail() {
        return detail;
    }
}
```

`modules/exception/src/main/java/{packagePath}/exception/HttpException.java`
```java
package {packageRoot}.exception;

public interface HttpException {

    int httpStatusCode();

    String httpErrorMessage();
}
```

**구체 예외 4종** (모두 `extends DomainException implements HttpException`, 기본 생성자 + 오버라이드 가능한 3-arg 생성자 제공):

- `EntityNotFoundException` (abstract): 기본 404 / "해당 리소스를 찾을 수 없습니다"
- `ConflictException`: 기본 409 / "이미 존재하는 리소스입니다"
- `ForbiddenException`: 기본 403 / "접근 권한이 없습니다"
- `InvalidInputException`: 기본 400 / "잘못된 요청입니다"

`EntityNotFoundException` 은 도메인별 하위 클래스를 만들도록 **abstract** (e.g. `ArticleNotFoundException extends EntityNotFoundException`). 나머지 3종은 그대로 사용 가능.

예시:
```java
package {packageRoot}.exception;

public abstract class EntityNotFoundException extends DomainException implements HttpException {

    private final int statusCode;
    private final String clientMessage;

    protected EntityNotFoundException(String detail) {
        this(detail, 404, "해당 리소스를 찾을 수 없습니다");
    }

    protected EntityNotFoundException(String detail, int statusCode, String clientMessage) {
        super(detail);
        this.statusCode = statusCode;
        this.clientMessage = clientMessage;
    }

    @Override public int httpStatusCode() { return statusCode; }
    @Override public String httpErrorMessage() { return clientMessage; }
}
```

`ConflictException` / `ForbiddenException` / `InvalidInputException` 은 동일 패턴의 구체 클래스 (abstract 아님, 위 EntityNotFoundException 을 참고).

**ErrorResponse** (api-application 모듈):

`modules/applications/api-application/src/main/java/{packagePath}/application/config/ErrorResponse.java`
```java
package {packageRoot}.application.config;

public record ErrorResponse(
        int status,
        String error,
        String message
) {}
```

**GlobalExceptionHandler** (api-application 모듈):

`modules/applications/api-application/src/main/java/{packagePath}/application/config/GlobalExceptionHandler.java`
```java
package {packageRoot}.application.config;

import {packageRoot}.exception.DomainException;
import {packageRoot}.exception.HttpException;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        log.info("Validation failed: {}", e.getMessage());
        String message = e.getBindingResult().getFieldErrors().stream()
            .map(err -> err.getField() + ": " + err.getDefaultMessage())
            .collect(java.util.stream.Collectors.joining(", "));
        return ResponseEntity.badRequest().body(new ErrorResponse(400, "ValidationFailed", message));
    }

    @ExceptionHandler(DomainException.class)
    public ResponseEntity<ErrorResponse> handleDomain(DomainException e) {
        if (e instanceof HttpException httpEx) {
            int status = httpEx.httpStatusCode();
            log.warn("Domain error [{}] - {}: {}", status, e.getClass().getSimpleName(), e.getMessage());
            return ResponseEntity.status(status).body(
                new ErrorResponse(status, e.getClass().getSimpleName(), httpEx.httpErrorMessage())
            );
        }
        log.error("Unhandled domain error - {}: {}", e.getClass().getSimpleName(), e.getMessage(), e);
        return ResponseEntity.internalServerError().body(
            new ErrorResponse(500, e.getClass().getSimpleName(), "Internal server error")
        );
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception e) {
        log.error("Unexpected error: {}", e.getMessage(), e);
        return ResponseEntity.internalServerError().body(
            new ErrorResponse(500, "InternalServerError", "Internal server error")
        );
    }
}
```

**배치 이유**: `GlobalExceptionHandler` 와 `ErrorResponse` 는 `DomainException` + `HttpException` 두 심볼을 모두 알아야 한다. `exception` 모듈에 두면 `api` 모듈이 Spring Web 을 추가 의존해야 해서 관심사가 섞인다. `api-application` 에 두면 조립 경계에서 예외↔HTTP 매핑을 하나로 관리할 수 있다.

### 4.8 프로파일 YAML

`modules/applications/api-application/src/main/resources/application.yml`
```yaml
spring:
  profiles:
    active: local
  application:
    name: {projectName}
  data:
    jdbc:
      repositories:
        enabled: true
springdoc:
  swagger-ui:
    path: /swagger-ui.html
```

`application-local.yml` (H2 file)
```yaml
spring:
  datasource:
    url: jdbc:h2:file:./data/{projectName};AUTO_SERVER=TRUE;MODE=MySQL;DB_CLOSE_ON_EXIT=FALSE;CASE_INSENSITIVE_IDENTIFIERS=TRUE;DATABASE_TO_LOWER=TRUE
    driver-class-name: org.h2.Driver
    username: sa
    password:
  sql:
    init:
      mode: always
      schema-locations: classpath:local_h2/schema.sql
  h2:
    console:
      enabled: true
      path: /h2-console
```

`application-test.yml` (H2 mem — api-application·repository-jdbc 양쪽 `src/test/resources/` 에 동일 배치)
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:test;MODE=MySQL;DB_CLOSE_DELAY=-1;CASE_INSENSITIVE_IDENTIFIERS=TRUE;DATABASE_TO_LOWER=TRUE
    driver-class-name: org.h2.Driver
    username: sa
    password:
  sql:
    init:
      mode: always
      schema-locations: classpath:local_h2/schema.sql
  data:
    jdbc:
      repositories:
        enabled: true
```

**H2 옵션 근거**:
- `MODE=MySQL`: 추후 MySQL 전환 시 SQL 호환성 확보.
- `AUTO_SERVER=TRUE` (local only): 앱 + H2 console + IDE 가 같은 파일에 동시 접속 가능.
- `DB_CLOSE_ON_EXIT=FALSE` (local only): Spring shutdown 시 H2가 파일을 닫지 않도록 해 재기동 간 일관성 확보.
- `DB_CLOSE_DELAY=-1` (test only): JVM 동안 mem DB 유지.
- `CASE_INSENSITIVE_IDENTIFIERS=TRUE` + `DATABASE_TO_LOWER=TRUE`: Spring Data JDBC 3.x 가 식별자를 쌍따옴표로 감싸 보내는데, H2 기본 모드는 unquoted 를 대문자 저장 / quoted 는 원본 유지 → 미일치로 `Table not found` 발생. 두 플래그로 모든 식별자를 소문자로 맞춤.
- `./data/{projectName}`: `.gitignore`에 `data/` 추가하여 DB 파일이 커밋되지 않게 함.
- `spring.data.jdbc.repositories.enabled: true`: 프로젝트에 JPA starter 가 혼재해도 JDBC repository 스캔을 확정.

### 4.9 테스트 인프라 (repository-jdbc 슬라이스 + service 통합)

be-init 은 두 모듈에 테스트 베이스를 각각 제공한다.

#### 4.9.1 `repository-jdbc` — JDBC 슬라이스

경로: `modules/repository-jdbc/src/test/java/{packagePath}/jdbc/TestApplication.java`
```java
package {packageRoot}.jdbc;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jdbc.repository.config.EnableJdbcAuditing;

@SpringBootApplication
@EnableJdbcAuditing
public class TestApplication {}
```

경로: `modules/repository-jdbc/src/test/java/{packagePath}/jdbc/IntegrationTestBase.java`
```java
package {packageRoot}.jdbc;

import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase.Replace;
import org.springframework.boot.test.autoconfigure.data.jdbc.DataJdbcTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.transaction.annotation.Transactional;

@DataJdbcTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
@ActiveProfiles("test")
@Transactional
public abstract class IntegrationTestBase {}
```

**근거**: `@DataJdbcTest` 슬라이스 + `TestApplication` (classpath 루트 Spring Boot 앱) + `@EnableJdbcAuditing` (@CreatedDate/@LastModifiedDate 활성화). 하위 테스트는 필요한 `*JdbcRepository` bean 을 `@ComponentScan(basePackages=...)` 또는 `@Import(...)` 로 로드 (be-test-repository-jdbc 참조).
`@AutoConfigureTestDatabase(Replace.NONE)`: 슬라이스가 임의의 임베디드 DB를 선택하지 않고 `application-test.yml` 의 H2 mem 설정을 그대로 사용하도록.

#### 4.9.2 `service` — 통합 테스트

Reader/Writer/Usecase 는 JDBC 어댑터가 실제 H2 에 붙어야 검증 가능한 경우(동시성·순서·트랜잭션) 가 있어 `@SpringBootTest` 기반 베이스를 따로 제공한다.

경로: `modules/service/src/test/java/{packagePath}/service/IntegrationConfig.java`
```java
package {packageRoot}.service;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.data.jdbc.repository.config.EnableJdbcAuditing;
import org.springframework.data.jdbc.repository.config.EnableJdbcRepositories;

@SpringBootApplication
@EnableJdbcAuditing
@ComponentScan(basePackages = {
    "{packageRoot}.service",
    "{packageRoot}.jdbc"
})
@EnableJdbcRepositories(basePackages = "{packageRoot}.jdbc")
class IntegrationConfig {}
```

경로: `modules/service/src/test/java/{packagePath}/service/IntegrationTestBase.java`
```java
package {packageRoot}.service;

import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.transaction.annotation.Transactional;

@SpringBootTest(classes = IntegrationConfig.class)
@ActiveProfiles("test")
@Transactional
public abstract class IntegrationTestBase {}
```

**근거**: `@DataJdbcTest` 가 service 레이어 로딩에 맞지 않아 별도 통합 베이스 필요. `IntegrationConfig` 가 service + jdbc 패키지를 스캔하고 JDBC Repository 를 명시 활성화. service 의 단순 Mockito 단위 테스트는 이 베이스가 필요 없으며, **실 H2 DB 를 타는 통합 테스트**에만 이 베이스를 extend 한다.

service 테스트용 `application-test.yml` (H2 mem, repository-jdbc 와 동일 내용) 을 `modules/service/src/test/resources/` 에도 배치. service test 의 testImplementation 에 `project(":modules:repository-jdbc")`, `project(":modules:schema")`, `spring-boot-starter-data-jdbc`, `com.h2database:h2` 필요 — 해당 의존성은 이 시점에 테스트가 없으므로 be-test-service 가 첫 실행 시 build.gradle.kts 에 추가한다.

### 4.10 `modules/schema/src/main/resources/local_h2/schema.sql`

```sql
-- schema.sql
-- 각 도메인 DDL 은 add-entity-specs 스킬이 append 한다.
```

**단일 파일 근거**: Spring Boot의 `spring.sql.init.schema-locations`는 단일 파일을 선호한다. `classpath:schema/*.sql` 패턴은 알파벳 정렬에 종속돼 FK 부모-자식 순서 제어가 어렵다. 따라서 단일 `schema.sql`에 `-- === {Domain} ===` 블록으로 append하는 규약을 표준으로 삼는다.

### 4.11 기타

`.gitignore`:
```
.gradle/
build/
out/
!gradle/wrapper/gradle-wrapper.jar
.idea/
*.iml
.vscode/
data/
*.mv.db
*.trace.db
.DS_Store
*.log
```

`README.md` (최소):
```markdown
# {projectName}

Spring Boot 3.2.1 / Java 21 / Spring Data JDBC / H2 기반 Hexagonal 멀티모듈 백엔드.

## 실행

./gradlew :modules:applications:api-application:bootRun --args='--spring.profiles.active=local'

- Swagger: http://localhost:8080/swagger-ui.html
- H2 Console: http://localhost:8080/h2-console

## 모듈

- applications/api-application — 실행 진입점
- api — Controller / DTO
- service — Reader / Writer / Usecase
- infrastructure — 외부 IO (HTTP/FS/이메일 등)
- repository-jdbc — Spring Data JDBC 구현
- model — Domain model + AuditFields
- schema — DDL (schema.sql)
- exception — 공통 예외
```

각 모듈의 `src/main/java/{packagePath}/...` 에 빈 패키지 + `.gitkeep`.

## 5. 실행 순서 & 자가 수정 루프

### 5.1 실행 순서

1. **입력 파라미터 로딩**: `{projectRoot}/dk-be-bundle.config.json` 우선, 없으면 대화로 4개 값을 받는다. 기본값 제시 금지(특히 `packageRoot`는 사용자 고유).
2. **사전 불변식 검사**: `packagePath == packageRoot.replace('.','/')`. 깨지면 abort.
3. **재실행 가드**: `{projectRoot}/settings.gradle.kts`가 존재하면 abort.
4. **디렉터리 생성**: `mkdir -p`로 8개 모듈의 `src/main/java/{packagePath}/...`와 `.gitkeep` 배치.
5. **파일 작성**: §4의 각 파일을 플레이스홀더(`{projectName}`, `{group}`, `{packageRoot}`, `{packagePath}`) 치환 후 Write.
6. **Gradle Wrapper 생성**: `cd {projectRoot} && gradle wrapper --gradle-version 8.5 --distribution-type bin` 실행, 후 `chmod +x gradlew`. 시스템 `gradle` 없으면 사용자에게 설치 안내 후 중단 (스킬이 wrapper 바이너리를 직접 찍지 않는 이유: 권한/체크섬/인코딩 이슈).
7. **검증 실행**: §5.2 명령 실행.
8. 실패 시 §5.3 원인 분류표로 판정 → 수정 → 재시도 (최대 3회). 표에 없는 에러는 **추측 없이 escalate**.
9. (선택) 기동 스모크: `timeout 30 ./gradlew :modules:applications:api-application:bootRun --args='--spring.profiles.active=local'` 를 백그라운드로 30초 실행해 `Started Application` 로그 확인. 실패해도 스킬은 성공으로 보고(컴파일 통과가 기준).
10. 완료 리포트 (§6).

### 5.2 검증 명령

```bash
./gradlew \
  :modules:applications:api-application:compileJava \
  :modules:applications:api-application:compileTestJava \
  :modules:repository-jdbc:compileTestJava \
  --no-daemon
```

**이 조합 근거**:
- `./gradlew build`는 테스트까지 돌리는데 스캐폴드 시점엔 테스트가 없다.
- `./gradlew assemble`은 jar까지 만든다 — 스캐폴드 검증엔 과함.
- `compileJava` 단독은 테스트 소스(`IntegrationTestBase`)를 검증하지 못한다.
- **3개 태스크**: api-application 컴파일 + 테스트 컴파일(main 모듈 슬라이스 확인) + repository-jdbc 테스트 컴파일(슬라이스·schema classpath 확인). 비용 최소화로 커버리지 확보.

### 5.3 원인 분류표 (보정된 휴리스틱)

| 에러 패턴 | 진단 | 조치 |
| --- | --- | --- |
| `Unsupported class file major version 65` | Gradle 실행 JDK < 21 (클래스 파일 버전 65 = Java 21) | `JAVA_HOME` 을 JDK 21 로 설정하도록 안내. **wrapper 버전은 그대로.** |
| `Unsupported class file major version 61~64` | 동일 (JDK 17~20 필요) | 동일 조치 |
| `Could not find method bootJar(...)` | api-application 에 boot 플러그인 미적용 | §4.3 plugins 블록 확인·재주입 |
| `Task 'bootJar' not found in root project` | 루트에서 `bootJar` 호출 | 모듈 경로로 실행 (`:modules:applications:api-application:bootJar`) |
| `package X does not exist` / `cannot find symbol` | `implementation(project(...))` 누락 | §4.4 의존성 그래프와 대조, 누락 의존 추가 |
| `FAILED ... schema.sql ... not found` | schema 모듈이 classpath 에 없음 | api-application → schema 의존 및 repository-jdbc test 에 `testImplementation(project(":modules:schema"))` 재확인 |
| `Could not resolve ... spring-tx` | 잘못된 좌표 | `org.springframework:spring-tx` (버전은 BOM 관리) |
| `Plugin [id: 'application'] not found` | 구문 인식 이슈 | `plugins { application }` → `plugins { id("application") }` 로 교체 |
| 위 표에 없는 에러 | 모름 | **추측 금지 — stderr 전문 포함해 escalate** |

### 5.4 자가 수정 루프 의사코드

```
MAX_RETRY = 3
for attempt in 1..MAX_RETRY:
    result = run(verifyCmd)
    if result.ok: break
    cause = classify(result.stderr, table=§5.3)
    if cause is None:
        report(result.stderr); abort
    apply(cause.fix)
else:
    report("3회 재시도 소진, 파일은 보존"); abort
```

**MAX_RETRY=3 근거**: 1회차는 타이포, 2회차는 의존 누락, 3회차는 버전/플러그인 이슈가 일반적. 4회 이상은 구조적 문제로 사람이 개입해야 하므로 시도 가치가 낮음.

## 6. 완료 후 상태

- 모든 모듈 `compileJava` + 대상 `compileTestJava` green.
- `./gradlew :modules:applications:api-application:bootRun --args='--spring.profiles.active=local'` 시 `/swagger-ui.html`, `/h2-console` 접근 가능.
- 도메인 테이블은 없다. `schema.sql`은 주석만 포함.
- 각 `src/main/java/{packagePath}/...` 에 `.gitkeep`.

완료 리포트 형식:
```
be-init 완료

- 생성 위치: {projectRoot}
- 모듈: 8개 (api-application 포함)
- 컴파일: OK (retry {N}회)
- 다음 단계: be-planner로 첫 도메인 추가
  └ 예: /dk-be-bundle:be-planner "POST /articles"
```

## 7. Downstream 컨트랙트 (C-1 ~ C-9)

be-planner 및 add-* 스킬이 의존하는 공식 계약.

| # | 내용 |
| --- | --- |
| C-1 | **모듈 include 는 be-init 가 고정.** downstream 은 `settings.gradle.kts` 를 수정하지 않는다. |
| C-2 | 패키지 루트는 `{packageRoot}`. downstream 은 하위 세그먼트만 추가한다. |
| C-3 | 감사 필드명은 `createdAt` / `updatedAt`, 타입 `Instant`, 각 엔티티 본인 필드에 `@CreatedDate` / `@LastModifiedDate` 를 **직접** 선언. |
| C-4 | 엔티티는 `AuditFields` 인터페이스를 **선택적이지만 권장** 으로 구현 (null timestamp 조용한 버그 예방). |
| C-5 | 스키마 위치: `modules/schema/src/main/resources/local_h2/schema.sql` 단일 파일. 도메인별 DDL 은 `-- === {Domain} ===` 블록으로 append. 프로덕션 DB 전환 시 `mysql/schema.sql` 등 형제 디렉토리를 추가할 수 있도록 `local_h2/` 서브디렉토리 구조를 채택. |
| C-6 | 도메인 예외는 `DomainException` 을 extend + `HttpException` 을 implement. `httpStatusCode()` / `httpErrorMessage()` 노출. 404 계열은 `EntityNotFoundException` 을 extend 한 도메인별 하위 클래스로 정의. |
| C-7 | `GlobalExceptionHandler` 는 be-init 소유. 추가 핸들러는 **별도 `@RestControllerAdvice`** 로 분리. |
| C-8 | 테스트 베이스는 be-init 이 제공하는 `IntegrationTestBase` 를 extend (repository-jdbc 슬라이스용, service 통합용 각각). 추가 Bean 은 필요한 Repository 패키지를 `@ComponentScan(basePackages = {...})` 에 명시하거나 단일 Bean 은 `@Import` 로 로드. |
| C-9 | 프로파일은 `local`(H2 file) / `test`(H2 mem). 추가 프로파일은 별도 yml 파일로 분리. |

## 8. 실패 모드

| 모드 | 증상 | 대응 |
| --- | --- | --- |
| JDK 21 미설치 | `compileJava` 1회차에서 class file 65 에러 | §5.3 표에 따라 JAVA_HOME 안내 후 중단 |
| wrapper 생성 실패 | `gradle wrapper` 명령 없음 | `brew install gradle` 또는 sdkman 안내 후 중단 |
| `packagePath` ≠ `packageRoot.replace('.','/')` | 파일 생성 전 즉시 중단 | 파라미터 재입력 요청 |
| BOM vs 명시 버전 충돌 | 루트 BOM이 관리하는 아티팩트에 하위 모듈이 버전을 명시 | 하위 모듈에서 버전 제거, BOM 위임 |
| 3회 재시도 소진 | 분류표 미해당 에러 반복 | **파일은 보존**, 전체 빌드 로그 포함해 escalate (사용자가 이어서 수정) |

## 9. 스킬 경계 (재강조)

- **One-shot**. `settings.gradle.kts` 가 이미 있으면 abort.
- 도메인 생성은 be-planner 및 add-model / add-infrastructure / add-jdbc-query / add-usecase / add-api 스킬의 책임.
- DB 교체 (H2 → MySQL/Postgres) 는 범위 밖. 별도 `be-db-switch` 스킬로 분리 예정.
- 인증/ES/Docker/CI 도 범위 밖. 각각 별도 스킬 또는 수동 작업.
