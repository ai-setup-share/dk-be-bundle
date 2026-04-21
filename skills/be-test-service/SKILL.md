---
name: be-test-service
description: service 모듈의 단위 테스트(Mockito)를 평가하고 작성한다.
---

# be-test-service

service 모듈의 Mockito 단위 테스트를 3단계로 수행한다.

---

## 입력

```
대상 서비스 클래스 이름 (예: SetupCreateUsecase, DefaultSetupReader)
```

입력이 없으면 "어떤 클래스를 테스트할까요?" 로 확인한다.

---

## Phase 0. 프로젝트 컨텍스트 해석

`{packageRoot}`, `{modules.service}` 플레이스홀더 사용. 실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

---

## Phase 1. assess-infra

| 항목 | 경로 | 확인 |
|------|------|------|
| build.gradle.kts | `{modules.service}/build.gradle.kts` | `spring-boot-starter-test` 있는지 |

있으면 PASS, 없으면 추가.

**불필요**: application.yml, TestApplication, IntegrationConfig, H2 — Spring 컨텍스트 없음.

---

## Phase 2. assess-scope

### 2-1. 유형 판별

```
파일 이름?
├─ *Usecase      → USECASE
├─ Default*Reader → READER
└─ Default*Writer → WRITER
```

### 2-2. 의존성 추출 + specs lookup

1. 대상 클래스의 `final` 필드 전부 → `@Mock` 후보
2. 각 의존성의 **API 시그니처**를 specs에서 읽는다:
   - `specs/usecase.md` — Reader/Writer 인터페이스 메서드 시그니처 + 동작 설명
   - `specs/repository.md` — Repository 인터페이스 메서드 시그니처
   - `specs/models.md` — 도메인 팩토리, 상태 변경 메서드, 반환 타입
3. **실제 인터페이스 소스**도 Read로 확인한다 (specs와 실제 반환 타입이 다를 수 있음)

### 2-3. 테스트 케이스 도출

대상 클래스의 각 public 메서드에 대해, **의존 API의 반환 타입**으로 분기를 결정한다.

#### 분기 판단 규칙

| 의존 API 반환 타입 | 분기 | TC |
|-------------------|------|-----|
| `Optional<X>` | present / empty | 정상 처리 + 예외(NotFoundException 등) |
| `boolean` | true / false | 각 분기의 후속 동작 |
| `OptionalInt` | present / empty | 각 분기의 후속 동작 |
| `List<X>` | 비어있지 않음 / 빈 리스트 | (Reader에서만 — 양쪽 TC) |
| `long` / `int` / `void` | 단일 | 위임 결과 반환 or verify |

#### 추가 TC 규칙

| 코드 패턴 | TC 추가 |
|----------|---------|
| `userId.equals(entity.getUserId())` | 소유권 불일치 → ForbiddenException |
| `if/else` 또는 `?:` 분기 | 분기별 TC 각각 |
| `verify()` 대상 부수효과 (stats 갱신 등) | happy path에서 verify 포함 |
| 예외 경로 후 위임 호출 | `verify(xxx, never())` 포함 |

#### 예시: SetupCreateUsecase

```
specs lookup:
  UserRepository.findById(UserIdentity) → Optional<User>    ... present/empty 분기
  SetupWriter.create(SetupCreateCommand) → SetupIdentity     ... 단일 반환
  UserStatsRepository.increasePostCount(Long) → void          ... verify 대상

TC 도출:
  1. should_createSetup_when_validRequest     — Optional.of → create → verify increasePostCount
  2. should_throwUserNotFound_when_noUser     — Optional.empty → assertThrows, verify never create
```

### 2-4. 산출물

```
## 분석 결과: {ClassName}
### 유형: USECASE / READER / WRITER
### Mock 의존성 (specs 출처)
| 의존성 | 메서드 | 반환 타입 | specs 위치 |
### 테스트 케이스
| # | 테스트명 | Mock 설정 요약 | 검증 | 패턴 |
```

사용자 확인 후 Phase 3 진행.

---

## Phase 3. add-test

### 파일 위치

```
{modules.service}/src/test/java/{packagePath}/service/{domain}/
├── impl/Default{Xxx}ReaderTest.java
├── impl/Default{Xxx}WriterTest.java
└── usecase/{Xxx}UsecaseTest.java
```

### 클래스 템플릿

```java
@ExtendWith(MockitoExtension.class)
class {ClassName}Test {
    @Mock private XxxRepository xxxRepository;
    @InjectMocks private {ClassName} sut;
}
```

### 작성 규칙

1. **네이밍**: `should_예상결과_when_조건`
2. **구조**: `// given` → `// when` → `// then`
3. **예외**: `assertThatThrownBy(() -> ...).isInstanceOf(XxxException.class)`
4. **부수효과**: `verify(repo).xxx()` / `verify(repo, never()).xxx()`
5. **저장 반환**: `when(repo.save(any())).thenAnswer(inv -> inv.getArgument(0))`
6. **도메인 객체**: specs/models.md의 팩토리 메서드로 생성 우선, ID 필요 시만 `mock()`
7. **Spring 컨텍스트 금지**: `@ExtendWith(MockitoExtension.class)` 만 사용

코드 예제는 `${CLAUDE_PLUGIN_ROOT}/references/be-test-refs/service-patterns.md`의 패턴 레터 참조.

### 검증

```bash
./gradlew :{modules.service.gradlePath}:test --tests "{packageRoot}.service.{domain}.{subpackage}.{TestClass}"
```

---

## 공통 규칙

- Phase 1은 최초 1회만 실행
- Phase 2 산출물을 사용자에게 보여주고 확인 후 Phase 3 진행
- 테스트 코드 내에서 프로덕션 코드를 수정하지 않는다
- 기존 테스트 파일이 있으면 덮어쓰지 않고 추가/수정한다
