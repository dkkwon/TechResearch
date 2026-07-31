메모리를 대체하기 위해 **HDD/SSD를 DRAM의 확장 또는 대체 메모리로 사용하는 연구**는 크게 몇 가지 방향으로 발전해 왔습니다.

먼저 중요한 점은 용어를 구분해야 합니다.

| 개념                             | 의미                             | 대표 기술                               |
| ------------------------------ | ------------------------------ | ----------------------------------- |
| **Swap / Paging**              | DRAM 부족 시 HDD/SSD를 보조 메모리처럼 사용 | Linux Swap, Windows Page File       |
| **Memory Extension**           | SSD를 DRAM 용량 확장처럼 사용           | SSDAlloc, NVMe Memory Extension     |
| **Storage Class Memory (SCM)** | SSD와 DRAM 사이의 새로운 메모리 계층       | Intel Optane, Z-NAND                |
| **Disaggregated Memory**       | SSD/NVMe를 네트워크/CXL로 메모리 풀처럼 사용 | CXL Memory Pool, NVMe Memory Fabric |

아래는 이 분야에서 중요한 논문과 연구 흐름입니다.

---

# 1. SSD를 RAM 확장으로 사용하는 연구

## 1) SSDAlloc: Hybrid RAM/SSD Memory Allocation Made Easy (NSDI 2011)

**논문**

* Anirudh Badam, Vivek Pai
* USENIX NSDI 2011
* "SSDAlloc: Hybrid RAM/SSD Memory Allocation Made Easy" ([Microsoft][1])

### 목표

SSD를 단순 저장장치가 아니라 **느린 대용량 RAM**처럼 사용.

기존 구조:

```
Application
    |
    |
  DRAM
    |
    |
  SSD (Swap)
```

SSDAlloc:

```
Application
    |
Hybrid Memory Allocator
    |
 +---------+
 | DRAM    |  Fast
 +---------+
 | SSD     |  Large
 +---------+
```

### 주요 결과

| 항목        | 결과                |
| --------- | ----------------- |
| SSD 활용    | RAM extension     |
| 메모리 증가    | 수백 GB 이상 가능       |
| SSD 성능 활용 | Raw SSD 성능의 약 90% |
| SSD 수명    | 최대 32배 증가         |

([Microsoft][1])

### 핵심 아이디어

기존 Swap 문제:

* Page 단위(4KB) 이동
* Random access 많음
* SSD write amplification 증가

개선:

* SSD를 직접 memory allocator 수준에서 관리
* Access pattern 기반 migration
* Write 최소화

---

# 2. SSD 기반 Swap 성능 연구

## 2) Performance Evaluation of the SSD-based Swap System for Big Data Processing (IEEE TrustCom 2014)

([Macquarie University][2])

### 연구 목적

"SSD를 Swap 영역으로 사용하면 실제 메모리 확장 효과가 있는가?"

실험:

```
DRAM 부족 상황

        Memory Pressure
              |
              v

       Linux Swap

        HDD vs SSD 비교
```

### 결론

SSD Swap:

장점:

* HDD 대비 latency 감소
* Random access 성능 증가
* Big data 처리 개선

하지만:

SSD는 DRAM을 완전히 대체하지 못함.

이유:

| 항목        | DRAM    | SSD     |
| --------- | ------- | ------- |
| Latency   | ~100 ns | ~100 μs |
| 차이        | -       | 약 1000배 |
| Bandwidth | 수백 GB/s | 수 GB/s  |

즉:

> SSD는 "느린 RAM"이 아니라 "빠른 Storage"에 가까움

---

# 3. NAND Flash를 DRAM Cache처럼 사용하는 연구

## 3) Flash-based Extended Cache for Higher Throughput and Faster Recovery (FaCE)

([arXiv][3])

### 목적

SSD를 DRAM cache 확장으로 사용.

구조:

```
Application
      |
Database Buffer
      |
 +---------+
 | DRAM    |
 +---------+
      |
 +---------+
 | SSD     |
 | Cache   |
 +---------+
```

### 적용 분야

* Database
* Big Data
* Cloud Storage

특징:

* SSD를 write cache로 활용
* Recovery 시간 감소
* Throughput 증가

---

# 4. HDD → SSD → Memory 계층 변화 연구

## 4) Unifying buffer replacement and prefetching with data migration for heterogeneous storage devices

([Ewha Womans University][4])

연구 배경:

기존:

```
CPU
 |
DRAM
 |
HDD
```

변경:

```
CPU
 |
DRAM
 |
SSD
 |
HDD
```

SSD를 중간 계층으로 사용.

목표:

* 자주 쓰는 데이터 → SSD
* 오래된 데이터 → HDD

즉:

SSD를 "Memory hierarchy의 한 단계"로 보는 연구.

---

# 5. 최신 방향: SSD를 Memory Device로 사용하는 연구

최근에는 단순 Swap보다 더 적극적인 방향으로 이동하고 있습니다.

---

## 5-1) NVM 기반 Memory Expansion

## Extending Memory Capacity in Consumer Devices with Emerging Non-Volatile Memory

([arXiv][5])

사용 기술:

* Intel Optane SSD
* NVM

구조:

```
Application

      |
      v

DRAM
(Primary Memory)

      |
      v

NVM SSD
(Memory Extension)
```

실험:

* Chromebook
* Optane SSD Swap

결과:

* DRAM 부족 상황에서 성능 유지 가능
* NAND SSD보다 Optane이 우수

---

# 6. CXL 기반 SSD Memory Pool 연구 (최신)

최근 AI/Cloud 분야에서는 SSD를 CXL Memory처럼 활용하려는 연구가 증가하고 있습니다.

## CXL-Based SSD Memory System

([arXiv][6])

구조:

```
CPU
 |
CXL
 |
Memory Pool
 |
+-------------+
| DRAM Cache |
+-------------+
| SSD/NAND   |
+-------------+
```

목표:

* 서버 메모리 부족 해결
* AI Model Memory 확장
* GPU Memory Extension

---

# 연구 흐름 정리

| 시기        | 기술                    | 목적              | 대표 연구          |
| --------- | --------------------- | --------------- | -------------- |
| 2000년대    | HDD Swap              | 부족한 RAM 보완      | Virtual Memory |
| 2010년     | SSD Swap              | HDD보다 빠른 Paging | SSD Swap 연구    |
| 2011년     | SSD Memory Allocation | SSD를 RAM처럼 사용   | SSDAlloc       |
| 2012~2015 | SSD Cache             | DB/Data 처리 향상   | FaCE           |
| 2015~2022 | NVM Memory            | DRAM 보완         | Optane 연구      |
| 2023~현재   | CXL + SSD             | Memory Pool     | CXL SSD        |

---

# 성능 관점 비교

대략적인 Memory Hierarchy:

| Device     | Latency    | Bandwidth    | Capacity |
| ---------- | ---------- | ------------ | -------- |
| CPU Cache  | ns         | TB/s         | MB       |
| DRAM       | 50~100 ns  | 100~500 GB/s | GB       |
| Optane/NVM | 300ns~10us | GB/s         | TB       |
| NVMe SSD   | 10~100us   | GB/s         | TB       |
| SATA SSD   | 100us      | 500MB/s      | TB       |
| HDD        | ms         | 100MB/s      | TB       |

---

# 현재 연구 방향 (2025~2026)

특히 AI workload 때문에 방향이 바뀌고 있습니다.

기존:

```
SSD = 느린 Storage
```

현재:

```
SSD = Large Capacity Memory Tier
```

예:

```
GPU HBM
   |
   |
DDR Memory
   |
   |
NVMe SSD
   |
   |
Object Storage
```

AI 모델:

* LLM parameter
* KV Cache
* Embedding
* Dataset

때문에 SSD를 **Memory Hierarchy의 하위 Tier**로 사용하는 연구가 활발합니다. ([TechRadar][7])

---

## 결론

현재 연구 흐름은:

1. **HDD를 SSD로 교체**

   * 이미 상용화

2. **SSD를 Swap 장치로 사용**

   * Linux/Cloud에서 사용

3. **SSD를 DRAM 확장 메모리로 사용**

   * SSDAlloc, NVM 연구

4. **SSD + CXL 기반 Memory Pool**

   * 미래 서버/AI 구조

입니다.

특히 사용 목적이 **"RAM 부족 해결"인지, "AI/GPU 메모리 확장인지, "네트워크 장비(RAN/DPDK 등)에서 메모리 비용 절감인지"**에 따라 관련 논문 분야가 조금 달라집니다. 이전에 질문하셨던 Kubernetes/DPDK/RAN 환경이라면 **Memory Expansion + CXL/NVMe 연구** 쪽이 가장 관련성이 높습니다.

[1]: https://www.microsoft.com/en-us/research/publication/ssdalloc-hybrid-ramssd-memory-allocation-made-easy/?utm_source=chatgpt.com "SSDAlloc: Hybrid RAM/SSD Memory Allocation Made Easy - Microsoft Research"
[2]: https://researchers.mq.edu.au/en/publications/performance-evaluation-of-the-ssd-based-swap-system-for-big-data-/?utm_source=chatgpt.com "Performance evaluation of the SSD-based swap system for big data processing - Macquarie University"
[3]: https://arxiv.org/abs/1208.0289?utm_source=chatgpt.com "Flash-based Extended Cache for Higher Throughput and Faster Recovery"
[4]: https://pure.ewha.ac.kr/en/publications/unifying-buffer-replacement-and-prefetching-with-data-migration-f/?utm_source=chatgpt.com "Unifying buffer replacement and prefetching with data migration for heterogeneous storage devices - Ewha Womans University"
[5]: https://arxiv.org/abs/2111.02325?utm_source=chatgpt.com "Extending Memory Capacity in Consumer Devices with Emerging Non-Volatile Memory: An Experimental Study"
[6]: https://arxiv.org/abs/2501.02524?utm_source=chatgpt.com "A Full-System Simulation Framework for CXL-Based SSD Memory System"
[7]: https://www.techradar.com/pro/did-kioxia-just-unveil-the-fastest-ssd-ever-gp-series-uses-storage-class-memory-to-feed-the-hbm-gpu-with-millions-of-iops?utm_source=chatgpt.com "Did Kioxia just unveil the fastest SSD ever? GP series uses Storage Class Memory to feed the HBM GPU with millions of IOPS"
