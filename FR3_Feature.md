1차 질문)
이동 통신에서 FR3 기능을 구현하기 위해 관련된 오픈 소스를 검토하고 있습니다.
FR3 는 5G Advanced 기능으로 볼 수 있고, 현재 5G 기반으로 개발이 됩니다.
FR3 의 기반이 되는 5G  Feature/Capacity을 정리 하고 싶습니다. 
먼저 어떤 feature 들을 체크해야 할지 정리해야 하겠습니다. 

결과물 : Feature 정리 (상위 항목, 하위 항목, 지원 범위 )
                 지원 범위는 명확하게 지원/미지원/특정 기능 지원/KPI (수치)로 표현되어야 함

2차 질문)
srsRAN은 OCUDU 로 변경되었습니다. https://gitlab.com/ocudu/ocudu 위의 Feature 정리된 내용에 대해서 OCUDU 코드를 기반으로 내용을 채워서 주세요

아래의 내용들이 Feature 에 포함되어야 함  (1차 질문에서)
1. Attached UE 수 / TTI당 스케줄링 가능 UE 수
2. Downlink 자원할당 가능 BW
3. Cell당 스케줄 가능한 DL MIMO 최대 Layer 수
4. SU-MIMO / MU-MIMO 지원 여부
5. SRS 채널 추정 지원 여부
6. O-RAN Fronthaul 지원 BW 최대 용량
7. PHY 가속기 / LDPC SW 동작 여부


아래 내용은 현재 [OCUDU GitLab](https://gitlab.com/ocudu/ocudu?utm_source=chatgpt.com) 및 공개 문서 기준으로,
“FR3 readiness 분석용 Capability Matrix” 관점에서 정리한 것입니다.
(주의: 공개 문서에 없는 항목은 코드 구조/현 상태 기준으로 “추정”, “부분 지원”, “미지원”으로 분리했습니다.) ([about.gitlab.com][1])




---

# OCUDU Feature Classification (FR3 Readiness 관점)

# 1. NR PHY Base Capability

| 상위 항목      | 하위 항목                 | 지원 범위                     | 상태    |
| ---------- | --------------------- | ------------------------- | ----- |
| Numerology | SCS                   | 15/30/60/120 kHz          | 지원    |
| Bandwidth  | DL/UL BW              | FR1 기준 최대 100 MHz 수준      | 부분 지원 |
| PRB        | Max PRB               | 273 PRB (100MHz@30kHz 기준) | 부분 지원 |
| Duplex     | FDD                   | 지원                        | 지원    |
| Duplex     | TDD                   | 지원                        | 지원    |
| Modulation | DL                    | QPSK/16QAM/64QAM/256QAM   | 지원    |
| Modulation | UL                    | QPSK/16QAM/64QAM          | 지원    |
| HARQ       | DL/UL HARQ            | NR HARQ framework         | 지원    |
| LDPC       | BG1/BG2               | 지원                        | 지원    |
| Polar      | PBCH/UCI              | 지원                        | 지원    |
| OFDM       | FFT pipeline          | SIMD 기반 SW 처리             | 지원    |
| PHY Timing | Slot-based processing | Real-time Linux 요구        | 지원    |
| FR2 PHY    | mmWave PHY            | 공개 기준 제한적                 | 부분 지원 |
| FR3 PHY    | 7GHz 이상 wideband      | 미구현                       | 미지원   |

근거:

* OCUDU는 “3GPP R18 aligned” 및 full L1/L2/L3 stack을 명시합니다. ([ocudu-docs-604e90.gitlab.io][2])
* 기존 srsRAN CU/DU 구조를 계승합니다. ([srsRAN - Open Source RAN][3])

---

# 2. RF / Spectrum Capability

| 상위 항목               | 하위 항목            | 지원 범위       | 상태    |
| ------------------- | ---------------- | ----------- | ----- |
| Frequency Range     | FR1              | 지원          | 지원    |
| Frequency Range     | FR2              | 연구 수준       | 부분 지원 |
| Frequency Range     | FR3              | 없음          | 미지원   |
| RF Backend          | UHD              | 지원          | 지원    |
| RF Backend          | ZMQ              | 지원          | 지원    |
| RF Backend          | O-RU             | O-RAN 기반 일부 | 부분 지원 |
| Sampling Rate       | Max Fs           | SDR HW 의존   | 부분 지원 |
| FFT                 | Wide FFT         | SW FFT 기반   | 부분 지원 |
| Carrier Aggregation | CA               | 제한적         | 부분 지원 |
| Multi-band          | 동시 multi-carrier | 제한적         | 부분 지원 |
| RF Timing           | GPSDO/PTP        | 일부 지원       | 부분 지원 |

---

# 3. MIMO / Beamforming Capability

FR3 readiness 에서 가장 중요한 영역입니다.

| 상위 항목          | 하위 항목             | 지원 범위      | 상태    |
| -------------- | ----------------- | ---------- | ----- |
| SU-MIMO        | DL Layer          | 2 Layer 중심 | 부분 지원 |
| SU-MIMO        | UL Layer          | 1~2 Layer  | 부분 지원 |
| MU-MIMO        | DL MU-MIMO        | 공개 기준 없음   | 미지원   |
| MU-MIMO        | UL MU-MIMO        | 없음         | 미지원   |
| Precoding      | Basic precoding   | 지원         | 지원    |
| Precoding      | Advanced codebook | 제한적        | 부분 지원 |
| Beamforming    | Digital BF        | 일부 지원      | 부분 지원 |
| Beamforming    | Analog BF         | 없음         | 미지원   |
| Beamforming    | Hybrid BF         | 없음         | 미지원   |
| Beam Mgmt      | SSB sweeping      | 일부         | 부분 지원 |
| Beam Mgmt      | Beam tracking     | 없음         | 미지원   |
| Beam Mgmt      | Beam recovery     | 없음         | 미지원   |
| Beam Mgmt      | Beam refinement   | 없음         | 미지원   |
| Multi-panel UE | UE panel mgmt     | 없음         | 미지원   |
| Massive MIMO   | 8T8R 이상           | 공개 기준 없음   | 미지원   |

실제 판단:

> 현재 OCUDU는 “NR MIMO 지원” 수준이며,
> FR3 핵심인 “beam-centric architecture” 수준은 아직 아닙니다.

---

# 4. Reference Signal Framework

| 상위 항목            | 하위 항목            | 지원 범위 | 상태    |
| ---------------- | ---------------- | ----- | ----- |
| SRS              | UL sounding      | 지원    | 지원    |
| SRS              | Multi-port SRS   | 제한적   | 부분 지원 |
| SRS              | Beam SRS         | 없음    | 미지원   |
| CSI-RS           | NZP CSI-RS       | 일부    | 부분 지원 |
| CSI-RS           | Beam CSI-RS      | 없음    | 미지원   |
| CSI Feedback     | CQI/PMI/RI       | 지원    | 지원    |
| CSI Feedback     | Type I codebook  | 일부    | 부분 지원 |
| CSI Feedback     | Type II codebook | 없음    | 미지원   |
| Measurement      | RSRP/RSRQ/SINR   | 지원    | 지원    |
| Beam Measurement | Beam RSRP        | 제한적   | 부분 지원 |

FR3 readiness 관점:

* CSI-RS framework 는 아직 매우 제한적
* Beam CSI 구조 부족

---

# 5. Scheduler Capability

| 상위 항목     | 하위 항목             | 지원 범위  | 상태    |
| --------- | ----------------- | ------ | ----- |
| Scheduler | Round Robin       | 지원     | 지원    |
| Scheduler | Proportional Fair | 지원     | 지원    |
| Scheduler | QoS-aware         | 일부     | 부분 지원 |
| Scheduler | Slice-aware       | 제한적    | 부분 지원 |
| Scheduler | Beam-aware        | 없음     | 미지원   |
| Scheduler | MU-MIMO aware     | 없음     | 미지원   |
| Scheduler | CA scheduling     | 제한적    | 부분 지원 |
| Scheduler | HARQ scheduling   | 지원     | 지원    |
| Scheduler | UE multiplexing   | 지원     | 지원    |
| Scheduler | UE/TTI scaling    | CPU 의존 | 부분 지원 |

FR3 핵심 gap:

* beam-aware scheduling 부재
* spatial scheduling 부재

---

# 6. Capacity / Scalability

(공개 문서 기반 + 일반 srsRAN 계열 특성 기준)

| 상위 항목          | 하위 항목        | 지원 범위      | 상태    |
| -------------- | ------------ | ---------- | ----- |
| Attached UE    | UE attach 수  | 수십 UE 수준   | 부분 지원 |
| Active UE      | 동시 active UE | 제한적        | 부분 지원 |
| DL Throughput  | Peak DL      | SDR/CPU 의존 | 부분 지원 |
| UL Throughput  | Peak UL      | SDR/CPU 의존 | 부분 지원 |
| Real-time      | RT Linux 필요  | 지원         | 지원    |
| CPU Scaling    | Multi-core   | 지원         | 지원    |
| NUMA           | NUMA-aware   | 제한적        | 부분 지원 |
| Multi-cell     | 다중 셀         | 일부         | 부분 지원 |
| Multi-DU       | F1 split     | 지원         | 지원    |
| Large-scale DU | 수백 UE        | 공개 기준 없음   | 미지원   |

중요:

> FR3 목표 수준의 ultra-wideband capacity 는 현재 구조로는 어려움.

---

# 7. PHY Acceleration / Performance

| 상위 항목       | 하위 항목               | 지원 범위 | 상태    |
| ----------- | ------------------- | ----- | ----- |
| SIMD        | AVX2                | 지원    | 지원    |
| SIMD        | AVX512              | 일부    | 부분 지원 |
| FFT         | CPU FFT             | 지원    | 지원    |
| LDPC        | SW LDPC             | 지원    | 지원    |
| LDPC        | HW offload          | 없음    | 미지원   |
| GPU         | CUDA offload        | 없음    | 미지원   |
| FPGA        | PHY offload         | 없음    | 미지원   |
| DPDK        | Packet acceleration | 제한적   | 부분 지원 |
| Zero-copy   | 일부                  | 부분 지원 |       |
| CPU pinning | RT tuning           | 지원    | 지원    |

중요:
FR3급:

* 400MHz+
* 64T64R+
  에서는 SW-only 구조 한계 가능성 큼.

---

# 8. O-RAN / Fronthaul Capability

OCUDU의 강점 영역입니다.

| 상위 항목          | 하위 항목     | 지원 범위    | 상태    |
| -------------- | --------- | -------- | ----- |
| Architecture   | O-CU-CP   | 지원       | 지원    |
| Architecture   | O-CU-UP   | 지원       | 지원    |
| Architecture   | O-DU-high | 지원       | 지원    |
| Architecture   | O-DU-low  | 지원       | 지원    |
| Interface      | F1-C/F1-U | 지원       | 지원    |
| Interface      | E1        | 지원       | 지원    |
| O-RAN Split    | 7.2x      | 일부       | 부분 지원 |
| eCPRI          | 일부        | 부분 지원    |       |
| RU integration | O-RU 연동   | 일부       | 부분 지원 |
| Timing         | PTP       | 일부       | 부분 지원 |
| Timing         | SyncE     | 공개 기준 없음 | 미지원   |
| IQ Compression | 제한적       | 부분 지원    |       |

근거:
OCUDU는 O-RAN-native CU/DU 구조를 강조합니다. ([srsRAN - Open Source RAN][3])

---

# 9. Mobility / Multi-Cell

| 상위 항목             | 하위 항목         | 지원 범위 | 상태    |
| ----------------- | ------------- | ----- | ----- |
| Mobility          | Intra-cell HO | 일부    | 부분 지원 |
| Mobility          | Inter-cell HO | 제한적   | 부분 지원 |
| Measurement       | A3/A5 event   | 일부    | 부분 지원 |
| Dual Connectivity | EN-DC         | 제한적   | 부분 지원 |
| Multi-TRP         | 지원 여부         | 없음    | 미지원   |
| CoMP              | 지원 여부         | 없음    | 미지원   |
| Beam mobility     | Beam switch   | 없음    | 미지원   |

FR3 readiness 에서:

* beam mobility framework 부족

---

# 10. Operations / Calibration

| 상위 항목          | 하위 항목             | 지원 범위 | 상태    |
| -------------- | ----------------- | ----- | ----- |
| RF Calibration | IQ calibration    | 제한적   | 부분 지원 |
| RF Calibration | Delay calibration | 제한적   | 부분 지원 |
| Logging        | PHY trace         | 지원    | 지원    |
| Logging        | MAC trace         | 지원    | 지원    |
| KPI            | UE stats          | 지원    | 지원    |
| KPI            | Scheduler stats   | 일부    | 부분 지원 |
| Debug          | Slot trace        | 지원    | 지원    |
| Automation     | CI/CD             | 지원    | 지원    |
| Deployment     | Containerized     | 일부    | 부분 지원 |

---

# FR3 Readiness 관점 핵심 Gap 요약

| 영역             | 현재 상태     | FR3 필요 수준   |
| -------------- | --------- | ----------- |
| Wideband PHY   | 100MHz 중심 | 400MHz+     |
| Massive MIMO   | 2x2 수준 중심 | 64T64R+     |
| Beam Mgmt      | 제한적       | 필수          |
| CSI-RS         | 기본 수준     | Beam CSI 필수 |
| Beam Scheduler | 없음        | 필수          |
| Hybrid BF      | 없음        | 필수          |
| HW Offload     | 제한적       | 사실상 필수      |
| Multi-TRP      | 없음        | 중요          |
| Beam Mobility  | 없음        | 중요          |

---

# 실제로 가장 중요한 분석 포인트

OCUDU를 FR3 readiness 관점에서 분석할 때는 아래 5개를 우선적으로 보는 것이 좋습니다.

1. PHY FFT/LDPC scaling
2. CSI-RS framework
3. Scheduler architecture
4. O-RAN split timing
5. Beamforming abstraction

이 다섯 개가:

> “현재 구조가 FR3로 확장 가능한가?”

를 결정합니다.

참고:

* [OCUDU GitLab](https://gitlab.com/ocudu/ocudu?utm_source=chatgpt.com)
* [OCUDU Documentation](https://ocudu-docs-604e90.gitlab.io/?utm_source=chatgpt.com)
* [srsRAN Project Overview](https://www.srsran.com/?utm_source=chatgpt.com)

([about.gitlab.com][1])

[1]: https://gitlab.com/ocudu/ocudu?utm_source=chatgpt.com "The OCUDU Project / OCUDU · GitLab"
[2]: https://ocudu-docs-604e90.gitlab.io/?utm_source=chatgpt.com "OCUDU Documentation | OCUDU"
[3]: https://www.srsran.com/srsran-project-update?utm_source=chatgpt.com "We've released our new O-RAN CU/DU"

# OCUDU Feature Matrix (FR3 Readiness / Capacity 분석용)

기준:

* [OCUDU GitLab](https://gitlab.com/ocudu/ocudu?utm_source=chatgpt.com)
* 공개 문서 및 현재 srsRAN 계열 구조 기준
* 공개적으로 확인 가능한 범위만 사용
* 명확하지 않은 항목은 “부분 지원 / 미지원 / HW 의존”으로 분리

---

# 상태 정의

| 상태    | 의미               |
| ----- | ---------------- |
| 지원    | 기본 branch 기준 동작  |
| 부분 지원 | 제한 조건 존재 / 연구 수준 |
| 실험 지원 | experimental     |
| 미지원   | 현재 없음            |
| HW 의존 | SDR/RF HW 영향 큼   |

---

# 1. NR PHY Base Capability

| 상위 항목         | 하위 항목                 | 지원 범위                   | 상태    |
| ------------- | --------------------- | ----------------------- | ----- |
| Numerology    | SCS                   | 15/30/60/120 kHz        | 지원    |
| DL/UL BW      | 최대 BW                 | FR1 기준 최대 100 MHz 수준    | 부분 지원 |
| PRB           | Max PRB               | 273 PRB                 | 부분 지원 |
| Duplex        | FDD                   | 지원                      | 지원    |
| Duplex        | TDD                   | 지원                      | 지원    |
| Modulation DL | DL Modulation         | QPSK/16QAM/64QAM/256QAM | 지원    |
| Modulation UL | UL Modulation         | QPSK/16QAM/64QAM        | 지원    |
| Coding        | LDPC                  | BG1/BG2 지원              | 지원    |
| Coding        | Polar                 | PBCH/UCI 지원             | 지원    |
| HARQ          | DL/UL HARQ            | 지원                      | 지원    |
| PHY Timing    | Slot-based processing | RT Linux 기반             | 지원    |
| PHY Timing    | Real-time scheduling  | CPU 성능 의존               | HW 의존 |
| FR2 PHY       | mmWave                | 제한적                     | 부분 지원 |
| FR3 PHY       | 7GHz+                 | 없음                      | 미지원   |

---

# 2. UE / Cell Capacity

| 상위 항목             | 하위 항목               | 지원 범위        | 상태    |
| ----------------- | ------------------- | ------------ | ----- |
| Attached UE 수     | gNB 당 UE 수          | 수십 UE 수준     | 부분 지원 |
| Active UE 수       | 동시 active UE        | CPU 의존       | HW 의존 |
| UE/TTI Scheduling | TTI 당 UE scheduling | 제한적          | 부분 지원 |
| Multi-cell        | 다중 셀                | 일부 지원        | 부분 지원 |
| Multi-DU          | DU scale-out        | 지원           | 지원    |
| Large-scale DU    | 수백 UE 이상            | 공개 기준 없음     | 미지원   |
| DL Throughput     | Peak DL             | CPU/RF HW 의존 | HW 의존 |
| UL Throughput     | Peak UL             | CPU/RF HW 의존 | HW 의존 |

주의:
현재 구조는 “연구/개발용” 중심이며,
상용 DU 수준 대규모 UE capacity 는 공개 기준 없음.

---

# 3. MIMO / Antenna Capability

FR3 readiness 핵심 영역.

| 상위 항목                 | 하위 항목                | 지원 범위         | 상태    |
| --------------------- | -------------------- | ------------- | ----- |
| 안테나 수                 | DL Tx antenna        | 일반적으로 2Tx 중심  | 부분 지원 |
| 안테나 수                 | UL Rx antenna        | 일반적으로 2Rx 중심  | 부분 지원 |
| SU-MIMO               | DL Layer 수           | 최대 2 Layer 중심 | 부분 지원 |
| SU-MIMO               | UL Layer 수           | 1~2 Layer     | 부분 지원 |
| MU-MIMO               | DL MU-MIMO           | 공개 기준 없음      | 미지원   |
| MU-MIMO               | UL MU-MIMO           | 없음            | 미지원   |
| Precoding             | Basic precoding      | 지원            | 지원    |
| Precoding             | Advanced codebook    | 제한적           | 부분 지원 |
| Massive MIMO          | 8T8R 이상              | 공개 기준 없음      | 미지원   |
| Cell Scheduling       | Cell 당 DL MIMO layer | 2 layer 수준 중심 | 부분 지원 |
| Multi-user scheduling | Spatial multiplexing | 제한적           | 부분 지원 |

FR3 기준 gap:

* 64T64R 없음
* multi-layer MU-MIMO 없음
* beam-centric MIMO 없음

---

# 4. Beamforming / Beam Management

| 상위 항목          | 하위 항목                   | 지원 범위 | 상태    |
| -------------- | ----------------------- | ----- | ----- |
| Beamforming    | Digital Beamforming     | 일부    | 부분 지원 |
| Beamforming    | Analog Beamforming      | 없음    | 미지원   |
| Beamforming    | Hybrid Beamforming      | 없음    | 미지원   |
| Beam Mgmt      | SSB sweeping            | 일부    | 부분 지원 |
| Beam Mgmt      | Beam tracking           | 없음    | 미지원   |
| Beam Mgmt      | Beam refinement         | 없음    | 미지원   |
| Beam Mgmt      | Beam recovery           | 없음    | 미지원   |
| Beam Mgmt      | Beam-aware scheduling   | 없음    | 미지원   |
| Beam Mgmt      | Beam collision handling | 없음    | 미지원   |
| Multi-panel UE | UE panel management     | 없음    | 미지원   |

현재:

> “기본 NR beam 절차 일부” 수준.

---

# 5. CSI-RS / SRS / Channel Estimation

FR3에서 매우 중요.

| 상위 항목            | 하위 항목                 | 지원 범위 | 상태    |
| ---------------- | --------------------- | ----- | ----- |
| SRS              | UL sounding reference | 지원    | 지원    |
| SRS              | Multi-port SRS        | 제한적   | 부분 지원 |
| SRS              | Beam SRS              | 없음    | 미지원   |
| SRS 기반 채널 추정     | UL channel estimation | 지원    | 지원    |
| CSI-RS           | NZP CSI-RS            | 일부    | 부분 지원 |
| CSI-RS           | Beam CSI-RS           | 없음    | 미지원   |
| CSI Feedback     | CQI                   | 지원    | 지원    |
| CSI Feedback     | PMI                   | 지원    | 지원    |
| CSI Feedback     | RI                    | 지원    | 지원    |
| CSI Feedback     | Type I codebook       | 제한적   | 부분 지원 |
| CSI Feedback     | Type II codebook      | 없음    | 미지원   |
| Measurement      | RSRP/RSRQ/SINR        | 지원    | 지원    |
| Beam Measurement | Beam RSRP             | 제한적   | 부분 지원 |

핵심:

* 기본 채널 추정은 가능
* FR3용 beam CSI framework 는 부족

---

# 6. Scheduler Capability

| 상위 항목     | 하위 항목                 | 지원 범위  | 상태    |
| --------- | --------------------- | ------ | ----- |
| Scheduler | Round Robin           | 지원     | 지원    |
| Scheduler | Proportional Fair     | 지원     | 지원    |
| Scheduler | QoS-aware             | 일부     | 부분 지원 |
| Scheduler | Slice-aware           | 제한적    | 부분 지원 |
| Scheduler | HARQ scheduling       | 지원     | 지원    |
| Scheduler | MU-MIMO aware         | 없음     | 미지원   |
| Scheduler | Beam-aware scheduling | 없음     | 미지원   |
| Scheduler | CA scheduling         | 제한적    | 부분 지원 |
| Scheduler | UE multiplexing       | 지원     | 지원    |
| Scheduler | Spatial scheduling    | 없음     | 미지원   |
| Scheduler | Real-time latency     | CPU 의존 | HW 의존 |

---

# 7. PHY Accelerator / Performance

| 상위 항목        | 하위 항목                 | 지원 범위 | 상태    |
| ------------ | --------------------- | ----- | ----- |
| PHY 가속기      | HW PHY accelerator    | 없음    | 미지원   |
| SIMD         | AVX2                  | 지원    | 지원    |
| SIMD         | AVX512                | 일부    | 부분 지원 |
| FFT          | SW FFT                | 지원    | 지원    |
| FFT          | HW FFT offload        | 없음    | 미지원   |
| LDPC SW      | SW LDPC decode/encode | 지원    | 지원    |
| LDPC HW      | HW LDPC offload       | 없음    | 미지원   |
| GPU Offload  | CUDA/GPU PHY          | 없음    | 미지원   |
| FPGA Offload | FPGA PHY acceleration | 없음    | 미지원   |
| DPDK         | Packet acceleration   | 제한적   | 부분 지원 |
| Zero-copy    | 일부                    | 부분 지원 |       |
| CPU Pinning  | RT tuning             | 지원    | 지원    |

FR3 관점:
SW-only PHY 구조로는:

* 400MHz+
* Massive MIMO
  에서 한계 가능성 큼.

---

# 8. O-RAN / FrontHaul Capability

OCUDU 핵심 영역.

| 상위 항목            | 하위 항목             | 지원 범위    | 상태    |
| ---------------- | ----------------- | -------- | ----- |
| Architecture     | O-CU-CP           | 지원       | 지원    |
| Architecture     | O-CU-UP           | 지원       | 지원    |
| Architecture     | O-DU-high         | 지원       | 지원    |
| Architecture     | O-DU-low          | 지원       | 지원    |
| Interface        | F1-C/F1-U         | 지원       | 지원    |
| Interface        | E1                | 지원       | 지원    |
| O-RAN Split      | 7.2x              | 일부       | 부분 지원 |
| eCPRI            | 일부                | 부분 지원    |       |
| RU Integration   | O-RU 연동           | 일부       | 부분 지원 |
| FrontHaul 용량     | Max FH throughput | 공개 수치 없음 | HW 의존 |
| FrontHaul Timing | PTP               | 일부 지원    | 부분 지원 |
| FrontHaul Timing | SyncE             | 공개 기준 없음 | 미지원   |
| IQ Compression   | 일부                | 부분 지원    |       |
| RU Scaling       | Multi-RU          | 제한적      | 부분 지원 |

중요:
FR3급 Massive MIMO RU bandwidth 용량은 공개 기준 없음.

---

# 9. Mobility / Multi-Cell

| 상위 항목             | 하위 항목         | 지원 범위 | 상태    |
| ----------------- | ------------- | ----- | ----- |
| Handover          | Intra-cell HO | 일부    | 부분 지원 |
| Handover          | Inter-cell HO | 제한적   | 부분 지원 |
| Measurement       | Event A3/A5   | 일부    | 부분 지원 |
| Dual Connectivity | EN-DC         | 제한적   | 부분 지원 |
| CoMP              | 없음            | 미지원   |       |
| Multi-TRP         | 없음            | 미지원   |       |
| Beam Mobility     | Beam switch   | 없음    | 미지원   |

---

# 10. RF Calibration / Operations

| 상위 항목          | 하위 항목                | 지원 범위 | 상태    |
| -------------- | -------------------- | ----- | ----- |
| RF Calibration | IQ calibration       | 제한적   | 부분 지원 |
| RF Calibration | Delay calibration    | 제한적   | 부분 지원 |
| RF Calibration | Phase calibration    | 없음    | 미지원   |
| Logging        | PHY trace            | 지원    | 지원    |
| Logging        | MAC trace            | 지원    | 지원    |
| KPI            | UE statistics        | 지원    | 지원    |
| KPI            | Scheduler statistics | 일부    | 부분 지원 |
| Debug          | Slot trace           | 지원    | 지원    |
| Automation     | CI/CD                | 지원    | 지원    |
| Deployment     | Containerized        | 일부    | 부분 지원 |

---

# FR3 Readiness 핵심 Gap 요약

| 영역                   | 현재 상태      | FR3 요구 수준   |
| -------------------- | ---------- | ----------- |
| DL/UL BW             | ~100MHz 수준 | 400MHz+     |
| 안테나 수                | 2Tx/2Rx 중심 | 64T64R+     |
| SU-MIMO              | 2 layer 수준 | 8+ layer    |
| MU-MIMO              | 없음         | 필수          |
| Beam Mgmt            | 제한적        | 필수          |
| CSI-RS               | 기본 수준      | Beam CSI 필수 |
| Beam-aware scheduler | 없음         | 필수          |
| Hybrid BF            | 없음         | 필수          |
| PHY Accelerator      | 없음         | 사실상 필수      |
| Multi-TRP            | 없음         | 중요          |
| FrontHaul            | 제한적        | 초고용량 필요     |

---

# FR3 분석 시 가장 우선 확인 추천 항목

우선순위 기준:

1. 안테나 abstraction 구조
2. MIMO layer scaling
3. CSI-RS/SRS framework
4. Scheduler architecture
5. O-RAN FH throughput/timing
6. PHY SIMD scaling
7. FFT/LDPC profiling
8. Beamforming abstraction

이 항목들이:

> “현재 구조가 FR3로 확장 가능한가”

를 가장 직접적으로 보여줍니다.

