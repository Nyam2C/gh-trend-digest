---
title: "별 5만 5천 개와 '스캠' 의혹 사이 — ruvnet/RuView, WiFi로 정말 벽 너머를 보나"
repo: "ruvnet/RuView"
url: "https://github.com/ruvnet/RuView"
language: "rust"
stars: 55102
date: "2026-05-14"
summary: "WiFi 신호로 사람을 자세까지 잡아낸다고 주장하는 RuView. 별은 55K인데 HN·Cybernews는 '스캠' 의혹 진행 중. 코드를 직접 까보니 진짜 알고리즘은 있고, 학습된 모델·실증은 없는 중간 지대였다."
tags: [rust, wifi, csi, esp32, densepose, pose-estimation, edge, controversy, ruvnet]
---

# 별 5만 5천 개와 '스캠' 의혹 사이 — ruvnet/RuView, WiFi로 정말 벽 너머를 보나

## 도입

며칠 전 GitHub 트렌딩에서 묘한 레포를 만났다. `ruvnet/RuView`. 카피는 "WiFi 신호를 카메라 없이 공간 지능으로 — 벽 너머 사람 감지, 호흡·심박, 자세 추정까지." 별 5만 5천 개, 포크 7천 3백 개. 그런데 검색 결과 상단에 함께 뜨는 게 [Cybernews의 회의 기사](https://cybernews.com/security/viral-github-project-wifi-see-through-walls/)와 ["RuView WiFi Scam 2026"](https://digitalbiztalk.com/article/ruview-wifi-wall-penetration-github-s-viral-scam-exposed) 폭로 글이었다.

별 55K짜리가 동시에 "스캠" 의심을 받는 풍경이 흥미로웠다. 한국어 리뷰도 거의 없다. 한국 독자가 처음 만났을 때 "이거 진짜야?"라고 묻는다면 어디까지가 사실이고 어디부터가 과장인지 정도는 짚어주고 싶었다. 그래서 별점·HN 댓글 말고 **코드를 직접 까봤다.** 결론부터 — RuView는 "사기"도 "혁신"도 아닌 어색한 중간 지대에 있다.

## 정체

한 줄로 — WiFi 라우터가 이미 공간에 채워둔 전파의 미세한 흐트러짐(CSI, Channel State Information)을 ESP32 메쉬로 잡아 사람·자세·호흡을 추정한다는 시스템이다. 카메라도 웨어러블도 안 쓰고 노드 한 개에 9달러짜리 ESP32-S3 한 장이면 된다는 게 카피의 핵심.

근거 없는 SF는 아니다. 2022년 CMU 로보틱스 연구소의 ["DensePose From WiFi" 논문](https://arxiv.org/abs/2301.00250)이 뿌리다. WiFi 신호의 위상·진폭을 인체 영역의 UV 좌표로 매핑하는 신경망을 학계가 먼저 만들어놨고, RuView는 그걸 ESP32 메쉬 + Rust 워크스페이스로 재구현했다고 주장한다. README가 PCK@20 같은 학계 지표를 쓰는 이유도 그래서다 — Percentage of Correct Keypoints, 정답 위치 20% 반경 안에 키포인트 추정이 들어갔는지 보는 표준 평가법. CMU 원논문이 PCK@50로 약 87%를 보고했고, RuView는 본인 입으로 "현재 프록시 라벨로 약 2.5%, 카메라 슈퍼바이즈드로 35%+ 목표"라고 적어두고 있다.

만든 사람은 Reuven Cohen(`ruvnet`). 이미 [`claude-flow`](https://github.com/ruvnet/claude-flow)·`ruflo`·`ruvector` 같은 Claude Code 멀티에이전트 도구로 영향력이 큰 메이커다. RuView는 그가 만든 생태계의 통합 플래그십이다 — 그래서 레포 루트에 `.claude/`, `.claude-flow/`, `.claude-plugin/`, `.swarm/`, `CLAUDE.md`가 코드만큼 눈에 띈다. 참고로 리브랜드 중이라 README는 "π RuView"인데 코드 안 패키지·crate·Docker 이미지는 전부 `wifi-densepose`로 남아 있다.

## 아키텍처·구현 디테일

세 층이다. 맨 아래 ESP-IDF v5.2+ C 펌웨어 약 9,972줄, 가운데 Rust 워크스페이스 22개 크레이트, 위에 Vite + Lit + TypeScript 대시보드. ESP32 메쉬가 채널 1/6/11을 TDM으로 호핑하며 100~500Hz로 CSI를 잡아 UDP로 쏘면, Rust `hardware::aggregator`가 모아 `signal::ruvsense`가 특징을 만들고, `nn::inference`가 ONNX/LibTorch/Candle 셋 중 한 백엔드로 추론한다.

특히 한 곳이 인상적이었다. `signal` 크레이트의 `tomography.rs` — 745줄에서 "벽 너머 감지"의 수학적 토대인 RF tomography를 직접 푼다. ISTA(Iterative Shrinkage-Thresholding Algorithm)로 L1 정규화 역문제를 푸는 핵심 루프다.

```rust
// v2/crates/wifi-densepose-signal/src/ruvsense/tomography.rs:244-307
// ISTA for L1 minimization: min ||Wx - y||^2 + lambda * ||x||_1
let frobenius_sq: f64 = self.weight_matrix.iter()
    .flat_map(|ws| ws.iter().map(|&(_, w)| w * w)).sum();
let step_size = 1.0 / frobenius_sq.max(1e-10);

for iter in 0..self.config.max_iterations {
    let mut gradient = vec![0.0_f64; self.n_voxels];
    for (link_idx, weights) in self.weight_matrix.iter().enumerate() {
        let predicted: f64 = weights.iter().map(|&(idx, w)| w * x[idx]).sum();
        let diff = predicted - attenuations[link_idx];
        for &(idx, w) in weights { gradient[idx] += w * diff; }
    }
    // Gradient + soft thresholding (proximal L1) + non-negativity
    for i in 0..self.n_voxels {
        let new_val = x[i] - step_size * gradient[i];
        let threshold = self.config.lambda * step_size;
        let shrunk = if new_val > threshold { new_val - threshold }
                     else if new_val < -threshold { new_val + threshold }
                     else { 0.0 };
        x[i] = shrunk.max(0.0);
    }
}
```

링크별 감쇠량을 받아 3D voxel 점유 확률을 역산하는 역문제 풀이다. Lipschitz를 Frobenius norm으로 안전하게 잡고, soft thresholding을 세 갈래 분기로 한 줄에, 음수는 0 클램핑. 외부 라이브러리 의존 없이 직접 짰고 Wilson & Patwari 2010("Radio Tomographic Imaging") 논문을 모듈 주석에 인용해뒀다. 알고리즘 교과서를 잘 옮긴 흔적이지 빈 함수가 아니다.

펌웨어 쪽도 보자. `csi_collector.c`에 박혀 있는 주석.

```c
// firmware/esp32-csi-node/main/csi_collector.c:27-45
/* Defensive fix (#232, #375, #385, #386, #390): capture NVS config fields into
 * module-local statics BEFORE wifi_init_sta() runs, because WiFi driver init
 * can corrupt g_nvs_config (confirmed on device 80:b5:4e:c1:be:b8).
 * ...causing LoadProhibited panics (observed: Core 0 panic after ~2400 callbacks). */
```

"vaporware 아니냐"는 의심에 대한 가장 강한 반박이 이런 종류의 주석이다. 실제 디바이스 MAC(80:b5:4e:c1:be:b8), 구체적 관찰("Core 0 panic after ~2400 callbacks"), 다섯 개 GitHub 이슈 번호. 누군가 정말 ESP32를 굽고 죽이고 다시 굽었다는 흔적이 한 페이지에 다 있다.

## 박수 포인트

코드 한 시간 둘러본 뒤 박수칠 만했던 부분 셋.

**1. 알고리즘 본체가 진짜 들어 있다.** 위 ISTA L1 외에도 `field_model.rs`(1,349줄)가 방을 SVD eigenstructure로 모델링하고, `pose_tracker.rs`(1,539줄)가 17-keypoint를 Kalman 필터링하고 AETHER 임베딩으로 사람 재식별(re-ID)까지 한다. 비판자의 "도킹스트링이 코드보다 길다"는 표현은 적어도 현재 트리에서는 사실이 아니다. 모듈마다 `//!` 주석으로 알고리즘 출처·논문·ADR을 인용해둔 것도 인상적이었다.

**2. 펌웨어가 진지하다.** ESP-IDF 정식 빌드 구조 + C 9,972줄, `release_bins/`에 실제 빌드 산출물 `esp32-csi-node.bin`(967KB)까지 동봉. OTA·NVS·swarm bridge·mmwave 통합이 다 들어 있다. 위 디버깅 주석을 보면 책상에서만 컴파일된 코드가 아니다.

**3. 비판을 의식한 'Trust Kill Switch'.** 이건 흥미로운 패턴이었다. 메이커가 회의론을 의식해서 "이게 mock 아니라는 걸 결정론적으로 증명하는" 코드를 따로 짜뒀다.

```python
# archive/v1/data/proof/verify.py:21-33
# The reference signal is SYNTHETIC and is used purely for pipeline
# determinism verification. The point is not that the signal is real
# -- the point is that the PIPELINE CODE is real.
#
# If someone claims "it is mocked":
#   1. Run: ./verify
#   2. If PASS: the pipeline code is the same code that produced the published hash
```

합성 데이터로 SHA-256 해시가 발행본과 일치하는지 보는 식. ADR 97개와 함께 — 입증 강박이 코드를 더 다듬게 만든 흔적이라 좀 신선했다. 사기꾼이 이렇게까지 메타 인프라를 짜진 않는다.

## 비판·한계

박수 다음엔 비판이다. 외부 비판 6개를 코드 분석가가 한 줄씩 교차 검증한 결과를 표로.

| 비판 주장 | 코드 확인 결과 | 판정 |
|----------|---------------|------|
| `rf_tomography.py`의 도킹스트링이 구현보다 길다 | 동명 파일 없음. 가장 가까운 `tomography.rs`(745줄)는 구현이 훨씬 큼 | 거짓 |
| 첫 커밋 통째, 이후엔 문서만 갱신 | 첫 두 커밋이 108만 줄로 통째 적재 사실, 그러나 이후 30일 81 커밋·36 머지 PR 다수가 코드 변경 | 부분 사실 |
| requirements.txt에 안 쓰는 라이브러리 다수 | 25개 중 약 13개(opencv, scikit-learn, scapy 등) import 0회 | 부분 사실(과장) |
| 사전학습 가중치·데이터셋 부재 | `.pt/.pth/.onnx/.safetensors` 0개 | 대부분 사실 |
| 통합 테스트가 mock뿐 | `#[test]` 2,786개지만 하드웨어 강제 테스트 0건, 모두 합성·단위 | 부분 사실 |
| 펌웨어 빌드 가능한가 | ESP-IDF 정식 구조 + 빌드 산출물 + MAC 디버깅 흔적 모두 존재 | 사실(빌드 가능) |

가장 묵직한 건 네 번째. **모델 가중치가 레포에 하나도 없다.** `*.pt`, `*.pth`, `*.onnx`, `*.safetensors` 어느 것도 없다. `data/recordings/`에 60MB짜리 CSI 캡처 두 개뿐, MM-Fi·Wi-Pose 같은 표준 학습 데이터셋도 없다. 학습 스크립트는 만 줄 단위로 있지만, 그걸 돌려 어떤 모델·어떤 데이터에서 어떤 PCK 숫자가 나왔다는 기록은 README의 "PCK@20 ≈ 2.5%" 자백 외엔 없다. 이게 코드 안에서도 그대로 보인다.

```rust
// v2/crates/wifi-densepose-train/src/model.rs:25-29
//! # No pre-trained weights
//! Weights are initialised from scratch (Kaiming uniform).
//! Pre-trained ImageNet weights are not loaded because network access
//! is not guaranteed during training runs.
```

학습 모듈이 자기 입으로 적어뒀다. 진실한 자백이지만 동시에 "사용자가 자기 환경에서 직접 학습해야 한다"는 뜻이다. 9달러 ESP32를 사도 박스에서 꺼내 사람 자세가 바로 나오는 게 아니다. CSI 모으고 라벨링하고(혹은 카메라 ground-truth 만들고) 시간 들여 학습시켜야 하고, 그래도 README 약속(PCK@20 35%)까지 갈지는 보증 없다. [CNX Software 평가](https://www.cnx-software.com/2026/03/26/ruview-project-leverages-esp32-nodes-for-presence-detection-pose-estimation-and-breathing-heart-rate-monitoring/)도 똑같이 짚었다 — "자세 추정은 플러그앤플레이가 아니고 사용자가 자기 환경에 맞춰 모델을 훈련해야 한다." Home Assistant 커뮤니티에서 [한 사용자가 한 달 전 "실제로 동작시켜본 사람 있나요"라고 올린 글](https://community.home-assistant.io/t/has-anyone-actually-gotten-ruview-wifi-densepose-to-work-in-real-world/1002053)은 지금도 답변 0개.

검증된 대안과 비교하면 위치가 더 분명해진다. [Espressif esp-csi](https://github.com/espressif/esp-csi)는 공식·검증됐지만 모션·존재 감지 한정. [ESPectre](https://github.com/francescopace/espectre)는 Home Assistant 네이티브 연동에 NBVI 알고리즘으로 F1>96%를 실측 보고하는 — "WiFi 모션 감지" 영역에서 가장 신뢰받는 오픈소스 대안이다. RuView가 차별화하려는 영역은 그 위에 얹은 "자세 추정 + 생체신호 + WASM 엣지 + Claude 생태계"인데, 정확히 그 위층의 정확도가 코드만으로 증명되지 않는다.

마지막 한 가지. **사실상 솔로 메인테이너 프로젝트다.** 커밋 99%가 `ruvnet`/`rUv` 본인이고 AI 어시스턴트 `Claude`가 9개, 외부 컨트리뷰터 4명이 1~2 커밋씩. 별 5만 5천·포크 7천 3백 규모와 외부 활동성의 갭이 크다. [HN 47305076](https://news.ycombinator.com/item?id=47305076)에서 한 사용자는 메이커를 "현대판 Kip Kay"(수십 년 전 클릭베이트 사기 영상의 마스터)에 비유했다. 거친 표현이지만 메이커의 다른 레포 [`ruflo`도 stub 구현·공급망 보안 의혹](https://github.com/ruvnet/ruflo/issues/1482)을 받는 패턴까지 보면 회의를 그냥 무시하긴 어렵다.

## 결론

**잘 맞을 사람:** CSI 센싱·RF tomography 알고리즘을 연구·학습 목적으로 보고 싶은 사람(ISTA L1, SVD, Kalman이 의존성 거의 없이 짜여 읽기 좋다), ESP-IDF + Rust 22크레이트 워크스페이스 구조를 참고하고 싶은 사람, Claude Code 플러그인 워크플로 예제가 필요한 사람.

**피해야 할 사람:** 지금 당장 "방에 누가 있나"를 프로덕션에 넣고 싶은 사람(ESPectre가 정답), 자세 추정·생체신호를 신뢰성 있게 뽑아야 하는 의료·돌봄·보안 분야(PCK@20 35%는 아직 목표일 뿐 사전학습 가중치도 없다), 9달러 ESP32 박스 열면 사람 자세가 나올 줄 알았던 사람.

내 생각엔 가장 정확한 프레임은 **"진짜인가 가짜인가"가 아니라 "연구 코드인가 제품인가"** 다. 코드는 진지한 연구 코드다 — 누군가 ESP32를 진짜로 굽고 죽이고, 알고리즘을 의존성 없이 짜고, ADR 97개로 기록하고, 결정론적 입증까지 만들었다. 그러나 5만 5천 별짜리 제품으로서는 비어 있는 게 많다. 학습된 모델 없고, 실증 데모 영상 없고, 외부 사용 사례 보고 없고, 통합 테스트는 다 mock. 별 55K의 정체는 결국 **메이커 자본 × SF급 카피 × 논쟁 자체의 추진력**으로 보인다.

다음에 볼 건 두 가지. ADR-079("camera-supervised PCK@20 35%+ 목표")의 평가 단계가 끝나 측정 숫자가 공개되는 시점, 그리고 외부 사용자의 end-to-end 동작 영상이 처음 올라오는 시점. 둘 중 하나라도 일어나면 다시 글을 쓸 것 같다. 그전까지 RuView는 진지하게 짠 연구 코드이자 아직 약속을 지키지 못한 트렌딩 별, 그 사이 어딘가에 있다.

---

**참고:** [원본 레포](https://github.com/ruvnet/RuView) · [CMU 원논문](https://arxiv.org/abs/2301.00250) · [CNX Software 균형 분석](https://www.cnx-software.com/2026/03/26/ruview-project-leverages-esp32-nodes-for-presence-detection-pose-estimation-and-breathing-heart-rate-monitoring/) · [Sarah Chen 비판](https://digitalbiztalk.com/article/ruview-wifi-wall-penetration-github-s-viral-scam-exposed) · [Cybernews](https://cybernews.com/security/viral-github-project-wifi-see-through-walls/) · [HN 47305076](https://news.ycombinator.com/item?id=47305076) · [ESPectre 대안](https://github.com/francescopace/espectre)
