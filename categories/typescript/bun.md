---
title: "Bun은 더 이상 Zig 프로젝트가 아니다 — Claude가 6일 만에 96만 줄을 옮긴 사건 이후"
repo: "oven-sh/bun"
url: "https://github.com/oven-sh/bun"
language: "typescript"
date: "2026-05-16"
summary: "README는 여전히 'Bun is written in Zig'라고 적혀 있다. 그런데 main 브랜치를 열어보면 Rust 크레이트 200개가 굴러간다. 2026년 5월 4일 머지된 PR #30412 '​Rewrite Bun in Rust'를 중심으로, 이 정체성 충돌이 왜 일어났고 코드에선 어떻게 보이는지를 따라가봤다."
tags: [bun, javascript, typescript, rust, zig, runtime, bundler, package-manager, anthropic, claude]
---

# Bun은 더 이상 Zig 프로젝트가 아니다 — Claude가 6일 만에 96만 줄을 옮긴 사건 이후

## 도입

며칠 전 GitHub 트렌딩에 Bun이 또 올라와 있길래 "또 1.x 릴리스인가" 하고 무심코 열었다. 그런데 언어 비율 그래프가 이상했다. 내 머릿속 Bun은 분명 Zig 프로젝트인데, 첫 줄에 Rust 46.9%라고 박혀 있었다. Zig는 32.5%로 밀려 있었다.

오타인가 싶어서 `git log`를 찍어봤더니, 2026년 5월 4일에 `Rewrite Bun in Rust (#30412)`라는 머지 커밋이 있었다. 손이 잠시 멈췄다. 그러니까 — 그 사건이 진짜였구나. AI가 96만 줄짜리 시스템 코드를 옮겼다는 그 PR. 그래서 이번 글은 단순한 "Bun이 빠르다" 후기가 아니라, **"README가 거짓말을 하고 있는 레포의 정체성"** 이야기다.

## 정체

Bun을 한 줄로 설명하면 "JS/TS 런타임 + 번들러 + npm 호환 패키지 매니저 + Jest 호환 테스트 러너가 단일 바이너리에 다 들어 있는 올인원 툴체인"이다. 엔진은 V8이 아니라 WebKit의 JavaScriptCore. Node와 Deno가 둘 다 V8을 쓰는 마당에 의도적으로 다른 길을 골랐다.

만든 사람은 Jarred Sumner — 전 Stripe 엔지니어다. 2020년 여름에 Next.js+React로 브라우저 보크셀 게임을 만들다가 한 줄 고칠 때마다 **dev 서버 리로드에 45초**가 걸려서 분노에 차서 런타임 자체를 다시 짜기로 결심했다고 한다([Ihor Chyshkala, The Bun Story](https://chyshkala.com/blog/the-bun-story)). 개인 프로젝트로 시작해 2022년 7월에 베타 공개, 2023년 9월 8일에 1.0이 떨어졌고, 그동안 Oven, Inc.라는 회사가 됐다. 그리고 2026년에 **Anthropic이 회사를 인수**한 것으로 보도됐다([Programming-helper, Bun 2026 Anthropic acquisition](https://www.programming-helper.com/tech/bun-anthropic-acquisition-2026-ai-javascript-runtime)). 이 인수가 뒤에 나오는 "Claude가 코드를 다시 썼다" 사건의 배경이다.

여기서 좀 비틀린 지점이 있다. README는 지금 이 순간에도 첫 화면에 **"Bun is written in Zig"**라고 적혀 있다. 그런데 같은 레포의 `CLAUDE.md`(레포 운영 문서)는 첫 줄에 이렇게 적혀 있다.

> "This is the Bun repository — written **primarily in Rust** with C++ for JavaScriptCore integration."

같은 레포에서 자기 정체성을 다르게 말한다. README는 마케팅용 표지고, `CLAUDE.md`는 안에서 일하는 사람들이 보는 진짜 문서다. 2026-05 현재 진실은 후자에 가깝다. 통계가 그걸 말해준다 — Rust 1,433개 파일, Zig 1,290개 파일이 같은 `src/` 안에 나란히 누워 있고, `src/AGENTS.md`는 친절하게 알려준다.

> "You will see `.zig` siblings next to many `.rs` files — those are the original implementation kept as a porting reference for behavior; they are not compiled and are not where new code goes."

다시 말해 `.zig` 파일은 박물관이다. 빌드되지 않고, 배포되지 않는다. 컴파일 파이프라인이 빨아들이는 건 Rust 쪽이다. 카테고리로 따지면 Bun은 여전히 "올인원 JS 툴체인"이지만, **언어로 따지면 더 이상 Zig 프로젝트가 아니다**. 흥미로운 건 이 전환이 마이그레이션 레포로 따로 분리되지 않고, 메인 레포 안에서 두 언어가 같은 디렉토리에 동거하는 방식으로 이루어졌다는 점이다.

## 아키텍처·구현 디테일

큰 그림부터 보자. 빌드되면 결과물은 `bun-debug`(혹은 릴리스용 `bun`) 단일 실행 파일 하나다. 그 안에 세 덩어리가 합쳐져 있다.

```
bun-debug (단일 실행 파일)
   ├── libbun_rust.a    ← cargo build -p bun_bin
   ├── C++ 오브젝트 셋   ← WebKit/JSC 바인딩, bindgen 생성 코드
   └── JavaScriptCore    ← WebKit fork 정적 라이브러리
```

진입점은 `src/bun_bin/lib.rs`이고, 주석에 의도가 노골적이다.

> "`libbun_rust.a` — the Rust-port staticlib. Built by `cargo build -p bun_bin` (emitted from `scripts/build/rust.ts`) and linked into the final `bun-debug` executable by ninja's link step, occupying the slot `bun-zig.o` used to."

문장이 슬프다. "the slot `bun-zig.o` used to" — 이전엔 Zig 오브젝트가 들어가던 자리를 이제 Rust staticlib이 메운다. Zig 컴파일러는 더 이상 호출되지 않고, `build.zig`도 사라졌다. 빌드 흐름은 cargo + ninja + clang + 자체 codegen TS 스크립트의 4종 세트로 굴러간다.

데이터 흐름을 따라가보면 — JS 파일이 들어오면 `bun_resolver`가 경로를 풀고, `bun_js_parser`가 lexer+parser를 단일 패스로 굴려서 AST를 만든다. 트랜스파일 결과가 `bun_jsc::VirtualMachine`을 통해 JSC에 주입되고, JSC가 실제 실행을 맡는다. 번들 모드에서는 같은 파이프라인이 `bun_bundler::LinkerContext`를 거쳐 디스크로 떨어진다.

그리고 그 번들러 한가운데 — 내가 이 레포에서 가장 좋아하는 한 문단이 있다.

```zig
// src/bundler/ThreadPool.zig:1-6
pub const ThreadPool = struct {
    /// macOS holds an IORWLock on every file open.
    /// This causes massive contention after about 4 threads as of macOS 15.2
    /// On Windows, this seemed to be a small performance improvement.
    /// On Linux, this was a performance regression.
    /// In some benchmarks on macOS, this yielded up to a 60% performance improvement in microbenchmarks that load ~10,000 files.
    io_pool: *ThreadPoolLib,
    worker_pool: *ThreadPoolLib,
```

코드 첫 줄에 OS 3종의 측정 결과가 들어 있다. macOS만 IO 풀이 의미 있고, Windows는 살짝 좋아지고, Linux는 오히려 느려진다. 그런데도 추상화는 세 OS 모두에 동일하게 적용한다. 컴파일 타임에 플랫폼 분기 비용을 0으로 만들 수 있으니까. **"JS 툴이 느린 건 우리가 OS별 파일 시스템 동작까지 측정하지 않아서다"**라는 메시지가 코드 헤더에 그대로 박혀 있는 셈이다. 이 한 문단이 Bun이 왜 빠른지에 대한 가장 정직한 답이라고 생각했다.

## 박수 포인트

**1. tarball integrity를 매니저가 직접 잠근다.**

```zig
// src/install/extract_tarball.zig:13-46
pub inline fn run(this: *const ExtractTarball, log: *logger.Log, bytes: []const u8) !Install.ExtractData {
    if (!this.skip_verify and this.integrity.tag.isSupported()) {
        if (!this.integrity.verify(bytes)) {
            log.addErrorFmt(...);
            return error.IntegrityCheckFailed;
        }
    }
    var result = try this.extract(log, bytes);

    // Compute and store SHA-512 integrity hash for GitHub / URL / local tarballs
    // so the lockfile can pin the exact tarball content. On subsequent installs
    // the hash stored in the lockfile is forwarded via this.integrity and verified
    // above, preventing a compromised server from silently swapping the tarball.
    switch (this.resolution.tag) {
        .github, .remote_tarball, .local_tarball => {
            if (this.integrity.tag.isSupported()) result.integrity = this.integrity;
            else result.integrity = Integrity.forBytes(bytes);
        },
        else => {},
    }
    return result;
}
```

npm은 보통 registry가 보내준 `integrity` 필드만 믿고 끝낸다. 그런데 Bun은 GitHub·임의 URL·로컬 tarball처럼 출처가 mutable한 모든 케이스에 대해 **첫 설치 시점에 SHA-512를 직접 계산해서 락파일에 박아둔다**. 주석이 위협 모델까지 친절하게 적어두는데, "compromised serverから silently swapping"이라는 시나리오를 명시한 게 좋았다. supply chain 보안이 모두의 관심사가 된 시대에, "패키지 매니저가 자기 책임 영역을 명확히 잡는다"는 자세가 코드에 그대로 드러난다.

**2. comptime으로 파서 변형을 8개 만들어버린다.**

```zig
// src/js_parser/p.zig:7-30
pub fn NewParser(comptime parser_features: ParserFeatures) type {
    return NewParser_(
        parser_features.typescript,
        parser_features.jsx,
        parser_features.scan_only,
    );
}
pub fn NewParser_(
    comptime parser_feature__typescript: bool,
    comptime parser_feature__jsx: JSXTransformType,
    comptime parser_feature__scan_only: bool,
) type {
    return struct {
        pub const is_typescript_enabled = parser_feature__typescript;
        pub const is_jsx_enabled = js_parser_jsx != .none;
        pub const only_scan_imports_and_do_not_visit = parser_feature__scan_only;
        // ...
```

TS 여부, JSX 종류, scan-only 여부 — 이 세 조합의 카르테시안 곱이 컴파일 타임에 각자 별개의 함수 본체로 모노모피제이션된다. 런타임에 `if (is_typescript)` 분기 비용이 0이고, JIT가 죽은 코드를 제거해줄 거란 희망도 필요 없다. swc가 Rust generics로, esbuild가 Go 인터페이스로 비슷한 일을 하지만 Zig comptime이 가장 노골적이고 직접적이다. import 스캐너용 `.scan_only` 모드도 같은 소스에서 떨어진다 — 두 도구를 안 만들어도 된다.

**3. 락파일 청크 태그가 인간 친화적인 ASCII 캐멀케이스다.**

```zig
// src/install/lockfile/bun.lockb.zig:1-12
const Serializer = @This();
pub const version = "bun-lockfile-format-v0\n";
const header_bytes: string = "#!/usr/bin/env bun\n" ++ version;

const has_patched_dependencies_tag: u64 = @bitCast(@as([8]u8, "pAtChEdD".*));
const has_workspace_package_ids_tag: u64 = @bitCast(@as([8]u8, "wOrKsPaC".*));
const has_trusted_dependencies_tag: u64 = @bitCast(@as([8]u8, "tRuStEDd".*));
const has_empty_trusted_dependencies_tag: u64 = @bitCast(@as([8]u8, "eMpTrUsT".*));
const has_overrides_tag: u64 = @bitCast(@as([8]u8, "oVeRriDs".*));
const has_catalogs_tag: u64 = @bitCast(@as([8]u8, "cAtAlOgS".*));
const has_config_version_tag: u64 = @bitCast(@as([8]u8, "cNfGvRsN".*));
```

`hexdump`로 바이너리 락파일을 열어본 적이 있는 사람이면 안다 — 무슨 옵션 섹션인지 추측하느라 진을 빼는 게 일상이다. Bun은 그냥 사람이 읽을 수 있는 8바이트 ASCII로 태그를 두고, 그걸 `u64`로 비트캐스팅해서 단일 비교 인스트럭션으로 만든다. 새 옵션 추가도 enum이나 스키마 파일 손댈 일 없이 새 상수 한 줄 박으면 끝. 셔뱅(`#!/usr/bin/env bun`)을 헤더에 넣은 것도 `file` 명령으로 정체 식별이 되라는 배려다. 이런 작은 인간 친화 설계가 곳곳에 흩어져 있는 게 마음에 든다.

**4. DNS 프리페치를 install 시작 직후에 깐다.**

이건 코드보다 발상이 인상적이었다. `install_with_manager.zig` 초반에 보면 락파일 로드와 매니페스트 페칭이 진행되는 동안 OS가 백그라운드에서 registry 호스트 IP를 미리 캐싱하도록 DNS 쿼리를 던져둔다. 거기다 잡 풀을 한 개로 안 쓰고 7개로 쪼개놨다 — `network_resolve_batch`, `network_tarball_batch`, `patch_apply_batch`, `patch_calc_hash_batch` 같은 식으로. 작업마다 IO 특성(네트워크 vs 디스크 vs CPU)이 다르니까 한 풀에 섞으면 head-of-line blocking이 생긴다는 판단이다. 200개 deps 설치에 **bun 3초 / pnpm 14초 / yarn 21초 / npm 48초**([Pockit Tools, 2026 Package Manager Showdown](https://dev.to/pockit_tools/pnpm-vs-npm-vs-yarn-vs-bun-the-2026-package-manager-showdown-51dc))라는 차이는 이런 디테일이 누적된 결과지, 마법이 아니다.

## 비판·한계

이제 박수 친 만큼 비판도 해야 균형이 잡힌다. 솔직히 이번 글에서 가장 쓰고 싶었던 섹션은 여기다.

**1. AI가 쓴 96만 줄을 누가 책임지는가.**

이게 가장 큰 질문이다. PR #30412는 [The Register 보도](https://www.theregister.com/devops/2026/05/14/anthropics-bun-rust-rewrite-merged-at-speed-of-ai/5240381)에 따르면 **Claude가 6일 만에 96만 줄의 Zig 코드를 Rust로 포팅**했다. 테스트는 99.8% 통과. 숫자는 충격적이고, "AI가 시스템 코드를 옮길 수 있는가"라는 질문에 답이 된 사건이다. 그런데 그 PR을 인간이 어떻게 리뷰했을까? [byteiota의 분석](https://byteiota.com/bun-rust-rewrite-merged-the-13000-unsafe-block-problem/)은 머지된 코드에 **`unsafe` 블록이 13,000개** 있다는 사실을 짚는다. Rust로 옮긴 가장 큰 이유가 메모리 안전성이었는데, 그 안전성을 unsafe로 다시 풀어버린 비중이 만만치 않다는 우려다. "AI-generated code review debt"라는 새 카테고리가 진지하게 논의되기 시작했다.

Zig 진영의 반응은 더 직설적이다. [Stork.AI의 분석](https://www.stork.ai/blog/buns-rust-rewrite-the-betrayal-that-killed-zig)은 "배신"이라는 단어를 쓴다. Bun은 Zig의 깃발 프로젝트였기 때문이다. Sumner의 명시적 동기 — "use-after-free, double-free, free-path leak 같은 클래스의 버그에 지쳤다"([Eva Daily, Bun Rust Rewrite](https://www.evadaily.com/article/bun-runtime-rewrites-zig-to-rust-998-tests)) — 가 합리적임에도, 한 언어의 가장 큰 쇼케이스가 다른 언어로 갈아탔다는 사실은 Zig 생태계에 상처다.

**2. 안정성이 후퇴했다는 체감이 누적되고 있다.**

[Issue #27664 "My Thoughts on the Current State"](https://github.com/oven-sh/bun/issues/27664)는 한 사용자가 정성껏 쓴 장문의 우려 글이다. 오픈 이슈가 **약 5,000개**(실측: 5,037)에 달하고, segfault 보고가 Windows뿐 아니라 macOS·Linux에서도 늘고 있다고 지적한다. 본인이 낸 segfault PR이 문제를 실제로 못 고쳤음에도 머지된 사례, `robobun`(AI 자동 수정 봇)이 진짜 리뷰 없이 PR을 머지하는 듯한 의혹까지 적어놨다. Markdown 렌더링 같은 사이드 기능 늘리지 말고 **안정성에 집중**하라는 절절한 요청이다. AI에 의존하는 개발 워크플로가 단기 속도에선 이기지만 장기 품질에선 어떻게 될지는 아직 답이 안 나왔다.

**3. Windows 지원은 여전히 약점.**

[Issue #18715](https://github.com/oven-sh/bun/issues/18715), [#18680](https://github.com/oven-sh/bun/issues/18680), [#24379](https://github.com/oven-sh/bun/issues/24379) 같은 보고가 꾸준히 올라온다. fresh `bun install`이 극단적으로 느리고, C:\ 드라이브가 아닌 곳에 설치하면 더 느려지고, node_modules 안에 과도하게 많은 파일이 생기는 식. unix 환경과의 격차가 명확하다.

**4. npm 호환성의 마지막 2%가 핵심 패키지에 걸려 있다.**

공식 자료는 "top 1000 패키지 중 98%"라고 하는데, 남은 2%에 bcrypt, canvas 일부 구현, node-sass 엣지케이스 같은 게 들어 있다([Tech Insider, Bun vs Node.js 2026](https://tech-insider.org/bun-vs-nodejs-2026/)). 구조적 한계다 — V8 NAPI/internals에 직접 컴파일된 native addon을 JSC 위에 올리는 게 본질적으로 어렵다. Bun 1.2가 V8 Public API를 JSC 위에 에뮬레이트하는 큰 엔지니어링을 했음에도 100%엔 닿지 못했다([InfoQ, Bun 1.2](https://www.infoq.com/news/2025/04/bun-12-node-compat-postgres/)).

**5. 한국에서도 "전면 교체"보단 "부분 채택"이 현실이었다.**

여기어때는 ["한Bun 써보는 거 어때?"](https://techblog.gccompany.co.kr/%ED%95%9Cbun-%EC%8D%A8%EB%B3%B4%EB%8A%94-%EA%B1%B0-%EC%96%B4%EB%95%8C-fa3cb32ac76f) 글에서 패키지 설치 시간 약 5배 단축을 얻고도 **런타임은 Node를 유지하고 Bun은 패키지 매니저로만** 쓰는 하이브리드 패턴을 택했다. 컬리도 ["Nx에서 Bun 더 잘 사용하기"](https://helloworld.kurly.com/blog/nx-bun-migration/) 글에서 Bun 1.0.33→1.2.x + Nx 18→21 동시 마이그레이션 후 CI 13m30s→11m40s(14% 단축), 아티팩트 330MB→280MB(15% 감소)를 얻었다. 인상적인 결과지만, 그 과정에서 `yarn.lock` 동시 유지하느라 Nx의 lock 감지 충돌, 바이너리 `bun.lockb`와 webpack `generatePackageJson`의 충돌 같은 **현실적인 마이그레이션 통증**도 함께 기록해뒀다. "그냥 깔면 끝"이 아니라는 뜻이다.

**6. god object와 마이그레이션 중간 상태.**

`VirtualMachine.zig`는 4,173줄, Rust 포팅판은 6,640줄. `LinkerContext`도 비슷한 양상. JSC 콜백마다 VM 컨텍스트 전체에 접근해야 해서 분리 비용이 너무 크다는 게 이유로 보이지만, 새 기여자에겐 진입 장벽이 높다. 거기에 `bun_jsc` 크레이트는 아직 `bun_bin`에 완전히 게이트되지 않은 상태라, 현 시점의 `libbun_rust.a`는 JSC 브리지를 아직 Rust 측이 다 못 들고 있고 일부는 Zig 측 통합을 쓰는 **하이브리드**다. README가 거짓말을 하는 것까지 합치면, "전환 중"이 정말 진행형이라는 사실을 사용자가 의식해야 한다.

## 결론

**잘 맞는 사람:**
- 새 프로젝트(특히 사이드 프로젝트)를 시작하는데 `npm i`의 30초가 매번 거슬렸던 사람.
- 모노레포 CI 시간을 단축하려는 팀 — 컬리 사례처럼 패키지 매니저만 갈아끼우는 것도 충분히 의미가 있다.
- `Bun.serve`+`Bun.sql`+`Bun.s3` 스택으로 zero-config 백엔드를 빠르게 띄우고 싶은 개인.
- TinyCC가 들어 있어서 `bun:ffi`로 C 코드를 즉시 컴파일/호출하는 마법을 즐기는 사람.

**피해야 할 사람:**
- bcrypt, canvas 같은 native addon 의존이 깊은 프로덕션 서비스 — 호환성 2% 빈틈이 그쪽에 모여 있다.
- 안정성 최우선인 미션 크리티컬 서비스 — 오픈 이슈 5,000개와 AI-rewrite 후폭풍이 안정화되기까지는 시간이 더 필요해 보인다.
- Windows를 1차 개발/배포 환경으로 쓰는 팀 — unix 갭이 여전히 크다.
- AI가 작성한 시스템 코드를 신뢰하지 못하는 보수적인 조직 — 13,000개의 unsafe 블록은 작은 숫자가 아니다.

내 결론은 그래서 "당장 회사 메인 서비스를 옮기진 않겠지만, 새 사이드 프로젝트엔 한번 더 진지하게 써본다"다. 특히 패키지 매니저 측면에선 이미 충분히 검증됐다고 본다. 그리고 솔직히 — Bun의 가장 흥미로운 지점은 이제 속도가 아니라, **"AI가 96만 줄짜리 시스템 코드를 옮긴 첫 메이저 OSS 사례"**라는 역사적 위치다. 6개월 뒤, 1년 뒤에 이 13,000개의 unsafe 블록이 어떻게 정리되는지, README가 언제쯤 "written in Rust"로 갱신되는지를 지켜보는 게 가장 큰 재미가 될 것 같다. 다음에 Bun을 다시 들여다볼 땐 그 변화를 추적하는 글이 되지 않을까 싶다.
