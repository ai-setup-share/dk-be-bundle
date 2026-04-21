---
name: be-test-service-concurrency
description: service 모듈의 동시성/순서 보장 통합 테스트를 평가하고 작성한다.
---

# be-test-service-concurrency

읽고→계산→쓰기 패턴이 있는 Writer 메서드의 동시성 통합 테스트를 3단계로 수행한다.

---

## 입력

```
대상 Writer 클래스 이름 (예: DefaultCommentWriter)
```

입력이 없으면 "어떤 Writer를 테스트할까요?" 로 확인한다.

---

## Phase 0. 프로젝트 컨텍스트 해석

`{packageRoot}`, `{modules.service}` 플레이스홀더 사용.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

---

## Phase 1. assess-infra

| 항목 | 경로 | 확인 |
|------|------|------|
| build.gradle.kts | `{modules.service}/build.gradle.kts` | h2, schema, repository-jdbc, starter-data-jdbc, starter-test |
| application.yml | `{modules.service}/src/test/resources/application.yml` | H2 mem + MODE=MySQL |
| IntegrationConfig | `...service/IntegrationConfig.java` | @ComponentScan(service, jdbc) + @EnableJdbcRepositories(jdbc) |
| ConcurrencyTestBase | `...service/ConcurrencyTestBase.java` | 존재 여부 |

ConcurrencyTestBase가 없으면 즉시 생성:

```java
package {packageRoot}.service;

import org.junit.jupiter.api.AfterEach;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.test.context.ActiveProfiles;

@SpringBootTest(classes = IntegrationConfig.class)
@ActiveProfiles("test")
public abstract class ConcurrencyTestBase {

    @Autowired
    protected JdbcTemplate jdbcTemplate;

    @AfterEach
    void cleanUp() {
        // FK 순서: 자식 → 부모
        jdbcTemplate.execute("DELETE FROM comments");
        jdbcTemplate.execute("DELETE FROM ratings");
        jdbcTemplate.execute("DELETE FROM saved_posts");
        jdbcTemplate.execute("DELETE FROM series_items");
        jdbcTemplate.execute("DELETE FROM series");
        jdbcTemplate.execute("DELETE FROM setup_stats");
        jdbcTemplate.execute("DELETE FROM setup_categories");
        jdbcTemplate.execute("DELETE FROM setup_tags");
        jdbcTemplate.execute("DELETE FROM setups");
        jdbcTemplate.execute("DELETE FROM user_stats");
        jdbcTemplate.execute("DELETE FROM users");
    }
}
```

---

## Phase 2. assess-scope

### 2-1. 대상 판별

대상 Writer의 각 public 메서드에 대해 specs를 읽고 동시성 위험을 판단한다.

**읽을 specs:**
- `specs/repository.md` — 의존 Repository 메서드의 구현 방식
- `specs/usecase.md` — Writer 인터페이스 동작 설명

### 2-2. 동시성 위험 판단 규칙

| Repository 메서드 구현 방식 (specs/repository.md에서 확인) | 원자성 | 위험 |
|--------------------------------------------------------|--------|------|
| `@Modifying @Query` — 단일 UPDATE/DELETE 문 | 원자적 | 안전 |
| `findXxx → 계산 → save` — 조회 후 저장 | 비원자 | **위험** |
| `getMaxXxx → +1 → save` — 최댓값 기반 생성 | 비원자 | **위험** |
| `existsXxx → save/update` — 존재 확인 후 변경 | 비원자 | **위험** |

**판단 흐름:**

```
Writer 메서드 내부에서 Repository를 어떻게 호출하는가?

Q1: 조회 결과를 기반으로 계산 후 쓰기를 하는가?
├─ NO  → 동시성 TC 불필요 (이 메서드 건너뜀)
└─ YES
    Q2: 그 쓰기가 @Modifying @Query (단일 SQL)인가?
    ├─ YES → 쓰기 자체는 안전하지만 조회-판단 갭 존재 → check-then-act TC
    └─ NO  → 전체 비원자 → 순서/합산 TC
```

### 2-3. TC 유형 결정

| 위험 패턴 | TC 유형 | 검증 방법 |
|----------|---------|----------|
| 최댓값 기반 순번 생성 (max+1) | **순서 유일성** | N개 동시 생성 → 번호 중복 없는지 |
| 카운터 갱신 (조회→계산→UPDATE) | **합산 정확성** | N개 동시 갱신 → 최종값 == 기대값 |
| 존재 확인 후 생성/수정 | **중복 방지** | N개 동시 시도 → 정확히 1개만 생성 or 합산 정확 |

### 2-4. 산출물

```
## 분석 결과: {ClassName}
### 동시성 대상 메서드
| 메서드 | 위험 패턴 | TC 유형 | 검증 |
```

사용자 확인 후 Phase 3 진행.

---

## Phase 3. add-test

### 파일 위치

```
{modules.service}/src/test/java/{packagePath}/service/{domain}/
└── impl/Default{Xxx}WriterConcurrencyTest.java
```

### 템플릿

```java
class Default{Xxx}WriterConcurrencyTest extends ConcurrencyTestBase {

    @Autowired private {Xxx}Writer writer;
    @Autowired private UserRepository userRepository;
    @Autowired private SetupRepository setupRepository;
    @Autowired private SetupStatsRepository setupStatsRepository;
    // 필요시 추가: UserStatsRepository 등

    private Long testUserId;
    private Long testSetupId;

    @BeforeEach
    void setUp() {
        User user = userRepository.save(User.create("tester", "test@test.com", "google123", null));
        testUserId = user.getUserId();

        Setup setup = setupRepository.save(Setup.create(
                testUserId, SetupType.SETUP, "테스트", null, "본문", null, List.of(), List.of()));
        testSetupId = setup.getSetupId();

        setupStatsRepository.save(SetupStats.init(testSetupId));
        // 필요시: userStatsRepository.save(UserStats.init(testUserId));
    }
}
```

### 동시 실행 패턴 (모든 TC 공통)

```java
int threadCount = 10;
ExecutorService executor = Executors.newFixedThreadPool(threadCount);
CountDownLatch ready = new CountDownLatch(threadCount);
CountDownLatch start = new CountDownLatch(1);
List<Future<?>> futures = new ArrayList<>();

for (int i = 0; i < threadCount; i++) {
    futures.add(executor.submit(() -> {
        ready.countDown();
        start.await();
        // 동시 실행할 Writer 메서드 호출
        return null;
    }));
}
ready.await();
start.countDown();
for (Future<?> f : futures) f.get(10, TimeUnit.SECONDS);
executor.shutdown();
```

### TC 유형별 검증

**순서 유일성:**
```java
List<Integer> values = jdbcTemplate.queryForList("SELECT {column} FROM {table} WHERE ...", Integer.class);
assertThat(values).hasSize(threadCount);
assertThat(new HashSet<>(values)).hasSize(threadCount);
```

**합산 정확성:**
```java
Long total = jdbcTemplate.queryForObject("SELECT {counter_column} FROM {table} WHERE ...", Long.class);
assertThat(total).isEqualTo(threadCount);  // 또는 기대 합산값
```

### 검증

```bash
./gradlew :{modules.service.gradlePath}:test --tests "{packageRoot}.service.{domain}.impl.{ConcurrencyTestClass}"
```

---

## 공통 규칙

- Phase 1은 최초 1회만 실행
- Phase 2 산출물을 사용자에게 보여주고 확인 후 Phase 3 진행
- 테스트 코드 내에서 프로덕션 코드를 수정하지 않는다
- 기존 테스트 파일이 있으면 덮어쓰지 않고 추가/수정한다
- 선행 데이터는 반드시 도메인 Repository API를 통해 생성한다 (jdbcTemplate 직접 사용 금지 — cleanUp 제외)
