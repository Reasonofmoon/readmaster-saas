# Task 10 Harness Bootstrap Verification

## 실행 범위와 상태

- Run ID: `2026-08-24-harness-bootstrap`
- 범위: Lesson Studio 하네스 문서의 실행 기록 형식, 교정 기록 형식, 트리거 경계 예시와 구조·링크·불변식·레드플래그 검증
- 상태: `complete`
- 실행 유형: 문서 전용 검증

이 run-record는 Main/Sol Final Review Gate가 Critical `0`, Important `0`, Minor `0`으로
승인된 뒤 Main이 `complete`로 봉인했다. 이 값은 handoff packet 상태 enum이 아니다.

## 허용 필드 매핑

상단 요약 필드는 `runs/README.md`의 허용 필드에 다음처럼 매핑된다.

- `Run ID` → 입력의 비민감 식별자와 hash: 원문 대신 실행을 식별하는 비민감 run ID를 사용한다.
- `상태` → 담당 역할과 handoff 상태: Main/Sol Final Review Gate 승인 뒤의 `complete` 상태를 기록한다.
- `실행 유형` 및 `범위` → 작업 목적과 범위: 문서 전용 검증의 목적과 대상 범위를 설명한다.
- 명령·종료 코드, 검증 결과·미검증 항목, 변경 파일은 아래 검증 섹션과 변경 파일 목록에 기록한다.

원문 교재, OCR 본문, 정답·해설 내용, 내부 프롬프트, 비밀값, 학생 개인정보는 이
기록에 넣지 않았다. 입력은 비민감한 파일 경로·문서 식별자와 검증 결과로만 다뤘다.

## 변경 파일

- `docs/harness/manifest.md` — should-trigger와 should-not-trigger 경계 예시 및 변경 이력
- `docs/harness/runs/README.md` — 실행 기록 디렉터리·필드·민감 정보 경계
- `docs/harness/runs/2026-08-24-harness-bootstrap/verification.md` — 이 실행의 검증 증거
- `tasks/lessons.md` — 사용자 교정 기반 lessons 형식
- `.superpowers/sdd/task-10-report.md` — Task 10 handoff report

## 검증 명령과 결과

### Step 4 — 계획된 핵심 파일 구조

실행 명령은 Task 10 brief의 `$required` 배열과 `Test-Path -LiteralPath` 검사를
그대로 사용했다. 최종 증거 파일은 이 단계 뒤에 생성했으므로 배열에는 포함하지
않았다.

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

- 종료 코드: `0`
- 결과: `required_files=13`
- 해석: 계획된 핵심 파일 13개가 모두 존재했다.

### Step 5 — 링크·역할·계약·핵심 불변식

실행 명령:

```powershell
rg -n "orchestrator.md|document-parser.md|lesson-builder.md|evidence-content-qa.md|renderer-export.md|handoff-packet.md|canonical-document.md|review-gates.md|export-manifest.md" docs/harness/manifest.md
rg -l "미션" docs/harness/agents -g "*.md"
rg -l "Handoff" docs/harness/agents -g "*.md"
rg -n "sourceRefs|structureReview|contentReview|waitingForHuman|stale|idempotency" docs/harness AGENTS.md
```

- 종료 코드: 각 명령 `0`
- 결과: manifest에서 역할 5개와 계약 4개, 총 9개 링크가 확인됐다. `미션`과
  `Handoff` 검색 모두 5개 역할 카드에서 확인됐다. 핵심 불변식과 상태·재시도
  용어가 `AGENTS.md`, manifest, 역할 카드와 계약 문서에서 확인됐다.

### Step 6 — 트리거 경계 예시와 분류

#### 최초 assertion (재현 가능한 실패)

실행 명령:

```powershell
$manifest = Get-Content -Raw -LiteralPath 'docs\harness\manifest.md'
$shouldTrigger = @(
  '이 HWPX 교재를 Canonical Document로 파싱하는 기능을 설계해줘.',
  '생성 문항의 정답 근거를 페이지와 block까지 검증해줘.',
  '학생용과 교사용 PDF/DOCX 패키징 흐름을 구현해줘.'
)
$shouldNotTrigger = @(
  '이 교재 소개 글을 블로그에 게시해줘.',
  '이 영어 지문을 MP3로 읽어줘.',
  'NotebookLM 인포그래픽을 만들어줘.'
)
$triggerHeading = $manifest.IndexOf('### should-trigger 예시')
$notTriggerHeading = $manifest.IndexOf('### should-not-trigger 예시')
$missing = @($shouldTrigger + $shouldNotTrigger | Where-Object { -not $manifest.Contains($_) })
$misclassified = @($shouldTrigger | Where-Object { $_ -notin $manifest.Substring($triggerHeading, $notTriggerHeading - $triggerHeading) })
$misclassified += @($shouldNotTrigger | Where-Object { $_ -notin $manifest.Substring($notTriggerHeading) })
if ($triggerHeading -lt 0 -or $notTriggerHeading -lt 0 -or $missing.Count -gt 0 -or $misclassified.Count -gt 0) {
  "missing_or_misclassified=$($missing.Count + $misclassified.Count)"
  exit 1
}
"trigger_examples=$($shouldTrigger.Count); should_not_trigger_examples=$($shouldNotTrigger.Count); classification=pass"
```

- 종료 코드: `1`
- 출력: `missing_or_misclassified=6`
- stderr/error: 없음. 원인은 PowerShell에서 문자열과 배열의 `-notin` 비교를 사용해
  섹션 문자열 포함 여부를 판정한 assertion 로직 오류이며, manifest 분류 실패가 아니다.

#### 교정 assertion (재현 가능한 통과)

실행 명령:

```powershell
$manifest = Get-Content -Raw -LiteralPath 'docs\harness\manifest.md'
$shouldTrigger = @(
  '이 HWPX 교재를 Canonical Document로 파싱하는 기능을 설계해줘.',
  '생성 문항의 정답 근거를 페이지와 block까지 검증해줘.',
  '학생용과 교사용 PDF/DOCX 패키징 흐름을 구현해줘.'
)
$shouldNotTrigger = @(
  '이 교재 소개 글을 블로그에 게시해줘.',
  '이 영어 지문을 MP3로 읽어줘.',
  'NotebookLM 인포그래픽을 만들어줘.'
)
$triggerHeading = $manifest.IndexOf('### should-trigger 예시')
$notTriggerHeading = $manifest.IndexOf('### should-not-trigger 예시')
$triggerSection = if ($triggerHeading -ge 0 -and $notTriggerHeading -gt $triggerHeading) { $manifest.Substring($triggerHeading, $notTriggerHeading - $triggerHeading) } else { '' }
$notTriggerSection = if ($notTriggerHeading -ge 0) { $manifest.Substring($notTriggerHeading) } else { '' }
$missing = @($shouldTrigger + $shouldNotTrigger | Where-Object { -not $manifest.Contains($_) })
$misclassified = @($shouldTrigger | Where-Object { -not $triggerSection.Contains($_) })
$misclassified += @($shouldNotTrigger | Where-Object { -not $notTriggerSection.Contains($_) })
if ($triggerHeading -lt 0 -or $notTriggerHeading -lt 0 -or $missing.Count -gt 0 -or $misclassified.Count -gt 0) {
  "missing_or_misclassified=$($missing.Count + $misclassified.Count)"
  exit 1
}
"trigger_examples=$($shouldTrigger.Count); should_not_trigger_examples=$($shouldNotTrigger.Count); classification=pass"
```

- 종료 코드: `0`
- 출력: `trigger_examples=3; should_not_trigger_examples=3; classification=pass`
- 해석: 3개 Lesson Studio 요청은 should-trigger 영역에, 3개 외부 콘텐츠·미디어
  요청은 should-not-trigger 영역에 정확히 포함됐다.

### Step 7 — 금지 문자열·민감 정보 경계

실행 명령: Task 10 brief의 forbidden pattern 배열을 구성하고 다음 검색을 실행했다.
패턴의 원문 literal은 실행 기록에 반복하지 않았다.

```powershell
$pattern = $forbidden -join '|'
$redFlags = rg -n $pattern AGENTS.md docs/harness tasks/lessons.md
if ($LASTEXITCODE -eq 0) { $redFlags; exit 1 }
'red_flags=0'
```

- 종료 코드: `0`
- 결과: `red_flags=0`
- 해석: 대상 문서와 lessons 파일에서 금지 문자열·비밀값 형태의 red flag가 발견되지
  않았다.
- handoff report를 변경 파일 목록에 추가한 뒤 동일 검사를 다시 실행했고, 종료 코드
  `0`과 `red_flags=0`을 재확인했다.

### Step 8 — 최종 증거 파일 존재 확인

```powershell
Test-Path -LiteralPath 'docs\harness\runs\2026-08-24-harness-bootstrap\verification.md'
```

- 실행 시점: 이 증거 파일 생성 직후
- 결과: `True`
- 종료 코드: `0`

## 미검증·미해결 항목

- 문서 전용 범위이므로 TypeScript, 런타임 앱, 의존성, 네트워크와 외부 서비스 검증은
  수행하지 않았다.
- `npx tsc --noEmit`은 Task 10 brief의 문서 전용 예외에 따라 실행하지 않았다.

## Final Sol Gate

- 최종 판정: `APPROVED`
- findings: Critical `0`, Important `0`, Minor `0`
- 스펙 섹션 `14`, Plan Task `10`, 미체크 단계 `0`, 깨진 manifest 링크 `0`
- 비-Orchestrator lifecycle 직접 assignment `0`, 보안 marker 오류 `0`
- Answer-leak retry 표 `3`개 동일, categorical routing contradiction `0`
- `ExportManifest` 필드 `9`, `ExportFile` 필드 `5`
- candidate → renderer preflight → immutable output-QA → manifest seal 순서 확인
- 민감정보 red flag `0`
- 문서 전용 범위이며 `package.json`과 TypeScript 대상 파일이 없어 `npx tsc --noEmit`은
  실행하지 않았다.

## Final Main verification

- 첫 renderer 순서 assertion은 `IndexOf('candidate를 생성')`가 실제 문구와 일치하지 않아
  exit `1`, `renderer flow marker missing`을 반환했다. 문서 흐름 실패가 아니라 검사 marker
  선택 오류였다.
- 실제 단계 제목과 본문 marker인 `### 4. Render candidate와 renderer preflight` →
  `candidate에 대해 semantic·layout·integrity preflight` →
  `### 6. Output-QA로 Evidence Content QA에 전달` →
  `### 7. Evidence output-QA와 manifest seal`로 assertion을 교정했다.
- 교정된 전체 검증은 exit `0`, `FINAL_VERIFICATION=PASS`를 반환했다.
- 결과: 필수 파일 `19`/누락 `0`, 스펙 섹션 `14`, Plan Task `10`, 미체크 단계 `0`,
  역할 카드 `5`, 비-Orchestrator lifecycle assignment `0`, 보안 marker `pass`,
  동일 Answer-leak retry 표 `3`, ExportManifest/ExportFile `9`/`5`, 트리거 `3+3`/
  오분류 `0`, red flag `0`, run state `complete`.
