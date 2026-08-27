# Review Gates Contract

Review gate는 AI 생성 흐름을 멈추고 강사 또는 권한 있는 사람이 특정 immutable version을 검토했다는 사실을 기록하는 계약이다. 사람 응답이 필요한 상태에서 자동 승인·자동 발행하지 않는다.

## Gate record

- `gateName`: `structureReview` 또는 `contentReview`
- `targetVersion`: 검토 대상 immutable version
- `reviewer`: 강사 또는 권한 있는 사람 식별자
- `decision`: `approved`, `changesRequested`, `rejected`
- `reviewedAt`: ISO 8601 UTC 시각
- `reason`: 결정 사유

`targetVersion`은 검토 시점의 산출물 version과 정확히 일치해야 한다. `reviewer`, `reviewedAt`, `reason`을 포함한 감사 필드는 생략하지 않는다. `decision`은 위 세 값 외에는 허용하지 않는다.

## 전이 규칙

- 구조 승인 전 `lesson-builder` 실행 금지
- 콘텐츠 승인 전 `renderer-export` 실행 금지
- 응답 없음 → `waitingForHuman`
- 수정 요청 → 기존 승인 version 보존 + 새 draft version 생성
- AI에 의한 gate 승인 금지

추가로, `structureReview`는 Canonical Document의 섹션·블록·문항·보기·정답·표·이미지·수식 연결을 확인한 뒤 결정한다. `contentReview`는 생성 자료의 원문 근거, 정답·오답, 난도·CEFR, 학생용 공개 범위를 확인한 뒤 결정한다. 어느 gate든 승인되지 않은 version은 다음 단계로 이동하지 않는다.

## 허용 결정과 상태

| 결정 | packet 상태 | 허용되는 다음 동작 |
| --- | --- | --- |
| `approved` | `approved` | 해당 immutable version을 다음 gate 또는 다음 단계로 전달 |
| `changesRequested` | `waitingForHuman` 또는 새 draft의 `pending` | 사유와 `retryScope`를 보존하고 새 draft version을 만들어 같은 gate 재검토 |
| `rejected` | `rejected` | 해당 version을 격리하고 새 작업·version을 사람 결정에 따라 시작 |

검토 응답이 없거나 `targetVersion`이 현재 packet의 version과 다르면 승인을 기록하지 않는다. version 불일치 packet은 전달하지 않고 `stale` 또는 `failed`로 격리한다. 승인된 version은 덮어쓰지 않으며, 수정은 새 version으로만 수행한다.

## Gate 순서와 완료 조건

`structureReview` 승인 기록이 없는 version은 생성 단계로 진입할 수 없다. `contentReview` 승인 기록이 없는 version은 렌더링·export 단계로 진입할 수 없다. `renderer-export`는 두 gate가 모두 `approved`이고 동일한 immutable version을 가리킬 때만 실행할 수 있다.

강사 또는 권한 있는 사람이 남긴 `reviewer`, `decision`, `reviewedAt`, `reason`과 대상 version을 감사 기록에 보존한다. AI는 검토 자료와 권고를 제공할 수 있지만 `approved` 결정을 만들거나 사람의 공개·배포 결정을 대신할 수 없다.
