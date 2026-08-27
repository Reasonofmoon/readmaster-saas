# Canonical Document Contract

Canonical Document는 `document-parser`가 원본 교재를 정규화한 뒤 `lesson-builder`와
`evidence-content-qa`가 같은 필드와 같은 원문 위치를 사용하는 기준 표현이다. 이 문서는
원본 본문을 예시로 저장하지 않으며, 원문·OCR·정답·해설은 하네스 문서와 실행 로그에
기록하지 않는다.

## 계약 경계와 version identity

Canonical Document의 입력 identity는 handoff packet의 다음 필드와 일치해야 한다.

| 필드 | 규칙 |
| --- | --- |
| `documentVersion` | 이 Document의 불변 version 식별자. 수정 시 기존 값을 재사용하지 않고 새 version을 만든다. |
| `sourceHash` | Document가 가리키는 원본 입력의 무결성 해시. 원본이 바뀌면 기존 version과 함께 재사용하지 않는다. |
| `schemaVersion` | 이 계약을 해석하는 schema version. parser, builder, QA와 handoff packet에서 동일해야 한다. |

세 값은 `Document.metadata`에 보존하고, 전달 packet의 값과 대조한다. 하나라도 불일치하면
Document를 다음 단계로 전달하지 않고 `failed` 또는 영향받은 version의 `stale`로 격리한다.

## Document 구조

Document의 최상위 필드는 아래 7개로 고정한다. 필드를 생략하거나 별도 포맷 전용 최상위
필드를 추가하지 않는다.

```text
Document
├── metadata
├── pages[]
├── sections[]
├── blocks[]
├── questions[]
├── assets[]
└── parseWarnings[]
```

### `metadata`

`metadata`는 최소한 다음 identity 필드를 가진다.

- `documentId`: `SourceRef.documentId`가 참조하는 안정적인 문서 식별자
- `documentVersion`: 현재 Canonical Document의 immutable version
- `sourceHash`: 현재 version이 파생된 원본 입력의 해시
- `schemaVersion`: 이 계약의 version
- `sourceFormat`: 사용한 입력 adapter를 식별하는 값(예: `hwp5`, `hwpx`)

문서 제목, 원본 경로, OCR 본문과 같은 민감하거나 불필요한 값은 실행 로그에 복제하지
않는다. metadata의 identity는 해당 Document와 handoff packet 사이의 연결을 확인할 때만
사용한다.

### `pages[]`

각 page는 원본 페이지를 식별하고 순서와 포함 관계를 보존한다. page에는 최소한
`pageNumber`, `sectionIds[]`, `blockIds[]`를 둔다. 입력 포맷이 페이지 크기와 좌표를
제공하면 `coordinates`를 함께 보존하며, page 경계는 그 좌표의 기준이 된다.

### `sections[]`

각 section은 문서의 논리적 순서를 보존한다. section에는 최소한 `sectionId`,
`pageNumbers[]`, `blockIds[]`를 두고, 연결된 문항이 있으면 `questionIds[]`로 역참조할 수
있게 한다. section과 page/block 연결이 불완전하면 조용히 보정하지 않고
`parseWarnings[]`에 영향 범위를 남긴다.

### `blocks[]`

block은 `SourceRef`의 offset 기준이 되는 원문 단위다. block에는 최소한 `blockId`,
`blockType`, `text`, `pageNumber`, `sectionId`, `childBlockIds[]`를 두며, 지원되는 입력에
대해 `coordinates`와 `assetIds[]`를 보존한다.

`blockType`은 적어도 `paragraph`, `heading`, `list`, `table`, `tableCell`, `image`,
`formula`를 구분할 수 있어야 한다. 표는 table → row/cell 순서와 부모 관계를, 이미지와
수식은 해당 block과 `assets[]`의 연결·페이지·좌표를 보존한다. 원본 순서, 표의 행·열,
이미지, 수식 또는 좌표를 표현할 수 없으면 해당 범위를 `parseWarnings[]`에 기록하고
구조 검수 대상으로 올린다.

### `questions[]`

`questions[]`의 각 항목은 아래 Question 계약을 그대로 따른다. `passageRefs[]`는 문항이
읽는 Canonical block을 가리키는 구조 연결이고, `sourceRefs[]`는 정답·해설을 원문까지
검증하는 근거 연결이다. `passageRefs[]`만 있고 `sourceRefs[]`가 없는 항목은 유효한
생성 항목으로 취급하지 않는다.

### `assets[]`

각 asset은 최소한 `assetId`, `assetType`, `pageNumber`, `blockId`, `coordinates`를
보존한다. `assetType`은 이미지·수식 등 원본 종류를 식별해야 하며, asset을 분리 저장할
때도 부모 page와 block 관계를 잃지 않는다. 좌표를 원본이 제공하지 않는 경우 임의 좌표를
만들지 말고 `parseWarnings[]`에 사유와 범위를 기록한다.

### `parseWarnings[]`

파싱이 불완전하거나 구조 검수가 필요한 모든 문제는 최소한 `warningId`, `code`,
`severity`, `message`, `retryScope`를 가진 구조화 경고로 남긴다. 가능하면
`pageNumber`, `blockId`, `sectionId`를 포함해 영향 범위를 좁힌다. 경고가 있다는 이유만으로
자동 승인하지 않으며, 표·이미지·수식·좌표·원문 위치의 누락은 구조 QA에서 실패하거나
사람 검수로 올라간다.

## Question 필드

Question은 아래 필드를 정확히 가진다. 생성·변형 문항은 모든 필드를 채워야 하며,
`sourceRefs[]`는 하나 이상이어야 한다.

```text
Question
- questionId
- questionNumber
- questionType
- stem
- passageRefs[]
- options[]
- correctAnswer
- explanation
- difficulty
- cefrLevel
- skillTags[]
- sourceRefs[]
- confidence
- reviewStatus
```

| 필드 | 계약 |
| --- | --- |
| `questionId` | 현재 Document version 안에서 안정적인 문항 식별자. 수정 시 새 draft version에서 추적 가능해야 한다. |
| `questionNumber` | 표시·정렬에 사용하는 원본 또는 생성 문항 번호. 보기 순서와 함께 보존한다. |
| `questionType` | 문항 유형을 나타내는 명시적 값. 유형을 추측하게 만드는 자유 형식 설명으로 대체하지 않는다. |
| `stem` | 문항 발문. 원문 passage와 혼동하지 않으며, 근거는 `sourceRefs[]`로 연결한다. |
| `passageRefs[]` | 문항이 참조하는 Canonical `blockId`의 순서 있는 배열. 참조 block은 현재 Document에 존재해야 한다. |
| `options[]` | 보기의 순서 있는 배열. 보기 식별자와 표시 텍스트를 보존하고, 원문 보기 순서를 임의로 재배열하지 않는다. |
| `correctAnswer` | 정답의 canonical 표현. 단일·복수 정답 여부와 보기 식별자를 모호하지 않게 표현한다. |
| `explanation` | 정답을 설명하는 내용. 설명의 근거 위치는 `sourceRefs[]`로 검증한다. |
| `difficulty` | 합의된 난도 값. QA가 생성 설정과 지정 수준을 대조할 수 있어야 한다. |
| `cefrLevel` | 목표 CEFR 수준 값. 알 수 없거나 불확실하면 추측하지 않고 QA 경고로 남긴다. |
| `skillTags[]` | 문항이 측정하는 기능·영역 tag 배열. |
| `sourceRefs[]` | 원문 근거의 비어 있지 않은 배열. 각 원소는 아래 SourceRef의 6개 필드를 모두 가진다. |
| `confidence` | parser/builder가 산출한 신뢰도. 사용 단위와 범위를 일관되게 유지하고 QA가 확인한다. |
| `reviewStatus` | 아래 전용 Question item enum 중 하나. handoff packet의 `status`나 review gate의 `decision`을 대신하지 않는다. |

### `reviewStatus` 전용 enum과 권한

`reviewStatus`는 문항 단위의 검수·승인 결과를 나타내는 **projection**이다. gate 감사
기록 자체가 아니며, 아래 7개 값만 허용한다.

| 값 | 의미 | 값을 기록할 수 있는 주체 |
| --- | --- | --- |
| `draft` | 생성 또는 재생성 중인 문항. 아직 QA나 사람 승인을 주장하지 않는다. | `lesson-builder`가 새 문항을 만들거나 새 version으로 재시작할 때만 설정한다. |
| `qaPassed` | `evidence-content-qa`가 근거·구조·콘텐츠 검사를 통과시킨 문항. 사람 승인은 아니다. | `evidence-content-qa`만 설정한다. |
| `qaFailed` | `evidence-content-qa`가 검증에 실패시킨 문항. | `evidence-content-qa`만 설정한다. |
| `changesRequested` | 사람 gate가 정확한 immutable `targetVersion`에 수정 요청을 기록한 뒤, 그 version에 속한 모든 Question에 적용되는 projection. | `review-gates.md`의 정확히 일치하는 `targetVersion`과 `decision: changesRequested`에서 orchestrator가 해당 version 전체에 projection한다. |
| `humanApproved` | 사람 gate가 정확한 immutable `targetVersion`을 승인한 뒤, 그 version에 속한 모든 Question에 적용되는 projection. | `review-gates.md`의 정확히 일치하는 `targetVersion`과 `decision: approved`에서 orchestrator가 해당 version 전체에 projection한다. |
| `humanRejected` | 사람 gate가 정확한 immutable `targetVersion`을 반려한 뒤, 그 version에 속한 모든 Question에 적용되는 projection. | `review-gates.md`의 정확히 일치하는 `targetVersion`과 `decision: rejected`에서 orchestrator가 해당 version 전체에 projection한다. |
| `stale` | parser 수정, 원본 변경 또는 dependency invalidation으로 현재 문항 근거가 더 이상 최신이 아닌 상태. | orchestrator의 stale 전파 정책만 설정한다. |

### Version-scoped human gate

기존 `review-gates.md` 계약에서 `gateName`과 `targetVersion`은 해당 **immutable
`targetVersion` 전체**에 적용된다. 이 MVP 계약에는 Question별 human gate scope가 없다.
따라서 orchestrator는 정확히 같은 `targetVersion`에 속한 모든 Question에 gate 결과를
동일하게 projection한다.

- `decision: approved` → 해당 `targetVersion`의 모든 Question은 `humanApproved`
- `decision: rejected` → 해당 `targetVersion`의 모든 Question은 `humanRejected`
- `decision: changesRequested` → 해당 `targetVersion`의 모든 Question은 `changesRequested`

Question 단위 QA와 `retryScope`는 사람 gate를 요청하기 **전** 실패한 Question을 격리하고
수정 범위를 줄이는 데 사용할 수 있다. 그러나 격리된 Question을 수정하면 부분 상태를
승인 version에 덮어쓰지 않고, 모든 Question과 계약 필드를 포함한 새 완전한 draft version을
만든다. 새 draft version은 동일한 version-scoped gate를 다시 통과해야 하며, 이전
version의 human decision은 바뀌지 않는다.

허용 전이는 다음과 같다.

- `lesson-builder`는 생성·재생성 시 `draft`만 만들거나 새 draft version으로 reset할 수 있다. `qaPassed`, `qaFailed`, `changesRequested`, `humanApproved`, `humanRejected`를 직접 설정하지 않는다.
- `evidence-content-qa`는 검증 결과에 따라 `draft`에서 `qaPassed` 또는 `qaFailed`만 설정한다. QA 통과는 사람 승인과 같지 않다.
- `qaFailed`, `changesRequested`, `humanRejected` 또는 승인 version을 수정해야 하는 경우 같은 version의 상태를 덮어쓰지 않고 모든 Question을 포함한 새 완전한 draft version을 만든다.
- 사람 gate의 `decision`만 `changesRequested`·`humanApproved`·`humanRejected`의 근거가 된다. orchestrator는 `review-gates.md` 기록의 `gateName`, `targetVersion`, `reviewer`, `reviewedAt`, `reason`이 현재 immutable `targetVersion`과 정확히 일치할 때만 그 version의 모든 Question에 projection을 반영한다.
- parser/dependency invalidation은 orchestrator 정책을 통해 현재 작업 항목을 `stale`로 만들 수 있다. 기존 승인 version의 기록과 상태는 변경하지 않는다.

handoff packet의 `status`, review gate의 `decision`, Question의 `reviewStatus`는 서로 다른
축이다. `status`는 packet lifecycle(`pending`, `running`, `waitingForHuman`, `approved`,
`rejected`, `stale`, `failed`, `completed`)이고, `decision`은 gate 감사 기록의
`approved`, `changesRequested`, `rejected`이며, `reviewStatus`는 위 7개 문항 projection이다.
어느 하나를 다른 하나로 복사하거나 대체하지 않는다. `humanApproved` projection이
있더라도 `review-gates.md`의 `gateName`과 정확히 일치하는 immutable `targetVersion`의
사람 gate 기록이 없으면 pipeline을 진행할 수 없다. Question 단위의 상태만으로 gate
승인을 주장하거나, Question별 human gate를 만들어 version-scoped gate를 우회할 수 없다.

`sourceRefs[]`가 비어 있거나, 존재하지 않는 page/block을 가리키거나, 현재
`documentVersion`·`sourceHash`와 연결되지 않으면 해당 Question은 다음 단계로 넘기지
않는다. 오류 필드 또는 문항 단위로 격리·재시도하고, 반복되면 `waitingForHuman` 또는
`failed`로 올린다.

## SourceRef 필드와 원문 위치 검증

SourceRef는 아래 6개 필드를 정확히 가진다. SourceRef는 원문 본문을 실행 로그에 복사하기
위한 자료가 아니라, 내부 Canonical Document 안의 검증 가능한 위치 포인터다.

```text
SourceRef
- documentId
- pageNumber
- blockId
- startOffset
- endOffset
- sourceQuote
```

| 필드 | 계약 |
| --- | --- |
| `documentId` | 참조 대상 Document의 `metadata.documentId`와 같아야 한다. |
| `pageNumber` | 참조 block이 속한 page 번호와 같아야 하며 양의 정수다. |
| `blockId` | 현재 Document의 `blocks[]`에 존재하는 ID여야 한다. |
| `startOffset` | 원문 block text 기준 0부터 시작하는 시작 offset이며, 끝 offset보다 작아야 한다. |
| `endOffset` | 같은 원문 block text 기준의 끝 offset이며, 범위 끝은 exclusive로 해석한다. |
| `sourceQuote` | 위 offset으로 잘라낸 원문 block의 정확한 문자열. 임의 trim, 대소문자 변경, 생략 부호 삽입을 허용하지 않는다. |

offset은 문서 전체, 페이지 전체 또는 렌더링 좌표 기준이 아니라 **원문 block 기준**이다.
QA는 다음을 모두 확인한다.

1. `documentId`, `pageNumber`, `blockId`가 현재 Document의 동일한 관계를 가리킨다.
2. `0 <= startOffset < endOffset`이고 `endOffset`이 해당 block text의 길이를 넘지 않는다.
3. `sourceQuote`가 해당 block text의 `[startOffset, endOffset)` slice와 정확히 일치한다.
4. block·asset의 `coordinates`가 존재할 경우 page 경계 안의 유한한 값이고, page/block 관계와 일치한다.
5. `passageRefs[]`와 `sourceRefs[]`의 block 연결이 서로 모순되지 않는다.

quote와 offset이 불일치하거나 위치가 존재하지 않으면 QA 실패로 처리한다. parser는
불일치를 묵인하거나 quote를 추측해서 고치지 않으며, 영향을 받은 block·question과
`retryScope`를 기록한다. 텍스트가 아닌 table/image/formula block의 원문 표현을 adapter가
확정할 수 없는 경우에도 경고를 남기고 근거 불충분 항목을 다음 단계로 전달하지 않는다.

## HWP 5.x·HWPX parser adapter 분리

HWP 5.x와 HWPX는 **별도의 parser adapter**를 사용한다.

- HWP 5.x adapter는 HWP 5.x 바이너리 구조를 직접 해석하고, HWPX XML/ZIP 전용 로직을 공용 parser로 재사용하지 않는다.
- HWPX adapter는 HWPX XML/ZIP 구조를 해석하고, HWP 5.x 바이너리 전용 로직을 공용 parser로 재사용하지 않는다.
- 두 adapter의 출력은 `sourceFormat`만으로 갈라지는 별도 문서가 아니라 동일한 위 Document·Question·SourceRef 계약이어야 한다.
- adapter가 생성한 페이지·section·block·question·asset ID와 관계는 downstream에서 포맷별 예외 처리 없이 검증할 수 있어야 한다.
- 페이지·블록·표·이미지·수식 관계를 지원하지 못하는 경우 조용히 버리지 않고 해당 page/block/asset 범위의 `parseWarnings[]`를 만든다. 구조 QA와 강사 `structureReview`가 끝나기 전에는 builder로 전달하지 않는다.

parser는 원문 순서와 텍스트를 교육 콘텐츠로 재작성하지 않는다. 표의 부모·셀 순서,
이미지·수식의 부모 block과 asset 연결, 가능한 경우 페이지 좌표를 보존하여
`SourceRef`가 같은 원문 위치를 재현할 수 있게 한다.

## `sourceRefs` 필수 및 handoff 규칙

어휘, 문항, 해설, 수업안, 테스트 변형을 포함한 **모든 생성 항목**은 하나 이상의
유효한 `sourceRefs[]`를 가져야 한다. Question은 위 필드를 직접 가지며, 다른 생성
artifact도 동일한 SourceRef 구조로 원문 근거를 연결한다.

다음 조건을 하나라도 만족하지 않으면 해당 항목은 handoff packet의 다음 역할로 전달하지
않는다.

- `sourceRefs[]`가 없거나 빈 배열이다.
- SourceRef의 6개 필드 중 하나라도 빠졌다.
- page/block/Document identity가 현재 version과 다르다.
- offset 범위 또는 `sourceQuote`가 원문 block과 일치하지 않는다.
- QA가 근거를 확인하지 못했거나 parser warning이 근거 범위를 덮는다.

이 경우 항목만 격리해 오류 필드 또는 문항 단위로 재시도하고, 계속 해결되지 않으면
`failed` 또는 `waitingForHuman`으로 올린다. 근거 없는 항목을 빈 sourceRef로 통과시키거나
사람 gate를 자동 승인하지 않는다.

## stale dependency 전파와 version 불변성

파서 수정, 원본 변경 또는 schema 해석 변경으로 block이 달라지면 새 `sourceHash` 또는
`documentVersion`을 만든다. 기존 Canonical Document와 승인 기록은 수정·삭제·덮어쓰지
않는다.

영향 전파는 다음 순서로 수행한다.

1. 이전 Document와 새 Document를 비교해 변경·삭제·위치 이동된 `blockId`와 그 부모 section을 식별한다. 좌표·표 셀 순서·asset 연결만 달라져도 위치 근거가 달라진 것으로 간주한다.
2. 변경된 block을 `sourceRefs[].blockId` 또는 `passageRefs[]`로 참조하는 Question을 현재 작업 version에서 `stale` 처리한다.
3. 그 Question 또는 변경된 block/document version을 dependency로 기록한 lesson artifact와 export를 `stale` 처리한다. export는 export manifest의 input version·source reference 연결을 기준으로 판정한다.
4. 영향받은 항목의 handoff packet에 `status: stale`과 좁은 `retryScope`(block, question, section 또는 export)를 기록한다. 영향이 없는 항목은 불필요하게 재생성하지 않는다.
5. 새 parser 결과에서 근거를 다시 연결하고 구조 QA, 필요한 `structureReview`, 생성·콘텐츠 QA, `contentReview`를 해당 범위에 대해 다시 수행한다. `stale` artifact는 승인 전 renderer-export로 보내지 않는다.

파서 수정으로 새 작업 version이 stale이 되더라도, **이전 승인 version과 그 export는 삭제하지
않고 immutable snapshot으로 보존**한다. 이전 승인 version의 `approved` 결정과 감사 기록은
역사적 사실로 남기며, 새 version이 그 기록을 덮어쓰지 않는다. 새 version이 승인되기 전에는
이전 export를 현재 version의 출력으로 가장하거나 자동 재발행하지 않는다.

## 역할별 사용과 QA 체크리스트

- `document-parser`: 입력 adapter를 선택하고 위 7개 Document 필드, 관계, `parseWarnings[]`, identity를 만든다. 교육적 정답이나 해설을 생성하지 않는다.
- `lesson-builder`: `structureReview`가 승인한 동일 Document version을 읽고 Question의 14개 필드와 유효한 `sourceRefs[]`를 채우며, `reviewStatus`는 생성·재생성 시 `draft`만 설정한다. QA 결과나 version-scoped 사람 gate 승인을 설정하지 않는다.
- `evidence-content-qa`: schema·identity·관계·좌표·offset·quote·근거 완전성·난도·CEFR·공개 범위를 독립 검증하고, 검증 결과에 따라 `reviewStatus`를 `qaPassed` 또는 `qaFailed`로만 설정한다.
- `renderer-export`: `structureReview`와 `contentReview`를 통과한 동일 version만 사용하며, stale dependency가 있는 export를 렌더링하지 않는다.

최소 QA 순서는 다음과 같다.

1. Document의 최상위 7개 필드와 metadata identity 존재 여부
2. page/section/block/question/asset ID 및 순서·부모 관계
3. 표·이미지·수식 block/asset 연결과 좌표 범위
4. Question의 정확한 14개 필드와 `passageRefs[]` 존재 여부
5. SourceRef 6개 필드, block 기준 offset, `sourceQuote` slice 일치 여부
6. 모든 생성 항목의 비어 있지 않은 `sourceRefs[]`와 현재 version 일치 여부
7. 파서 변경 시 영향 question/export의 `stale` 전파와 이전 승인 version 보존 여부
8. Question `reviewStatus` 전용 enum·writer·전이와 packet `status`·gate `decision`의 분리, 전체 immutable `targetVersion`에 적용되는 gate 기록 여부

이 계약은 데이터 구조와 검증 규칙만 정의한다. 실제 HWP/HWPX parser, runtime schema
validator, QA 코드, 저장소와 export 구현은 별도 작업 범위다.
