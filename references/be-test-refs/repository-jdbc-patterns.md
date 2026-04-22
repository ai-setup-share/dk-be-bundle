# repository-jdbc 테스트 패턴 레퍼런스

카테고리별 테스트 작성 패턴. `be-test-repository-jdbc` 스킬의 Phase 3에서 참조한다.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## A. Entity↔Domain 변환

save → findById → 모든 필드 왕복 검증. **모든 Repository에 필수**.

```java
@Test
void should_returnAllFields_when_savedAndFoundById() {
    // given
    Setup toSave = Setup.create(testUserId, SetupType.ARTICLE,
            "제목", "요약", "본문", "https://link.com",
            List.of(SetupCategory.VIBE_CODING), List.of(SetupTag.CLAUDE));

    // when
    Setup saved = repository.save(toSave);
    Optional<Setup> found = repository.findWriteById(
            new SetupIdentity(saved.getSetupId()));

    // then
    assertThat(found).isPresent();
    Setup result = found.get();
    assertThat(result.getSetupId()).isNotNull();
    assertThat(result.getUserId()).isEqualTo(testUserId);
    assertThat(result.getTitle()).isEqualTo("제목");
    assertThat(result.getType()).isEqualTo(SetupType.ARTICLE);
}
```

---

## B. Collection 변환 (@MappedCollection)

Collection 자식 테이블의 save/load 왕복 검증.

```java
// Setup — categories, tags
@Test
void should_persistCollections_when_savedWithCategoriesAndTags() {
    // given
    Setup toSave = Setup.create(testUserId, SetupType.ARTICLE,
            "제목", "요약", "본문", null,
            List.of(SetupCategory.VIBE_CODING, SetupCategory.DEV_EFFICIENCY),
            List.of(SetupTag.CLAUDE, SetupTag.CURSOR));

    // when
    Setup saved = repository.save(toSave);
    Optional<Setup> found = repository.findWriteById(
            new SetupIdentity(saved.getSetupId()));

    // then
    assertThat(found).isPresent();
    assertThat(found.get().getCategories())
            .containsExactlyInAnyOrder(SetupCategory.VIBE_CODING, SetupCategory.DEV_EFFICIENCY);
    assertThat(found.get().getTags())
            .containsExactlyInAnyOrder(SetupTag.CLAUDE, SetupTag.CURSOR);
}

// Series — items (SeriesItemCollection)
@Test
void should_persistItems_when_savedWithItems() {
    // given — Setup 2건 선행 생성
    Series series = Series.create(testUserId, "시리즈명");
    Series saved = repository.save(series);

    // when
    repository.replaceItems(new SeriesIdentity(saved.getSeriesId()),
            List.of(new SetupIdentity(setupId1), new SetupIdentity(setupId2)));

    // then
    SeriesRead found = repository.findById(new SeriesIdentity(saved.getSeriesId())).get();
    assertThat(found.getItems()).hasSize(2);
}
```

---

## C. JOIN 결과 검증

@Query LEFT JOIN으로 가져오는 필드 검증. User 선행 데이터 필요.

```java
// Comment — nickname from users
@Test
void should_returnNickname_when_foundByIdWithUserJoin() {
    // given — User 선행 생성 (@BeforeEach)
    Comment comment = Comment.create(testUserId, articleId, "SETUP", null, "댓글 내용");
    Comment saved = commentRepository.save(comment);

    // when
    Optional<CommentRead> found = commentRepository.findById(
            new CommentIdentity(saved.getCommentId()));

    // then
    assertThat(found).isPresent();
    assertThat(found.get().getNickname()).isEqualTo("TestUser");
    assertThat(found.get().getText()).isEqualTo("댓글 내용");
}

// Setup — nickname + viewCount + ratingSum from users/setup_stats
@Test
void should_returnJoinedFields_when_foundByIdWithDetails() {
    // given — User, SetupStats 선행 생성
    Setup setup = createAndSaveSetup();
    setupStatsRepository.save(SetupStats.init(setup.getSetupId()));

    // when
    Optional<SetupRead> found = repository.findById(
            new SetupIdentity(setup.getSetupId()));

    // then
    assertThat(found).isPresent();
    assertThat(found.get().getNickname()).isEqualTo("TestUser");
    assertThat(found.get().getViewCount()).isEqualTo(0);
}
```

---

## D. @Modifying 쿼리

실행 → 재조회 → 변경값 확인.

```java
// SetupStats — increaseViewCount
@Test
void should_incrementViewCount_when_increaseViewCountCalled() {
    // given
    SetupStats stats = setupStatsRepository.save(SetupStats.init(setupId));

    // when
    setupStatsRepository.increaseViewCount(new SetupIdentity(setupId), 5);

    // then
    SetupStats found = setupStatsRepository.findBySetupId(setupId).get();
    assertThat(found.getViewCount()).isEqualTo(5);
}

// 누적 검증
@Test
void should_accumulate_when_increaseViewCountCalledMultipleTimes() {
    SetupStats stats = setupStatsRepository.save(SetupStats.init(setupId));

    setupStatsRepository.increaseViewCount(new SetupIdentity(setupId), 3);
    setupStatsRepository.increaseViewCount(new SetupIdentity(setupId), 7);

    SetupStats found = setupStatsRepository.findBySetupId(setupId).get();
    assertThat(found.getViewCount()).isEqualTo(10);
}

// Comment — incrementSortNumbersAbove (조건부 UPDATE)
@Test
void should_incrementOnlyMatchingRows_when_incrementSortNumbersAboveCalled() {
    // given — commentOrder=1인 댓글 3개 (sortNumber: 1, 2, 3)
    save(commentOrder=1, sortNumber=1);
    save(commentOrder=1, sortNumber=2);
    save(commentOrder=1, sortNumber=3);

    // when — sortNumber > 1인 것만 +1
    commentRepository.incrementSortNumbersAbove(articleId, "SETUP", 1, 1);

    // then — 재조회로 검증
    // sortNumber=1 → 변경 안 됨
    // sortNumber=2 → 3
    // sortNumber=3 → 4
}
```

---

## E. 동적 SQL (SetupJdbcRepository 전용)

JdbcTemplate 기반 동적 필터링/정렬 테스트.

```java
// 빈 필터 — 전체 조회
@Test
void should_returnAll_when_noFilters() {
    saveSetups(3);

    List<SetupRead> result = repository.findAll("recent", "", List.of(), List.of(), 0, 20);

    assertThat(result).hasSize(3);
}

// 카테고리 필터
@Test
void should_filterByCategory_when_categoryProvided() {
    saveSetup(categories = [VIBE_CODING]);
    saveSetup(categories = [DEV_EFFICIENCY]);

    List<SetupRead> result = repository.findAll(
            "recent", "", List.of(SetupCategory.VIBE_CODING), List.of(), 0, 20);

    assertThat(result).hasSize(1);
    assertThat(result.get(0).getCategories()).contains(SetupCategory.VIBE_CODING);
}

// 검색어 필터
@Test
void should_filterByQuery_when_queryProvided() {
    saveSetup(title = "Spring Boot 가이드");
    saveSetup(title = "React 입문");

    List<SetupRead> result = repository.findAll(
            "recent", "Spring", List.of(), List.of(), 0, 20);

    assertThat(result).hasSize(1);
}

// 정렬
@Test
void should_sortByPopularity_when_sortIsPopular() {
    // viewCount 다른 Setup 2건 생성
    Long id1 = saveSetup(); setViewCount(id1, 10);
    Long id2 = saveSetup(); setViewCount(id2, 50);

    List<SetupRead> result = repository.findAll(
            "popular", "", List.of(), List.of(), 0, 20);

    assertThat(result.get(0).getViewCount()).isGreaterThan(result.get(1).getViewCount());
}

// countAll도 동일 필터로 테스트
@Test
void should_countFiltered_when_categoryProvided() {
    saveSetup(categories = [VIBE_CODING]);
    saveSetup(categories = [VIBE_CODING]);
    saveSetup(categories = [DEV_EFFICIENCY]);

    long count = repository.countAll("", List.of(SetupCategory.VIBE_CODING), List.of());

    assertThat(count).isEqualTo(2);
}
```

---

## F. 페이징

```java
@Test
void should_returnPagedResults_when_pageAndSizeProvided() {
    for (int i = 0; i < 5; i++) {
        saveSetup();
    }

    List<SetupRead> page1 = repository.findAll("recent", "", List.of(), List.of(), 0, 3);
    List<SetupRead> page2 = repository.findAll("recent", "", List.of(), List.of(), 1, 3);

    assertThat(page1).hasSize(3);
    assertThat(page2).hasSize(2);
}
```

---

## G. 집계 함수

```java
// Comment — findMaxCommentOrder
@Test
void should_returnMaxOrder_when_commentsExist() {
    save(commentOrder = 3);
    save(commentOrder = 7);
    save(commentOrder = 5);

    int max = commentRepository.getMaxCommentOrder(articleId, "SETUP");
    assertThat(max).isEqualTo(7);
}

@Test
void should_returnZero_when_noComments() {
    int max = commentRepository.getMaxCommentOrder(999L, "SETUP");
    assertThat(max).isEqualTo(0);
}

// Comment — countByArticle
@Test
void should_countByArticle_when_commentsExist() {
    save(articleId = 100L);
    save(articleId = 100L);
    save(articleId = 200L);

    long count = commentRepository.countByArticle(100L, "SETUP");
    assertThat(count).isEqualTo(2);
}
```

---

## H. 단건 조회 (존재 / 미존재)

```java
@Test
void should_returnPresent_when_existingId() {
    Rating saved = repository.save(sampleRating);

    Optional<Rating> found = repository.findBySetupIdAndUserId(
            saved.getSetupId(), saved.getUserId());

    assertThat(found).isPresent();
}

@Test
void should_returnEmpty_when_nonExistingId() {
    Optional<Rating> found = repository.findBySetupIdAndUserId(999L, 999L);
    assertThat(found).isEmpty();
}
```

---

## I. 삭제

모든 Entity 가 `isDeleted`/`deletedAt` 필드를 가지므로 정석 soft-delete 를 쓰는 도메인에서는 아래 패턴을 따른다 (hide/show 류·upsert-only 류는 해당 없음).

```java
@Test
void should_notFindById_when_deleted() {
    Setup saved = repository.save(sampleSetup);
    SetupIdentity identity = new SetupIdentity(saved.getSetupId());

    repository.deleteById(identity);

    assertThat(repository.findWriteById(identity)).isEmpty();
}
```

---

## J. 숨김 처리 (Comment 전용)

```java
// isHidden=true인 댓글은 텍스트가 마스킹되어 반환
@Test
void should_maskText_when_commentIsHidden() {
    Comment comment = Comment.create(testUserId, articleId, "SETUP", null, "원본 텍스트");
    Comment saved = commentRepository.save(comment);

    // hide 처리 (직접 SQL 또는 별도 메서드)

    Optional<CommentRead> found = commentRepository.findById(
            new CommentIdentity(saved.getCommentId()));

    assertThat(found.get().getText()).isNotEqualTo("원본 텍스트");
    assertThat(found.get().getIsHidden()).isTrue();
}
```
