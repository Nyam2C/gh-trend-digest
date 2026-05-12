---
title: "HKUDS/AI-Trader 뜯어보기 — '100% 풀오토 AI 트레이딩'이라는 카피 뒤에 뭐가 있나"
repo: HKUDS/AI-Trader
url: https://github.com/HKUDS/AI-Trader
language: python
date: 2026-05-12
summary: "별 16k짜리 트렌딩 1위 레포 HKUDS/AI-Trader를 코드 레벨까지 뜯어봤다. 마케팅 카피는 '100% Fully-Automated Agent-Native Trading'인데, 서버 17k LOC를 다 훑어도 LLM 추론 코드 한 줄이 없다. 그래서 이게 뭐냐면, AI 에이전트끼리 같은 조건으로 종이거래를 시키고 그 결과를 학술 분석 파이프라인까지 돌려서 측정하는 — 거래소가 아니라 운동장이다."
tags:
  - ai-agent
  - trading
  - benchmark
  - hkuds
  - fastapi
  - paper-trading
  - skill-md
  - copy-trading
  - llm-evaluation
  - python
---

# HKUDS/AI-Trader 뜯어보기 — "100% 풀오토 AI 트레이딩"이라는 카피 뒤에 뭐가 있나

GitHub 트렌딩 첫 줄에 별 16,000개짜리 레포가 떠 있길래 들어가 봤다. 이름은 [HKUDS/AI-Trader](https://github.com/HKUDS/AI-Trader). README 첫 줄에 적힌 한 문장이 좀 거창했다. **"AI-Trader: 100% Fully-Automated Agent-Native Trading."** 솔직히 처음엔 한숨이 좀 나왔다. AI 트레이딩 봇 떡밥은 지난 6개월 동안만 해도 다섯 번은 본 것 같은데, "100% 풀오토"라니, 또 마케팅 래퍼인가 싶었다.

그런데 별 16k에 포크 2.6k. 포크 비율이 16% 가까이 된다는 게 좀 이상했다. 보통 트렌딩 레포 평균이 5~8%인데, 이건 거의 그 두 배다. **걸어두는 별이 아니라 실제로 들고 가서 굴려보는 사람들이 많다**는 신호다. 그래서 클론을 떠서 코드를 직접 뜯어봤다. 결론부터 말하면, 이 레포는 카피가 절반은 맞고 절반은 좀 다르다. 그리고 "다른 절반"이 훨씬 흥미롭다.

## 의외의 사실 하나 — 서버 코드 안에 LLM이 없다

처음 `service/server/`를 열어봤을 때 좀 당황했다. 17,119줄짜리 백엔드인데 `import openai`도 없고, `anthropic`도 없고, `transformers`도 없다. `grep -r "openai\|anthropic\|langchain"`을 돌려도 안 잡힌다. 그러니까 **이 레포 자체에는 AI 추론 코드가 한 줄도 없다.**

처음엔 "응? 그럼 'AI Trader'는 무슨 의미야?"라고 의문이 들었는데, `skills/ai4trade/SKILL.md`를 열어보고 나서야 그림이 잡혔다. 이 레포는 AI 모델이 직접 매매하는 봇이 아니라, **외부 AI 에이전트들이 와서 매매를 하도록 만든 거래소·운동장**이다. 정확히는 종이거래(paper trading) 거래소다.

사용 흐름을 풀어보면 이렇다. 사용자가 자기 에이전트 — Claude Code든, Codex든, Cursor든 — 한테 이렇게 한 줄 던진다.

> "Read https://ai4trade.ai/SKILL.md and register"

그러면 에이전트는 그 SKILL.md(838줄짜리)를 읽고, 자기 자신을 ai4trade.ai에 등록하고, 토큰을 받고, 트레이딩 신호를 발행하거나 다른 에이전트를 카피하기 시작한다. **서버는 그 에이전트의 행동을 받아 검증·기록·점수·정산하는 인프라일 뿐**이다. "AI"는 외부 에이전트가 가져온다.

이걸 깨닫고 나니 README의 "100% Fully-Automated"가 다르게 읽혔다. 자동성은 *외부 에이전트 자기 자신*의 자동성이지, *AI가 알아서 돈을 벌어준다*는 의미의 자동성이 아니었다. 이 정정이 별 거 아닌 것 같지만, 다음 단락부터 모든 게 달라진다.

## 그래서 이게 진짜 뭐냐면 — 거래소가 아니라 운동장이다

`service/server/worker.py`라는 42줄짜리 파일이 있다. 백그라운드 워커의 진입점인데, 여기서 `start_background_tasks()`가 켜는 루프 목록이 흥미롭다. `tasks.py:841-859`에서 가져오면 이렇다.

```python
# tasks.py:841-856
BACKGROUND_TASK_REGISTRY = {
    "prices": update_position_prices,
    "profit_history": record_profit_history,
    "polymarket_settlement": settle_polymarket_positions,
    "challenge_settlement": settle_challenges_loop,
    "team_mission_form": form_team_missions_loop,
    "team_contribution_score": score_team_contributions_loop,
    "team_mission_settlement": settle_team_missions_loop,
    "signal_quality_score": score_signal_quality_loop,
    "agent_metric_snapshots": refresh_agent_metric_snapshots_loop,
    "network_edges": build_network_edges_loop,
    "market_news": refresh_market_news_snapshots_loop,
    "macro_signals": refresh_macro_signal_snapshots_loop,
    "etf_flows": refresh_etf_flow_snapshots_loop,
    "stock_analysis": refresh_stock_analysis_snapshots_loop,
}
```

루프가 14개다. 가격 폴링, 이익 기록, 폴리마켓 정산, 챌린지 정산, 팀 형성, 팀 기여 점수, 팀 미션 정산, 시그널 품질 점수, 에이전트 메트릭 스냅샷, 네트워크 엣지 빌드, 시장 뉴스, 매크로 시그널, ETF 흐름, 종목 분석. 한 번 더 천천히 읽어보길 권한다. 이게 어디 "거래 봇 서버" 목록처럼 보이나? 나는 **실험실의 데이터 수집 데몬** 목록 같았다.

특히 `build_network_edges_loop`나 `score_team_contributions_loop` 같은 항목이 결정적이다. 누가 누구를 팔로우하고 누가 누구의 답글을 채택했는지를 그래프 데이터로 쌓고, 팀 단위 기여도를 따로 점수화한다. 이건 거래 인프라보다는 **사회과학 실험 측정 도구**에 훨씬 가깝다.

`research/` 디렉토리에 들어가 보면 이 가설이 확정된다. `research/scripts/research_common.py:21-27`이 정의하는 5개 출력 테이블 이름을 보자.

```python
# research/scripts/research_common.py:21-27
TABLE_NAMES = [
    "rq_hypothesis_metrics.csv",
    "competition_effects.csv",
    "cooperation_effects.csv",
    "hybrid_effects.csv",
    "network_effects.csv",
]
```

**경쟁(Competition) · 협력(Cooperation) · 둘 다 섞은 하이브리드(Hybrid) · 네트워크 효과(Network).** 이게 그냥 데이터 모델 이름이라고 보면 곤란하다. 이건 **연구 가설의 구조**다. "AI 에이전트들이 경쟁만 시키면 어떻게 되나, 협력만 시키면, 둘 다 섞으면, 사회적 네트워크 효과는 어떻게 나타나나"를 측정하기 위해 미리 짜둔 4축의 실험 매트릭스다. AI-Trader라는 이름의 절반은 거래 플랫폼이고, 나머지 절반은 **에이전트 사회 시뮬레이션**이다.

이 시각으로 다시 보면, 14개 백그라운드 루프가 동시에 도는 게 비로소 자연스럽게 읽힌다. 거래 봇은 가격 폴링 하나만 있으면 되지만, **실험 인프라는 14개가 다 필요하다**.

## 한 번의 매매가 일으키는 후폭풍

코드를 읽다가 가장 인상적이었던 부분은 `routes_signals.py:95-555`의 `push_realtime_signal` 함수다. 에이전트가 "BTC 1개 매수"라고 시그널 하나만 던지면, 서버 내부에서 펼쳐지는 일이 어이없을 정도로 많다.

먼저 토큰으로 인증을 하고, 그 에이전트가 어떤 활성 실험의 어떤 variant 그룹에 속해 있는지 확인한다. US 주식이면 ET 기준 장 시간(9:30~16:00 평일)인지 체크하고, 폴리마켓이나 암호화폐면 에이전트가 보낸 가격을 무시하고 서버가 직접 외부 API에서 가격을 받아온다 — 4월 12일 머지된 PR(`c93f276 Force server pricing for crypto and Polymarket trades`)이 이 가드를 추가했다. **에이전트가 자기에게 유리한 가격을 거짓 보고하는 걸 막기 위한 거다.** 종이거래라도 거짓 보고는 막아야 한다는 게 좀 진지해서 좋았다.

그 다음이 진짜 후폭풍이다. 시그널 ID를 시퀀스 테이블에서 발급하고, `signals` 테이블에 INSERT를 박고, `positions` 테이블에서 평균 진입가를 재계산해서 갱신하고, `agents.cash`에서 거래대금+수수료를 차감한다. 여기까지가 "원래 거래" 부분이다.

그리고 이제 진짜 시작인데 — 이 에이전트가 참여 중인 모든 활성 챌린지에 대해 `challenge_trades` 테이블에 동일 거래의 *스냅샷*을 별도로 적는다. 챌린지가 끝났을 때 라이브 포지션이 아니라 *그 챌린지 동안만의 거래들*만 가지고 결과를 재계산할 수 있게 하기 위해서다. 그 다음 `signal_quality.py`로 가서 5축 점수를 매기고, `_reward_for_context`로 실험 variant별 보상 룰을 적용하고, `experiment_events` 테이블에 'signal_published' 이벤트를 적재한다.

여기서 끝이 아니다. 이 에이전트를 팔로우하는 모든 follower를 `subscriptions` 테이블에서 가져와서 **각 follower마다 똑같은 거래를 복사한다**. 이때 `routes_signals.py:407`에 좀 영리한 부분이 있다.

```python
# routes_signals.py:407
cursor.execute(f'SAVEPOINT follower_{follower_id}')
```

각 follower의 거래 시도를 SAVEPOINT 단위로 격리해뒀다. 한 follower가 현금이 부족해서 거래를 못 따라가도 SAVEPOINT만 ROLLBACK하면 되고, leader의 거래나 다른 follower의 거래에는 영향이 없다. PostgreSQL SAVEPOINT를 이런 식으로 카피트레이드에 쓴 건 좀 박수치고 싶었다.

종합해서 보면, 시그널 하나가 평균 10개 테이블에 INSERT/UPDATE를 박는다. `agents`, `positions`, `signals`, `signal_sequence`, `signal_predictions`, `signal_quality_scores`, `agent_reward_ledger`, `experiment_events`, `challenge_trades`, `agent_messages`. **한 거래의 후폭풍이 이렇게 넓다**는 사실 자체가, 이 시스템이 단순 매매 기록보다 *실험 이벤트 수집*에 훨씬 무게가 가 있다는 증거다.

## 시그널 품질 점수, 의외로 정직했다

`signal_quality.py`에서 가장 먼저 눈에 띈 건 13번째 줄이었다.

```python
# signal_quality.py:13
MODEL_VERSION = "heuristic-v1"
```

`MODEL_VERSION="heuristic-v1"`. 이게 뭔 뜻이냐면 — **LLM으로 평가하는 게 아니라 휴리스틱이라고 정직하게 이름을 박아둔 거다.** 솔직히 "AI Trader"라는 이름의 프로젝트에서 시그널 평가까지 AI로 안 한다는 게 좀 의외였는데, 코드를 더 보고 나니 합리적 선택이라는 생각이 들었다.

5축 점수 공식은 이렇다. `signal_quality.py:200-212`에서.

- **verifiability (30%)**: 방향성·종목·목표가가 명시되어 있나 — 즉 사후 검증 가능한가
- **evidence (25%)**: 본문 길이 + "because/risk/evidence/data/chart/catalyst" 같은 키워드 빈도
- **specificity (20%)**: 종목·태그·길이 — 구체적인가
- **novelty (15%)**: 본문이 다른 시그널과 중복되면 감점
- **review (10%)**: 다른 에이전트의 답글이 채택됐는가

이걸 가중합해서 overall_score를 만든다. **이게 무서운 게 어디서 본 것 같지 않나? 학술 논문 평가 기준이다.** 검증 가능성, 근거, 구체성, 신규성, 동료 리뷰. HKUDS가 홍콩대 데이터 인텔리전스 랩이라는 걸 생각하면, 이 평가 축은 의도된 디자인이다.

그리고 키워드 매칭이 한·중·영 다중 언어다.

```python
# signal_quality.py:91-101
if any(word in lower for word in ("buy", "long", "bull", "upside", "breakout", "看多", "上涨")):
    direction = "up"
elif any(word in lower for word in ("sell", "short", "bear", "downside", "breakdown", "看空", "下跌")):
    direction = "down"
```

`看多`, `看空`, `目标价`(목표가), `置信度`(신뢰도) 같은 중국어 키워드가 영어 키워드와 나란히 들어가 있다. 한국어가 없는 건 좀 아쉽지만, 코드베이스가 중국어권 학술 그룹에서 시작했다는 흔적이 곳곳에 남아있는 게 정직해 보였다.

LLM 평가를 안 쓴 이유는 명백하다. 4,000+ 에이전트가 매 시그널마다 발행할 텐데, 그걸 일일이 GPT-4한테 점수 매기게 하면 비용이랑 지연이 폭발한다. v1 단계에서 정규식·키워드로 가는 건 솔직한 선택이고, 진짜 ground truth는 시그널 텍스트가 아니라 **그 시그널대로 거래한 결과 PnL**이다. 이 PnL은 따로 챌린지 시스템에서 정확하게 추적한다.

## 챌린지 시스템 — 라이브 거래를 백테스트 메트릭으로 채점한다

이 부분이 어쩌면 이 레포에서 가장 영리한 부분일지 모른다. AI-Trader는 전통적 의미의 백테스트 엔진이 아니다. 과거 데이터에 가설을 돌리지 않는다. 모든 거래는 라이브 종이거래로 일어난다. 그런데 챌린지 시스템에는 **결정적 portfolio replay**가 들어있다.

`challenge_scoring.py:67-91`을 보자.

```python
# challenge_scoring.py:67-83
cash = starting_cash
positions: dict[tuple[str, str], dict[str, Any]] = {}
marks: dict[tuple[str, str], float] = {}
equity_curve = [starting_cash]
peak = starting_cash
max_drawdown = 0.0
...
ordered_trades = sorted(
    [_row_dict(trade) for trade in trades],
    key=lambda item: (item.get('executed_at') or '', item.get('id') or 0),
)
```

각 참가자의 거래를 시간순으로 정렬해서, 빈 포트폴리오에서 다시 한 번 재실행한다. 매 거래마다 equity curve를 갱신하고, max drawdown을 누적 추적한다. **이건 표준 백테스트 엔진의 미니어처다.**

그리고 잘못된 거래 시퀀스가 나오면 자동 실격까지 시킨다.

```python
# challenge_scoring.py:107-109
if side == 'buy':
    if current_qty < 0:
        disqualified_reason = f'buy_used_while_short:{symbol}'
        break
```

숏 포지션에서 buy로 청산하려는 식의 잘못된 시퀀스를 만나면 즉시 실격이다. 최대 포지션 비중이나 최대 낙폭 룰도 정산 시점에 강제한다.

마지막으로 점수 방식이 두 종류로 갈린다. 단순 수익률만 보는 'return-only' 모드와, 드로다운 페널티까지 적용하는 'risk-adjusted' 모드. **arXiv 논문에서 강조한 "risk control capability determines cross-market robustness"라는 결론을 측정하기 위한 도구가 바로 이거다.**

그래서 GitHub 이슈 [#207](https://github.com/HKUDS/AI-Trader/issues/207)에서 누가 "백테스트·아웃오브샘플 증거가 어디 있냐"고 따져 묻는데, 이건 절반은 맞고 절반은 빗나간 비판이다. AI-Trader가 *전통적 백테스트*는 안 하는 게 맞다. 하지만 *라이브 거래를 챌린지 윈도우 안에서 재정산*하는 메커니즘은 있다. **Forward-test + replay scoring**이지 backward-test가 아니다. 솔직히 어느 쪽이 더 정직한 평가인지는 의견이 갈릴 일이다.

## 학술 결론을 한 번 더 짚어야 한다

이 레포가 단순 마케팅 래퍼가 아닌 가장 강한 증거는 [arXiv:2512.10971](https://arxiv.org/abs/2512.10971)에 실린 테크니컬 리포트다. 제목이 *AI-Trader: Benchmarking Autonomous Agents in Real-Time Financial Markets*. 저자는 Tianyu Fan, Yuhao Yang, Yangqin Jiang, Yifei Zhang, Yuxuan Chen, Chao Huang. Chao Huang은 HKU의 데이터 인텔리전스 랩 디렉터다.

그런데 논문 결론이 마케팅 카피와 미묘하게 어긋난다. 논문 abstract 원문을 그대로 가져와 보면 이렇다.

> "General intelligence does not automatically translate to effective trading capability, with most agents exhibiting poor returns and weak risk management. We demonstrate that risk control capability determines cross-market robustness, and that AI trading strategies achieve excess returns more readily in highly liquid markets than policy-driven environments."
> — [arXiv:2512.10971](https://arxiv.org/abs/2512.10971)

번역하면 — "일반 지능이 자동으로 효과적인 트레이딩 능력으로 변환되지는 않는다. 대부분의 에이전트는 빈약한 수익과 약한 리스크 관리를 보였다." 이 한 줄을 깔아두고 다시 READ.ME의 "100% Fully-Automated Agent-Native Trading"을 읽으면, 두 문장이 정확히 같은 프로젝트를 가리키고 있다는 게 좀 어색하게 느껴진다.

내 해석은 이렇다. 이 프로젝트는 **AI 트레이더가 잘 한다는 걸 자랑하는 도구가 아니라, 대부분의 AI 트레이더가 어떻게 망하는지를 측정하는 벤치마크**다. 6개 메인스트림 LLM × 3개 시장(미국 주식 / A주 / 암호화폐) × 다중 거래 주기로 비교한 결과를 정직하게 공개한 거다.

저자 본인도 X에서 "**battle between leading [models]**"이라고 표현했다. ([Chao Huang on X](https://x.com/huang_chao4969/status/2004756041498861588)) "AI 모델끼리의 결투장"이라는 프레이밍이 마케팅보다 더 정확한 본질이다.

## HKUDS 라인업, 한 번 짚고 가자

이 랩이 어떤 곳인지 모르고 글을 끝낼 수는 없을 것 같다. HKUDS는 홍콩대학교 데이터 사이언스 학부 산하 Data Intelligence Lab이고, 디렉터는 Chao Huang 조교수다. KDD·WWW·SIGIR 같은 메이저 컨퍼런스에 꾸준히 베스트 페이퍼급 논문을 내는 곳이다.

그런데 무엇보다 인상적인 건 라인업이다. 별 수만 추려 보면 이렇다.

| 레포 | 별 수 | 한 줄 |
|------|-------|-------|
| [nanobot](https://github.com/HKUDS) | ~42k | 초경량 개인 AI 에이전트 |
| [LightRAG](https://github.com/HKUDS/LightRAG) | ~35k | 빠른 RAG 프레임워크 (EMNLP 2025) |
| [CLI-Anything](https://github.com/HKUDS) | ~34k | 모든 소프트웨어를 Agent-Native로 |
| [DeepTutor](https://github.com/HKUDS) | ~24k | Agent-Native 학습 어시스턴트 |
| [RAG-Anything](https://github.com/HKUDS) | ~20k | 올인원 RAG 프레임워크 |
| **AI-Trader** | **~16k** | **Agent-Native Trading 벤치마크** |
| [DeepCode](https://github.com/HKUDS) | ~16k | 논문을 코드로 변환하는 에이전트 |
| [Vibe-Trading](https://github.com/HKUDS/Vibe-Trading) | ~7.1k | 개인용 트레이딩 에이전트 |

총합 별 150k+. 한 랩이 이런 페이스로 히트작을 찍어내는 건 진짜 드문 경우다. **"Agent-Native"라는 단어를 거의 브랜드처럼 쓰는 것**도 눈에 띈다. DeepTutor도 Agent-Native, CLI-Anything도 Agent-Native, AI-Trader도 Agent-Native. 한 랩에서 농축된 세계관이 보인다.

한국 커뮤니티에서도 PyTorchKR 운영자 9bow(박정환) 님이 [AI-Trader 소개글](https://discuss.pytorch.kr/t/ai-trader-ai/8094)을 올린 데 이어, 한 달도 안 돼서 자매 프로젝트 [Vibe-Trading 소개글](https://discuss.pytorch.kr/t/vibe-trading-ai-feat-hkuds/9732)까지 올렸다. 다음 한국어 인용은 PyTorchKR AI-Trader 소개글에서 가져온 것이다.

> "AI-Trader는 GPT, Claude, Qwen 등 다양한 AI 모델이 인간의 개입 없이 나스닥 100 종목을 자율적으로 거래하는 오픈소스 프로젝트입니다. 이는 'AI가 시장에서 수익을 낼 수 있는가?'라는 질문을 탐구하는 실험적 플랫폼입니다."
> — [PyTorchKR AI-Trader 소개](https://discuss.pytorch.kr/t/ai-trader-ai/8094)

"실험적 플랫폼"이라는 표현이 핵심을 짚었다고 본다.

## 그래서 비판은 어디서 오나

마냥 좋은 글로 끝내면 안 될 것 같아 비판도 정리해본다. 가장 강한 비판은 위에서 잠깐 언급한 [Issue #207](https://github.com/HKUDS/AI-Trader/issues/207)이다. 작성자 Anic888이 논문 출시 직후인 2026-05-10에 올린 글의 핵심 문장은 이거다.

> "The repository appears to be 'a marketing wrapper around ai4trade.ai with no edge evidence' rather than a legitimate trading system."
> — [Issue #207](https://github.com/HKUDS/AI-Trader/issues/207)

"ai4trade.ai에 대한 마케팅 래퍼이지 정통 트레이딩 시스템은 아니다"라는 직설. 그리고 `skills/` 디렉토리가 단순 REST 클라이언트일 뿐이라는 지적. **부분적으로는 맞는 말이다.** 레포 자체는 신호 *생성* 로직을 갖고 있지 않다. 신호 생성은 외부 에이전트가 한다.

다만 이 비판에서 빠진 게 있다. 논문(arXiv 2512.10971)이 6개 LLM × 3개 시장으로 평가한 결과를 publish했다는 사실이다. 즉 **edge evidence는 학술 논문에 있지 레포에는 없다.** 이 분리 자체에 대한 비판은 정당하지만 — 학술 표가 README에 임베드되거나 별도 evaluation/ 디렉토리에 깔끔하게 정리됐으면 좋겠다 — "전혀 근거가 없다"는 단정은 부정확하다.

다른 종류 비판도 있다. Issue [#8](https://github.com/HKUDS/AI-Trader/issues/8)에서 jyan1999가 lookahead bias 가능성을 제기했다. Jina Search에서 날짜 파싱 실패 시 결과를 그냥 유지하거나, `get_price_local`이 당일 OHLCV 전체를 반환해서 종가 정보가 결정 시점에 노출될 수 있다는 지적이다. "Is this by design?"으로 끝난 이 이슈는 라이브 벤치마크가 아무리 잘 설계돼도 데이터 누수 문제에서 자유롭지 않다는 걸 보여준다.

실용적 트러블도 적지 않다. `.env.example`이 깨져 있어서 그대로 source하면 shell이 망가지는 문제([Issue #205](https://github.com/HKUDS/AI-Trader/issues/205))나, Docker 설정과 로컬 SQLite 기본값이 어긋난다는 [#206](https://github.com/HKUDS/AI-Trader/issues/206), Market Intelligence API가 1~2일 지연된 가격을 반환한다는 #188 같은 게 OPEN으로 남아있다.

조직 측면에서 한 가지 더 짚자면, 메인테이너 단일성이 좀 두드러진다. 최근 머지된 PR을 보면 TianyuFan0504(논문 1저자 Tianyu Fan과 동일인 추정)가 분 단위 self-merge를 하는 패턴이 보인다. 외부 PR들 — 보안 강화 PR #204, 마켓 인텔 버그 PR #199, 100% 유닛 테스트 커버리지 PR #198 등 — 은 OPEN으로 쌓여 있다. 학술 프로젝트 특성상 자연스러운 면도 있지만, OSS로서는 외부 기여 사이클이 거의 없는 상태다.

라이선스도 작은 의문이다. README에 MIT 배지가 달려 있는데 실제 LICENSE 파일이 클론에 없고, gh API도 license를 `null`로 반환한다. 외부 기여자 입장에서 명확성이 부족하다.

## 라이브 벤치마크 시장에서의 위치

이 영역에 경쟁 프로젝트가 많다. [TradingAgents (Tauric Research)](https://github.com/TauricResearch/TradingAgents)는 실제 트레이딩 펌의 직제(펀더멘털 애널리스트, 센티먼트 전문가, 테크니컬 애널리스트, 트레이더, 리스크 관리)를 LLM 에이전트로 미러링한다. [FinGPT](https://github.com/AI4Finance-Foundation/FinGPT)는 금융 도메인에 파인튜닝된 모델 그 자체를 제공한다. [FinRobot](https://github.com/AI4Finance-Foundation/FinRobot)은 LLM + RL + 양적 분석의 종합 툴킷이다.

여기서 [Strategy Arena](https://strategyarena.io/en/blog/compare-ai-trading-bots-2026) 같은 라이브 벤치마크도 같이 비교해볼 만하다. Claude / GPT / Grok / Gemini / DeepSeek / Perplexity에 각각 $10,000씩 주고 Binance 실시간 BTC 거래를 시킨다. 그런데 결론이 인상적이다.

> "어떤 AI 모델도 신뢰할 만한 자율 트레이더는 아니다." (2026년 2월 기준)
> — [aitradingtools 2026 비교](https://aitradingtools.org/blog/chatgpt-vs-gemini-vs-claude-trading-research-2026)

AI-Trader의 차별점은 명확하다. **arXiv 논문이 뒷받침되는 거의 유일한 라이브 벤치마크다.** Strategy Arena나 GPTrader는 블로그/마케팅 중심이지만, AI-Trader는 학술 컨퍼런스에서 인용 가능한 표준 후보로 디자인됐다. 마케팅 부분이 부풀어 있는 건 사실이지만, 그 부풀린 카피 뒤에 진짜 학술 인프라가 깔려 있다는 게 다른 라이브 벤치마크와의 차이다.

## 누가 써야 좋고 누가 피해야 좋나

여기까지 다 읽고 나면 "그래서 나는 이거 깔아? 말아?"가 남는다. 솔직히 정리해본다.

**잘 맞는 사람**

- **AI 에이전트 행동 연구가 궁금한 사람.** 4축(competition / cooperation / hybrid / network) 매트릭스로 정리된 4,000+ 에이전트 실험 데이터셋은 다른 곳에선 못 구한다.
- **자기 에이전트(Claude Code, Codex 등)에게 "실제 같지만 안전한 매매 환경"을 제공하고 싶은 사람.** SKILL.md 한 줄로 가입에서 매매까지 자동화된다.
- **LLM 벤치마크 디자인 자체에 관심 있는 사람.** SHA-256 결정적 버켓팅, Benjamini-Hochberg FDR 보정, 결정적 bootstrap CI까지 학술 그레이드 인프라가 그대로 코드로 들어있다.
- **카피트레이드의 SAVEPOINT 격리 같은 거래 시스템 디테일 보는 게 재밌는 사람.** 트랜잭션 디자인의 작은 박수 포인트가 많다.

**피하면 좋은 사람**

- **AI가 알아서 돈을 벌어주는 봇을 찾는 사람.** 이 레포는 그게 아니다. 외부 에이전트가 매매하고, 대부분의 에이전트는 논문 결론대로 빈약한 수익을 낸다.
- **실거래 시스템을 찾는 사람.** 종이거래 전용이다. 실제 자금 흐름은 일절 없다.
- **잘 정리된 API 키 셋업 가이드를 기대하는 신규 개발자.** `.env.example`이 깨져 있고, Docker 설정도 어긋난다. Alpha Vantage API 키 없으면 미국 주식 path가 일부 막힌다. 운영에 어느 정도 익숙해야 한다.
- **전통적 백테스트로 전략을 검증하려는 사람.** 챌린지 시스템에 replay scoring은 있지만 walk-forward나 out-of-sample 검증은 따로 안 한다.

## 마무리

며칠 동안 코드를 뜯어보면서 처음 생각과 결론이 꽤 많이 달라졌다. "또 마케팅 래퍼겠지" → "이건 거래 봇이 아니라 사회 실험 인프라구나" → "그런데 학술 결론은 마케팅 카피랑 좀 어긋나는데" → "그래도 이 정도 디테일까지 짜둔 라이브 벤치마크는 흔치 않다."

당장 사이드 프로젝트에 넣지는 않을 것 같다. 트레이딩 보다 학술 측정 도구라는 성격이 강한 시스템에 내 코드를 더할 자신이 없다. 다만 `experiments.py:161-178`의 SHA-256 버켓팅이나 `research_common.py:139-149`의 결정적 bootstrap CI는 다른 프로젝트에서 A/B 실험 짤 때 참고하고 싶어졌다. 그리고 다음번에 HKUDS가 새 레포를 풀면 이번엔 좀 빨리 들어가 봐야겠다는 생각이 들었다. 이 랩, 페이스가 무섭다.

---

### 참고

- [HKUDS/AI-Trader 레포](https://github.com/HKUDS/AI-Trader)
- [arXiv:2512.10971 — AI-Trader 테크니컬 리포트](https://arxiv.org/abs/2512.10971)
- [AI-Trader 벤치마크 페이지](https://hkuds.github.io/AI-Trader/index.html) / [ai4trade.ai](https://ai4trade.ai/)
- [Chao Huang HKU 페이지](https://datascience.hku.hk/people/chao-huang/) / [X 프로필](https://x.com/huang_chao4969)
- [PyTorchKR — AI-Trader 한국어 소개](https://discuss.pytorch.kr/t/ai-trader-ai/8094)
- [PyTorchKR — Vibe-Trading 한국어 소개](https://discuss.pytorch.kr/t/vibe-trading-ai-feat-hkuds/9732)
- [Issue #207 — 방법론 비판](https://github.com/HKUDS/AI-Trader/issues/207)
- [Issue #8 — lookahead bias 우려](https://github.com/HKUDS/AI-Trader/issues/8)
- [TradingAgents 비교 프로젝트](https://github.com/TauricResearch/TradingAgents)
- [aitradingtools 2026 라이브 벤치마크 비교](https://aitradingtools.org/blog/chatgpt-vs-gemini-vs-claude-trading-research-2026)
