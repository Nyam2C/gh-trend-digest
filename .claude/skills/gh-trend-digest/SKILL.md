---
name: gh-trend-digest
description: GitHub 트렌딩 레포 URL을 받아 한국어 블로그 톤의 심층 리뷰 글(3000자 이상)을 작성하고 categories/{language}/{repo}.md에 저장한 뒤 자동으로 커밋·푸쉬한다. 사용자가 GitHub 레포 URL(github.com/owner/repo)을 건네며 "리뷰해줘", "소개글 써줘", "트렌딩 정리해줘", "디제스트", "이 레포 분석해줘"라고 하면 반드시 이 스킬을 실행하라. "다시 써줘", "재실행", "이 부분만 보완", "톤 가볍게", "최신 반응 추가" 등 후속 작업 요청도 이 스킬로 처리한다.
---

# gh-trend-digest — GitHub 트렌딩 레포 리뷰 오케스트레이터

## 실행 모드

**에이전트 팀** (기본) — 5명 팀: `repo-investigator`, `code-analyst`, `context-researcher`, `content-writer`, `quality-reviewer`. 자료 수집은 병렬, 글 작성·검증은 순차.

## Phase 0: 입력 파싱 및 컨텍스트 확인

### 0-1. URL 파싱

사용자 메시지에서 GitHub 레포 URL을 추출한다. 지원 패턴:
- `https://github.com/owner/repo`
- `https://github.com/owner/repo/`
- `github.com/owner/repo`
- `owner/repo` (단축 표기)

추출 실패 시 사용자에게 "어떤 레포인가요? URL을 알려주세요"로 묻고 중단.

### 0-2. 컨텍스트 확인

작업 모드를 결정:

1. **`_workspace/` 디렉토리에 이전 산출물이 있는가?**
   - 없음 → **초기 실행** (Phase 1로)
   - 있음 + 같은 `repo_name` + 사용자가 부분 수정 요청("이 섹션만", "톤 더 가볍게", "최신 반응 추가") → **부분 재실행** (해당 에이전트만 호출, 다른 산출물 보존)
   - 있음 + 다른 `repo_name` 또는 사용자가 "새로 써줘" → **새 실행** (기존 `_workspace/`를 `_workspace_prev_{timestamp}/`로 백업 후 새로)

2. 부분 재실행 시 어느 에이전트를 다시 호출할지 매핑:
   - "메타데이터 갱신" → `repo-investigator`
   - "코드 분석 보강", "이 모듈 깊이" → `code-analyst`
   - "최신 반응", "경쟁 도구" → `context-researcher`
   - "톤", "분량", "구조" → `content-writer` (자료는 보존)
   - "다시 검수" → `quality-reviewer`

## Phase 1: 사전 준비

### 1-1. 의존성 확인

```bash
command -v gh >/dev/null 2>&1  # gh CLI (선택, 통계 정확도 향상)
command -v git >/dev/null 2>&1  # 필수
```

`gh`가 없어도 진행 가능. `git`이 없으면 중단.

### 1-2. 얕은 클론

```bash
mkdir -p _workspace
git clone --depth 50 {repo_url} _workspace/repo
```

- depth 50: 최근 활동 분석에 충분, 큰 레포에서도 빠름
- 클론 실패 시 사용자에게 보고하고 중단

### 1-3. 메타 정보 미리 추출

```bash
# 추후 에이전트들이 빠르게 참조할 수 있도록
git -C _workspace/repo log -1 --format='%ad' --date=short > _workspace/.meta_last_commit
git -C _workspace/repo log --reverse -1 --format='%ad' --date=short > _workspace/.meta_first_commit
```

## Phase 2: 자료 수집 (에이전트 팀)

**실행 모드:** 에이전트 팀 (병렬 자료 수집)

### 2-1. 팀 생성 및 작업 할당

`TeamCreate`로 다음 3명을 한 팀에 묶는다:
- `repo-investigator`
- `code-analyst`
- `context-researcher`

`TaskCreate`로 3개 작업을 만들고 각 에이전트에게 owner 할당:

| Task ID | Owner | Description |
|---------|-------|-------------|
| T1 | repo-investigator | 표면 정보 수집 → `_workspace/01_investigator_facts.md` |
| T2 | code-analyst | 코드 분석 → `_workspace/02_analyst_review.md` (T1 진입점 정보 활용 — T1 완료 후 시작) |
| T3 | context-researcher | 외부 컨텍스트 조사 → `_workspace/03_researcher_context.md` (T1 결과로 검색 키워드 좁히면 더 좋음 — T1과 병렬도 OK) |

의존성:
- T2 `addBlockedBy` T1 (진입점 정보 필요)
- T3는 T1과 병렬 가능 (`one_line_summary`는 T2 완료 후 받으면 검색 좁히기에 좋지만, 필수 아님 — 너무 늦으면 T3가 자기 가설로 진행)

### 2-2. 진행 모니터링

오케스트레이터(메인 Claude)는:
- 팀원들이 `SendMessage`로 서로 보강 요청을 주고받게 두고, 자체 조율을 신뢰
- 30분 이상 진행 없는 작업이 있으면 직접 개입 (해당 에이전트에게 상태 확인)

### 2-3. 자료 수집 완료 확인

세 파일이 모두 존재하고 비어 있지 않은지 확인. 누락 시 해당 에이전트 1회 재시도, 그래도 실패면 누락 사실을 글에 반영하기로 하고 진행.

## Phase 3: 글 작성 및 검증

**실행 모드:** 에이전트 팀 (작가-리뷰어 생성-검증)

Phase 2 팀을 정리(`TeamDelete`)하고 새 팀 구성 — 작가·리뷰어만. 자료 3개는 파일로 보존되어 있으므로 팀 전환에 손실 없음.

### 3-1. 작가 호출

`content-writer`에게 다음 입력과 함께 작업 요청:
- `repo_name`, `repo_url`
- `investigator_path`, `analyst_path`, `researcher_path`
- `output_path`: `_workspace/04_writer_draft.md`
- `primary_language`: `_workspace/01_investigator_facts.md`에서 추출한 주 언어 (소문자)

작가는 글 작성 중 자료 부족 시 직접 분석가/리서처에게 `SendMessage`로 보강 요청 가능. (단, Phase 3 팀에는 분석가가 없으므로 — 이 경우 작가가 오케스트레이터에 보고하고, 오케스트레이터가 분석가를 일시 호출해 보강 자료 추가 후 작가 재개. 혹은 작가가 자료의 한계 내에서 작성하기로 결정 가능.)

### 3-2. 리뷰어 검증 (iteration 1)

`quality-reviewer`에게:
- `draft_path`: `_workspace/04_writer_draft.md`
- 원자료 3개 경로
- `output_path`: `_workspace/05_reviewer_verdict.md`
- `iteration`: 1

판정 결과:
- **통과** → Phase 4로
- **재작업** → 3-3으로
- **소프트 패스** (iteration 1에서는 발생하지 않음 — 객관 기준 미달 시 반드시 재작업)

### 3-3. 작가 재작업 (iteration 2)

리뷰어의 재작업 요청서를 받아 `content-writer`가 수정. 같은 파일에 덮어쓰기.

### 3-4. 리뷰어 재검증 (iteration 2)

`quality-reviewer` 재호출, `iteration: 2`. 판정:
- 통과 → Phase 4
- 소프트 패스 → Phase 4 (한계를 사용자에게 보고할 수 있도록 verdict 보존)
- 재작업 요청? → 발생 시 무시하고 소프트 패스로 처리 (무한 루프 방지)

## Phase 4: 산출물 배치 및 게시

### 4-1. 카테고리 폴더 결정

`_workspace/04_writer_draft.md`의 frontmatter에서 `language` 추출.

언어 정규화 규칙:
- `TypeScript` / `JavaScript` → `typescript` / `javascript` (소문자)
- `C++` → `cpp`
- `C#` → `csharp`
- `Objective-C` → `objective-c`
- 알 수 없으면 → `other`

### 4-2. 파일명 결정

`{repo}` 소문자 + 특수문자 → `-` (예: `owner.dot/Repo_Name` → `repo-name.md`)
중복 회피: 이미 같은 이름이 있으면 `-2`, `-3` 식으로 suffix.

### 4-3. 파일 이동/저장

```bash
mkdir -p "categories/{language}"
cp _workspace/04_writer_draft.md "categories/{language}/{repo-name}.md"
```

### 4-4. 커밋·푸쉬

`references/git-publish-protocol.md`의 절차를 그대로 따른다. 사용자는 사전 승인했으므로 매번 확인하지 않는다. 단:
- 의도하지 않은 변경(추적 외 파일 등)이 있으면 보고 후 중단
- 푸쉬 충돌 발생 시 자동 처리 금지, 보고 후 사용자 결정 대기

## Phase 5: 사용자에게 보고

마무리 메시지 형식:

```
✅ {owner}/{repo} 리뷰 완료
- 📝 글: categories/{language}/{repo-name}.md ({N}자)
- 🔍 검증: {통과 / 소프트 패스 — 한계: ...}
- 📦 커밋: {sha} → origin/{branch}
```

소프트 패스인 경우 리뷰어의 한계 항목을 같이 표시.

## Phase 6: 진화 트리거 (선택)

사용자가 글에 대한 피드백을 주면 (예: "톤이 좀 딱딱하다", "비교가 약하다"), 같은 유형 피드백이 2회 누적되면 `blog-tone-guide.md` 또는 해당 에이전트 정의를 갱신할지 사용자에게 제안한다.

## 데이터 흐름 다이어그램

```
사용자 입력 (URL + 옵션)
        ↓
[Phase 0] URL 파싱 + 컨텍스트 확인
        ↓
[Phase 1] git clone --depth 50 → _workspace/repo/
        ↓
[Phase 2 — 팀 A: 자료 수집]
   ┌────────────────────────────┐
   │ T1: investigator  → 01.md  │  (병렬)
   │ T2: analyst       → 02.md  │  (T1 후)
   │ T3: researcher    → 03.md  │  (병렬)
   └────────────────────────────┘
        ↓
[Phase 3 — 팀 B: 작가-리뷰어]
   writer → 04.md
        ↓
   reviewer → 05.md (iter 1)
        ↓ (재작업 시 1회만 루프)
   writer → 04.md (수정)
        ↓
   reviewer → 05.md (iter 2, 강제 통과)
        ↓
[Phase 4] 04.md → categories/{lang}/{repo}.md → commit → push
        ↓
[Phase 5] 사용자에게 보고
```

## 에러 핸들링

| 에러 유형 | 처리 |
|----------|------|
| URL 파싱 실패 | 사용자에 재질문, 중단 |
| 클론 실패 (private/typo) | 사용자에 보고, 중단 |
| 자료 에이전트 1개 실패 | 1회 재시도 → 그래도 실패면 누락 표시하고 진행 (작가가 가용 자료로 작성) |
| 작가 분량 미달 (iter 1) | 리뷰어가 재작업 요청, iter 2 진행 |
| 작가 분량 미달 (iter 2) | 소프트 패스, 사용자 보고에 한계 명시 |
| 푸쉬 충돌 | 자동 처리 금지, 사용자 결정 대기 (로컬 커밋은 보존) |
| pre-commit 훅 실패 | 원인 보고, 우회 금지 |

## 테스트 시나리오

### 정상 흐름

입력: `https://github.com/anthropics/anthropic-sdk-python 리뷰해줘`

기대 동작:
1. URL 파싱 → `anthropics/anthropic-sdk-python`
2. `_workspace/` 비어있음 → 초기 실행
3. 클론 (depth 50)
4. 팀 A로 3개 자료 수집 (병렬)
5. 팀 B로 글 작성 (한국어 친근 톤, 3500자 안팎)
6. 리뷰어 iter 1 통과 가정
7. `categories/python/anthropic-sdk-python.md` 저장
8. 커밋 메시지 `docs(digest): anthropics/anthropic-sdk-python 리뷰 추가`
9. push 성공
10. 사용자에 한 줄 요약 보고

### 에러 흐름: 푸쉬 충돌

입력: 동일 (단, 원격 main이 로컬보다 앞서 있는 상태)

기대 동작:
1. Phase 4까지 정상 진행
2. `git push` → non-fast-forward 에러
3. `git fetch`로 원격 상태 확인
4. 사용자에 보고: "푸쉬 충돌 발생, 원격에 N개 커밋 앞섬. pull --rebase로 진행해도 될까요?"
5. 자동 진행 금지 — 사용자 응답 대기

### 후속 작업: 톤 수정

입력: `방금 그 글 톤 좀 더 가볍게 다시`

기대 동작:
1. `_workspace/` 존재 + `_workspace/04_writer_draft.md` 존재 → 부분 재실행
2. `content-writer`만 재호출 (자료 1~3은 재사용)
3. 작가가 톤 가이드 재참조 후 04.md 재작성
4. 리뷰어 재검증
5. categories/.../...md 덮어쓰기
6. 새 커밋(`docs(digest): {repo} 리뷰 톤 수정`)으로 푸쉬

## 추가 참고

- 한국어 블로그 톤 규칙: `references/blog-tone-guide.md` (작가 에이전트가 매 작업 시 참조)
- Git 커밋·푸쉬 절차: `references/git-publish-protocol.md` (Phase 4에서 참조)
