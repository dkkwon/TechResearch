최근 1~2년(2025~2026)의 CXL/NVMe 연구를 조사해 보면, 과거처럼 **"CXL vs NVMe"를 비교하는 연구는 거의 사라지고**, **"CXL과 NVMe를 함께 사용하는 메모리 계층(Hierarchical Memory)"** 이 핵심 주제가 되었습니다.

특히 AI/LLM의 KV Cache, 벡터 DB, 대용량 메모리 분석이 등장하면서 CXL과 NVMe의 역할이 명확하게 분리되고 있습니다.

---

# 전체 연구 동향

| 연구 분야                 | 2022~2024 | 2025~2026      |
| --------------------- | --------- | -------------- |
| CXL Memory Expansion  | 메모리 증설    | AI Memory Pool |
| CXL Memory Pooling    | 연구 단계     | 실제 제품 등장       |
| CXL + NVMe Hybrid     | 거의 없음     | 매우 활발          |
| GPU + CXL             | 초기        | 주요 연구 분야       |
| KV Cache 저장           | DRAM 중심   | CXL + SSD Tier |
| Computational Storage | 실험        | CXL SSD와 결합    |

즉,

> **CXL은 메모리 계층**
>
> **NVMe는 대용량 저장 계층**

으로 자리잡고 있습니다.

---

# 1. 가장 활발한 분야

## CXL + NVMe Hybrid Memory

현재 가장 많은 논문가 나오는 분야입니다.

구조는 아래와 같습니다.

```
CPU/GPU

    │

 DRAM
    │
 CXL Memory
    │
 NVMe SSD
```

메모리를 여러 단계(Tier)로 나누어 사용합니다.

예를 들면

```
Hot data
↓

DRAM

Warm data
↓

CXL

Cold data
↓

NVMe SSD
```

입니다.

이 구조는

* LLM
* Database
* Analytics

에서 매우 많이 연구되고 있습니다.

---

# 대표 논문 ①

**ITME: Inference Tiered Memory Expansion with Disaggregated CXL-Hybrid Memories (2026)**

핵심 내용

* CXL Memory
* PCIe Gen5 NVMe SSD
* Tiered Memory

를 하나의 메모리처럼 사용합니다. ([arXiv][1])

논문의 결과

* KV Cache를 SSD까지 확장
* Host Memory 한계를 극복
* 최대 **35.7% throughput 증가** ([arXiv][1])

이 논문은 현재 가장 주목받는 연구 중 하나입니다.

---

# 2. CXL SSD 연구

최근에는 SSD 자체가 CXL 장치가 되는 연구가 많습니다.

기존 SSD

```
CPU

↓

NVMe Driver

↓

SSD
```

새로운 구조

```
CPU

↓

CXL.mem

↓

CXL SSD
```

즉

SSD가

* Memory처럼 보이고
* Load/Store 가능

하도록 만드는 것입니다.

---

대표 연구

**WIO: Upload-Enabled Computational Storage on CXL SSDs (2026)**

핵심 아이디어

기존

```
Read SSD

↓

CPU 계산
```

대신

```
CXL SSD 내부에서 계산

↓

결과만 전달
```

입니다.

논문 결과

* 최대 **2배 throughput**
* **3.75배 write latency 개선** ([arXiv][2])

---

# 3. GPU Memory Expansion

AI 때문에 가장 큰 분야입니다.

기존

```
GPU

↓

HBM
```

이었지만

HBM이 매우 비싸고 용량이 부족합니다.

새로운 구조

```
GPU

↓

HBM

↓

CXL DRAM

↓

NVMe SSD
```

입니다.

대표 연구

**CXL-GPU (2025)**

특징

* GPU에서 직접 CXL 접근
* DRAM과 SSD를 모두 CXL 뒤에 연결
* Storage Latency Hide 기술

결과

GPU Memory를 크게 확장하면서 기존 방식보다 우수한 성능을 보였습니다. ([arXiv][3])

---

# 4. LLM KV Cache

2026년 가장 큰 연구 분야입니다.

KV Cache는

```
수백 GB
↓

수 TB
```

까지 커지고 있습니다.

따라서

```
HBM
↓

CXL

↓

NVMe
```

계층을 사용합니다.

최신 연구에서는

* Hot KV Cache → CXL
* 오래된 KV → NVMe

로 자동 이동합니다. ([arXiv][1])

---

# 5. Memory Pooling

예전에는

```
CPU
↓

Local DRAM
```

만 사용했습니다.

현재는

```
Server A

↓

CXL Switch

↓

Shared Memory Pool

↑

Server B
```

구조가 연구 및 제품화되고 있습니다.

AI Cluster에서 특히 중요합니다. CXL 4.0은 이러한 메모리 풀링과 패브릭 확장을 주요 기능으로 발전시키고 있습니다. ([Compute Express Link -][4])

---

# 6. CXL + NVMe-over-Fabrics

예전에는

```
CPU

↓

NVMe-oF
```

만 사용했습니다.

최근에는

```
CPU

↓

CXL Memory

↓

NVMe-oF
```

형태가 늘고 있습니다.

즉

CXL이 Cache 역할

NVMe-oF가 Storage 역할을 담당합니다. ([arXiv][1])

---

# 산업계 동향

학계뿐 아니라 산업계도 같은 방향으로 움직이고 있습니다.

| 회사       | 방향                                      |                                     |
| -------- | --------------------------------------- | ----------------------------------- |
| NVIDIA   | GPU + CXL Memory                        |                                     |
| Samsung  | CXL Memory Pool                         |                                     |
| SK hynix | CXL Memory Module + SSD Hybrid          |                                     |
| Marvell  | CXL Switch 및 Memory Pool                |                                     |
| Micron   | CXL Memory Expansion                    |                                     |
| Meta     | DDR4를 CXL 메모리 계층으로 재활용하는 메모리 확장 ASIC 개발 | ([Samsung Semiconductor Global][5]) |

---

# 사용자께서 관심을 가질 만한 연구 방향

이전 대화에서 말씀하신 것처럼 **"RAM 부족을 SSD로 보완"** 하는 목적과 가장 관련성이 높은 연구는 아래 세 가지입니다.

| 연구                                | 관련성   | 이유                           |
| --------------------------------- | ----- | ---------------------------- |
| Tiered Memory (DRAM + CXL + NVMe) | ★★★★★ | SSD를 메모리의 하위 계층으로 활용         |
| CXL Hybrid Memory                 | ★★★★★ | 메모리 부족 문제를 직접 해결             |
| KV Cache Tiering                  | ★★★★☆ | 접근 빈도에 따라 CXL과 SSD를 자동 활용    |
| CXL SSD                           | ★★★★☆ | SSD를 Load/Store 가능한 메모리처럼 사용 |
| Computational Storage             | ★★★☆☆ | 데이터 이동을 줄여 CPU 부담 감소         |

## 귀하의 프로젝트와의 연관성

현재 검토 중인 **"RAM이 부족한 데이터를 SSD에 두고 필요 시 메모리처럼 활용"** 하는 아이디어는, 최신 연구에서는 다음과 같은 형태로 발전하고 있습니다.

1. **DRAM → CXL Memory → NVMe SSD**의 3단계 메모리 계층 구성
2. 접근 빈도(Hot/Warm/Cold)에 따라 데이터를 자동 이동(Tiering)
3. 애플리케이션이 가능한 한 기존 메모리 접근(load/store) 방식으로 사용할 수 있도록 추상화
4. AI뿐 아니라 대용량 데이터베이스, 분석 시스템에도 적용

이 때문에 단순한 스왑(swap)이나 mmap 기반 접근보다 **메모리 계층 관리 정책**과 **데이터 배치 알고리즘**이 최근 연구의 핵심 주제가 되고 있습니다.

다음 단계로는 **"CXL 없이 Linux에서 SSD를 메모리처럼 활용하는 최신 오픈소스 및 연구(예: user-space paging, mmap 기반, DAMON, Tiered Memory 등)"**를 조사하면, 현재 개발 환경에 바로 적용 가능한 기술들을 중심으로 정리해 드릴 수 있습니다.

[1]: https://arxiv.org/abs/2606.12556?utm_source=chatgpt.com "ITME: Inference Tiered Memory Expansion with Disaggregated CXL-Hybrid Memories"
[2]: https://arxiv.org/abs/2604.02442?utm_source=chatgpt.com "WIO: Upload-Enabled Computational Storage on CXL SSDs"
[3]: https://arxiv.org/abs/2506.15601?utm_source=chatgpt.com "CXL-GPU: Pushing GPU Memory Boundaries with the Integration of CXL Technologies"
[4]: https://computeexpresslink.org/blog/advancing-coherent-connectivity-highlights-from-the-cxl-consortium-at-sc25-4351/?utm_source=chatgpt.com "Advancing Coherent Connectivity: Highlights from the CXL ..."
[5]: https://semiconductor.samsung.com/news-events/tech-blog/breaking-ai-memory-limits-with-cxl-memory-pooling/?utm_source=chatgpt.com "Breaking AI Memory Limits with CXL Memory Pooling"
