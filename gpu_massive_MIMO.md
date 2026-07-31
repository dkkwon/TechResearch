좋은 질문입니다. **Table 1. "Value of GPU by RAN workload"는 이 기사의 핵심 표**입니다. 이 표가 설명하는 것은 단순히 "GPU가 빠르다"가 아니라,

> **RAN의 어떤 알고리즘들이 계산량 제한 때문에 단순화되어 왔으며, GPU를 사용하면 어떤 부분을 더 고도화하여 스펙트럼 효율을 높일 수 있는가**

를 정리한 것입니다. ([NVIDIA Developer][1])

표 내용을 RAN 개발 관점에서 풀어서 설명하겠습니다.

---

# Table 1 전체 구조

NVIDIA는 각 RAN workload를 다음 3가지 관점으로 설명합니다. ([NVIDIA Developer][1])

| 항목                          | 의미          |
| --------------------------- | ----------- |
| RAN workload                | 어떤 RAN 기능인가 |
| Key compute characteristics | 왜 계산량이 큰가   |
| Why GPUs matter             | GPU가 왜 필요한가 |

대상 workload는 6개입니다.

1. MU-MIMO UE Pairing
2. Beamforming & Precoding
3. DRL Link Adaptation
4. Channel Estimation
5. Scheduling
6. Neural Receiver

---

# 1. MU-MIMO UE Pairing

## 기존 문제

Massive MIMO에서는 많은 UE 중에서 어떤 UE들을 동시에 묶을지가 매우 중요합니다.

예:

```
100 UE 후보

↓

동시에 전송할 16 UE 선택

↓

MU-MIMO Group 생성
```

해야 합니다.

좋은 조합:

```
UE1  방향 10°
UE2  방향 90°
UE3  방향 180°
```

→ 서로 간섭 적음

나쁜 조합:

```
UE1
UE2
UE3

모두 비슷한 방향
```

→ 간섭 증가

---

## 계산량

최적화하려면

```
UE 후보 N개

↓

가능한 조합 탐색

↓

Channel Orthogonality 계산

↓

Throughput 예측

↓

최적 Group 선택
```

이 필요합니다.

조합 문제가 발생합니다.

---

## GPU 가치

기존:

```
CPU

↓

제한된 후보만 검색

↓

Heuristic
```

GPU:

```
GPU

↓

많은 후보 병렬 평가

↓

AI 기반 UE Pairing
```

가능합니다.

즉:

> GPU는 더 많은 UE 조합을 실시간 탐색하여 MU-MIMO 효율을 높인다.

입니다. ([NVIDIA Developer][1])

---

# 2. Beamforming & Precoding ⭐ 가장 중요

Massive MIMO에서 GPU 필요성을 설명할 때 가장 핵심입니다.

---

## 기존 방식

대표적으로:

```
Channel Matrix H

↓

ZF / rZF

↓

Precoding Weight
```

를 계산합니다.

하지만 계산량 때문에 단순화합니다.

---

## GPU 사용 시

더 복잡한 모델 가능:

```
Channel 정보

+
UE 위치

+
Interference

↓

AI Beamforming

↓

최적 Weight 생성
```

---

예를 들어 기사에서는:

64T64R

16 UE

2 Layer/UE

환경에서:

기존 rZF:

```
272M FLOPs
```

AI Beamforming:

```
2.58B FLOPs
```

약 10배 계산량 증가가 필요하지만,

결과:

* 16 Layer: 약 1.28배 throughput 증가
* 32 Layer: 약 1.62배 throughput 증가

가능하다고 설명합니다. ([NVIDIA Developer][1])

---

## GPU가 필요한 이유

핵심은:

```
더 좋은 알고리즘

↓

더 많은 계산량

↓

GPU가 처리
```

입니다.

즉:

> GPU는 기존 Beamforming을 빠르게 하는 것이 아니라, 기존에는 계산량 때문에 사용할 수 없었던 더 좋은 Beamforming 알고리즘을 가능하게 한다.

입니다.

---

# 3. DRL Link Adaptation

## 기존 방식

현재 대부분:

```
CQI

↓

OLLA

↓

MCS 선택
```

입니다.

규칙 기반입니다.

---

## AI 방식

입력:

```
CQI History

ACK/NACK

Channel 변화

Mobility

Interference
```

↓

DRL Model

↓

최적 MCS 선택

---

장점:

* Cell Edge 성능 개선
* 빠른 채널 변화 대응
* BLER 최적화

---

문제:

AI 모델이 커질수록

```
Inference 계산량 증가
```

합니다.

---

GPU 가치:

CPU:

```
작은 모델

낮은 정확도
```

GPU:

```
큰 모델

Batch inference

낮은 latency
```

가능.

기사에서는 DRL Link Adaptation이 기존 OLLA 대비 약 1.3배 throughput 향상을 보여준다고 설명합니다. ([NVIDIA Developer][1])

---

# 4. Channel Estimation

Massive MIMO에서 매우 중요한 부분입니다.

---

## 기존

Channel estimation:

```
SRS / DMRS

↓

Channel Matrix H 계산
```

입니다.

안테나 증가:

```
64T64R

↓

128T128R

↓

512T512R
```

하면 계산량 증가.

---

## GPU 가치

더 많은 정보를 사용할 수 있습니다.

예:

기존:

```
현재 Pilot만 사용
```

GPU:

```
Pilot

+
과거 Channel

+
공간 정보

+
AI 모델
```

사용 가능.

결과:

* 더 정확한 CSI
* Pilot overhead 감소
* 더 높은 Layer 가능

입니다. ([NVIDIA Developer][1])

---

# 5. Scheduling

## 기존 Scheduler

CPU 기반:

```
UE 선택

↓

PRB 할당

↓

Fairness 계산
```

입니다.

---

하지만 미래:

```
Multi-cell

+
MU-MIMO

+
Beam 정보

+
Traffic prediction
```

까지 고려해야 합니다.

---

문제:

조합 폭발

---

GPU:

```
많은 후보 평가

병렬 최적화

AI Scheduler
```

가능.

즉:

> GPU는 Scheduler의 검색 공간을 확장한다.

입니다. ([NVIDIA Developer][1])

---

# 6. Neural Receiver

가장 6G에 가까운 항목입니다.

---

기존 Receiver:

```
FFT

↓

Channel Equalization

↓

Detection
```

---

AI Receiver:

```
IQ Sample

↓

Neural Network

↓

Detection
```

입니다.

---

문제:

Waveform 수준 AI는

매 샘플마다

대량 Tensor 계산 필요.

CPU:

불가능

GPU:

가능

이라는 논리입니다. ([NVIDIA Developer][1])

---

# 표를 한 장으로 재정리하면

| Workload           | 기존 한계          | GPU가 제공하는 것         |
| ------------------ | -------------- | ------------------- |
| MU-MIMO Pairing    | 제한된 UE 검색      | 대규모 조합 탐색           |
| Beamforming        | 단순 Beam Weight | 고복잡도 AI Beamforming |
| Link Adaptation    | 규칙 기반 OLLA     | 대형 AI 모델            |
| Channel Estimation | 제한된 CSI        | 더 많은 정보 활용          |
| Scheduling         | CPU 계산 한계      | Multi-cell 최적화      |
| Neural Receiver    | CPU 불가능        | Waveform AI 처리      |

---

# 이 표가 말하는 NVIDIA의 전략

중요한 것은 **"PHY를 GPU로 옮긴다"가 최종 목표가 아닙니다.**

더 큰 그림은:

기존:

```
계산량 제한

↓

가능한 알고리즘 선택
```

입니다.

---

AI-RAN:

```
최고 성능 알고리즘 선택

↓

필요한 계산량 GPU로 확보
```

입니다.

즉 NVIDIA가 말하는 방향은:

> **Compute-constrained RAN → Algorithm-first RAN**

으로 전환하자는 것입니다. ([NVIDIA Developer][1])

---

## 앞서 논의한 16-Layer MU-MIMO와 연결하면

이 표의 관계는 다음과 같습니다.

```
GPU
 |
 +-- Beamforming 계산 증가 가능
 |
 +-- UE Pairing 고도화 가능
 |
 +-- Channel Estimation 개선
 |
 +-- Scheduler 최적화
 |
 v
더 많은 Layer 지원

4 Layer
   ↓
16 Layer
   ↓
32 Layer 이상
```

따라서 **16-layer MU-MIMO 실증은 GPU의 "처리 가능성"을 보여준 것이고, Table 1은 GPU가 앞으로 RAN 성능을 더 끌어올릴 수 있는 이유를 설명하는 자료**라고 보면 됩니다. ([NVIDIA Developer][1])

개인적으로 RAN 개발 관점에서 이 표에서 가장 중요한 순서는:

**① Beamforming/Precoding → ② UE Pairing → ③ Channel Estimation → ④ Scheduling**

이라고 봅니다. 이 네 가지가 Massive MIMO의 실제 스펙트럼 효율 차이를 만드는 핵심 영역입니다.

[1]: https://developer.nvidia.com/blog/maximize-spectral-efficiency-with-ai-native-ran-and-nvidia-ai-aerial/?utm_source=chatgpt.com "Maximize Spectral Efficiency with AI-Native RAN and NVIDIA AI Aerial | NVIDIA Technical Blog"




Massive MIMO는 GPU 기반 RAN에서 **가장 GPU 친화적인 영역**으로 평가됩니다. 이유는 대부분의 연산이 **대규모 행렬 연산(Matrix Computation)** 으로 구성되어 CUDA와 Tensor Core의 장점을 그대로 활용할 수 있기 때문입니다.

현재 동향을 이해하는 데 도움이 되는 자료를 **기업 기술자료**, **학술 논문**, **향후 연구 방향**으로 나누어 정리했습니다.

---

# 1. NVIDIA Aerial 기술 자료 (가장 먼저 읽을 자료)

## ① NVIDIA Aerial CUDA-Accelerated RAN Documentation

**추천도 : ★★★★★**

이 문서는 NVIDIA가 실제 상용 제품에서 Massive MIMO를 어떻게 GPU에서 처리하는지 설명하는 공식 자료입니다.

### 핵심 내용

GPU에서 수행하는 PHY 작업

* FFT / IFFT
* Channel Estimation
* Equalization
* Beamforming
* Precoding
* LDPC
* MIMO Detection

특히 Beamforming과 Massive MIMO는 CUDA 커널을 이용해 병렬 처리합니다.

### 중요한 포인트

기존 ASIC

```
Antenna별 DSP
```

↓

GPU

```
Thousands CUDA cores

↓

Matrix Operation

↓

Beamforming
```

즉,

GPU는 안테나별 처리가 아니라

행렬 전체를 한번에 계산합니다.

이것이 GPU가 Massive MIMO에서 강한 이유입니다.

---

# 2. NVIDIA AI Aerial

**추천도 : ★★★★★**

최근 NVIDIA가 가장 밀고 있는 방향입니다.

핵심 메시지는

> GPU를 이용하면 Massive MIMO와 AI Beamforming을 하나의 플랫폼에서 동시에 수행할 수 있다. ([Facebook][1])

주요 내용

* AI 기반 CSI 예측
* AI Beamforming
* Massive MIMO Scheduling
* GPU Memory 공유
* TensorRT 기반 AI 추론

기존 Beamforming

```
CSI

↓

ZF

↓

Beamforming
```

새로운 방식

```
CSI

↓

Neural Network

↓

Beamforming Weight
```

---

# 3. Multi-User MIMO Detection and Precoding using GPU

**추천도 : ★★★★★**

최근 GPU Massive MIMO 연구 중 가장 실용적인 논문 중 하나입니다. ([arXiv][2])

### 연구 목적

GPU에서

* MU-MIMO Detection
* Precoding

을 실시간 수행 가능한지 검증합니다.

---

### 핵심 아이디어

GPU에서

```
H

↓

QR

↓

MMSE

↓

ZF

↓

Detection
```

모든 연산을 CUDA Kernel로 수행합니다.

---

### 결과

16×16 Massive MIMO에서

* 실시간 처리 가능
* 5G 슬롯 시간 제약 충족
* 사용자당 처리량 증가

을 보여줍니다. ([arXiv][2])

---

# 4. Deep Learning Beamforming Survey

**추천도 : ★★★★☆**

최근 Beamforming 연구를 정리한 Survey입니다.

주요 내용

기존 방식

* MRT
* ZF
* MMSE

AI 기반

* CNN
* Transformer
* Reinforcement Learning

비교

| 기존 방식    | AI 방식      |
| -------- | ---------- |
| CSI 필요   | CSI 일부만 사용 |
| 행렬역행렬 필요 | NN 추론      |
| 계산량 큼    | GPU 최적화 가능 |

최근에는 AI 기반 Beamforming이 기존 기법 대비 성능 향상 가능성을 보이며 활발히 연구되고 있습니다. 다만 채널 환경과 학습 데이터에 크게 의존하므로, 기존 ZF/MMSE를 완전히 대체했다기보다는 보완하는 방향이 주류입니다. ([ijeer.forexjournal.co.in][3])

---

# 5. Hybrid Beamforming for O-RAN mmWave Massive MIMO

**추천도 : ★★★★☆**

O-RAN과 Massive MIMO를 함께 다루는 연구입니다. ([ResearchGate][4])

핵심 내용

Hybrid Beamforming

```
RF Beamforming

+

Digital Beamforming
```

GPU에서는

Digital Beamforming

부분을 수행합니다.

---

논문의 주요 기여

* O-RAN 구조 적용
* Fronthaul 부하 감소
* Deep Learning Precoding
* Hybrid Beamforming

---

# 6. 최근 학계 연구 방향

최근 IEEE, WCNC, ICC, Globecom 등에서 많이 나오는 키워드는 다음과 같습니다. ([eejzhang.people.ust.hk][5])

### ① Near-field Massive MIMO

기존

Far-field

↓

6G

Near-field Beamforming

---

### ② AI CSI Estimation

CSI 계산 대신

AI Prediction

---

### ③ AI Precoding

기존

ZF

↓

Transformer

---

### ④ Cell-free Massive MIMO

기존

```
1 BS
```

↓

```
Many Distributed BS
```

---

### ⑤ Full Duplex Massive MIMO

동시에

TX

RX

수행

---

### ⑥ Holographic MIMO

안테나 수백~수천 개를 연속적인 전자기 표면처럼 활용하는 차세대 안테나 구조로, 6G 핵심 후보 기술 중 하나입니다. ([위키백과][6])

---

# GPU가 Massive MIMO에 적합한 이유

Massive MIMO에서 가장 많은 연산 시간을 차지하는 항목은 다음과 같습니다.

| 연산                    | GPU 적합성 | 특징                     |
| --------------------- | ------- | ---------------------- |
| FFT/IFFT              | ★★★★★   | OFDM 처리                |
| Matrix Multiplication | ★★★★★   | Tensor Core 활용         |
| QR Decomposition      | ★★★★★   | 병렬화 용이                 |
| SVD                   | ★★★★★   | 대규모 선형대수               |
| ZF/MMSE               | ★★★★★   | 행렬 역연산 기반              |
| Beamforming           | ★★★★★   | 안테나 수 증가에 따라 GPU 효율 증가 |
| LDPC                  | ★★★★☆   | 병렬 디코딩 가능              |
| Channel Estimation    | ★★★★★   | PRB 단위 병렬 처리           |

이 연산들은 대부분 **BLAS(L3), cuBLAS, cuSolver, CUTLASS**와 같은 GPU 선형대수 라이브러리를 활용하여 최적화할 수 있으며, Tensor Core를 활용하면 FP16/BF16 기반 연산 성능을 크게 높일 수 있습니다.

---

# RAN 개발자 관점에서 우선순위

귀하처럼 **O-RAN DU**, **DPDK**, **PHY 구현**에 관심이 있다면 다음 순서로 학습하는 것을 추천합니다.

1. **Massive MIMO PHY 처리 흐름**

   * CSI 추정 → Precoding → Beamforming → Detection
2. **CUDA 기반 행렬 연산**

   * cuBLAS, cuSolver, CUTLASS 활용
3. **NVIDIA Aerial의 cuPHY/cuMAC 아키텍처**
4. **GPU 메모리 최적화**

   * Shared Memory, GPUDirect RDMA, CUDA Stream
5. **AI 기반 Beamforming 및 Precoding**

   * 기존 ZF/MMSE와 AI 모델의 역할 및 장단점 비교
6. **O-RAN Split 7.2x에서 GPU 오프로딩 구조**

## 추가로 추천드리는 심화 주제

현재 RAN 업계에서 가장 활발한 연구는 **"GPU에서 Massive MIMO를 어떻게 구현하는가?"**보다 **"기존 DSP 알고리즘(ZF, MMSE, Channel Estimation)을 AI 모델로 대체할 수 있는가?"**로 이동하고 있습니다. 따라서 다음 세 가지를 함께 살펴보면 최신 흐름을 이해하는 데 큰 도움이 됩니다.

* **GPU 기반 Massive MIMO 소프트웨어 아키텍처** (CUDA 커널, 메모리 구조, 파이프라인)
* **AI Beamforming/AI Precoding** (Transformer, GNN 등)
* **NVIDIA Aerial SDK의 실제 구현 구조** (cuPHY, cuMAC, TensorRT 연계)

이 세 분야를 함께 보면 2026년 기준 GPU 기반 RAN의 핵심 기술 흐름을 거의 모두 파악할 수 있습니다.

[1]: https://www.facebook.com/NVIDIA.AP/posts/as-ai-native-models-grow-to-handle-real-world-network-complexity-legacy-hardware/1082357084114754/?utm_source=chatgpt.com "As AI-native models grow to handle real-world network ..."
[2]: https://arxiv.org/html/2510.01579v1?utm_source=chatgpt.com "Multi-User MIMO Detection and Precoding using GPU- ..."
[3]: https://ijeer.forexjournal.co.in/papers-pdf/ijeer-130316.pdf?utm_source=chatgpt.com "Optimizing Beamforming in Massive MIMO Systems Using ..."
[4]: https://www.researchgate.net/publication/395376272_Hybrid_Beamforming_and_Deep-Learning-Enabled_Precoding_for_O-RAN_mmWave_Massive_MIMO?utm_source=chatgpt.com "(PDF) Hybrid Beamforming and Deep-Learning-Enabled ..."
[5]: https://eejzhang.people.ust.hk/publicationsyear.html?utm_source=chatgpt.com "Publications by year"
[6]: https://en.wikipedia.org/wiki/MIMO?utm_source=chatgpt.com "MIMO"
