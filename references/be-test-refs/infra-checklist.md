# 테스트 인프라 체크리스트

모듈별 테스트 인프라 요구사항. `be-test-*` 스킬의 Phase 1에서 참조한다.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## repository-jdbc 모듈

### 필수 파일

| 파일 | 경로 | 역할 |
|------|------|------|
| build.gradle.kts | `modules/repository-jdbc/build.gradle.kts` | testImplementation 선언 |
| application.yml | `modules/repository-jdbc/src/test/resources/application.yml` | H2 + schema 설정 |
| TestApplication.java | `modules/repository-jdbc/src/test/java/share/ai/setup/jdbc/TestApplication.java` | @DataJdbcTest 진입점 |
| schema.sql | `modules/schema/src/main/resources/local_h2/schema.sql` | DDL (classpath 자동 로드) |

### build.gradle.kts 필수 의존성

```kotlin
testImplementation("com.h2database:h2")
testImplementation(project(":modules:schema"))
testImplementation("org.springframework.boot:spring-boot-starter-test")
```

### application.yml 필수 키

```yaml
spring.datasource.url          # jdbc:h2:mem:testdb;MODE=MySQL;...
spring.datasource.driver-class-name  # org.h2.Driver
spring.sql.init.mode           # always
spring.sql.init.schema-locations     # classpath:local_h2/schema.sql
spring.data.jdbc.repositories.enabled  # true
```

### schema.sql 필수 테이블

도메인 테이블 + Collection 자식 테이블:

```
users
user_stats
setups
setup_categories    ← @MappedCollection
setup_tags          ← @MappedCollection
setup_stats
comments
ratings
series
series_items        ← @MappedCollection
saved_posts
```

---

## service 모듈 (단위 테스트)

### 필수 파일

| 파일 | 경로 | 역할 |
|------|------|------|
| build.gradle.kts | `modules/service/build.gradle.kts` | testImplementation 선언 |

### build.gradle.kts 필수 의존성

```kotlin
testImplementation("org.springframework.boot:spring-boot-starter-test")
```

### 불필요

- application.yml — Spring 컨텍스트 안 띄움
- TestApplication.java — Mockito만 사용
- schema.sql — DB 없음

---

## service 모듈 (통합 테스트 — 동시성)

### 필수 파일

| 파일 | 경로 | 역할 |
|------|------|------|
| build.gradle.kts | `modules/service/build.gradle.kts` | 추가 testImplementation |
| application.yml | `modules/service/src/test/resources/application.yml` | H2 + schema |
| IntegrationConfig.java | `modules/service/src/test/java/.../IntegrationConfig.java` | Bean 스캔 설정 |
| IntegrationTestBase.java | `modules/service/src/test/java/.../IntegrationTestBase.java` | 통합 테스트 베이스 |

### build.gradle.kts 추가 의존성

```kotlin
testImplementation("org.springframework.boot:spring-boot-starter-data-jdbc")
testImplementation("com.h2database:h2")
testImplementation(project(":modules:repository-jdbc"))
testImplementation(project(":modules:schema"))
```

---

## exception 모듈

### 필수 파일

build.gradle.kts만 — `spring-boot-starter-test`는 루트에서 전역 적용됨.

### 불필요

나머지 전부. 순수 JUnit + AssertJ만 사용.

---

## api 모듈

### 필수 파일

build.gradle.kts만.

### 불필요

나머지 전부. Mockito로 Controller 직접 호출.

---

## api-application 모듈

### 필수 파일

| 파일 | 경로 | 역할 |
|------|------|------|
| build.gradle.kts | `modules/applications/api-application/build.gradle.kts` | testImplementation |

### build.gradle.kts 필수 의존성

```kotlin
testImplementation("org.springframework.boot:spring-boot-starter-test")
```

MockMvc는 `spring-boot-starter-test`에 포함. `@WebMvcTest` 사용.
