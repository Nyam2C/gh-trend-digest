---
title: "코드가 거의 없는 '코드 저장소' — Anthropic 공식 플러그인 디렉토리를 까봤다"
repo: "anthropics/claude-plugins-official"
url: "https://github.com/anthropics/claude-plugins-official"
language: "python"
stars: 20469
date: "2026-05-20"
summary: "GitHub은 주 언어를 Python이라 적어두지만, 까보면 마크다운 194개·JSON 68개가 본체인 큐레이션 디렉토리다. marketplace.json 단 한 파일이 202개 플러그인을 묶고, 그 신뢰는 LLM 보안 스캔과 SHA 핀이 CI에서 떠받친다. '공식인데 통제하지 않는다'는 README 면책이 이 레포의 정직함이자 한계다."
tags: [claude-code, mcp, plugins, anthropic, marketplace, supply-chain-security, ci-cd, llm-security]
---

# 코드가 거의 없는 '코드 저장소' — Anthropic 공식 플러그인 디렉토리를 까봤다

## 도입

트렌딩 목록에서 `anthropics/claude-plugins-official`을 봤을 때 솔직히 처음엔 시큰둥했다. "공식 플러그인 모음집"이라니, 그냥 링크 잔뜩 박아둔 awesome 리스트 같은 거겠지 싶었거든. 그런데 GitHub 언어 배지에 **Python**이라고 적혀 있는 게 좀 의아했다. 플러그인 카탈로그가 왜 Python이지?

클론해서 파일을 세어봤더니 답이 나왔다. 마크다운 194개, JSON 68개, 그리고 Python은 고작 22개(수집 시각 2026-05-20 기준). GitHub의 언어 집계는 바이트 수로 매기니까 Python이 1등으로 찍힌 거지, 이 레포의 본체는 코드가 아니라 **문서와 설정**이다. 별이 약 2만 개(같은 시점) 붙은 이 레포는 "코드 저장소"의 탈을 쓴 큐레이션 디렉토리였다. 이 긴장이 흥미로워서 좀 더 파보기로 했다.

## 정체

한 줄로 줄이면 이렇다. **Claude Code용 플러그인을 Anthropic이 직접 골라 담은 공식 마켓플레이스 디렉토리.** 설명란에도 그대로 적혀 있다 — "Official, Anthropic-managed directory of high quality Claude Code Plugins."

여기서 "플러그인"이 뭔지부터 짚어야 글이 풀린다. Anthropic이 2025-10-09 블로그에서 플러그인 시스템을 퍼블릭 베타로 발표했는데, 그때 정의가 "슬래시 커맨드 + 서브에이전트 + MCP 서버 + 훅(hooks)을 한 번에 묶어 설치하는 가벼운 패키지"였다([claude.com/blog](https://claude.com/blog/claude-code-plugins)). 발표 당시엔 skills가 빠져 있었는데 이후 추가돼서, 지금 README의 표준 구조엔 `skills/` 디렉토리가 들어 있다. 그러니까 플러그인은 커맨드·에이전트·스킬·MCP·훅 중 아무거나 조합해서 한 명령으로 깔 수 있는 묶음이고, 이 레포는 그 묶음들의 **공식 카탈로그**다.

만든 곳은 두말할 것 없이 Anthropic 본진(`anthropics` org)이다. 그런데 이게 갑자기 튀어나온 건 아니다. 플러그인 시스템은 처음부터 "커뮤니티가 만들고 우리가 카탈로그를 잇는다"는 분산 모델로 출발했고, 발표 블로그에서부터 Dan Ávila(DevOps·문서·PM·테스트 마켓플레이스)나 Seth Hobson(서브에이전트 80개+ 큐레이션) 같은 커뮤니티 제작자를 직접 호명했다. 이 공식 디렉토리는 그 위에 얹힌 **큐레이션 레이어**인 셈이다. 규모도 빠르게 컸다 — 2025-12엔 36개였는데(출처: [Pete Gypps](https://www.petegypps.uk/blog/claude-code-official-plugin-marketplace-complete-guide-36-plugins-december-2025)), 2026-02엔 엔터프라이즈용으로 확장됐고([gHacks](https://www.ghacks.net/2026/02/25/anthropic-expands-claude-with-enterprise-plugins-and-marketplace/)), 지금은 매니페스트 기준 202개가 등재돼 있다.

## 아키텍처·구현 디테일

이 레포의 심장은 딱 한 파일이다. `.claude-plugin/marketplace.json`, 2,690줄짜리 단일 매니페스트가 202개 플러그인을 전부 등재한다. 사용자가 `/plugin install {name}@claude-plugins-official`을 치면 Claude Code가 이 파일을 읽어서 해당 엔트리의 `source`를 따라간다. 그게 전부의 시작점이다.

그런데 까보면서 "오, 이거 영리한데" 했던 게 `source` 필드의 다형성이다. 같은 배열 안인데 어떤 건 문자열, 어떤 건 딕셔너리다.

```json
// .claude-plugin/marketplace.json (형태 발췌)
{ "name": "agent-sdk-dev", "source": "./plugins/agent-sdk-dev", ... }
{ "name": "42crunch-...", "source": {
    "source": "git-subdir",
    "url": "https://github.com/42Crunch-AI/claude-plugins.git",
    "path": "plugins/api-security-testing",
    "ref": "v1.5.5",
    "sha": "faf5305385de8afed9468904e8639be737aff39e" } }
```

`source`가 문자열이면 "이 레포 안의 경로", 딕셔너리면 "외부 레포의 특정 커밋"이다. 이 단순한 타입 분기 하나로 "Anthropic이 직접 동봉한 플러그인"과 "남의 레포를 가리키는 포인터"를 한 디렉토리에 매끄럽게 섞는다. 그래서 숫자가 헷갈리는데, 정리하면 이렇다 — 마켓플레이스 등재 **202개**, 그중 레포 안에 실제 파일이 들어 있는 건 **49개**(`./plugins/` 34 + `./external_plugins/` 15), 나머지 **153개는 외부 git을 commit SHA로 핀 고정**해서 참조만 한다. 레포 안에 동봉된 디렉토리는 50개인데 등재가 49개인 차이 1개는 `example-plugin`이다. 레퍼런스용이라 일부러 마켓플레이스에 안 올렸다(분석가가 직접 파싱해서 확인한 사실이다).

그러니까 이 레포는 대부분의 외부 플러그인 코드를 보관하지 않는다. **검증된 주소록**에 가깝다. 그럼 "주소록"이라는 게 어떻게 신뢰를 담보하느냐 — 여기가 진짜 재밌는 부분이다. SHA 핀이 핵심이다. 외부 153개 엔트리 전부(100%)가 `sha` 필드를 갖고 있다. 그래서 upstream 레포가 나중에 브랜치를 force-push로 갈아엎어도, 핀이 가리키는 커밋은 그대로라 사용자가 받는 코드는 변하지 않는다.

그리고 그 핀과 등재 품질을 떠받치는 게 `.github/`의 CI 7종이다. 그중 보안 게이트인 `scan-plugins.yml`이 이 레포에서 기술적으로 가장 정교한 코드다. 변경된 외부 엔트리를 LLM(Claude)으로 보안·프라이버시 심사하는 필수 status check인데, 흥미로운 건 그 위협 모델을 정직하게 코드로 박아뒀다는 점이다.

```yaml
# .github/workflows/scan-plugins.yml:243-252
# Defense in depth: the scan action runs Claude with Read access over
# a cloned external repo and ANTHROPIC_API_KEY in its process env. A
# successful prompt injection could coerce the model to put key
# material into `summary`/`violations`. ... Scrub any key-shaped token
# here so it never reaches the cache, artifact, or comment.
jq -c '(.. | strings) |= gsub("sk-ant-[A-Za-z0-9_-]{8,}"; "[REDACTED]")' \
  "$CACHE_DIR/scanned-raw.json" > "$CACHE_DIR/scanned-raw.json.tmp"
```

심사 대상이 신뢰할 수 없는 외부 코드이고, 심사자가 LLM이라는 점을 그대로 받아들인 거다. "외부 레포가 모델을 프롬프트 인젝션해서 API 키를 verdict 텍스트로 흘릴 수 있다"는 시나리오를 가정하고, `sk-ant-` 패턴을 정규식으로 스크럽한다. 게다가 이 출력이 PR 코멘트라는 공개 sink로 나가니까, 마크다운 제어문자(`| \n \r [ ] < > \``)까지 중화한다(`:325`). LLM을 신뢰 경계 안에 가둬두는 솜씨가 꽤 인상적이었다.

데이터 흐름을 한 바퀴 돌려보면 이렇다. 신규 엔트리는 PR로만 들어오는데(외부 PR은 `close-external-prs.yml`이 자동 종료하고 제출은 폼 전용이다), 머지 전 `scan-plugins`가 LLM 심사를 하고, 머지 후엔 야간에 `bump-plugin-shas.yml`이 외부 SHA를 upstream HEAD로 갱신하면서 같은 스캔을 다시 돌린다. 고정과 신선도를 동시에 잡는 루프다.

## 박수 포인트

까보면서 "이건 잘 만들었다" 싶었던 게 네 개 있다.

**첫째, SHA 핀 + 야간 bump + 재스캔의 3박자.** SHA로 못 박으면 재현성과 보안은 잡히는데 코드가 낡는다. 그래서 `bump-plugin-shas.yml`이 매일 밤 외부 SHA를 upstream HEAD로 올리고, 올린 SHA를 다시 스캔한다. "고정성"과 "신선도"라는 본질적 모순을 봉합하는 설계다. 심지어 외부 공유 액션(`anthropics/claude-plugins-community`)마저 SHA로 핀해서 공급망 자체를 고정해뒀다. 디테일에 진심이다.

**둘째, 멱등 스캔 캐시.** `(plugin, sha)`가 캐시 키이고 SHA는 불변이니까, 정책이 고정인 한 같은 커밋은 딱 한 번만 LLM 심사한다. 야간 force-reset으로 같은 SHA가 매일 diff에 다시 등장해도 ~90초짜리 Claude 시간을 또 태우지 않는다. 게다가 정책 프롬프트의 해시를 캐시 키에 넣어서, 심사 기준이 바뀌면 캐시 전체가 무효화되도록 했다(`:105-107`). 비용을 의식한 똑똑한 캐싱이다.

**셋째, fail-closed 게이트.** 공유 액션은 `ANTHROPIC_API_KEY`가 없으면 조용히 no-op하도록 돼 있는데, 이 레포는 그걸 "required check의 고무도장화"로 보고 키가 없으면 **명시적으로 실패**시킨다(`:79-90`). 인프라 에러도 "전부 통과"로 오독하지 않고 시끄럽게 실패시킨다(`:377-380`). 보안 게이트는 의심스러우면 막아야지 통과시키면 안 된다는 원칙을 코드로 지켰다.

**넷째, "양보다 신뢰"라는 포지셔닝 자체.** smithery(MCP 서버 수천 개), mcp.so(5,000+), Glama(6,000+) 같은 대형 디렉토리가 "양과 발견"으로 경쟁할 때, 이 레포는 정반대로 갔다. 정책 기준선부터 다르다 — `policy/prompt.md`의 통과선이 "악성 아님"이 아니라 "사용자 데이터를 책임 있게 다루는가"다. 미공개 텔레메트리, 훅 범위, 설명과 실제 동작의 일치까지 본다. 무명 MCP가 난립하는 판에서 "검증된 출처 묶음"을 내세운 건 영리한 차별화다.

## 비판·한계

다만 박수만 칠 글은 아니다. 까보면 구조적으로 아쉬운 지점이 분명히 있다.

가장 먼저 걸린 건 **SHA 핀이 생각만큼 안전을 보장하지 못한다**는 점이다. 외부 플러그인 15개를 보면 전부 MCP 연동인데, 그 MCP 서버가 거의 다 원격 HTTP/SSE 엔드포인트다(`github`→`api.githubcopilot.com`, `linear`→`mcp.linear.app`). 그러니까 SHA가 고정하는 건 `.mcp.json`이라는 **URL 선언**일 뿐, 그 URL 너머 서버가 무슨 짓을 하는지는 SHA로 못 묶는다. 핀은 클라이언트 설정의 재현성만 보장하고 서버 측 거동은 보장하지 못하는 구조적 한계가 있다. SHA 핀의 멋진 서사가 원격 MCP 앞에서는 절반만 통하는 셈이다.

보안 게이트가 LLM이라는 점도 양날이다. 인젝션 방어는 훌륭하지만, 결국 통과/탈락의 최종 판단이 모델의 `passes` boolean에 달려 있다. 거짓 음성(놓친 악성)과 거짓 양성(과차단)의 여지가 남는다. 이게 추상적 우려가 아닌 게, 외부 맥락이 무겁다. Snyk의 ToxicSkills 감사(2026-02)에선 스캔한 3,984개 스킬 중 1,467개에서 악성 페이로드가 나와 결함률 36%였다(출처: [SecurityWeek/Snyk 보도](https://www.securityweek.com/claude-code-oauth-tokens-can-be-stolen-through-stealthy-mcp-hijacking/) — 수치 1차 출처 재확인 권장). CVE-2025-59536, CVE-2026-21852처럼 신뢰 안 되는 리포를 여는 것만으로 RCE·API 키 탈취가 트리거되는 사건도 보고됐다([MintMCP](https://www.mintmcp.com/blog/claude-code-supply-chain-attacks)). 이런 판에서 LLM 한 명에게 게이트키퍼를 맡기는 건 잘했어도 완벽할 수는 없다.

그리고 인프라는 프로덕션급인데 **정작 동봉 플러그인 코드의 테스트는 사실상 0**이다. `*test*` 매칭 파일은 죄다 문서거나(`test-engineer.md` 같은) 훅 테스트 도구다. hookify의 진짜 규칙 엔진(`plugins/hookify/core/rule_engine.py`)에도 단위 테스트 스위트는 없고 `:277-313`에 인라인 스모크 테스트만 있다. 엔지니어링 밀도가 게이트 인프라에 쏠려 있고 플러그인 본문엔 편차가 크다.

라이선스도 좀 애매하다. 루트에 LICENSE 파일이 없고(GitHub API `license: null`), README는 "각 플러그인의 LICENSE를 참조하라"고 위임한다. 동봉 플러그인엔 LICENSE가 있지만, 153개 외부 핀의 라이선스 일관성은 레포 차원에서 강제되지 않는다. 매니페스트 책임도 비대칭이다 — LSP 13종은 `plugin.json` 없이 marketplace 엔트리가 `lspServers` 같은 런타임 설정까지 떠안아서, 플러그인 종류에 따라 진실 공급원의 위치가 갈린다.

마지막으로 이 모든 한계를 README가 최상단에 솔직하게 못 박는다.

> "⚠️ Important: Make sure you trust a plugin before installing... Anthropic does not control what MCP servers, files, or other software are included in plugins and cannot verify that they will work as intended or that they won't change."

"공식"인데 "통제하지 않는다"는 이 역설이 핵심이다. 큐레이션은 **입점 심사이지 런타임 안전 보증이 아니다.** 신뢰 책임은 결국 사용자에게 남는다. 참고로 플러그인 시스템 전반에 대한 비판도 있는데, HN에는 "GitHub Actions의 복잡성을 답습했다"는 취지의 강한 비판 스레드 제목도 있었다([HN 46363490](https://news.ycombinator.com/item?id=46363490) — 본문은 미확인). 다만 이건 시스템 전반 얘기지 이 디렉토리를 단독으로 겨눈 화제는 확인하지 못했으니, 곧이곧대로 옮기진 않겠다.

## 결론

**누구에게 가치 있나.** 팀에 Claude Code를 도입하면서 "일단 안전한 시작점부터" 찾는 사람한테는 확실히 유용하다. 마켓플레이스 설정을 git에 커밋해두면 팀원에게 자동으로 깔린다는 온보딩 효용도 크다([Dale Seo](https://daleseo.com/claude-code-plugin-marketplaces/)). MCP·CI·LLM 보안 게이트를 어떻게 엮는지 공부하려는 엔지니어에게도 `scan-plugins.yml`은 그 자체로 좋은 교본이다.

**누가 피해야 하나.** "공식 = 안전 보증"이라고 믿고 검토 없이 깔려는 사람은 README 면책을 다시 읽어야 한다. 그리고 압도적인 플러그인 양이 필요한 사람한테는 202개가 부족할 수 있다 — 그건 smithery 같은 대형 디렉토리의 영역이다. 실사용 리뷰 중엔 "10개 깔아보니 4개만 쓸 만하더라"는 평가도 있었으니([Build to Launch](https://buildtolaunch.substack.com/p/best-claude-code-plugins-tested-review)), 등재됐다고 다 같은 품질은 아니다.

까보고 나서 든 솔직한 생각은, 이 레포의 진짜 값어치는 플러그인 목록이 아니라 **목록을 신뢰하게 만드는 인프라**에 있다는 거다. SHA 핀과 fail-closed LLM 게이트를 보면 "공급망 신뢰를 코드로 강제하면 이렇게 한다"는 한 가지 답안이 보인다. 다음엔 `bump-plugin-shas`와 `revert-failed-bumps`가 실제로 야간에 어떻게 맞물려 돌아가는지, 실패 격리 로직을 더 파보고 싶어졌다. 별 2만 개가 괜히 붙은 건 아니었다.
