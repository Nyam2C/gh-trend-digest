# gh-trend-digest

GitHub 트렌딩 레포 URL을 받아 한국어 친근 블로그 톤의 심층 리뷰 글을 작성·저장·게시하는 저장소.

## 하네스: GitHub 트렌딩 리뷰

**목표:** 사용자가 GitHub 레포 URL을 건네면, 자동으로 레포를 분석하고 한국어 블로그 톤 심층 리뷰(3000자 이상)를 작성해 `categories/{language}/{repo-name}.md`에 저장하고 git commit·push까지 마친다.

**트리거:** GitHub 레포 URL(`github.com/owner/repo` 또는 `owner/repo` 단축형)과 함께 "리뷰", "소개글", "디제스트", "트렌딩 정리", "분석" 등의 요청이 오면 `gh-trend-digest` 스킬을 사용하라. "다시 써줘", "톤 수정", "최신 반응 추가" 등 후속 작업도 같은 스킬로 처리. 단순 질문(예: "이 레포 뭐 하는 거야?")은 직접 응답 가능.

**산출물 경로:** `categories/{language}/{repo-name}.md` (예: `categories/typescript/some-repo.md`). 중간 산출물은 `_workspace/`에 로컬 보존하며 **커밋·푸쉬에서 제외**한다 (`.gitignore` 등재).

**변경 이력:**

| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-05-12 | 초기 구성 | 전체 (5 에이전트 + 1 오케스트레이터 스킬 + 2 references) | - |
| 2026-05-12 | `_workspace/` 커밋 제외 | `.gitignore`, `git-publish-protocol.md`, CLAUDE.md | 중간 산출물은 로컬 감사용으로만 보존하고 원격에는 푸쉬하지 않기로 정책 변경 |
| 2026-05-12 | 중복 체크 게이트 추가 (Phase 0-2) | `gh-trend-digest/SKILL.md` | 이미 리뷰가 있는 레포면 재작업하지 않고 알리고 중단. 후속 키워드("다시", "수정" 등)가 있을 때만 재실행 허용 |
