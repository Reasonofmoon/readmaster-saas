# Export Manifest Contract

Export manifest는 `renderer-export`가 **같은 승인 content version**에서 만든 출력
파일의 불변 snapshot이다. 이 계약은 파일을 생성하는 renderer의 구현을 정의하지 않고,
orchestrator가 입력 version·템플릿·renderer와 실제 결과 파일을 대조할 수 있는 기록
경계를 정의한다. 원문 본문, OCR text, 정답·해설 내용, 학생 개인정보와 비밀값은 이
manifest 또는 출력 QA 기록에 복사하지 않는다.

## 입력 전제와 승인 경계

Export를 시작하려면 다음 조건을 모두 만족해야 한다.

1. `review-gates.md`의 `structureReview`와 `contentReview`가 모두 `approved`여야 한다.
2. 두 gate record의 `targetVersion`이 정확히 같은 immutable version이어야 한다.
3. 그 gate record의 `targetVersion`을 이 manifest의 `contentApprovalVersion`에 그대로
   기록해야 한다. 값이 없거나 다른 version을 가리키면 export하지 않는다.
4. `documentVersion`은 승인 content가 읽은 Canonical Document의
   `canonical-document.md` 계약상 immutable version이어야 한다. `documentVersion`과
   `contentApprovalVersion`은 각각 다른 identity 축이므로 임의로 서로 대체하지 않는다.
5. 입력 packet과 manifest의 version·source identity가 일치하고, stale dependency,
   미해결 QA 실패, 승인되지 않은 draft가 없어야 한다.
6. 출력 요청의 audience와 format은 아래 enum에 속해야 하며, renderer와 template의
   version은 생성 시점에 확인된 값이어야 한다.

두 gate 중 하나라도 없거나 version이 불일치하면 `renderer-export`를 실행하지 않는다.
사람 응답이 필요한 상태를 AI가 `approved`로 바꾸거나, 이전 승인 version을 현재 출력으로
가장해서 export하는 것도 허용하지 않는다.

## Lifecycle와 보안 실패 경계

`ExportManifest`의 `qaResult`와 renderer-preflight/output-QA evidence는 export 결과
projection이며 handoff packet의 lifecycle `status`나 `requiredHumanGate`가 아니다.
Renderer와 QA는 `lifecycleRecommendation`(`recommendedStatus`,
`recommendedRequiredHumanGate`, `reason`, `retryScope`)을 반환할 수 있지만 packet
lifecycle을 직접 쓰지 않는다. Orchestrator만 recommendation을 검증해 packet을 전이한다.

보안·무결성·scanner-untrusted 조건(candidate, archive, path, checksum/size 또는 검증
결과가 신뢰되지 않는 경우)은 항상 artifact를 quarantine하고 `qaResult.status=failed`,
`recommendedStatus=failed`, `recommendedRequiredHumanGate=none`으로 반환한다.
Orchestrator가 packet `status=failed`를 기록하며, 사람/security team 통지는 lifecycle과
분리된 `securityEscalation` metadata다. 보안 조건을 ordinary `waitingForHuman`으로
매핑하지 않는다.

## Manifest shape

`ExportManifest`의 최상위 필드는 아래 **9개만** 허용한다. 필드를 생략하거나 다른
audience·format 필드를 최상위에 추가하지 않는다.

```text
ExportManifest
- exportId
- projectId
- documentVersion
- contentApprovalVersion
- templateVersion
- rendererVersion
- generatedAt
- files[]
- qaResult
```

### ExportManifest 필드

| 필드 | 계약 |
| --- | --- |
| `exportId` | 이 export snapshot의 전역적으로 유일한 immutable 식별자. 재시도나 내용 변경 시 기존 값을 재사용하지 않는다. |
| `projectId` | 출력이 속한 ReadMaster 프로젝트의 안정적인 식별자. 학생 이름·이메일 등 개인정보를 넣지 않는다. |
| `documentVersion` | 출력 입력인 Canonical Document의 immutable version. 원본 또는 parser 결과가 바뀌면 새 version과 새 export를 만든다. |
| `contentApprovalVersion` | 두 사람 gate가 동일하게 승인한 `targetVersion`의 값. 모든 `files[]`는 이 승인 content version에서 파생되어야 한다. |
| `templateVersion` | 조판에 사용한 immutable template version. 레이아웃 변경은 기존 manifest를 수정하지 않고 새 export snapshot을 만든다. |
| `rendererVersion` | 실제 파일을 만든 renderer build/version. renderer 변경은 기존 manifest와 파일을 수정하지 않는다. |
| `generatedAt` | manifest와 파일이 생성된 시각. ISO 8601 UTC 형식이며 immutable이다. |
| `files[]` | 하나 이상의 `ExportFile` 목록. 선언된 모든 파일은 실제 존재하고, 목록 안에서 `path`가 중복되지 않아야 한다. |
| `qaResult` | 출력 preflight와 파일 무결성 검증 결과. 아래 semantic·layout·integrity 결과와 실패 범위·라우팅을 구조화해 기록한다. 원문·정답 본문은 기록하지 않는다. |

`files[]`의 각 원소는 아래 `ExportFile` 필드를 **정확히 5개** 가진다.

```text
ExportFile
- audience: student | teacher | answer
- format: pdf | docx | zip
- path
- checksum
- sizeBytes
```

| 필드 | 계약 |
| --- | --- |
| `audience` | 출력 공개 대상을 나타내는 소문자 enum. 허용값은 `student`, `teacher`, `answer`뿐이다. 한 파일이 둘 이상의 audience를 겸하지 않는다. |
| `format` | 파일 형식을 나타내는 소문자 enum. 허용값은 `pdf`, `docx`, `zip`뿐이다. 다른 확장자나 MIME 추측값으로 대체하지 않는다. |
| `path` | export 저장소의 opaque 내부 artifact 경로 또는 식별자. 원본 경로, 학생 개인정보, 정답 내용을 파일명에 넣지 않으며, 같은 경로를 다른 bytes로 재사용하지 않는다. |
| `checksum` | 이 파일의 정확한 bytes에 대한 암호학적 무결성 checksum. 알고리즘과 digest를 함께 기록하는 `sha256:<hex digest>` 형식을 사용한다. 파일을 변경하면 checksum도 새 manifest에서만 바뀔 수 있다. |
| `sizeBytes` | checksum을 계산한 동일한 파일의 정확한 byte 수. 0 이상 정수여야 하며, 저장소의 실제 크기와 대조한다. |

### Enum과 audience 격리

- `audience`와 `format`은 위에 선언한 값 외에는 허용하지 않는다. `student`, `teacher`,
  `answer`를 대소문자·별칭·복수형으로 변형하지 않는다.
- `student` 파일은 학습자에게 공개하는 문제·지시·답안 공간만 포함한다. 정답, 해설,
  정답을 직접 추론하게 하는 answer key, 교사용 메모와 내부 QA metadata를 포함하지
  않는다.
- `teacher` 파일은 승인된 강사용 자료만 포함한다. 강사용 지침이나 운영 메모를
  학생용 파일에 복사하지 않는다.
- `answer` 파일은 승인된 정답·해설 audience로 분리한다. 접근 권한은 manifest의 enum만으로
  부여하지 않고 외부 권한 경계에서 확인한다.
- `zip`은 audience별 패키지다. `audience: student`인 ZIP에는 student 파일만, `teacher`
  ZIP에는 teacher 파일만, `answer` ZIP에는 answer 파일만 담는다. audience가 섞인 ZIP을
  어느 한 audience로 표시하지 않는다.
- 단일 `path`를 여러 audience가 공유하지 않는다. 같은 원본에서 여러 audience를 만들더라도
  각 파일과 패키지의 내용·경로·checksum은 독립적으로 검증한다.

## Immutable version, checksum, and no-overwrite rules

Manifest와 `files[]`는 생성 후 수정·삭제·부분 덮어쓰기를 하지 않는 immutable snapshot이다.

1. `exportId`, `documentVersion`, `contentApprovalVersion`, `templateVersion`,
   `rendererVersion`, `generatedAt`는 생성 시 고정한다. 승인 content가 바뀌면 기존
   `contentApprovalVersion`을 재사용하지 않고 새 draft와 gate, 새 `exportId`를 만든다.
2. 같은 `contentApprovalVersion`이라도 template 또는 renderer가 바뀌면 기존 manifest를
   수정하지 않고 새 `templateVersion` 또는 `rendererVersion`, 새 `exportId`, 새 파일
   `path`를 사용한다.
3. `checksum`과 `sizeBytes`는 실제 저장된 bytes에서 계산한 뒤 기록한다. 기록 후 bytes가
   바뀌거나 checksum이 맞지 않으면 해당 export는 `failed` 또는 `stale`로 격리하고
   `completed`로 표시하지 않는다.
4. `path` 충돌은 기존 파일을 덮어쓰는 방식으로 해결하지 않는다. 충돌이 발견되면 새
   `exportId`와 새 opaque path를 예약하거나 export를 실패시키며, 기존 파일과 manifest는
   그대로 보존한다.
5. 출력 QA 실패를 고쳐 재렌더링할 때도 실패한 manifest와 파일을 지우거나 덮어쓰지 않는다.
   template/hidden-layer/archive/path/render-expression-only 원인의 재렌더링은
   `contentApprovalVersion`을 그대로 유지할 수 있지만, 새 candidate·`exportId`·path와
   전체 renderer preflight 및 별도 immutable output-QA를 다시 수행해야 한다. 변경된
   candidate에 대해 passed manifest나 passed output-QA를 재사용하지 않는다. Semantic
   projection/content 수정은 새 immutable package/documentVersion을 만들며, 기존 gate
   record를 재사용하지 않고 새 target에 대해 `structureReview`와 `contentReview` 두 human
   gate를 다시 수행한다.
6. 동일 입력의 재실행은 orchestrator의 idempotency 정책으로 처리한다. 이미 존재하는
   동일한 immutable `exportId`·파일 bytes를 재사용할 수는 있지만, 다른 bytes를 같은
   `exportId`나 path에 쓰지 않는다. 이 규칙은 manifest에 idempotency field를 추가하지
   않고도 적용된다.

`documentVersion` 또는 source dependency가 바뀌면 영향을 받은 이전 export를 `stale`로
표시할 수 있으나 삭제하지 않는다. 이전 승인 version·gate record·export manifest는
역사적 snapshot으로 보존하고, 새 version의 출력으로 가장하지 않는다.

### Answer-leak retry semantics

Answer leak 원인의 분류와 재시도는 `evidence-content-qa.md` 및
`renderer-export.md`와 동일하다.

| 원인 | version/gate | 재시도 identity와 검증 |
| --- | --- | --- |
| semantic projection/content 원인 | 새 immutable package/documentVersion을 만들고 새 target으로 `structureReview`와 `contentReview` 두 human gate를 모두 다시 수행한다. | 기존 package, gate record, manifest와 output-QA를 재사용하지 않는다. |
| template/hidden-layer/archive/path/render-expression-only 원인 | content가 바뀌지 않으면 같은 `contentApprovalVersion`을 유지할 수 있다. | 새 candidate, 새 `exportId`, 새 path를 만들고 전체 renderer preflight와 별도 immutable output-QA를 다시 수행한다. 변경된 candidate에 대해 passed manifest나 passed output-QA를 재사용하지 않는다. |

semantic projection/content인지 expression-only인지 판별할 수 없으면 semantic 경로로
상향하며, 어떤 경로도 변경된 candidate를 기존 passed manifest/output-QA로 seal하지 않는다.

## Output preflight

`qaResult`는 최소한 전체 `status`, 검사 시각, 각 검사 항목의 `status`, 영향 `scope`,
실패 `reasonCode`, 다음 `route`를 포함하는 구조화 결과다. 상태·사유에는 원문 본문,
정답·해설 문장, 학생 개인정보를 넣지 않고 opaque ID·version·파일 범위만 사용한다.

### Semantic preflight

의미 검사는 파일에서 추출한 구조와 승인 content version을 대조한다.

- `meaningMatch` / **의미 일치**: 문제·지시·자료의 의미와 승인된 동일 content version의
  관계, 순서, 정답 근거가 일치하는지 확인한다.
- `answerLeak` / **정답 노출**: `student` PDF·DOCX·ZIP의 본문, 문서 속성, 숨은 텍스트,
  주석과 ZIP entry에 정답·해설·answer key·내부 QA metadata가 없는지 확인한다.
- `audienceBoundary`: 각 파일과 ZIP entry가 선언한 audience의 공개 범위를 넘지 않는지
  확인한다. 학생 패키지에 teacher/answer artifact를 넣지 않는다.
- `approvedVersionMatch`: 모든 파일이 동일 `documentVersion`과
  `contentApprovalVersion`에서 파생되었고, 두 gate의 `targetVersion`과 일치하는지
  확인한다.

의미 불일치, 정답 노출, audience 경계 위반 또는 승인 version 불일치가 발생하면 먼저
원인을 분류한다. semantic projection/content 원인이면 새 immutable package/documentVersion과
두 human gate를 다시 수행하고, template/hidden-layer/archive/path/render-expression-only
원인이면 같은 `contentApprovalVersion`을 유지할 수 있지만 새 candidate/exportId/path와
전체 renderer preflight·immutable output-QA를 다시 수행한다. 보안·무결성·scanner-untrusted
원인은 quarantine 및 lifecycle `failed`로 처리하고, 원인이 불명확하면 semantic
projection/content로 분류한다. 어느 경우에도 변경된 candidate에 대해 passed
manifest/output-QA를 재사용하지 않는다. 해당 파일을 renderer가 조용히 고쳐서 통과시키지
않으며, semantic route의 `retryScope`는 `lesson-builder`와 `evidence-content-qa`로
돌려 근거·콘텐츠를 재검증한다. Semantic 수정으로 새 immutable
`targetVersion`/`contentApprovalVersion`을 만든 순간부터는 `canonical-document.md`와
`review-gates.md`의 version-scoped 규칙을 따른다. 이전 `targetVersion`의
`structureReview` 또는 `contentReview` 기록을 새 version에 재사용하지 않으며, export
전에 새 `targetVersion`에 정확히 일치하는 두 gate record가 각각 존재하고 모두 `approved`
여야 한다. 둘 중 하나라도 없거나 다른 version을 가리키면 export하지 않는다.

### Layout preflight

레이아웃 검사는 실제 PDF/DOCX를 렌더링하거나 추출 가능한 페이지 정보와 대조한다.

- `overflow` / **overflow**: 텍스트·표·이미지·수식이 페이지 경계 밖으로 잘리거나 겹치지
  않는지 확인한다.
- `missingGlyph` / **누락 glyph**: 폰트·문자·수식 glyph가 네모나 공백으로 대체되지
  않는지 확인한다.
- `blankPage` / **빈 페이지**: 의도하지 않은 빈 페이지가 생기지 않는지 확인한다.
- `margins` / **여백**: 템플릿이 요구하는 인쇄 여백과 안전 영역을 지키는지 확인한다.
- `answerSpace` / **답안 공간**: 학생용 문항의 답안 공간이 요청된 크기·위치로 남고,
  teacher/answer 출력의 필요한 공간과 겹치지 않는지 확인한다.
- `pageNumbers` / **페이지 번호**: 페이지 번호가 누락·중복·순서 역전 없이 실제 페이지와
  일치하고, 머리말·꼬리말과 겹치지 않는지 확인한다.

overflow, 누락 glyph, 빈 페이지, 여백, 답안 공간, 페이지 번호와 같은 레이아웃 실패는
콘텐츠를 재작성하지 않고 `renderer-export`로 돌려보낸다. renderer는 출력 범위와
`retryScope`만 좁혀 재조판하며, 조판 중 정답·해설·원문을 임의 변경하지 않는다.

### File integrity preflight

파일 무결성 검사는 아래를 확인한다.

- `checksum`: manifest의 checksum이 저장된 파일 bytes의 `sha256` digest와 정확히 같은지
  확인한다.
- `sizeBytes`: manifest의 크기가 실제 파일 byte 수와 같은지 확인한다.
- `filePresence`: `files[]`의 각 path가 정확히 하나의 파일을 가리키고, 선언되지 않은
  파일이 패키지에 몰래 추가되지 않았는지 확인한다.
- `archiveAudience`: ZIP의 모든 entry가 manifest의 audience 규칙과 일치하는지 확인한다.

checksum·size·파일 존재·archive 구성 실패는 integrity failure로서 renderer/orchestrator가
artifact를 quarantine하고 `qaResult.status=failed`, lifecycle `failed`로 기록한 뒤
격리한다. content가 변하지 않은 output-only 재생성으로 진행할 때만 같은
`contentApprovalVersion`의 새 candidate/exportId/path에서 전체 renderer preflight와
immutable output-QA를 다시 수행한다. bytes가 달라졌는데 기존 checksum을 유지하거나,
기존 path에 새 bytes를 덮어써서 통과시키지 않으며, 변경 candidate의 passed
manifest/output-QA를 재사용하지 않는다.

## QA 결과와 완료 조건

`qaResult.status`가 `passed`가 되려면 다음을 모두 만족해야 한다.

- semantic preflight의 모든 필수 검사가 통과했다.
- layout preflight의 모든 필수 검사가 통과했다.
- checksum·size·파일 존재·ZIP audience 무결성이 통과했다.
- 모든 `files[]`가 동일한 `documentVersion`과 `contentApprovalVersion`의 승인 결과에서
  파생되었다.
- student 파일과 student ZIP에 정답·해설·내부 QA metadata가 없고, 공개 범위가 분리됐다.
- 기존 manifest·파일을 덮어쓰지 않고 새 immutable snapshot으로 저장됐다.

하나라도 실패하면 `qaResult.status`는 `failed`로 남기며, 비보안으로 사람의 판단이
필요한 경우에만 Orchestrator가 별도 lifecycle recommendation을 검토해 packet을
`waitingForHuman`으로 둘 수 있다. 보안·무결성·scanner-untrusted 조건은 항상
`qaResult.status=failed`, quarantine, `recommendedStatus=failed`,
`recommendedRequiredHumanGate=none`이며 ordinary `waitingForHuman`이 아니다.
`passed`가 아닌 manifest는 다음 단계의 완료·공개·배포 대상으로 전달하지 않는다. AI는
출력 QA 결과를 기록할 수 있지만 강사의 공개·배포 결정을 대신하지 않는다.

## 역할 경계와 개인정보 보호

- `renderer-export`는 승인된 immutable content를 읽어 audience별 파일을 만들고, 실제
  파일의 checksum·size·preflight 결과를 기록한다. 콘텐츠를 임의 변경하거나 승인 gate를
  우회하지 않는다.
- `orchestrator`는 gate target version, input version, 파일 checksum, QA 상태와 retry
  routing을 대조한다. 불일치·stale·충돌은 격리한다.
- 오류 원인을 먼저 분류한다. semantic projection/content 오류는 새
  package/documentVersion과 두 human gate를 거쳐 `lesson-builder`·`evidence-content-qa`로,
  template/hidden-layer/archive/path/render-expression-only 오류는 같은
  contentApprovalVersion의 새 candidate/exportId/path에서 전체 renderer preflight와
  immutable output-QA를 거쳐 `renderer-export`로 돌린다. 보안·무결성·scanner-untrusted
  오류는 quarantine + lifecycle `failed`로 renderer/orchestrator에 돌리고, 불명확하면
  semantic projection/content로 분류한다. 변경 candidate에 passed manifest/output-QA를
  재사용하지 않는다.
- manifest와 QA 결과에는 원본 경로·OCR/Canonical 본문·정답/해설 문장·학생 개인정보·API
  key·인증정보를 기록하지 않는다. `path`와 `reasonCode`는 opaque 값과 범위만 사용한다.
- student 파일에는 정답·해설·내부 QA metadata가 없어야 한다. 이를 확인할 수 없으면
  학생용 export를 통과시키지 않고 answer-leak 실패로 격리한다.

이 계약은 ExportManifest/ExportFile의 구조와 검증 규칙만 정의한다. 실제 PDF·DOCX·ZIP
renderer, archive scanner, checksum 계산기, 저장소와 권한 구현은 별도 작업 범위다.
