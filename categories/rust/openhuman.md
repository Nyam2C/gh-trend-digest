---
title: "'당신을 day-1에 아는' AI 비서 — tinyhumansai/openhuman, 코드를 까보니 '프라이빗'에 단서가 붙었다"
repo: "tinyhumansai/openhuman"
url: "https://github.com/tinyhumansai/openhuman"
language: "rust"
stars: 22726
date: "2026-05-20"
summary: "OAuth로 메일·캘린더·메신저를 한 번에 묶어 '메모리 트리'로 압축, 콜드 스타트 없이 당신을 안다고 주장하는 데스크톱 AI 비서. 생성 3개월 만에 별 2.2만. 메모리 트리와 토큰 압축은 코드에 실제로 있었지만, 기본값이 클라우드 요약이라 '로컬 우선/프라이빗' 카피에는 큰 단서가 붙는다."
tags: [rust, tauri, cef, ai-agent, agent-harness, memory-tree, tokenjuice, oauth, privacy, gpl-3, productivity]
---

# '당신을 day-1에 아는' AI 비서 — tinyhumansai/openhuman, 코드를 까보니 '프라이빗'에 단서가 붙었다

## 도입

며칠 전 GitHub 트렌딩 상단에 `tinyhumansai/openhuman`이 며칠째 박혀 있는 걸 봤다. 카피가 좀 셌다. "당신의 개인 AI 초지능. Private, Simple, extremely powerful." 솔직히 첫인상은 "또 거대한 약속 하나 추가됐네"였다. 그런데 설명을 한 줄 더 읽고 손이 멈췄다.

대부분의 AI 에이전트는 빈손으로 시작한다. 새 채팅 창을 열면 걔는 나를 모른다. 매번 컨텍스트를 떠먹여야 한다. OpenHuman이 내세우는 후크는 정확히 그 지점이다 — 계정만 연결하면 20분 주기로 메일·캘린더·메신저·문서를 로컬로 끌어와 "메모리 트리"로 압축해두고, 첫 동기화만으로 나를 파악한다는 거다. 이른바 "day-1 컨텍스트". 생성된 지 3개월밖에 안 된 레포가 별 2.2만 개를 받은 데는 이 한 문장의 힘이 컸을 거다. 그런데 이런 카피일수록 코드를 직접 까봐야 한다. 정말 로컬에 남는지, "80% 절감"이 진짜인지. 그래서 까봤다.

## 정체

먼저 오해 하나를 풀고 가자. 이름이 `openhuman`이고 데스크톱에 마스코트 "얼굴"이 떠서 말도 하고 Google Meet에 실제 참가자로 들어가기까지 하니까, HeyGen이나 D-ID 같은 디지털 휴먼/AI 아바타로 착각하기 쉽다. 아니다. 마스코트는 부가 UX일 뿐이고, 본질은 **개인 데이터를 통합해 기억하는 로컬-퍼스트 AI 비서**, 그러니까 요즘 말로 "에이전트 하네스"다. 비교 대상도 영상 아바타가 아니라 OpenClaw, Hermes Agent, Claude Cowork 같은 에이전트들이다.

만든 곳은 **TinyHumans**라는 AI 랩이고, 자기들을 "인공 의식에 다가가는 AI를 만드는 곳"이라 소개한다. README 하단엔 대놓고 "Building toward AGI and artificial consciousness?"라는 문구까지 박혀 있다. 창업자는 **Steven Enamakel(@senamakel)**, deep-tech/web3/AI 엔지니어로 소개된다. 흥미로운 건 타임라인이다. 레포 자체는 2026년 2월 18일에 생겼는데, 창업자가 4월 23일에 직접 올린 "Show HN: OpenHuman, an AI agent with a subconscious loop"는 [2 points에 댓글 0개](https://news.ycombinator.com/item?id=47876182), 사실상 무반응이었다. 그러다 5월 12~13일 정식 런칭과 함께 트렌딩에 올랐고, [Product Hunt에선 #1 Product of the Day](https://www.producthunt.com/products/openhuman)(488 upvotes)를 찍었다. 별 2.2만은 진짜다. 다만 이 화제성이 어디서 왔는지는 비판 섹션에서 다시 다루겠다.

라이선스는 **GPLv3**, 기술 스택은 Rust 코어 + React/Tauri 데스크톱(macOS/Windows/Linux). 모바일은 없다. 정리하면, 마케팅 카피의 "초지능"을 한 꺼풀 벗기면 실체는 "OAuth로 내 일상 데이터를 통째로 빨아들여 로컬에 기억으로 쌓는 데스크톱 에이전트"다. 야심은 거대하지만 카테고리 위치는 꽤 명확하다.

## 아키텍처·구현 디테일

코드를 열어보면 구조는 의외로 깔끔하게 갈린다. 데스크톱 셸은 `app/src-tauri`(크레이트 이름 `OpenHuman`)이고, 비즈니스 로직의 권위는 전부 루트 `src/`의 Rust 코어(`openhuman`)에 있다. 예전엔 코어를 사이드카 프로세스로 띄웠는데 지금은 제거됐고, 셸이 코어를 in-process tokio 태스크로 직접 spawn한다. UI(React 19)는 이 코어에 오직 RPC로만 말을 건다.

여기서 좀 놀란 부분. 이 데스크톱 셸이 stock Tauri가 아니다. 보통 Tauri는 가볍다는 게 셀링포인트인데, OpenHuman은 **CEF(Chromium Embedded Framework)를 벤더링**해서 쓴다. 왜? 메신저 데이터를 긁는 네이티브 스캐너(WhatsApp/Telegram/Slack/Discord/iMessage 등)가 임베드된 webview의 DOM·IndexedDB를 CDP(원격 디버깅 포트 19222)로 긁어야 하는데, 기본 Tauri 엔진(wry)으로는 그게 안 되기 때문이다(`whatsapp_scanner/mod.rs`). "가벼운 Tauri"라는 통념과 정반대로, 빌드 복잡도를 감수하고 Chromium을 통째로 안고 가는 선택이다.

데이터 흐름을 한 줄로 요약하면 이렇다. 스캐너가 webview를 긁어서 `memory_doc_ingest` JSON-RPC로 코어에 던지면, 코어가 **canonicalize → chunk → 빠른 점수화 → SQLite/디스크 저장 → 추출 잡 enqueue** 순으로 처리한다. 엔티티 추출, 임베딩, 트리 요약 같은 무거운 일은 SQLite 백드 잡 큐에서 백그라운드로 돈다. 핫 패스의 심장은 `src/openhuman/memory/tree/ingest.rs`다.

그 ingest 파이프라인에서 인상적이었던 게 청크 ID를 만드는 방식이다.

```rust
// src/openhuman/memory/tree/types.rs:256-277
pub fn chunk_id(source_kind: SourceKind, source_id: &str, seq_in_source: u32, content: &str) -> String {
    let mut hasher = Sha256::new();
    hasher.update(source_kind.as_str().as_bytes());
    hasher.update([0u8]);
    hasher.update(source_id.as_bytes());
    hasher.update([0u8]);
    hasher.update(seq_in_source.to_be_bytes());
    hasher.update([0u8]);
    hasher.update(content.as_bytes()); // ← content를 해시에 넣은 한 줄이 핵심
    let digest = hasher.finalize();
    /* hex[..32] */
}
```

별것 아닌 ID 생성 함수처럼 보이는데, 마지막 `content`를 해시에 넣은 한 줄이 까다로운 두 가지를 동시에 푼다. Slack처럼 스트림형 소스는 6시간 버킷마다 `seq`가 0부터 다시 시작하는데, content를 안 넣으면 서로 다른 배치의 `seq=0`이 충돌한다. content를 넣으면 충돌이 안 나면서, 동시에 같은 내용을 다시 ingest하면 같은 ID가 나와 upsert 멱등성까지 공짜로 얻는다. 분산 ingest의 코너케이스를 32자 해시 하나로 흡수한 거다. 이런 설계 의도가 주석에 또박또박 적혀 있다는 게 더 인상적이었다.

## 박수 포인트

직접 뜯어보고 "이건 진짜 잘 짰다" 싶었던 게 몇 군데 있다.

**첫째, "트리"를 만드는 버킷-실(bucket-seal) 캐스케이드.** 메모리 트리의 심장은 `bucket_seal.rs`인데, 청크(L0 리프)를 버퍼에 쌓다가 임계치를 넘으면 요약(L1)으로 봉인(seal)하고, 그게 또 쌓이면 상위 레벨로 캐스케이드하며 계층 트리를 만든다. 여기서 `should_seal`을 보자.

```rust
// src/openhuman/memory/tree/tree_source/bucket_seal.rs:326-335
pub(crate) fn should_seal(buf: &Buffer) -> bool {
    if buf.is_empty() { return false; }
    if buf.level == 0 {
        // 토큰 예산 OR 형제 수 — 작은 청크 수백 개가 50k 못 채우고 멈추는 것 방지
        buf.token_sum >= INPUT_TOKEN_BUDGET as i64 || (buf.item_ids.len() as u32) >= SUMMARY_FANOUT
    } else {
        // 상위 레벨은 형제 수만 — 약한 요약기에서 1:1:1 체인 붕괴 방지
        (buf.item_ids.len() as u32) >= SUMMARY_FANOUT
    }
}
```

"왜 이렇게 짰을까"의 답이 코드에 그대로 있다. 토큰 기준만 쓰면 요약기가 출력 예산을 꽉 채울 때 트리가 1:1:1 사슬로 무너진다 — 이 통찰을 레벨별로 게이트를 분기하는 8줄로 막았다. 게다가 봉인 과정에서 느린 LLM 호출과 임베딩은 DB 트랜잭션 밖에서 돌리고(LLM이 DB 락을 잡지 않게), 정작 summary insert + buffer clear + 후속 잡 enqueue는 단일 트랜잭션으로 원자 처리한다. "봉인은 커밋됐는데 후속 잡 enqueue가 크래시로 유실되는 윈도우"를 트랜잭션 안에서 enqueue함으로써 없앤 거다. 분산 파이프라인 짜본 사람이라면 이게 얼마나 자주 놓치는 함정인지 안다.

**둘째, 요약기의 soft-fallback 계약.** `LlmSummariser`는 전송/HTTP/JSON 어떤 실패가 나도 절대 `Err`를 던지지 않는다(`summariser/llm.rs:16-23`). 실패하면 결정적 동작(inert)으로 폴백한다. 외부 LLM이 죽어도 봉인 캐스케이드 전체가 멈추지 않게 만든 우아한 degradation이다. 영리하다.

**셋째, 신생 레포답지 않은 거버넌스.** 이게 3개월 된 레포라는 게 안 믿긴다. Rust 테스트 파일 213개, TS 테스트 291개, 변경 라인 80% 이상 커버리지를 강제하는 머지 게이트(`coverage.yml`). 모듈마다 `//!` 독에 "왜"가 적혀 있고 디렉토리마다 README가 있다. TODO/FIXME/HACK가 `src/` 전체에서 6건뿐이라는 것도 코드 위생을 보여준다. `AGENTS.md`(46KB), `CLAUDE.md`(25KB), `.claude`/`.codex` 디렉토리를 보면 AI 코딩 에이전트를 1급 협업자로 상정한 워크플로우의 산물 같다.

**넷째, TokenJuice가 데드코드가 아니다.** 토큰 압축 엔진(`vincentkoc/tokenjuice`의 Rust 포팅)이 실제로 에이전트 루프에 연결돼 있다(`tool_loop.rs:733, 823`에서 `compact_tool_output` 호출). builtin 96개 JSON 규칙 위에 user/project 레이어를 오버레이하는 3단 구조고, 512바이트 미만이거나 압축 후 비율이 0.95를 넘으면 원문 그대로 반환하는 pass-through 안전장치까지 있다. 압축이 데이터를 망가뜨리지 않게 보수적으로 짠 점이 마음에 들었다.

## 비판·한계

박수만 치면 광고지, 리뷰가 아니다. 여기서부터가 진짜 짚을 부분이다.

**가장 큰 단서: "프라이빗/로컬 우선"인데 요약 LLM 기본값이 클라우드다.** 이게 핵심이다. `MemoryTreeConfig::default()`의 `llm_backend`는 `Cloud`다(`storage_memory.rs:193`, 테스트 이름이 아예 `llm_default_is_cloud`). 로컬 모델을 쓰려면 사용자가 명시적으로 설정해야 한다. 무슨 뜻이냐면, 기본 설정에서 메모리 트리가 요약을 만들 때 **내 청크 본문이 tinyhumans 클라우드 백엔드(`summarization-v1`)로 전송된다**는 거다. README의 "워크플로우 데이터가 기기에 남는다"는 주장은 *저장*에는 맞지만 *처리*에는 기본값 기준으로 맞지 않는다. Ollama 로컬은 opt-in이다. 저장과 처리를 구분하지 않으면 오해하기 딱 좋은 지점이다.

**둘째, OAuth 토큰 집중 + 약한 키 보관.** 외부 리뷰어들이 가장 걱정한 부분([TechTimes](https://www.techtimes.com/articles/316731/20260516/agent-that-reads-you-first-openhuman-tops-github-trending-inverting-playbook.htm), [KnightLi](https://www.knightli.com/en/2026/05/15/openhuman-open-source-personal-ai-agent/))인데, 코드를 보니 실재하는 우려였다. 100개 넘는 통합의 OAuth `access_token`/`refresh_token`이 단일 `auth-profiles.json`에 모인다(`credentials/profiles.rs`). 메일·캘린더·코드 레포·채팅에 **결제 툴링**까지 토큰이 한곳에 쌓인다. 암호화는 ChaCha20-Poly1305 AEAD로 견고한데, 정작 그 키가 암호문 바로 옆 `.secret_key`에 평문으로(권한만 0600) 저장된다.

```rust
// src/openhuman/security/secrets.rs:1-17, 49-62
// Secrets are encrypted using ChaCha20-Poly1305 AEAD with a random key stored
// in `{data_dir}/openhuman/.secret_key` with restrictive file permissions (0600).
// For sovereign users who prefer plaintext, `secrets.encrypt = false` disables this.
pub fn encrypt(&self, plaintext: &str) -> Result<String> {
    if !self.enabled || plaintext.is_empty() {
        return Ok(plaintext.to_string()); // ← 비활성 시 평문 그대로
    }
    let key_bytes = self.load_or_create_key()?; // ← 키는 암호화 안 된 채 디스크에
    /* ChaCha20Poly1305 ... enc2:<hex> */
}
```

주석을 보면 위협 모델을 솔직하게 적어뒀다. 이 암호화는 "config 평문 노출 / grep·git 유출 / 실수 커밋"을 막으려는 거지 로컬 멀웨어나 디스크 접근 공격자를 막는 게 아니다. 솔직한 건 칭찬할 만하지만, 마케팅의 "로컬 암호화"가 디스크 접근까지 막아준다는 인상은 분명 과장이다. 견고한 AEAD를 쓰면서 키 관리가 그 강도를 못 받쳐주는 셈이다.

**셋째, 셀링포인트인 메모리 트리 자체가 마이그레이션 중이다.** `memory/README.md`를 보면 레거시 `store/`와 신규 `tree/`가 공존하며 점진 교체 중이고, 본문 곳곳에 "Phase 1/2/3a"와 이슈 번호가 박혀 있다. 두 retrieval 표면을 동시에 유지하는 부담이 현재의 복잡도다. "early beta" 자기표기와 일치하긴 한데, 가장 내세우는 기능이 아직 안정 단계가 아니라는 건 알고 써야 한다.

**넷째, "80% 절감"과 화제성의 실체.** TokenJuice의 "최대 80% 비용·지연 절감"은 코드 어디에도 그 수치를 보장하는 곳이 없다. 압축은 규칙이 매칭되는 장황한 도구 출력(git status 200줄 같은)에서나 효과가 크고, 비압축 입력은 그냥 pass-through(ratio 1.0)다. 80%는 평균치가 아니라 best-case다. 게다가 [KnightLi](https://www.knightli.com/en/2026/05/15/openhuman-open-source-personal-ai-agent/)가 지적했듯 압축은 "무엇을 남기고 무엇을 버릴지 결정"하므로, 계약서·청구서·의료기록처럼 손실이 치명적인 자료엔 위험하다. 화제성도 따져볼 필요가 있다. 앞서 봤듯 HN 유기적 토론은 거의 없었고, 별 2.2만은 Product Hunt #1 + Trendshift 배지 + X 홍보 + AI 디렉토리 동시 노출의 산물에 가깝다. TechTimes도 이걸 "durable community scale이 아니라 rapid-start momentum"이라고 신중하게 표현했다. 참고로 비교표에서 OpenClaw/Hermes(MIT) 대비 자사 "GNU"를 강조하지만, GPLv3는 카피레프트라 MIT보다 오히려 더 제약적이다 — "더 오픈"이라는 뉘앙스는 결이 다르다.

## 결론

**잘 맞는 사람.** (1) Rust로 짠 견고한 메모리 파이프라인·분산 ingest 설계가 궁금한 개발자 — `bucket_seal.rs`와 `ingest.rs`는 공부할 거리가 많다. (2) 로컬 LLM(Ollama)을 직접 붙여 처리까지 온디바이스로 가둘 의지가 있는 프라이버시 매니아. (3) "내 일상 데이터를 하나로 묶는 제2의 뇌"라는 컨셉을 실험해보고 싶고, beta의 거친 면을 감수할 얼리어답터.

**피해야 할 사람.** (1) "프라이빗"이라는 카피만 믿고 민감 데이터를 기본 설정으로 던질 사람 — 기본값이 클라우드 요약이라는 걸 모르면 안 된다. (2) 결제·의료·법무처럼 토큰 유출이나 압축 손실이 치명적인 환경. OAuth 토큰이 단일 파일에 모이는 모델이 부담스럽다면 지금은 아니다. (3) `curl | bash` 설치를 그냥 못 넘기는 사람. 외부 리뷰어도 "설치 스크립트를 열어 직접 확인하라"고 권한다.

프로덕션이냐 실험이냐로 묻는다면, 지금은 명백히 실험용이다. 인프라(전송층·이벤트 버스·잡 큐·CI)는 프로덕션 지향으로 단단하게 짜여 있지만, 정작 셀링포인트인 메모리 트리가 마이그레이션 중이고 보안 모델에 솔직한 구멍이 있다.

직접 까보고 든 느낌은 — 마케팅은 과했지만 코드는 진지하다는 거다. "80% 절감"이나 "초지능" 같은 카피만 보고 지나쳤다면 아까웠을 만큼, 메모리 트리 구현 자체는 잘 만들었다. 다음엔 `subconscious` 도메인이 실제로 뭘 하는지, 그리고 cloud-default가 언제쯤 local-default로 뒤집힐지를 지켜보고 싶어졌다. 카피와 실체 사이의 거리가 좁혀지는 순간이 이 프로젝트의 진짜 변곡점일 거다.
