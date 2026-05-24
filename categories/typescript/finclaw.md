---
title: "스타 0개 짜리 1인 레포를 진지하게 까보는 이유 — Nyam2C/FinClaw, 'OpenClaw 시대'를 금융에 이식한 학습 결과물"
repo: "Nyam2C/FinClaw"
url: "https://github.com/Nyam2C/FinClaw"
language: "typescript"
date: "2026-05-24"
summary: "Nyam2C/FinClaw는 스타 0·이슈 0의 1인 학습용 레포다. 트렌딩한 건 이게 아니라 그 모태인 OpenClaw(247K 스타)와 NanoClaw(Karpathy 호평)다. 그런데 코드를 까보면 RAG 인용 역추적, LLM 환각 차단 라우터, DNS 핀닝 SSRF 같은 디테일이 1인 프로젝트 평균을 한참 넘는다. '한 사람이 전부 감사할 수 있는 코드'라는 NanoClaw식 철학을 금융 도메인에 정직하게 이식한 사례로 읽었다."
tags:
  - github-trending
  - ai-agent
  - openclaw
  - nanoclaw
  - typescript
  - monorepo
  - rag
  - anthropic-claude
  - fintech
  - 1인-프로젝트
---

# 스타 0개 짜리 1인 레포를 진지하게 까보는 이유 — Nyam2C/FinClaw

## 도입

솔직하게 시작하자. `Nyam2C/FinClaw`는 스타 0개, 포크 0개, 워치 0개, 이슈 0개짜리 레포다. GitHub About 칸은 텅 비어 있고, 토픽도 홈페이지도 라이선스도 없다. 평소 같으면 그냥 지나쳤을 종류의 레포다.

그런데 README를 여는 순간 손이 멈췄다. 460줄짜리, 거의 운영 매뉴얼 수준의 문서가 한국어로 빼곡하게 채워져 있었다. mermaid 의존성 그래프, 시퀀스 다이어그램, 환경변수 전수표, 트러블슈팅 실측표, 보안 모델까지. "1인 학습·감사용 포트폴리오"라고 본인이 못 박은 레포가 왜 이렇게까지 진지하지? 그래서 코드를 직접 클론해서 까봤고, 까보고 나니 글을 쓰지 않을 수가 없었다. 한마디로 미리 말하자면 — 이건 "스타가 없어서 안 본 사람만 손해"인 레포다.

## 정체

먼저 이게 뭔지부터. FinClaw는 Anthropic Claude를 코어로 박고, Discord·Web UI·TUI 세 채널에서 동시에 호출할 수 있는 **1인용 금융 특화 AI 비서**다. 시세·뉴스·알림·거래이력·기억(RAG)·cron 자동화를 SQLite 단일 파일에 전부 영속화한다. TypeScript 99.5%, 11개 패키지짜리 pnpm 모노레포고, 2026년 2월 10일 첫 커밋부터 5월 21일까지 약 3.4개월 동안 222커밋이 쌓였다. 사실상 1인(Nyam2C) 개발이고, 나머지 10커밋은 dependabot이다.

여기서 정직하게 짚어야 할 게 있다. **트렌딩한 건 FinClaw가 아니다.** 검색을 아무리 돌려도 `Nyam2C/FinClaw`에 대한 외부 언급은 단 한 건도 없다. HN, Reddit, X, 블로그 전부 무소득. 그럼 왜 지금 이걸 보냐고? 이 레포가 자칭하는 "OpenClaw의 금융 특화 축소판"이라는 정체성이, 2026년 상반기 오픈소스 씬에서 가장 뜨거웠던 사건의 한복판에 있기 때문이다.

그 사건의 주인공은 **OpenClaw**다. Peter Steinberger(PSPDFKit 창업자)가 만든 자율 AI 비서로, 2026년 1월 말 입소문이 터지면서 며칠 만에 9천 → 6만 스타를 찍었고 이후 수십만 스타(2026년 3월 기준 247K)까지 올라간, "GitHub 역사상 가장 빠르게 성장한 오픈소스"라 불리는 프로젝트다. `input → context → model → tools → repeat → reply`라는 에이전트 루프(Claude Code와 똑같은 패턴)에 멀티채널과 메모리, 50개 넘는 통합을 얹은 물건이다. 참고로 이름이 `~Claw`로 끝나는 건 사연이 있다. 원래 이름이 `Clawdbot`이었는데, Anthropic이 "Clawd"가 "Claude"랑 너무 비슷하다고 상표 이의를 걸어서 `Moltbot`("탈피한 갑각류")으로 바꿨다가, 3일 만에 또 `OpenClaw`로 바꿨다. "오픈소스 역사상 가장 빠른 삼중 리브랜드"라는 소리까지 들었다. FinClaw가 이름에 `Claw`를 박고 코어엔 Claude를 박은 건, 이 분쟁의 진원지를 대놓고 계승한 작명인 셈이다. 좀 짓궂은 오마주다.

그리고 또 하나, 더 중요한 계보가 있다. OpenClaw 본체는 40만 줄, 수백 개 의존성으로 비대해졌고 스킬 마켓플레이스엔 악성 패키지가 1천 개 넘게 쌓였다. 이 "너무 커서 아무도 감사 못 한다"는 비판에 대한 응답으로 Gavriel Cohen이 **NanoClaw**를 내놨다. OpenClaw를 4천 줄 TypeScript — 원본의 1% — 로 재작성해서 "사람이나 보조 AI가 8분이면 전체를 감사할 수 있다"고 표방한 물건이다. Andrej Karpathy가 이걸 두고 "코어가 내 머리와 AI 에이전트의 머리 양쪽에 들어가서, 다룰 수 있고 감사 가능하며 유연하게 느껴진다"고 호평했고, 이로써 "사람뿐 아니라 AI도 읽을 수 있는(컨텍스트 윈도우에 들어가는) 코드베이스"가 새 아키텍처 기준으로 떠올랐다.

FinClaw README가 설계 1순위로 못 박은 두 가지 — "(1) 전체 코드를 한 사람이 읽고 수정 가능한 크기, (2) 감사 가능성·환각 방지" — 가 바로 이 NanoClaw 사상의 직접 계승이다. 내 생각엔 이게 이 레포를 읽는 가장 정확한 렌즈다. FinClaw가 "OpenClaw의 축소판"이라 할 때 그 "축소"는 단순한 기능 줄이기가 아니라, NanoClaw가 촉발하고 Karpathy가 정당화한 "소규모·감사 가능·AI-readable 코드베이스" 운동을 금융이라는 고위험 도메인에 가져온 거다. (다만 정직하게: FinClaw는 약 66K LOC라 NanoClaw의 4K보다 훨씬 크다. 표방의 정도 차이는 분명히 있다.)

참고로 GitHub에 "FinClaw"/"OpenFinClaw"라는 이름은 최소 5~6개가 굴러다닌다. `Fin-Chelae/FinClaw`(Python, 밈코인 자동배포까지), `martinpmm/Finclaw`(Python, 콘셉트가 가장 유사), `NeuZhou/finclaw`(퀀트/백테스트 엔진) 등등. "FinClaw"는 한 사람의 작명이라기보다 OpenClaw 생태계가 만들어낸 하나의 장르에 가깝다. `Nyam2C/FinClaw`는 그중 TypeScript로, 포크가 아니라 패턴만 차용해 처음부터 재구현한 한 구현체다. 이 글에서 그냥 FinClaw라고 하면 항상 `Nyam2C/FinClaw`를 가리킨다.

## 아키텍처·구현 디테일

코드를 까보면 가장 먼저 눈에 띄는 건 `packages/server/src/main.ts`다. 706줄짜리 단일 함수 `main()`이 전체의 "조립 루트(composition root)" 역할을 한다. env 검증 → 로거 → config → storage → Discord 로그인 → provider/도구 레지스트리 → 메모리 3서비스 → 파이프라인 → 라우터 → 게이트웨이 → 스케줄러 순으로 모든 객체를 손으로 생성하고 주입한 뒤, `ProcessLifecycle`에 cleanup까지 등록한다. DI 컨테이너 같은 마법(decorator/reflection)을 일부러 안 쓴다. 의존성이 위에서 아래로 한 방향으로만 흐르고, 하위 모듈은 전부 인터페이스로 주입받는다. "한 사람이 전부 읽고 감사 가능"이라는 목표가 가장 선명하게 드러나는 파일이다. 읽다 보면 "아, 이 사람 진짜 자기가 한 말 지키려고 하는구나" 싶다.

데이터 흐름도 깔끔하다. Discord 멘션이 들어오면 `MessageRouter`가 dedupe → binding → queue → lane을 거쳐 `AutoReplyPipeline`으로 넘기고, 파이프라인은 8단계 스테이지(`normalize → command → memory-capture → ack → context → memory-retrieval → execute → deliver`)를 직렬로 흘린다. execute 단계에서 `Runner`가 LLM 스트림을 돌리며 tool_use 루프를 태우고, 도구가 storage나 외부 API를 건드린 결과를 다시 채널로 돌려보낸다. 별도 경로로 `SchedulerService`가 매 분 폴링하며 due한 cron을 같은 `Runner`로 실행한다. 그리고 Web/TUI/CLI/수동 트리거 같은 외부 표면은 전부 `createGatewayServer`의 JSON-RPC 한 곳(39개 메서드)으로 수렴한다. tui와 web 패키지는 코드상 `types`만 import하고 런타임엔 WebSocket으로 게이트웨이를 호출한다. 결합이 느슨하다.

그 8단계 파이프라인에서 인상적이었던 건 취소와 타임아웃을 다루는 방식이다.

```typescript
// packages/server/src/auto-reply/pipeline.ts:117-118
// AbortSignal.any: 외부 취소 + 파이프라인 타임아웃 결합
const combinedSignal = AbortSignal.any([signal, AbortSignal.timeout(this.config.timeoutMs)]);
```

별다른 라이브러리 없이 웹 표준 `AbortSignal.any`/`AbortSignal.timeout` 두 줄로 "인터럽트 모드(새 메시지가 오면 진행 중이던 처리를 취소)"와 타임아웃을 합쳐버린다. `MessageRouter`가 세션별 `AbortController`를 들고 있다가 새 메시지가 오면 `existing.abort()`로 이전 처리를 끊는(message-router.ts:99-106) 것과 정확히 짝을 이룬다. OpenClaw류 에이전트의 "interrupt" 동작을 외부 의존성 없이 표준 API만으로 구현한 거다. 깔끔하다는 말이 절로 나왔다. 참고로 LLM을 돌리는 `Runner`는 `maxTurns`(기본 10) while 루프로 tool_use를 반복하면서, LLM 호출만 `classifyFallbackError`로 rate-limit/server-error/timeout만 골라 재시도하고 4xx 클라이언트 에러는 재시도하지 않는다(runner.ts:129-138). 재시도 분류를 정확히 나눈 게 보였다.

## 박수 포인트

코드를 한참 보다 보니 박수 칠 만한 지점이 한두 개가 아니었다. 1인 프로젝트라는 사실을 자꾸 잊게 만든다.

**1. RAG 인용 마커를 응답에서 역추출한다 (감사 가능성의 진심).** 이게 제일 인상적이었다. 메모리 검색 단계에서 system prompt에 주입하는 기억마다 `[mem:abc123]` 같은 마커를 붙인다(memory-retrieval.ts:189). 그리고 모델이 응답을 내놓으면, 그 텍스트에서 마커를 정규식으로 다시 긁어낸다.

```typescript
// packages/server/src/auto-reply/execution-adapter.ts:121-136
const re = /\[mem:([a-f0-9]{6,8}(?:,[a-f0-9]{6,8})*)\]/g;
// ... 응답 텍스트의 마커 prefix 수집 후
for (const c of candidates) {
  for (const p of prefixes) {
    if (c.id.startsWith(p)) { matched.push(c.id); break; }
  }
}
```

이렇게 "어떤 기억이 실제로 인용됐는가"를 `agent_runs.used_memory_ids`에 DB로 남긴다. RAG가 환각을 부려도 사후에 추적할 수 있게 만든 거다. "감사 가능성"을 그냥 README에 적은 게 아니라 코드로 증명한 셈이라, 이 부분에서 좀 감탄했다.

**2. LLM에게 도구명을 절대 안 맡긴다 (환각 차단).** Phase 34에서 키워드 휴리스틱을 LLM 라우터로 교체했는데, 그러면서도 LLM에겐 intent 6종만 분류시키고 실제 도구 화이트리스트는 코드 테이블로 매핑한다.

```typescript
// packages/agent/src/router/llm-router.ts:54-72, 105-120
// intent 만 분류시키고 modelTier/allowedToolNames 는 코드 매핑으로 결정 —
// LLM 이 도구명을 환각하는 위험 차단 + 결정론성 확보.
function resolveAllowedToolNames(intent, availableToolNames) {
  const { toolPatterns } = INTENT_POLICY[intent];
  if (toolPatterns.length === 0) return [];          // greeting/memory → 도구 미전달
  if (toolPatterns.includes('*')) return [...availableToolNames];
  return availableToolNames.filter((name) =>
    toolPatterns.some((p) => name === p || name.startsWith(p)));  // 교집합으로 환각 제거
}
```

LLM이 존재하지도 않는 도구명을 지어내는 사고를 구조적으로 막는다. JSON mode가 없으니 `JSON.parse + zod safeParse`로 검증하고 1회 재시도, 그래도 실패하면 키워드 폴백으로 떨어진다. LLM의 신뢰 경계를 의도적으로 좁게 잡은 게 마음에 들었다. "LLM은 의도만 말해, 결정은 코드가 해" 라는 태도.

**3. structured-output 위반을 한 번은 봐준다.** 도구 출력이 Zod 스키마를 어기면 곧장 실패하지 않는다. 다음 턴에 `tool_choice`로 같은 도구를 강제 호출시켜 한 번 더 기회를 주고, 두 번째 위반에서만 throw한다(runner.ts:156-172). LLM의 비결정성을 "재시도 1회"라는 현실적 타협으로 흡수한다. 너무 빡빡하지도, 너무 헐겁지도 않은 균형이 좋았다.

**4. cron이 죽으면 알아서 꺼진다.** 스케줄러가 매 분 폴링하는데, 연속 실패하면 `[30s, 1m, 5m, 15m, 60m]` 지수 백오프를 타다가(scheduler.ts:81-92) 임계(기본 3회)에 도달하면 status를 'failing'→'disabled'로 전이하며 아예 비활성화한다(scheduler.ts:386-393). 주석에 출처까지 박아놨다 — "OpenClaw src/cron/service/timer.ts:107-119 이식". 죽은 cron이 무한 재시도로 시스템을 갉아먹는 걸 막는 실전 디테일이다. 여기에 더해 DNS 핀닝 기반 SSRF 방어(ssrf.ts)까지 있다. `web_fetch` 도구가 LLM 지시로 임의 URL을 가져올 때 사설망(10/8, 127/8, link-local, CGNAT)을 IPv4/IPv6/IPv4-mapped 전부 검사하고, 검증한 IP를 핀닝해서 DNS 재해석(TOCTOU) 공격까지 막는다. 1인 학습 프로젝트치고 위협 모델이 진지하다 못해 좀 과하다 싶을 정도다.

그리고 이 모든 걸 떠받치는 기반이 단단하다. src(테스트 제외)에서 `as any`/`: any`가 **0건**, `@ts-ignore`는 1건뿐이다. 브랜드 타입(`SessionKey`, `AgentId`, `TickerSymbol`)으로 타입을 강제하고, 도구 입력·RPC 파라미터·config·라우터 출력 전부 Zod 런타임 검증을 건다. 테스트는 217파일에 약 903케이스고, 커버리지 임계(statements 70%)를 강제한다. RAG는 벡터 검색과 FTS5를 0.7:0.3 가중으로 융합하고 신선도 감쇠(`rawScore * exp(-daysOld / 90)`)까지 건다. 3개월 지난 기억은 가중치가 약 37%로 떨어진다. 이걸 한 사람이 3.4개월에 만들었다는 게 잘 안 믿긴다.

## 비판·한계

박수만 치면 PR이지 리뷰가 아니다. 정직하게 아쉬운 점도 비슷한 무게로 짚어야겠다.

**문서가 코드를 못 따라간다.** 가장 상징적인 게 README는 "스키마 v6"라고 적혀 있는데 실제 코드는 `SCHEMA_VERSION = 12`(database.ts:23)다. 6단계나 뒤처졌다. 거의 운영 매뉴얼급으로 두꺼운 README가 오히려 코드 속도를 못 따라간 신호다. 문서가 친절한 게 함정이 될 수도 있다는 역설. 처음 보는 사람은 v6라고 믿고 들어갔다가 헷갈릴 거다.

**server 패키지가 너무 비대하다.** 11개 패키지 중 `server` 하나가 24K LOC로 전체의 43%를 차지한다. wiring·gateway·pipeline·automation·CLI·plugins·observability가 다 한 패키지에 응집돼 있다. 앞서 칭찬한 `main()` 706줄도 사실 가독성 한계에 근접한다. "1인이 전부 감사 가능"을 표방하는데, 정작 한 패키지가 이렇게 비대해지면 그 표방이 흔들린다. Phase가 더 쌓이면 분할 압력이 커질 거다.

**단일 SQLite의 동시성은 구조적 한계다.** gateway(WS 100 conn) + scheduler + pipeline이 한 DB 파일을 공유한다. WAL로 read 동시성은 확보했지만 write는 직렬이다. 다행히 본인도 이걸 알고 lane(`maxConcurrent: 1`)으로 agent.run과 schedule을 직렬화해 막아뒀다 — 즉 한계를 인지하고 우회한 구조다. 1인·단일 프로세스 가정에선 안전하지만, 이 가정이 깨지는 순간 병목이 된다.

**금융 도메인인데 한국 시장 종목을 놓친다.** 이게 좀 아이러니한데, 메모리 검색의 티커 심볼 추출이 "대문자 2~5자 정규식"이라 **한국어 종목명이나 6자리 코드는 미지원**이다(memory-retrieval.ts 주석에 본인도 명시). KIS(한국투자증권) 연동을 넣어놓고 정작 KR 종목은 RAG 거래 주입에서 빠진다. 금융 비서로서 도메인상 아쉬운 공백이다.

**라이선스가 없다.** LICENSE 파일도, GitHub licenseInfo도 없다. MIT가 사실상 표준인 OpenClaw 생태계(OpenClaw도 NanoClaw도 MIT)에서 FinClaw만 라이선스가 비어 있다. 의도인지 누락인지는 알 수 없지만, 공개 레포에 라이선스가 없으면 법적으로는 "모든 권리 보유" 상태라 남이 가져다 쓰기 애매하다. truncate 임계가 400K로 하드코딩(runner.ts:190)돼 있고 코드가 스스로 "multi-provider 도입 시 동적 산정 필요"라고 인정한 부분, RPC 메서드가 전역 Map이라 인스턴스 격리가 깨지는 부분(본인 TODO로 인지) 같은 자잘한 빚들도 있다. 그래도 이런 걸 죄다 주석에 솔직히 적어둔다는 게 오히려 신뢰가 갔다.

생태계 차원의 우려도 짚어둘 만하다. OpenClaw 전체가 "로컬 머신에 깊이 접근한 채 상주하는 봇"이라 보안 논쟁의 중심에 있었고(VentureBeat가 "AI 에이전트가 도래했고 혼돈도 도래했다"는 프레임으로 다뤘다), 금융 AI는 환각이 특히 치명적이다(2026년 37개 모델 벤치마크에서 환각률 15~52%). 잘못된 가격이나 존재하지 않는 출처 링크가 오가격·규제 노출로 직결되니까. 내 생각엔 FinClaw가 "프로덕트 출시·다인 사용·자동매매 비대상"이라고 선 긋고, KIS를 read-only(잔고 조회만, 매매 불가)로 제한하고, RAG로 출처를 붙들려는 건 단순한 겸양이 아니라 이런 생태계 비판을 의식한 선제적 방어선으로 읽힌다. 다만 라이선스 미명시는 그 진지함과 따로 노는 빈틈이다.

## 결론

정리하자면, FinClaw는 "지금 트렌딩이라 봐야 하는 레포"가 아니다. 트렌딩한 건 OpenClaw와 NanoClaw고, FinClaw는 그 흐름의 개인적·교육적 반향이다. 그런데도 볼 가치가 있는 이유는, 2026년 상반기 오픈소스 씬을 지배한 서사(OpenClaw 폭발 → 상표 분쟁 → NanoClaw "감사 가능성" 논쟁)를 한 사람이 금융이라는 고위험 도메인에 정직하게 이식했고, 그걸 코드로 증명했기 때문이다.

**이런 사람한테 잘 맞는다.** 첫째, LLM 에이전트의 "제대로 된" 구조 — tool-calling 루프, 인터럽트, RAG 인용 추적, 환각 차단 라우팅 — 를 실제 코드로 배우고 싶은 개발자. 둘째, NanoClaw식 "1인 감사 코드베이스" 철학이 실전에서 어떻게 구현되는지 샘플이 필요한 사람. 셋째, AI(Claude Code 하네스)로 만든 코드를 어떻게 감사 가능하게 유지하는가에 관심 있는 사람. FinClaw 자체가 phase 기반 plans로 AI가 만든 결과물이라 그 메타 서사까지 흥미롭다.

**이런 사람은 피하는 게 낫다.** 당장 프로덕션에 올릴 다인용 금융 서비스를 찾는 사람(애초에 비대상이라 못 박혀 있다), 자동매매나 백테스팅 엔진이 필요한 사람(read-only 조회만 된다), 라이선스가 명확해야 채택할 수 있는 회사. 그리고 한국 시장 종목 RAG가 핵심인 사람도 지금은 막힌다.

프로덕션에 쓸 물건이냐 하면 아니다. 본인도 "프로덕트 아님"이라 못 박았다. 하지만 "프로덕션 품질을 학습·연습한" 1인 포트폴리오로서는 내가 최근 본 것 중 손에 꼽게 진지하다. 스타 0개라는 숫자만 보고 넘기면 아까운 레포다. 개인적으로는 이 사람이 server 패키지 비대화를 어떻게 쪼개는지, 한국 종목 추출을 언제 붙이는지 phase가 더 쌓이는 걸 지켜보고 싶어졌다. 그리고 다음에 LLM 에이전트 사이드 프로젝트를 만들 일이 생기면, 이 RAG 인용 역추출 패턴은 꼭 한번 훔쳐 써볼 생각이다.

---

### 참고

- OpenClaw 전반: [Wikipedia](https://en.wikipedia.org/wiki/OpenClaw), [ByteByteGo - Top AI GitHub Repos 2026](https://blog.bytebytego.com/p/top-ai-github-repositories-in-2026)
- 삼중 개명 사가: [Trending Topics](https://www.trendingtopics.eu/clawdbot-moltbot-anthropic/), [Fortune - Who is Peter Steinberger](https://fortune.com/2026/02/19/openclaw-who-is-peter-steinberger-openai-sam-altman-anthropic-moltbook/)
- NanoClaw·"감사 가능성" 철학: [VentureBeat](https://venturebeat.com/orchestration/nanoclaw-solves-one-of-openclaws-biggest-security-issues-and-its-already), [DataCamp](https://www.datacamp.com/blog/nanoclaw-vs-openclaw), [The Register](https://www.theregister.com/2026/03/01/nanoclaw_container_openclaw/)
- 같은 장르 형제 프로젝트: [Fin-Chelae/FinClaw](https://github.com/Fin-Chelae/FinClaw), [martinpmm/Finclaw](https://github.com/martinpmm/Finclaw), [NeuZhou/finclaw](https://github.com/NeuZhou/finclaw)
- 금융 AI 환각·규제: [Baytech - AI Hallucinations in Financial Services](https://www.baytechconsulting.com/blog/hidden-dangers-of-ai-hallucinations-in-financial-services), [BCLP - AI Regulation in Financial Services](https://www.bclplaw.com/en-US/events-insights-news/ai-regulation-in-financial-services-turning-principles-into-practice.html)
