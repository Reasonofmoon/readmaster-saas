# Handoff Packet Contract

모든 역할 사이의 전달은 이 공통 packet을 사용한다. packet은 원문 본문이나 민감한 내용을 담지 않고, 내부 산출물과 해당 version을 식별하는 메타데이터만 전달한다.

## 필수 필드

| 필드 | 필수 | 의미 |
| --- | --- | --- |
| `jobId` | 예 | 한 실행의 안정적인 식별자 |
| `projectId` | 예 | 소속 프로젝트 식별자 |
| `sourceHash` | 예 | 원본 입력 무결성 확인값 |
| `documentVersion` | 예 | Canonical Document version |
| `schemaVersion` | 예 | packet과 산출물 계약 version |
| `artifactPath` | 예 | 민감 본문이 아닌 내부 산출물 위치 |
| `status` | 예 | 현재 단계 상태 |
| `warnings[]` | 예 | 빈 배열을 허용하는 구조화 경고 |
| `requiredHumanGate` | 예 | `none`, `structureReview`, `contentReview` 중 하나 |
| `retryScope` | 예 | 재실행할 field, question, section 또는 export 범위 |
| `createdAt` | 예 | ISO 8601 UTC 시각 |

`warnings[]`와 `retryScope`는 내용이 없을 때도 필드를 생략하지 않는다. `artifactPath`에는 원문·OCR 본문·정답·해설·학생 개인정보를 넣지 않으며, 실행 기록에는 내부 산출물 식별자만 남긴다.

## 허용 상태

`status`는 다음 값 중 하나만 사용할 수 있다.

`pending`, `running`, `waitingForHuman`, `approved`, `rejected`, `stale`, `failed`, `completed`

사람의 결정이 필요한 동안에는 `waitingForHuman`을 사용한다. 응답이 없다는 이유로 `approved` 또는 `completed`로 바꾸지 않는다. 파서 수정이나 원본 변경의 영향을 받은 section과 파생 산출물은 `stale`로 표시하고, 승인된 이전 version은 보존한다.

## 전달 전 검증과 금지

다음 조건을 모두 확인한 packet만 다음 역할로 전달한다.

- 11개 필수 필드가 모두 존재하고 빈 값이 계약상 허용되는 경우에도 올바른 구조를 가진다.
- `status`가 허용 상태 목록에 포함된다.
- `schemaVersion`, `documentVersion`, `sourceHash`가 packet이 가리키는 산출물·원본과 일치한다.
- `artifactPath`가 내부 산출물을 가리키며, 필요한 경우 해당 산출물이 실제로 존재한다.
- 생성 항목에는 유효한 원문 `sourceRefs`가 있다. `sourceRefs`가 없는 항목은 다음 단계로 넘기지 않는다.
- `requiredHumanGate`가 `none`, `structureReview`, `contentReview` 중 하나이며, 필요한 사람 승인이 완료되지 않은 경우 해당 gate를 표시한다.

필수 필드 누락, 허용되지 않은 상태, `schemaVersion`·`documentVersion`·`sourceHash`의 version/무결성 불일치가 하나라도 있으면 전달을 금지한다. packet은 `failed` 또는 영향받은 version에 대해 `stale`로 격리하고, 오류와 `retryScope`를 기록한 뒤 재시도 또는 사람 검수로 올린다. 승인된 version은 덮어쓰지 않는다.

## Gate와 version 불변성

`requiredHumanGate`가 `structureReview`인 packet은 강사의 구조 승인이 기록되기 전 `lesson-builder`로 전달하지 않는다. `contentReview`인 packet은 콘텐츠 승인이 기록되기 전 `renderer-export`로 전달하지 않는다. 두 게이트를 통과한 동일한 `documentVersion`과 산출물 version만 렌더링·패키징 대상으로 삼는다.

강사가 수정 요청 또는 반려를 하면 기존 승인 version을 변경하지 않고 새 draft version을 만든다. 재시도는 `retryScope`에 지정된 field, question, section 또는 export 범위로 제한한다.
