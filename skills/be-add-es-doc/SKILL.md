---
name: be-add-es-doc
description: 도메인 모델 스펙을 ES Document + Mapper + Indexer로 변환한다. Elasticsearch 인덱싱이 필요한 도메인이 있을 때 사용한다. "ES 문서 만들어줘", "도메인을 ES에 넣고 싶어", "인덱싱 구조 만들어줘" 등의 요청에 트리거한다.
---

# be-add-es-doc

도메인 모델을 Elasticsearch Document 구조로 변환하고, 인덱싱에 필요한 컴포넌트(Document, Mapper, Indexer)를 생성한다.

## Phase 0. 프로젝트 컨텍스트 해석

`{esModule}`, `{esPackage}` 플레이스홀더 사용.
실행 전 `${CLAUDE_PLUGIN_ROOT}/references/plugin-config-resolver.md` 절차로 해석.

## 입력

유저에게 아래를 확인한다:

1. **대상 도메인 모델 클래스** (예: Setup, SetupStats)
2. **ES에 포함할 필드** — 도메인의 전체 필드를 넣을 필요 없음. 검색/필터/정렬에 필요한 필드만 선별
3. **vectorText 구성 필드** — 벡터 임베딩 대상 텍스트를 조합할 필드 목록 (예: title + categories + tags + summary)
4. **searchWord 구성 필드** — BM25 텍스트 검색용 합성 텍스트 필드 목록 (vectorText와 다를 수 있음)
5. **dense_vector 차원** — 기본값 1536 (text-embedding-3-small). 다른 모델 사용 시 차원이 달라지므로 반드시 유저에게 확인
6. **JOIN 데이터** — 다른 도메인에서 가져와야 하는 필드 (예: userId → authorName)

## 생성 결과물 구조

```
{esModule}/src/main/java/{esPackage-as-path}/
├── document/
│   ├── DocBase.java                ← 공통 인터페이스 (getDocId) — 최초 1회만 생성
│   └── {Domain}Doc.java            ← ES 문서 (@Value, flat, vectorText/searchWord 자동생성)
├── mapper/
│   └── {Domain}DocMapper.java      ← Domain → Doc 변환 + 임베딩 호출
├── indexer/
│   ├── DocIndexer.java             ← 공통 인터페이스 (indexOne) — 최초 1회만 생성
│   ├── AbstractDocIndexer.java     ← ES client 기반 추상 클래스 — 최초 1회만 생성
│   ├── IndexResponseType.java      ← 인덱스 응답 enum — 최초 1회만 생성
│   └── {Domain}Indexer.java        ← 도메인별 Indexer (인덱스명 주입)
└── util/
    └── InstantConverter.java       ← Instant → epoch millis 유틸 — 최초 1회만 생성
```

공통 인프라(DocBase, DocIndexer, AbstractDocIndexer, IndexResponseType, InstantConverter)는 이미 존재하면 생성하지 않는다.

## 생성 규칙

### 1. Document (@Value, flat 구조)

- `@Value` (Lombok) + `@JsonInclude(JsonInclude.Include.NON_NULL)`
- **모든 필드를 flat하게** 펼친다. 중첩 객체 없음
  - 예: `popularity.viewCount` → `popularityViewCount`
  - 예: `List<SetupCategory>` → `List<String> categories` (enum.name())
- `DocBase` 인터페이스를 implements (`getDocId()` 제공)
- 각 필드에 `@JsonProperty("snake_case")` 명시
- 필드마다 ES 타입을 주석으로 표기

**필수 메타 필드:**

| 필드 | ES 타입 | 설명 |
|---|---|---|
| `docId` | keyword | `"{domain}_{id}"` 형식, ES 문서 ID |
| `vector` | dense_vector | 임베딩 벡터 (`List<Float>`) |
| `vectorText` | text | 벡터화 대상 합성 텍스트 |
| `searchWord` | text | BM25 검색용 합성 텍스트 |
| `deleted` | boolean | soft delete 상태 |
| `createdAt` | date (epoch_millis) | `Long` 타입 |
| `updatedAt` | date (epoch_millis) | `Long` 타입 |

**vectorText와 searchWord:**
- 둘 다 `of()` 정적 팩토리에서 `buildVectorText()`, `buildSearchWord()`를 호출하여 자동 생성
- 주요 텍스트 필드를 `\n`으로 join
- null/blank 체크 후 조합
- vectorText: 임베딩 모델에 전달될 텍스트. 의미적 유사도에 중요한 필드 위주
- searchWord: BM25 키워드 매칭에 사용. 키워드가 풍부한 필드 위주
- 두 필드의 구성이 동일할 수도, 다를 수도 있음 — 유저에게 확인

**정적 팩토리 패턴:**
```java
public static {Domain}Doc of(
        /* 모든 필드 (vector, vectorText, searchWord 제외) */
) {
    String vectorText = buildVectorText(title, categories, summary);
    String searchWord = buildSearchWord(title, categories, tags, summary);
    return new {Domain}Doc(..., vectorText, searchWord);
}
```

vector 필드는 Mapper에서 vectorText를 임베딩한 결과를 직접 주입한다. of()에서는 생성하지 않는다.

### 2. Mapper (Domain → Doc 변환)

- `@Component` + `@RequiredArgsConstructor`
- `OpenAiVectorClient`를 주입받아 `vectorText`를 임베딩
- vectorText가 비어있으면 예외 발생 (벡터 없이 인덱싱 불가)
- enum → `Enum.name()` 변환
- Instant → `InstantConverter.toEpochMillis()` 변환
- docId 생성: `"{domain}_{domainId}"` 형식

```java
@Component
@RequiredArgsConstructor
public class {Domain}DocMapper {

    private final OpenAiVectorClient vectorClient;

    public {Domain}Doc toDoc({Domain} domain, /* JOIN 데이터 */) {
        // 1. Doc 생성 (vectorText, searchWord 자동 생성됨)
        {Domain}Doc doc = {Domain}Doc.of(...);

        // 2. vectorText로 임베딩 생성
        float[] embedding = vectorClient.embed(doc.getVectorText());
        List<Float> vector = toFloatList(embedding);

        // 3. vector가 포함된 최종 Doc 반환
        return doc.withVector(vector);
        // 또는 of()에 vector를 별도 파라미터로 전달하는 방식도 가능
    }
}
```

> 주의: vector 주입 방식은 `withVector()` 메서드를 추가하거나, of()에서 vector를 별도 파라미터로 받는 방식 중 선택. @Value 불변 객체이므로 새 인스턴스를 반환해야 한다.

### 3. Indexer (ES 저장)

- `AbstractDocIndexer`를 상속
- 인덱스명은 `@Value("${elasticsearch.index.{domain}}")` 로 외부 설정에서 주입
- 구현할 추상 메서드 3개:

```java
@Component
public class {Domain}Indexer extends AbstractDocIndexer {

    private final String indexName;

    public {Domain}Indexer(
            ElasticsearchClient esClient,
            @Value("${elasticsearch.index.{domain}}") String indexName
    ) {
        super(esClient);
        this.indexName = indexName;
    }

    @Override
    protected String getIndex() { return indexName; }

    @Override
    protected String getDocId(DocBase doc) { return doc.getDocId(); }

    @Override
    protected boolean validateDoc(DocBase doc) { return doc.getDocId() != null; }
}
```

## 공통 인프라 (최초 1회 생성)

### DocBase
```java
public interface DocBase {
    String getDocId();
}
```

### DocIndexer
```java
public interface DocIndexer {
    IndexResponseType indexOne(DocBase doc);
}
```

### AbstractDocIndexer
- `ElasticsearchClient`를 주입받아 `esClient.index()` 호출
- `indexOne(DocBase doc)`: validate → sendRequest → IndexResponseType 반환
- 참고: references/abstract-doc-indexer.md

### IndexResponseType
- Created, Updated, Deleted, NotFound, NoOp, Unknown
- `from(Result result)`: ES 응답 → enum 매핑

### InstantConverter
```java
public static Long toEpochMillis(Instant instant) {
    return instant != null ? instant.toEpochMilli() : null;
}
```

### float[] → List<Float> 변환
Mapper 내부 private 메서드:
```java
private List<Float> toFloatList(float[] arr) {
    List<Float> list = new ArrayList<>(arr.length);
    for (float f : arr) list.add(f);
    return list;
}
```

## 실행 흐름

1. 유저에게 입력 항목 확인 (대상 도메인, 필드 선별, vectorText/searchWord 구성, 차원, JOIN 데이터)
2. 공통 인프라가 없으면 먼저 생성
3. Document 생성 — flat 필드 + vectorText/searchWord 빌더
4. Mapper 생성 — 도메인 변환 + 임베딩 호출
5. Indexer 생성 — AbstractDocIndexer 상속체
6. 생성된 파일 목록 + ES 인덱스 매핑 JSON(참고용) 출력

## 기존 도메인 추가 시

새 도메인이 추가되면 Document + Mapper + Indexer만 추가하면 된다. 공통 인프라는 재사용.

## 산출물

- 생성된 파일 목록
- ES 인덱스 매핑 JSON (참고용으로 출력)
- vectorText / searchWord 구성 공식
