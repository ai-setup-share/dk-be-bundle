# Service References

`be-add-usecase` 스킬이 참고하는 Reader/Writer/Usecase 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 요소

### 클래스 구성과 네이밍

| 역할 | 인터페이스 (service 패키지) | 구현 (impl 패키지) | 배치 |
|---|---|---|---|
| 조회 | `{Domain}Reader` | `Default{Domain}Reader` | public interface / public class |
| 쓰기 | `{Domain}Writer` | `Default{Domain}Writer` | public interface / public class |
| 조합 (선택) | — (인터페이스 없음) | `{Domain}{Action}Usecase` 또는 `{Action}Usecase` | public class 직접 노출 |

- 구현체는 `@Service` + `@RequiredArgsConstructor` + `@Slf4j`
- Usecase 는 인터페이스 없이 클래스 직접 노출 (재사용 Reader/Writer 와 달리 API 단위 전용)

### 의존 방향

- Reader/Writer/Usecase 는 **infra port** (`{Domain}Repository`) 만 의존. `{Domain}JdbcRepository`, `{Domain}EntityRepository`, `{Domain}Entity` 직접 참조 금지.
- Usecase 는 Reader, Writer, 다른 도메인의 Repository 를 자유롭게 주입.

### 메서드 네이밍

- **Reader**: `getById(...)`, `getAll(...)`, `getByUserId(...)` — `get` prefix. 없으면 `orElseThrow(NotFoundException)`. `findXxx` 는 port(Repository) 에서만 사용.
- **Writer**: 동사로 — `create`, `update`, `delete`, `increaseViewCount`, `hide`, `show` 등.
- **Usecase**: public `execute(...)` 1개. 입력은 인증된 userId + Command DTO.

### @Transactional

- Reader: **붙이지 않음** (조회 전용).
- Writer: 쓰기 메서드마다 `@Transactional`.
- Usecase: 클래스 레벨 or 메서드 `execute` 에 `@Transactional`. 선검증 + Writer 호출 + 부수효과를 한 트랜잭션으로.

### Command DTO

- `public record {Action}Command(...)` — Java record
- 패키지: `service/{domain}/dto/`
- 첫 필드 관례: `Long requestUserId` (인증된 사용자 ID)
- 예: `SetupCreateCommand`, `SetupUpdateCommand`, `RatingCommand`, `CommentCreateCommand`

```java
public record SetupCreateCommand(
    Long requestUserId,
    String title,
    String body,
    List<SetupCategory> categories
) {}
```

### 예외 처리

- `EntityNotFoundException` 하위 도메인별 클래스 (`{Domain}NotFoundException`) — 조회 실패 시
- `ForbiddenException` — 소유권/권한 체크 실패
- `ConflictException` — 중복 / 상태 충돌
- `InvalidInputException` — 비즈니스 규칙 위반 입력
- be-init §4.7 의 `DomainException` + `HttpException` 2층 구조 준수 (be-init C-6)

---

## Pattern Foo — Reader 단독 (조회 전용 도메인)

**언제 쓰나**: 사용자 action 으로 변경되지 않는 정적 / reference 성격 도메인.

```java
public interface FooReader {
    List<FooRead> getAll();
    FooRead getById(FooIdentity identity);
}
```

```java
@Service
@RequiredArgsConstructor
public class DefaultFooReader implements FooReader {

    private final FooRepository fooRepository;

    @Override
    public List<FooRead> getAll() {
        return fooRepository.findAll();
    }

    @Override
    public FooRead getById(FooIdentity identity) {
        return fooRepository.findById(identity)
                .orElseThrow(() -> new FooNotFoundException(
                        "foo not found: " + identity.getFooId()));
    }
}
```

**특징**:
- Writer 파일 생성 금지 (변경 로직 없음)
- Usecase 불필요 (조회만이라 조합 없음)
- 구성 파일: `FooReader.java`, `impl/DefaultFooReader.java` 2개

---

## Pattern Bar — Reader + Writer 쌍 (기본 CRUD)

**언제 쓰나**: 일반 CRUD 도메인. 대부분 도메인이 여기에 해당.

### Reader

```java
public interface BarReader {
    BarRead getById(BarIdentity identity);
    List<BarRead> getByUserId(Long userId, int page, int size);
    long countByUserId(Long userId);
}
```

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DefaultBarReader implements BarReader {

    private final BarRepository barRepository;

    @Override
    public BarRead getById(BarIdentity identity) {
        return barRepository.findById(identity)
                .orElseThrow(() -> new BarNotFoundException(
                        "bar not found: " + identity.getBarId()));
    }

    @Override
    public List<BarRead> getByUserId(Long userId, int page, int size) {
        return barRepository.findByUserId(userId, page, size);
    }

    @Override
    public long countByUserId(Long userId) {
        return barRepository.countByUserId(userId);
    }
}
```

### Writer

```java
public interface BarWriter {
    BarIdentity create(BarCreateCommand command);
    Bar update(BarUpdateCommand command);
    void delete(BarIdentity identity);
}
```

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class DefaultBarWriter implements BarWriter {

    private final BarRepository barRepository;

    @Override
    @Transactional
    public BarIdentity create(BarCreateCommand command) {
        Bar bar = Bar.create(command.requestUserId(),
                command.title(), command.body());
        Bar saved = barRepository.save(bar);
        log.info("Bar created: id={}", saved.getBarId());
        return new BarIdentity(saved.getBarId());
    }

    @Override
    @Transactional
    public Bar update(BarUpdateCommand command) {
        BarIdentity id = new BarIdentity(command.barId());
        Bar existing = barRepository.findWriteById(id)
                .orElseThrow(() -> new BarNotFoundException(
                        "bar not found: " + command.barId()));
        Bar updated = existing.withUpdate(command.title(), command.body());
        return barRepository.save(updated);
    }

    @Override
    @Transactional
    public void delete(BarIdentity identity) {
        barRepository.findWriteById(identity)
                .orElseThrow(() -> new BarNotFoundException(
                        "bar not found: " + identity.getBarId()));
        barRepository.deleteById(identity);
        log.info("Bar deleted: id={}", identity.getBarId());
    }
}
```

**특징**:
- Writer 의 update/delete 는 **선검증** (`findWriteById().orElseThrow()`) 으로 404 의미 확보 — JdbcRepository.deleteById 가 조용한 no-op 이므로 여기서 막는다.
- Reader 에는 `@Transactional` 없음.
- 구성 파일: 4개 (`BarReader`, `BarWriter`, `DefaultBarReader`, `DefaultBarWriter`) + Command DTO(s)

---

## Pattern Baz — Usecase (API 단위 조합 + 부수효과)

**언제 쓰나**: API 1개가 여러 도메인을 건드리거나, 부수효과(Stats/Outbox/Notification) 를 동반.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class BazCreateUsecase {

    private final UserRepository userRepository;
    private final UserStatsRepository userStatsRepository;
    private final BazWriter bazWriter;
    private final OutboxEventRecorder outboxEventRecorder;

    @Transactional
    public BazIdentity execute(Long userId, BazCreateCommand command) {
        // 1. 선검증
        userRepository.findById(new UserIdentity(userId))
                .orElseThrow(() -> new UserNotFoundException(
                        "User not found: " + userId));

        // 2. Writer 호출
        BazIdentity created = bazWriter.create(command);

        // 3. 부수효과
        userStatsRepository.increasePostCount(userId);
        outboxEventRecorder.record(
                RecordOutboxEventCommand.created(TargetType.BAZ, created.getBazId()));

        return created;
    }
}
```

권한 체크가 필요한 Usecase:

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class BazDeleteUsecase {

    private final UserRepository userRepository;
    private final BazRepository bazRepository;
    private final BazWriter bazWriter;

    @Transactional
    public void execute(Long userId, BazIdentity identity) {
        userRepository.findById(new UserIdentity(userId))
                .orElseThrow(() -> new UserNotFoundException(
                        "User not found: " + userId));

        Baz existing = bazRepository.findWriteById(identity)
                .orElseThrow(() -> new BazNotFoundException(
                        "Baz not found: " + identity.getBazId()));

        if (!existing.getUserId().equals(userId)) {
            throw new ForbiddenException("not owner");
        }

        bazWriter.delete(identity);
    }
}
```

**특징**:
- public 메서드는 `execute(...)` 1개
- `@Transactional` 메서드 (또는 클래스) 레벨
- 선검증 → Writer/Reader → 부수효과 순서
- 부수효과 예: `{Domain}StatsRepository.xxx`, `UserStatsRepository.xxx`, `OutboxEventRecorder.record(...)`, `NotificationTriggerUsecase.onXxx(...)`
- 인터페이스 없이 **클래스 직접 노출** — API 단위라 재사용성 없음

**파일 배치**: `service/{domain}/usecase/{Domain}{Action}Usecase.java`
- 예: `SetupCreateUsecase`, `SetupDeleteUsecase`, `RatingUsecase`, `BookmarkUsecase`

---

## Pattern Qux — Usecase 단독 (Reader/Writer 없이 Repository 직접 조립)

**언제 쓰나**: 대시보드/마이페이지처럼 여러 도메인의 집계를 조합하지만 **API 1개 전용** — 중간 Reader 로 추상화할 가치 낮음.

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class MypageUsecase {

    private final UserRepository userRepository;
    private final SetupRepository setupRepository;
    private final CommentRepository commentRepository;
    private final UserStatsRepository userStatsRepository;

    public MypageRead execute(Long userId) {
        User user = userRepository.findById(new UserIdentity(userId))
                .map(UserRead -> UserRead.toWrite())  // 또는 Read 가 적절하면 그대로
                .orElseThrow(() -> new UserNotFoundException(
                        "User not found: " + userId));

        List<SetupRead> mySetups = setupRepository.findByUserId(userId, 0, 5);
        List<CommentRead> myComments = commentRepository.findByUserId(new UserIdentity(userId), 0, 5);
        UserStats stats = userStatsRepository.findByUserId(new UserIdentity(userId))
                .orElse(UserStats.init(userId));

        return new MypageRead(user, mySetups, myComments, stats);
    }
}
```

**특징**:
- Reader/Writer 인터페이스 없음
- Repository 를 직접 다수 주입
- 대부분 읽기만 → `@Transactional` 생략하거나 `readOnly = true`
- 구성 파일: Usecase 1개

---

## Pattern Quux — 인메모리 버퍼링 + 주기 플러시 (특수)

**언제 쓰나**: 조회수·좋아요 같은 **고빈도 이벤트** 가 그대로 DB 에 가면 락 경합이 심할 때. 메모리에 누적 → 주기적으로 배치 flush.

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class FooViewMemory {

    private final FooStatsRepository fooStatsRepository;
    private final FooRepository fooRepository;
    private final UserStatsRepository userStatsRepository;
    private final OutboxEventRecorder outboxEventRecorder;

    private final Map<Long, AtomicLong> viewCounts = new ConcurrentHashMap<>();

    public void countUp(Long fooId) {
        viewCounts.computeIfAbsent(fooId, k -> new AtomicLong(0))
                .incrementAndGet();
    }

    @Scheduled(fixedDelay = 10000)
    public void flush() {
        if (viewCounts.isEmpty()) return;

        Map<Long, AtomicLong> snapshot = new ConcurrentHashMap<>(viewCounts);
        viewCounts.clear();

        int count = 0;
        for (Map.Entry<Long, AtomicLong> entry : snapshot.entrySet()) {
            Long fooId = entry.getKey();
            long increment = entry.getValue().get();

            try {
                FooIdentity identity = new FooIdentity(fooId);
                fooStatsRepository.increaseViewCount(identity, increment);

                fooRepository.findWriteById(identity)
                        .ifPresent(foo -> userStatsRepository.increaseViewCount(foo.getUserId()));

                outboxEventRecorder.record(
                        RecordOutboxEventCommand.popularityOnly(TargetType.FOO, fooId));
                count++;
            } catch (Exception e) {
                log.error("[FooViewMemory] 플러시 실패: fooId={}, error={}", fooId, e.getMessage());
            }
        }

        if (count > 0) {
            log.info("[FooViewMemory] 플러시 완료: {}건", count);
        }
    }
}
```

**특징**:
- `@Component` + `@Scheduled` + `@EnableScheduling` (애플리케이션 설정)
- `ConcurrentHashMap` + `AtomicLong` — 스레드 안전
- flush 중 새 데이터 유실 방지: snapshot → clear 순서
- 예외는 **개별 row 단위로 격리** (catch 내부 log + continue)
- 배치 위치: `service/{domain}/view/`
- 일반 도메인엔 쓰지 않음. 고부하 지표 계측이 필요할 때만.

---

## 패턴 선택 가이드

| 도메인 특성 | Pattern |
|---|---|
| 조회만 (정적 / reference 데이터) | **Foo** — Reader 단독 |
| 기본 CRUD | **Bar** — Reader + Writer 쌍 |
| API 1개가 여러 도메인 + 부수효과 | **Baz** — Usecase (Bar 위에 얹음) |
| 집계/대시보드, 중간 Reader 불필요 | **Qux** — Usecase 단독 |
| 고빈도 이벤트 집계 (특수) | **Quux** — ViewMemory 류 |

조합 예:
- 일반 도메인: **Bar + Baz** (Reader/Writer 쌍 + Create/Update/Delete 각 Usecase)
- 단순 정적 데이터: **Foo** 만
- 대시보드: **Qux** 만
- 조회수 높은 도메인: **Bar + Baz + Quux**

---

## 💡 팁 — 위 패턴들로 풀리지 않을 때

1. **중간 레이어 추가**를 우선 고민하지 않기 — Reader/Writer 로 충분하면 거기서 끝낸다. 과한 추상화는 유지 비용만 증가.
2. **Usecase 수가 급증**하면 도메인 경계 재검토 — 한 도메인의 Usecase 가 10개 이상이면 aggregate 를 쪼개야 할 신호.
3. **여러 서비스 호출 조합이 복잡**하면 자바 레이어에서 명시적 helper 메서드로 추출 (내부 package-private static 또는 Usecase 내부 private).
4. 그래도 표현이 어색하면 **유저에게 질의** — 이벤트 드리븐 / 비동기 파이프라인 / 스케줄러 도입 등 구조적 선택지.

---

## 파일 구성 요약

```
service/src/main/java/{packagePath}/service/{domain}/
├── {Domain}Reader.java                 ← interface (Pattern Foo/Bar)
├── {Domain}Writer.java                 ← interface (Pattern Bar)
├── dto/
│   ├── {Domain}CreateCommand.java      ← record
│   └── {Domain}UpdateCommand.java
├── impl/
│   ├── Default{Domain}Reader.java      ← @Service
│   └── Default{Domain}Writer.java
├── usecase/                            ← Pattern Baz / Qux (선택)
│   ├── {Domain}CreateUsecase.java
│   ├── {Domain}UpdateUsecase.java
│   └── {Domain}DeleteUsecase.java
└── view/                               ← Pattern Quux (특수, 드묾)
    └── {Domain}ViewMemory.java
```

---

## 공통 금지 사항

- `@Component` 대신 `@Service` 사용 (Quux 의 ViewMemory 는 예외적으로 @Component)
- Default 구현에서 `*JdbcRepository` / `*Entity` / `*EntityRepository` 직접 참조 금지 — 항상 port 인터페이스
- Reader 에서 mutation (save/delete 등) 호출 금지
- Writer 의 조회 반환은 Write 모델 (`{Domain}`) — Read 모델(`{Domain}Read`) 반환은 Reader 의 책임
- Usecase 간 상호 호출은 지양. 공통 로직은 Writer 나 Repository 로 내리기 (Usecase 체이닝은 트랜잭션·의존 그래프 혼란)
- Command record 에 `setter`/가변 필드 금지 (record 이므로 자동 금지됨)
