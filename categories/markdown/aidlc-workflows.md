---
title: "AWS Labs가 던진 마크다운 한 뭉치 — aidlc-workflows로 AI 코딩 SDLC를 우길 수 있을까"
repo: "awslabs/aidlc-workflows"
url: "https://github.com/awslabs/aidlc-workflows"
language: "markdown"
date: "2026-05-22"
summary: "AWS Labs가 MIT-0로 풀어버린 AI 코딩 에이전트용 워크플로 룰셋. Kiro·Q·Cursor·Claude Code·Copilot·Codex 어디서나 같은 마크다운 파일을 복사만 하면 동작한다는 야심찬 주장과, '진짜 병목은 따로 있다'는 HN 비판이 맞붙는 지점을 직접 뜯어봤다."
tags: [aws-labs, ai-dlc, spec-driven-development, ai-coding, sdlc, methodology, mit-0, bedrock, agentic-workflow]
---

# AWS Labs가 던진 마크다운 한 뭉치 — aidlc-workflows로 AI 코딩 SDLC를 우길 수 있을까

## 도입

며칠 전 트렌딩 탭에서 `awslabs/aidlc-workflows`라는 이름을 봤다. 처음엔 그러려니 했는데, 클릭하고 나서 좀 멈췄다. 코드 라이브러리가 아니라 **마크다운 파일 한 뭉치**가 본체였기 때문이다. 그것도 AWS Labs가 MIT-0(MIT No Attribution, 출처 표시 의무도 없는 사실상 퍼블릭 도메인)로 던져놓은 마크다운.

레포 설명은 친절하게도 두루뭉술하다. "An intelligent software development workflow that adapts to your needs, maintains quality standards, and keeps you in control of the process." 이걸 읽고 무슨 도구인지 알 수 있는 사람은 없을 것 같다. 그런데 README를 41KB·922줄까지 펼치고 나면 의도가 보인다. AWS가 자기네 IDE인 Kiro에 묶지 않고 **Kiro, Amazon Q Developer, Cursor, Cline, Claude Code, GitHub Copilot, OpenAI Codex** 어디서나 같은 룰셋이 굴러가도록 표준화하겠다고 선언한 셈이다. 흥미로운 건 그 다음이다. "어디서나 동작한다"라면서, 정작 같이 공개한 보조 도구 세 개는 전부 AWS Bedrock에 의존한다. 이 모순이 이 레포의 가장 매력적인 지점이었다.

## 정체

한 줄로 줄이면, 이건 **"AI 코딩 에이전트가 읽어서 실행하는 마크다운 규칙집"** 이다. 정식 이름은 AI-DLC (AI-Driven Development Life Cycle). 산출물은 두 갈래다. 핵심은 `aidlc-rules/` 밑의 마크다운 룰셋 — 진입 룰 `core-workflow.md` 1개 + 단계별 상세 룰 24개를 합쳐 약 25개 파일, 도합 약 4,600줄. 곁에 붙은 건 `scripts/` 밑의 Python 도구 3종(design review, evaluator, traceability). 룰만 가져다 쓰면 어떤 에이전트에서도 같은 3단계(Inception → Construction → Operations) 워크플로가 굴러가고, Bedrock 계정이 있는 사람은 보조 도구로 그 산출물을 검사·평가·추적할 수 있다.

만든 쪽은 AWS Labs(awslabs org). 실제 설계자는 Raja SP(AWS Principal Solutions Architect)와 Will Matos, Raj Jain, Siddhesh Jog. re:Invent 2025 세션 **DVT214**에서 Anupam Mishra(Director, Solutions Architecture)와 Raja가 공식 발표했다. 타임라인을 보면 의도가 더 또렷하다. 2025년 7월 31일 AWS DevOps Blog에 개념 글이 먼저 떴고, 11월 29일 정식 오픈소스 발표, 12월 5일 re:Invent 발표가 이어졌다. 그러니까 이건 깜짝 공개가 아니라 **재밌는 타이밍 설계**다. re:Invent라는 가장 큰 무대 직전에 깃허브에 올려두고, 발표에서 "이게 우리가 풀었다는 그 룰셋이다"를 외치는 흐름.

AWS Labs라는 출처가 가벼운 무게는 아니다. 같은 awslabs org의 `mcp`(MCP 서버 모음)가 이미 큰 인기를 끌었고, 사촌격인 `strands-agents`는 Kiro·Amazon Q·AWS Glue가 실제 프로덕션에서 쓴다고 [AWS Open Source Blog](https://aws.amazon.com/blogs/opensource/introducing-strands-agents-an-open-source-ai-agents-sdk/)가 밝혔다. 이 맥락 위에 MIT-0로 푼다는 건 "그냥 가져다 써라, 기업 내부에서 복제·포크·rename 다 좋다"는 신호로 읽힌다.

## 아키텍처·구현 디테일

이 레포의 진짜 진입점은 단 하나다. `aidlc-rules/aws-aidlc-rules/core-workflow.md`. 539줄짜리 이 파일이 모든 시작이고, 나머지 24개 룰 파일은 이 파일이 **필요한 시점에만 끌어다 쓰는 상세 문서들**이다.

전체 워크플로는 3 phase × 14 stage 구조다. INCEPTION에 7단계(Workspace Detection / Reverse Engineering / Requirements Analysis / User Stories / Workflow Planning / Application Design / Units Generation), CONSTRUCTION에 6단계(Functional Design / NFR Requirements / NFR Design / Infrastructure Design / Code Generation / Build and Test), OPERATIONS에 1단계. 핵심 디자인 결정은 "고정 시퀀스 없음(No fixed sequences)"이다. AI가 요청·워크스페이스·복잡도를 평가해 어떤 conditional 단계를 켤지 정한다. 단순 버그 픽스면 4~5단계로 끝나고, 신규 마이크로서비스면 14단계를 다 거친다.

구조 자체보다 더 인상적이었던 건 **마크다운만으로 lazy loading을 구현해놨다**는 점이다. `core-workflow.md`는 "무엇을 언제 한다"만 쓴다. "어떻게 하는지"는 매번 `aws-aidlc-rule-details/{phase}/{stage}.md`를 그 시점에 로드. 14단계 룰을 미리 다 읽히면 에이전트 컨텍스트 토큰이 거덜나니까, "필요할 때만 펼친다"를 마크다운 파일 분할로 해결한 셈이다.

그 다음으로 영리한 건 **경로 캐스케이드** 패턴이다.

```markdown
# core-workflow.md:13-20
.aidlc/aidlc-rules/aws-aidlc-rule-details/   ← AI-assisted setup
.aidlc-rule-details/                          ← Cursor, Cline, Claude Code, Copilot, Codex
.kiro/aws-aidlc-rule-details/                 ← Kiro IDE/CLI
.amazonq/aws-aidlc-rule-details/              ← Amazon Q Developer
```

룰 본문은 어떤 에이전트에서나 동일하다. 위치만 다르다. 룰 안에 "이 경로들을 순서대로 찾아라"가 박혀 있고, 사용자는 자기 에이전트에 맞는 폴더에 같은 zip을 풀기만 하면 된다. 마크다운으로 OS의 PATH 환경변수 같은 걸 구현한 모양새. 추상화의 결이 곱다.

그리고 `scripts/` 밑의 Python 도구 3종도 같이 봐야 정체가 잡힌다. 셋 다 **strands-agents + boto3 Bedrock** 위에 올려둔 멀티 에이전트 시스템이다.

- `aidlc-designreview/` — Claude 모델을 Bedrock으로 불러 디자인 산출물을 비평. README 인용 기준 최소 2개 이상의 에이전트가 묶인 다중 에이전트 파이프라인이고, 확인된 흐름은 Critique Agent → Alternatives Agent다. 743개 테스트를 자랑한다. CLI와 Claude Code Hook(pre-tool-use에서 자동 인터셉트) 두 형태로 동작.
- `aidlc-evaluator/` — 본 레포 자체의 회귀 테스트 프레임워크. uv workspace로 11개 서브패키지(execution, qualitative, quantitative, contracttest, nonfunctional, reporting, shared, ide-harness, cli-harness, trend-reports, 본체)를 묶었다. 6-stage 파이프라인: Execution → Post-Run → Quantitative(lint·security·duplication) → Contract(API 검증) → Qualitative(Bedrock LLM judge) → Report. **Docker 샌드박스에서 실제로 생성된 앱을 띄워 contract test까지 돌린다.**
- `aidlc-traceability/` — 요구사항 → 유저 스토리 → unit → component → 소스 코드를 networkx 그래프로 추적. 다크 모드 HTML 리포트, gap analysis(고아 산출물 자동 검출)까지 들어있다.

세 도구 모두 마크다운 산출물을 1급 데이터로 다룬다는 게 일관된 정체성이다. 룰이 만든 게 마크다운이고, 평가도 마크다운을 읽는다. **마크다운이 데이터 포맷이자 실행 포맷이자 감사 포맷**인 셈.

## 박수 포인트

**1. "질문은 채팅 말고 파일에" 룰.** `common/question-format-guide.md`에 절대 규칙이 박혀 있다.

> CRITICAL: You must NEVER ask questions directly in the chat. ALL questions must be placed in dedicated question files.

질문은 `{phase-name}-questions.md`로 만들고, 각 질문은 A/B/C/D/E(맨 마지막은 "Other") 객관식, 답은 `[Answer]:` 태그 뒤에 적도록 강제한다. 채팅이라는 휘발성 채널에서 의사결정을 빼서 git diff 가능한 문서로 옮긴 거다. 단순한 규칙인데, AI 협업의 결정적 차이를 만든다. 트레이서빌리티 도구가 이 파일을 파싱해 요구사항·스토리·코드 매트릭스를 만들 수 있는 것도 이 규약 덕분이다.

**2. Extension의 opt-in 분리.** `extensions/security/baseline/` 같은 디렉토리에 두 파일이 들어있다. `security-baseline.opt-in.md`(작음, 항상 로드 — 사용자에게 "이거 켤래?"를 묻는 객관식 질문만)와 `security-baseline.md`(큼, 옵트인 시에만 로드). 컨텍스트 비용·사용자 결정권·옵트인 후의 강제력을 마크다운 명명 규약만으로 풀어낸 결정이 우아하다.

**3. Overconfidence Prevention 룰의 회고록.** `common/overconfidence-prevention.md` 100줄 전체가 메타 룰이다. "Only ask questions if absolutely necessary"였던 옛 규칙을 "When in doubt, ask the question"으로 뒤집고, 다섯 가지 원칙(Default to Asking / Comprehensive Coverage / Thorough Analysis / Mandatory Follow-up / No Proceeding with Ambiguity)을 박아뒀다. **자기네 룰의 회고를 룰 안에 넣었다**는 게 인상적이었다. "예전엔 이래서 망했고 그래서 지금은 이렇게 하라"는 식의 메모리가 룰셋에 박혀 있는 셈.

**4. 보안 게이트의 두께.** bandit, checkov, gitleaks(45KB baseline), grype, semgrep, CodeQL, markdownlint, pre-commit. 룰셋 레포에 이렇게 두텁게 까는 케이스는 드물다. 룰 자체가 빈약하면 두께만 두텁고 알맹이가 없는데, 룰의 강조어("MANDATORY", "CRITICAL", "ALWAYS", "NEVER")가 일관되게 쓰이고 거의 모든 stage가 "Wait for Explicit Approval — DO NOT PROCEED until user confirms"로 끝난다는 점에서 알맹이도 있다.

## 비판·한계

이 부분이 솔직히 글의 무게 중심이라고 느꼈다.

**1. "Wrong Bottleneck" — 가장 날카로운 비판.** wakamoleguy가 [블로그](https://wakamoleguy.com/p/ai-dlc-solves-wrong-bottleneck)에 올린 글이 2026년 2월 HN에 올라왔다. 요지: "엔지니어링·프로덕트·디자인이 모두 AI로 사이클 타임을 줄이면, 병목은 비동기·인간 의존 영역(사용자 이해, 검증, 이해관계자 정렬)으로 이동한다. 그건 외부 피드백 루프에 묶여 있어 빠르게 줄일 수 없다. AI-DLC는 진짜 병목을 안 건드린다." 이 비판은 방법론 자체의 근본을 친다. arXiv의 "The Novelty Bottleneck" 논문이 자주 같이 인용된다 — "사용자 의도 명세·결과 검증·오류 수정에 드는 인간 시간은 작업 크기에 비례해 stubbornly 남는다." AWS의 마케팅 카피("10–15배 생산성", "2개월 작업을 48시간에")가 이 지점을 정면으로 못 받아치는 한, 비판은 살아 있다.

**2. learning vs calibration의 혼동.** Peter Tilsen이 [Medium](https://medium.com/data-science-collective/the-ai-driven-development-lifecycle-ai-dlc-a-critical-yet-hopeful-view-edc966173f2f)에 쓴 비평의 핵심. "현재 AI-DLC의 개념은 learning을 calibration과 거의 혼용한다. 진짜 학습 메커니즘이 부재하고, 맥락 메모리 작동 원리가 불투명하며, 구현 중심 서술이 AI-DLC의 진짜 혁신을 가린다." 룰 안에서 "Mandatory Follow-up" 같은 메타 인지를 강제하는 것은 좋은데, 그게 진짜 학습이냐 단순 calibration이냐는 질문이 남는다.

**3. OPERATIONS 단계가 PLACEHOLDER 19줄.** 3-phase를 외치는데 마지막 phase가 사실상 비어 있다. 배포·모니터링·인시던트 룰이 없으니 "Life Cycle"이라는 작명이 무색해진다. Construction 단계 중 가장 짧은 룰(`code-generation.md`)도 100줄이 넘는데 operations만 19줄.

**4. "Agnostic" tenet과 Bedrock 의존의 충돌.** 룰셋은 멀티 에이전트 호환이지만 보조 도구(designreview·evaluator·traceability)는 전부 Bedrock 필수다. MIT-0로 풀어놨지만, 보조 도구 가치를 다 누리려면 AWS 계정·요금이 필요하다. "Agnostic"이라는 다섯 가지 Tenet 중 하나와 살짝 어긋난다. 어찌 보면 이게 가장 AWS다운 결정이긴 하다 — 표준은 풀고, 차별화는 자기 클라우드로.

**5. v0.1.x의 실전 한계.** 일본 Fusic의 [워크숍 후기](https://tech.fusic.co.jp/posts/awslabs-aidlc-workflows-inception-construction-practice/)가 가장 솔직한 검증이다. Claude Code·Codex·Cursor·Copilot 네 도구에서 React 19 + Hono + DynamoDB + CDK 스택의 TODO 앱으로 돌려봤다. 호평은 이거였다 — "AIは優秀な実装者ですが、要件・設計・品質の構造を人間側が整えないと力を引き出せません"(AI는 우수한 구현자지만 인간이 요건·설계·품질 구조를 짜놓지 않으면 힘을 못 낸다). 비판은 세 가지였다. ① 단일 `aidlc-docs/` 폴더 가정이라 멀티-피처 팀 작업 시 컨벤션 자체를 다시 정의해야 함. ② v0.1.6 시점이라 per-feature state·rollback이 미지원. ③ "고속 사이클 철학"과 "프로덕션 버그 리스크" 간 긴장이 해소되지 않음. 최신 5월 22일 커밋이 `fix(session-continuity): scope per-unit artifact loading to unit dirs (#276)`인 걸 보면 이 문제 일부를 지금 손보고 있는 중이긴 하다.

**6. 첫 5분의 마찰.** README가 41KB·922줄인데 절반 이상이 OS·에이전트별 설치 셸 명령이다. 5개 OS × 5개 에이전트 = 25가지 명령이 도배되어 있고, 트리거 문장 `"Using AI-DLC, ..."`까지 도달하는 경로가 멀다. "Users shouldn't need to install anything"이라는 Tenet과 실제 README의 풍경이 다르다. `feat: agent-driven setup (#109)` 커밋으로 자동화를 시작했지만 아직 "Experimental" 딱지가 붙어 있다.

박수 4개, 비판 6개. 글로 정리하니 비판이 더 많은데, 그게 이 레포의 솔직한 인상이다. **방향은 똑똑하고, 디테일도 정성이 들어갔지만, 마케팅 카피가 실체보다 한 발짝 앞서 있다.**

## 결론

누구한테 잘 맞을까. 내 감으론 세 부류다. 첫째, **사내에 AI 코딩 에이전트를 도입했는데 산출물이 채팅창에 떠다녀서 골치인 팀.** 질문을 파일로 받아 git에 commit되게 하는 규약 하나만으로 트레이서빌리티가 생긴다. 둘째, **AWS 생태계 안에 발을 담그고 있고 Bedrock 사용에 거부감이 없는 조직.** 룰셋만 가져가도 가치가 있지만 도구 3종까지 켜면 진짜 회로가 완성된다. 셋째, **자기 조직의 SDLC 룰을 처음부터 쓰기 두려워 출발점이 필요한 사람.** MIT-0니까 통째로 포크해 자기 이름으로 rename해도 되고, Extension 패턴만 가져다 자기 룰셋에 얹어도 된다.

피해야 할 사람도 분명하다. 첫째, **"AI한테 알아서 시키고 손 떼고 싶다"는 바이브 코더.** Human-in-the-loop가 룰 레벨에서 강제되어 있어서 단계마다 사용자 승인 게이트에 막힌다. 그게 미덕이지만, 원하는 사람한테만 미덕. 둘째, **이미 GitHub spec-kit이나 BMAD로 자기 워크플로를 굳힌 팀.** 패러다임이 비슷해서 갈아타는 비용 대비 차이가 작다. 셋째, **AWS 클라우드를 안 쓰는 조직 중 보조 도구까지 다 누리고 싶은 경우.** 룰만 가져가면 되지만, 그러면 절반쯤만 가져가는 셈이다. 넷째, **운영 단계 가이드가 절실한 팀.** OPERATIONS phase가 19줄짜리 placeholder다.

프로덕션에 쓸만한지 묻는다면, 현재 v0.1.8 단계에서는 **"실험에는 충분히 좋고, 프로덕션에는 너무 이르다"** 가 솔직한 답이다. 최신 커밋이 `session-continuity` 버그 픽스인 것에서 보듯이 핵심 기능이 아직 다듬어지는 중이고, `aidlc-docs/`가 단일 디렉토리 가정이라 멀티 팀 동시 작업이 어렵다. 그래도 v2 alpha를 시작한 `#284` 커밋이 5월에 들어간 걸 보면 곧 한 단계 점프가 있을 듯하다.

마지막으로 내 느낌. 이 레포의 진짜 정체는 **"AI 코딩 에이전트 시대의 SDLC 거버넌스 미들웨어"** 라고 부르고 싶다. 룰셋 자체가 아니라 룰셋 + 평가 + 트레이서빌리티가 한 묶음으로 묶여 있다는 점이 차별점이다. 그리고 그 묶음을 MIT-0로 푸는 결정에 AWS의 야심이 보인다 — 표준은 공짜로 풀고, 본격 운용은 Bedrock 위에서 하게 만든다. 이 전략이 GitHub spec-kit·BMAD·OpenSpec 같은 동시대 SDD 진영 사이에서 어떻게 자리잡을지, 0.1.x가 1.0이 되는 그 사이를 좀 더 지켜보고 싶어졌다. 다음에는 `aidlc-evaluator`를 진짜로 돌려서 골든 테스트가 어떻게 굴러가는지 손으로 만져볼 생각이다.
