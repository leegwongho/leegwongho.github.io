---
title: "PCIe / QDMA Study - 002: PCIe 패킷(TLP)과 Transaction"
date: 2026-10-22 14:10:00 +0900

categories:
  - Interface
  - PCIe Study

tags:
  - PCIe
  - TLP
  - BAR
  - Memory Read
  - Memory Write

toc: true

comments: true
---

# Introduction

001에서 PCIe가 직렬 링크이고 세 개의 계층으로 나뉜다는 것을 봤습니다. 이번 글에서는 사용자 로직이 실제로 다루는 **Transaction Layer**에서 **어떤 프로토콜로 어떤 정보를 주고받는지**를 정리합니다.

---

# Transaction이란

PCIe에서 장치들은 **요청(Request)** 과 **응답(Completion)** 으로 통신합니다. 요청을 보내는 쪽을 **Requester**, 처리하는 쪽을 **Completer** 라고 합니다.

이 요청과 응답은 모두 **TLP(Transaction Layer Packet)** 라는 패킷에 담겨 전달됩니다.

---

# TLP 구조

```
| Header (3 or 4 DW) | Data Payload (선택) | ECRC (선택) |
```

> **DW(Double Word)** 는 4 Byte입니다. PCIe에서는 길이를 대부분 DW 단위로 표현합니다.

| 필드 | 설명 |
|------|------|
| **Header** | 패킷의 종류, 길이, 주소, 요청자 ID 등. 3DW(12 B) 또는 4DW(16 B) |
| **Data Payload** | 실제 데이터. 쓰기 요청과 읽기 응답에만 있음 |
| **ECRC** | End-to-end CRC (선택 기능) |

Physical과 Data Link 계층이 이 TLP를 앞뒤로 감쌉니다. 링크 위에서는 앞뒤에 프레이밍 심볼, **시퀀스 번호, LCRC** 가 추가되어 전송됩니다. 하지만 사용자 로직이 보는 것은 TLP입니다.

---

# TLP의 종류

TLP의 종류는 헤더의 **Fmt**(형식)와 **Type** 필드로 구분합니다. 주로 쓰이는 것은 다음과 같습니다.

| TLP | 이름 | 데이터 | 응답 필요 |
|-----|------|--------|-----------|
| **MRd** | Memory Read Request | 요청에는 없음 | **Completion 필요** |
| **MWr** | Memory Write Request | payload 있음 | **필요 없음** (Posted) |
| **CfgRd / CfgWr** | Configuration Read / Write | 있음 | 필요 |
| **IORd / IOWr** | I/O Read / Write (레거시) | 있음 | 필요 |
| **Cpl / CplD** | Completion (데이터 없음 / 있음) | CplD만 있음 | 응답 자체 |
| **Msg** | Message (인터럽트, 에러 등) | 선택 | 일반적으로 없음 |

## Posted와 Non-Posted

- **Posted**: 응답 없이 보내고 끝냅니다. **Memory Write**가 대표적입니다.
- **Non-Posted**: 반드시 Completion이 돌아와야 합니다. **Memory Read, Configuration, I/O 요청**이 해당합니다.

쓰기는 보내고 끝나므로 빠르지만 "수행됐는지"를 확인할 수 없고, 읽기는 응답을 기다려야 하므로 **지연(latency)이 큽니다.** 고속 데이터 전송에서 읽기 지연이 성능에 영향을 주는 이유가 여기에 있습니다.

---

# 헤더에 들어 있는 정보

Memory Request 헤더(3DW, 32비트 주소 기준)는 대략 다음 정보를 담습니다.

| 필드 | 의미 |
|------|------|
| **Fmt / Type** | TLP의 종류, 3DW/4DW 여부, payload 유무 |
| **TC** | Traffic Class. 우선순위 구분 |
| **Length** | 데이터 길이(DW 단위, 10비트). 0이면 1024 DW |
| **Requester ID** | 요청한 장치의 Bus/Device/Function |
| **Tag** | 요청마다 붙이는 번호. 응답과 짝을 맞추는 데 사용 |
| **First / Last DW BE** | 첫/마지막 DW 안에서 유효한 Byte 선택(Byte Enable) |
| **Address** | 접근할 메모리 주소. 64비트 주소면 헤더가 4DW |

주소가 4GB 이하면 3DW 헤더, 이상이면 4DW 헤더를 씁니다. 서버의 호스트 메모리 주소는 보통 4GB를 넘기 때문에 **DMA 요청은 4DW 헤더**인 경우가 많습니다.

## Completion 헤더

응답(Cpl/CplD)에는 다음이 들어 있습니다.

| 필드 | 의미 |
|------|------|
| **Completer ID** | 응답한 장치 |
| **Requester ID + Tag** | 어느 요청에 대한 응답인지 (요청과 동일한 값) |
| **Status** | 성공(SC), 지원하지 않는 요청(UR), 중단(CA) 등 |
| **Byte Count / Lower Address** | 남은 데이터 양과 현재 응답의 시작 위치 |

**Tag** 덕분에 요청자는 여러 읽기 요청을 한꺼번에 보내 놓고, 응답이 순서와 상관없이 도착해도 짝을 맞출 수 있습니다. 한 번에 보낼 수 있는 미해결 요청의 수가 읽기 처리량을 좌우합니다.

---

# 읽기와 쓰기는 이렇게 흐른다

## Memory Write (Posted)

```
Requester ── MWr(주소, 데이터) ──► Completer
            (응답 없음)
```

## Memory Read (Non-Posted)

```
Requester ── MRd(주소, 길이, Tag) ──► Completer
Requester ◄── CplD(Tag, 데이터) ──── Completer
```

## Payload 크기 제한

한 TLP에 담을 수 있는 데이터 양은 제한이 있습니다.

- **MPS (Max Payload Size)**: 한 TLP의 최대 payload. 128, 256, 512, 1024 B 등. 링크 양단이 지원하는 값 중 협상한 값을 씁니다.
- **MRRS (Max Read Request Size)**: 한 번의 읽기 요청으로 요청할 수 있는 최대 크기.

예를 들어 MRRS가 크고 MPS가 256 B일 때, 1024 B를 읽으면 **Completion이 여러 개(예: 256 B × 4)로 쪼개져서** 돌아옵니다. 쓰기도 MPS보다 크면 여러 MWr로 나뉩니다.

작은 payload는 헤더 오버헤드 비율이 커집니다. 예를 들어 4DW 헤더(16 B)에 payload 64 B면 헤더만 20% 수준이라, 001에서 말한 "실제 처리량이 이론보다 낮다"가 여기서 나타납니다.

---

# 호스트는 어떻게 FPGA에 접근하는가

## Configuration Space

모든 PCIe 장치는 **Configuration Space**를 가집니다. 장치의 신분증과 설정 창구입니다.

- Vendor ID, Device ID, Class code
- Command / Status 레지스터
- **BAR (Base Address Register)**
- 이후 Capability 목록 (MSI-X, PCIe Capability 등)

부팅할 때 BIOS와 OS는 **Configuration 읽기/쓰기(CfgRd/CfgWr)** 로 모든 장치를 찾아내고(**Enumeration**), 각 장치의 자원을 할당합니다.

## BAR와 MMIO

**BAR**는 "이 장치가 CPU의 어느 주소 구간을 차지할지"를 정하는 레지스터입니다.

1. 장치가 BAR에 **필요한 주소 크기**를 알려줍니다.
2. 호스트가 비어 있는 주소 구간을 배정해서 BAR에 **기록**합니다.
3. 이후 CPU가 그 주소 구간을 읽고 쓰면 Root Complex가 이를 **Memory Read/Write TLP**로 바꿔서 장치로 보냅니다.

이것이 **MMIO(Memory-Mapped I/O)** 입니다. 드라이버가 FPGA의 레지스터를 읽고 쓰는 것은 결국 **BAR 주소로 보내는 MRd/MWr TLP**입니다.

```
CPU: *(volatile u32*)(bar0 + 0x10) = 1;
  → Root Complex → MWr TLP(주소 = BAR0 + 0x10, 데이터 = 1) → FPGA
```

FPGA 쪽에서는 BAR 구간으로 들어온 TLP를 해석해서 제어 레지스터로 연결합니다.

---

# 방향이 반대인 경우: 장치가 호스트 메모리에 접근

Endpoint도 **Requester**가 될 수 있습니다. 이를 **Bus Master** 기능이라고 합니다. FPGA가 직접 호스트 메모리 주소로 MWr 또는 MRd TLP를 보내면 CPU의 개입 없이 데이터를 옮길 수 있습니다. 이것이 **DMA(Direct Memory Access)** 의 기본 원리이고, 다음 글들의 주제입니다.

| 방향 | 누가 Requester | 용도 |
|------|---------------|------|
| 호스트 → 장치 | CPU (Root Complex) | 레지스터 제어 (MMIO) |
| 장치 → 호스트 | Endpoint (Bus Master) | 데이터 전송 (DMA), 인터럽트 |

인터럽트도 마찬가지로, **MSI/MSI-X**는 장치가 정해진 주소로 **Memory Write TLP**를 보내는 방식입니다.

---

# Flow Control과 신뢰성

## Flow Control (Credit)

수신 측 버퍼가 넘치지 않도록 PCIe는 **크레딧 기반 흐름 제어**를 합니다.

- 수신 측은 자신의 버퍼 공간을 크레딧으로 상대에게 알려줍니다.
- 송신 측은 **크레딧이 있을 때만** TLP를 보냅니다.
- 크레딧은 Posted / Non-Posted / Completion, 그리고 Header / Data로 나뉘어 관리됩니다.

크레딧이 소진되면 송신이 멈춥니다. 이 흐름 제어는 Data Link Layer가 주고받는 **DLLP(Update FC)** 로 갱신됩니다.

## ACK/NAK

각 TLP에는 **시퀀스 번호와 LCRC**가 붙습니다. 수신 측이 오류 없이 받으면 **ACK DLLP**를, 오류가 있으면 **NAK**를 보내고, 송신 측은 **재전송 버퍼**의 TLP를 다시 보냅니다. 그래서 사용자 로직은 링크 오류를 신경 쓰지 않아도 됩니다.

---

# 한눈에 정리

| 질문 | 답 |
|------|----|
| 무엇을 주고받나 | **TLP** (요청과 응답) |
| 어떤 요청이 있나 | Memory Read/Write, Configuration, I/O, Message |
| 응답이 필요한 것은 | Memory Read, Configuration, I/O (Non-Posted) |
| 응답이 필요 없는 것은 | Memory Write (Posted), 대부분의 Message |
| 헤더에 무엇이 있나 | 종류, 길이, 주소, Requester ID, Tag, Byte Enable |
| 호스트가 FPGA 레지스터에 접근하는 방법 | BAR 주소로 MRd/MWr TLP (MMIO) |
| FPGA가 호스트 메모리에 접근하는 방법 | Bus Master로 호스트 주소에 MRd/MWr TLP (DMA) |
| 신뢰성과 흐름 제어 | Data Link Layer의 ACK/NAK, 크레딧(DLLP) |

---

# What I Learned

- PCIe의 모든 통신은 **TLP**(요청과 응답)로 이루어지고, 종류는 Fmt/Type 필드로 구분한다.
- **Memory Write는 Posted**(응답 없음), **Memory Read는 Non-Posted**(Completion 필요)라서 읽기가 지연에 민감하다.
- **Tag** 로 요청과 응답을 짝지으며, 큰 읽기는 MPS 단위의 여러 Completion으로 쪼개져 돌아온다.
- 호스트가 FPGA 레지스터를 읽고 쓰는 것은 **BAR 주소로 가는 MMIO TLP**이다.
- 반대 방향에서 FPGA가 **Bus Master**로 호스트 메모리에 TLP를 보내는 것이 DMA이다.
- 크레딧 흐름 제어와 ACK/NAK는 하위 계층이 처리해 준다.

---

# Next

TLP로 호스트 메모리에 직접 접근할 수 있다는 것까지 확인했습니다. 다음 글에서는 이 원리를 이용해 **CPU의 개입을 줄이고 데이터를 옮기는 DMA**, 그리고 QDMA가 필요한 이유를 다룹니다.

> **PCIe / QDMA Study - 003: DMA와 QDMA의 역할**

---

# References

- PCI-SIG – PCI Express Base Specification
- AMD PG213 – UltraScale+ Devices Integrated Block for PCI Express
- AMD PG302 – QDMA Subsystem for PCI Express
