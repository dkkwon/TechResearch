네, 제가 찾아본 바로는 **생각보다 공개된 정보가 있습니다.** 다만 **논문 수준의 상세 구현**은 아니고, NVIDIA Aerial SDK 문서, SoftBank 발표, NVIDIA 기술 블로그를 조합해야 전체 그림이 보입니다.

오히려 흥미로운 것은, 이 자료들을 보면 **GPU가 PHY를 어떻게 구현했는지**와 **아직 공개하지 않는 부분**이 명확히 구분된다는 점입니다.

---

# 1. SoftBank가 실제로 공개한 내용

SoftBank 발표에서 가장 중요한 문장은 다음과 같습니다.

> **GPU에서 Massive MIMO에 필요한 대규모 행렬 연산과 PHY 무선 신호 처리를 모두 소프트웨어로 실행하여, RAN의 규정 처리 시간 내에서 안정적으로 동작함을 확인했다.** ([softbank.jp][1])

여기서 중요한 키워드는

* GPU 상에서
* CUDA
* Entire PHY Processing
* Software Only
* RAN Timing Requirement

입니다.

즉,

```
FPGA
ASIC

↓

없음

↓

GPU만 사용
```

이라는 의미입니다.

---

# 2. PHY 전체를 구현했다고 하지만 실제로 무엇을 의미할까?

NVIDIA는 Aerial SDK를 공개하고 있습니다.

거기에는

```
cuPHY
```

라는 라이브러리가 존재합니다.

공식 문서에 따르면

```
RU
↓

eCPRI

↓

GPU

↓

cuPHY

↓

MAC
```

구조입니다. ([NVIDIA Docs][2])

즉

GPU 안에서

* FFT
* Channel Estimation
* Equalizer
* LDPC
* Rate Matching
* DMRS
* Precoding
* Beamforming

등을 수행합니다.

---

# 3. NVIDIA가 공개한 실제 성능

최신 Aerial SDK(25.3)에서 공개한 성능은 상당히 구체적입니다.

예를 들어

```
6 Cells

100 MHz

64T64R

16 DL Layers

8 UL Layers

Early HARQ
```

지원이라고 명시되어 있습니다. ([NVIDIA Docs][3])

또

```
Single Cell

100 MHz

64T64R

MU-MIMO

16 UE

1 Layer / UE
```

Field Trial Validation도 공개되어 있습니다. ([NVIDIA Docs][3])

이 정보는 SoftBank 기사보다 훨씬 구체적입니다.

---

# 4. 재미있는 부분

Aerial Release Note를 보면

```
Wireless 32 DL Layers

without timing closure
```

라는 문장이 있습니다. ([NVIDIA Docs][3])

이 표현은 상당히 의미심장합니다.

통신 SW에서는

```
Timing Closure
```

는

> 실시간 deadline을 만족한다

는 의미입니다.

즉

32 Layer 기능은

```
동작은 하지만

↓

실시간은 아직 아님
```

이라는 뜻입니다.

반대로

16 Layer는

```
Timing Closure OK
```

라는 의미가 됩니다.

즉,

SoftBank가

굳이

```
16 Layer
```

를 발표한 이유가 설명됩니다.

---

# 5. GPU 안에서는 어떤 구조일까?

공개 자료를 종합하면

대략 이런 구조입니다.

```
NIC

↓

GPUDirect RDMA

↓

GPU Memory

↓

FFT

↓

Channel Estimation

↓

Equalizer

↓

LDPC

↓

Precoding

↓

Beamforming

↓

GPU Memory

↓

NIC
```

여기서 핵심은

CPU Memory를 거의 거치지 않는 것입니다.

```
NIC

↓

GPU Direct

↓

GPU
```

입니다.

그래야 PCIe 복사가 없어집니다. ([NVIDIA Docs][2])

---

# 6. GPU가 잘하는 부분

NVIDIA가 반복해서 강조하는 것은

GPU가

```
Tensor Core
```

때문이 아니라

```
Massive Parallel Processor
```

라는 점입니다.

예를 들어

Massive MIMO에서는

```
Channel Matrix

↓

Matrix Multiply

↓

Matrix Inverse

↓

Beamforming
```

이 반복됩니다.

GPU는

수천 개의 CUDA Core가

동시에 계산합니다.

ASIC은

특정 회로를

매우 빠르게 수행합니다.

GPU는

동일한 계산을

수천 Thread가 나누어 수행합니다.

---

# 7. 아직 공개하지 않는 부분

반대로 가장 궁금한 정보는 거의 공개되지 않았습니다.

예를 들면

* CUDA Kernel 구조
* FFT Kernel 구현
* LDPC Kernel 최적화
* Shared Memory 사용량
* Warp Scheduling
* Stream 개수
* CUDA Graph 구성
* GPU Occupancy
* GPU Utilization
* Worst-case Latency
* Slot별 Execution Time

이런 자료는 없습니다.

왜냐하면

이 부분이 NVIDIA의 핵심 IP이기 때문입니다.

---

# 8. 오래된 SoftBank Benchmark 자료도 흥미롭다

2020년경 SoftBank와 NVIDIA가 공개했던 Benchmark 자료도 상당히 참고가 됩니다.

당시에는

```
100 MHz

256QAM

DL 16 Layers

100 MHz PHY
```

를 GPU에서 Benchmark 했고,

CPU와 GPU 역할을 분리하여 PHY 성능을 측정했습니다. ([NVIDIA Developer][4])

즉,

2020년

```
Benchmark
```

↓

2023~2024년

```
Aerial SDK
```

↓

2025년

```
Outdoor Field Trial
```

순으로 발전한 것입니다.

---

# 제 생각: 연구해볼 만한 부분

질문하신 목적이 **GPU 기반 RAN 개발 동향을 파악하는 것**이라면, 앞으로 가장 가치 있는 분석은 **"GPU가 PHY의 어떤 기능을 맡고, 어떤 기능은 CPU에 남기는가?"**입니다.

현재 공개 자료를 종합하면 역할 분담은 대략 다음과 같습니다.

| CPU(제어 중심)       | GPU(연산 중심)             |
| ---------------- | ---------------------- |
| MAC Scheduler    | FFT/IFFT               |
| RLC/PDCP         | Channel Estimation     |
| HARQ Control     | Equalization           |
| FAPI 처리          | LDPC Encode/Decode     |
| OAM              | Precoding              |
| 제어 및 관리          | Beamforming            |
| AI Orchestration | Massive MIMO Matrix 연산 |

이 구조를 보면 **GPU는 단순한 AI 가속기가 아니라, 기존 FPGA/ASIC가 담당하던 "고연산 PHY 엔진"을 대체하는 역할**을 맡고 있음을 알 수 있습니다. 반면 CPU는 여전히 제어와 프로토콜 처리를 담당하며, GPU와 역할을 분담하는 구조입니다.

**추가로 추천드리는 주제**는 **NVIDIA Aerial cuPHY의 내부 아키텍처 분석**입니다. Aerial SDK 릴리스 노트와 공개 논문, 특허를 종합하면 **각 PHY 블록(FFT, LDPC, Precoding 등)이 GPU에서 어떻게 병렬화되는지**를 상당한 수준까지 유추할 수 있습니다. 이는 "왜 GPU가 PHY를 실시간으로 처리할 수 있는가?"를 이해하는 데 가장 핵심적인 자료가 될 것입니다.

[1]: https://www.softbank.jp/en/corp/news/press/sbkk/2025/20251029_02/?utm_source=chatgpt.com "SoftBank Corp.'s AI-RAN Realizes \"Software-only Massive MIMO\" on GPU | About Us | SoftBank"
[2]: https://docs.nvidia.com/aerial/aerial-cuphy/current/text/overview.html?utm_source=chatgpt.com "Overview - NVIDIA Docs"
[3]: https://docs.nvidia.com/aerial/cuda-accelerated-ran/25-3/index.html?utm_source=chatgpt.com "Aerial CUDA-Accelerated RAN — Aerial CUDA-Accelerated RAN"
[4]: https://developer.nvidia.com/blog/performing-the-vran-benchmark-with-gpus-using-the-nvidia-aerial-sdk/?utm_source=chatgpt.com "Softbank Benchmarks vRAN with GPUs and the NVIDIA Aerial SDK | NVIDIA Technical Blog"
