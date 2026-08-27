# Git 저장소 초기화 및 첫 배포 계획

## 목표

현재 ReadMaster Lesson Studio 하네스 문서를 Git 저장소로 초기화하고,
`https://github.com/Reasonofmoon/readmaster-saas.git`의 `main` 브랜치에 첫 커밋을 게시한다.

## 범위

- 공개 가능한 프로젝트 문서와 하네스 계약을 첫 커밋에 포함한다.
- 비밀값, 로컬 환경 파일, 의존성·빌드 결과와 `.superpowers/` 실행 스냅샷은 제외한다.
- 문서 구조와 민감정보 검사를 통과한 뒤 Conventional Commit을 생성한다.
- push 후 원격 `main`의 commit hash를 로컬 `HEAD`와 대조한다.

## 실행 단계

- [x] 커밋 대상과 민감정보 경계를 감사한다.
- [x] Git 저장소, `main` 브랜치와 `origin` 원격을 구성한다.
- [x] 문서 검증 후 `docs: bootstrap ReadMaster Lesson Studio harness` 커밋을 생성한다.
- [x] `origin/main`에 push하고 원격 commit hash를 확인한다.
