네, 가능합니다.

오히려 현재 하시려는 **"Linux Kernel ↔ App_B(HW Accelerator) Fragmentation/Reassembly 호환성 검증"** 용도라면 Linux에서 Raw Socket을 사용하여 직접 IP 헤더를 작성하는 것이 가장 좋은 테스트 방법입니다.

---

# 방법 1. Raw IP Socket (추천)

소켓 생성

```c
int fd = socket(AF_INET, SOCK_RAW, IPPROTO_UDP);
```

그리고

```c
int on = 1;
setsockopt(fd,
           IPPROTO_IP,
           IP_HDRINCL,
           &on,
           sizeof(on));
```

설정하면

```text
User Space
    ↓
IP Header
UDP Header
Payload
```

전체를 직접 작성해서 전송할 수 있습니다.

예:

```c
struct iphdr  *ip;
struct udphdr *udp;

ip->id      = htons(1234);
ip->frag_off= htons(IP_MF);
ip->protocol= IPPROTO_UDP;

udp->source = htons(5000);
udp->dest   = htons(6000);
```

이런 식으로 설정 가능합니다.

---

# Fragment까지 직접 생성 가능

예를 들어

```text
Original UDP Datagram
4000 Bytes
```

를

```text
Fragment #0
Offset=0
MF=1

Fragment #1
Offset=185
MF=1

Fragment #2
Offset=370
MF=0
```

로 직접 생성하여

```c
sendto()
sendto()
sendto()
```

하면 됩니다.

Linux는 그대로 네트워크로 전송합니다.

즉 App_B에서 구현할 Reassembly를 Linux 상에서 먼저 검증할 수 있습니다.

---

# 방법 2. AF_PACKET

더 아래 계층까지 제어하려면

```c
socket(AF_PACKET,
       SOCK_RAW,
       htons(ETH_P_IP));
```

를 사용합니다.

그러면

```text
Ethernet Header
IP Header
UDP Header
Payload
```

전체를 직접 작성합니다.

```text
L2
L3
L4
```

모두 제어 가능합니다.

---

# 질문하신 테스트 목적에는?

저는 AF_PACKET보다는

```text
SOCK_RAW
+
IP_HDRINCL
```

을 추천합니다.

왜냐하면 검증하려는 것은

```text
IPv4 Identification
Fragment Offset
MF Flag
UDP Header
```

이기 때문입니다.

MAC Header는 필요 없습니다.

---

# 수신 시 헤더 확인

수신도 Raw Socket으로 가능합니다.

```c
int fd =
    socket(AF_INET,
           SOCK_RAW,
           IPPROTO_UDP);
```

그러면

```text
IP Header
UDP Header
Payload
```

가 그대로 올라옵니다.

즉

```c
recv()
```

후

```c
struct iphdr* ip =
    (struct iphdr*)buf;
```

로 확인 가능합니다.

---

# 중요한 주의점

Linux Kernel은 기본적으로

```text
IP Fragment
↓
Reassembly
↓
UDP
```

를 수행합니다.

따라서

```c
socket(AF_INET,
       SOCK_DGRAM,
       IPPROTO_UDP);
```

로 수신하면

이미 재조립된 데이터만 보게 됩니다.

Fragment 자체를 보고 싶으면

```c
SOCK_RAW
```

를 사용해야 합니다.

---

# Linux를 App_B 테스트 장비로 활용하는 방법

현재 목적에 가장 적합한 구조는

```text
Test App

↓
Raw Socket

↓
직접 Fragment 생성

↓
Linux NIC

==================

App_B

↓
Recv()

↓
Reassembly

↓
UDP Message 복구
```

입니다.

반대로

```text
App_B

↓
Fragment 생성

↓

Linux Raw Socket 수신
```

으로도 검증 가능합니다.

---

# 추가 팁

Linux에는 이미 Fragmentation 테스트용 툴도 있습니다.

대표적으로:

* `Scapy`
* `hping3`
* `packetdrill`

특히 Python의 Scapy 는 몇 줄만으로 Fragment 패킷을 생성할 수 있습니다.

예:

```python
send(fragment(IP(dst="1.1.1.1")/
              UDP()/
              ("A"*4000)))
```

이렇게 하면 Linux Kernel이 아니라 사용자 공간에서 RFC791 Fragment를 직접 생성하여 보낼 수 있습니다.

현재 상황이라면 **Raw Socket + IP_HDRINCL 방식으로 먼저 App_B의 Reassembly를 검증하고, 이후 Scapy를 이용해 순서 변경, Fragment 유실, 중복 Fragment 등의 예외 케이스를 테스트하는 것이 가장 효율적입니다.**
