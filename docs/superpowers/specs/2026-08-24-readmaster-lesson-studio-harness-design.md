# ReadMaster Lesson Studio 프로젝트 하네스 설계

## 1. 목적

ReadMaster Lesson Studio를 단순한 AI 문제 생성기가 아니라, 교재를 원문 근거가 연결된 학생용·강사용 수업 패키지로 변환하는 제작 시스템으로 개발하기 위한 프로젝트 로컬 Codex 하네스를 정의한다.

이 하네스는 두 층을 분리한다.

- Codex 개발 하네스: 제품의 계약, 코드, 테스트, 출력 품질을 설계·구현·검증한다.
- Lesson Studio 런타임: 업로드, 파싱, 생성, 강사 승인, 조판을 실제로 수행한다.

역할 카드는 런타임 노드를 흉내 내는 프롬프트가 아니라 각 런타임 영역의 개발 책임과 검증 경계를 정의한다.

## 2. 현재 상태와 범위

`C:\Users\sound\projects\ReadMaster-Saas`는 설계 시점에 파일이 없고 Git 저장소도 아니다. 기존 하네스, 코드, 빌드 명령, CI, 배포 명령과 충돌할 대상이 없으므로 신규 프로젝트 로컬 하네스로 설계한다.

이번 하네스의 범위는 다음과 같다.

- 프로젝트 목표, 트리거, 단계, 역할, 역할 간 계약을 고정한다.
- Canonical Document와 원문 근거 추적을 핵심 불변식으로 둔다.
- 구조 검수와 콘텐츠 검수를 분리한다.
- 강사의 승인 없이는 다음 단계로 진행하지 않는다.
- 문항 또는 Section 단위의 좁은 재시도와 버전 보존을 정의한다.
- 학생용·교사용·정답해설 출력의 분리와 검증을 정의한다.
- Codex의 설계, 구현, QA, 통합 검증 책임을 구분한다.

이번 하네스만으로 구현하지 않는 항목은 다음과 같다.

- 실제 Next.js, FastAPI, LangGraph, Redis, Supabase 코드
- PDF·DOCX·HWP·HWPX 파서와 렌더러
- 데이터베이스 마이그레이션과 배포 인프라
- 결제, 외부 SaaS 테넌시, 실시간 공동편집
- 글로벌 Codex 스킬 신규 생성

## 3. 선택한 아키텍처

기본 패턴은 체크포인트 파이프라인에 제한적 Fan-out/Fan-in과 Generate-Verify를 결합한다.

```text
업로드
→ 문서 파싱
→ 구조 QA
→ [강사 구조 승인]
→ 어휘·문항·해설·수업안 병렬 생성
→ 근거·교육 품질 QA
→ [강사 콘텐츠 승인]
→ 학생용·교사용·정답지 렌더링
→ 출력 검증
→ 버전 패키징
```

MVP에서는 고정 단계와 좁은 병렬 구간이 실패 범위를 가장 명확하게 만든다. 포맷과 프리셋이 크게 늘어나는 외부 SaaS 단계가 되면 Supervisor와 Expert Pool을 이 파이프라인 위에 추가할 수 있다. 계층적 위임은 현재 범위에서 사용하지 않는다.

## 4. 역할

### orchestrator

- 미션: 작업 상태, 체크포인트, 승인 대기, 재시도, 통합과 최종 완료 판정을 관리한다.
- 입력: 프로젝트 설정, 단계별 handoff packet, QA 결과, 강사 결정.
- 출력: 실행 계획, 상태 전이 기록, 승인 기록, 재시작 지점, 최종 manifest 또는 차단 보고서.
- 금지: 직접 파싱·콘텐츠 생성·조판을 수행하지 않고 강사 승인을 대신하지 않는다.
- 실패 처리: 스키마나 승인 기록이 없거나 반복 실패가 발생하면 범위를 넓히지 않고 차단 상태로 보고한다.

### document-parser

- 미션: PDF, DOCX, HWP 5.x, HWPX, 이미지 문서를 Canonical Document JSON으로 변환한다.
- 입력: 업로드 파일, 파일 해시, 포맷 정책, OCR 결과, 파서 fixture.
- 출력: 페이지·Section·Block·Question·Asset 인덱스, `parseWarnings`, source reference index.
- 금지: 파싱 중 원문을 재작성하거나 콘텐츠를 생성하지 않는다. HWP 5.x와 HWPX를 같은 파서로 처리하지 않는다.
- 실패 처리: 손상, 낮은 OCR 커버리지, 표·수식·이미지 누락은 생성 단계로 넘기지 않고 구조 검수로 보낸다.

### lesson-builder

- 미션: 구조 승인된 문서에서 어휘, 테스트 A/B, 해설, 수업안과 분석 리포트를 생성한다.
- 입력: 승인된 Canonical Document version, 프리셋, 학년·CEFR·난도, 승인 예시.
- 출력: 안정적인 ID와 `sourceRefs`를 가진 버전형 draft package.
- 금지: 구조 승인 전 생성, 근거 없는 정답·해설, 학생용 정답 노출, 직접 발행을 하지 않는다.
- 실패 처리: 오류 필드 재생성, Section 축소, 대체 모델, 문항 격리 순으로 좁게 복구한다.

### evidence-content-qa

- 미션: 파서 구조와 생성 콘텐츠를 독립적으로 검증한다.
- 입력: Canonical Document, 파서 리포트, 생성 package, source reference index, 프리셋.
- 출력: 심각도별 QA 리포트, 실패 문항, 재생성 범위, 사람 판단이 필요한 항목.
- 검증: 문항 번호·보기·정답 구조, 원문 도출 가능성, 근거 위치, 중복, CEFR, 복수 정답, 오답 품질, 학생·교사 버전 일치, 정답 노출.
- 금지: QA가 직접 승인하거나 근거가 없는 항목을 추측으로 보완하지 않는다.

### renderer-export

- 미션: 승인된 immutable content version을 학생용·교사용·정답해설 PDF/DOCX와 ZIP으로 출력한다.
- 입력: 콘텐츠 승인본, 템플릿과 브랜딩 설정, audience manifest.
- 출력: 파일, checksum, export manifest, 의미 일치와 시각 검수 결과.
- 금지: 승인된 콘텐츠 변경, 이전 버전 덮어쓰기, 학생용 정답 삽입, 실제 페이지 검수 전 `print-ready` 선언을 하지 않는다.
- 실패 처리: 의미 오류는 builder·QA로 돌리고 레이아웃 오류만 렌더 단계에서 복구한다.

## 5. 핵심 불변식과 사람 승인

모든 단계는 다음 두 규칙을 만족해야 한다.

```text
sourceRefs가 없으면 다음 단계로 전달하지 않는다.
승인된 version이 아니면 조판하지 않는다.
```

사람 승인은 AI 역할이 아니라 durable checkpoint다.

- `structureReview`: 문항 번호, 본문 연결, 보기 순서, 정답, Section 경계, 표·이미지·수식 누락을 강사가 수정·승인·반려한다.
- `contentReview`: 문항, 해설, 난도와 오답을 강사가 승인·재생성·보류·삭제한다.

각 결정은 actor, timestamp, 대상 version, 결정, 사유를 남긴다. 응답이 없으면 `waitingForHuman`에 머물며 자동 승인이나 자동 발행을 하지 않는다.

## 6. 파일 구조

```text
AGENTS.md
docs/harness/
├── manifest.md
├── agents/
│   ├── orchestrator.md
│   ├── document-parser.md
│   ├── lesson-builder.md
│   ├── evidence-content-qa.md
│   └── renderer-export.md
├── contracts/
│   ├── handoff-packet.md
│   ├── canonical-document.md
│   ├── review-gates.md
│   └── export-manifest.md
└── runs/
    └── README.md

tasks/
├── SPEC-readmaster-lesson-studio-harness.md
└── lessons.md
```

`AGENTS.md`는 전체 지침을 복제하지 않고 `docs/harness/manifest.md`를 가리키는 짧은 진입점으로 유지한다. 실행 기록에는 민감한 원문 대신 해시, 버전, 상태와 안전한 경로만 기록한다.

## 7. 역할 간 계약

공통 handoff packet은 다음 필드를 가진다.

```text
jobId
projectId
sourceHash
documentVersion
schemaVersion
artifactPath
status
warnings[]
requiredHumanGate
retryScope
createdAt
```

Canonical Document 계약은 페이지, Section, Block, Question, Asset, ParseWarning과 필수 `sourceRefs`를 정의한다. Export manifest는 audience별 파일, 입력 version, 템플릿·렌더러 version과 checksum을 기록한다.

파서 수정으로 입력 version이 바뀌면 영향받는 Section과 파생 산출물만 `stale` 처리한다. 승인 이력이 없는 version과 stale 산출물은 렌더링할 수 없다.

## 8. Codex 실행 라우팅과 스킬 지도

- Main/Sol: 요구사항 해석, 아키텍처와 계약 결정, 통합 판단, 최종 검증.
- `luna_worker`: 파일 범위와 완료 조건이 독립적인 조사·구현 작업.
- QA 역할: 구현자와 분리해 생산자와 소비자 양쪽 계약을 동시에 검증.
- 사람: 스펙 승인, 범위 확장, 공개·배포 결정.

재사용할 기존 스킬은 `harness`, `brainstorming`, `writing-plans`, `ocr`, `edu-content`, `system-reference`, `test-driven-development`, `systematic-debugging`, `code-review`, `verification-before-completion`, `supabase-patterns`다. `workbook`과 `nonfiction`은 학생·교사 분리와 인쇄 preflight 패턴이 필요할 때만 참고한다.

초기에는 글로벌 스킬을 만들지 않는다. 반복되는 ReadMaster 고유 계약이 확인되면 `readmaster-canonical-document`와 `readmaster-package-gates`를 프로젝트 로컬 스킬 후보로 검토한다.

블로그, TTS, NotebookLM, Google ADK, Adobe 템플릿과 미디어 생성 스킬은 이 하네스의 핵심 경로에서 제외한다.

## 9. 검증

### 구조 검증

- 필수 파일과 링크가 실제로 존재한다.
- 모든 Phase에 입력, 출력과 담당 역할이 있다.
- 역할 카드와 계약 문서의 handoff 필드가 일치한다.
- 트리거와 near-miss 사례의 경계가 명확하다.

### 구현 검증

- 구현 변경은 관련 테스트를 먼저 실행한다.
- TypeScript 변경 후 `npx tsc --noEmit`을 실행한다.
- `package.json`이 생긴 뒤 실제 존재가 확인된 명령만 manifest에 기록한다.
- API 응답과 프론트 타입, 상태 전이와 상태 업데이트, DB 필드와 API 변환처럼 경계의 양쪽을 함께 검증한다.
- 빌드 성공만으로 완료를 선언하지 않는다.

### 제품 품질 검증

- `sourceRefs` 완전성, 승인 version, 문항 단위 재시도, 학생용 정답 차단을 확인한다.
- PDF와 DOCX의 의미 일치, overflow, glyph, 빈 페이지와 실제 페이지 출력 품질을 확인한다.

현재 저장소에는 실행 코드와 `package.json`이 없으므로 이번 문서 단계에서는 TypeScript, 테스트와 빌드 명령을 실행하지 않는다.

## 10. 공개·비공개 경계

비공개 기본값은 원본 교재, OCR·추출 텍스트, Canonical Document JSON, 정답·해설 초안, 강사 메모, 내부 프롬프트, 모델 비용, 작업 로그다.

공유 가능한 항목은 강사가 승인하고 권한을 부여한 학생용·교사용·정답해설 출력물뿐이다. 학생용 산출물에는 정답과 내부 검수 메타데이터가 포함될 수 없다.

## 11. 실패와 재시도

- 손상·악성·제한 초과 업로드는 자동 재시도하지 않는다.
- 일시적인 파싱 실패는 제한 횟수만 재시도하고 포맷별 fallback 또는 OCR로 전환한다.
- 구조화 출력 오류는 검증, 오류 필드 재생성, Section 축소, 대체 모델 순으로 처리한다.
- 콘텐츠 QA 실패는 해당 문항만 최대 2~3회 재생성하고 반복 실패하면 강사 검수로 격리한다.
- 렌더링과 파일 저장은 idempotency key로 재시도하며 승인 콘텐츠를 바꾸지 않는다.
- 부분 성공을 보존하며 승인 대기는 자동으로 해제하지 않는다.

## 12. 진화 규칙

- 사용자 교정은 `tasks/lessons.md`에 재사용 가능한 실패 패턴으로 기록한다.
- 같은 실패가 반복되면 역할 카드, 계약과 검증 항목을 함께 수정한다.
- 재사용성이 입증된 절차만 프로젝트 로컬 스킬로 승격한다.
- 변경 이력에는 날짜, 대상, 이유와 검증 결과를 기록한다.
- 중복 역할이나 스킬을 추가하지 않고 기존 책임을 확장하거나 병합한다.

## 13. 완료 기준

하네스 구현은 다음 조건을 모두 만족할 때 완료된다.

- 계획된 모든 파일이 존재하고 링크가 유효하다.
- 다섯 역할 카드에 미션, 입력, 출력, 허용 도구·스킬, 금지 사항, handoff, 실패·상승 규칙이 있다.
- manifest에 트리거, 단계 순서, 입력·출력 경로, 승인 gate, 검증, fallback과 변경 이력이 있다.
- 네 계약 문서가 역할 카드와 동일한 필드·상태를 사용한다.
- 공개·비공개 경계와 버전 덮어쓰기 금지가 명시돼 있다.
- should-trigger, should-not-trigger, 정상 흐름과 실패 흐름을 문서 수준에서 검증할 수 있다.
- 실제 코드 명령은 존재가 확인되기 전까지 하네스의 실행 명령으로 주장하지 않는다.
