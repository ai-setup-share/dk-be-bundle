# API Controller 단위 테스트 패턴 레퍼런스

SKILL.md의 패턴 레터에서 참조하는 코드 예제 모음. 1 레터 = 1 예제.

> **이 문서의 성격 (공통)**
>
> - 기존 서비스 패턴에서 도출된 **참고용** 자료다. 절대 규칙이 아니라 결과물의 경향을 맞추기 위한 가이드.
> - 요구사항이 이 패턴들로 해결되지 않으면, 주변 컨텍스트(폴더/파일/연관 모듈)를 **한 번 더 탐색·검토** 후 필요한 변형을 적용한다.
> - 여러 레퍼런스로 해결 가능하면 **가장 간단한 방식**을 택한다.

---

## A. GET — 단건 정상 (200)

```java
@Test
void should_returnOk_when_setupExists() {
    // given
    SetupRead read = mock(SetupRead.class);
    when(read.getSetupId()).thenReturn(1L);
    when(read.getType()).thenReturn(SetupType.SETUP);
    when(read.getCategories()).thenReturn(List.of());
    when(read.getTags()).thenReturn(List.of());
    when(read.getSeriesList()).thenReturn(List.of());
    when(setupDetailUsecase.execute(new SetupIdentity(1L))).thenReturn(read);

    // when
    ResponseEntity<SetupDetailResponse> response = controller.getSetup(1L);

    // then
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    assertThat(response.getBody()).isNotNull();
}
```

## B. GET — 단건 미존재 (404 검증)

```java
@Test
void should_throw404_when_setupNotFound() {
    // given
    when(setupDetailUsecase.execute(any()))
            .thenThrow(new SetupNotFoundException("Setup not found: 999"));

    // when & then
    assertThatThrownBy(() -> controller.getSetup(999L))
            .isInstanceOf(SetupNotFoundException.class)
            .satisfies(ex -> {
                HttpException httpEx = (HttpException) ex;
                assertThat(httpEx.httpStatusCode()).isEqualTo(404);
                assertThat(httpEx.httpErrorMessage()).isEqualTo("해당 리소스를 찾을 수 없습니다");
            });
}
```

## C. GET — 목록 정상 (200)

```java
@Test
void should_returnList_when_validParams() {
    // given
    when(setupReader.getAll(anyString(), anyString(), anyList(), anyList(), anyInt(), anyInt()))
            .thenReturn(List.of(mock(SetupRead.class)));
    when(setupReader.countAll(anyString(), anyList(), anyList())).thenReturn(1L);

    // when
    ResponseEntity<SetupListResponse> response = controller.getSetups(
            "popular", "", "", "", 0, 20);

    // then
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    assertThat(response.getBody().getData()).hasSize(1);
    assertThat(response.getBody().getTotal()).isEqualTo(1L);
}
```

## D. POST — 생성 정상 (201)

```java
@Test
void should_returnCreated_when_validRequest() {
    // given
    when(setupCreateUsecase.execute(eq(userId), any(SetupCreateCommand.class)))
            .thenReturn(new SetupIdentity(100L));

    // when
    ResponseEntity<SetupCreateResponse> response = controller.createSetup(sessionUser, request);

    // then
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
    assertThat(response.getBody().getId()).isEqualTo(100L);
}
```

## E. PUT — 수정 정상 (200)

```java
@Test
void should_returnOk_when_updateValid() {
    // given
    Setup updated = mock(Setup.class);
    when(updated.getSetupId()).thenReturn(1L);
    when(setupUpdateUsecase.execute(eq(userId), any(SetupUpdateCommand.class)))
            .thenReturn(updated);

    // when
    ResponseEntity<SetupCreateResponse> response = controller.updateSetup(1L, sessionUser, request);

    // then
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
    assertThat(response.getBody().getId()).isEqualTo(1L);
}
```

## F. DELETE — 삭제 정상 (204)

```java
@Test
void should_returnNoContent_when_deleteSuccess() {
    // given
    doNothing().when(setupDeleteUsecase).execute(userId, new SetupIdentity(1L));

    // when
    ResponseEntity<Void> response = controller.deleteSetup(1L, sessionUser);

    // then
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NO_CONTENT);
    verify(setupDeleteUsecase).execute(userId, new SetupIdentity(1L));
}
```

## G. 예외 — 소유권 불일치 (403)

```java
@Test
void should_throw403_when_notOwner() {
    // given
    when(setupUpdateUsecase.execute(eq(userId), any()))
            .thenThrow(new ForbiddenException("Not owner of setup: 1"));

    // when & then
    assertThatThrownBy(() -> controller.updateSetup(1L, sessionUser, request))
            .isInstanceOf(ForbiddenException.class)
            .satisfies(ex -> {
                HttpException httpEx = (HttpException) ex;
                assertThat(httpEx.httpStatusCode()).isEqualTo(403);
                assertThat(httpEx.httpErrorMessage()).isEqualTo("접근 권한이 없습니다");
            });
}
```

## H. 예외 — 잘못된 입력 (400)

```java
@Test
void should_throw400_when_invalidCategory() {
    // when & then — Controller 내부 parseCategories에서 발생
    assertThatThrownBy(() -> controller.getSetups("popular", "", "INVALID", "", 0, 20))
            .isInstanceOf(InvalidInputException.class)
            .satisfies(ex -> {
                HttpException httpEx = (HttpException) ex;
                assertThat(httpEx.httpStatusCode()).isEqualTo(400);
                assertThat(httpEx.httpErrorMessage()).isEqualTo("잘못된 요청입니다");
            });
}
```

## I. 예외 — 충돌 (409)

```java
@Test
void should_throw409_when_alreadyBookmarked() {
    // given
    doThrow(new ConflictException("Already bookmarked: setupId=1"))
            .when(bookmarkUsecase).execute(eq(userId), anyLong());

    // when & then
    assertThatThrownBy(() -> controller.bookmark(1L, sessionUser))
            .isInstanceOf(ConflictException.class)
            .satisfies(ex -> {
                HttpException httpEx = (HttpException) ex;
                assertThat(httpEx.httpStatusCode()).isEqualTo(409);
                assertThat(httpEx.httpErrorMessage()).isEqualTo("이미 존재하는 리소스입니다");
            });
}
```

## J. SessionUser 전달 — userId 추출

```java
// SessionUser fixture
private final SessionUser sessionUser = new SessionUser(1L);
private final Long userId = 1L;

@Test
void should_passUserId_when_authenticated() {
    // given
    when(setupCreateUsecase.execute(eq(userId), any())).thenReturn(new SetupIdentity(1L));

    // when
    controller.createSetup(sessionUser, request);

    // then
    verify(setupCreateUsecase).execute(eq(userId), any());
}
```

## K. Request DTO 생성 — Reflection 헬퍼

```java
private <T> T createRequest(Class<T> clazz, Map<String, Object> fields) {
    try {
        T instance = clazz.getDeclaredConstructor().newInstance();
        for (var entry : fields.entrySet()) {
            Field field = clazz.getDeclaredField(entry.getKey());
            field.setAccessible(true);
            field.set(instance, entry.getValue());
        }
        return instance;
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

## L. void 반환 Usecase — verify 패턴

```java
@Test
void should_callUsecase_when_validRequest() {
    // given
    doNothing().when(usecase).execute(userId, command);

    // when
    ResponseEntity<Void> response = controller.doSomething(sessionUser, request);

    // then
    assertThat(response.getStatusCode()).isEqualTo(HttpStatus.NO_CONTENT);
    verify(usecase).execute(eq(userId), any());
}
```
