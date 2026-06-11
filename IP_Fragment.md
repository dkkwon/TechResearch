


네, 충분히 구현 가능합니다. 오히려 질문에 적어주신 환경이라면 IP Fragment를 커널에 맡기지 않고 Application Layer에서 직접 Fragment/Reassembly 하는 것이 일반적으로 더 권장되는 방법입니다.

특히 DPDK, FPGA, SmartNIC, Packet Accelerator, RAN 장비 등에서는 Linux IP Fragment 기능을 거의 사용하지 않고 자체 구현하는 경우가 많습니다.

구현 가능 여부
제시한 API:

Send(IP_HDR, UDP_HDR, Payload)
Recv(IP_HDR, UDP_HDR, Payload)
만으로도 구현 가능합니다.

송신 측:

원본 Message (100KB)

↓ Fragmentation

Fragment #0
IP_HDR
UDP_HDR
FRAG_HDR
Payload[0~1399]

Fragment #1
IP_HDR
UDP_HDR
FRAG_HDR
Payload[1400~2799]

...

Fragment #N
수신 측:

Recv()

↓

FRAG_HDR 분석

↓

Reassembly Buffer 저장

↓

모든 Fragment 수신

↓

원본 Message 복구

↓

상위 Application 전달
IP Fragment를 그대로 모방할 필요는 없음
Linux IP Fragment처럼

Identification
Fragment Offset
MF(More Fragment)
를 사용할 수도 있지만,

Application 전용이라면 더 단순하게 만드는 것이 좋습니다.

예:

struct FragHdr
{
    uint32_t msg_id;
    uint16_t frag_idx;
    uint16_t frag_cnt;
};
예시

100KB 메시지

msg_id = 1234

frag_cnt = 4

Fragment0
frag_idx=0

Fragment1
frag_idx=1

Fragment2
frag_idx=2

Fragment3
frag_idx=3
수신측:

bitmap[frag_cnt]

frag_idx 수신 시

bitmap[frag_idx] = 1

모든 비트가 1

→ Reassembly 완료
IP Fragment보다 훨씬 구현이 쉽습니다.

msg_id 충돌 문제
이전 대화에서 질문하셨던 내용과 연결됩니다.

여러 송신자가 존재하면

Sender A
msg_id = 100

Sender B
msg_id = 100
가 발생할 수 있습니다.

따라서 Reassembly Key는

Src IP
Dst IP
Src Port
Dst Port
msg_id
조합을 사용해야 합니다.

예:

struct ReassemblyKey
{
    uint32_t src_ip;
    uint32_t dst_ip;

    uint16_t src_port;
    uint16_t dst_port;

    uint32_t msg_id;
};
그러면 msg_id가 중복되어도 충돌하지 않습니다.

Timeout 처리
RAN 환경이라면 일반 서버와 다르게 Timeout을 매우 짧게 잡을 수 있습니다.

예:

0.5ms
1ms
2ms
5ms
수준도 가능합니다.

Reassembly Table:

msg_id = 100
fragment 0 수신
fragment 1 수신

2ms 경과

fragment 2 없음

→ 폐기
이렇게 하면 메모리 사용량을 제한할 수 있습니다.

순서가 바뀌어 도착하는 경우
UDP에서는

2
0
1
3
순서로 도착할 수 있습니다.

따라서

frag_idx
기반 저장이 필요합니다.

buffer[frag_idx]
형태로 저장하면 문제 없습니다.

Fragment 유실 처리
예:

0
1
3
4
수신

2
유실

그러면

Reassembly 실패
처리해야 합니다.

IP Fragment와 동일합니다.

실제 RAN/DPDK 장비에서 많이 쓰는 방식
IP Fragment 헤더를 그대로 사용하는 경우보다

다음과 같은 전용 헤더를 사용하는 경우가 훨씬 많습니다.

struct FragHdr
{
    uint32_t msg_id;

    uint16_t frag_idx;
    uint16_t frag_cnt;

    uint32_t total_len;
};
또는

struct FragHdr
{
    uint32_t msg_id;

    uint16_t offset;
    uint16_t len;

    uint8_t  last_frag;
};
이 방식이:

DPDK

O-RAN

DU/CU 내부 인터페이스

FPGA ↔ CPU 인터페이스

User-space Networking

에서 흔히 사용됩니다.

추천
질문하신 환경이라면 IP Fragment 형식을 재현하지 말고, UDP Payload 앞에 전용 Fragment Header를 추가하는 방식이 가장 적합합니다.

예를 들면:

struct FragHdr
{
    uint32_t msg_id;
    uint16_t frag_idx;
    uint16_t frag_cnt;
    uint32_t total_len;
};
Reassembly Key:

(src_ip,
 dst_ip,
 src_port,
 dst_port,
 msg_id)
Reassembly Timeout:

1~5ms
(RAN 요구사항에 맞게 조정)
이 구조가 Linux Kernel, DPDK, HW Accelerator, FPGA 등 어떤 전송 엔진을 사용하더라도 동일하게 동작하므로 가장 이식성이 높고 유지보수하기 쉽습니다.


Send(), Recv() API 는 어플리케이션이 제공하는 라이브러리입니다.
IP/UDP 패킷의 헤더를 채워주고 패킷을 전송하는 역할입니다.
App_A(Kernel Application) <-> App_B(HW Accelator) 에서 App_B 에서 기능 구현을 해서 App_A 에서는 App_B 가 커널을 쓰는 것과 동일한 메시지를 주고 받는 것이 목표입니다. 

그 목적이라면 설계 방향이 조금 달라집니다.

앞서 설명드린 "Application Layer Fragmentation"은 App_A와 App_B가 모두 해당 프로토콜을 이해할 때 적합한 방법입니다.

하지만 지금 설명하신 구조는

App_A (Linux Kernel 사용)
    ↓
 IP
 UDP
 Kernel Fragmentation

=======================

App_B (HW Accelerator)
    ↓
 Send()
 Recv()
에서

"App_A 입장에서는 상대방이 Linux Kernel을 사용하는 것처럼 보이게 만들고 싶다"

가 목표입니다.

그렇다면 App_B에서 Application Fragmentation을 구현하면 안 됩니다.

대신 App_B 내부에서 IP Fragmentation/Reassembly를 에뮬레이션 해야 합니다.

Linux Kernel 관점
Linux App은

sendto(sock, buf, 100KB);
만 호출합니다.

그러면 Kernel은

UDP Header

+
100KB Payload

↓

IP Layer

↓

Fragment 생성
을 수행합니다.

예를 들어 MTU=1500이면

IP Frag #0
IP Frag #1
IP Frag #2
...
가 전송됩니다.

수신측 Linux Kernel은

IP Reassembly

↓

UDP Layer

↓

recvfrom()

↓

100KB 원본 데이터
를 전달합니다.

Application은 Fragment 존재 자체를 모릅니다.

App_B가 해야 하는 일
App_B가 Kernel을 대체하려면

Recv()에서 받은 Fragment를

IP Reassembly

↓

UDP Datagram 복구

↓

상위 App 전달
해야 합니다.

즉 App_B 내부에

Mini IP Layer
가 필요합니다.

가장 중요한 차이
앞서 설명한 방식

UDP Payload 안에

msg_id
frag_idx
frag_cnt
삽입
→ Application Fragmentation


은 Linux Kernel과 호환되지 않습니다.

Linux는 해당 헤더를 이해하지 못합니다.

반면

```text
IP Identification
IP Fragment Offset
IP MF Flag
를 사용하면

Linux와 완전 호환됩니다.

송신 방향
App_B → App_A

원본 메시지

100KB
수신

↓

App_B 내부

IP Identification 생성

Fragment Offset 계산

MF 설정
↓

여러 개의 IP Fragment 생성

↓

Send()


App_A Kernel

```text
IP Reassembly

↓

recvfrom()

↓

100KB
수신 방향
App_A → App_B

Linux Kernel

sendto(100KB)
↓

IP Fragment 생성

↓

App_B Recv()

Fragment #0
Fragment #1
Fragment #2
...
↓

App_B Reassembly

↓

UDP Datagram 복구

↓

상위 Application 전달


---

# 구현 난이도

사실상 필요한 것은 Linux의

```text
net/ipv4/ip_fragment.c
기능의 일부입니다.

필요한 필드:

struct ipv4_frag_key
{
    uint32_t src_ip;
    uint32_t dst_ip;

    uint16_t id;

    uint8_t protocol;
};
Reassembly Key:

src_ip
dst_ip
id
protocol
주의할 점
Linux와 동일하게 동작하려면

Fragment Offset 단위가

8-byte 단위
입니다.

예:

Offset=0

Offset=185
이면

185 * 8 = 1480 bytes
위치를 의미합니다.

이 부분을 잘못 구현하면 Linux와 호환되지 않습니다.

RAN 장비 관점에서의 추천
목표가

App_A (Linux Kernel)

↔

App_B (HW Accelerator)
간 완전한 상호운용성이라면,

전용 Fragment Header를 만들지 말고 RFC 791 IPv4 Fragmentation을 그대로 구현하는 것이 맞습니다.

즉 App_B 내부에

IP Fragmentation Engine
IP Reassembly Engine
을 두고

Send()/Recv() API 위에서 동작시키는 구조가 가장 적합합니다.

이렇게 하면 App_A는 상대방이 Linux Kernel인지 HW Accelerator인지 전혀 구분할 수 없고, sendto()/recvfrom() 수준에서 동일한 동작을 보장할 수 있습니다.

다만 구현 전에 한 가지 확인이 필요합니다.

Send() API가 "IP 헤더를 사용자가 직접 채워서 보내는 Raw IP 수준"인지, 아니면 "UDP Payload만 넘기면 아래 계층이 IP 헤더를 생성하는 수준"인지에 따라 구현 방식이 크게 달라집니다.

현재 설명으로는 전자(Raw IP에 가까운 형태)로 보이는데, 맞다면 Linux의 IPv4 Fragmentation을 거의 그대로 구현할 수 있습니다.


Send() API를 두 가지 방식 모두 구현 가능합니다.  

그렇다면 설계 선택권이 생깁니다.

제가 보기에는 현재 목표가

App_A (Linux Kernel)
        ↔
App_B (HW Accelerator)
에서

App_A는 상대방이 Linux Kernel인지 HW Accelerator인지 알 수 없어야 함

이라면 RFC791 IPv4 Fragmentation/Reassembly를 구현하는 것이 가장 바람직합니다.

다만 실제 구현 관점에서는 3가지 선택지가 있습니다.

1. Linux와 완전 호환 (추천)
App_B 내부

Large UDP Message

↓
IPv4 Fragmentation

↓
Send(IP Header, UDP Header, Fragment)

====================

Recv(Fragment)

↓
IPv4 Reassembly

↓
Complete UDP Datagram
특징

Linux ↔ Linux 와 동일

App_A 수정 없음

Wireshark 분석 가능

표준 IP Fragment 사용

구현해야 하는 필드

IP Identification
IP Fragment Offset
IP MF Flag
Reassembly Key

(src_ip,
 dst_ip,
 protocol,
 identification)
장점

가장 깨끗한 구조

향후 Linux 장비 추가되어도 문제 없음

단점

구현량이 가장 많음

2. App_B 내부 전용 Fragmentation
예:

struct FragHdr
{
    uint32_t msg_id;
    uint16_t frag_idx;
    uint16_t frag_cnt;
};
사용

장점

구현 매우 단순

단점

Linux Kernel
   ↔
HW Accelerator
직접 연동 불가

중간 변환기가 필요

따라서 현재 요구사항에는 부적합

3. Hybrid 방식 (실무에서 가장 많이 사용)
실제로 DPDK, FPGA, SmartNIC에서는 이것을 많이 합니다.

외부 인터페이스

RFC791 Fragment
내부 구현

msg_id
frag_idx
frag_cnt
형태 사용

예:

수신

IP Fragment

↓
Internal Fragment Descriptor

msg_id
frag_idx
frag_cnt
변환

↓

Reassembly

송신

Internal Fragment Descriptor

↓

RFC791 Fragment 생성

↓

Send()
현재 환경에서 제가 추천하는 구조
Send API가

Send(
    IpHeader* ip,
    UdpHeader* udp,
    void* payload,
    uint32_t len);
수준이라면

Fragmentation Layer를 그 위에 하나 추가합니다.

Upper Application

↓
Fragmentation Layer

↓
Send()
송신

100KB UDP Datagram

↓

MTU 1500

↓

68 Fragments 생성

↓

Send() 68회 호출
수신

Recv()

↓

Fragment 검사

↓

Reassembly Table 저장

↓

Complete Datagram 생성

↓

Upper Application 전달
RAN 환경에서 추가 고려사항
질문하신 환경은 일반 서버가 아니라

DU
CU
RU
L2/L3
User Plane
과 비슷한 것으로 보입니다.

이 경우 Linux와 동일하게 구현하더라도 보통 추가 기능을 넣습니다.

예:

struct FragContext
{
    uint64_t timestamp;

    uint32_t received_len;

    uint32_t total_len;

    uint64_t bitmap;
};
그리고

1ms
2ms
5ms
안에 완성 안 되면 폐기

추가로 추천하는 부분
만약 App_A가 Linux이고 App_B가 HW Accelerator라면,

IP Fragmentation을 새로 구현하기 전에 Path MTU를 맞춰서 Fragment 자체를 없앨 수 있는지 먼저 검토하는 것이 좋습니다.

통신장비 업계에서는

Jumbo Frame (9000B)

또는

MTU 4000~8000
를 사용해서 Fragment 발생을 최소화하는 경우가 많습니다.

왜냐하면 Fragment Reassembly는

메모리 사용

Timeout 관리

DoS 취약성

Out-of-order 처리

때문에 생각보다 복잡하기 때문입니다.

현재 설명하신 목표만 놓고 보면,

"Linux Kernel과 동일한 동작을 제공해야 한다"가 최우선이면 RFC791 IPv4 Fragmentation/Reassembly를 구현하는 것이 정답이고, 내부 구현은 Linux와 동일하게 할 필요 없이 더 단순한 자료구조(msg_id, bitmap 기반)로 구현해도 전혀 문제 없습니다.


위의 IPv4 Fragment 구현을 위한 오픈소스나 참고할만한 코드가 있을까요?

네. IPv4 Fragmentation/Reassembly는 워낙 오래된 기능이라 참고할 수 있는 구현체가 많습니다.

질문하신 환경(HW Accelerator


답변이 중단되었습니다.
답변이 중단되었어요

답변이 중간에 잘렸네요. 이어서 설명드리겠습니다.

질문하신 환경(HW Accelerator + Linux 호환)이면 Linux Kernel 전체 구현을 가져오기보다는 DPDK나 BSD 계열 구현을 참고하는 것이 훨씬 좋습니다.

1. 가장 추천: DPDK IP Fragment Library
DPDK에는 IPv4 Fragmentation/Reassembly가 이미 구현되어 있습니다.

DPDK 공식 사이트

관련 소스:

lib/ip_frag/
    rte_ip_frag.h
    rte_ipv4_fragmentation.c
    rte_ip_frag_common.h
특히

rte_ipv4_fragment_packet()
rte_ipv4_frag_reassemble_packet()
가 핵심 API입니다.

장점:

User Space 구현

Linux Kernel 의존성 없음

DPDK 없이도 로직 참고 가능

성능 최적화가 잘 되어 있음

질문하신 구조와 가장 유사합니다.

2. Linux Kernel 구현
Linux는 다음 파일을 참고하면 됩니다.

Linux Kernel 소스

주요 파일:

net/ipv4/ip_fragment.c
송신:

net/ipv4/ip_output.c
핵심 함수:

ip_defrag()
ip_frag_queue()
ip_frag_reasm()
하지만 Linux 코드는

RCU
spinlock
skb
namespace
netfilter
등이 섞여 있어서 읽기 어렵습니다.

참고는 가능하지만 그대로 가져오기는 힘듭니다.

3. FreeBSD 구현
FreeBSD 소스 저장소

관련 파일:

sys/netinet/ip_reass.c
sys/netinet/ip_input.c
Linux보다 훨씬 단순합니다.

특히 Reassembly 알고리즘을 이해하기 좋습니다.

4. lwIP
Embedded 환경이라면 가장 읽기 쉽습니다.

lwIP 프로젝트

관련 파일:

src/core/ipv4/ip4_frag.c
크기가 매우 작습니다.

대략

Fragmentation
300~400줄

Reassembly
500~600줄
정도라서 구조 파악에 좋습니다.

질문하신 환경에 적합한 구현
사실 RAN 장비나 HW Accelerator에서는 Linux처럼 구현하지 않습니다.

보통 아래와 같은 구조를 사용합니다.

struct FragKey
{
    uint32_t src_ip;
    uint32_t dst_ip;

    uint16_t ip_id;
    uint8_t  protocol;
};
struct FragContext
{
    uint64_t timestamp;

    uint32_t total_len;

    uint32_t received_len;

    uint64_t bitmap;

    uint8_t* buffer;
};
수신 시

IP Fragment

↓

Hash Lookup

↓

Fragment Copy

↓

Bitmap Update

↓

Complete ?

↓

UDP Message 전달
입니다.

Linux와 완전 호환하려면
최소한 아래 필드는 해석해야 합니다.

IPv4 Header:

Identification
Fragment Offset
MF Flag
Total Length
Protocol
Source Address
Destination Address
실제로는 이것만 있으면 됩니다.

TTL, TOS 등은 Reassembly에 필요 없습니다.

구현 난이도
질문하신 목적이라면 Linux Kernel 수준 전체 구현이 필요하지 않습니다.

필수 기능만 구현하면:

IPv4 Fragmentation
  약 500줄

IPv4 Reassembly
  약 1000~1500줄
정도 규모로 충분합니다.

특히 RAN 환경이면

Fragment 중복 처리 간소화

Overlap Fragment 미지원

Timeout 1~5ms

최대 Fragment 수 제한

등을 적용하면 더 단순해집니다.

제 경험상이라면 참고 순서를 다음과 같이 추천합니다.

lwIP의 ip4_frag.c → 전체 알고리즘 이해

DPDK lib/ip_frag → 실제 고성능 구현 참고

Linux ip_fragment.c → Linux 호환 세부 동작 확인

이 순서가 가장 효율적입니다. 특히 현재 설명하신 "App_B가 Linux Kernel처럼 보이게 만들기" 목표에는 DPDK의 rte_ipv4_frag_reassemble_packet() 구현이 거의 가장 가까운 참고 코드입니다.


DPDK 코드의 구현을 참고하려면 어떤 파일을 참조하면 될까요? 재사용 가능한 코드도 있나요?

네. 질문하신 목적이라면 DPDK 전체를 가져다 쓰기보다는 IP Fragmentation/Reassembly 부분만 발췌해서 참고하는 것이 좋습니다.

우선 DPDK에서 실제로 공개 API로 제공하는 함수는 다음과 같습니다. 

rte_ipv4_fragment_packet()
rte_ipv4_frag_reassemble_packet()
꼭 봐야 하는 파일
DPDK 소스 트리 기준

lib/ip_frag/
├── rte_ip_frag.h
├── rte_ip_frag_common.h
├── rte_ipv4_fragmentation.c
├── rte_ip_frag_internal.c
├── rte_ip_frag_tbl.c
└── rte_ip_frag_common.c
정도입니다.

역할을 나누면

파일	역할
rte_ipv4_fragmentation.c	송신 Fragmentation
rte_ip_frag_internal.c	Reassembly 핵심
rte_ip_frag_tbl.c	Fragment Table 관리
rte_ip_frag_common.h	자료구조 정의
rte_ip_frag.h	공개 API
입니다. 

송신 측 참고 코드
가장 먼저 볼 파일

rte_ipv4_fragmentation.c
여기에는 사실상 RFC791 Fragmentation 알고리즘이 들어 있습니다.

핵심 로직:

payload_per_frag =
    (mtu - ip_hdr_len) & ~7;
즉

MTU = 1500

IP Header = 20

1480 Byte
로 맞추고

8 Byte Alignment
를 보장합니다.

그 다음

fragment_offset
MF Flag
Total Length
Header Checksum
을 수정하며 Fragment를 생성합니다.

질문하신 App_B 구현에서는 거의 그대로 참고 가능합니다.

수신 측 참고 코드
실제로 더 중요한 부분입니다.

rte_ip_frag_internal.c
여기를 가장 먼저 읽는 것을 추천합니다.

핵심 함수는

rte_ipv4_frag_reassemble_packet()
입니다. 

동작은

Fragment 수신

↓

Hash Key 생성

↓

Fragment Table 조회

↓

Fragment 저장

↓

모든 Fragment 수신 확인

↓

Reassembly

↓

UDP Datagram 반환
입니다.

특히 참고할 자료구조
오래된 DPDK 버전 문서에는 내부 자료구조가 노출되어 있습니다.

예:

struct ip_frag_key
{
    uint64_t src_dst[4];
    uint32_t id;
    uint32_t key_len;
};
struct ip_frag
{
    uint16_t ofs;
    uint16_t len;
};
struct ip_frag_pkt
{
    struct ip_frag_key key;
    ...
};
등입니다. 

이걸 보면 DPDK도 결국

src ip
dst ip
ip identification
protocol
조합으로 Reassembly Context를 관리한다는 것을 알 수 있습니다.

재사용 가능한 코드인가?
결론부터 말씀드리면

부분 재사용은 가능하지만 그대로 가져다 쓰기는 어렵습니다.

이유는

struct rte_mbuf
의존성이 매우 큽니다.

예:

rte_ipv4_frag_reassemble_packet(
    struct rte_ip_frag_tbl *tbl,
    struct rte_ip_frag_death_row *dr,
    struct rte_mbuf *mb,
    ...
)
처럼 모든 API가

rte_mbuf
rte_hash
rte_cycles
rte_malloc
에 의존합니다. 

App_B에 적합한 방법
현재 설명하신 구조라면

App_B
  |
  +-- Send()
  +-- Recv()
정도만 있으므로

DPDK 코드를 그대로 포팅하기보다

알고리즘만 가져오는 것이 좋습니다.

즉

struct FragKey
{
    uint32_t src_ip;
    uint32_t dst_ip;

    uint16_t ip_id;
    uint8_t protocol;
};
struct FragContext
{
    uint64_t timestamp;

    uint32_t total_len;
    uint32_t received_len;

    uint8_t* reassembly_buf;

    uint64_t bitmap;
};
형태로 새로 작성하는 편이 훨씬 간단합니다.

제가 추천하는 참고 순서
현재 프로젝트 기준

1단계
DPDK

rte_ipv4_fragmentation.c
→ Fragmentation 알고리즘 확인

2단계
DPDK

rte_ip_frag_internal.c
→ Reassembly 알고리즘 확인

3단계
Linux

net/ipv4/ip_fragment.c
→ Linux와 동일 동작 확인

구현 범위 추정
질문하신 App_B가

Linux Kernel과 상호 운용
UDP만 사용
IPv4만 사용
RAN 환경
이라면 실제 필요한 기능은

IPv4 Fragmentation      300~500줄
IPv4 Reassembly         700~1200줄
Hash Table              200줄
Timeout                 100줄
정도입니다.

따라서 Linux 코드 1만 줄 이상을 분석할 필요는 없고,

DPDK의 rte_ipv4_fragmentation.c + rte_ip_frag_internal.c 두 파일만 제대로 이해해도 구현의 80~90%는 해결됩니다. 특히 App_B처럼 Kernel bypass 환경에서는 Linux보다 DPDK 구현이 훨씬 좋은 참고 자료입니다. 


