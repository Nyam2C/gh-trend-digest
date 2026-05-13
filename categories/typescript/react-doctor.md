---
repo: react-doctor
full_name: millionco/react-doctor
url: https://github.com/millionco/react-doctor
language: typescript
stars: 8926
date: 2026-05-13
one_line: "AI 에이전트가 짠 엉성한 React 코드를 한 줄로 진단하고 0~100 헬스 스코어로 채점하는, oxlint + ESLint + knip을 하나의 정전 룰셋으로 묶은 React 전용 코드 진단기."
---

# 16살에 Million.js 만든 애가 이번엔 "AI가 짠 React 코드 잡는 의사"를 풀었다

## 도입

X 타임라인에서 며칠째 같은 카피가 떠다녔다. "Your agent writes bad React, this catches it." 누가 이런 도발적인 문장을 또 던지나 봤더니 Aiden Bai였다. Million.js 만든 그 친구. 솔직히 처음엔 또 React 영역에서 마케팅 잘 뽑은 거구나 정도로 흘려봤다. 그런데 별이 3개월 만에 8,900개를 넘기는 걸 보고 — 게다가 Aiden 본인이 "fuck it i'm gonna cleanup every react codebase with React Doctor starting with Cal.com"이라는 [트윗](https://x.com/aidenybai/status/2024193672603193487)을 박았기에 — 진짜 코드를 열어볼 수밖에 없었다.

열어보고 나니 단순한 카피 승리가 아니었다. AI 에이전트 시대의 React 품질이라는 새 카테고리를 정조준한 도구인데, 정조준 방식이 묘하게 정직하다. 그래서 글이 길어졌다.

## 정체

한 줄로 요약하면 이렇다. `npx -y react-doctor@latest .` 하면 React 코드베이스를 스캔해서 0~100점 헬스 스코어와 카테고리별 진단을 뱉어낸다. 75점 이상이면 Great, 50~74면 Needs work, 그 밑은 Critical. 룰 카테고리는 state & effects, performance, architecture, security, accessibility, dead code 여섯 묶음. 엔진은 Rust로 짠 oxlint, dead code 검사는 knip, 거기에 자체 정전 룰셋을 얹었다.

그런데 "React 린터"라는 표현은 좀 좁다. README가 강조하는 건 두 가지 워크플로다. 하나는 방금 말한 코드 진단. 다른 하나가 핵심인데, `react-doctor install` 한 줄이면 Claude Code·Cursor·Codex·OpenCode·Gemini 같은 50+ 코딩 에이전트에 SKILL.md / AGENTS.md / .cursorrules를 동시에 설치해서 에이전트가 **처음부터** 좋은 React를 쓰게 만든다. 사전 학습 + 사후 검사 이중 게이트. 이 발상이 진짜 이 도구의 정체다.

만든 사람도 흥미롭다. Aiden Bai는 16세에 [Million.js](https://github.com/aidenybai/million)를 시작했고, YC W24 배치로 Million Software, Inc.를 창업했다. [YC 페이지](https://www.ycombinator.com/companies/million-js)의 회사 한 줄 소개가 "Copilot for optimizing web performance"인데, 라인업이 묘하다. Million.js(2023, React를 70% 가속) → Million Lint(2024, VSCode 익스텐션) → [React Scan](https://github.com/aidenybai/react-scan)(2025, 런타임 리렌더 시각화) → 그리고 2026년의 React Doctor와 [claude-doctor](https://github.com/millionco/claude-doctor). "React 성능"이라는 좁은 도메인에서 출발해 "AI 에이전트가 만든 모든 프론트엔드 산출물의 품질 인프라"로 사세가 확장되는 중이다. react-doctor의 카피가 이 전환점을 가장 또렷이 보여준다.

[midudev](https://x.com/midudev/status/2024135933474296228)도, [Camille Roux](https://x.com/CamilleRoux/status/2024395870838354229)도 같은 주에 같은 톤으로 받았다. 유럽권 인플루언서 동시 다발 트윗. 마케팅이라기보단 "이 인물이 또 뭔가 냈다"는 시그널에 가까웠다.

## 아키텍처·구현 디테일

이 도구의 진짜 핵심은 **하나의 룰 본체를 여러 런타임에 동시에 노출하는** 패턴이다. `packages/react-doctor/src/oxlint-config.ts`(719줄)가 README가 "the full rule list lives in"이라고 가리키는 정전(canonical config)이고, 그 위에 `packages/react-doctor/src/plugin/rules/` 디렉토리의 16개 카테고리 룰 파일이 약 1만 줄의 AST 비지터로 박혀 있다. 그 다음, 같은 룰 본체가 oxlint(Rust 엔진)에도 ESLint(JS 엔진)에도 그대로 들어간다.

비결은 어댑터다. `packages/react-doctor/src/eslint-plugin.ts:59-70`을 보자.

```ts
const wrapAsEslintRule = (ruleName: string, ruleImpl: PluginRule): EslintRule => ({
  meta: {
    type: "problem",
    docs: {
      description: ruleNameToDescription(ruleName),
      url: `${RULE_DOCS_BASE_URL}/${ruleName}`,
      recommended: recommendedRuleKeys.has(`${PLUGIN_NAMESPACE}/${ruleName}`),
    },
    schema: [],
  },
  create: (context: EslintRuleContext) => ruleImpl.create(context),
});
```

이게 끝이다. oxlint용 `{ create }` 룰에 ESLint가 요구하는 `meta` 껍데기만 씌워서 그대로 내보낸다. 어댑터 본체가 117줄. 이게 가능한 건 두 런타임의 공통 분모 — visitor-callback 셰이프 — 에 맞춰 룰 인터페이스를 처음부터 설계한 덕분이다. `plugin/types.ts`가 23줄짜리 단일 파일에 두 런타임 공통 인터페이스를 정의해두었다. 한 번 짠 룰을 두 시스템에 공짜로 노출하는 구조.

데이터 흐름은 단순하다. `cli.ts`(743줄)가 commander로 플래그를 파싱하고 7개 모드(`--diff`/`--staged`/`--full`/`--explain`/`--json`/`--score`/`--annotations`)의 상호배타를 검증한다. 그다음 `loadConfig` → `discoverProject`(React 버전·프레임워크 자동 감지) → `runOxlint`와 `runKnip`을 `Promise.allSettled`로 병렬 실행 → 진단 결과 합치고 억제(suppression) 적용 → 스코어 계산 → `scan.ts`가 카테고리 그룹·점수바·share URL을 사람-가독 출력.

여기서 capability 게이팅이 또 재밌다. `packages/react-doctor/src/oxlint-config.ts:594-643`:

```ts
export const buildCapabilities = (project: ProjectInfo): ReadonlySet<string> => {
  const capabilities = new Set<string>();
  capabilities.add(project.framework);
  // HACK: when version detection fails (null), assume the latest React
  // major so every version-gated rule fires. Silently dropping rules
  // on detection failure was the worse outcome in practice.
  const reactMajor = project.reactMajorVersion;
  const effectiveReactMajor = reactMajor ?? 99;
  for (let major = 17; major <= effectiveReactMajor; major++) {
    capabilities.add(`react:${major}`);
  }
  // ...
};
```

각 룰에 `requires`(필요 capability 집합)와 `tags`(블랙리스트)를 붙여 두고, 프로젝트의 React 버전·Tailwind 버전·프레임워크를 capability 집합으로 환원해 매칭한다. React 17 코드베이스에서 `no-react19-deprecated-apis`가 안 뜨고, Tailwind 3.4 미만에서 `size-N` 단축형 권유가 침묵한다. 그런데 — 이 부분이 진짜 흥미로운데 — 버전 감지 실패 시 99로 폴백한다. "조용히 룰 누락"보다 "혹시 모르니 다 켜기"를 택한 것. HACK 주석에 그 정당화까지 박혀 있다. 이런 선택의 흔적이 코드베이스 전체에 일관된다.

## 박수 포인트

### 1. 솔직함의 컨벤션화

`AGENTS.md:7-8`에 이런 줄이 있다.

```
- MUST: Never comment unless absolutely necessary.
  - If the code is a hack (like a setTimeout or potentially confusing code),
    it must be prefixed with // HACK: reason for hack
```

"주석은 절대 쓰지 마라, 다만 해킹은 강제로 표시해라." 이 정책에 맞춰 코드베이스에 `// HACK:` 주석이 196개 박혀 있다. TODO·FIXME·XXX는 0개. 미완성 흔적이 없고, 대신 "왜 못생겼지만 이 모양인지" 솔직한 시인이 일관된다. 가장 인상적인 건 `action.yml:91-102`다.

```yaml
- id: score
  if: always()
  shell: bash
  run: |
    # HACK: --score is an output-collection step, not a gate. Force
    # --fail-on none so older react-doctor releases (which exit
    # non-zero when the score lands in the "Needs work" band, even
    # though the value itself is the only meaningful signal here)
    # don't fail the composite action under `set -e -o pipefail`.
    SCORE_ARGS=("$INPUT_DIRECTORY" "--score" "--fail-on" "none")
```

GitHub Action에서 같은 명령을 두 번 호출한다. 한 번은 실제 스캔, 한 번은 점수만 뽑으려고. 이 비효율을 숨기지 않고, 왜 임시방편이 영구가 됐는지(과거 버전 호환 + composite action의 `set -e` 함정)까지 댓글로 박아뒀다. 일반적인 "HACK"이 아니라 자기 시인이다. 다음 사람(또는 다음 에이전트)이 "왜 이거 안 고치지?" 했을 때 답이 코드 안에 있다.

### 2. 점수 공식의 비대칭 인센티브

`packages/react-doctor/src/utils/calculate-score-locally.ts:26-47`을 보자.

```ts
const collectUniqueRuleSets = (diagnostics: Diagnostic[]) => {
  const errorRules = new Set<string>();
  const warningRules = new Set<string>();
  for (const diagnostic of diagnostics) {
    const ruleKey = `${diagnostic.plugin}/${diagnostic.rule}`;
    if (diagnostic.severity === "error") errorRules.add(ruleKey);
    else warningRules.add(ruleKey);
  }
  return { errorRules, warningRules };
};

const scoreFromRuleCounts = (errorRuleCount: number, warningRuleCount: number): number => {
  const penalty = errorRuleCount * ERROR_RULE_PENALTY + warningRuleCount * WARNING_RULE_PENALTY;
  return Math.max(0, Math.round(PERFECT_SCORE - penalty));
};
```

100점 만점에서 error 룰 하나당 1.5점, warning 룰 하나당 0.75점을 깐다. 그런데 페널티 기준이 **인스턴스 수**가 아니라 **고유 룰 수**다. 같은 룰에서 100건이 터지든 1건이 터지든 페널티는 1.5점으로 동일. 결과적으로 "한 룰을 49개만 고치면 점수가 안 변함, 50개째에 비로소 페널티 제거"라는 비대칭이 생긴다.

처음 보면 비직관적이다. 그런데 인센티브 설계로 보면 무릎을 친다. 부분 수정으로 점수만 올리는 짓을 막고, 한 룰을 잡으면 전수로 잡게 만든다. 동시에 false positive 한 건의 영향력을 막는다(한 건만 떠도 1.5점, 백 건이 떠도 1.5점이라 노이즈에 점수가 무너지지 않음). 이건 단순한 점수 함수가 아니라 행동 디자인이다.

### 3. 억제(suppression) 시스템의 자가 진단

`packages/react-doctor/src/utils/evaluate-suppression.ts:39-58`이 압권이다.

```ts
const buildGapHint = (comment, diagnosticLineIndex, ruleId): string => {
  const gapLineCount = diagnosticLineNumber - commentLineNumber - 1;
  return (
    `A react-doctor-disable-next-line for ${ruleId} sits at line ${commentLineNumber}, but ${formatLineGap(gapLineCount)} of code separate it from the diagnostic on line ${diagnosticLineNumber}. ` +
    `Move the comment immediately above line ${diagnosticLineNumber}, or extract the surrounding code into a helper so the suppression is adjacent.`
  );
};
```

`// react-doctor-disable-next-line` 주석을 달았는데 룰이 안 꺼졌을 때, 이유를 정확히 짚어준다. 룰 이름이 리스트에 없으면 "콤마 형식으로 추가해라", 주석과 진단 사이에 코드가 끼면 "주석을 진단 바로 위로 옮기거나 surrounding code를 헬퍼로 빼라"고 안내한다. `--explain file:line` 플래그가 정확히 이 텍스트를 출력한다. 사용자가 "왜 disable이 안 먹지?" 라고 짜증낼 자리를 도구가 먼저 채우고 있다. issue #159 같은 PR 번호가 코드 안의 HACK 주석에 박혀 있는데, 사용자 피드백이 코드 디테일로 환원되는 사이클이 작동 중이라는 흔적이다.

### 4. OSS와 SaaS 사이를 깔끔하게 가른 듀얼 스코어 백엔드

`packages/react-doctor/src/utils/calculate-score.ts:8-9`는 한 줄이다.

```ts
export const calculateScore = async (diagnostics: Diagnostic[]): Promise<ScoreResult | null> =>
  (await tryScoreFromApi(diagnostics, proxyFetch)) ?? calculateScoreLocally(diagnostics);
```

원격 API 우선, 실패 시 로컬 폴백. 10초 타임아웃, CI 환경 자동 감지, `--offline` 플래그까지. 그리고 가장 마음에 든 디테일은 `try-score-from-api.ts:27-57`의 `stripFilePaths` — POST 본문에서 파일 경로를 제거하고 룰명·심각도·라인 번호만 보낸다. README의 "anonymous, not stored, only used to calculate score" 약속을 코드로 강제한 셈. 리더보드·share URL을 운영하면서도 SaaS 의존을 강요하지 않는 절묘한 경계다.

## 비판·한계

박수 잘 쳐줬으니 이번엔 솔직하게.

### 1. 130+ 룰 큐레이션의 권한 집중

`oxlint-config.ts`의 정전 ruleset이 너무 한 곳에 집중돼 있다. 룰 하나의 false-positive가 1.5 또는 0.75점을 깎는데, 누가 어떤 기준으로 룰을 추가·제거하는지 거버넌스가 안 보인다. shortlog가 Aiden Bai 42 / Cursor Agent 4 / Nisarg Patel 2 / github-actions 2니까 사실상 한 명 + AI 에이전트 보조 패턴이다. 게다가 README의 "design" 카테고리에 `no-gradient-text`, `no-dark-mode-glow`, `no-justified-text` 같이 명백히 취향인 룰이 있다. `tags: DESIGN_AND_TEST_NOISE_TAGS`로 분류해 사용자가 끌 수 있게 한 건 좋지만 디폴트로 켜져 있어 첫 인상을 좌우한다.

### 2. "잘 구성된 ESLint 대체제 아님"이라는 자기 인정

이건 도구 자체의 README도, 외부 리뷰도 한목소리로 인정한다. [Better Stack 가이드](https://betterstack.com/community/guides/scaling-nodejs/react-doctor/)는 "이미 엄격한 ESLint 설정이 있다면 일정 부분 중복은 불가피"라고 명시. 흡수형 통합(react-hooks v6/v7, you-might-not-need-an-effect를 optional peer로 흡수)이 우아하긴 한데, 결국 기존 lint 스택 위에 또 하나의 룰 레이어를 얹는다. 엔터프라이즈 채택 시 lockfile 영향(`react-doctor` 하나 깔면 `oxlint ^1.63`, `knip ^6.10`이 자동으로 따라옴)도 따져야 한다.

### 3. False positive와 점수 정밀도

가장 잘 정리된 실전 감사 사례가 [Pink Lemon8의 Next.js 16 감사 블로그](https://pinklemon8.com/blog/react-doctor-audit-nextjs)인데, 거기서 짚은 게 두 가지다. motion 라이브러리를 `package.json` 의존성으로만 감지해 CSS `prefers-reduced-motion`이나 `useReducedMotion` 훅을 못 잡고, dead-code 분석이 함수 반환 타입으로만 쓰이는 타입을 미사용으로 오인. [fireup.pro 리뷰](https://fireup.pro/news/react-doctor-cli-auditing-for-react-projects)와 [DEV.to 글](https://dev.to/arshtechpro/react-doctor-is-this-the-missing-health-check-for-your-react-codebase-5015)이 공통적으로 "72 vs 75는 의미 거의 없다, 숫자가 아니라 개별 진단을 봐라"라고 못 박는다. tldraw 70, excalidraw 63, payload 60 — 베테랑 팀들도 60~70점대니까, 만점은 비현실적이고 의도된 안티패턴(JSON-LD용 `dangerouslySetInnerHTML` 같은)도 경고로 잡힌다.

### 4. AST 룰 본체의 단위 테스트 분포

테스트는 44개 파일에 5,266줄, 본체 약 1만 줄 대비 38% 비율로 나쁘진 않다. 그런데 1,014줄짜리 `performance.ts`나 2,592줄짜리 `state-and-effects.ts` 룰 본체에 대한 직접 단위 테스트가 파일명 기준으론 안 보인다. 회귀 픽스처에 의존하는 모양인데, 룰의 false positive·negative 회귀 검출이 늦을 수 있다. 룰의 정확성은 결국 issue/PR 피드백 루프에 맡긴 셈 — 그래서 HACK 주석에 issue 번호가 박혀 있는 거다.

### 5. `install` 명령의 강력함은 곧 위험함

50+ 에이전트의 설정 디렉토리를 자동 감지해서 SKILL.md를 심는 동작은 강력한데, 진짜 본체는 외부 npm `agent-install@0.0.5`에 있다. react-doctor 코드만 봐선 기존 SKILL.md가 있을 때 병합하는지 덮어쓰는지를 검증할 수 없다. `--dry-run`은 있지만 디폴트 동작이 `mode: "copy"`라 한 번에 여러 에이전트의 룰 파일을 갈아치울 수 있다. 처음 쓰는 사람은 꼭 `--dry-run`부터 돌려보자.

### 6. 에이전트가 룰을 끄는 방향으로 학습할 가능성

이건 좀 메타한 비판인데, 도구가 잘 될수록 두드러질 문제다. SKILL.md가 "점수 떨어지면 고쳐라"고 명령하는 순간, 에이전트의 단기 최적해는 룰을 disable하는 것일 수 있다. `// react-doctor-disable-next-line` 주석이 코드베이스에 늘어나는 패턴을 어떻게 막을지에 대한 답이 도구 자체엔 아직 안 보인다. 사람이 PR 리뷰에서 잡아야 하는데, 그러면 또 도구의 가치가 줄어든다. 정직히 말해 이건 react-doctor의 결함이라기보단 AI 에이전트 시대의 구조적 문제이긴 하다.

## 결론

이 도구가 잘 맞는 사람.

- 에이전트(Claude Code, Cursor, Codex)로 React 코드를 대량 생성하는 팀. `install`로 사전 학습, PR에 점수 다는 사후 검사, 두 게이트가 다 작동한다.
- ESLint 설정조차 안 잡힌 레거시·외주 React 코드베이스를 막 인수한 사람. 1분 만에 약점 지도가 그려진다.
- 모노레포에서 워크스페이스 단위 PR 게이팅이 필요한 팀. `--project`, `--diff`, `--staged`가 받쳐줘서 변경된 부분만 강제 가능.

피해야 할 사람.

- 이미 ESLint·Biome·jsx-a11y를 빡빡하게 깔아둔 팀. 중복이 크고, 헬스 스코어 자체가 KPI가 되지 않으면 도입 가치가 줄어든다.
- 코드를 외부 API에 보내는 게 정책적으로 안 되는 환경. `--offline` 강제 정책을 운영해야 하는데, 그러면 share URL·리더보드 같은 흡인 포인트가 다 빠진다.
- 정밀한 false positive 컨트롤이 필요한 곳. 1.5점·0.75점 페널티가 가벼워 보여도 룰 큐레이션 권한이 한 명에게 집중돼 있다는 점은 기억해두자.

프로덕션 도구로 쓸만한가는 미묘하다. 코드 자체는 명백히 프로덕션 수준이다. SIGABRT·SIGTERM·SIGINT·EPIPE 다 처리하고, Windows 명령줄 길이 한도 회피하고, 자식 프로세스 env까지 살균한다. 그런데 v0.1.6, 출시 3개월차다. 룰셋은 빠르게 진화 중이고 며칠 만에 6개 마이너 버전이 찍힌 고속 이터레이션이라 PR 게이트로 박는다면 버전 고정은 필수다.

마지막으로 — 한국어권에서 이걸 진지하게 다룬 글이 거의 안 보였다. velog, 디스코드, 국내 블로그를 뒤져도 별 내용이 없었으니까. 그래서 한국어 사용자가 첫 도입을 검토할 때 이 글이 출발점이 됐으면 한다. 나는 다음 사이드 프로젝트 React 17 코드베이스에 한번 돌려볼 생각이다. 점수가 두 자릿수일 게 뻔하지만 그게 정확히 이 도구가 원하는 시작점이니까. Aiden이 또 뭘 내는지는 계속 지켜봐야겠다 — Million.js에서 React Doctor까지, "느리고 잘못 쓴 React를 자동으로 잡는다"는 한 줄에 4년째 같은 인물이 같은 답으로 수렴하고 있다는 게 무섭다.
