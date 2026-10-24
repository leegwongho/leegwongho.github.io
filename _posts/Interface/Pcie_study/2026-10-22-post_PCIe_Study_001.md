---
title: "PCIe / QDMA Study - 001: PCIe란 무엇인가?"
date: 2026-10-22 14:00:00 +0900

categories:
  - Interface
  - PCIe Study

tags:
  - PCIe
  - Lane
  - Root Complex
  - Endpoint
  - FPGA

toc: true

comments: true
---

# Introduction

Project 01에서는 FPGA와 외부 장비 사이의 100G Ethernet 경로를 다뤘습니다. 이 데이터가 서버의 CPU와 메모리까지 전달되려면 **PCIe**를 거쳐야 합니다.

이 시리즈는 PCIe와 QDMA를 공부하면서 이해한 내용을 정리하는 **기술 학습 글**입니다. 개발 프로젝트 기록(Project 시리즈)과 구분해서, 개념과 구조를 중심으로 다룹니다.

첫 글에서는 PCIe가 무엇이고 어떤 구조로 연결되는지, 그리고 대역폭을 어떻게 계산하는지 정리합니다.

---

# What is PCIe?

**PCIe(PCI Express)** 는 CPU와 주변 장치(GPU, NIC, SSD, FPGA 등)를 연결하는 **고속 직렬 point-to-point 인터페이스**입니다.

기존 PCI는 여러 장치가 하나의 병렬 버스를 **공유**했지만, PCIe는 장치마다 **전용 직렬 링크**를 둡니다.

| | PCI | PCIe |
|---|-----|------|
| 구조 | 병렬 공유 버스 | 직렬 point-to-point |
| 대역폭 | 장치들이 나눠 씀 | 링크마다 전용 |
| 방향 | 반이중 | **전이중** (송신/수신 동시) |
| 확장 | 클럭을 올리기 어려움 | Lane 수와 세대로 확장 |

소프트웨어 관점에서는 PCI와 호환됩니다. 장치를 찾는 방법(Configuration Space)이나 주소 체계가 이어져서, 하드웨어는 바뀌어도 기존 드라이버 모델을 그대로 쓸 수 있습니다.

---

# Topology: Root Complex, Switch, Endpoint

PCIe는 **트리 구조**입니다.

```
              CPU / Host Memory
                    │
             ┌──────┴──────┐
             │ Root Complex │
             └──┬───────┬───┘
                │       │
          Endpoint     Switch
          (FPGA)     ┌───┴───┐
                  Endpoint  Endpoint
                  (NIC)     (SSD)
```

| 구성 요소 | 역할 |
|-----------|------|
| **Root Complex (RC)** | CPU·메모리와 PCIe 계층을 연결하는 최상위 노드. 트리의 뿌리 |
| **Switch** | 하나의 링크를 여러 개로 나누는 중간 노드 |
| **Endpoint (EP)** | 트리의 말단 장치. NIC, GPU, SSD, **FPGA** 등 |

FPGA에 PCIe IP를 올리면 FPGA는 보통 **Endpoint** 로 동작하고, 서버 쪽 메인보드의 Root Complex와 연결됩니다.

각 장치는 **Bus / Device / Function (BDF)** 번호로 식별됩니다. 리눅스에서 `lspci`로 보이는 `01:00.0` 같은 주소가 이것입니다.

---

# Lane과 Link

PCIe의 물리적 단위는 **Lane** 입니다.

- 1 Lane = 송신 차동쌍 1개 + 수신 차동쌍 1개 (총 4개 신호선)
- 송신과 수신이 분리되어 있어 **동시에** 양방향으로 전송할 수 있습니다.
- **Link** 는 Lane을 묶은 것입니다: **x1, x2, x4, x8, x16**

FPGA에서는 각 Lane이 **GT Transceiver**(003 글에서 본 고속 직렬 송수신기)에 연결됩니다. Ethernet의 QSFP28과 마찬가지로 직렬 Lane을 여러 개 묶어 대역폭을 키우는 구조입니다.

서버 슬롯의 물리적 크기(x16 슬롯 등)와 실제로 연결된 Lane 수는 다를 수 있습니다. 링크는 양쪽 장치가 지원하는 Lane 수와 속도 중 **낮은 쪽으로 협상**됩니다.

---

# Generation과 속도

PCIe는 세대마다 Lane당 전송 속도가 약 2배씩 늘어납니다.

| Gen | 전송 속도 (GT/s) | 인코딩 | Lane당 단방향 대역폭 (대략) |
|-----|-----------------|--------|---------------------------|
| Gen1 | 2.5 | 8b/10b | 250 MB/s |
| Gen2 | 5.0 | 8b/10b | 500 MB/s |
| Gen3 | 8.0 | 128b/130b | 약 985 MB/s |
| Gen4 | 16.0 | 128b/130b | 약 1.97 GB/s |
| Gen5 | 32.0 | 128b/130b | 약 3.94 GB/s |

> Gen6(64 GT/s)부터는 PAM4와 FLIT 방식을 사용해 인코딩 구조가 달라져서 이 표의 계산 방식을 그대로 적용하지 않습니다.

## 대역폭 계산

**GT/s** 는 초당 전송하는 *비트 심볼* 수이고, 인코딩 오버헤드를 빼야 실제 데이터 비트가 됩니다.

- Gen1/2의 8b/10b는 10비트로 8비트를 보내므로 효율이 **80%** 입니다.
- Gen3 이상의 128b/130b는 130비트로 128비트를 보내므로 효율이 약 **98.5%** 입니다.

Gen3 한 Lane의 계산입니다.

```
8 GT/s × (128/130) = 7.877 Gb/s ≈ 0.985 GB/s   (단방향)
```

여기에 Lane 수를 곱합니다.

| 구성 | 단방향 대역폭 (대략) |
|------|--------------------|
| Gen3 x4 | 약 3.9 GB/s (약 31.5 Gb/s) |
| Gen3 x8 | 약 7.9 GB/s (약 63 Gb/s) |
| Gen3 x16 | 약 15.8 GB/s (약 126 Gb/s) |
| Gen4 x8 | 약 15.8 GB/s (약 126 Gb/s) |

## 실제 처리량은 더 낮다

이 숫자는 **링크의 이론상 최대치**입니다. 실제 데이터 처리량은 다음 때문에 더 낮아집니다.

- 모든 패킷에 붙는 헤더, CRC, 프레이밍
- Data Link Layer의 ACK, Flow Control 패킷
- 한 패킷에 담을 수 있는 payload 크기의 제한(다음 글에서 다룹니다)

그래서 이론 대역폭의 대략 80~90% 정도를 현실적인 상한으로 보는 경우가 많고, 정확한 값은 payload 크기와 시스템 설정에 따라 달라집니다.

## 100G Ethernet과 비교하면

100Gb/s는 약 12.5 GB/s입니다. Gen3 x8(약 7.9 GB/s)로는 부족하고, **Gen3 x16 또는 Gen4 x8** 정도가 필요합니다. 헤더 오버헤드까지 감안하면 여유가 크지 않다는 점도 같이 봐야 합니다. 100G NIC가 x16 슬롯을 쓰는 이유가 이것입니다.

---

# PCIe의 세 계층

PCIe는 세 개의 계층으로 나뉘어 동작합니다.

```
  Device A                          Device B
┌────────────────┐             ┌────────────────┐
│ Transaction    │ ── TLP ───► │ Transaction    │
├────────────────┤             ├────────────────┤
│ Data Link      │ ── DLLP ──► │ Data Link      │
├────────────────┤             ├────────────────┤
│ Physical       │ ── 전기 신호 ─► │ Physical       │
└────────────────┘             └────────────────┘
```

| 계층 | 주고받는 단위 | 하는 일 |
|------|--------------|---------|
| **Transaction Layer** | **TLP** (Transaction Layer Packet) | 읽기/쓰기 요청과 응답 같은 **실제 요청**을 만들고 해석 |
| **Data Link Layer** | **DLLP** (Data Link Layer Packet) | 패킷의 **신뢰성** 보장: 순서 번호, CRC, ACK/NAK, 재전송, Flow Control 갱신 |
| **Physical Layer** | 심볼, 비트 | 인코딩, 직렬화, Lane 정렬, **링크 트레이닝** |

각 계층은 위 계층의 데이터를 감싸서 아래로 내려보냅니다. Ethernet 프레임에 헤더가 쌓이는 것(005 글)과 비슷한 구조입니다.

한 가지만 기억하면 됩니다. FPGA 설계자가 PCIe IP를 쓸 때 사용자 로직이 직접 다루는 것은 **TLP 수준(Transaction Layer)** 이고, Data Link와 Physical 계층은 IP가 알아서 처리합니다. 따라서 다음 글에서는 **TLP가 어떤 정보를 담고 있는지**를 집중적으로 다룹니다.

---

# Link Training

전원이 켜지면 두 장치는 **링크 트레이닝**을 거쳐 연결 상태를 협상합니다. 물리 계층의 상태 머신(LTSSM)이 담당합니다.

1. **Detect**: 반대편 장치가 연결되어 있는지 확인
2. **Polling**: 양쪽이 서로의 존재와 속도를 확인
3. **Configuration**: 실제로 사용할 Lane 수와 정렬 방식을 결정
4. **L0**: 정상 동작 상태

협상 결과는 양쪽이 모두 지원하는 최대 Lane 수와 속도입니다. 예를 들어 x16 장치를 x8 슬롯에 꽂으면 x8로 올라옵니다.

FPGA에서 링크가 올라왔는지는 PCIe IP의 **link up** 상태 신호로 확인하고, 호스트에서는 `lspci`에서 장치가 보이는지와 협상된 Lane 수·속도로 확인합니다. Ethernet CMAC에서 `stat_rx_aligned`로 링크를 판단한 것(004 글)과 같은 역할입니다.

---

# What I Learned

- PCIe는 **직렬 point-to-point 링크**이고, Lane(x1~x16)과 세대(Gen1~5)로 대역폭이 정해진다.
- 구조는 Root Complex를 뿌리로 하는 **트리**이고, FPGA는 보통 **Endpoint**다.
- 인코딩 효율(8b/10b vs 128b/130b)을 반영해서 대역폭을 계산하고, 실제 처리량은 헤더·ACK 오버헤드로 더 낮아진다.
- 100G Ethernet을 호스트까지 올리려면 Gen3 x16 또는 Gen4 x8 정도가 필요하다.
- PCIe는 Transaction / Data Link / Physical 세 계층이며, 사용자 로직은 **TLP**를 다룬다.

---

# Next

다음 글에서는 Transaction Layer에서 **어떤 정보가 어떤 형식으로 오가는지**, 즉 TLP의 종류와 구조를 살펴봅니다.

> **PCIe / QDMA Study - 002: PCIe 패킷(TLP)과 Transaction**

---

# References

- PCI-SIG – PCI Express Base Specification
- AMD PG213 – UltraScale+ Devices Integrated Block for PCI Express
