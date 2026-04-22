# Infrastructure References

`be-add-infrastructure` 스킬이 참고하는 port interface (Repository) 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## 공통 요소

### 역할과 배치

- **port** (이 문서의 대상): service 레이어가 사용하는 **추상 계약**. 구현은 모른다.
- 파일 위치: `infrastructure/src/main/java/{packagePath}/infra/{domain}/repository/{Domain}Repository.java`
- 구현체 위치: `repository-jdbc/.../jdbc/{domain}/repository/{Domain}JdbcRepository.java` (be-add-jdbc-query 가 생성)

### 인터페이스 작성 규칙

- **순수 `public interface`** — 어노테이션 없음, default method 없음, static method 없음
- JavaDoc 한 줄로 각 메서드 의도 명시 — 어댑터 구현자가 기대값을 알 수 있게
- 반환/파라미터 타입에 **JDBC `*Entity` 누설 금지** — 도메인 모델 (`{Domain}`, `{Domain}Read`, `{Domain}Identity`) 만 사용

### 메서드 네이밍

| 용도 | 네이밍 | 반환 |
|---|---|---|
| 단건 조회 (읽기 전용) | `findById(...)` | `Optional<{Domain}Read>` (JOIN/집계 포함 가능) |
| 단건 조회 (쓰기 전 선검증) | `findWriteById(...)` | `Optional<{Domain}>` (순수 엔티티) |
| 리스트 조회 | `findByXxx(..., int page, int size)` | `List<{Domain}Read>` |
| 카운트 | `countByXxx(...)` | `long` |
| 존재 확인 | `existsByXxx(...)` | `boolean` |
| 저장 (insert+update) | `save({Domain})` | `{Domain}` (id 포함) |
| 삭제 | `deleteById({Domain}Identity)` | `void` — soft-delete, 이미 삭제면 no-op |
| 원자 증감 (Stats) | `increaseXxx` / `incrementXxx` / `addXxx` | `void` 또는 갱신 결과 |

### 파라미터 관례

- **Identity VO 우선** — Long 직접 받지 않는다 (`SetupIdentity identity`, `UserIdentity userIdentity`)
- **원시 Long**은 page/size/userId (인증 컨텍스트에서 받은 정수 키) 정도에서만 허용
- `ArticleType` 같은 enum 은 그대로 (String 변환 금지)

### Read 모델 vs Write 모델 분리 (Bar 패턴 핵심)

- `findById` → **`{Domain}Read`** 반환 (JOIN·집계 포함, 화면용)
- `findWriteById` → **`{Domain}`** 반환 (순수 엔티티, update/delete 전 선검증 전용)
- 두 메서드를 같은 interface 에 나란히 둔다

---

## Pattern Foo — 단순 CRUD port

**언제 쓰나**: JOIN/집계 불필요, 한 도메인 혼자 처리. 조회·저장·존재 확인만.

```java
public interface FooRepository {

    /** Foo 단건 조회. */
    Optional<Foo> findById(FooIdentity identity);

    /** 이메일로 Foo 조회. */
    Optional<Foo> findByEmail(String email);

    /** 닉네임 중복 확인. */
    boolean existsByNickname(String nickname);

    /** Foo 저장 (신규·수정). */
    Foo save(Foo foo);
}
```

**특징**:
- `findById` 반환이 **Write 모델 `Foo`** — Read 모델 분리 가치 낮음 (JOIN 없음)
- `deleteById` 있으면 추가. Rating 처럼 upsert only 면 생략
- 메서드 3~5개가 보통
- 예: `RatingRepository` (findByArticleAndUserId + save), `UserRepository` (findById + findByEmail + save + existsByNickname)

---

## Pattern Bar — Read/Write 분리 + 복수 조회 (주력 aggregate)

**언제 쓰나**: 조회 시 작성자명·통계·자식 컬렉션 JOIN 이 필요한 핵심 aggregate. 리스트/카운트/상세 조회·쓰기·삭제를 다 담당.

```java
public interface BarRepository {

    /** Bar 상세 조회. 작성자·통계·자식 컬렉션 포함. */
    Optional<BarRead> findById(BarIdentity identity);

    /** Bar Write 모델 조회. 수정/삭제 선검증 시 사용. */
    Optional<Bar> findWriteById(BarIdentity identity);

    /** Bar 목록 조회. 필터·정렬·페이지 지원. */
    List<BarRead> findByUserId(Long userId, int page, int size);

    /** Bar 목록 총 건수. */
    long countByUserId(Long userId);

    /** Bar 저장 (신규·수정). */
    Bar save(Bar bar);

    /** Bar 소프트 삭제. 이미 삭제된 건 no-op. */
    void deleteById(BarIdentity identity);

    /** 소프트 삭제된 Bar 조회. ES 삭제 동기화 등 특수 용도. */
    Optional<Bar> findDeletedById(BarIdentity identity);

    /** 주어진 ID 목록 중 존재(삭제되지 않음) 하는 ID 집합. 배치 존재 확인용. */
    Set<Long> findExistingIds(List<Long> ids);
}
```

**특징**:
- **핵심 관례: `findById` (Read) + `findWriteById` (Write) 병존** — 서비스가 용도로 선택
- `findAll` 은 보통 없음 (화면 요구사항이 필터·정렬을 동반하므로 `findByUserId` 같이 구체적 이름)
- `deleteById` 반환 `void` — soft-delete. 404 의미는 Usecase 에서 선검증으로 막음
- `findDeletedById` 는 검색/알림/ES 동기화 같은 **특수 용도** 가 있을 때만 추가
- `findExistingIds` 는 배치 작업·outbox 등에서 존재 확인 최적화용, 필요 없으면 생략
- 예: `SetupRepository`, `CommentRepository`

---

## Pattern Baz — Stats 집계 전용 port (본 aggregate 와 **별도**)

**언제 쓰나**: 통계 테이블의 카운터 원자적 증감. 쓰기 경쟁 격리 목적.

```java
public interface BarStatsRepository {

    /** Bar 통계 조회. */
    Optional<BarStats> findByBarId(BarIdentity identity);

    /** 통계 저장 (신규). Bar 생성 시 init(0,0,0,0) 으로. */
    BarStats save(BarStats stats);

    /** 조회수 원자적 증가. */
    void increaseViewCount(BarIdentity identity, long increment);

    /** 평점 합계/건수 원자적 갱신 (평가 추가 시). 갱신된 통계 반환. */
    BarStats addRating(BarIdentity identity, int score);

    /** 평점 합계/건수 원자적 갱신 (평가 수정 시). 갱신된 통계 반환. */
    BarStats updateRating(BarIdentity identity, int oldScore, int newScore);

    /** 댓글 수 원자적 증가. */
    void incrementCommentCount(BarIdentity identity);
}
```

**특징**:
- **본 aggregate port 와 별도 파일** (`BarRepository` 와 `BarStatsRepository` 분리)
- 증감 메서드는 **도메인 의도로 네이밍** — `increaseViewCount`, `addRating` 같이
- 반환: 값 필요 없으면 `void`, 갱신된 상태가 필요하면 `BarStats`
- 예: `SetupStatsRepository`, `UserStatsRepository`

---

## Pattern Qux — 트리·순서 조작 (Bar 에 병합)

**언제 쓰나**: 댓글·메뉴·카테고리 같은 트리 구조. 정렬·자식 카운트 조정이 필요.

**별도 interface 로 분리하지 않고 Bar 에 병합**:

```java
public interface QuxRepository {

    // --- Bar 기본 (findById/findWriteById/save/deleteById/...) ---
    Optional<QuxRead> findById(QuxIdentity identity);
    Optional<Qux> findWriteById(QuxIdentity identity);
    Qux save(Qux qux);

    // --- 트리·순서 조작 ---

    /** 아티클의 최대 commentOrder 조회. 다음 댓글 순서 결정용. */
    int getMaxCommentOrder(Long articleId, ArticleType articleType);

    /** 같은 commentOrder 그룹에서 부모의 sortNumber 이후 값들을 +1 밀어내기. */
    void incrementSortNumbersAbove(Long articleId, ArticleType articleType,
                                   int commentOrder, int sortNumber);

    /** 부모 댓글의 childCount 를 1 증가. 재귀적으로 조상 체인까지 증가. */
    void incrementChildCount(QuxIdentity parentId);
}
```

**특징**:
- **Bar 와 별도 분리하지 않는다** — 트리 연산이 같은 테이블에 작용하므로 한 port 에서 관리
- 트리 연산은 호출자(Usecase) 의 트랜잭션 경계에서 보호
- 예: `CommentRepository`

---

## 패턴 선택 가이드

| 도메인 특성 | Pattern |
|---|---|
| 단순 CRUD, JOIN 불필요 | **Foo** |
| Read 모델에 JOIN/집계 필요, 리스트·카운트 조회 | **Bar** (주력) |
| 통계 카운터 증감 전용 | **Baz** (본 aggregate 의 **사이드** port, 파일 별도) |
| 트리·순서 연산 | **Qux** (Bar 에 병합) |

조합 예:
- Rating 도메인: **Foo**
- Setup 도메인: **Bar** (SetupRepository) + **Baz** (SetupStatsRepository) — 파일 2개
- Comment 도메인: **Bar + Qux** 병합 (CommentRepository 한 파일)
- User 도메인: **Foo** (UserRepository) + **Baz** (UserStatsRepository)

---

## 💡 팁 — 위 패턴들로 풀리지 않을 때

1. **별도 port 파일 분리**를 우선 고민하지 않기 — Bar 나 Qux 처럼 한 파일에 병합이 낫다. 파일이 커질 때 분리.
2. **새 도메인 등장**이면 보통 Foo 로 시작 — 필드가 단순하고 JOIN 필요 없으면.
3. **JOIN/집계가 생기면** Bar 로 승격 — `findById` 반환을 `Read` 로 바꾸고 `findWriteById` 추가.
4. **쓰기 경쟁이 보이면** Baz 로 통계 분리 — 조회수·평점·댓글수가 별 테이블이 낫다.
5. 그래도 표현이 어색하면 **유저에게 질의** — 도메인 경계 재설계 / aggregate 분할 / 이벤트 기반 컴포즈 등 선택지.

---

## 파일 구성 요약

```
infrastructure/src/main/java/{packagePath}/infra/{domain}/repository/
├── {Domain}Repository.java          ← Pattern Foo / Bar / Qux
└── {Domain}StatsRepository.java     ← Pattern Baz (Bar 와 별도)
```

---

## 공통 금지 사항

- 어노테이션 금지 (`@Repository` 는 adapter 쪽, port 는 순수 interface)
- default / static method 금지 — 구현은 어댑터 전담
- JDBC `*Entity` / `*JdbcRepository` / `*EntityRepository` import 금지 — port 는 구현을 모른다
- `Pageable` / `Page` 같은 Spring Data 타입 노출 금지 — `int page, int size` 로 받아 내부 변환
- `throws` 명시 금지 — 예외는 adapter 에서 도메인 예외로 변환해 던짐
- null 반환 금지 — `Optional<>` 사용
