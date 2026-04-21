# Entity References

add-entity-specs 스킬이 참고하는 레퍼런스.

## BaseEntity

모든 Entity가 상속하는 공통 클래스.

```java
@Getter
@AllArgsConstructor
public abstract class BaseEntity {
    private Boolean isDeleted;
    private Instant deletedAt;
    private Instant createdAt;
    private Instant updatedAt;
}
```

## 트리 구조

### 단순 (Reaction, Bookmark)

```
repository/
└── {Domain}Entity.java
```

### Embedded VO 포함 (CommunityPost, Comment)

```
repository/
└── {Domain}Entity.java          ← @Embedded.Nullable 직접 사용
```

### 복합 Embedded + Collection (Job)

```
repository/
├── {Domain}Entity.java          ← @MappedCollection으로 컬렉션 참조
├── embedded/
│   ├── ExperienceRequirementEmbeddable.java
│   ├── InterviewProcessEmbeddable.java
│   └── JobCompensationEmbeddable.java
└── collection/
    ├── JobTechCategory.java
    ├── JobLocation.java
    ├── JobResponsibility.java
    └── JobQualification.java
```

## 간단한 Entity 예시 (Reaction)

```java
@Table("reactions")
@Getter
@AllArgsConstructor
public class ReactionEntity extends BaseEntity {
    @Id
    private Long id;
    private Long userId;
    private TargetType targetType;
    private Long targetId;
    private ReactionType reactionType;

    public Reaction toDomain() {
        return new Reaction(id, userId, targetType, targetId,
            reactionType, getCreatedAt(), getUpdatedAt());
    }

    public static ReactionEntity from(Reaction domain) {
        return new ReactionEntity(
            false, null, domain.getCreatedAt(), domain.getUpdatedAt(),
            domain.getReactionId(), domain.getUserId(),
            domain.getTargetType(), domain.getTargetId(),
            domain.getReactionType());
    }

    public ReactionEntity softDelete() {
        return new ReactionEntity(
            true, Instant.now(), getCreatedAt(), Instant.now(),
            id, userId, targetType, targetId, reactionType);
    }
}
```

## Entity에서 @MappedCollection 사용 예시 (Job)

```java
@Table("jobs")
@Getter
@AllArgsConstructor
public class JobEntity extends BaseEntity {
    @Id
    private Long id;
    // ... 기본 필드 ...

    @Embedded.Nullable
    private ExperienceRequirementEmbeddable experience;

    @Embedded.Nullable
    private InterviewProcessEmbeddable interviewProcess;

    @MappedCollection(idColumn = "job_id", keyColumn = "job_id")
    private Set<JobTechCategory> techCategories;

    @MappedCollection(idColumn = "job_id", keyColumn = "job_id")
    private Set<JobLocation> locations;

    @MappedCollection(idColumn = "job_id", keyColumn = "job_id")
    private Set<JobResponsibility> responsibilities;
}
```

## 컬렉션 클래스 예시 (JobTechCategory)

```java
@Table("job_tech_categories")
@Getter
@AllArgsConstructor
public class JobTechCategory {
    @Column("category_name")
    private TechCategory categoryName;
}
```

## 복합 Embeddable 예시 (ExperienceRequirementEmbeddable)

```java
@Getter
@AllArgsConstructor
public class ExperienceRequirementEmbeddable {
    Integer minYears;
    Integer maxYears;
    @Column("experience_required")
    Boolean required;
    @Column("fresher_only")
    Boolean fresherOnly;
    CareerLevel careerLevel;

    public static ExperienceRequirementEmbeddable from(ExperienceRequirement domain) {
        if (domain == null) return null;
        return new ExperienceRequirementEmbeddable(
            domain.getMinYears(), domain.getMaxYears(),
            domain.getRequired(), domain.getFresherOnly(),
            domain.getCareerLevel());
    }

    public ExperienceRequirement toDomain() {
        return ExperienceRequirement.of(minYears, maxYears, required, fresherOnly, careerLevel);
    }
}
```
