# Git 커밋·푸쉬 프로토콜

오케스트레이터가 글 작성 완료 후 자동 커밋·푸쉬할 때 따르는 절차. 사용자는 "글 저장 후 자동 커밋·푸쉬"를 사전 승인했으므로 매번 확인하지 않는다.

## 사전 점검

푸쉬 직전에 한 번만 확인:

1. **현재 디렉토리**가 `gh-trend-digest` 레포 루트인가?
2. **`git status`**로 추적되지 않거나 의도치 않은 변경이 있는지 확인
   - 의도된 변경: `categories/{language}/{repo-name}.md`, `_workspace/*` (선택적)
   - 의도하지 않은 변경이 있으면 → 사용자에게 보고하고 진행 중단
3. **원격 브랜치 추적** 상태 확인 (`git rev-parse --abbrev-ref --symbolic-full-name @{u}`)
   - 추적 브랜치 없으면 → `git push -u origin {branch}` 사용

## 스테이징 규칙

```bash
# 의도된 산출물만 스테이징
git add categories/{language}/{repo-name}.md
```

**`_workspace/`는 커밋하지 않는다.** 중간 산출물(01~05 자료, 클론된 레포)은 로컬 감사 추적용으로만 보존한다. `.gitignore`에 `_workspace/`가 등재되어 있어야 한다 — 없으면 추가하고 같이 커밋한다.

**전체 스테이징 금지**: `git add -A` 또는 `git add .`은 쓰지 않는다. `.env`나 자격증명 등 민감 파일이 섞일 위험.

## 커밋 메시지 컨벤션

```
docs(digest): {owner}/{repo} 리뷰 추가

- 언어: {primary_language}
- 분량: {N}자
- 카테고리: categories/{language}/{repo-name}.md

원본: {repo_url}
```

규칙:
- 타입은 `docs(digest)` 고정 — 리뷰 글 추가는 문서 변경으로 취급
- 제목 줄 한국어 OK, 50자 이내 권장
- 본문에 메타데이터(언어·분량·경로·원본 URL) 4개 항목 포함
- Co-Authored-By 라인은 추가하지 않음 (사용자가 명시적으로 요청하지 않음)

## 커밋 실행

HEREDOC으로 메시지 전달 (포매팅 안전):

```bash
git commit -m "$(cat <<'EOF'
docs(digest): owner/repo 리뷰 추가

- 언어: typescript
- 분량: 3247자
- 카테고리: categories/typescript/repo-name.md

원본: https://github.com/owner/repo
EOF
)"
```

훅 우회 금지: `--no-verify` 사용하지 않는다. pre-commit 훅이 있으면 통과시킨다.

## 푸쉬

```bash
git push
# 추적 브랜치가 없으면:
git push -u origin "$(git rev-parse --abbrev-ref HEAD)"
```

**force push 금지**: 어떤 경우에도 `--force` / `--force-with-lease` 사용 금지. 충돌 발생 시 사용자에게 보고하고 중단.

## 실패 처리

| 실패 유형 | 처리 |
|----------|------|
| pre-commit 훅 실패 | 원인을 사용자에 보고. 임시 우회 시도 금지. |
| 인증 실패 | 사용자에게 `gh auth login` 안내. |
| 푸쉬 충돌 (non-fast-forward) | `git fetch` → 충돌 내용 보고 → 사용자 결정 대기. 자동 rebase/merge 금지. |
| 원격 없음 | 푸쉬 단계만 스킵, 커밋은 보존. 사용자에게 보고. |

## 푸쉬 후 보고

성공 시 사용자에게 한 줄로 요약:

```
✅ 커밋 푸쉬 완료
- {commit_sha} categories/{language}/{repo-name}.md ({N}자)
- 원격: origin/{branch}
```

푸쉬 URL이 있으면 (예: GitHub UI에서 볼 수 있는 경로) 같이 보여준다.
