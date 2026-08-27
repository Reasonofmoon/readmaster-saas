# ReadMaster Lesson Studio Agent Entry Point

프로젝트 작업 전 `docs/harness/manifest.md`와 현재 작업에 해당하는 역할·계약 문서를 읽는다.

핵심 불변식:
- `sourceRefs`가 없는 생성 항목은 다음 단계로 넘기지 않는다.
- `structureReview`와 `contentReview`를 통과한 version만 조판한다.
- 강사 승인과 공개·배포 결정을 AI가 대신하지 않는다.
- 코드 변경 후 `npx tsc --noEmit`을 실행하고 관련 테스트만 먼저 수행한다.
