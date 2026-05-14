---
repo: gstack
full_name: garrytan/gstack
url: https://github.com/garrytan/gstack
language: typescript
stars: 96163
date: 2026-05-14
one_line: "YC 사장 Garry Tan이 본인 Claude Code 워크플로우를 통째로 묶어 푼 23개 슬래시 커맨드 + 헤드리스 브라우저 + ETHOS 자동 주입 메타 시스템 — 한 사람의 미적 강박을 인프라로 박제한, 사랑과 미움이 동시에 viral의 연료가 된 트렌딩 1위 레포."
---

# 별 96K짜리 "exact setup" — Garry Tan은 본인 워크플로우를 어떻게 도구로 박제했나

## 도입

타임라인에 두 달째 같은 단어가 떠다녔다. "god mode." 처음엔 또 인플루언서 마케팅이구나 하고 흘렸다. 그런데 별이 두 달 만에 96,163개를 찍는 걸 보고, 더는 외면이 안 됐다. YC 사장 Garry Tan이 본인이 매일 쓰는 Claude Code 설정을 통째로 푼 [gstack](https://github.com/garrytan/gstack). 첫 트윗은 정확히 이랬다 — *"I've been having such an amazing time with Claude Code I wanted you to be able to have my exact skill setup."* "exact"라는 단어가 묘하게 거슬렸다. 도구는 보통 "내 워크플로우의 패턴을 추출했다"고 말하지, "내가 쓰는 그대로다"라고는 안 한다.

레포를 열고 한 시간을 헤맸다. 마크다운이 코드보다 훨씬 많고, README 한 페이지가 사실상 본인 출력 기록 자랑이고, CHANGELOG가 590KB다. 그런데 한 겹 더 들어가니 `browse/src/server.ts` 2,406줄짜리 헤드리스 브라우저 데몬이 깔려 있다. 가벼운 도구라 칭하긴 어렵고, 단순 프롬프트 모음이라 비웃기엔 코드가 너무 정성스럽다. 이 글은 그 양면 — 마케팅처럼 보이지만 실제로는 어디까지 만들어졌는가, 그리고 그게 왜 별 96K를 정당화하거나 정당화하지 못하는가 — 를 같이 풀어본다.

## 정체

한 줄로 말하면 이렇다. **Claude Code(와 9개 다른 AI 에이전트) 위에 얹는 "23개 슬래시 커맨드 + 헤드리스 브라우저 CLI" 묶음.** 사용자는 설치 후 `/plan-ceo-review`, `/plan-eng-review`, `/plan-design-review`, `/ship`, `/qa`, `/cso`, `/canary` 같은 명령을 친다. 각 명령 뒤에는 "CEO·Eng Manager·Designer·보안 책임자·QA 리드·릴리즈 엔지니어" 같은 가상 역할이 박혀 있고, 그 역할들이 차례로 사용자 코드를 비판하고 다듬는다. 1인 개발자가 가상의 20인 팀처럼 출시한다는 게 핵심 카피.

설치는 한 줄이다. `git clone --depth 1 https://github.com/garrytan/gstack.git ~/.claude/skills/gstack && ./setup`. 호스트 어댑터로 Claude Code, OpenAI Codex CLI, OpenCode, Cursor, Factory Droid, Slate, Kiro, Hermes, GBrain까지 10개 에이전트를 지원한다. `--host claude` 같은 플래그로 같은 스킬 템플릿에서 호스트별 변형이 한 번에 떨어진다.

만든 사람은 [Garry Tan](https://www.augmentcode.com/learn/garry-tan-gstack-claude-code) — Palantir 초기 멤버, Posterous를 Twitter에 매각한 공동창업자, Initialized Capital 공동창업자, 현재 Y Combinator 사장 겸 CEO. 본인 라인업이 이미 디자이너/엔지니어/투자자/임원을 한 몸에 얹은 사람이라, 가상 팀이라는 메타포가 우연이 아니다. README는 Andrej Karpathy의 "12월 이후 코드 한 줄도 안 쳤다"를 인용하면서 시작하고, 곧장 "2026 logical LOC가 2013 대비 810배"라는 본인 자랑으로 넘어간다. 도구 README라기보단 YC 블로그 한 편을 옮긴 톤이다. 카테고리상으로는 "Claude Code 위에 얹는 워크플로우 레이어"라서 Cursor(IDE 통합) / Aider(자체 에이전트)와는 다른 계층이고, [Anthropic 공식 skills 마켓플레이스](https://github.com/anthropics/claude-plugins-official)에 대한 한 개인의 큐레이션 묶음 — 그 자리에 있다.

공개 시점은 2026-03-12. 트윗 하나로 [Product Hunt 트렌딩](https://www.marktechpost.com/2026/03/14/garry-tan-releases-gstack-an-open-source-claude-code-system-for-planning-code-review-qa-and-shipping/), 며칠 만에 별 ~2만, 4월 6일에 66K, 5월 현재 96K. 트렌딩 곡선이 잘 빠진 정도가 아니라 거의 수직이다.

## 아키텍처

전체 구조를 한 그림으로 그리면 이렇다.

```
┌── 마크다운 레이어 (스킬 = 절차서) ─────────────────────┐
│ qa/SKILL.md, ship/SKILL.md, review/SKILL.md ... (50+)│
│  └ frontmatter(YAML) + Preamble(bash) + 단계별 지시문 │
│  └ scripts/gen-skill-docs.ts 가 .tmpl → .md 생성     │
└──────────────────────────┬────────────────────────────┘
                           │ (Claude가 읽고 실행)
                           ▼
┌── bin/ 셸·TS 유틸 (60+ 진입점) ────────────────────────┐
│ gstack-config, gstack-update-check, gstack-slug ...   │
└──────────────────────────┬────────────────────────────┘
                           │
                           ▼
┌── browse/ 서브시스템 (TS+Bun, 55개 .ts 파일) ─────────┐
│  CLI(thin) ──HTTP──▶ 데몬(Bun.serve) ──CDP──▶ Chrome │
│  상태파일 .gstack/browse.json (PID·포트·토큰)         │
└───────────────────────────────────────────────────────┘
```

사용자가 `/ship`을 치면 Claude가 `ship/SKILL.md`(3,056줄)을 그대로 실행 절차로 읽는다. 각 Step에서 `~/.claude/skills/gstack/bin/`의 셸 유틸이나 `$B`(컴파일된 browse 바이너리)를 호출한다. browse 데몬은 30분 idle timeout, 랜덤 포트(10000~60000), bearer 토큰 인증으로 localhost에서만 떠 있는다. 핵심은 **마크다운 절차서가 1차 산출물이고, 그 아래 TypeScript가 보조 인프라**라는 점이다. 통상 도구의 위계가 뒤집힌 구조.

코드 통계도 인상적이다. SKILL.md 누계 56,039줄 vs `browse/src` TypeScript. 즉 절차서 분량이 자체 코드보다 많다. "프롬프트 엔지니어링 = 빌드 타겟"으로 격상시킨 시도라고 해도 과언이 아니다. 의존성은 의외로 적다. Playwright/puppeteer-core로 브라우저, HuggingFace transformers로 prompt injection 분류기, ngrok으로 터널, 나머지는 모두 자체 코드. 런타임 dep 7개, devDep 4개. Bun 엔진(node_modules 없이 단일 바이너리로 컴파일).

## 구현 디테일

세 개만 짚어본다. 다 보면 한참 걸리는 코드라서.

**1) SKILL.md frontmatter — 슬래시 커맨드 등록이 별도 코드가 아니다**

출처: `qa/SKILL.md:1-19` (대표 예)

```yaml
---
name: qa
description: |
  Systematically QA test a web application and fix bugs found. ...
  Use when asked to "qa", "QA", "test this site", "find bugs", ...
  Proactively suggest when the user says a feature is ready for testing
  or asks "does this work?". ...
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
triggers:
  - quality check
  - test the app
  - run QA
---
```

이게 gstack의 진짜 영리한 부분이다. **SKILL.md는 사람이 읽는 README이자 동시에 Claude가 파싱하는 라우팅 매니페스트**다. `description`에 트리거 문구를 자연어로 박아두면 Claude가 사용자 발화와 매칭해서 자동 호출한다. 슬래시 커맨드 등록 메커니즘이 별도 코드 한 줄도 없이 마크다운 frontmatter로 끝난다는 것 — 이 단순함이 본질이다.

**2) Chromium fail-fast — 숨기지 말고 죽여라**

출처: `browse/src/browser-manager.ts:2-7`

```ts
/**
 * Chromium crash handling:
 *   browser.on('disconnected') → log error → process.exit(1)
 *   CLI detects dead server → auto-restarts on next command
 *   We do NOT try to self-heal — don't hide failure.
 */
```

Chromium이 죽으면 서버도 같이 죽고, CLI가 다음 호출에서 새 데몬을 띄운다. "self-heal 하지 마라"는 명시적 주석. 복잡한 상태 복구 로직 대신 프로세스 경계를 신뢰하는 단순함, 이걸 모듈 헤더에 한 단락으로 솔직히 적어놓은 게 인상적이다. `server.ts`, `cli.ts`도 같은 톤으로 "왜·어떻게·실패 모드"를 적는다.

**3) AUTH_TOKEN sanitize — 유니코드 화이트스페이스까지 막는다**

출처: `browse/src/server.ts:71-84`

```ts
function sanitizeAuthToken(raw: string | undefined): string | null {
  if (!raw) return null;
  // strip ALL unicode whitespace (not just ASCII — .trim() misses
  // U+200B / U+FEFF / U+00A0 / etc.)
  const stripped = raw.replace(/[\s​-‍﻿]/g, '');
  if (stripped.length < 16) return null;
  return stripped;
}
const AUTH_TOKEN = sanitizeAuthToken(process.env.AUTH_TOKEN) || crypto.randomUUID();
```

임베더가 BOM 하나짜리 토큰을 실수로 넘겨도 거부하고 randomUUID로 fallback. 보안 경계가 "조용히 약해지지 않게" 하는 디테일이 잘 박혀 있다. 한 함수에 의도와 실패 모드가 다 적혀 있다.

데이터 흐름 보면, 모든 스킬 호출이 매 시작마다 `bin/gstack-config get`, `find ~/.gstack/sessions -mmin -120`, `gstack-update-check` 같은 동기 셸 호출 60~100줄을 Preamble로 echo한다. 매번 같은 컨텍스트를 다시 깐다는 뜻이다. 일관성은 보장되는데 컨텍스트 윈도우가 그만큼 빠진다. 이 트레이드오프는 박수 섹션에서 한 번, 비판 섹션에서 한 번 더 짚는다.

## 박수 포인트

박수칠 만한 것들은 분명하다. 진심으로 정리한다.

### 1. 마크다운 매니페스트 = 빌드 타겟이라는 발상

방금 본 frontmatter 한 덩이가 곧 라우팅이고, 곧 매뉴얼이고, 곧 사용자 문서다. `scripts/gen-skill-docs.ts`가 `.tmpl`에서 `{{PLACEHOLDERS}}`를 치환해 `.md`를 생성하고, `--host` 플래그 하나로 10개 에이전트용 변형이 동시에 떨어진다. 빌드 시스템의 1차 산출물이 마크다운이라는 시각 — 이걸 처음부터 끝까지 일관되게 밀어붙인 도구는 흔하지 않다.

### 2. ETHOS 자동 주입이라는 시그니처 디자인

이게 진짜 가장 가운데 손가락이 까딱이는 부분이다. `ETHOS.md`에 3원칙이 박혀 있다 — (1) **Boil the Lake** "완전성은 싸다, 지름길은 옛날 사고", (2) **Search Before Building** "Layer 1 검증된 / Layer 2 인기 / Layer 3 first principles", (3) **User Sovereignty** "AI는 추천, 사용자가 결정". 그리고 `CLAUDE.md`에 따르면 이 ETHOS와 "AI effort compression" 테이블이 **모든 스킬 preamble에 자동 인젝션된다**. 사용자가 `/review`를 부르든 `/qa`를 부르든 Claude는 매번 Garry Tan의 3원칙을 먼저 읽고 본문을 시작한다.

이게 왜 박수인가. 도구를 쓰는 행위 자체에 본인의 의견을 슬래시 커맨드 한 줄마다 주입하는 메타 아키텍처다. 일관된 의견 = 일관된 산출물이라는 가설을 디자인 결정 한 줄로 박은 셈. 동시에 비판 섹션에서 양면을 짚을 텐데, "개인의 미적 강박을 인프라로 박제한다"는 비판도 정확히 같은 곳에서 출발한다.

### 3. browse/ 서브시스템 — 별도 OSS 수준의 정성

`browse/`는 그 자체로 별도 프로젝트 규모다. `server.ts` 2,406줄, `browser-manager.ts` 1,504줄, `cli.ts` 1,192줄. Bun 컴파일로 58MB 단일 바이너리. 듀얼 리스너로 local/tunnel 트래픽을 **물리적 포트로** 분리 — 헤더 추론 대신 소켓 분리가 더 강한 보안 보장이라는 판단이 박혀 있다. ngrok이 전달하는 포트에는 `/health`나 `/cookie-picker`가 아예 존재하지 않는다. 6레이어 prompt-injection 방어(content-security + TestSavantAI ONNX + Claude Haiku transcript classifier + canary), `~/.gstack/security/attempts.jsonl`에 공격 로그까지 남긴다. 쿠키 DB는 read-only 복사 후 메모리에서만 복호화. 보안 모델이 `ARCHITECTURE.md` 한 섹션을 다 차지한다. HN에서 비판 진영조차 [브라우저 자동화는 호평했다](https://news.ycombinator.com/item?id=47418576)는 게 이해된다.

### 4. 테스트와 도큐먼트의 진심

테스트는 `test/`(136) + `browse/test/`(93) + `make-pdf/test/` 합쳐서 240+ `.test.ts`. Free tier(`bun test` 무료, 2초 이하), paid eval tier(LLM judge, 1회 ~$4), E2E, gate, periodic으로 분류. `test/helpers/touchfiles.ts`로 diff 기반 테스트 선택까지 만들어두었다. 도큐먼트는 ARCHITECTURE.md 30KB, CLAUDE.md 45KB, BROWSER.md 60KB, ETHOS.md 7.5KB — 정보 과잉이긴 한데 "왜 이렇게 했나"가 솔직하게 적혀 있다. ARCHITECTURE.md의 "Bun을 고른 이유" 한 섹션은 누구든 한 번 읽어볼 만하다 (native SQLite로 쿠키 DB 직접 읽기, native TS로 빌드 단계 없음, 빌트인 HTTP).

### 5. 스킬 합성 — 함수처럼 호출되는 워크플로우

`/autoplan`이 `/plan-ceo-review`, `/plan-eng-review`, `/plan-design-review`를 순차 호출한다. `/ship`이 `/review`를 부르고, `/land-and-deploy`가 `/ship` 다음에 들어온다. 스킬이 함수처럼 컴포지블하게 호출되도록 인터페이스가 설계되어 있다. [Pulumi 블로그](https://www.pulumi.com/blog/claude-code-orchestration-frameworks/)와 [DEV.to의 스킬 스택 가이드](https://dev.to/imaginex/a-claude-code-skills-stack-how-to-combine-superpowers-gstack-and-gsd-without-the-chaos-44b3)가 gstack을 Superpowers, GSD 같은 다른 프레임워크와 stack 형태로 조합하는 워크플로우를 별도로 다루는 게 우연이 아니다. 스킬을 빌딩블록으로 노출하는 인터페이스가 진짜로 잘 굴러간다는 외부 증거다.

## 비판·한계

박수 잘 쳤으니 이제 솔직하게.

### 1. 별 96K의 절반 이상은 Garry Tan 개인 브랜드의 영향이다

피하기 어려운 가설이다. [HN 상위 댓글](https://news.ycombinator.com/item?id=47418576)이 가장 정확하게 짚는다 — *"If he weren't the CEO of YC, this wouldn't be on PH"* (Product Hunt). 같은 레포가 무명 dev 손에서 나왔다면 별 1만은 어려웠을 거라는 nepotism 비판. [TechCrunch도 종합 기사](https://techcrunch.com/2026/03/17/why-garry-tans-claude-code-setup-has-gotten-so-much-love-and-hate/)에서 "다른 개발자들도 이미 사적으로 비슷한 걸 만들어 썼다"는 Mo Bitar 인용을 박는다. 객관적 증거 하나만 보태면 — 최초 커밋부터 50번째 커밋까지 100% Garry Tan 단독 commit. 외부 PR이 머지될 때도 squash해서 Garry 이름으로 들어간다. 14,303개 fork와 285개 PR이라는 관심 대비, 실제 contributor 다양성은 거의 0이다. 이 비대칭은 "exact setup" 카피와 정합적이지만, 트렌딩 1위급 별 수의 정당성을 코드만으로 설명하긴 어렵다.

거기에 본인이 [지른 트윗](https://x.com/garrytan/status/2032196172430131498)이 백래시를 키웠다. CTO 친구 문자 인용 — *"Your gstack is crazy. This is like god mode. I will make a bet that over 90% of new repos from today forward will use gstack."* 이 "god mode"·"90% 트윗"이 백래시의 핵심 도화선이 됐다. HN 상위 댓글 *"LOC will never be a good metric of software engineering"*(57표), *"overengineered and deeply into its own form — it will not make your agents better"*(74점)이 모두 같은 텐션에서 출발한다.

### 2. 셸 스크립트 의존도와 실제 보안 이슈

`bin/` 60+ 진입점 중 다수가 `#!/usr/bin/env bash`. 모든 스킬 Preamble에서 `gstack-config get`, `find ~/.gstack/sessions -mmin -120` 같은 호출이 동기 실행된다. WSL/Windows 호환을 위해 별도 `server-node.mjs` 빌드 경로까지 만든 흔적이 있다. 그리고 이게 실제 이슈로 등록됐다.

- [Issue #965](https://github.com/garrytan/gstack/issues/965): `/codex`와 `/autoplan`이 codex CLI에 인증 프로파일 게이트나 consent prompt 없이 그냥 shell out. 여러 codex 계정이 있으면 사용자가 모르는 채로 임의 계정이 쓰임.
- [Issue #1193](https://github.com/garrytan/gstack/issues/1193): `which codex`로 detect하는데 POSIX 비표준이라 플랫폼별 동작이 다름. `command -v`로 바꿔야 함. 결과적으로 `/review`가 "Codex CLI not installed"라고 거짓 출력하고 사용자가 Codex 검증 없이 PR을 승인하는 사례가 발생.

보안 모델은 정성스러운데, 사용자 신원·외부 CLI 신뢰 경계 같은 부분에서는 아직 거친 면이 있다.

### 3. 한 사람의 워크플로우를 강제하는 컬렉션이라는 양면성

`CLAUDE.md`에 명문화된 **Community PR 가드레일**이 흥미롭다 — ETHOS.md를 수정하는 PR, YC 언급을 제거하는 PR, Garry의 보이스를 변경하는 PR은 무조건 사용자 확인. 외부 기여의 범위가 의도적으로 제한된다. 오픈소스인데 오픈소스 같지 않은 경계다.

여기에 CHANGELOG 590KB의 강제 보이스 포맷이 겹친다. 모든 엔트리가 "Two-line bold headline that lands like a verdict, not marketing" + 측정 테이블 + "What this means for builders" + Itemized changes 4섹션을 강제. "AI vocabulary 금지(delve, robust, comprehensive, nuanced...)", "em dash 금지" 같은 보이스 규칙을 자기 자신에게 자동화로 적용한다. 자기 톤을 인프라로 박제하려는 시도다.

좋게 보면 일관성. 비판적으로 보면 한 명의 미적 강박을 590KB짜리 CHANGELOG와 모든 PR 가드레일과 모든 스킬 preamble로 강제하는 구조다. 이 도구를 쓰는 모든 사용자는 Garry Tan의 ETHOS와 보이스를 매 요청마다 컨텍스트로 끌고 가야 한다. 사용자의 컨텍스트 윈도우에서 본인 코드 자리가 줄어든다.

### 4. 코드 자체의 거친 면들

- **컴파일된 바이너리가 git에 트래킹된다.** `browse/dist/browse`, `design/dist/design`, `make-pdf/dist/pdf`가 저장소에 들어 있어서 레포 크기 97MB. CLAUDE.md에 본인이 "역사적 실수, `git rm --cached` 예정"이라고 자인은 했는데, 두 달째 안 빠졌다.
- **TypeScript 단일 파일 비대화.** `server.ts` 2,406줄, `browser-manager.ts` 1,504줄. 모듈 분리는 되어 있는데 핵심 파일은 여전히 길다.
- **거대한 SKILL.md.** `ship/SKILL.md` 3,056줄, `review/SKILL.md` 1,744줄, 루트 `SKILL.md` 1,649줄. 매 스킬 호출마다 동일한 Preamble bash 60~100줄을 다시 echo. 컨텍스트 비용이 누적된다.

### 5. "결국 프롬프트 모음"이라는 카테고리 비판

[TechCrunch 인용의 Mo Bitar](https://techcrunch.com/2026/03/17/why-garry-tans-claude-code-setup-has-gotten-so-much-love-and-hate/)와 HN 비평가들이 공통적으로 말한다 — *"a bunch of prompts."* 이 비판이 부분적으론 정당하다. 자체 코드 양보다 마크다운 절차서가 더 많고, 핵심 가치는 결국 Garry가 큐레이션한 "Claude에게 이렇게 일하라고 시키는 방식"에 있다. browse/는 분명 별도 프로젝트 수준이지만, gstack의 *고유* 자산은 browse/가 아니라 SKILL.md 컬렉션이다. 그리고 그 컬렉션이 곧 Garry 의 의견 — 이라는 점은 박수 섹션과 비판 섹션의 양면이 만나는 지점이다.

## 결론

이 도구가 잘 맞는 사람.

- **1인 또는 소규모 팀으로 Claude Code에서 진심으로 출시 사이클을 돌리는 사람.** plan → review → ship → canary 흐름을 처음부터 짜는 비용이 진짜 크다. 누군가 의견 강한 디폴트로 깔아주면 첫 인프라가 빠르게 잡힌다.
- **AI 코딩 에이전트 스킬 생태계의 reference implementation이 궁금한 사람.** SKILL.md frontmatter + .tmpl 빌드 파이프라인은 [Anthropic 공식 스킬 레퍼런스](https://github.com/anthropics/skills) 옆에 두고 비교하기 좋은 본격 예시다.
- **헤드리스 브라우저를 AI 워크플로우에 박을 일이 있는 사람.** `browse/` 단독으로도 학습 가치가 있다. 듀얼 리스너 보안, fail-fast 데몬, prompt injection 6레이어 방어가 한자리에 있는 코드는 흔치 않다.

피해야 할 사람.

- **이미 본인 워크플로우가 잡힌 시니어.** Garry의 ETHOS와 보이스가 매 스킬 preamble로 따라오고, PR 가드레일이 ETHOS 수정을 막는다. 본인 의견과 충돌하면 피곤하다.
- **엔터프라이즈/규제 환경.** 셸 스크립트 의존, 외부 CLI(codex) shell out, 컴파일된 바이너리 git 트래킹, 1인 메인테이너. 감사 통과 시키기 어렵다.
- **"도구 자체가 신호"가 아닌 곳을 찾는 사람.** Garry Tan의 페르소나가 이 레포의 본질이다. 그 부분이 거슬리면 본인이 직접 마크다운 절차서 모음을 짜는 게 빠르다 — 그게 정확히 비판자들의 주장이고, 일리 있다.

프로덕션에서 쓸만한가. browse/와 보안 모델은 명백히 프로덕션 수준이다. SKILL.md 절차서는 *Garry의 출시 사이클*에는 프로덕션이고, 다른 사람의 사이클에는 출발점이다. 한 달간 v1.1.0 → v1.34.1.0이라는 페이스도 둘로 읽힌다. 활발하게 돌아간다는 신호일 수도, 큐레이션 권한이 한 명에게 묶인 채 빠르게 흘러간다는 신호일 수도. 둘 다 맞다.

마지막으로 솔직한 감상 두 마디. 첫째, 이 레포는 코드 리뷰 대상이 아니라 **인물 시그널과 도구가 한 몸이 된 케이스 스터디**로 읽는 게 정확하다. "사랑과 미움이 동시에 viral의 연료"가 됐다는 게 그래서 가장 정확한 표현이다. 둘째, 그럼에도 ETHOS 자동 주입, SKILL.md 매니페스트 패턴, browse/의 듀얼 리스너 — 이 세 가지만큼은 별 96K가 아니어도 따로 한 번 더 살펴볼 만한 디자인이다. 다음에 들여다볼 건 [anthropics/claude-plugins-official](https://github.com/anthropics/claude-plugins-official)의 큐레이션과 [Superpowers·GSD 조합](https://dev.to/imaginex/a-claude-code-skills-stack-how-to-combine-superpowers-gstack-and-gsd-without-the-chaos-44b3)이 어떻게 다른 답을 내는지다. 한 사람의 의견을 인프라로 박제하는 길과, 의견 없이 빌딩블록만 까는 길 — 어느 쪽이 더 오래 갈까. 지금은 모르겠다.
