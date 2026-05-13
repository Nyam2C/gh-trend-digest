---
title: "한 달 만에 1만 스타 — Datawhale의 'Agent 교과서' hello-agents 뜯어보기"
repo: "datawhalechina/hello-agents"
owner: "datawhalechina"
repo_url: "https://github.com/datawhalechina/hello-agents"
language: "python"
stars: 48559
generated_at: "2026-05-13"
summary_one_line: "0부터 LLM Agent를 짜는 16장 중국어 무료 교재 — 4.8만 스타짜리 'Agent 元年' 학습서를 한 번 뜯어봤다."
tags: [agent, llm, tutorial, datawhale, korean-blog, rag, mcp, react]
---

# 한 달 만에 1만 스타 — Datawhale의 'Agent 교과서' hello-agents 뜯어보기

## 도입

GitHub 트렌딩을 둘러보다가 4만 8천 스타짜리 레포가 눈에 박혔다. `datawhalechina/hello-agents` — README가 통째로 중국어다. 영어 버전(`README_EN.md`)도 있긴 한데, 본진은 명백히 중문이다. 그런데 스타는 2025년 9월에 생긴 레포가 8개월 만에 5만에 육박한다. 공개 일주일 만에 2.2k, 한 달 만에 1만 스타. 곡선이 거의 수직이다.

처음엔 "또 LangChain 래퍼 정리집인가?" 싶었다. 그런데 디렉토리 트리를 펼쳐보고 좀 흥미가 생겼다. `docs/chapter1` ~ `docs/chapter16`까지 챕터 폴더가 줄지어 있고, `code/chapter1` ~ `code/chapter16`가 1:1로 대응한다. 라이브러리가 아니라 책이다. 4.8만 스타짜리 책. 그리고 라이선스가 흔한 MIT/Apache가 아니라 **CC BY-NC-SA 4.0** — 비상업·동일조건. 코드 레포 라이선스로는 보기 드물다. 이 한 줄이 "이건 코드가 아니라 콘텐츠다"라는 정체성을 그대로 박아준다. 도대체 무슨 책이길래.

## 정체

한 줄 요약하면 이렇다. **Datawhale 커뮤니티가 만든 16장짜리 중국어 Agent 교과서 + 학습 코드 모음**. README가 가장 먼저 던지는 문장이 인상적인데, "2024년이 백모대전(百模大战, '백 개 모델의 전쟁')의 해였다면 2025년은 의심할 여지 없는 'Agent 元年(원년)'"이라고 한다. 그러면서 칼끝을 분명히 한다 — Dify·Coze·n8n 같은 워크플로우 기반 에이전트가 아니라 **AI Native Agent**, 즉 LLM이 진짜 의사결정 주체인 에이전트를 만드는 게 목표라고. 이 구분이 책 전체를 관통한다.

만든 곳은 **Datawhale**. 2018년 설립된 중국 최대 규모의 AI 오픈소스 학습 커뮤니티다. 수십만 학습자, 1,088개 대학 앰배서더 네트워크, 613개 기업 연계 — 한 [Substack 분석 글](https://mingai01.substack.com/p/rethinking-open-source-the-atypical)은 이걸 두고 "'학습자를 위해(For the learner)'라는 미션을 지키는 비전형적 오픈소스 조직"이라고 평했다. 한국으로 치면 "혼자 공부하는 AI"가 동아리 단위가 아니라 거의 학회 규모로 굴러간다고 보면 비슷할까. Datawhale의 자매 시리즈로는 [happy-llm](https://github.com/datawhalechina/happy-llm)(0부터 LLM 만들기), self-llm(오픈소스 LLM 파인튜닝), eat-pytorch-in-20-days 등이 있다. hello-agents는 이 "0부터 X 만들기" 계보의 최신작이다.

저자 라인업은 **jjyaoao(陈思州=Sizhou Chen)가 프로젝트 리더**이고, 부쩍 9장을 맡은 Tao Sun(fengju0213), 연습문제 담당 Shufan Jiang(Tsumugii24), 5장 Peilin Huang(HeteroCat), 14장 Xinmin Zeng(fancyboi999, Niuke Tech의 Agent 엔지니어), 그리고 절강사범대 항저우 AI 연구원 교수이자 Datawhale 수석 과학자 朱信忠 지도교수까지 6명이 핵심이다. 즉 이건 동아리 사이드 프로젝트가 아니라 교수 한 명 + 현직 Agent 엔지니어 + 커뮤니티 리더가 같이 묶인, 약간 학회/교양서 같은 결의 프로젝트다. README에 `@misc{hello_agents2025, ...}` 형식의 BibTeX 인용 정보까지 박혀 있다. 학술 인용을 염두에 둔다는 뜻.

## 아키텍처·구현 디테일

레포를 처음 펼치면 좀 멈칫한다. 루트에 통합 `pyproject.toml`도, `requirements.txt`도 없다. 그 대신 `docs/`(교재 본문), `code/`(챕터별 실행 코드), `Extra-Chapter/`(면접 문제·GUI Agent·Skill 작성법 등 보조 10편), `Co-creation-projects/`(커뮤니티 졸업작품 39개), 이렇게 4겹이 적층돼 있다. 의존성은 챕터별로 자기 폴더에 자기 `requirements.txt`를 둔다. 즉 이 레포는 **단일 패키지가 아니라 17~20개 미니 프로젝트가 한 지붕 아래 사는 형태**다.

학습 흐름은 5부 16장으로 짜여 있는데, 코드를 따라 들여다보면 이런 4단 부스터 구조가 보인다.

1. **챕터 1~3** — 개념 도입. ELIZA(`code/chapter2/ELIZA.py`)부터 Transformer 기초까지. 거의 강의 노트 톤.
2. **챕터 4** — "원리부터". 외부 프레임워크 없이 `openai` SDK + `dotenv` + 정규식만으로 ReAct·Plan-and-Solve·Reflection을 100~160줄에 끝낸다. **이 레포 전체에서 유일하게 프레임워크 의존이 0인 장**이다.
3. **챕터 5~6** — 산업 표준 둘러보기. Dify/n8n yaml 덤프, 그리고 AutoGen·CAMEL·LangGraph·AgentScope 4개 프레임워크를 한 폴더씩 떼서 비교 데모.
4. **챕터 7~12** — 자체 프레임워크 `HelloAgents`와 그 위에 얹은 RAG·메모리·MCP/A2A/ANP 통신 프로토콜·Agentic-RL(SFT→LoRA→GRPO)·BFCL/GAIA 벤치마크.
5. **챕터 13~15** — 풀스택 캡스톤 3개. Vue + FastAPI 여행 플래너, uv 기반 DeepResearch, Godot 4 + FastAPI로 만든 AI 마을(15장은 GDScript까지 등장한다).
6. **챕터 16** — 졸업 설계 가이드. 그래서 `Co-creation-projects/`에 39개의 학습자 졸업작품이 쌓이는 구조다.

가장 교과서적인 코드 한 컷을 보면 책 전체 톤이 거의 다 잡힌다. `code/chapter4/ReAct.py`의 메인 루프다.

```python
# code/chapter4/ReAct.py:33-73
def run(self, question: str):
    self.history = []
    current_step = 0

    while current_step < self.max_steps:
        current_step += 1
        print(f"\n--- 第 {current_step} 步 ---")

        tools_desc = self.tool_executor.getAvailableTools()
        history_str = "\n".join(self.history)
        prompt = REACT_PROMPT_TEMPLATE.format(tools=tools_desc, question=question, history=history_str)

        messages = [{"role": "user", "content": prompt}]
        response_text = self.llm_client.think(messages=messages)
        if not response_text: ...

        thought, action = self._parse_output(response_text)
        if action.startswith("Finish"):
            final_answer = self._parse_action_input(action)
            return final_answer

        tool_name, tool_input = self._parse_action(action)
        observation = tool_function(tool_input) if tool_function else f"错误：未找到名为 '{tool_name}' 的工具。"

        self.history.append(f"Action: {action}")
        self.history.append(f"Observation: {observation}")
```

LangChain도, LlamaIndex도, OpenAI 함수콜링도 없다. `Thought:/Action:/Observation:` 키워드를 LLM이 텍스트로 뱉도록 프롬프트로 강제하고, 정규식 `r"(\w+)\[(.*)\]"` 두 줄로 파싱한다. 99줄. 한 화면에 다 들어온다. 학습자가 ReAct의 정체를 "프레임워크 추상화에 가려지지 않은 상태로" 처음 보게 만든다는 의도가 분명하다. 이게 챕터 4의 칼끝이다.

그리고 같은 챕터의 `llm_client.py`(73줄)에 또 한 가지 디테일이 있다. OpenAI 호환 클라이언트를 래핑하면서 `temperature=0`을 디폴트로, `stream=True`를 고정한 다음, 청크를 모아서 `"".join`으로 돌려준다. 안에선 스트리밍이지만 밖에선 동기 함수처럼 보인다. 비동기·제너레이터 개념을 학습자가 모른 채로도 따라올 수 있게 일부러 한 겹 덮어준 거다. 작은 선택인데 교육 도구로서의 결이 그대로 드러난다.

## 박수 포인트

**첫째, "원리 → 표준 프레임워크 → 자체 프레임워크 → 풀스택"의 4단 부스터 곡선.** 같은 ReAct 패턴을 챕터 4에서 정규식으로 짜고, 6장에서 LangGraph·AutoGen으로 다시 풀고, 7장에서 자체 프레임워크 추상화로 보고, 13~15장에서 그걸로 풀스택 앱을 만든다. 같은 개념을 4번 다른 각도로 만지게 한다. 단순 반복이 아니라 추상화 레벨이 한 단씩 올라간다. 학습 곡선 설계가 좀 작정하고 짠 티가 난다.

**둘째, MCP를 1급 시민으로 다룬다.** 챕터 10에서 `code/chapter10/my_mcp_server.py`(240줄)는 `fastmcp` 데코레이터로 `@mcp.tool()` / `@mcp.resource()` / `@mcp.prompt()` 3종을 한 파일에 보여주는 "쇼케이스 서버"다. 그 다음 챕터 13에서는 MCP를 **"에이전트당 1개 도구"가 아니라 여러 에이전트가 공유하는 단일 백엔드 버스**로 쓴다.

```python
# code/chapter13/helloagents-trip-planner/backend/app/agents/trip_planner_agent.py:166-203
self.amap_tool = MCPTool(
    name="amap",
    description="高德地图服务",
    server_command=["uvx", "amap-mcp-server"],
    env={"AMAP_MAPS_API_KEY": settings.amap_api_key},
    auto_expand=True
)

self.attraction_agent = SimpleAgent(name="景点搜索专家", ...)
self.attraction_agent.add_tool(self.amap_tool)

self.weather_agent = SimpleAgent(name="天气查询专家", ...)
self.weather_agent.add_tool(self.amap_tool)

self.hotel_agent = SimpleAgent(name="酒店推荐专家", ...)
self.hotel_agent.add_tool(self.amap_tool)
```

`uvx amap-mcp-server`로 외부 MCP 서버를 그대로 가져와서 4개 에이전트가 공유하는 도구 버스로 쓴다. MCP의 실용적 디자인 패턴 하나를 정말 깔끔하게 보여준다. 2025~2026 시점에 MCP를 처음부터 교과서급으로 다룬 중문 자료는 사실상 이게 처음에 가깝다.

**셋째, 15장 AI 마을의 호감도 시스템.** 이건 좀 흥미롭다 못해 약간 광기 어린 디자인이다. `code/chapter15/Helloagents-AI-Town/backend/relationship_manager.py`(324줄)는 NPC와 플레이어 사이 0~100 호감도를 관리하는데, 전통 게임처럼 if/else나 룩업 테이블을 쓰지 않는다. 별도의 `AffinityAnalyzer` 에이전트가 매 대화를 JSON으로 평가한다.

```python
# code/chapter15/.../relationship_manager.py:47-103
return """你是一个情感分析专家...
【好感度变化规则】
- 赞美、感谢、请教: +3 到 +8
- 友好问候、正常交流: +1 到 +3
- 普通闲聊、中性话题: 0
- 批评、质疑、不耐烦: -3 到 -8
- 侮辱、攻击、恶意: -8 到 -15

【输出格式】(严格遵守JSON格式,不要添加任何其他文字)
{
    "should_change": true/false,
    "change_amount": -15到+10之间的整数,
    "reason": "简短说明原因(10字以内)",
    ...
}
"""
```

게임 룰을 자연어 프롬프트 안에 인라인 표로 박아 넣고 LLM이 JSON으로 평가하게 한다. 코드는 정규식으로 JSON만 추출. **모든 게임 로직이 자연어 안에 산다.** AI 네이티브 게임의 새로운 디자인 패턴 — 그리고 그 트레이드오프(모델 변동성 = 게임 일관성 변동성)를 학습자가 직접 마주하게 만드는 케이스. 좋아하든 싫어하든 한 번은 봐야 할 코드다.

**넷째, 풀스택 캡스톤의 빌드 도구 다양성.** 13장은 Vue + FastAPI + 평범한 pip, 14장은 `uv` + `pyproject.toml` + Lance Martin의 langgraph-deep-research 포크(MIT), 15장은 Godot 4 + GDScript + FastAPI. 세 캡스톤이 빌드 도구도 언어도 다 다르다. 학습 부담은 분명히 늘지만, 한 책 안에서 모던 Python 패키징, 게임 엔진 연동, SPA 프론트엔드까지 다 만져보게 한다는 야심은 좀 박수칠 만하다.

## 비판·한계

칭찬만 늘어놓으면 진짜 모습이 안 보인다. 직접 코드를 펼쳐보면서 마음에 걸린 게 몇 가지 있다.

**가장 큰 건 7장의 "프레임워크 만들기" 라벨이 살짝 어긋난다는 점.** 챕터 제목은 「构建你的Agent框架(자네 자신의 Agent 프레임워크 만들기)」지만, `code/chapter7/my_simple_agent.py`(252줄)의 첫 줄은 이렇게 시작한다.

```python
# code/chapter7/my_simple_agent.py:1-24
from hello_agents import SimpleAgent, HelloAgentsLLM, Config, Message
import re

class MySimpleAgent(SimpleAgent):
    """
    重写的简单对话Agent
    展示如何基于框架基类构建自定义Agent
    """
    def __init__(self, name, llm, ...):
        super().__init__(name, llm, system_prompt, config)
        ...
```

`from hello_agents import ...` 해서 `super().__init__()`을 호출한다. 즉 **"처음부터 만든다"고 표방한 챕터가 실제로는 외부 PyPI 패키지 `hello-agents==0.2.9`를 서브클래싱하는 예제**다. 책에서 만드는 자체 프레임워크 `HelloAgents` 본체 코드는 이 레포가 아니라 [jjyaoao/helloagents](https://github.com/jjyaoao/helloagents) 별도 레포에 있다. 학습자 입장에선 "책 한 권, 레포 두 개"가 된다. 챕터 7부터 16까지 거의 모든 코드(85개 이상 파일)가 `from hello_agents import ...`로 시작하니까, 진짜 구현을 보려면 두 번째 레포를 따로 클론해야 한다. 책 본문이 만든 기대와 실제 코드의 갭이 큰 부분이다.

**둘째, 단일 메인테이너 의존도.** 얕은 클론 기준 325개 커밋 중 jjyaoao가 168개(절반 이상)다. 최근 30일 기준으로도 38건 중 23건(60%)이 한 사람한테서 나왔다. 외부 PR은 주로 `Co-creation-projects/` 학습자 졸업작품 머지 형태. 본문과 `code/` 메인 라인은 사실상 1인 운영이다. 그 한 사람이 별도 레포에서 자체 프레임워크 `HelloAgents` 본체까지 동시에 굴린다는 점에서 의존도는 더 크다. Bus factor 1이라는 흔한 비판이 여기엔 정말 적용된다.

**셋째, 코드 품질의 챕터별 편차.** 챕터 4는 외부 의존 거의 없이 깔끔한데, 챕터 6의 AutoGen 데모 폴더 `requirements.txt`에는 이런 게 박혀 있다.

```
# code/chapter6/AutoGenDemo/requirements.txt
autogen-agentchat
autogen-ext[openai,azure]
...
asyncio          ← stdlib 모듈, pip 설치 불필요
dotenv           ← 잘못된 패키지명 (실제: python-dotenv)
```

`asyncio`는 표준 라이브러리니까 설치할 게 없고, `dotenv`는 정확한 패키지명이 아니다(`python-dotenv`가 맞다). 학습자가 그대로 `pip install -r requirements.txt` 하면 후자에서 막힌다. 챕터 11(Agentic-RL)은 더 신경 쓰이는데, `05_grpo_training.py`의 실제 학습 호출이 `# result = tool.run(config)` 식으로 **주석 처리**되어 있다. 설정 dict만 보고 가라는 거다. GPU 없는 학습자를 배려한 디자인이지만, "직접 학습 돌려본다"는 캡스톤 가치는 사실상 없다. 챕터 13의 `amap_service.py`도 4군데에 `# TODO: 解析实际的POI数据`만 남기고 빈 리스트를 반환한다. 실제 파싱은 같은 챕터 다른 파일에서 진행되니, 학습자가 "이 파일은 왜 있지?" 하게 만든다.

**넷째, 라이선스 갈등.** 루트는 CC BY-NC-SA 4.0(비상업 + 동일조건)인데, `code/chapter14/helloagents-deepresearch/backend/pyproject.toml`은 `license = { text = "MIT" }` + `authors = [{ name = "Lance Martin" }]`이다. 14장 코드는 langgraph-deep-research를 포크해 와서 원본 MIT 라이선스를 유지했다. 정면 충돌은 아니지만, "이 레포의 코드를 가져다 쓸 때 어느 라이선스를 따라야 하나"가 한 번에 안 잡힌다. 그리고 더 중요한 건 **루트의 CC BY-NC-SA 4.0이 상업 사용을 명시적으로 차단한다**는 점. "라이브러리로 가져다 쓰세요" 같은 발상은 처음부터 막혀 있다. 이건 책이지 라이브러리가 아니다.

**다섯째, 외부 평가의 한계 인정.** [CSDN의 한 분석 글](https://blog.csdn.net/y525698136/article/details/155844769)에서 짚은 한계가 정확하다. (1) 엔터프라이즈 능력 부족 — 경량 설계라 고동시성·고가용성·멀티테넌트 같은 대형 상업 요구를 못 받친다. (2) 툴 생태계가 LangChain 등 성숙 프레임워크 대비 얕고, 직접 확장해야 한다. (3) 모델 적응 깊이 제한 — 일부 오픈소스 대형 모델 적응이 충분치 않고, 복잡한 파인튜닝·통합은 추가 작업이 필요. 결론도 분명하다: "학습·교육용으로는 동급 자원 가치 초월. 대규모 상업 응용용은 아님."

**여섯째, 영어권 침투 얕음.** README_EN이 있고 docs 영어 버전이 분리돼 있지만, **코드 내부 주석은 거의 100% 중국어**. 학습 스크립트마다 `print(f"🤔 思考: ...")`, `print(f"🎬 行动: ...")` 같은 식으로 한자가 깔린다. [DEV.to에 한 편의 영어 소개글](https://dev.to/wonderlab/one-open-source-project-a-day-61-hello-agents-a-practical-guide-to-building-ai-native-agents-224k)이 올라와 있고 [DeepWiki에도 인덱싱](https://deepwiki.com/datawhalechina/hello-agents)됐지만, Hacker News나 Reddit의 메이저 토론은 검색되지 않는다. 4.8만 스타 중 상당 비중이 중문권 학습자라는 추정이 자연스럽다. 영어권에서 검색해 들어온 사람은 코드 따라가다가 한자 주석에서 한 번 멈춘다.

## 결론

**잘 맞는 사람.** 첫째, 파이썬과 OpenAI API 호출은 해봤는데 Agent를 "사용자"가 아니라 "구축자"로 보고 싶은 사람. 둘째, LangChain/LlamaIndex 사용법은 알겠는데 그 안에서 뭐가 도는지 한 번 직접 짜보고 싶은 first-principles 학습자. 셋째, MCP·A2A·Agentic-RL 같은 최신 토픽을 한 권 안에서 둘러보고 어디로 더 파고들지 정하고 싶은 사람. 넷째, 중문 독해가 되거나 번역 도구로 한자 코멘트를 받아들일 수 있는 사람. 그리고 졸업 프로젝트나 포트폴리오용 풀스택 캡스톤을 찾는 학생.

**피해야 할 사람.** 첫째, 이걸 프로덕션 라이브러리로 가져다 쓰려는 회사. 라이선스(CC BY-NC-SA)가 상업 사용을 명시적으로 차단하고, 코드 품질도 곳곳에 학습용 단순화·TODO·주석 처리가 박혀 있다. 둘째, 중국어가 전혀 안 되고 번역 도구도 안 쓸 영어권 학습자. docs는 영어가 있지만 코드 주석은 아니다. 셋째, "Agent 프레임워크 비교 한 권" 정도로 가볍게 보고 싶은 사람 — 16장은 가볍지 않다.

직접 코드를 펼쳐보면서 가장 인상에 남은 건 챕터 4의 ReAct 99줄과 챕터 15의 호감도 시스템이다. 전자는 "프레임워크 없으면 이렇게 짧다"를 보여주고, 후자는 "프레임워크 없이도 게임 룰을 LLM에 통째로 위임할 수 있다"는 좀 미친 디자인을 보여준다. 같은 책에서 정반대 방향의 두 시연이 나란히 있다는 게 좀 좋았다. 한 달 만에 1만 스타가 박힌 데는 Datawhale 디스트리뷰션의 힘이 컸겠지만, 본문도 그 힘에 부끄럽지 않은 작품이라는 인상. 다음에 책을 펼친다면 챕터 7의 자체 프레임워크 본체를 보러 [jjyaoao/helloagents](https://github.com/jjyaoao/helloagents) 쪽으로 한 번 더 다이빙해볼 생각이다.
