---
title: "별 1만 1천 개 짜리 'AI로 SNS 12개에 동시 발행하고 돈 번다'는 OSS, 코드 까봤더니 절반은 비공개였다 — yikart/AiToEarn"
repo: "yikart/AiToEarn"
url: "https://github.com/yikart/AiToEarn"
language: "typescript"
date: "2026-05-12"
summary: "GitHub 트렌딩에 올라온 yikart/AiToEarn은 'AI가 12개 SNS에 한 번에 글 올리고 광고비도 받아준다'고 자랑하는 풀스택 OSS다. 그런데 코드를 직접 까보니 인프라는 잘 만들어졌지만 정작 'Earn'의 핵심은 OSS에 없고, 셀프호스팅 기본값은 위험했다. 한국에 처음 소개해본다."
tags:
  - github-trending
  - ai-agent
  - mcp
  - claude-agent-sdk
  - nestjs
  - electron
  - social-media-automation
  - opensource-review
  - 중국-oss
  - 크리에이터-이코노미
---

# 별 1만 1천 개짜리 'AI로 SNS 12개에 동시 발행하고 돈 번다'는 OSS, 코드 까봤더니 절반은 비공개였다 — yikart/AiToEarn

## 도입

며칠 전 GitHub 트렌딩을 훑다가 `yikart/AiToEarn`이라는 이름에 손이 멈췄다. 별 1만 1천 개, 포크 2천 개, 작년 2월에 처음 공개된 신생인데 오늘(2026-05-12)에도 커밋이 찍혀 있는 활화산이다. 슬로건이 더 노골적이다 — **"Monetize · Publish · Engage · Create — all in one platform"**. 한 줄로 풀면 "AI가 콘텐츠를 만들어 주고, 12개가 넘는 SNS에 한 방에 뿌려주고, 댓글도 자동으로 달고, 광고주 의뢰까지 받아서 돈도 벌어준다"는 얘기다. 어느 시점부터 OSS 카피라이팅이 SaaS 백서보다 더 자신감 있게 변했는데, 이 레포는 그 끝판왕에 가깝다.

처음엔 "또 Postiz·Buffer류 SNS 스케줄러 짝퉁이겠지" 하고 닫으려 했다. 그런데 v2.1.0 릴리스 노트에 **MCP(Model Context Protocol) 지원**과 **OpenClaw 플러그인**이 박혀 있는 걸 보고 호기심이 발동했다. Anthropic이 작년에 밀기 시작한 MCP를 진짜로 production에 박은 OSS가 얼마나 되겠나 싶어서. 그래서 그냥 클론해서 코드를 까봤다. 결론부터 말하면, **이건 단순 스케줄러가 아니다**. 그리고 동시에, **OSS에 공개된 건 절반이고 나머지 절반은 의도적으로 숨겨져 있다**.

## 정체

먼저 마케팅 카피와 실제 사이의 거리를 좁혀보자. 카테고리부터 헷갈리니까.

`yikart/AiToEarn`은 표면적으로는 **"AI 콘텐츠 마케팅 에이전트 풀스택 OSS"** 다. Nx 기반 NestJS 백엔드 2개(메인 서버 + AI 서비스)에 Next.js 14 웹 클라이언트, Electron 데스크톱까지 모노레포에 다 들어 있다. README 3개 언어(중국어/영어/일본어)가 강제 동기화 룰로 묶여 있고, `docker-compose up` 한 방이면 9개 컨테이너가 떠서 셀프호스팅이 된다고 광고한다. 여기까지는 "잘 만든 OSS" 컷.

문제는 **본질이 SNS 스케줄러가 아니라 광고주↔크리에이터 마켓플레이스**라는 점이다. 공식 사이트 `aitoearn.ai`를 페치해 보면 가격표가 이렇게 적혀 있다(2026-05-12 시점).

> "AiToEarn is a platform connecting creators with paid promotion tasks across social media platforms including TikTok, Instagram, Twitter, YouTube, Threads, and Xiaohongshu (XHS)."
> "$1.00 Per 1K Engagements" (CPE) / "$1.00 Per 1K Views" (CPM) / "$1.00 Earn per post" (Fixed)
> "Capped at $20,000 per post"

쉽게 말해, **광고주가 작업(Task)을 등록하면 크리에이터가 OSS 클라이언트로 그 작업을 수주하고, AI가 콘텐츠를 만들어 자기 SNS 계정으로 자동 발행하고, 좋아요·댓글 1건당 0.8달러식 단가로 정산받는** 구조다. Postiz·Mixpost 같은 "내 콘텐츠를 내 일정대로 발행" 류 스케줄러와는 본질 카테고리가 다르다. 더 정확한 비유는 **"TaskRabbit for influencers + Buffer + Jasper AI"** 다.

만든 사람은? 깃허브 조직 `yikart` 명의이고, 별칭은 "**爱团团(Aituantuan)**". README의 연락처는 텔레그램 `@harryyyy2025`와 WeChat QR이다. 1차 지원 플랫폼이 抖音·小红书·快手·视频号·B站 4종으로 시작해서 글로벌로 뻗어 나간 이력, 도메인 이중 운영(`aitoearn.cn` 중국 / `aitoearn.ai` 글로벌), 코어 컨트리뷰터 4명(gaozhenqiang/niuwenzheng/gao1234-prog/whh2333)이 전체 contribution의 99%를 차지하는 정황까지 모으면 **중국 발 OSS로 사실상 확정**이다. 다만 운영 법인이나 투자자 정보는 공개돼 있지 않다 — "OPC(One Person Company)"라는 자기 마케팅 용어를 강조하는 게 자기 자신을 가리키는 것인지 비즈니스 페르소나인지도 모호하다.

한국 커뮤니티(GeekNews, Disquiet, velog) 검색에는 흔적이 없다. 중국어권에서는 silenceper(2026-03-10), 小羿(xiaoyi.vc), CSDN, 텐센트 클라우드 개발자 커뮤니티에 이미 여러 리뷰가 올라와 있는 반면 한국어 자료는 거의 공백이라, **한국에 정식으로 처음 소개하는 글**이라는 포지셔닝이 가능하다. 이 글이 그 첫 글이 되는 셈이다.

## 아키텍처·구현 디테일

코드를 까기 전에는 "또 LangChain + 발행 API 묶음이겠지" 했다. 까보니 생각보다 야무지다. 핵심 그림을 먼저 던지면 이렇다.

```
project/aitoearn-backend/   ← Nx + NestJS 11 (모노레포 안의 모노레포)
   ├── apps/
   │   ├── aitoearn-server/  (포트 3002, REST + MCP + Webhook + Cron)
   │   └── aitoearn-ai/      (포트 3010, Claude Agent SDK 호스팅)
   └── libs/  ← 17개 공용 라이브러리 (nest-mcp 포함)
project/aitoearn-web/       ← Next.js 14 App Router + AntD/shadcn 혼합
project/aitoearn-electron/  ← Electron 33 + better-sqlite3 (사실상 레거시 0.8.0)
```

내가 가장 놀란 부분은 **두 가지**다. 하나는 AI 서비스가 Claude Agent SDK를 그대로 production에 박았다는 점, 또 하나는 자체 MCP 어댑터를 NestJS용으로 만들어 두고 거기에 `@Tool` 데코레이터로 SNS 발행 도구 10개를 깔끔하게 노출한다는 점.

먼저 AI 쪽. `apps/aitoearn-ai/src/core/agent/agent.service.ts:1-10`에서 Anthropic 공식 SDK를 직접 import한다.

```ts
// project/aitoearn-backend/apps/aitoearn-ai/src/core/agent/agent.service.ts:1-10
import {
  McpServerConfig,
  SpawnedProcess,
  SpawnOptions,
} from '@anthropic-ai/claude-agent-sdk'
```

그런데 진짜 흥미로운 건 그 옆이다. 같은 디렉토리에 `claude-code-router/`라는 모듈이 있고, NestJS `OnModuleInit`에서 OSS 패키지 `@musistudio/claude-code-router/dist/cli.js`를 **자식 프로세스로 띄워서**(`spawn('node', [cliPath, 'start'])`, 포트 3456) 자체 LLM 게이트웨이로 모델 호출을 우회시킨다. Provider config에 `claude-opus-4-6`, `claude-haiku-4-5-20251001` 같은 모델 ID 11개가 미리 박혀 있고, default → opus, background → haiku, think → opus로 라우팅까지 정해져 있다(`claude-code-router/claude-code-router.service.ts:43-167`). 즉 Anthropic SDK는 표면이고, 내부에선 OSS Claude Code Router가 모델 분기를 책임지는 합성 구조다. **2026년 5월 기준으로 "Claude로 SNS 운영하기"의 production 사례 중 손에 꼽을 만한 디테일**이라고 본다.

그리고 발행. 외부 LLM이 보는 발행 도구는 `apps/aitoearn-server/src/core/channel/publish.mcp.controller.ts:60-366`에 평면적으로 10개가 깔려 있다 — `publishPostToBilibili`, `publishPostToYoutube`, `publishPostToTiktok`, `publishPostToTwitter`, ... Bilibili는 `tid` 카테고리 ID를 먼저 `getBilibiliContentCategories`로 받아오라고 시키고, Kwai는 cover 필수라 없으면 `media_generation`으로 자동 생성하라고 시킨다. **워크플로 힌트가 도구 description에 자연어로 박혀 있어** Claude가 그대로 따라갈 수 있게 설계했다.

발행 큐도 만만치 않다. `publishing.service.ts:90-156`을 보면 **2-stage 큐 구조**다 — 일단 검증 → flowId 중복 체크 → 계정 조회 → 레코드 생성(queueId UUID) → 즉시/예약 분기. 미디어 업로드는 별도 큐로 빠지고(`base.service.ts:88-227`, `attempts:5, fixed backoff 15s`), 각 미디어를 폴링해서 `processing → completed/failed`로 수렴할 때까지 기다린 다음 `finalizePublish`로 실제 발행 commit이 떨어진다. TikTok처럼 5MB 청크 업로드 + 비동기 트랜스코딩이 필요한 플랫폼도 깨지지 않게 만든 설계. 거기에 BullMQ + Redlock 분산 락으로 멀티 워커 안전성까지 묶어둔다.

또 하나 코드 차원에서 인상적인 건 사내 코딩 표준이다. `project/aitoearn-backend/.claude/rules/` 디렉토리에 7개 룰 파일이 있는데, 이게 단순 가이드가 아니라 **팀이 Claude/Codex 같은 코딩 에이전트에게 강제하는 표준**이다. Repository 메서드 prefix는 `get/list/create/update/delete/count`만 허용(`findById → getById`), Service에서 `@InjectModel` 금지(Repository 경유 강제), `process.env` 직접 접근 금지(zod 기반 config 모듈 경유), `console.log` 금지, 응답은 `{ data, code, message }` 강제, `throw new Error()` 금지(오직 `new AppException(ResponseCode.XxxNotFound)`만). 별 1.1만 개짜리 OSS 중에 이 정도 룰 정합성을 갖춘 곳이 흔치 않다.

## 박수 포인트

코드를 읽으면서 진짜 박수치고 싶었던 디테일 네 개를 꼽아본다.

**1. NestJS용 자체 MCP 어댑터(`libs/nest-mcp`)가 거의 독립 OSS 수준이다.** `McpModule.forRoot({ name, version, apiPrefix, transport, ... })` 한 줄이면 SSE + Streamable HTTP + STDIO 세 트랜스포트가 자동으로 컨트롤러로 떠오른다. 그 다음 `McpRegistryService`가 `onApplicationBootstrap`에서 `DiscoveryService`로 임포트 트리를 훑어 `@Tool`/`@Resource`/`@Prompt` 메타데이터가 붙은 메서드를 자동으로 등록한다(`libs/nest-mcp/src/services/mcp-registry.service.ts:52-163`). 즉 백엔드 컨트롤러에 `@Tool` 데코레이터 한 줄만 붙이면 그 메서드가 즉시 MCP 도구로 노출된다. 공개 MCP TypeScript SDK를 NestJS DI에 맞게 재포장한 셈인데, 이 부분만 독립 라이브러리로 떼어내도 충분히 가치가 있다.

**2. LLM이 자기 자신을 사전 검증하게 만든 프롬프트 엔지니어링.** `publishRestrictions` 도구(`publish.mcp.controller.ts:33-45`)가 11개 플랫폼별 발행 제약을 한 줄 자연어로 박아 둔다 — "Bilibili는 제목 80자, 설명 250자, topic 10개, hashtag 1개 이상, tid 필수". Claude가 이걸 받아서 발행 전에 자기 자신을 검증한다. 백엔드가 validation 에러를 토하는 게 아니라, 도구 호출 직전에 LLM이 "아 이건 안 되겠네" 하고 알아채는 구조. 영리하다.

**3. 13개 Claude Skill을 그대로 production에 박았다.** `apps/aitoearn-ai/src/core/agent/skills/` 안에 frontmatter+markdown 포맷 SKILL.md가 13개 들어 있다(`editing-videos` 544줄, `generating-videos` 509줄, `generating-drama-recaps` 116줄...). 단편드라마 해설 영상 자동 생성(`generating-drama-recaps`)의 톤을 `生动有趣/幽默搞笑/悬疑紧张/...` 7가지로 enum화해 둔 부분이 백미다. **시장이 코드에 박혀 있다.** 중국 단편 드라마 해설 영상 생태계라는 매우 구체적인 시장을 워크플로 수준으로 코드화한 건, 이걸 만든 팀이 시장을 진짜로 보고 있다는 신호다.

**4. 광고주 매칭의 Relay 프록시 패턴이 흥미롭다.** `relay-exception.filter.ts:1-167`은 셀프호스팅 사용자가 직접 OAuth 키를 못 받는 플랫폼(예: Bilibili 콘텐츠 크리에이터 API)을 본사 Relay 서버로 자동 프록시하는 글로벌 NestJS 필터다. `RelayAccountException`을 던지면 필터가 catch해서 요청을 그대로 본사 서버로 재전송하고, body 안 로컬 미디어 URL을 발견하면 자동으로 본사에 업로드하고 그 URL로 치환한다. **OSS의 한계(중국 폐쇄 플랫폼 OAuth 직접 못 받음)를 본사 서비스가 매개로 메우는 하이브리드 설계.** 박수 포인트인 동시에 다음 섹션의 첫 비판 포인트이기도 하다.

silenceper의 평에 이런 표현이 있다. **"콘텐츠 증가가 이미 '단일 창작'에서 '프로세스 경쟁'으로 진입했다"** ([silenceper.com/article/2026-03-10-aitoearn-intro](https://silenceper.com/article/2026-03-10-aitoearn-intro/)). 이 말이 정확하다 — AiToEarn의 진짜 강점은 어느 한 기능이 아니라, **콘텐츠 생성 → 발행 → 인터랙션 → 정산까지를 큐와 워커로 일관되게 묶어둔 파이프라인**이다. 박수칠 만하다.

## 비판·한계

여기까지가 박수 섹션이고, 이제 균형을 위해 비판도 비슷한 분량으로 풀어보자. 코드를 까보면서 "이건 한국 독자에게 솔직히 말해야 한다"고 메모해둔 게 다섯 개다.

**1. 셀프호스팅 기본값이 위험하다.** `docker-compose.yml`을 그대로 띄우면 세 군데에 `JWT_SECRET: change-this-jwt-secret`이 박혀 있다. 사용자가 안 바꾸면 누구나 추측 가능한 시크릿으로 JWT를 위조할 수 있다. 거기에 `scripts/init.mjs:71-74`는 첫 부팅 시 `admin@aitoearn.local` 계정을 만들고 **`expiresIn:'100y'`로 100년짜리 영구 admin JWT**를 token.txt에 저장해 web 컨테이너에 자동 주입한다. JWT_SECRET 디폴트면 영구 admin이 사실상 공개라는 얘기. 한 술 더 떠 `project/aitoearn-web/package.json:33-35`에는 locize.app i18n SaaS의 api-key 2개가 평문으로 박혀 있고(`migrateToLocize`, `syncLocales` 스크립트 인자), `app.enableCors()`가 무옵션이라 모든 오리진을 허용한다. PR #501이 이 CORS 문제를 지적했지만 미머지로 닫혔다. **3분 셋업이라는 매끄러운 UX의 대가가 너무 크다.**

**2. "Earn"의 핵심은 OSS에 없다.** `libs/aitoearn-server-client/src/interfaces/task.interface.ts:1-67`을 보면 `Task`, `UserTask` 인터페이스와 상태기계(`DOING → WAITING → PENDING → APPROVED → SETTLED`)는 100% 노출되어 있는데, **이 모델의 Repository·Service·정산 로직은 이 모노레포 어디에도 없다.** `apps/aitoearn-server/src/`에서 `UserTask`를 검색하면 `publish-record.service.ts:238`의 `getPublishRecordToUserTask(userTaskId)` 한 줄 — Repository 호출만 있고 내부는 비공개 본사 서버에서 RPC로 제공받는다. **즉 OSS는 "AI 콘텐츠 자동화 + 발행 인프라"까지만 공개하고, 마켓플레이스의 광고주 매칭과 정산 결정자는 본사가 쥐고 있다.** 셀프호스팅한다고 정말로 독립이 되는 게 아니라, **본사 SaaS의 클라이언트 인프라를 OSS로 푼 dual model**이라고 보는 게 정확하다. 마케팅 카피의 "Best Open-Source AI Agent for Content Marketing"과 비교하면 거리가 있다.

**3. 셀프호스팅에 숨은 본사 의존성.** 위에서 박수쳤던 Relay 프록시는 뒤집어 보면 **OAuth 키 없는 사용자는 본사에 의존해야 한다**는 얘기다. 더 심각한 건 이슈 #507 — 도커로 띄운 셀프호스팅 사용자가 자체 LLM 키를 넣었는데도 작가 어시스턴트가 `aitoearn.ai/api/ai/chat`을 호출하더라는 버그 리포트. 코드 근거가 정확히 `claude-code-router.service.ts:81-125`에 있다. `config.agent.baseUrl`을 사용자가 비우면 본사 게이트웨이로 흐른다. **본사가 발 빼면 셀프호스팅이 멈출 수 있다는 뜻이다.**

**4. 중국 SNS 발행은 OAuth가 아니라 회색지대 자동화다.** 백엔드 Douyin은 `share-schema` 딥링크를 만들어 단축링크로 폰에 보내는 방식이다(`douyin.service.ts:27-89`). 사용자가 폰에서 "발행" 버튼을 직접 누르도록 만든 — 즉 공식 publish API가 아니라 **앱 인텐트**다. Electron의 Xiaohongshu 어댑터(`project/aitoearn-electron/electron/plat/xiaohongshu/index.ts:1-80`)는 더 노골적이다. BrowserWindow로 `creator.xiaohongshu.com`을 띄워 사용자를 로그인 시킨 뒤 쿠키를 추출하고, 비공식 internal API(`edith.xiaohongshu.com/api/sns/web/v2/...`)를 호출한다. 거기에 line 47에 **하드코딩된 `esec_token = 'ABrmhLmsdmsu9bCQ80qvGPN2CYSjEqwi5G1l2dirNUjaw%3D'`**. 본질적으로 리버싱된 internal API다. 중국 SNS 시장에 공식 마케팅 API가 부족해서 이런 회색지대 자동화 없이는 "한 번에 12개 발행"이 성립하지 않는 건 이해한다. 하지만 **플랫폼 정책 변경에 매우 취약**하고, 이슈 #510(`resolveStateDir` 미정의) 같은 호환성 이슈가 계속 터질 수밖에 없다.

**5. 거버넌스 성숙도가 별 수만큼 따라오지 않았다.** 별 1.1만 개, 포크 2천 개라는 인기에 비해 PR 머지율은 낮다. 보안 패치인 PR #501(CORS 무옵션 지적)도 미머지로 닫혔고, 외부 기여자 PR이 꾸준히 들어오지만 머지가 더디다. CONTRIBUTING.md가 백엔드 셋업 경로를 `project/backend/`로 잘못 안내한다(실제는 `project/aitoearn-backend/`). 모노레포 안 Electron 폴더는 버전 0.8.0이고 README는 별도 저장소 `yikart/AttAiToEarn`을 안내하는데, 이 안 코드가 레거시인지 활성인지 자체도 모호하다. silenceper도 같은 결의 평을 남겼다 — **"도구는 전략을 대체할 수 없으며, 운영 방법론에 크게 의존한다"** ([silenceper.com](https://silenceper.com/article/2026-03-10-aitoearn-intro/)). 트렌딩 통계 사이트 Trendshift는 이 레포의 성공 비결을 **"its success stems from having 'their goal in the title' rather than chasing broader AI prestige"** — 거창한 AGI 대신 "돈 번다"라는 구체적 목표를 제목에 박은 작명 전략 — 으로 풀었는데([trendshift.io/repositories/20785](https://trendshift.io/repositories/20785)), 그 작명만큼 거버넌스가 따라오고 있는지는 별개의 문제다.

## 결론

내가 이 글에서 가장 중요하게 짚고 싶은 한 줄은, **AiToEarn은 "단순 SNS 스케줄러"가 아니라 "광고주 마켓 + AI 콘텐츠 클라이언트의 dual model"이고, 그 dual 중 OSS는 절반이다**라는 점이다. 박수칠 만한 인프라이긴 한데, 마케팅 카피의 "Best Open-Source"와 실제 공개 범위 사이에는 분명한 거리가 있다.

**잘 맞는 사람**은 이렇게 정리할 수 있다. (1) **MCP·Claude Agent SDK를 production에 박는 구조가 궁금한 NestJS 개발자** — `libs/nest-mcp`의 데코레이터 + DiscoveryService 패턴, 그리고 `claude-code-router` 자식 프로세스 라우팅 구조는 학습 자료로 충분하다. (2) **중국·동남아 시장 타겟의 1인 콘텐츠 사업자** — Douyin·Xhs·Kuaishou·WeChat·Bilibili 통합이 1급으로 들어간 OSS는 흔치 않다. (3) **AI 에이전트가 SNS를 굴리는 워크플로를 학습 목적으로 까보고 싶은 사람** — 13개 SKILL.md가 그대로 시장 워크플로를 코드화해둔 좋은 사례다.

**피해야 할 사람**은 더 분명하다. (1) **셀프호스팅으로 본사 의존 없이 운영하고 싶은 사람** — Earn 정산은 OSS에 없고, LLM 게이트웨이도 본사 디폴트로 흐른다. (2) **한국 플랫폼(네이버 블로그·티스토리·당근·카카오)이 주력인 크리에이터** — 통합이 전혀 없다. (3) **보안 디폴트 그대로 띄울 셀프호스터** — `JWT_SECRET=change-this-jwt-secret`, 100년 영구 admin JWT, 평문 SaaS api-key, 무옵션 CORS의 콤보는 프로덕션엔 그대로 못 올린다. 가기 전에 `.env`와 `docker-compose.yml`을 통째로 다시 써야 한다. (4) **공식 API ToS를 엄격히 지켜야 하는 기업** — 중국 SNS 어댑터는 회색지대다.

다 까보고 나서 든 솔직한 감상은, **"이건 OSS 라기보단 'SaaS의 깊은 free tier'에 가깝다"** 는 것. 동시에 별 1만 1천 개가 그냥 붙은 건 아니다. 인프라는 진짜로 잘 만들었고, MCP 어댑터와 큐 설계는 다른 프로젝트에서 떼어가도 손색없다. 다음에 시간이 나면 `libs/nest-mcp`만 따로 떼어내서 작은 데모를 만들어보고 싶다. 그리고 6개월 뒤 PR #501 같은 보안 패치들이 머지되었는지, 영구 admin JWT가 사라졌는지를 한 번 더 점검해 볼 생각이다. 그때까지는 — 코드 학습용으로는 추천, 실서비스 셀프호스팅으로는 권하지 않음.
