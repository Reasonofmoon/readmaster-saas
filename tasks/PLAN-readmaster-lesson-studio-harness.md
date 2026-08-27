# ReadMaster Lesson Studio Project Harness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `subagent-driven-development` (recommended) or `executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 승인된 ReadMaster Lesson Studio 설계를 프로젝트 로컬 Codex 하네스 문서, 역할 카드, 데이터 계약과 검증 가능한 실행 규칙으로 구현한다.

**Architecture:** 루트 `AGENTS.md`는 단일 manifest를 가리키고, manifest가 체크포인트 파이프라인과 역할·계약 문서를 연결한다. 다섯 역할은 공통 handoff packet과 Canonical Document, 사람 승인, export manifest 계약으로 통신하며 구조·콘텐츠·출력 검증을 분리한다.

**Tech Stack:** Markdown, Codex 프로젝트 지침, PowerShell 7 구조 검증. 애플리케이션 코드와 `package.json`은 이 계획의 범위가 아니다.

## Global Constraints

- 작업 기준 스펙은 `tasks/SPEC-readmaster-lesson-studio-harness.md`다.
- 모든 수정 전에 대상 파일의 현재 내용을 읽거나 존재하지 않음을 확인한다.
- `sourceRefs`가 없는 생성 항목은 다음 단계로 전달하지 않는다.
- `structureReview`와 `contentReview`를 모두 통과한 version만 조판한다.
- 강사 응답이 없으면 `waitingForHuman`에 머물며 자동 승인·자동 발행하지 않는다.
- 원본 교재, OCR 본문, Canonical Document 본문, 정답 초안, 내부 프롬프트, 비밀값을 하네스 문서와 실행 기록에 넣지 않는다.
- 기존 출력과 승인 version을 덮어쓰지 않는다.
- 새 글로벌 Codex 스킬을 만들지 않는다.
- 실제 `package.json`에 존재가 확인되지 않은 빌드·테스트·배포 명령을 manifest에 등록하지 않는다.
- 이 계획은 Markdown 문서만 변경하므로 `npx tsc --noEmit` 대상이 아니다. 이후 TypeScript 변경에는 반드시 실행한다.
- 현재 폴더는 Git 저장소가 아니다. 실행자가 자동으로 `git init`하거나 커밋하지 않는다. 사용자가 Git과 커밋을 승인한 뒤에만 Conventional Commits 규칙으로 작업별 커밋한다.

## Target File Map

| 파일 | 책임 |
| --- | --- |
| `AGENTS.md` | 프로젝트 하네스 진입점과 전역 불변식 요약 |
| `docs/harness/manifest.md` | 트리거, 실행 단계, 역할 라우팅, 검증, fallback, 변경 이력 |
| `docs/harness/contracts/handoff-packet.md` | 역할 간 공통 전달 필드와 유효성 규칙 |
| `docs/harness/contracts/canonical-document.md` | Document·Question·SourceRef 계약과 stale 전파 규칙 |
| `docs/harness/contracts/review-gates.md` | 사람 승인 상태와 허용 전이 |
| `docs/harness/contracts/export-manifest.md` | audience별 출력·checksum·version 계약 |
| `docs/harness/agents/orchestrator.md` | 상태, gate, 재시도와 통합 책임 |
| `docs/harness/agents/document-parser.md` | 안전한 포맷별 파싱과 근거 위치 보존 책임 |
| `docs/harness/agents/lesson-builder.md` | 승인 원문 기반 수업자료 생성 책임 |
| `docs/harness/agents/evidence-content-qa.md` | 구조·근거·교육 품질 독립 검증 책임 |
| `docs/harness/agents/renderer-export.md` | 승인본 조판, audience 분리와 출력 preflight 책임 |
| `docs/harness/runs/README.md` | 실행 기록 형식과 민감 정보 금지 규칙 |
| `docs/harness/runs/2026-08-24-harness-bootstrap/verification.md` | 최초 하네스 구현의 구조·계약 검증 증거 |
| `tasks/lessons.md` | 사용자 교정에서 일반화한 재발 방지 패턴 |

---

### Task 1: 하네스 진입점과 오케스트레이터 manifest

**Files:**
- Create: `AGENTS.md`
- Create: `docs/harness/manifest.md`
- Read: `tasks/SPEC-readmaster-lesson-studio-harness.md`

**Interfaces:**
- Consumes: 승인 스펙의 아키텍처, 5개 역할명, 4개 계약명, 두 사람 승인 gate.
- Produces: 이후 모든 역할·계약 문서가 따라야 하는 phase 순서와 링크 목록.

- [x] **Step 1: 대상 파일이 아직 없는지 확인한다**

Run:

```powershell
Test-Path -LiteralPath 'AGENTS.md'
Test-Path -LiteralPath 'docs\harness\manifest.md'
```

Expected: 두 줄 모두 `False`. 파일이 이미 있으면 내용을 먼저 읽고 승인 스펙과 충돌하는 부분을 보고한다.

- [x] **Step 2: 짧은 루트 진입점을 작성한다**

`AGENTS.md`에는 다음 내용을 포함한다.

```markdown
# ReadMaster Lesson Studio Agent Entry Point

프로젝트 작업 전 `docs/harness/manifest.md`와 현재 작업에 해당하는 역할·계약 문서를 읽는다.

핵심 불변식:
- `sourceRefs`가 없는 생성 항목은 다음 단계로 넘기지 않는다.
- `structureReview`와 `contentReview`를 통과한 version만 조판한다.
- 강사 승인과 공개·배포 결정을 AI가 대신하지 않는다.
- 코드 변경 후 `npx tsc --noEmit`을 실행하고 관련 테스트만 먼저 수행한다.
```

- [x] **Step 3: manifest를 작성한다**

`docs/harness/manifest.md`에 다음 섹션과 실제 내용을 작성한다.

```markdown
# ReadMaster Lesson Studio Harness Manifest

## 핵심 목표
교재를 원문 근거가 연결된 학생용·강사용 수업 패키지로 변환하는 제품을 일관되게 개발·검증한다.

## 트리거
- Lesson Studio 기능 설계·구현·수정·검증
- PDF·DOCX·HWP 5.x·HWPX·이미지 교재 파싱
- 원문 근거형 문항·해설·수업안 생성과 QA
- 학생용·교사용·정답해설 PDF/DOCX 패키징

## 적용하지 않는 요청
- 단순 블로그 게시, 번역, TTS, NotebookLM 또는 미디어 생성

## 아키텍처
체크포인트 파이프라인 + 제한적 Fan-out/Fan-in + Generate-Verify

## Phase
1. 요청 분류와 계약 선택
2. 업로드 검증·파싱
3. 구조 QA와 `structureReview`
4. 승인 구조 기반 병렬 생성
5. 근거·콘텐츠 QA와 `contentReview`
6. 렌더링·출력 QA·version 패키징
7. 통합 검증과 실행 기록

## 완료 불변식
- 모든 생성 항목에 유효한 `sourceRefs`가 있다.
- 승인 대상 version과 출력 입력 version이 같다.
- 학생용 출력에 정답·해설·내부 메타데이터가 없다.
- 실제 존재가 확인된 명령만 실행 기록에 적는다.
```

같은 파일에 역할·계약 링크, Phase별 입력/출력, Codex Main/Sol과 `luna_worker` 라우팅, 실패·재시도, 공개·비공개 경계, 구조·트리거·명령·제품 QA, 정상·실패 흐름, 날짜별 변경 이력을 추가한다.

- [x] **Step 4: 진입점과 manifest 구조를 검증한다**

Run:

```powershell
$required = @(
  'AGENTS.md',
  'docs\harness\manifest.md'
)
$required | ForEach-Object { "$_=$((Test-Path -LiteralPath $_))" }
rg -n "핵심 목표|트리거|Phase|sourceRefs|structureReview|contentReview|waitingForHuman|변경 이력" docs/harness/manifest.md
```

Expected: 두 파일이 `True`이고, `rg`가 각 필수 개념을 한 번 이상 출력한다.

---

### Task 2: 공통 handoff와 사람 승인 계약

**Files:**
- Create: `docs/harness/contracts/handoff-packet.md`
- Create: `docs/harness/contracts/review-gates.md`
- Read: `docs/harness/manifest.md`

**Interfaces:**
- Consumes: manifest의 Phase와 `waitingForHuman` 원칙.
- Produces: 모든 역할 카드가 참조할 공통 packet 필드와 승인 전이.

- [x] **Step 1: 두 계약 파일의 부재를 확인한다**

Run:

```powershell
Test-Path -LiteralPath 'docs\harness\contracts\handoff-packet.md'
Test-Path -LiteralPath 'docs\harness\contracts\review-gates.md'
```

Expected: 두 줄 모두 `False`.

- [x] **Step 2: handoff packet 계약을 작성한다**

`docs/harness/contracts/handoff-packet.md`에 다음 필드 표를 작성한다.

```markdown
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
```

허용 상태 `pending`, `running`, `waitingForHuman`, `approved`, `rejected`, `stale`, `failed`, `completed`와 누락·version 불일치 시 전달 금지 규칙을 함께 적는다.

- [x] **Step 3: review gate 계약을 작성한다**

`docs/harness/contracts/review-gates.md`에 다음 계약을 작성한다.

```markdown
## Gate record

- `gateName`: `structureReview` 또는 `contentReview`
- `targetVersion`: 검토 대상 immutable version
- `reviewer`: 강사 또는 권한 있는 사람 식별자
- `decision`: `approved`, `changesRequested`, `rejected`
- `reviewedAt`: ISO 8601 UTC 시각
- `reason`: 결정 사유

## 전이 규칙

- 구조 승인 전 `lesson-builder` 실행 금지
- 콘텐츠 승인 전 `renderer-export` 실행 금지
- 응답 없음 → `waitingForHuman`
- 수정 요청 → 기존 승인 version 보존 + 새 draft version 생성
- AI에 의한 gate 승인 금지
```

- [x] **Step 4: 필드와 상태를 검증한다**

Run:

```powershell
rg -n "jobId|projectId|sourceHash|documentVersion|schemaVersion|artifactPath|status|warnings\[\]|requiredHumanGate|retryScope|createdAt" docs/harness/contracts/handoff-packet.md
rg -n "structureReview|contentReview|approved|changesRequested|rejected|waitingForHuman|targetVersion|reviewer|reviewedAt|reason" docs/harness/contracts/review-gates.md
```

Expected: handoff 11개 필드와 gate 상태·감사 필드가 모두 출력된다.

---

### Task 3: Canonical Document 계약

**Files:**
- Create: `docs/harness/contracts/canonical-document.md`
- Read: `docs/harness/contracts/handoff-packet.md`
- Read: `tasks/SPEC-readmaster-lesson-studio-harness.md`

**Interfaces:**
- Consumes: `documentVersion`, `sourceHash`, `schemaVersion`.
- Produces: parser, builder, QA가 동일하게 사용하는 Document·Question·SourceRef 필드.

- [x] **Step 1: 계약 파일 부재를 확인한다**

Run:

```powershell
Test-Path -LiteralPath 'docs\harness\contracts\canonical-document.md'
```

Expected: `False`.

- [x] **Step 2: Document와 Question 구조를 작성한다**

다음 필드를 정확히 정의한다.

```markdown
Document
├── metadata
├── pages[]
├── sections[]
├── blocks[]
├── questions[]
├── assets[]
└── parseWarnings[]

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

HWP 5.x와 HWPX는 별도 parser adapter를 사용하며, 페이지·블록·표·이미지·수식 관계를 조용히 버리지 않는다고 명시한다.

- [x] **Step 3: SourceRef와 영향 전파 규칙을 작성한다**

```markdown
SourceRef
- documentId
- pageNumber
- blockId
- startOffset
- endOffset
- sourceQuote
```

offset은 원문 block 기준이며, quote와 offset이 불일치하면 QA 실패로 처리한다. 파서 수정 시 변경된 block을 참조하는 question과 export를 `stale` 처리하되 이전 승인 version은 삭제하지 않는다고 적는다.

- [x] **Step 4: 계약 필드를 검증한다**

Run:

```powershell
rg -n "metadata|pages\[\]|sections\[\]|blocks\[\]|questions\[\]|assets\[\]|parseWarnings\[\]" docs/harness/contracts/canonical-document.md
rg -n "questionId|questionNumber|questionType|passageRefs\[\]|sourceRefs\[\]|documentId|pageNumber|blockId|startOffset|endOffset|sourceQuote|HWP 5.x|HWPX|stale" docs/harness/contracts/canonical-document.md
```

Expected: Document, Question, SourceRef, 포맷 분리와 stale 규칙이 모두 출력된다.

---

### Task 4: Export manifest 계약

**Files:**
- Create: `docs/harness/contracts/export-manifest.md`
- Read: `docs/harness/contracts/review-gates.md`
- Read: `docs/harness/contracts/canonical-document.md`

**Interfaces:**
- Consumes: 승인된 `targetVersion`, audience, renderer와 template version.
- Produces: renderer와 orchestrator가 검증할 파일 목록·checksum·QA 결과.

- [x] **Step 1: 계약 파일 부재를 확인한다**

Run:

```powershell
Test-Path -LiteralPath 'docs\harness\contracts\export-manifest.md'
```

Expected: `False`.

- [x] **Step 2: 출력 계약을 작성한다**

```markdown
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

ExportFile
- audience: student | teacher | answer
- format: pdf | docx | zip
- path
- checksum
- sizeBytes
```

학생용 파일에는 정답·해설·내부 QA 메타데이터가 없어야 하고, 모든 파일은 동일한 승인 content version에서 파생되어야 하며, 이전 export를 덮어쓰지 않는다고 명시한다.

- [x] **Step 3: 출력 preflight를 작성한다**

의미 일치, 정답 노출, overflow, 누락 glyph, 빈 페이지, 여백, 답안 공간, 페이지 번호와 checksum을 검증 항목으로 적는다. 의미 오류는 builder·QA로, 레이아웃 오류는 renderer로 돌려보낸다.

- [x] **Step 4: 출력 계약을 검증한다**

Run:

```powershell
rg -n "exportId|documentVersion|contentApprovalVersion|templateVersion|rendererVersion|generatedAt|files\[\]|qaResult|student|teacher|answer|pdf|docx|zip|checksum|sizeBytes" docs/harness/contracts/export-manifest.md
rg -n "정답 노출|overflow|glyph|빈 페이지|여백|답안 공간|페이지 번호|덮어쓰" docs/harness/contracts/export-manifest.md
```

Expected: manifest 필드, audience 분리와 출력 검증 항목이 모두 출력된다.

---

### Task 5: Orchestrator 역할 카드

**Files:**
- Create: `docs/harness/agents/orchestrator.md`
- Read: `docs/harness/manifest.md`
- Read: `docs/harness/contracts/handoff-packet.md`
- Read: `docs/harness/contracts/review-gates.md`

**Interfaces:**
- Consumes: 모든 역할의 handoff packet과 강사 gate record.
- Produces: 다음 작업 지시, 상태 전이, 최종 manifest 또는 blocked report.

- [x] **Step 1: 역할 카드 부재를 확인한다**

Run: `Test-Path -LiteralPath 'docs\harness\agents\orchestrator.md'`

Expected: `False`.

- [x] **Step 2: 역할 카드를 작성한다**

다음 섹션을 모두 포함한다.

```markdown
# Orchestrator
## 미션
## 필요한 입력
## 생성할 출력
## 사용할 도구·스킬
## 금지 사항
## Handoff 프로토콜
## 실패·상향 규칙
## 완료 조건
```

`harness`, `writing-plans`, `verification-before-completion`, `systematic-debugging`을 조건에 맞게 사용하고, 직접 파싱·문항 생성·조판·사람 승인하지 않는다고 명시한다. 모든 위임에는 exact file paths, acceptance criteria, out-of-scope와 evidence 요구를 포함한다.

- [x] **Step 3: 역할 계약을 검증한다**

Run:

```powershell
rg -n "미션|필요한 입력|생성할 출력|도구·스킬|금지 사항|Handoff|실패·상향|완료 조건|sourceRefs|waitingForHuman|exact file|acceptance|out-of-scope|evidence" docs/harness/agents/orchestrator.md
```

Expected: 역할 카드 필수 섹션과 위임 계약이 모두 출력된다.

---

### Task 6: Document Parser 역할 카드

**Files:**
- Create: `docs/harness/agents/document-parser.md`
- Read: `docs/harness/contracts/canonical-document.md`
- Read: `docs/harness/contracts/handoff-packet.md`

**Interfaces:**
- Consumes: 검증된 원본, `sourceHash`, 포맷 정책.
- Produces: Canonical Document, assets, `parseWarnings`, handoff packet.

- [x] **Step 1: 역할 카드 부재를 확인한다**

Run: `Test-Path -LiteralPath 'docs\harness\agents\document-parser.md'`

Expected: `False`.

- [x] **Step 2: 역할 카드를 작성한다**

공통 역할 카드 8개 섹션을 사용한다. `ocr`, `test-driven-development`, `systematic-debugging`, `verification-before-completion`의 사용 조건을 적는다. HWP 5.x와 HWPX parser 분리, 외부 네트워크 차단, 파일·페이지 제한, 낮은 OCR 커버리지와 표·이미지·수식 누락 시 구조 검수로 올리는 규칙을 포함한다.

- [x] **Step 3: 역할 계약을 검증한다**

Run:

```powershell
rg -n "미션|필요한 입력|생성할 출력|도구·스킬|금지 사항|Handoff|실패·상향|완료 조건|HWP 5.x|HWPX|OCR|외부 네트워크|parseWarnings|sourceRefs" docs/harness/agents/document-parser.md
```

Expected: 포맷·보안·근거·상향 규칙이 모두 출력된다.

---

### Task 7: Lesson Builder 역할 카드

**Files:**
- Create: `docs/harness/agents/lesson-builder.md`
- Read: `docs/harness/contracts/canonical-document.md`
- Read: `docs/harness/contracts/review-gates.md`

**Interfaces:**
- Consumes: `structureReview`가 승인된 Canonical Document와 생성 프리셋.
- Produces: 문항별 `sourceRefs`와 audience 정보를 가진 version형 draft package.

- [x] **Step 1: 역할 카드 부재를 확인한다**

Run: `Test-Path -LiteralPath 'docs\harness\agents\lesson-builder.md'`

Expected: `False`.

- [x] **Step 2: 역할 카드를 작성한다**

공통 역할 카드 8개 섹션에 `edu-content`, `test-driven-development`, `verification-before-completion` 사용 조건을 적는다. 구조 승인 전 생성 금지, 근거 없는 정답 금지, 학생용 정답 분리, 어휘·문항·해설·수업안의 제한적 병렬 생성, 오류 field → question → section → fallback → 강사 검수 순서를 포함한다.

- [x] **Step 3: 역할 계약을 검증한다**

Run:

```powershell
rg -n "미션|필요한 입력|생성할 출력|도구·스킬|금지 사항|Handoff|실패·상향|완료 조건|structureReview|sourceRefs|학생용|field|question|section|fallback" docs/harness/agents/lesson-builder.md
```

Expected: 승인·근거·audience·좁은 재시도 규칙이 모두 출력된다.

---

### Task 8: Evidence Content QA 역할 카드

**Files:**
- Create: `docs/harness/agents/evidence-content-qa.md`
- Read: `docs/harness/contracts/canonical-document.md`
- Read: `docs/harness/contracts/review-gates.md`
- Read: `docs/harness/agents/lesson-builder.md`

**Interfaces:**
- Consumes: Canonical Document와 생성 draft package.
- Produces: severity별 QA report, 실패 항목, `retryScope`, gate recommendation.

- [x] **Step 1: 역할 카드 부재를 확인한다**

Run: `Test-Path -LiteralPath 'docs\harness\agents\evidence-content-qa.md'`

Expected: `False`.

- [x] **Step 2: 역할 카드를 작성한다**

공통 역할 카드 8개 섹션에 `edu-content`, `code-review`, `systematic-debugging`, `verification-before-completion` 사용 조건을 적는다. 구조 번호·보기·정답, source quote/page/block, 중복, 복수 정답, CEFR, distractor, 학생·교사 일치와 answer leak을 검사한다. QA는 직접 승인하거나 실패 항목을 추측 보완하지 않는다고 명시한다.

- [x] **Step 3: 양쪽 동시 읽기 원칙을 기록한다**

생산자·소비자 계약을 함께 비교하도록 다음 쌍을 명시한다.

```text
parser output ↔ Canonical Document contract
builder draft ↔ sourceRefs and preset
review gate targetVersion ↔ rendered documentVersion
student audience manifest ↔ actual student output
```

- [x] **Step 4: 역할 계약을 검증한다**

Run:

```powershell
rg -n "미션|필요한 입력|생성할 출력|도구·스킬|금지 사항|Handoff|실패·상향|완료 조건|source quote|page|block|중복|복수 정답|CEFR|distractor|answer leak|양쪽" docs/harness/agents/evidence-content-qa.md
```

Expected: 교육 QA와 통합 정합성 검증이 모두 출력된다.

---

### Task 9: Renderer Export 역할 카드

**Files:**
- Create: `docs/harness/agents/renderer-export.md`
- Read: `docs/harness/contracts/export-manifest.md`
- Read: `docs/harness/contracts/review-gates.md`

**Interfaces:**
- Consumes: 두 gate가 승인한 immutable version과 출력 설정.
- Produces: audience별 PDF/DOCX/ZIP, checksum, export manifest, preflight 결과.

- [x] **Step 1: 역할 카드 부재를 확인한다**

Run: `Test-Path -LiteralPath 'docs\harness\agents\renderer-export.md'`

Expected: `False`.

- [x] **Step 2: 역할 카드를 작성한다**

공통 역할 카드 8개 섹션에 `workbook`, `nonfiction`, `systematic-debugging`, `verification-before-completion`의 조건부 사용을 적는다. 승인 콘텐츠 변경 금지, 학생·교사·정답 audience 분리, 이전 version 덮어쓰기 금지, idempotency key, 의미 일치와 실제 페이지 preflight를 포함한다.

- [x] **Step 3: 역할 계약을 검증한다**

Run:

```powershell
rg -n "미션|필요한 입력|생성할 출력|도구·스킬|금지 사항|Handoff|실패·상향|완료 조건|contentReview|student|teacher|answer|PDF|DOCX|ZIP|checksum|idempotency|preflight|덮어쓰" docs/harness/agents/renderer-export.md
```

Expected: 승인본·audience·재실행·출력 검증 규칙이 모두 출력된다.

---

### Task 10: 실행 기록, 교정 기록과 통합 검증

**Files:**
- Create: `docs/harness/runs/README.md`
- Create: `docs/harness/runs/2026-08-24-harness-bootstrap/verification.md`
- Create: `tasks/lessons.md`
- Modify: `docs/harness/manifest.md`
- Verify: `AGENTS.md`
- Verify: `docs/harness/**/*.md`

**Interfaces:**
- Consumes: 앞선 모든 manifest, 계약과 역할 카드.
- Produces: 실행별 증거 형식, 교정 학습 형식, 하네스 전체 구조 검증 결과.

- [x] **Step 1: 기록 파일 부재를 확인한다**

Run:

```powershell
Test-Path -LiteralPath 'docs\harness\runs\README.md'
Test-Path -LiteralPath 'tasks\lessons.md'
```

Expected: 두 줄 모두 `False`.

- [x] **Step 2: 실행 기록 규칙을 작성한다**

`docs/harness/runs/README.md`에 `YYYY-MM-DD-task-name/` 규칙과 다음 허용 필드를 적는다.

```markdown
- 작업 목적과 범위
- 입력의 비민감 식별자와 hash
- 담당 역할과 handoff 상태
- 실행한 실제 명령과 종료 코드
- 검증 결과와 미검증 항목
- 사람 gate 결정의 비민감 메타데이터
- 변경 파일과 재시도 범위
```

원문, OCR 본문, 정답 초안, 내부 프롬프트, API key, 개인정보는 기록하지 않는다고 명시한다.

- [x] **Step 3: 교정 기록 형식을 작성한다**

`tasks/lessons.md`에 다음 형식을 작성한다.

```markdown
# Lessons

사용자 교정이 있을 때만 다음 필드를 추가한다.

- Date
- Observed failure pattern
- Generalized cause
- Affected role or contract
- Prevention rule
- Verification added
```

초기에는 실제 교정 사례를 만들지 않는다.

- [x] **Step 4: 계획된 파일 전체를 검증한다**

Run:

```powershell
$required = @(
  'AGENTS.md',
  'docs\harness\manifest.md',
  'docs\harness\agents\orchestrator.md',
  'docs\harness\agents\document-parser.md',
  'docs\harness\agents\lesson-builder.md',
  'docs\harness\agents\evidence-content-qa.md',
  'docs\harness\agents\renderer-export.md',
  'docs\harness\contracts\handoff-packet.md',
  'docs\harness\contracts\canonical-document.md',
  'docs\harness\contracts\review-gates.md',
  'docs\harness\contracts\export-manifest.md',
  'docs\harness\runs\README.md',
  'tasks\lessons.md'
)
$missing = $required | Where-Object { -not (Test-Path -LiteralPath $_) }
if ($missing.Count -gt 0) { $missing; exit 1 }
"required_files=$($required.Count)"
```

Expected: `required_files=13`, exit code `0`. 최종 증거 파일은 Step 8에서 생성·확인한다.

- [x] **Step 5: 링크·역할·계약 일관성을 검증한다**

Run:

```powershell
rg -n "orchestrator.md|document-parser.md|lesson-builder.md|evidence-content-qa.md|renderer-export.md|handoff-packet.md|canonical-document.md|review-gates.md|export-manifest.md" docs/harness/manifest.md
rg -l "미션" docs/harness/agents -g "*.md"
rg -l "Handoff" docs/harness/agents -g "*.md"
rg -n "sourceRefs|structureReview|contentReview|waitingForHuman|stale|idempotency" docs/harness AGENTS.md
```

Expected: manifest가 9개 하위 문서를 참조하고, 5개 역할 카드 모두 `미션`과 `Handoff`를 포함하며, 핵심 불변식이 관련 문서에 나타난다.

- [x] **Step 6: 트리거 경계 예문을 추가하고 검토한다**

manifest의 should-trigger 사례에 다음 요청을 추가하고 포함 여부를 확인한다.

```text
이 HWPX 교재를 Canonical Document로 파싱하는 기능을 설계해줘.
생성 문항의 정답 근거를 페이지와 block까지 검증해줘.
학생용과 교사용 PDF/DOCX 패키징 흐름을 구현해줘.
```

should-not-trigger 사례에는 다음 요청을 추가하고 포함 여부를 확인한다.

```text
이 교재 소개 글을 블로그에 게시해줘.
이 영어 지문을 MP3로 읽어줘.
NotebookLM 인포그래픽을 만들어줘.
```

- [x] **Step 7: 금지 문자열과 민감 정보 경계를 검증한다**

Run:

```powershell
$forbidden = @(
  ('T' + 'BD'),
  ('T' + 'ODO'),
  ('implement' + ' later'),
  ('fill in' + ' details'),
  'sk-[A-Za-z0-9_-]{20,}',
  'AKIA[0-9A-Z]{16}',
  'BEGIN PRIVATE KEY'
)
$pattern = $forbidden -join '|'
$redFlags = rg -n $pattern AGENTS.md docs/harness tasks/lessons.md
if ($LASTEXITCODE -eq 0) { $redFlags; exit 1 }
'red_flags=0'
```

Expected: `red_flags=0`, exit code `0`.

- [x] **Step 8: 최종 실행 증거를 기록한다**

`docs/harness/runs/2026-08-24-harness-bootstrap/verification.md`를 생성해 변경 파일 목록, Step 4~7의 명령, 종료 코드와 결과를 기록한다. 입력 원문이나 내부 프롬프트는 포함하지 않는다.

Run: `Test-Path -LiteralPath 'docs\harness\runs\2026-08-24-harness-bootstrap\verification.md'`

Expected: `True`. 구조·링크·불변식·민감 정보 검증이 모두 통과하거나, 실패한 항목이 명확히 기록되어 완료 선언이 차단된다.

## Final Review Gate

실행 완료 전 Main/Sol은 다음을 직접 확인한다.

- 스펙 14개 섹션의 각 요구사항이 Task 1~10 중 하나로 구현됐는지 대조한다.
- 생산자와 소비자가 같은 field·status·version 이름을 사용하는지 비교한다.
- 역할 카드가 직접 사람 승인하거나 다른 역할의 책임을 침범하지 않는지 확인한다.
- manifest가 존재하지 않는 코드·서비스·명령을 구현된 것으로 주장하지 않는지 확인한다.
- 검증 증거의 실제 종료 코드가 완료 주장과 일치하는지 확인한다.

문서만 변경한 이번 범위에서는 `npx tsc --noEmit`을 실행하지 않는다. 이후 TypeScript 코드 변경이 포함되면 관련 테스트와 함께 반드시 실행한다.
