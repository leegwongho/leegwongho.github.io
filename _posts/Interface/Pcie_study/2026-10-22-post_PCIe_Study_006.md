---
title: "PCIe / QDMA Study - 006: 멀티 큐와 RSS"
date: 2026-10-22 15:20:00 +0900

categories:
  - Interface
  - PCIe Study

tags:
  - RSS
  - Toeplitz
  - Multi Queue
  - QDMA
  - Ethernet

toc: true

comments: true
---

# Introduction

004에서 QDMA가 **큐 단위로 동작**한다는 것을 봤습니다. 큐가 많다는 것은 장점이지만, 패킷이 들어올 때 **어느 큐로 보낼지**를 누군가 정해야 합니다.

이번 글은 그 방법 중 가장 널리 쓰이는 **RSS(Receive Side Scaling)** 를 이론 중심으로 정리합니다.

> RSS를 직접 구현해 본 경험이 있지만, 이 글은 일반적인 개념과 공개된 규격을 중심으로 정리했습니다.

---

# 왜 멀티 큐인가

최소 크기 프레임(64 B)은 Preamble/SFD(8 B)와 IFG(12 B)를 합쳐 회선상 **84 B**를 차지합니다. 이를 기준으로 계산하면 링크가 처리해야 하는 초당 최대 패킷 수는 다음과 같습니다.

```
84 B × 8 = 672 bit

10G  : 10 × 10^9  / 672 ≈  14.88 Mpps
100G : 100 × 10^9 / 672 ≈ 148.8  Mpps
```

100G 링크에서 작은 패킷이 계속 들어오면 초당 1억 개가 넘는 패킷을 처리해야 합니다. 수신 큐가 **하나**뿐이라면 다음과 같은 한계가 생깁니다.

- 모든 패킷을 **하나의 CPU 코어**가 처리해야 해서 그 코어가 병목이 된다.
- 여러 코어가 같은 큐를 공유하면 **락(lock) 경쟁**이 생긴다.
- 코어 간에 데이터가 오가면서 **캐시 효율**이 떨어진다.

해결책은 **큐를 여러 개** 두고, 큐마다 **전담 코어**를 배정하는 것입니다. 003에서 본 QDMA의 "큐별 독립 설정"이 이 구조를 위한 것입니다.

```
들어온 패킷 ──► [ 분배 ] ──► Queue 0 ──► Core 0
                         ├─► Queue 1 ──► Core 1
                         ├─► Queue 2 ──► Core 2
                         └─► Queue 3 ──► Core 3
```

---

# 분배 기준: Flow

분배 방법이 아무렇게나 나누면 안 됩니다. 같은 연결(Flow)의 패킷이 서로 다른 큐로 흩어지면 **순서가 뒤바뀔 수 있기** 때문입니다. TCP는 순서가 바뀌면 성능이 떨어집니다.

그래서 분배 기준은 다음 원칙을 만족해야 합니다.

- **같은 Flow의 패킷은 항상 같은 큐**로 간다.
- **서로 다른 Flow는 큐들에 골고루** 퍼진다.

Flow는 보통 **5-tuple**로 식별합니다.

| 요소 | 위치 |
|------|------|
| Source IP | IP 헤더 |
| Destination IP | IP 헤더 |
| Protocol (TCP/UDP) | IP 헤더 |
| Source Port | TCP/UDP 헤더 |
| Destination Port | TCP/UDP 헤더 |

이 값들을 **해시**에 넣으면, 같은 Flow는 항상 같은 해시값이 나오고 다른 Flow는 고르게 퍼진 해시값이 나옵니다. 이것이 RSS의 핵심 아이디어입니다.

---

# RSS 동작 구조

```
Ethernet Frame
      │
      ▼
 ① 헤더 파싱      (Ethernet → VLAN? → IPv4/IPv6 → TCP/UDP)
      │
      ▼
 ② 해시 입력 추출   (IP 주소, 포트)
      │
      ▼
 ③ Toeplitz 해시   (입력 + 40바이트 키 → 32비트 해시)
      │
      ▼
 ④ 해시의 하위 비트로 Indirection Table 조회
      │
      ▼
 ⑤ 큐 번호 결정 ──► 해당 큐로 DMA
```

각 단계를 하나씩 보겠습니다.

## ① 헤더 파싱

해시에 필요한 필드의 위치를 알려면 프레임의 계층 구조를 따라가야 합니다. Project 01의 Ethernet Frame 글(005)에서 본 **EtherType**이 여기서 쓰입니다.

- EtherType `0x0800`이면 IPv4, `0x86DD`이면 IPv6입니다.
- VLAN 태그가 있으면 Ethernet 헤더가 4 Byte 늘어납니다.
- IP 헤더의 Protocol(또는 Next Header) 필드로 TCP/UDP 여부를 확인합니다.

## ② 해시 입력 추출

해시에 넣는 입력은 **패킷 순서대로** 이어 붙인 바이트열입니다. Microsoft의 RSS 규격 기준으로 네 가지 경우가 있습니다.

| 경우 | 입력 (순서대로 이어 붙임) | 길이 |
|------|-------------------------|------|
| IPv4 + TCP | Source IP, Dest IP, Source Port, Dest Port | 12 B |
| IPv4 only | Source IP, Dest IP | 8 B |
| IPv6 + TCP | Source IP, Dest IP, Source Port, Dest Port | 36 B |
| IPv6 only | Source IP, Dest IP | 32 B |

주소와 포트는 **패킷에 실린 그대로(네트워크 바이트 순서)** 이어 붙이고, 첫 바이트의 최상위 비트가 입력의 가장 왼쪽 비트입니다.

앞에서 Flow를 5-tuple로 식별한다고 했지만, 이 규격에서 **Protocol은 해시 입력에 들어가지 않습니다.** Protocol은 위 표의 네 가지 중 **어떤 입력 조합을 쓸지 고르는 기준**으로만 쓰입니다.

VLAN 태그와 IP 옵션이 없는 IPv4 프레임에서 각 필드의 위치는 다음과 같습니다. (Project 01의 005 글에서 본 Ethernet 헤더 14 B와 IPv4 헤더 20 B 기준)

| 필드 | 프레임 시작부터의 Byte 위치 |
|------|----------------------------|
| Source IP | 26 ~ 29 |
| Destination IP | 30 ~ 33 |
| Source Port | 34 ~ 35 |
| Destination Port | 36 ~ 37 |

VLAN 태그가 있으면 위치가 4 B 밀리고, IP 옵션이 있으면 IP 헤더 길이(IHL)에 따라 L4 헤더 위치가 달라집니다. 그래서 하드웨어는 EtherType과 IHL을 보고 위치를 **계산**해야 합니다.

## ③ Toeplitz 해시

RSS의 해시 함수는 **Toeplitz 해시**입니다. 40바이트(320비트)의 **비밀 키**를 사용하고, 결과는 32비트입니다. 알고리즘은 단순합니다.

```
result = 0
입력의 비트를 왼쪽(최상위)부터 하나씩 보면서:
    비트가 1이면  result ^= (키의 왼쪽 32비트)
    키를 왼쪽으로 1비트 shift
return result
```

입력의 **1인 비트마다, 그 위치에 해당하는 키의 32비트 구간을 XOR**하는 것입니다. 입력이 n비트이면 키는 n + 31비트가 필요합니다. 가장 긴 입력인 IPv6 + TCP(36 B = 288비트)는 288 + 31 = 319비트로, 40바이트 키(320비트)에 맞춰져 있습니다.

### 검증 예제

규격에 공개된 검증 값으로 구현이 맞는지 확인할 수 있습니다. 아래 키를 사용합니다.

```
6d 5a 56 da 25 5b 0e c2 41 67 25 3d 43 a3 8f b0
d0 ca 2b cb ae 7b 30 b4 77 cb 2d a3 80 30 f2 0c
6a 42 b7 3b be ac 01 fa
```

| Source | Destination | IPv4 only | IPv4 + TCP |
|--------|-------------|-----------|-----------|
| 66.9.149.187 : 2794 | 161.142.100.80 : 1766 | `0x323e8fc2` | `0x51ccc178` |
| 199.92.111.2 : 14230 | 65.69.140.83 : 4739 | `0xd718262a` | `0xc626b0ea` |
| 24.19.198.95 : 12898 | 12.22.207.184 : 38024 | `0xd2d0a5de` | `0x5c2b394a` |

간단한 Python 구현입니다.

```python
import socket, struct

KEY = bytes.fromhex(
    "6d5a56da255b0ec24167253d43a38fb0d0ca2bcbae7b30b477cb2da38030f20c"
    "6a42b73bbeac01fa")

def toeplitz(data: bytes, key: bytes = KEY) -> int:
    kbits, klen = int.from_bytes(key, "big"), len(key) * 8
    assert klen >= len(data) * 8 + 31
    h = 0
    for i in range(len(data) * 8):
        if (data[i // 8] >> (7 - i % 8)) & 1:
            h ^= (kbits >> (klen - 32 - i)) & 0xFFFFFFFF
    return h

src = socket.inet_aton("66.9.149.187")
dst = socket.inet_aton("161.142.100.80")
data = src + dst + struct.pack(">HH", 2794, 1766)   # src port, dst port
print(hex(toeplitz(data)))                          # 0x51ccc178
print(hex(toeplitz(src + dst)))                     # 0x323e8fc2
```

> 위 코드는 표의 값 5건을 모두 재현하는 것을 확인했습니다. RTL을 검증할 때 같은 입력으로 **소프트웨어 결과와 하드웨어 결과를 비교**하는 기준으로 쓸 수 있습니다.

## ④ Indirection Table

32비트 해시를 큐 번호로 바꾸는 단계입니다. 해시의 **하위 몇 비트**로 작은 테이블을 조회하고, 테이블의 값이 큐 번호입니다.

```
해시 = 0x51ccc178, 테이블 크기 128 (하위 7비트 사용)
인덱스 = 0x51ccc178 & 0x7F = 120
큐 번호 = Table[120]
```

테이블을 한 번 거치는 이유는 다음과 같습니다.

- 큐 수와 해시 공간을 분리할 수 있다. 해시는 32비트인데 큐는 몇 개뿐이다.
- 테이블 내용만 바꾸면 **해시를 다시 계산하지 않고** 큐 배분 비율을 조정할 수 있다. 예를 들어 어떤 코어를 빼고 싶을 때 테이블 항목만 바꿉니다.

테이블이 4큐를 균등하게 쓰는 경우 항목을 0, 1, 2, 3, 0, 1, 2, 3, ... 으로 채웁니다.

---

# 구현할 때 흔한 실수

해시 알고리즘 자체는 단순해서 틀리기 쉬운 곳은 **입력을 만드는 단계**입니다. 같은 알고리즘이라도 입력이 달라지면 결과가 완전히 달라집니다. 위의 첫 번째 예제 입력으로 직접 확인해 보면 다음과 같습니다.

| 입력 구성 | 결과 |
|-----------|------|
| 올바른 순서 (Src IP, Dst IP, Src Port, Dst Port) | `0x51ccc178` |
| Source와 Destination을 **바꿔서** 입력 | `0xfde799b2` |
| IP 주소의 **바이트 순서를 뒤집어서** 입력 | `0x91ab7659` |

- **Source와 Destination의 순서**를 바꾸는 경우
- **IP 바이트 순서**(네트워크 바이트 순서 vs 호스트/리틀엔디안)를 잘못 이어 붙이는 경우
- 하드웨어에서 데이터 폭(AXI4-Stream 512비트)의 **바이트 레인 위치**를 잘못 계산해 필드를 엉뚱한 위치에서 읽는 경우

이런 실수는 해시 로직이 아니라 **필드 추출 단계**에서 생깁니다. 그래서 위의 검증 값으로 소프트웨어와 하드웨어를 비교하는 것이 가장 확실한 확인 방법입니다.

---

# 응용과 한계

## 대칭 해시 (Symmetric RSS)

양방향 트래픽(요청과 응답)을 **같은 큐**로 보내고 싶을 때가 있습니다. 보통의 해시는 Source와 Destination을 바꾸면 결과가 달라집니다. 하지만 키를 **`6d 5a` 를 반복한 패턴**으로 두면 Source와 Destination을 바꿔도 같은 해시가 나오게 만들 수 있다고 알려져 있습니다. 위의 예제 입력으로 확인해 보면 양방향 모두 `0x9fcc9fcc`가 나옵니다.

## 한계

- **큰 Flow 하나**는 해시가 하나이므로 하나의 큐로만 갑니다. Flow 수가 적으면 큐가 고르게 쓰이지 않습니다.
- IP 단편화된 패킷의 뒷부분에는 포트 정보가 없어서 **IP 주소만으로** 해시해야 합니다.
- TCP/UDP가 아닌 프로토콜은 포트 입력이 없어서 구현에 따라 **기본 큐**로 보내기도 합니다.

---

# QDMA와의 관계

RSS는 **어느 큐로 보낼지**만 정하고, 실제 전송은 QDMA가 합니다.

```
CMAC → [ RSS: 헤더 파싱 + 해시 + 테이블 → 큐 번호 ] → QDMA C2H (해당 큐의 Ring) → 호스트
```

- QDMA는 큐마다 **독립된 링과 PIDX/CIDX**를 가지므로(004), 큐가 달라도 서로 영향을 주지 않습니다.
- 어떤 패킷을 어느 큐 번호로 보낼지 알려 주는 것은 **사용자 로직의 역할**입니다. 구체적인 신호는 PG302의 C2H Stream 인터페이스 절에서 확인해야 합니다.
- 큐와 CPU 코어를 짝짓는 것은 **드라이버**의 몫입니다. 큐별 인터럽트나 폴링 스레드를 어느 코어에 묶을지 정합니다.

## 호스트에서 확인하기

Linux에서는 `ethtool`로 NIC의 RSS 설정을 확인합니다. 드라이버가 지원하는 경우에 한합니다.

| 명령 | 용도 |
|------|------|
| `ethtool -l <if>` | 큐(채널) 개수 확인 |
| `ethtool -x <if>` | Indirection Table과 해시 키 확인 |
| `ethtool -X <if> ...` | Indirection Table, 키 변경 |

## TX 방향은 다르다

RSS는 **수신(RX)** 방향의 분배입니다. 송신(TX)은 소프트웨어가 어느 큐로 보낼지 **직접 고릅니다.** 보통 코어마다 자기 TX 큐를 가지고 H2C 링에 Descriptor를 올리며, 하드웨어는 여러 TX 큐를 중재해서 한 포트로 내보냅니다.

---

# What I Learned

- 멀티 큐는 **CPU 코어마다 전담 큐**를 두어 병목과 락 경쟁을 없애기 위한 것이다.
- 분배의 원칙은 **같은 Flow는 같은 큐, 다른 Flow는 고르게**이며, Flow는 5-tuple로 식별한다.
- RSS는 **헤더 파싱 → 입력 추출 → Toeplitz 해시 → Indirection Table → 큐 번호** 순서로 동작한다.
- Toeplitz 해시는 입력의 1인 비트마다 키의 32비트 구간을 XOR하는 단순한 알고리즘이다.
- 구현에서 틀리기 쉬운 곳은 해시가 아니라 **입력 구성(순서, 바이트 순서, 필드 위치)** 이며, 공개된 검증 값으로 확인할 수 있다.

---

# Next

PCIe와 QDMA의 이론 부분은 여기까지 정리했습니다. 다음 글에서는 시리즈 전체를 돌아보고, 링크, 큐, 완료 처리 단계별로 **문제가 생겼을 때 어디를 볼지**를 체크리스트로 묶습니다.

> **PCIe / QDMA Study - 007: 시리즈 정리와 디버깅 체크리스트**

---

# References

- Microsoft – RSS Hashing Functions / Verifying the RSS Hash Calculation (Windows Hardware Drivers, Network)
- AMD PG302 – QDMA Subsystem for PCI Express
