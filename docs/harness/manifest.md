# ReadMaster Lesson Studio Harness Manifest

## 핵심 목표

교재를 원문 근거가 연결된 학생용·강사용 수업 패키지로 변환하는 제품을 일관되게 개발·검증한다.

## 트리거

다음 요청에는 이 하네스를 적용한다.

- Lesson Studio 기능 설계·구현·수정·검증
- PDF·DOCX·HWP 5.x·HWPX·이미지 교재 파싱
- 원문 근거형 문항·해설·수업안 생성과 QA
- 학생용·교사용·정답해설 PDF/DOCX 패키징

### should-trigger 예시

- 이 HWPX 교재를 Canonical Document로 파싱하는 기능을 설계해줘.
- 생성 문항의 정답 근거를 페이지와 block까지 검증해줘.
- 학생용과 교사용 PDF/DOCX 패키징 흐름을 구현해줘.

### 적용하지 않는 요청과 근접 사례

- 단순 블로그 게시, 번역, TTS, NotebookLM 또는 미디어 생성
- 교재를 수업 자료로 변환하지 않고 원문을 단순 번역만 하는 요청
- 이미 검수된 파일의 음성·썸네일·영상만 만드는 요청
- Lesson Studio와 무관한 일반 웹 UI·배포·마케팅 작업

### should-not-trigger 예시

- 이 교재 소개 글을 블로그에 게시해줘.
- 이 영어 지문을 MP3로 읽어줘.
- NotebookLM 인포그래픽을 만들어줘.

요청이 근접 사례와 Lesson Studio 패키지 제작을 함께 포함하면, 패키지 제작 범위만 이 하네스로 분류하고 나머지는 해당 작업의 별도 절차로 넘긴다. 하네스가 적용되지 않는다고 해서 원문·정답·학생 개인정보를 로그에 기록하지 않는다.

## 아키텍처

체크포인트 파이프라인 + 제한적 Fan-out/Fan-in + Generate-Verify

```text
업로드·검증·파싱
  → 구조 QA
  → [강사 structureReview]
  → 어휘·문항·해설·수업안 병렬 생성
  → 근거·콘텐츠 QA
  → [강사 contentReview]
  → 학생용·교사용·정답해설 렌더링
  → 출력 QA
  → 불변 version 패키징
```

생성 항목은 독립 단위로 제한적으로 병렬화한다. Fan-in 전에 각 결과의 계약, `sourceRefs`, 공개 범위를 확인한다. 실패한 항목만 격리·재시도하고, 승인된 version은 덮어쓰지 않는다. 부작용이 있는 출력·저장은 같은 입력에서 중복 실행해도 중복 결과가 생기지 않도록 idempotency key를 사용한다.

## 역할·계약 링크

역할 카드와 계약 문서는 현재 이 하네스가 참조하는 실행·검증 문서다. 아래 링크는 각
역할의 책임 경계와 계약의 현재 필드를 고정하며, 변경 시 링크 대상 문서와 이 manifest의
불변식을 함께 검토한다.

| 역할 | 역할 카드 | 핵심 책임 |
| --- | --- | --- |
| `orchestrator` | [agents/orchestrator.md](agents/orchestrator.md) | 상태·단계·체크포인트·승인 대기·재시도·통합을 조정한다. 문항과 해설을 직접 생성하거나 강사 승인을 대신하지 않는다. |
| `document-parser` | [agents/document-parser.md](agents/document-parser.md) | 검증된 원본을 Canonical Document로 변환하고 원문 순서·근거 위치·에셋·파싱 경고를 보존한다. 교육적 정답을 창작하지 않는다. |
| `lesson-builder` | [agents/lesson-builder.md](agents/lesson-builder.md) | 구조 승인된 원문에서 어휘·문항·해설·수업안·테스트 변형을 생성하고 항목별 `sourceRefs`를 붙인다. QA나 승인을 스스로 판정하지 않는다. |
| `evidence-content-qa` | [agents/evidence-content-qa.md](agents/evidence-content-qa.md) | 구조·근거·교육 품질·학생용 공개 범위를 독립 검증하고 항목별 판정·사유·`retryScope`를 남긴다. 실패를 묵인하거나 승인을 대신하지 않는다. |
| `renderer-export` | [agents/renderer-export.md](agents/renderer-export.md) | 두 승인 게이트를 통과한 version만 학생용·교사용·정답해설 출력물로 조판하고 export manifest·체크섬·출력 QA를 만든다. 콘텐츠를 임의 변경하지 않는다. |

| 계약 | 계약 문서 | 사용 시점 |
| --- | --- | --- |
| Handoff packet | [contracts/handoff-packet.md](contracts/handoff-packet.md) | 모든 역할 사이 전달. `jobId`, `projectId`, `sourceHash`, `documentVersion`, `schemaVersion`, `artifactPath`, `status`, `warnings[]`, `requiredHumanGate`, `retryScope`, `createdAt`을 포함한다. packet lifecycle `status`와 `requiredHumanGate`는 `orchestrator`만 전이하고, 다른 역할은 결과와 `lifecycleRecommendation`을 반환한다. |
| Canonical Document | [contracts/canonical-document.md](contracts/canonical-document.md) | 파싱과 생성의 기준. `metadata`, `pages[]`, `sections[]`, `blocks[]`, `questions[]`, `assets[]`, `parseWarnings[]`와 문항별 근거를 보존한다. |
| Review gates | [contracts/review-gates.md](contracts/review-gates.md) | `structureReview`와 `contentReview`의 대상 version, reviewer, decision, 시각, 사유를 기록한다. |
| Export manifest | [contracts/export-manifest.md](contracts/export-manifest.md) | 학생용·교사용·정답해설 파일, 입력·템플릿·렌더러 version, 체크섬, 생성 시각, 출력 QA를 구분해 기록한다. |

공통 계약의 원문은 위 링크를 기준으로 삼는다. 계약 파일이 아직 없는 동안에도 이름·필드·게이트의 의미는 [승인 스펙](../../tasks/SPEC-readmaster-lesson-studio-harness.md)과 일치해야 한다.

## Phase

| Phase | 입력 | 담당·게이트 | 출력·통과 조건 |
| --- | --- | --- | --- |
| 1. 요청 분류와 계약 선택 | 사용자 요청, 프로젝트 설정, 현재 version·실행 맥락 | Codex Main/Sol이 트리거와 근접 사례를 분류하고 `orchestrator`가 계약·역할을 선택한다. | 적용 여부, `jobId`, 선택 계약, 범위, 필요한 사람 게이트가 정해진다. 비적용 요청은 이 파이프라인에 넣지 않는다. |
| 2. 업로드 검증·파싱 | 검증할 원본 파일, 파일 메타데이터, 파서 설정 | `orchestrator`가 순서를 조정하고 `document-parser`가 수행한다. | 형식·크기·무결성 확인, `sourceHash`, Canonical Document, 에셋 연결, 파싱 경고가 handoff packet으로 남는다. |
| 3. 구조 QA와 `structureReview` | Canonical Document, 파싱 경고, 원문 위치 정보 | `evidence-content-qa`가 구조를 검사하고 강사가 `structureReview`를 결정한다. | 섹션·블록·문항·보기·정답·표·이미지·수식 연결과 누락이 검토된다. 승인 전에는 다음 생성 단계로 가지 않는다. |
| 4. 승인 구조 기반 병렬 생성 | `structureReview` 승인 version, 생성 프리셋, 학년·수준·CEFR·목표·출력 설정 | `lesson-builder`가 어휘·문항·해설·수업안·테스트 변형을 독립 lane으로 생성한다. | 각 항목이 계약을 만족하고 유효한 `sourceRefs`를 가진 초안과 항목별 handoff packet이 생긴다. 근거 없는 항목은 다음 단계로 넘기지 않는다. |
| 5. 근거·콘텐츠 QA와 `contentReview` | Canonical Document, 생성 초안, 생성 설정, 이전 QA 결과 | `evidence-content-qa`가 독립 QA를 수행하고 강사가 `contentReview`를 결정한다. | 정답 근거·중복·복수 정답·오답·난도·CEFR·학생용 공개 범위가 통과되어야 렌더링으로 간다. 미승인 version은 `waitingForHuman`에 머문다. |
| 6. 렌더링·출력 QA·version 패키징 | 두 게이트를 통과한 동일 version, 출력 설정, 템플릿 version, 렌더러 version | `renderer-export`가 조판·패키징하고 `orchestrator`가 입력 version과 결과를 대조한다. | 학생용·교사용·정답해설 출력물 분리, export manifest, 파일 체크섬, 인쇄·의미 일치 QA가 생성된다. 학생용에는 정답·해설·내부 메타데이터가 없다. |
| 7. 통합 검증과 실행 기록 | 모든 handoff packet, 승인 기록, QA 결과, 출력 manifest | Codex Main/Sol이 통합 검토하고 `orchestrator`가 상태·경고·재시도·승인을 기록한다. | 승인 대상 version과 출력 입력 version이 같고 모든 불변식·계약·체크섬이 확인된 `completed` version만 완료로 표시된다. |

## 완료 불변식

- 모든 생성 항목에 유효한 `sourceRefs`가 있다.
- 승인 대상 version과 출력 입력 version이 같다.
- `structureReview`와 `contentReview`를 통과한 version만 조판·패키징한다.
- 학생용 출력에 정답·해설·내부 메타데이터가 없다.
- 승인 version은 덮어쓰지 않고 변경 시 새 version과 새 상태를 만든다.
- 실제 존재가 확인된 명령만 실행 기록에 적는다.
- 사람 응답이 필요한 상태에서는 자동 승인·자동 발행하지 않는다.

허용 상태는 최소 `pending`, `running`, `waitingForHuman`, `approved`, `rejected`, `stale`, `failed`, `completed`로 구분한다. 파서 수정의 영향을 받은 섹션과 파생 산출물은 `stale`로 표시하고 승인된 이전 version은 보존한다.

공통 packet의 `status`와 `requiredHumanGate`는 `orchestrator`만 전이한다. 다른 역할은
Question `reviewStatus`, `qaResult`, immutable evidence와 `lifecycleRecommendation`을
반환한다. 보안·무결성·scanner-untrusted 조건은 quarantine 및 lifecycle `failed`로만
처리하고, `securityEscalation`은 사람/security 통지를 위한 별도 metadata로 남긴다.

## Codex Main/Sol과 `luna_worker` 라우팅

| 주체 | 맡는 일 | 경계 |
| --- | --- | --- |
| Codex Main/Sol | 의도·범위·아키텍처 결정, 계약 선택, 단계 순서와 승인 체크포인트 설계, 독립 작업 위임, 최종 통합·검증 | 문항·해설을 직접 생성해 QA를 겸하지 않으며, 강사 승인·공개·배포 결정을 대신하지 않는다. |
| `luna_worker` | Main/Sol이 지정한 독립적인 파일 범위의 조사·구현·검증. 역할 카드·계약·문서별 작업을 분리해 수행하고 변경 파일, 명령, 결과를 반환한다. | 정확한 파일 범위를 벗어나지 않고, 다른 lane의 공유 상태를 임의 변경하지 않는다. 막힘·불확실성은 범위를 넓히지 않고 Main/Sol에 보고한다. |
| 강사 | `structureReview`, `contentReview`, 공개 대상과 배포 결정 | AI가 응답 부재를 승인으로 간주하지 않는다. |

위 라우팅은 작업을 독립적으로 나눌 수 있을 때만 사용한다. 최종 gate와 전략 판단은 Main/Sol이 유지하며, 모든 lane의 결과는 Fan-in 전에 계약·근거·공개 범위를 재검증한다.

## 실패 처리와 재시도

실패는 전체 파이프라인보다 좁은 범위에서 복구한다.

```text
스키마 오류
  → 오류 필드 재생성
  → 원문 section 축소
  → 같은 계약을 지원하는 폴백 경로
  → 문제 항목 격리
  → 강사 검수 대기
```

- 형식 판별·구조·근거 위치가 불확실한 파싱은 경고와 실패 범위를 남기고 파서 재시도, section 축소 또는 사람 구조 검수로 올린다.
- 구조 오류는 영향을 받은 section·문항과 파생 산출물만 `stale`로 만들고 parser 또는 `structureReview`로 되돌린다.
- 생성 오류는 오류 필드·문항만 재생성하며 통과한 항목은 불필요하게 재생성하지 않는다.
- QA 오류는 실패 사유·근거·`retryScope`를 기록하고 해당 항목만 `lesson-builder`로 되돌린다. 근거가 불명확하면 격리한다.
- 강사 반려는 사유를 보존한 새 초안 version을 만들고 승인된 이전 version은 변경하지 않는다.
- 렌더링 오류는 원인을 먼저 분류한다. semantic projection/content면 새
  package/documentVersion과 `structureReview`/`contentReview` 두 human gate로 되돌리고,
  template/hidden-layer/archive/path/render-expression-only면 같은 contentApprovalVersion의
  새 candidate/exportId/path에서 전체 renderer preflight·immutable output-QA를 다시
  수행한다. 보안·무결성·scanner-untrusted면 quarantine + lifecycle `failed`로 처리하고,
  불명확하면 semantic projection/content로 분류한다. 변경 candidate의 passed
  manifest/output-QA는 재사용하지 않는다.
- 반복 실패·판단 불가·사람 결정 필요는 자동 발행하지 않고 Orchestrator recommendation에
  따라 `waitingForHuman` 또는 `failed`로 격리한다. 보안·무결성·scanner-untrusted 조건은
  항상 quarantine 및 `failed`이며 ordinary `waitingForHuman`으로 매핑하지 않는다.

## 공개·비공개 경계

### 비공개

원본 교재, OCR·파싱 텍스트, Canonical Document, 정답·해설 초안, 강사 메모, 내부 프롬프트, 모델 비용·실행 세부정보는 기본 비공개다. 실행 기록에는 본문 대신 해시, version, 상태, 경고, 내부 산출물 식별자만 남긴다.

비밀값·API 키·인증 정보·원문 내용·학생 개인정보는 이 하네스 문서와 공개 로그에 기록하지 않는다.

### 공유 가능

강사가 승인하고 적절한 권한이 부여된 학생용·교사용·정답해설 출력물만 공유 대상이 될 수 있다. 승인 전 초안과 근거 원문은 공유 경계를 넘지 않는다.

## 검증

### 구조 검증

- 목표 레이아웃의 역할·계약·실행·기록 링크가 서로 연결된다.
- 각 역할 카드에는 임무, 입력, 출력, 허용 능력, 금지사항, 실패·상향 경로가 있어야 한다.
- 모든 Phase에 입력, 담당, 출력·통과 조건이 있다.
- Handoff packet, Canonical Document, review gates, export manifest의 필드가 서로 일관된다.

### 트리거 검증

Lesson Studio 개발, 교재 파싱, 원문 근거형 문항 생성·검수, 학생용·교사용 출력 패키지 요청에는 적용한다. 단순 블로그 게시, TTS, 번역, NotebookLM 산출물 요청에는 적용하지 않는다. 분류가 불명확하면 범위를 넓히지 않고 `waitingForHuman`으로 올린다.

### 명령·실행 검증

- `package.json`이 존재한 뒤 해당 파일의 scripts를 확인하고 실제 정의된 명령만 등록·실행한다.
- 존재하지 않는 빌드·테스트·린트·배포 명령, 의존성, 서비스, 경로를 실행 계약으로 발명하지 않는다.
- 코드가 변경된 작업은 변경 후 `npx tsc --noEmit`을 실행하고 관련 범위의 테스트를 먼저 실행한다. 결과와 실패 원인은 실행 기록에 남긴다.
- 문서 작업의 검증은 대상 파일 존재 여부와 필수 개념 검색 결과를 기록하며, 명령이 실제로 실행된 경우에만 기록한다.

### 제품 QA

- 모든 생성 항목의 `sourceRefs`가 문서·page·block 위치까지 유효하다.
- 정답이 원문에서 도출되고 해설이 근거 위치를 제시한다.
- 문항 중복·복수 정답·부적절한 오답·지정 수준 불일치를 탐지한다.
- 학생용 출력에 정답·해설·내부 메타데이터가 노출되지 않는다.
- 동일 승인 version의 학생용·교사용·정답해설 간 의미가 일치한다.
- 실제 출력의 페이지·여백·답안 공간·페이지 번호와 파일 체크섬을 확인한다.

## 정상 흐름

```text
요청 분류·계약 선택
  → 원본 검증·파싱
  → 구조 QA
  → waitingForHuman: structureReview
  → 강사 승인
  → lesson-builder 제한적 병렬 생성
  → 근거·콘텐츠 QA
  → waitingForHuman: contentReview
  → 강사 승인
  → renderer-export 및 출력 QA
  → 동일 version 패키징·통합 검증
  → completed
```

각 화살표 사이에는 handoff packet과 상태 전이가 기록된다. 강사 응답이 없으면 승인으로 진행하지 않고 `waitingForHuman`에 머문다.

## 실패 흐름

```text
파싱 구조 불확실
  → 경고·영향 범위 기록
  → section 단위 재시도 또는 구조 검수
  → stale 범위만 재생성

생성 항목 QA 실패
  → 실패 사유·근거·retryScope 기록
  → 해당 문항/필드만 재생성
  → 재검증
  → 반복 실패 시 격리·waitingForHuman

강사 반려
  → 사유 보존
  → 새 초안 version
  → 해당 게이트 재검토

렌더링·출력 의미 불일치 또는 answer/audience leak
  → 원인 분류: semantic projection/content인지, template/hidden-layer/archive/path/render-expression-only인지 확인
  → semantic projection/content 원인: 새 package/documentVersion + structureReview/contentReview 두 human gate 재수행
  → template/hidden-layer/archive/path/render-expression-only 원인: 같은 contentApprovalVersion 허용 + 새 candidate/exportId/path + 전체 renderer preflight·immutable output-QA 재수행
  → 원인 불명: semantic projection/content로 분류
  → 보안·무결성·scanner-untrusted 원인: quarantine + lifecycle failed
  → 변경 candidate의 passed manifest/output-QA 재사용 금지, 해결 전 자동 발행 금지
```

## 변경 이력

| 날짜 | 대상 | 변경 이유 | 검증 결과 |
| --- | --- | --- | --- |
| 2026-08-24 | `AGENTS.md`, `docs/harness/manifest.md` | 승인 스펙의 Lesson Studio 진입점, 단계·역할·계약·사람 게이트·실패 복구·공개 경계를 프로젝트 로컬 하네스로 고정 | Task 1 focused `Test-Path`와 필수 개념 `rg` 검사를 생성 후 실행해 통과 여부를 기록함 |
| 2026-08-24 | `docs/harness/manifest.md` | Task 10의 적용·비적용 경계 예문을 명시 | should-trigger와 should-not-trigger 예문 검색을 통과함 |
