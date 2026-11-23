---
title: "OpenNIC Study - 002: Shell의 데이터 경로와 클럭 도메인"
date: 2026-10-24 19:10:00 +0900
last_modified_at: 2026-10-27 11:00:00 +0900

categories:
  - Networking
  - OpenNIC Study

tags:
  - OpenNIC
  - Clock Domain
  - AXI-Stream
  - Packet Adapter
  - CMAC

toc: true

comments: true
---

# Introduction

001에서 OpenNIC Shell이 QDMA와 CMAC을 묶은 구조라는 것을 봤습니다. 이번 글은 Shell **안에서 데이터가 어떤 블록을 거쳐 흐르는지**, 그리고 왜 클럭이 여러 개로 나뉘는지를 정리합니다.

> 이 글은 OpenNIC Shell의 공개 README(2026년 10월 확인한 `main` 브랜치)를 기준으로 정리했습니다. README 밖의 근거(QDMA, CMAC 사용자 가이드)를 쓴 부분은 따로 표시했습니다.

---

# 주요 블록

README가 설명하는 Shell의 블록은 다음과 같습니다.

| 블록 | 설명 |
|------|------|
| **QDMA subsystem** | AMD QDMA IP와, QDMA 인터페이스를 **250MHz** 사용자 로직 박스에 연결하는 RTL |
| **CMAC subsystem** | AMD CMAC IP와 래퍼 로직. **322MHz**로 동작하며 CMAC 포트 1개 또는 2개 |
| **Packet adapter** | **250MHz와 322MHz의 AXI-Stream 인터페이스 사이를 변환**하는 블록 |
| **System configuration** | 리셋과 각 구성 요소의 레지스터 주소 할당 |
| **User logic box** | 사용자가 로직을 넣는 자리. **250MHz 박스**와 **322MHz 박스** 두 개 |

사용자 로직 박스에는 기본 플러그인으로 **포트 대 포트 연결(p2p)** 이 들어 있어서, 아무 로직도 넣지 않아도 NIC로 동작합니다.

---

# 데이터 경로

README의 블록 다이어그램을 기준으로 하면 데이터는 다음 순서로 흐릅니다.

```
 [250MHz 영역]                                  [322MHz 영역]
 QDMA ⇄ 250MHz 박스 ⇄ Packet adapter ⇄ 322MHz 박스 ⇄ CMAC
   │                         │                         │
   └──────────── 125MHz AXI4-Lite (System configuration) ┘
```

- TX(호스트 → 네트워크): QDMA → 250MHz 박스 → Packet adapter → 322MHz 박스 → CMAC
- RX(네트워크 → 호스트): 위의 역순

각 블록은 System configuration에서 125MHz AXI4-Lite로 레지스터에 접근합니다.

---

# 클럭이 세 개인 이유

README에 나오는 클럭은 세 개입니다.

| 클럭 | 쓰이는 곳 |
|------|-----------|
| **250MHz** | QDMA와의 경로, 250MHz 사용자 박스 |
| **322MHz** | CMAC subsystem, 322MHz 사용자 박스 |
| **125MHz** | AXI4-Lite 레지스터 인터페이스, 동기 리셋 |

## 왜 나뉘나

이유는 PCIe / QDMA Study 005와 Project 01 007에서 이미 본 내용과 이어집니다.

- **CMAC**은 100G 데이터를 처리하기 위해 약 **322MHz**의 클럭을 요구합니다. (Project 01에서 본 `txusrclk2`)
- **QDMA**의 사용자 클럭은 PCIe 링크 설정에 따라 정해지고, 이 Shell에서는 **250MHz**입니다. (클럭이 링크 설정에 따라 정해진다는 부분은 README가 아니라 PG302 기준)
- 두 값이 다르므로 **CMAC과 QDMA 사이에서는 클럭 변환(CDC)이 반드시 필요**합니다.

Project 01의 CMAC 내부에서는 TX와 RX가 같은 클럭이라 CDC가 필요 없었습니다. 하지만 **서로 다른 IP 사이**에서는 클럭이 다르다는 것을 Shell 구조가 보여 줍니다.

## 125MHz와 250MHz는 다르다

같은 "다른 클럭"이라도 **125MHz와 250MHz는 위상이 정렬(phase-aligned)** 되어 있습니다. 그래서 README는 두 영역 사이에서는 CDC 없이 신호를 샘플링할 수 있다고 설명합니다.

다만 주파수가 2배 차이 나므로 주의가 필요합니다.

- 250MHz 쪽에서 125MHz 신호를 읽으면 같은 값을 **두 번 샘플링**할 수 있습니다.
- 125MHz 쪽에서 250MHz의 한 사이클짜리 펄스를 읽으면 **놓칠** 수 있습니다.

| 클럭 쌍 | 관계 | 필요한 처리 |
|---------|------|-------------|
| 250MHz ⇄ 322MHz | 비동기 | CDC 필요 (Packet adapter) |
| 125MHz ⇄ 250MHz | 위상 정렬 | CDC 불필요, 이중 샘플링/누락만 주의 |

---

# Packet Adapter

두 클럭 영역 사이에서 **Packet adapter**가 하는 일을 README는 이렇게 설명합니다.

- 250MHz와 322MHz의 AXI-Stream 인터페이스를 **서로 변환**한다.
- TX와 RX 모두에서 **패킷 전체를 버퍼링한 뒤 내보내는 FIFO(packet-mode FIFO)** 역할을 한다.
- CMAC 인터페이스에는 없는 **RX 역압(back-pressure)을 복원**한다.

세 번째가 특히 중요합니다. Project 01 007에서 본 것처럼 **CMAC의 RX 인터페이스에는 `tready`(역압)가 없습니다.** Shell README도 322MHz 인터페이스에서 CMAC RX 쪽으로 들어오는 패킷에는 `tready`가 없고, 마스터는 `tready`가 항상 1이라고 가정한다고 명시합니다. 그래서 RX 데이터를 받는 쪽이 느려지면 그 앞에서 받아 줄 버퍼가 필요했고, 이 Shell에서는 그 역할을 Packet adapter가 합니다.

| 문제 | Shell에서의 해결 |
|------|------------------|
| 두 클럭이 다름 | Packet adapter가 250 ⇄ 322MHz 변환 |
| CMAC RX에 역압이 없음 | Packet adapter의 FIFO가 받아서 역압을 만들어 줌 |
| 패킷이 중간에서 끊기면 안 됨 | 패킷 전체를 모은 뒤 내보내는 FIFO |

Project 01에서 직접 FIFO를 만들며 알게 된 내용(필요한 것은 CDC가 아니라 rate matching, 그리고 유효 beat의 정의)과 같은 문제를 이 블록이 풀고 있는 셈입니다.

---

# 인터페이스

## 제어: AXI4-Lite

- 클럭은 125MHz
- 일부 신호(`wstrb`, `awprot`, `arprot`)는 **생략된** 간단한 형태
- 각 구성 요소의 레지스터는 주소 범위로 나뉨 (시스템 수준 주소)
  - 322MHz 박스: `0x100000 ~ 0x1FFFFF`
  - 250MHz 박스: `0x200000 ~ 0x2FFFFF`

> 주소 맵은 버전에 따라 달라질 수 있으므로, 사용하는 버전의 README와 `system_config` 소스를 확인해야 합니다.

## 데이터: AXI4-Stream

두 박스는 같은 AXI4-Stream이지만 **사이드밴드(`tuser`) 신호와 제약이 다릅니다.**

| 박스 | tuser 신호 | 추가 제약 |
|------|-----------|-----------|
| 250MHz | `tuser_size`, `tuser_src`, `tuser_dst`, `tuser_user` | |
| 322MHz | `tuser_err` | 패킷 중간에 `tvalid`를 내리지 않음, CMAC RX 방향에는 `tready` 없음 |

### tuser_src / tuser_dst

250MHz 박스의 `tuser_src`/`tuser_dst`는 **패킷이 어디서 와서 어디로 가는지**를 담는 16비트 필드입니다. README가 정의한 비트 구성은 다음과 같습니다.

```
 15                       6   5   4   3           0
┌──────────────────────────┬───────┬───────────────┐
│        MAC ports         │ Rsvd  │   PCIe PFs    │
└──────────────────────────┴───────┴───────────────┘
```

- `[15:6]`: MAC 포트
- `[5:4]`: 예약
- `[3:0]`: PCIe PF

Shell에서 **나가는** 패킷(박스로 들어오는 패킷)은 `tuser_src`에 출발지가 표시되고 `tuser_dst`는 모두 0입니다. 즉 목적지를 정하는 것은 사용자 로직(기본은 p2p 플러그인)의 몫입니다.

### tkeep

- Shell에서 나가는 패킷은 `tkeep`이 `tuser_size`와 항상 일치합니다.
- Shell로 들어가는 패킷은 `tkeep`을 모두 1로 둘 수 있고, 이 경우 `tuser_size`가 패킷 길이를 결정합니다. 덕분에 사용자 플러그인은 `tkeep`을 생략할 수 있습니다.

322MHz 박스에서 "패킷 중간에 `tvalid`를 내리지 않는다"는 제약은, CMAC TX가 프레임 전송 중에 데이터가 끊기면 안 되는 특성과 연결됩니다. Project 01 006에서 본 `tkeep`이나 `tlast`로 프레임 경계를 표현하는 방식도 이 인터페이스에서 그대로 쓰입니다.

---

# 사용자 로직은 어디에 넣나

두 박스는 **패킷이 지나가는 길목**입니다. 기본 상태에서는 들어온 패킷을 그대로 반대편으로 넘기는 p2p 플러그인이 들어 있어서, 사용자는 이 자리에 자신의 로직(필터, 변환, 통계 등)을 끼워 넣을 수 있습니다.

- **250MHz 박스**는 호스트(QDMA) 쪽에 가까워 호스트와 주고받는 데이터를 다루기 좋습니다. `tuser_src`/`tuser_dst`로 PF와 포트 정보도 쓸 수 있습니다.
- **322MHz 박스**는 네트워크(CMAC) 쪽에 가까워 회선 속도에 가까운 처리를 하기 좋습니다. 대신 `tvalid`를 끊을 수 없고 RX 역압이 없다는 제약을 지켜야 합니다.

플러그인은 빌드 옵션 `-user_plugin`으로 경로를 주며, 한쪽 박스만 쓰려면 다른 쪽에는 기본 p2p 플러그인을 넣으라고 README가 안내합니다.

---

# What I Learned

- Shell의 데이터 경로는 **QDMA ⇄ 250MHz 박스 ⇄ Packet adapter ⇄ 322MHz 박스 ⇄ CMAC** 순서다.
- **250MHz와 322MHz는 비동기**라 Packet adapter가 CDC를 맡고, **125MHz와 250MHz는 위상 정렬**이라 CDC 없이 샘플링하되 이중 샘플링/누락만 주의하면 된다.
- CMAC RX에는 `tready`가 없는데, Packet adapter가 **패킷 단위 FIFO**로 역압을 복원한다.
- 제어는 **125MHz AXI4-Lite**, 데이터는 **250/322MHz AXI4-Stream**이며, 박스마다 `tuser` 신호와 제약이 다르다.
- `tuser_src`/`tuser_dst`는 **MAC 포트 / 예약 / PF** 비트로 구성되고, 목적지는 사용자 로직이 정한다.

---

# Next

다음 글에서는 호스트 쪽 **Linux 드라이버**가 이 Shell과 어떻게 연결되는지, 그리고 시험할 때 확인하는 방법을 정리합니다.

> **OpenNIC Study - 003: Linux 드라이버와 시험**

---

# References

- AMD OpenNIC Shell README – https://github.com/Xilinx/open-nic-shell
- AMD PG302 – QDMA Subsystem for PCI Express
- AMD PG203 – UltraScale+ Devices Integrated 100G Ethernet Subsystem
