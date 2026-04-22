---
name: be-test-repository-jdbc
description: repository-jdbc 모듈의 JdbcRepository 통합 테스트를 평가하고 작성한다.
---

# be-test-repository-jdbc

repository-jdbc 모듈의 통합 테스트를 3단계로 수행한다.

---

## 입력

```
대상 JdbcRepository 클래스 이름 (예: SetupJdbcRepository)
```

입력이 없으면 "어떤 Repository를 테스트할까요?" 로 확인한다.

---

## Phase 0. 프로젝트 컨텍스트 해석

`{packageRoot}`, `{modules.repository-jdbc}`, `{modules.schema}` 플레이스홀더 사용.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

## Phase 1. assess-infra — 테스트 인프라 점검

### 점검 항목

| 항목 | 경로 | 확인 내용 |
|------|------|----------|
| build.gradle.kts | `{modules.repository-jdbc}/build.gradle.kts` | `testImplementation`: h2, schema, starter-test |
| application.yml | `{modules.repository-jdbc}/src/test/resources/application.yml` | H2 mem + MODE=MySQL + schema-locations |
| TestApplication.java | `{modules.repository-jdbc}/src/test/java/{packagePath}/jdbc/TestApplication.java` | `@SpringBootApplication` 빈 클래스 |
| schema.sql | `{modules.schema}/src/main/resources/local_h2/schema.sql` | classpath 접근 가능 여부 |

### 점검 순서

1. 위 4개 파일을 Read로 확인
2. 누락된 파일이 있으면 **즉시 생성** (아래 템플릿 사용)
3. 모두 존재하면 "인프라 준비 완료" 출력 후 Phase 2로 진행

### 생성 템플릿

**TestApplication.java** (누락 시):
```java
package {packageRoot}.jdbc;

import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.data.jdbc.repository.config.EnableJdbcAuditing;

@SpringBootApplication
@EnableJdbcAuditing
public class TestApplication {
}
```

> `@EnableJdbcAuditing` 필수 — 없으면 `@CreatedDate`/`@LastModifiedDate` 필드가 런타임에 null 로 남는 조용한 버그.

**application.yml** (누락 시):
```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=MySQL;CASE_INSENSITIVE_IDENTIFIERS=TRUE;DATABASE_TO_LOWER=TRUE
    driver-class-name: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: false
  sql:
    init:
      mode: always
      schema-locations:
        - classpath:local_h2/schema.sql
  data:
    jdbc:
      repositories:
        enabled: true
```

**build.gradle.kts** (test deps 누락 시 — 기존 파일에 추가):
```kotlin
testImplementation("com.h2database:h2")
testImplementation(project(":modules:schema"))
testImplementation("org.springframework.boot:spring-boot-starter-test")
```

---

## Phase 2. assess-scope — 테스트 범위 분석

### 읽어야 할 파일

대상 Repository에 대해 아래 3개 파일을 반드시 읽는다:

1. `*JdbcRepository.java` — 어댑터 (public 메서드 = 테스트 대상)
2. `*EntityRepository.java` — @Query, @Modifying 확인
3. `*Entity.java` — @MappedCollection, @Embedded 확인

추가로 관련 DTO(`*Dto.java`)와 Collection 파일(`*Collection.java`)이 있으면 함께 읽는다.

### 분석 절차

#### 2-1. 메서드 분류

JdbcRepository의 모든 public 메서드를 아래 카테고리로 분류한다:

| 카테고리 | 판단 기준 | 예시 |
|---------|----------|------|
| Entity↔Domain 변환 | save + findById 왕복 | save → findById → 전 필드 일치 |
| Collection 변환 | @MappedCollection 존재 | categories, tags, items |
| JOIN 결과 | EntityRepository에 @Query + LEFT JOIN | CommentWithUserDto |
| @Modifying | EntityRepository에 @Modifying | increaseViewCount |
| 동적 SQL | JdbcTemplate / NamedParameterJdbcTemplate 사용 | findAll with filters |
| 페이징 | Pageable 또는 LIMIT/OFFSET | findByUserId(page, size) |
| 집계 | COUNT, MAX, COALESCE | countByArticle, findMaxCommentOrder |
| 단건 조회 | findByXxx → Optional | findByEmail, findWriteById |
| 삭제 | deleteById | soft delete 확인 |

#### 2-2. @ComponentScan 범위 결정

```
Q: JdbcRepository 내부에서 다른 도메인의 Repository를 사용하는가?
├─ NO → 자기 패키지만
│   @ComponentScan("{packageRoot}.jdbc.{domain}.repository")
│
└─ YES → 사용하는 Repository 패키지 추가
    예: SetupJdbcRepository가 JdbcTemplate만 사용 → 자기만
    예: findById에서 별도 쿼리로 categories/tags 조회 → 자기만 (같은 패키지)
```

**주의**: @Query JOIN은 SQL 레벨이므로 다른 Repository Bean이 필요하지 않다.
다만 **@BeforeEach나 테스트 내에서 선행 데이터를 생성**하려면 해당 Repository를 스캔해야 한다.

**원칙**: 테스트 데이터 생성에 필요한 모든 Repository를 `@ComponentScan`에 포함한다.
package-private인 JdbcRepository는 인터페이스(`*Repository`)로 주입한다.

#### 2-3. @BeforeEach 선행 데이터 판단

```
Q: @Query에 LEFT JOIN이 있는가?
├─ JOIN users → User 선행 생성 필요 → UserJdbcRepository 스캔
├─ JOIN setups → Setup 선행 생성 필요 → SetupJdbcRepository 스캔
├─ JOIN setup_stats → SetupStats 선행 생성 필요
└─ 없음 → 선행 데이터 불필요
```

현재 프로젝트 도메인별 의존:

| 대상 | JOIN 대상 | 선행 데이터 | 추가 스캔 |
|------|----------|-----------|----------|
| Setup | users, setup_stats | User, SetupStats | user, setup(stats) |
| Comment | users, setups | User | user |
| Series | setups | Setup → User, SetupStats | user, setup |
| SavedPost | users, setups, setup_stats | User, Setup, SetupStats | user, setup |
| Rating | — | — | 없음 |
| User | — | — | 없음 |
| UserStats | — | — | 없음 |
| SetupStats | — | — | 없음 |

#### 2-4. 산출물

분석 결과를 아래 형식으로 정리한다:

```
## 분석 결과: {ClassName}

### 테스트 설정
- @ComponentScan: [패키지 목록]
- @BeforeEach: [선행 데이터 목록]

### 테스트 케이스
| # | 메서드 | 카테고리 | 검증 포인트 |
|---|--------|---------|------------|
| 1 | save → findById | 변환 + Collection | categories, tags 왕복 |
| 2 | findAll | 동적 SQL | 필터 조합별 결과 |
| ... | ... | ... | ... |
```

---

## Phase 3. add-test — 테스트 코드 작성

### 파일 위치

```
{modules.repository-jdbc}/src/test/java/{packagePath}/jdbc/{domain}/repository/{Domain}JdbcRepositoryTest.java
```

### 클래스 템플릿

```java
package {packageRoot}.jdbc.{domain}.repository;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.data.jdbc.DataJdbcTest;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.test.context.ActiveProfiles;

import static org.assertj.core.api.Assertions.assertThat;

@DataJdbcTest
@ComponentScan(basePackages = {/* Phase 2 결과 */})
@ActiveProfiles("test")
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class {Domain}JdbcRepositoryTest {

    @Autowired
    private {Domain}JdbcRepository repository;

    // Phase 2에서 도출된 의존 Repository
    // @Autowired private UserJdbcRepository userRepository;

    // 테스트 픽스처
    // private Long testUserId;

    @BeforeEach
    void setUp() {
        // Phase 2에서 도출된 선행 데이터 생성
    }

    // Phase 2의 테스트 케이스 목록에 따라 @Test 메서드 작성
}
```

### 작성 규칙

1. **테스트 데이터는 반드시 Repository를 통해 생성한다.**
   - `JdbcTemplate`/`NamedParameterJdbcTemplate` raw SQL로 INSERT하지 않는다.
   - 다른 도메인 데이터가 필요하면 해당 Repository를 `@ComponentScan`에 추가하고 `@Autowired`로 주입한다.
   - package-private JdbcRepository는 public 인터페이스(`*Repository`)로 주입한다.
   - 이유: raw SQL은 정당한 테스트 데이터인지 코드 리뷰 시 판별이 어렵다.

2. **메서드 네이밍**: `should_예상결과_when_조건`
   ```java
   should_returnSetup_when_savedAndFoundById()
   should_returnEmpty_when_nonExistingId()
   should_incrementViewCount_when_increaseViewCountCalled()
   ```

3. **구조**: given / when / then 주석 블록

4. **카테고리별 패턴**: `${CLAUDE_PLUGIN_ROOT}/references/be-test-refs/repository-jdbc-patterns.md` 참조

5. **동적 SQL 테스트** (SetupJdbcRepository 전용):
   - 필터 조합을 여러 케이스로 분리
   - 빈 필터, 단일 필터, 복합 필터 각각 테스트
   - 정렬 옵션별 테스트

6. **soft delete 고려** (모든 Entity 가 isDeleted/deletedAt 필드 보유 전제):
   - 정석 soft-delete 를 쓰는 도메인은 `deleteById` 후 `findById` 가 empty 반환하는지 확인
   - isDeleted=true 인 데이터가 조회에서 제외되는지 확인
   - hide/show 류(Comment)나 upsert-only(Rating) 도메인은 해당 항목 생략

### 작성 후 검증

```bash
./gradlew :modules:repository-jdbc:test --tests "{패키지}.{테스트클래스}"
```

컴파일 실패 시 원인을 분석하고 수정한다.

---

## 공통 규칙

- Phase 1은 최초 1회만 실행. 이후 동일 모듈의 다른 Repository 테스트 시 건너뛴다.
- Phase 2 → Phase 3는 항상 순차 실행. Phase 2 산출물을 사용자에게 보여주고 확인 후 Phase 3 진행.
- 테스트 코드 내에서 프로덕션 코드를 수정하지 않는다.
- 기존 테스트 파일이 있으면 덮어쓰지 않고 추가/수정한다.
