---
repo: influxdata/telegraf
url: https://github.com/influxdata/telegraf
language: go
date: 2026-05-15
---

# 11년차 Go 단일 바이너리 — Telegraf는 어떻게 427개 플러그인을 한 채널 위에 얹었나

## 도입

홈랩 Raspberry Pi 모니터링 글, 공장 PLC 데이터 수집 가이드, HPC 클러스터 메트릭 워크숍 슬라이드 — 분야가 다른 이 셋이 검색에서 자꾸 같은 단어로 끝난다. "Telegraf로 받아서 InfluxDB로 보냈다." 도구 이름이 너무 자주 보이면 오히려 의심이 든다. 이게 진짜 좋은 도구라서 살아남은 건지, 아니면 그냥 11년 전부터 거기 있었어서 다들 관성으로 쓰는 건지.

이번 주 GitHub 트렌딩에 다시 올라온 [influxdata/telegraf](https://github.com/influxdata/telegraf)를 며칠 들여다봤다. 별 17,313개에 기여자 1,528명, 릴리스 214개, 최근 30일 머지된 PR 157개. 숫자만 보면 "안정적인 늙은 OSS"가 맞는데, 코드를 열어보니 인상이 좀 다르다. 4단 파이프라인을 채널로 깔끔하게 끊고, 427개 플러그인을 컴파일 타임에 자기 자신을 등록하게 만든 self-registration 패턴이 11년이 지나도 흐트러짐이 없다. 그리고 2026년 3월에 발표된 [Telegraf Enterprise 베타](https://www.influxdata.com/blog/telegraf-enterprise-beta/)가 트렌딩 곡선을 다시 끌어올렸다. 이 글은 그 두 면을 같이 본다 — 늙었지만 무뎌지지 않은 코드와, 다시 상용 카드를 꺼내 든 회사의 의도.

## 정체

한 줄로 말하면 이렇다. **InfluxData가 만든 Go 단일 바이너리 메트릭 수집 에이전트, 입력 248개·출력 71개·프로세서 39개·집계 12개·파서 26개·시리얼라이저 19개·시크릿스토어 12개 — 합쳐 427개 플러그인을 들고 어디든 던져넣으면 도는 OS-agnostic 콜렉터.** README는 "300+"라고 겸손하게 써 놨는데 실제로 `ls plugins/{category} | wc -l` 돌려보면 합이 427이다. 디렉토리가 곧 플러그인이라 셈이 정직하다.

원래는 [TICK 스택](https://www.influxdata.com/blog/introduction-to-influxdatas-influxdb-and-tick-stack/) — Telegraf + InfluxDB + Chronograf + Kapacitor — 의 첫 글자였다. 시계열 DB로 자리 잡으려면 "데이터를 어떻게 넣냐"가 가장 큰 마찰이라, InfluxData가 1st-party 수집기로 만든 게 시작. 만든 회사 자체는 [Paul Dix](https://www.crunchbase.com/person/paul-dix-2)가 2012년에 세운 InfluxData(구 InfluxDB Inc.)고, Telegraf는 회사 차원에서 운영해 단일 메인테이너가 따로 박혀 있진 않다. 라이선스는 MIT, 저작권은 2015~2025 InfluxData Inc. 명의.

지금은 출신을 잊을 만큼 멀리 갔다. `outputs/` 71개 안에 InfluxDB는 한 자리고, 나머지는 Prometheus·Kafka·Datadog·OpenTelemetry·ClickHouse·Splunk·Wavefront·Lightstep·Uptrace 같은 백엔드들이 줄줄이 붙는다. [Splunk Observability Cloud](https://help.splunk.com/en/splunk-observability-cloud/manage-data/available-data-sources/supported-integrations-in-splunk-observability-cloud/opentelemetry-other-ingestion-methods/monitor-services-with-telegraf-input-plugins-and-opentelemetry), [VMware Aria(Wavefront)](https://docs.wavefront.com/telegraf_details.html), [Broadcom DX OpenExplore](https://techdocs.broadcom.com/us/en/ca-enterprise-software/it-operations-management/dx-openexplore/saas/telegraf_details.html)가 다들 "외부 메트릭은 Telegraf로 받아라"고 공식 문서를 박아둔 상태다. InfluxDB 종속 도구가 아니라 "벤더 중립 메트릭 게이트웨이" 자리에 들어앉은 지 한참 됐다.

## 아키텍처·구현 디테일

코드를 열면 의외로 깔끔하다. 루트에 `input.go`, `output.go`, `processor.go`, `aggregator.go`, `accumulator.go`, `metric.go`, `secretstore.go`처럼 9개의 인터페이스 파일만 떠 있고, 합쳐 700줄 남짓이다. 모든 외부/내부 플러그인이 이 한 패키지를 import해 인터페이스를 구현한다. Java 식으로 거대한 추상 클래스를 만든 게 아니라 Go의 덕 타이핑을 끝까지 활용해서 — `Input` 인터페이스가 단 두 메서드(`SampleConfig() string`, `Gather(Accumulator) error`)뿐이다. 추가 능력은 모두 옵셔널 인터페이스로 분리됐다. `Initializer.Init()`, `ServiceInput.Start()/Stop()`, `StatefulPlugin.GetState()/SetState()`, `ProbePlugin.Probe()`, `PluginWithID.ID()`. 248개 입력이 같은 두 메서드 계약만 따르고 필요한 능력만 옵트인하는 구조.

### 채널로 끊은 4단 파이프라인

`agent/agent.go`(1,210 LOC)가 메인 루프다. 흐름은 단순하다. Inputs → Processors → Aggregator → Outputs, 각 단의 경계가 채널이고 단마다 별도 goroutine이다. 재밌는 건 **채널을 출력에서 입력 방향으로 거꾸로 만든다**는 점이다.

```go
// agent/agent.go 첫머리 ASCII 다이어그램 발췌 (agent/agent.go:38-103)
// inputUnit is a group of input plugins and the shared channel they write to.
//
// ┌───────┐
// │ Input │───┐
// └───────┘   │
// ┌───────┐   │     ______
// │ Input │───┼──▶ ()_____)
// └───────┘   │
type inputUnit struct {
    dst    chan<- telegraf.Metric
    inputs []*models.RunningInput
}
```

타입 정의 위에 채널 토폴로지를 그림으로 박아 놨다. `inputUnit`/`processorUnit`/`aggregatorUnit`/`outputUnit` 네 타입 모두 이런 식으로 주석 다이어그램을 끼고 있고, 마지막 outputUnit에는 1:N fan-out까지 그려져 있다. 신규 컨트리뷰터가 그림 한번 훑고 코드를 읽는 동선을 의도한 게 보인다. README보다 코드 주석이 더 풍부하게 설계 의도를 나르는 드문 사례.

### 컴파일 타임 self-registration — 5줄짜리 두 패턴

427개 플러그인이 어떻게 자동으로 붙냐. 답은 두 패턴, 합쳐 5줄이다.

```go
// plugins/inputs/all/cpu.go (전체 5줄)
//go:build !custom || inputs || inputs.cpu

package all

import _ "github.com/influxdata/telegraf/plugins/inputs/cpu" // register plugin
```

```go
// plugins/inputs/cpu/cpu.go:159-167
func init() {
    inputs.Add("cpu", func() telegraf.Input {
        return &CPU{
            PerCPU:   true,
            TotalCPU: true,
            ps:       psutil.NewSystemPS(),
        }
    })
}
```

레지스트리 본체는 14줄짜리 글로벌 맵(`plugins/inputs/registry.go`)이 전부다. `var Inputs = make(map[string]Creator)`, `func Add(name string, creator Creator)`. 새 입력을 추가하려면 (1) 두 메서드 인터페이스 구현 (2) `init()`에서 `Add` 호출 (3) `plugins/inputs/all/myinput.go`에 빌드 태그+빈 import 한 줄. 끝. reflect도 코드 생성도 DI 컨테이너도 없다.

여기서 머리가 좋아지는 건 **빌드 태그 한 줄**이다. `!custom`이 기본이라 평소엔 모든 플러그인이 들어가지만, `-tags=custom,inputs.cpu,outputs.influxdb_v2`로 빌드하면 그 둘만 들어간다. `tools/custom_builder/`가 사용자 .conf를 읽어 필요한 태그를 자동 생성한다. → 동적 로딩 없이도 selective compilation으로 바이너리 크기와 공격 표면을 줄이는 길이 마련돼 있다. "거대한 단일 바이너리"와 "필요한 것만 골라 컴파일하는 슬림 바이너리" 두 세계를 한 패턴으로 동시에 만족시킨 셈.

### Drift-free jitter — ticker가 떠밀려 나가지 않게

100대의 에이전트가 동시에 :00초에 모니터링 백엔드를 두드리면 그게 thundering herd다. 그래서 ticker에 jitter를 주는데, 순진하게 짜면 다음 schedule에 jitter가 누적돼 평균 jitter/2씩 시간선이 어긋난다. `internal/clock/ticker.go`가 그걸 푼 작은 보석.

```go
// internal/clock/ticker.go:88-100
case ts := <-timer.C:
    // Compute the next scheduled interval by adding the interval and
    // randomizing the timing with the given jitter (if any). Note, we
    // need to remember the next scheduling without adding the ticker
    // to avoid drifting of the ticks by jitter/2 on average!
    t.schedule = t.schedule.Add(t.interval)
    timer.Reset(t.clk.Until(t.schedule) + internal.RandomDuration(t.jitter))

    // Fire our event in a non-blocking fashion to avoid blocking the
    // ticker if the agent code did not read the channel yet
    select {
    case t.C <- ts:
    default:
    }
```

`schedule`은 jitter 없는 깨끗한 시간선으로 유지하고, 실제 wait만 jitter로 흔든다. 송신은 `select { case t.C <- ts: default: }`라 컨슈머가 느려도 ticker goroutine이 절대 막히지 않는다. 한 파일 안에서 분산 환경의 두 요구(thundering herd 방지 + 장기 드리프트 방지)를 동시에 푼 게 정말 마음에 든다.

### Backpressure 가드 — if를 합치면 안 되는 이유

`models/running_output.go:282-298`도 같은 결의 미세 디테일이다.

```go
func (r *RunningOutput) triggerBatchCheck() {
    // Make sure we trigger another batch-ready event in case we do have more
    // metrics than the batch-size in the buffer. We guard this trigger to not
    // be issued if a write is already ongoing to avoid event storms when adding
    // new metrics during write.
    if r.buffer.Len() >= r.MetricBatchSize && !r.lastWriteFailed.Load() {
        // Please note: We cannot merge this if into the one above because then
        // the compare-and-swap condition would always be evaluated and the
        // swap happens unconditionally from the buffer fullness.
        if r.writeInFlight.CompareAndSwap(false, true) {
            select {
            case r.BatchReady <- time.Now():
            default:
            }
        }
    }
}
```

3중 가드 — 버퍼가 배치 크기에 찼는지, 진행 중 쓰기가 없는지(atomic CAS), BatchReady 채널이 가득 차도 non-blocking. "이 if를 합치지 말라, CAS의 부수효과가 항상 평가된다"는 주석은 미래의 누군가가 리팩터링하다가 발 디딜 곳을 미리 표시해 둔 거다. 11년 운영하면서 한 번은 누군가 이걸 합쳤다가 사고 친 적이 있을 것 같은 향이 난다.

## 박수 포인트

**1) 시크릿을 1급 시민으로 다룬다.** `config/secret.go`가 `awnumar/memguard`로 메모리 보호된 secret container를 두고, TOML 안에서 `@{store_id:key}` 문법(정규식 `@\{(\w+:\w+)\}`, `config/secret.go:25`)으로 시크릿을 참조한다. 더 좋은 건 **dynamic vs static 구분**이다. resolver가 `dynamic=true`를 반환하면 매번 평가, `false`면 즉시 치환해 메모리에 잠근다. TOTP 같은 시간 기반 시크릿을 위한 hook까지 같은 인터페이스로 푼 게 영리하다. 12개의 secretstore(HashiCorp Vault, OS keyring, AWS Secrets Manager, …)가 모두 한 인터페이스 뒤에 숨고, 비교 함수는 timing attack 방어용 constant-time(`config/secret.go:175-185`), main.go 마지막 줄은 `defer memguard.Purge()`(`cmd/telegraf/main.go:438`)로 종료 시 모든 보호 메모리를 와이프. 11년 된 OSS의 시크릿 처리치고는 진심이다.

**2) Selfstat — 에이전트가 자기 자신을 메트릭으로 본다.** `models/running_input.go`의 모든 단마다 `metrics_gathered`, `gather_errors`, `gather_timeouts`, `write_errors`, `metrics_dropped` 같은 selfstat 카운터가 박혀 있다. 모니터링 에이전트가 정작 자기 헬스를 모르는 사례가 흔한데, Telegraf는 운영 가시성을 코드 디자인 시점에 끌어들였다. 이게 있으니 "왜 메트릭이 일부 빠지냐"를 추적할 수 있다.

**3) Migration 시스템이 데이터다.** `migrations/` 안에 `inputs_aerospike`, `inputs_cassandra`, `inputs_cisco_telemetry_gnmi` 같은 80+ 서브디렉토리가 있다. 구버전 .conf를 신버전으로 자동 변환하는 코드를 플러그인별로 별도 보관. Deprecation 정보도 `plugins/inputs/deprecations.go`의 `map[string]telegraf.DeprecationInfo`에 `Since`/`RemovalIn`/`Notice`로 데이터화돼 있고, `--deprecation-list` 플래그가 그걸 사람이 읽는 형태로 출력한다. **기능 폐기를 한 번 적고 끝내는 게 아니라 코드 안의 데이터 구조로 들고 있는 게** 11년 운영의 무게다.

**4) StartupErrorBehavior — 4가지 모드의 헬스체크.** `models/running_input.go:124-182`의 `StartupErrorBehavior`가 `error`/`retry`/`ignore`/`probe` 4모드를 지원한다. probe 모드는 `Probe()` 호출까지 성공해야 진짜 등록되는 강한 헬스체크. "에이전트가 사용자 잘못된 설정 한 줄 때문에 통째로 못 뜨는" 11년치 사용자 불만을 정면으로 푼 해법이다. `internal.StartupError` 표지자 에러를 `errors.As`로 잡아 분기하는 패턴도 깔끔.

**5) 코드 안에 ASCII 다이어그램.** 위에 인용한 `agent/agent.go:38-103`의 채널 다이어그램. 별것 아닌 듯 보이지만, 신규 컨트리뷰터가 1,210줄짜리 메인 루프를 처음 만났을 때 그림 하나로 "어디부터 읽어야 하는지"가 명확해진다. 코드와 문서의 거리를 0으로 줄이는 사소하지만 강한 디테일.

## 비판·한계

**1) 427개 플러그인이 한 바이너리에 들어간다는 결합 비용.** `cmd/telegraf/main.go:20-28`이 `plugins/{inputs,outputs,parsers,processors,aggregators,secretstores,serializers}/all`을 전부 빈 import로 끌어들인다. 기본 빌드는 모든 의존성을 링크 — `go.sum`이 344KB라는 게 그 결과다. AWS·GCP·Azure·Alibaba·IBM 클라우드 SDK, Kafka·MQTT·AMQP 메시징, ClickHouse·Netezza·SAP HDB DB 드라이버가 한꺼번에 들어가 있다. CVE 하나 터지면 영향 면적이 끔찍하게 넓다. `tools/custom_builder/`가 공식 해법이긴 한데, 사용자가 그걸 실제로 쓰는지는 별개 문제. 대부분은 distribution 바이너리를 그대로 받아 쓴다.

**2) panic 한 번에 에이전트 전체가 죽는다.** `agent/agent.go:1192-1204`의 `panicRecover`가 모든 goroutine 스택을 덤프한 뒤 `log.Fatalln`으로 프로세스를 종료한다. 한 입력 플러그인의 패닉이 248개 입력 전체를 함께 죽이는 정책. supervisor(systemd 등)가 재시작해 줄 거라는 가정이 깔려 있는데, 안정 운영에선 합리적이라도 첫 패닉이 이슈 진단 전에 이미 데이터 손실을 만든다. "한 플러그인 격리"를 하지 않는 결정.

**3) Long-running stall과 buffer drop이 오래된 페인 포인트.** [issue #2870](https://github.com/influxdata/telegraf/issues/2870)에서 "1.3.0이 몇 시간 처리 후 메트릭 전송이 멈춘다"는 보고가 있었고, [NashTech 블로그](https://blog.nashtechglobal.com/troubleshooting-common-issues-in-telegraf-deployments/)는 `metric_buffer_limit`을 충분히 키우지 않으면 메트릭이 드롭된다고 정리했다. [issue #1446](https://github.com/influxdata/telegraf/issues/1446)은 partial success/failure 처리 방식이 플러그인마다 제각각이라는 일관성 문제. 디스크 기반 버퍼(`buffer_disk.go`)와 PartialWriteError 처리(`models/running_output.go:407-430`)가 그동안 많이 개선됐지만, 운영 사용자들은 여전히 튜닝 끝에 만난다.

**4) 글로벌 변수 의존과 거대한 config.go.** `inputs.Inputs`, `outputs.Outputs`, `models.GlobalMetricsGathered` 같은 글로벌 맵이 다수다. Go 표준 라이브러리도 이 스타일이긴 한데, 단위 테스트 격리·동시 실행이 어려워질 수 있고 실제로 `config.ResetSecrets()` 같은 reset 함수를 일부 두고 관리한다. 거기에 `config/config.go`가 2,061 LOC 단일 파일이라, TOML 파싱·플러그인 인스턴스화·필터·시크릿 링킹이 다 한 파일에 있다. 변경할 때마다 거대한 파일을 만지는 부담.

**5) DEPRECATED는 많지만 제거는 더디다.** `cmd/telegraf/main.go`에 `--version`/`--sample-config`/`--input-list`/`--output-list`/`--plugin-directory`까지 5개 deprecated 플래그가 살아 있다. `inputs.Deprecations` 맵에는 `httpjson` `RemovalIn: "1.30.0"` 같은 항목이 있는데 현재 1.38.4 — 약속한 제거 시점을 한참 넘긴 게 보인다. `internal/goplugin`도 동적 로딩이 사실상 비활성인데 `--plugin-directory` 플래그는 DEPRECATED 마킹만 한 채 살아 있다. 후방 호환을 우선하는 정책의 부산물이지만, 신규 사용자에게는 혼란. 최근 [issue #17484](https://github.com/influxdata/telegraf/issues/17484)에서도 "신규 사용자가 `influxdb` vs `influxdb_v2` 출력 선택에서 헷갈린다"는 UX 마찰이 보고됐다.

**6) InfluxData의 상용 푸시에 대한 우려.** [HN 23913445](https://news.ycombinator.com/item?id=23913445)에서 반복 등장한 평. "문서, DB observability, cloud 서비스를 너무 공격적으로 푸시한다"는 정서가 있다. 그리고 2026년 3월 Telegraf Enterprise 베타 발표 — Controller라는 fleet 관리 UI/API를 들고 나왔는데, 유료 Enterprise Support 모델로 간다. 무료 티어가 살아있고 OSS 기능 자체는 줄지 않았지만, "Telegraf로 시작했다가 결국 컨트롤 플레인을 사야 한다"는 Datadog Agent + Datadog 패턴이 보일 가능성은 분명히 있다. (이건 추측에 가깝다.)

## 결론

Telegraf는 **인프라/IoT 메트릭이 본업이고 시계열 DB(특히 InfluxDB) 위주 백엔드를 쓰는 팀**에 잘 맞는다. 산업 프로토콜(SNMP, Modbus, OPC-UA, MQTT) 묶음이 한 바이너리에 다 들어 있어 [공장 PLC](https://www.hivemq.com/blog/real-time-plant-floor-data-monitoring-in-a-factory-using-mqtt/)·[발전소 모니터링](https://www.intel.com/content/www/us/en/developer/articles/technical/power-plant-monitoring-with-emq-and-eii.html)·[HPC 클러스터](https://hpc.fau.de/files/2022/07/NHR-Workshop2022-Michael-Ott-Telegraf.pdf) 같은 OT/IoT 영역에서 OpenTelemetry Collector가 쉽게 못 따라오는 자리에 박혀 있다. 라즈베리파이 홈랩 모니터링도 마찬가지 — TOML 30줄로 끝나는 셋업이 OTel + 별도 receivers 조합보다 압도적으로 가볍다. 단순 인프라 메트릭만 필요하고 Grafana로 그릴 거면, 2026년에도 여전히 가성비 1순위.

반대로 **traces·logs·metrics 풀스택 관측 표준이 필요한 팀**, **벤더 중립이 절대 조건인 팀**, **복잡한 라우팅·변환 파이프라인을 모델링해야 하는 팀**은 [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)나 [Grafana Alloy](https://github.com/grafana/alloy) 쪽이 맞는다. 아예 logs 중심이면 Vector나 Fluent Bit이 더 자연스럽고, Datadog SaaS만 쓴다면 그냥 Datadog Agent. Telegraf는 "딱 박스에 맞는" 메트릭 도구라서 박스를 벗어나는 순간 다른 도구가 더 낫다.

며칠 들여다보고 가장 인상적이었던 건 코드의 톤이다. 11년 된 OSS인데 핵심 코드의 댓글 밀도가 여전히 높고, "이 if를 합치지 마라"처럼 미래의 자신과 후배 컨트리뷰터에게 말을 거는 주석이 살아 있다. ASCII 다이어그램, drift-free jitter, dynamic secret resolver, StartupErrorBehavior 4모드 — 큰 한방보단 무수한 작은 결정이 잘 익어가는 코드베이스다. Telegraf Enterprise 베타가 어디로 갈지는 좀 더 지켜볼 일이지만(Controller가 살림은 좀 더 편하게 만들 듯, 가격이 어떻게 책정되느냐가 관건), OSS 본체는 당분간 흔들릴 것 같지 않다. 다음엔 `tools/custom_builder/`로 진짜 슬림 바이너리를 한번 빌드해보고, 그다음엔 OPC-UA 입력 플러그인 코드를 좀 더 들여다보고 싶다.
