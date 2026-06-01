# Plan: Notification CRUD (6 APIs)

## 요청 스펙

알림 도메인 신규 생성 + CRUD API 6개.

| # | Method | URL | 설명 |
|---|--------|-----|------|
| 1 | GET | /api/notifications?page&size | 알림 목록 조회 |
| 2 | GET | /api/notifications/unread-count | 안 읽은 개수 |
| 3 | PUT | /api/notifications/{id}/read | 개별 읽음 처리 |
| 4 | PUT | /api/notifications/read-all | 모두 읽기 |
| 5 | DELETE | /api/notifications/{id} | 개별 삭제 |
| 6 | DELETE | /api/notifications | 모두 삭제 |

인증: 전부 필요 (@AuthenticationPrincipal)

---

## Phase 1: 평가

### assess-api

**충족 여부: 미충족** — Notification 관련 엔드포인트 없음. 전부 신규.

**usecase 요청:**

```
신규 요청:
  - NotificationReader (조회)
    - List<NotificationRead> getByReceiver(Long userId, int page, int size)
    - long countByReceiver(Long userId)  // total용
    - long countUnread(Long userId)
  - NotificationWriter (명령)
    - void markAsRead(NotificationIdentity identity, Long requestUserId)
    - void markAllAsRead(Long requestUserId)
    - void delete(NotificationIdentity identity, Long requestUserId)
    - void deleteAll(Long requestUserId)
    - (create는 별도 계획에서 추가 예정)
```

**model 요청:**

```
신규 요청:
  - 대상: Notification (신규)
  - NotificationType enum (REACTION_1, REACTION_10, REACTION_100, COMMENT, REPLY)
  - NotificationTargetType enum (SETUP, AIDOC)
  - 필드: type, targetType, targetId, receiverUserId, actorUserId, actorName, preview, isRead
  - actorName/preview는 생성 시점 스냅샷 (이후 불변)
```

**API 계획:**

```
- [신규] GET /api/notifications
  - Params: page (default 0), size (default 10)
  - Response: NotificationListResponse { data: NotificationResponse[], total: long, hasMore: boolean }
  - 인증: 필요
  - 사용 usecase: NotificationReader.getByReceiver, NotificationReader.countByReceiver

- [신규] GET /api/notifications/unread-count
  - Response: UnreadCountResponse { count: long }
  - 인증: 필요
  - 사용 usecase: NotificationReader.countUnread

- [신규] PUT /api/notifications/{id}/read
  - Response: 200 OK (본문 없음)
  - 인증: 필요
  - 사용 usecase: NotificationWriter.markAsRead

- [신규] PUT /api/notifications/read-all
  - Response: 200 OK (본문 없음)
  - 인증: 필요
  - 사용 usecase: NotificationWriter.markAllAsRead

- [신규] DELETE /api/notifications/{id}
  - Response: 204 No Content
  - 인증: 필요
  - 사용 usecase: NotificationWriter.delete

- [신규] DELETE /api/notifications
  - Response: 204 No Content
  - 인증: 필요
  - 사용 usecase: NotificationWriter.deleteAll
```

---

### assess-usecase

**충족 여부: 미충족** — Notification 관련 usecase 없음. 전부 신규.

**infrastructure 요청:**

```
신규 요청:
  - NotificationRepository
    - Optional<Notification> findById(NotificationIdentity identity)
    - List<NotificationRead> findByReceiverUserId(Long userId, int page, int size)
    - long countByReceiverUserId(Long userId)
    - long countUnreadByReceiverUserId(Long userId)
    - Notification save(Notification notification)
    - void markAsRead(NotificationIdentity identity)
    - void markAllAsRead(Long receiverUserId)
    - void deleteById(NotificationIdentity identity)
    - void deleteAllByReceiverUserId(Long receiverUserId)
```

**model 요청:**

```
상위(assess-api)에서 도출한 model 요청과 동일.
(NotificationCreateCommand는 별도 계획에서 추가 예정)
```

**usecase 계획:**

```
- [신규] NotificationReader
  - getByReceiver(Long userId, int page, int size) → List<NotificationRead>
  - countByReceiver(Long userId) → long
  - countUnread(Long userId) → long
  - 의존: NotificationRepository
  - 트랜잭션: 없음

- [신규] NotificationWriter
  - markAsRead(NotificationIdentity identity, Long requestUserId) → void
    - findById → 수신자 확인 → markAsRead
  - markAllAsRead(Long requestUserId) → void
    - bulk 처리
  - delete(NotificationIdentity identity, Long requestUserId) → void
    - findById → 수신자 확인 → deleteById
  - deleteAll(Long requestUserId) → void
    - bulk 처리
  - 의존: NotificationRepository
  - 트랜잭션: 모든 public 메서드에 @Transactional
  - (create는 별도 계획에서 추가 예정)
```

---

### assess-infra

**충족 여부: 미충족** — NotificationRepository 없음. 신규 생성.

**model 요청:**

```
상위에서 도출한 model 요청과 동일.
infra 추가 관점:
  - NotificationIdentity (Long notificationId)
  - Notification Write 모델: save/findById용
  - NotificationRead: findByReceiverUserId 반환용
```

**infrastructure 계획:**

```
- [신규] NotificationRepository
  - findById(NotificationIdentity) → Optional<Notification>
  - findByReceiverUserId(Long userId, int page, int size) → List<NotificationRead>
  - countByReceiverUserId(Long userId) → long
  - countUnreadByReceiverUserId(Long userId) → long
  - save(Notification) → Notification
  - markAsRead(NotificationIdentity) → void  // 원자적 UPDATE
  - markAllAsRead(Long receiverUserId) → void  // 원자적 bulk UPDATE
  - deleteById(NotificationIdentity) → void  // hard delete
  - deleteAllByReceiverUserId(Long receiverUserId) → void  // hard delete
```

---

### assess-model

**충족 여부: 미충족** — Notification 도메인 없음. 전부 신규.

**model 계획:**

```
- [신규] NotificationType (enum)
  - REACTION_1, REACTION_10, REACTION_100, COMMENT, REPLY

- [신규] NotificationTargetType (enum)
  - SETUP, AIDOC

- [신규] Notification (Write)
  - Long notificationId
  - NotificationType type
  - NotificationTargetType targetType
  - Long targetId
  - Long receiverUserId
  - Long actorUserId        // nullable
  - String actorName        // nullable (리액션은 null)
  - String preview
  - boolean isRead
  - Instant createdAt
  - 팩토리: create(NotificationType, NotificationTargetType, Long targetId,
                   Long receiverUserId, Long actorUserId, String actorName, String preview)
    → isRead=false, createdAt=Instant.now()
  - 상태변경: withRead(boolean isRead) → 새 Notification

- [신규] NotificationRead (Read)
  - Notification 필드 전부 (JOIN 없음 — actorName/preview가 스냅샷이므로)

- [신규] NotificationIdentity
  - Long notificationId
```

---

## Phase 2: 구현 계획

### 평가 결과 요약

| 레이어 | 충족 여부 | 비고 |
|--------|----------|------|
| api | 미충족 | 6개 엔드포인트 신규 |
| usecase | 미충족 | NotificationReader + NotificationWriter 신규 |
| infra | 미충족 | NotificationRepository 신규 |
| model | 미충족 | Notification 도메인 전체 신규 |

**케이스 A: 새 도메인** — 전 레이어 구현.

### 설계 결정 사항 (확정)

**1. BaseEntity 상속 (soft delete)**
- 알림도 BaseEntity 상속. is_deleted, deleted_at, created_at, updated_at 포함.
- delete = soft delete.

**2. isRead 상태 변경은 원자적 UPDATE**
- markAsRead: `UPDATE notification SET is_read = true WHERE id = :id`
- markAllAsRead: `UPDATE notification SET is_read = true WHERE receiver_user_id = :userId AND is_read = false`
- 도메인 모델 로드 없이 직접 UPDATE.

**3. 이 계획의 범위: 조회 + 삭제 + 읽음 처리만**
- 알림 생성(create)은 별도 계획으로 분리.
- NotificationWriter에 create 메서드 없음.
- NotificationCreateCommand 없음.

**4. Instant 사용 (스펙의 LocalDateTime 대신)**
- 프로젝트 컨벤션: 모든 timestamp는 Instant.

**5. 응답 형식 — 프로젝트 컨벤션**
- PUT read → 200 OK (본문 없음), DELETE → 204 No Content.

### 구현 순서

```
1. add-model
   - NotificationType, NotificationTargetType enum
   - Notification (Write), NotificationRead (Read), NotificationIdentity

2. add-infrastructure
   - NotificationRepository 인터페이스

3. add-entity-specs
   - NotificationEntity (BaseEntity 미상속)
   - DDL: notification 테이블 (H2 + MySQL)

4. add-jdbc-query
   - NotificationEntityRepository
   - NotificationJdbcRepository

5. add-usecase
   - NotificationReader + DefaultNotificationReader
   - NotificationWriter + DefaultNotificationWriter

6. add-api
   - NotificationApiController
   - NotificationResponse, NotificationListResponse, UnreadCountResponse
```

---

## Phase 3: 구현 진행

### 1. add-model (완료)

생성된 파일:
- `model/.../notification/NotificationType.java` — enum (REACTION_1, REACTION_10, REACTION_100, COMMENT, REPLY)
- `model/.../notification/NotificationTargetType.java` — enum (SETUP, AIDOC)
- `model/.../notification/NotificationIdentity.java` — @Value, Long notificationId
- `model/.../notification/Notification.java` — Write 모델 (@Value, AuditProps)
  - create() 팩토리, withRead() 상태변경, toRead() 변환
- `model/.../notification/NotificationRead.java` — Read 모델 (Notification과 동일 필드, JOIN 없음)

specs/models.md 업데이트 완료. 빌드 성공.
