사용자께서 이전에 말씀하신 목적은 다음과 같았습니다.

> **"RAM 부족 시 SSD를 메모리처럼 활용하고, 읽기/쓰기가 조금 느려도 괜찮은 데이터는 SSD에 저장하고 싶다."**

최신 연구를 조사해 보면, 흥미롭게도 **운영체제의 swap을 개선하는 방향보다는 User-space에서 메모리 계층을 직접 관리하는 방향**이 주류가 되고 있습니다.

---

# 연구 분야의 변화

예전(2015~2020)

```
Application

↓

malloc()

↓

DRAM

↓

Swap
```

현재(2025~2026)

```
Application

↓

Memory Manager

↓

DRAM

↓

NVMe

(직접 관리)
```

즉,

OS Swap에 맡기지 않고

Application 또는 Runtime이

* 어느 데이터를 SSD에 둘지
* 언제 Prefetch할지
* 언제 DRAM으로 올릴지

를 직접 결정합니다.

---

# 1. Linux DAMON (가장 주목받는 기술)

현재 Linux Memory 연구에서 가장 중요한 프로젝트입니다.

DAMON(Data Access MONitor)은

메모리 접근 빈도를 지속적으로 추적합니다.

```
Page A
1000 accesses/sec

↓

DRAM 유지

----------------

Page B
1 access/min

↓

SSD 이동 후보
```

Linux Mainline에 포함되어 있으며, 최근에는 메모리 티어링, 사용자 공간 정책 연동, 데이터 속성 기반 관리 등 기능이 빠르게 확장되고 있습니다. ([LWN.net][1])

### 장점

* Hot/Cold 데이터 자동 분류
* 메모리 낭비 감소
* SSD 이동 정책과 결합 가능

---

# 2. vmcacheₙ (2026)

최근 발표된 매우 흥미로운 연구입니다.

구조는 다음과 같습니다.

```
Application

↓

Virtual Address

↓

DRAM

↓

Remote Memory

↓

NVMe SSD
```

핵심 아이디어는

> **Virtual Address는 절대 바꾸지 않는다.**

실제 Physical Page만 이동합니다.

예를 들면

```
0x10000000

↓

DRAM

↓

SSD

↓

다시 DRAM
```

주소는 항상

```
0x10000000
```

입니다.

따라서 Application은

```
char* ptr
```

를 계속 사용할 수 있습니다.

연구에서는 `move_pages()`를 확장한 새로운 페이지 이동 방식으로 DRAM–원격 메모리–NVMe 간 다단계 메모리 계층을 구현하여 데이터베이스 처리량을 최대 **4배** 향상시켰습니다. ([Yeasir Rayhan][2])

---

# 3. User-space Paging

최근에는

Kernel Page Fault를 줄이기 위한 연구가 매우 많습니다.

기존

```
Page Fault

↓

Kernel

↓

SSD Read
```

대신

```
Application

↓

Prefetch

↓

SSD Read

↓

No Page Fault
```

입니다.

대표적인 기술

* io_uring
* SPDK
* xNVMe

입니다.

이들은 모두

> SSD를 파일이 아니라

> "Memory Backend"

처럼 사용하는 것을 목표로 합니다.

특히 xNVMe는 다양한 NVMe 장치와 Linux의 `io_uring`, SPDK 등을 하나의 사용자 공간 API로 사용할 수 있도록 하는 프레임워크로 데이터베이스 연구에서도 활용되고 있습니다. ([VLDB][3])

---

# 4. mmap 기반 연구

과거에는

```
mmap()

↓

SSD
```

만 사용했습니다.

하지만

Page Fault가 너무 많았습니다.

최근에는

```
mmap

+

Prefetch

+

Huge Page

+

Async Read
```

를 결합합니다.

즉

Fault가 발생하기 전에

미리 SSD에서 읽어옵니다.

---

# 5. Async Memory

최근 논문의 공통 아이디어입니다.

기존

```
Read()

↓

Wait
```

새로운 방식

```
Request

↓

Continue

↓

Later

↓

Memory Ready
```

입니다.

즉

SSD를

비동기 메모리처럼 사용합니다.

---

# 6. SPDK 기반 Memory

SPDK를 사용하면

```
malloc()

↓

Huge Page

↓

SPDK DMA

↓

NVMe SSD
```

형태가 됩니다.

장점

* Kernel bypass
* Copy 감소
* CPU 사용률 감소
* 높은 IOPS

단점

* malloc처럼 사용할 수는 없음
* Runtime 구현 필요

---

# 7. LLM에서 사용하는 SSD Memory

최근 LLM 연구에서는

GPU Memory 부족 때문에

SSD를 적극 사용합니다.

구조는

```
GPU

↓

DRAM

↓

NVMe SSD
```

입니다.

하지만

SSD를 직접 읽지 않습니다.

대부분

```
Prediction

↓

Prefetch

↓

DRAM

↓

GPU
```

입니다.

즉

SSD Access를 숨기는 것이 핵심입니다. 최신 연구에서는 GPU–DRAM–SSD 계층을 구성하고 예측 기반 프리페치로 SSD 지연을 상쇄하는 기법을 제안하고 있습니다. ([arXiv][4])

---

# 8. 최근 Linux Read Stack 연구

2026년 발표된 **MARS**는 SSD 자체보다 **Linux의 Buffered Read 경로**가 병목이라는 점에 주목합니다.

기존 Linux는 큰 읽기 요청도 페이지 캐시 처리와 I/O 제출을 반복적으로 수행하지만, MARS는 이를 하나의 작업으로 묶어 병렬화합니다.

결과적으로

* NVMe RAID에서 최대 **4.4배** 읽기 성능
* DuckDB 질의 **1.8~2.1배** 향상
* 모델 로딩 **3배 이상** 향상

을 보고했습니다. 이는 SSD를 메모리처럼 사용할 때 **스토리지보다 운영체제 I/O 경로가 병목**이 될 수 있음을 보여줍니다. ([arXiv][5])

---

# 현재 가장 많이 사용되는 오픈소스

| 프로젝트           | 목적                  | SSD를 메모리처럼 사용 | 성숙도   |
| -------------- | ------------------- | ------------- | ----- |
| Linux DAMON    | Hot/Cold 페이지 모니터링   | ★★★★☆         | 매우 높음 |
| SPDK           | User-space NVMe I/O | ★★★★☆         | 매우 높음 |
| xNVMe          | 통합 NVMe API         | ★★★★☆         | 높음    |
| io_uring       | 비동기 I/O             | ★★★★☆         | 매우 높음 |
| mmap + madvise | 메모리 매핑              | ★★★☆☆         | 매우 높음 |
| vmcacheₙ       | 연구용 다단계 메모리         | ★★★★★         | 연구 단계 |

# 귀하의 프로젝트 관점에서의 추천

지금까지 나눈 대화를 보면 목표는 다음과 같습니다.

* RAM 부족 문제 해결
* SSD를 메모리처럼 활용
* CPU 오버헤드 최소화
* Linux 환경
* C/C++ 애플리케이션에서 직접 구현 가능

이 목표라면 **swap을 개선하는 접근보다는 사용자 공간 메모리 런타임(User-space Memory Runtime)** 을 구현하는 것이 최신 연구 흐름과 가장 잘 맞습니다.

권장 아키텍처는 다음과 같습니다.

```
Application
      │
      ▼
Memory Manager (사용자 공간)
      │
 ┌────┴────┐
 │         │
DRAM    NVMe SSD
 │         │
Hot      Cold
```

이 Memory Manager는 다음 기능을 담당합니다.

* 접근 빈도 기반 Hot/Cold 분류(DAMON과 유사한 정책)
* 비동기 읽기/쓰기(io_uring 또는 SPDK)
* 접근 예측 기반 Prefetch
* LRU/LFU와 같은 승격(Promotion)·강등(Demotion) 정책
* 동일한 가상 주소를 유지하는 객체 관리 계층

이 구조는 현재 데이터베이스, AI 추론, 대규모 인메모리 분석 시스템에서 가장 활발히 연구되는 방향과 일치합니다. 또한 기존 swap보다 정책을 애플리케이션 특성에 맞게 최적화할 수 있다는 장점이 있습니다.

현재까지의 대화를 고려하면, 다음 단계로 가장 도움이 될 주제는 **"C++에서 구현 가능한 SSD Memory Runtime 설계"**입니다. 여기서는 `malloc/free`를 대체하는 API, 메타데이터 구조, 페이지 캐시, LRU/LFU, io_uring/SPDK 기반 비동기 I/O, 그리고 Prefetch 알고리즘까지 포함한 실제 구현 아키텍처를 제안드릴 수 있습니다.

[1]: https://lwn.net/Articles/1071256/?utm_source=chatgpt.com "A 2026 DAMON update"
[2]: https://yeasirrayhanprince.github.io/data/damon26_vm_tiered_buffer_pool_camera_ready.pdf?utm_source=chatgpt.com "Virtual-Memory Assisted Buffer Management In Tiered ..."
[3]: https://vldb.org/cidrdb/papers/2026/p6-houlborg.pdf?utm_source=chatgpt.com "Flexible I/O for Database Management Systems with xNVMe"
[4]: https://arxiv.org/abs/2607.07388?utm_source=chatgpt.com "TF-Engram: A Train-Free Engram with SSD-Backed Memory for Large Language Models"
[5]: https://arxiv.org/abs/2607.13604?utm_source=chatgpt.com "MARS: Multi-stage Accelerated Read Stack for Large-buffer Buffered Reads"


지금까지의 대화를 종합해 보면, 귀하가 목표하는 시스템은 일반적인 `swap`이나 `mmap` 라이브러리가 아닙니다.

**RAN, DPDK, C++ 기반의 고성능 시스템에서 "SSD를 DRAM의 확장 메모리처럼 사용하는 Runtime"**에 가깝습니다.

최근(2025~2026) 연구에서도 이러한 방향을 **User-space Tiered Memory Runtime**이라고 부르는 경우가 많습니다.

---

# 목표 아키텍처

가장 추천하는 구조는 다음과 같습니다.

```
                Application
                      │
          malloc()/new 대신 Runtime API
                      │
        +-----------------------------+
        |     SSD Memory Runtime      |
        |-----------------------------|
        | Object Manager              |
        | Cache Manager               |
        | Access Monitor              |
        | Prefetch Engine             |
        | Eviction Engine             |
        | Async IO Scheduler          |
        | Metadata Manager            |
        +-----------------------------+
              │                │
          DRAM Cache      NVMe SSD
```

중요한 점은

SSD를 직접 사용하는 것이 아니라

**Runtime이 DRAM Cache를 관리**한다는 것입니다.

---

# Runtime 내부 모듈

저라면 다음과 같이 분리합니다.

```
MemoryRuntime
    │
    ├── Allocator
    ├── Cache
    ├── SSD Backend
    ├── Metadata
    ├── Prefetch
    ├── Eviction
    ├── Statistics
    └── IO Scheduler
```

각 모듈을 살펴보겠습니다.

---

# 1. Allocator

기존

```cpp
void* p = malloc(size);
```

대신

```cpp
auto p = runtime.allocate(size);
```

Allocator는

```
Object ID

↓

Metadata

↓

DRAM

or

SSD
```

를 결정합니다.

즉

실제 주소를 반환하는 것이 아니라

Runtime Object를 관리합니다.

---

# 2. Metadata Manager

가장 중요한 모듈입니다.

예를 들어

```cpp
struct PageMeta
{
    uint64_t page_id;

    bool in_dram;

    bool dirty;

    bool pinned;

    uint64_t ssd_offset;

    uint64_t access_count;

    uint64_t last_access;

    uint32_t refcnt;
};
```

모든 페이지가

Metadata를 하나씩 가집니다.

---

# 3. DRAM Cache

Runtime 내부에서는

SSD를 바로 접근하지 않습니다.

```
SSD

↓

DRAM Cache

↓

Application
```

입니다.

예를 들어

```
SSD

Page100

↓

DRAM Slot5
```

처럼 매핑됩니다.

---

# Cache 크기

예를 들면

```
SSD

1TB

↓

DRAM Cache

16GB
```

입니다.

16GB만 실제 메모리를 사용합니다.

---

# 4. Page Table

Runtime은

자체 Page Table을 가집니다.

```
PageID

↓

DRAM Slot
```

예

```
Page100

↓

Slot5

↓

0x70000000
```

---

# 5. Access Monitor

최근 논문에서는

가장 중요한 모듈입니다.

매 접근마다

```
Page100

↓

access++
```

합니다.

또는

```
last_access = now()
```

를 저장합니다.

이를 통해

```
Hot

Warm

Cold
```

를 판단합니다.

---

# 6. Eviction

메모리가 부족하면

```
Cold Page

↓

SSD Write

↓

DRAM Free
```

입니다.

알고리즘은

* LRU
* CLOCK
* LFU
* ARC
* TinyLFU

등을 선택할 수 있습니다.

최근 연구에서는 TinyLFU와 CLOCK-Pro 계열이 많이 사용됩니다.

---

# 7. Prefetch

가장 성능 차이가 큰 부분입니다.

예를 들어

Application이

```
1

2

3

4

5
```

를 접근했다면

Runtime은

```
6

7

8
```

을 미리 SSD에서 읽습니다.

이것이 최신 논문의 핵심입니다.

---

# Prefetch 종류

추천 순서는

```
Sequential

↓

Stride

↓

History

↓

ML Prediction
```

입니다.

초기 구현에서는

Sequential만으로도 효과가 큽니다.

---

# 8. IO Scheduler

가장 중요한 부분입니다.

절대

```
pread()
```

를 사용하지 않습니다.

추천은

```
io_uring
```

입니다.

또는

```
SPDK
```

입니다.

구조는

```
Application

↓

Queue IO

↓

Continue

↓

Completion
```

입니다.

Blocking이 없어집니다.

---

# 9. Async API

추천 API는

```cpp
Future<Page*> load(PageID);
```

또는

```cpp
runtime.prefetch(page);
```

입니다.

Blocking API보다

```
Request

↓

Compute

↓

Complete
```

가 훨씬 성능이 좋습니다.

---

# 10. Runtime API

추천 인터페이스는

```cpp
class MemoryRuntime
{
public:

    Handle allocate(size_t);

    void free(Handle);

    void* lock(Handle);

    void unlock(Handle);

    Future prefetch(Handle);

    void flush(Handle);

    Statistics stat();
};
```

여기서 **Handle**은 메모리 주소가 아니라 논리적 식별자입니다.

---

# Handle을 사용하는 이유

가장 큰 문제는

SSD로 이동하면

```
pointer

↓

무효
```

가 된다는 것입니다.

따라서

```
Pointer

×

Handle

○
```

입니다.

예를 들면

```
Handle 100

↓

Metadata

↓

DRAM Slot

↓

SSD Offset
```

입니다.

---

# 내부 상태 머신

Runtime 내부에서는

```
        Allocate

           │

           ▼

      DRAM Resident

           │

           ▼

        Hot Access

           │

           ▼

      Keep in DRAM

           │

   No Access

           │

           ▼

    Write to SSD

           │

           ▼

     SSD Resident

           │

      Access

           │

           ▼

 Async Read SSD

           │

           ▼

    DRAM Resident
```

이 상태 머신 하나가 Runtime의 핵심입니다.

---

# 귀하의 프로젝트에 맞춘 개선점

이전 대화에서 **"읽기/쓰는 데 시간이 조금 걸려도 상관없는 데이터"**와 **Linux 환경에서의 메모리 부족 해결**을 목표로 말씀하셨습니다. 이 조건이라면 범용 메모리 관리보다 **애플리케이션이 데이터의 특성을 알고 있다는 점을 활용**하는 것이 성능에 큰 도움이 됩니다.

예를 들어 객체를 생성할 때 데이터의 특성을 함께 전달하도록 설계할 수 있습니다.

```cpp
enum class TierHint
{
    Hot,
    Warm,
    Cold,
    Sequential,
    Persistent
};

Handle allocate(size_t size, TierHint hint);
```

그러면 Runtime은 다음과 같은 정책을 적용할 수 있습니다.

| 데이터 특성     | 권장 정책                      |
| ---------- | -------------------------- |
| Hot        | DRAM에 최대한 유지               |
| Warm       | DRAM과 SSD 사이를 접근 빈도에 따라 이동 |
| Cold       | 즉시 SSD에 배치                 |
| Sequential | 적극적인 선행 읽기(Prefetch)       |
| Persistent | Dirty 페이지를 우선적으로 SSD에 기록   |

이 방식은 운영체제가 페이지 접근 패턴을 추측하는 것보다 훨씬 효율적이며, 최근 AI 추론 엔진과 데이터베이스의 메모리 런타임에서도 비슷한 접근을 사용합니다.

## 제가 가장 추천하는 2단계 개발 로드맵

현재 목표라면 처음부터 모든 기능을 구현하기보다 다음 순서가 현실적입니다.

**1단계(MVP)**

* Handle 기반 메모리 관리
* 4KB 또는 2MB 페이지 단위 관리
* LRU 기반 캐시
* `io_uring` 기반 비동기 I/O
* 순차 접근(Sequential) Prefetch

**2단계(고도화)**

* 접근 패턴 분석(Hot/Warm/Cold)
* 비동기 Write-back 및 배치 쓰기
* 압축 및 중복 제거(선택 사항)
* NUMA 인식 DRAM 캐시
* DPDK Huge Page와 연계한 DRAM 풀 관리

이러한 구조라면 CXL이 없는 환경에서도, 향후 CXL 메모리를 추가할 경우 **DRAM → CXL → NVMe**의 3단계 메모리 계층으로 자연스럽게 확장할 수 있습니다. 이는 현재 연구와 산업계가 지향하는 방향과도 잘 맞는 설계입니다.

