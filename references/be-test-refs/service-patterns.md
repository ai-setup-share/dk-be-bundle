# Service 단위 테스트 패턴 레퍼런스

SKILL.md의 패턴 레터에서 참조하는 코드 예제 모음. 1 레터 = 1 예제.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## A. Usecase — happy path

```java
@Test
void should_createSetup_when_validRequest() {
    // given
    when(userRepository.findById(new UserIdentity(1L)))
            .thenReturn(Optional.of(mock(User.class)));
    when(setupWriter.create(command)).thenReturn(new SetupIdentity(100L));

    // when
    SetupIdentity result = sut.execute(1L, command);

    // then
    assertThat(result.getSetupId()).isEqualTo(100L);
    verify(userStatsRepository).increasePostCount(1L);
}
```

## B. Usecase — 유저 미존재

```java
@Test
void should_throwUserNotFound_when_userNotExists() {
    // given
    when(userRepository.findById(any(UserIdentity.class))).thenReturn(Optional.empty());

    // when & then
    assertThatThrownBy(() -> sut.execute(999L, command))
            .isInstanceOf(UserNotFoundException.class);
    verify(setupWriter, never()).create(any());
}
```

## C. Usecase — 엔티티 미존재

```java
@Test
void should_throwSetupNotFound_when_setupNotExists() {
    // given
    when(userRepository.findById(any())).thenReturn(Optional.of(mock(User.class)));
    when(setupRepository.findWriteById(any())).thenReturn(Optional.empty());

    // when & then
    assertThatThrownBy(() -> sut.execute(userId, command))
            .isInstanceOf(SetupNotFoundException.class);
}
```

## D. Usecase — 소유권 실패

```java
@Test
void should_throwForbidden_when_notOwner() {
    // given
    Setup setup = mock(Setup.class);
    when(setup.getUserId()).thenReturn(999L);
    when(userRepository.findById(any())).thenReturn(Optional.of(mock(User.class)));
    when(setupRepository.findWriteById(any())).thenReturn(Optional.of(setup));

    // when & then
    assertThatThrownBy(() -> sut.execute(1L, command))
            .isInstanceOf(ForbiddenException.class);
}
```

## E. Usecase — 비즈니스 규칙 위반

```java
@Test
void should_throwConflict_when_alreadyBookmarked() {
    // given
    when(userRepository.findById(any())).thenReturn(Optional.of(mock(User.class)));
    when(savedPostReader.isBookmarked(userId, setupId)).thenReturn(true);

    // when & then
    assertThatThrownBy(() -> sut.execute(userId, setupId))
            .isInstanceOf(ConflictException.class);
}
```

## F. Reader — 단건 정상

```java
@Test
void should_returnSetupRead_when_exists() {
    // given
    SetupRead expected = mock(SetupRead.class);
    when(setupRepository.findById(new SetupIdentity(1L))).thenReturn(Optional.of(expected));

    // when & then
    assertThat(sut.getById(new SetupIdentity(1L))).isSameAs(expected);
}
```

## G. Reader — 단건 미존재

```java
@Test
void should_throwNotFound_when_setupNotExists() {
    // given
    when(setupRepository.findById(any())).thenReturn(Optional.empty());

    // when & then
    assertThatThrownBy(() -> sut.getById(new SetupIdentity(999L)))
            .isInstanceOf(SetupNotFoundException.class);
}
```

## H. Reader — 목록

```java
@Test
void should_returnList_when_dataExists() {
    when(setupRepository.findByUserId(1L, 0, 10))
            .thenReturn(List.of(mock(SetupRead.class), mock(SetupRead.class)));
    assertThat(sut.getByUserId(1L, 0, 10)).hasSize(2);
}

@Test
void should_returnEmptyList_when_noData() {
    when(setupRepository.findByUserId(999L, 0, 10)).thenReturn(List.of());
    assertThat(sut.getByUserId(999L, 0, 10)).isEmpty();
}
```

## I. Reader — 카운트

```java
@Test
void should_returnCount_when_called() {
    when(setupRepository.countByUserId(1L)).thenReturn(5L);
    assertThat(sut.countByUserId(1L)).isEqualTo(5L);
}
```

## J. Reader — Optional 반환

```java
@Test
void should_returnScore_when_ratingExists() {
    Rating rating = mock(Rating.class);
    when(rating.getScore()).thenReturn(4);
    when(ratingRepository.findBySetupIdAndUserId(any(), any())).thenReturn(Optional.of(rating));
    assertThat(sut.getScore(1L, 2L).getAsInt()).isEqualTo(4);
}

@Test
void should_returnEmpty_when_noRating() {
    when(ratingRepository.findBySetupIdAndUserId(any(), any())).thenReturn(Optional.empty());
    assertThat(sut.getScore(1L, 2L)).isEmpty();
}
```

## K. Writer — 생성 + 부수효과

```java
@Test
void should_saveAndInitStats_when_create() {
    Setup saved = mock(Setup.class);
    when(saved.getSetupId()).thenReturn(100L);
    when(setupRepository.save(any(Setup.class))).thenReturn(saved);

    assertThat(sut.create(command).getSetupId()).isEqualTo(100L);
    verify(setupStatsRepository).save(any(SetupStats.class));
}
```

## L. Writer — 수정 (조회→변환→저장)

```java
@Test
void should_updateSetup_when_validCommand() {
    Setup existing = Setup.create(1L, SetupType.SETUP, "기존", "요약", "본문", null,
            List.of(SetupCategory.VIBE_CODING), List.of(SetupTag.CLAUDE));
    when(setupRepository.findWriteById(any())).thenReturn(Optional.of(existing));
    when(setupRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));

    sut.update(command);
    verify(setupRepository).save(any(Setup.class));
}
```

## M. Writer — 미존재 예외

```java
@Test
void should_throwNotFound_when_updateMissing() {
    when(setupRepository.findWriteById(any())).thenReturn(Optional.empty());

    assertThatThrownBy(() -> sut.update(command))
            .isInstanceOf(SetupNotFoundException.class);
    verify(setupRepository, never()).save(any());
}
```

## N. Writer — 상태 전이 (hide/show)

```java
@Test
void should_hideComment_when_notHidden() {
    Comment comment = Comment.createRoot(1L, ArticleType.SETUP, 1L, "내용", 1);
    when(commentRepository.findWriteById(any())).thenReturn(Optional.of(comment));

    sut.hide(new CommentIdentity(1L));
    verify(commentRepository).save(argThat(Comment::isHidden));
}

@Test
void should_throwWrongComment_when_alreadyHidden() {
    Comment comment = Comment.createRoot(1L, ArticleType.SETUP, 1L, "내용", 1).hide();
    when(commentRepository.findWriteById(any())).thenReturn(Optional.of(comment));

    assertThatThrownBy(() -> sut.hide(new CommentIdentity(1L)))
            .isInstanceOf(WrongCommentException.class);
}
```

## O. Writer — 분기 로직 (루트 vs 답글)

```java
@Test
void should_createRootComment_when_noParent() {
    CommentCreateCommand cmd = new CommentCreateCommand(1L, ArticleType.SETUP, 1L, "내용", null);
    when(commentRepository.getMaxCommentOrder(1L, ArticleType.SETUP)).thenReturn(3);
    when(commentRepository.save(any())).thenAnswer(inv -> inv.getArgument(0));
    when(commentRepository.findById(any())).thenReturn(Optional.of(mock(CommentRead.class)));

    sut.create(cmd);
    verify(commentRepository).save(argThat(c -> c.getCommentOrder() == 4 && c.getLevel() == 0));
}
```

## P. Usecase — 분기별 TC (신규 vs 수정)

```java
@Test
void should_addRating_when_firstTime() {
    when(ratingReader.getScore(anyLong(), anyLong())).thenReturn(OptionalInt.empty());
    // ... setup omitted
    sut.execute(1L, command);
    verify(userStatsRepository).addRating(anyLong(), anyInt());
    verify(userStatsRepository, never()).updateRating(anyLong(), anyInt(), anyInt());
}

@Test
void should_updateRating_when_alreadyRated() {
    when(ratingReader.getScore(anyLong(), anyLong())).thenReturn(OptionalInt.of(3));
    // ... setup omitted
    sut.execute(1L, command);
    verify(userStatsRepository).updateRating(anyLong(), eq(3), anyInt());
    verify(userStatsRepository, never()).addRating(anyLong(), anyInt());
}
```

## Q. Usecase — 다중 메서드

```java
@Test
void should_returnMyPosts_when_validUser() {
    when(userRepository.findById(any())).thenReturn(Optional.of(mock(User.class)));
    when(setupReader.getByUserId(1L, 0, 10)).thenReturn(List.of(mock(SetupRead.class)));
    assertThat(sut.getMyPosts(1L, 0, 10)).hasSize(1);
}
```
