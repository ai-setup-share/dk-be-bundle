# Model References

`be-add-model` 스킬이 참고하는 도메인 모델 작성 레퍼런스.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

모든 모델은 **불변 값 객체(@Value) + 팩토리/전이 메서드** 형태다. 아래 4개 바리에이션과 공통 요소를 조합해 도메인 모델을 구성한다.

---

## 공통 요소

### AuditFields 인터페이스 구현

모든 Write/Read 모델은 `{packageRoot}.model.audit.AuditFields` 구현 (be-init §4.6 / C-4). `createdAt: Instant`, `updatedAt: Instant` 필드 보유.

### Identity VO

단일 ID 래핑. 모든 도메인에 1개 존재.

```java
@Value
public class FooIdentity {
    Long fooId;
}
```

**이유**: 서비스/컨트롤러 시그니처에서 `Long` raw 타입보다 의미 명확. 여러 도메인의 ID가 메서드 파라미터에 섞일 때 타입 안전성.

### Lombok 어노테이션

- `@Value`: 불변 객체 (all-final + toString + equals + @AllArgsConstructor)
- 생성자 명시 선언 금지 — `@Value` 가 생성
- `@Builder` 금지 — 팩토리 메서드로 의도 표현

### enum 필드

값 유한·고정이면 enum (`company`, `category`, `type`, `role`, `source`).
자유 입력이면 String (`title`, `content`, `url`).

```java
public enum FooSource {
    COMPANY, OFFICIAL, SELF
}
```

VO 저장용이 아니라면 **추가 필드 없이 name()** 만 사용 (Spring Data JDBC 가 VARCHAR 로 자동 변환).

### 팩토리 메서드 명명

- `create(...)`: 새 도메인 생성. id=null, createdAt=now, updatedAt=now.
- `withXxx(...)`: 필드 일부 변경. id/createdAt 유지, updatedAt=now.
- `toRead(...)`: Write → Read 변환. 필요한 JOIN 데이터를 파라미터로 받음.

**생성자 직접 호출 금지** (내부 팩토리/전이 메서드 구현 제외).

### 하위 VO 분리 기준

3개 이상의 유사 의미 필드 묶음 → 별도 VO 로 추출. 같은 도메인 패키지 하위에 배치.

---

## Pattern Foo — 단순 Write-only (Read 없음)

**언제 쓰나**: Upsert only 도메인, 혹은 조회 시 JOIN/집계가 필요 없는 단순 레코드 (평점, 반응 등).

```java
@Value
public class Foo implements AuditFields {
    Long fooId;
    Long userId;
    Long targetId;
    FooSource source;
    int score;
    Instant createdAt;
    Instant updatedAt;

    public static Foo create(Long userId, Long targetId, FooSource source, int score) {
        Instant now = Instant.now();
        return new Foo(null, userId, targetId, source, score, now, now);
    }

    public Foo withScore(int score) {
        return new Foo(fooId, userId, targetId, source, score, createdAt, Instant.now());
    }
}
```

**특징**:
- Read 모델 없음 (`FooRead` 클래스 만들지 않음)
- 구성 파일: `Foo.java`, `FooIdentity.java`, `FooSource.java` (enum) 세 개

---

## Pattern Bar — Write + Read 분리 (단순 toRead)

**언제 쓰나**: 조회 결과에 민감 필드(내부 키, oauth 토큰 등) 제거나 구조 차이가 있는 기본 CRUD 도메인.

```java
@Value
public class Bar implements AuditFields {
    Long barId;
    String nickname;
    String email;
    String googleOauthKey;  // 민감 — Read 에서 제외
    String bio;
    Instant createdAt;
    Instant updatedAt;

    public static Bar create(String nickname, String email, String googleOauthKey, String bio) {
        Instant now = Instant.now();
        return new Bar(null, nickname, email, googleOauthKey, bio, now, now);
    }

    public Bar withProfile(String nickname, String bio) {
        return new Bar(barId, nickname, email, googleOauthKey, bio, createdAt, Instant.now());
    }

    public BarRead toRead() {
        return new BarRead(barId, nickname, email, bio, createdAt, updatedAt);
    }
}
```

```java
@Value
public class BarRead implements AuditFields {
    Long barId;
    String nickname;
    String email;
    String bio;          // googleOauthKey 없음
    Instant createdAt;
    Instant updatedAt;
}
```

**특징**:
- `toRead()` 가 **인자 없음** (self-contained)
- Read 가 Write 의 부분집합 (컬럼 추가 없이 프로젝션)
- 구성 파일: `Bar.java`, `BarRead.java`, `BarIdentity.java`

---

## Pattern Baz — 하위 VO + Read 에 JOIN/집계 필드

**언제 쓰나**: 조회 시 다른 도메인 데이터(작성자명, 통계)나 자식 컬렉션을 함께 노출하는 핵심 aggregate.

### 하위 VO

```java
@Value
public class BazLink {      // 여러 필드 묶음
    Long seriesId;
    String name;
}
```

### Write 모델

```java
@Value
public class Baz implements AuditFields {
    Long bazId;
    Long userId;
    BazType type;
    String title;
    String body;
    List<BazCategory> categories;   // enum 리스트
    List<BazTag> tags;
    BazSource source;
    Instant createdAt;
    Instant updatedAt;

    public static Baz create(Long userId, BazType type, String title, String body,
                             List<BazCategory> categories, List<BazTag> tags, BazSource source) {
        Instant now = Instant.now();
        return new Baz(null, userId, type, title, body, categories, tags, source, now, now);
    }

    public Baz withUpdate(String title, String body,
                          List<BazCategory> categories, List<BazTag> tags, BazSource source) {
        return new Baz(bazId, userId, type, title, body, categories, tags, source,
                createdAt, Instant.now());
    }

    public BazRead toRead(String authorNickname, BazStats stats, List<BazLink> links) {
        double rating = stats.getRatingCount() > 0
                ? (double) stats.getRatingSum() / stats.getRatingCount() : 0.0;
        return new BazRead(bazId, type, title, body, categories, tags,
                stats.getViewCount(), authorNickname,
                rating, stats.getRatingCount(), stats.getCommentCount(),
                links, createdAt, updatedAt, source);
    }
}
```

### Read 모델 (JOIN/집계 필드 포함)

```java
@Value
public class BazRead implements AuditFields {
    Long bazId;
    BazType type;
    String title;
    String body;
    List<BazCategory> categories;
    List<BazTag> tags;
    long views;              // ← 집계
    String author;           // ← JOIN (nickname)
    double rating;           // ← 집계 파생 (sum/count)
    int reviews;             // ← 집계
    int comments;            // ← 집계
    List<BazLink> links;     // ← JOIN (자식 컬렉션)
    Instant createdAt;
    Instant updatedAt;
    BazSource source;
}
```

**특징**:
- `toRead()` 시그니처에 **의존 데이터를 파라미터로 명시** (어떤 JOIN 이 필요한지 모델이 선언)
- Read 에 userId 같은 내부 키 대신 author 표시명
- 구성 파일: `Baz.java`, `BazRead.java`, `BazIdentity.java`, `BazStats.java` (분리 집계), `BazLink.java` (하위 VO), `BazType.java`/`BazCategory.java`/`BazTag.java`/`BazSource.java` (enum)

### 분리 Stats 모델

집계 컬럼은 빈번 업데이트로 쓰기 경쟁이 심하므로 본 테이블에서 **분리된 stats 테이블**로 관리. 모델도 별도.

```java
@Value
public class BazStats {
    Long bazId;
    long viewCount;
    long ratingSum;
    int ratingCount;
    int commentCount;

    public static BazStats init(Long bazId) {
        return new BazStats(bazId, 0, 0, 0, 0);
    }
}
```

**특징**:
- AuditFields 미구현 (audit 는 본 도메인에만)
- `init()` 팩토리로 0 초기화
- increment/decrement 같은 `withXxx` 메서드는 필요 시 추가

---

## Pattern Qux — 상태 전이 + 트리 구조

**언제 쓰나**: 댓글·알림 같이 상태 플래그 전환(hide/show, read/unread)이나 부모-자식 관계가 있는 도메인.

```java
@Value
public class Qux implements AuditFields {
    Long quxId;
    Long articleId;
    QuxArticleType articleType;
    Long userId;
    String text;
    Long parentId;          // null = 루트
    int quxOrder;
    int level;              // 0 = 루트
    int sortNumber;
    int childCount;
    boolean isHidden;       // 상태 플래그
    Instant createdAt;
    Instant updatedAt;

    public static Qux createRoot(Long articleId, QuxArticleType articleType,
                                 Long userId, String text, int quxOrder) {
        Instant now = Instant.now();
        return new Qux(null, articleId, articleType, userId, text, null,
                quxOrder, 0, 0, 0, false, now, now);
    }

    public static Qux createReply(Long articleId, QuxArticleType articleType,
                                  Long userId, String text,
                                  Long parentId, int quxOrder, int level, int sortNumber) {
        Instant now = Instant.now();
        return new Qux(null, articleId, articleType, userId, text, parentId,
                quxOrder, level, sortNumber, 0, false, now, now);
    }

    public Qux updateContent(String newText) {
        return new Qux(quxId, articleId, articleType, userId, newText, parentId,
                quxOrder, level, sortNumber, childCount, isHidden, createdAt, Instant.now());
    }

    public Qux hide() {
        return new Qux(quxId, articleId, articleType, userId, text, parentId,
                quxOrder, level, sortNumber, childCount, true, createdAt, Instant.now());
    }

    public Qux show() {
        return new Qux(quxId, articleId, articleType, userId, text, parentId,
                quxOrder, level, sortNumber, childCount, false, createdAt, Instant.now());
    }

    public QuxRead toRead(String authorNickname) {
        return new QuxRead(quxId, articleId, articleType, userId, authorNickname, text,
                parentId, quxOrder, level, sortNumber, childCount, isHidden,
                createdAt, updatedAt);
    }
}
```

**특징**:
- `createRoot()` vs `createReply()` — **생성 시 분기** 를 팩토리 이름으로 드러냄
- `hide()` / `show()` — **상태 전이는 동사 메서드** (withIsHidden 금지)
- soft-delete 미사용 (`isHidden` 이 대체)
- 구성 파일: `Qux.java`, `QuxRead.java`, `QuxIdentity.java`, `QuxArticleType.java` (enum)

---

## 패턴 선택 가이드

| 도메인 특성 | Pattern |
|---|---|
| Upsert 만 필요, 조회 시 추가 정보 없음 | Foo |
| 기본 CRUD, Read 는 Write 의 투영 | Bar |
| 조회 시 JOIN/집계 필요, 하위 VO 존재 | Baz |
| 상태 전이·트리·숨김 로직 있음 | Qux |

두 패턴이 섞이면 Baz 를 기본으로 하고 Qux 의 상태 전이 메서드를 추가하는 식으로 합성.

---

## 파일 구성 요약

```
model/src/main/java/{packagePath}/model/{domain}/
├── {Domain}.java                ← Write 모델 (implements AuditFields)
├── {Domain}Read.java            ← Read 모델 (Bar/Baz/Qux 만)
├── {Domain}Identity.java        ← Identity VO (필수)
├── {Domain}Stats.java           ← 분리 통계 (Baz 만)
├── {Name}Link.java / {Name}VO   ← 하위 VO (필요 시)
└── {Domain}Category.java
    {Domain}Type.java
    ...                          ← enum (필요 시)
```

---

## 공통 금지 사항

- `@Getter`/`@Setter` 단독 사용 금지 (항상 `@Value`)
- `@Builder` 사용 금지 (팩토리로 의도 표현)
- public 생성자 노출 금지 (`@Value` 자동 생성만 사용)
- Read 모델에서 `withXxx` / `toWrite()` 금지 (단방향 투영)
- enum 에 `description` 같은 필드 과도 추가 금지 — 표시 문자열은 프론트 책임
- Write 모델에서 List 인자를 **immutable 가정** — 호출자가 `List.copyOf()` 책임
