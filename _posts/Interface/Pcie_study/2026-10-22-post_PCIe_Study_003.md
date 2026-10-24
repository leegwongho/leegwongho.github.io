---
title: "PCIe / QDMA Study - 003: DMA와 QDMA의 역할"
date: 2026-10-22 14:20:00 +0900

categories:
  - Interface

tags:
  - PCIe
  - DMA
  - QDMA
  - H2C
  - C2H

toc: true

comments: true
---

# Introduction

002에서 Endpoint가 **Bus Master**가 되어 호스트 메모리 주소로 직접 TLP를 보낼 수 있다는 것을 봤습니다. 이 원리로 데이터를 옮기는 것이 **DMA(Direct Memory Access)** 입니다.

이번 글은 DMA가 왜 필요한지, 그리고 AMD의 **QDMA** 가 어떤 DMA인지를 정리합니다.

> 이 글의 QDMA 설명은 **AMD PG302(QDMA Subsystem for PCI Express)** 문서의 개요와 구조 설명을 기준으로 했습니다. IP 버전에 따라 세부 사항이 다를 수 있으므로, 실제로 사용하는 버전의 문서를 확인해야 합니다.

---

# CPU가 직접 옮기면 어떻게 되나

FPGA의 데이터를 호스트 메모리로 가져오는 가장 단순한 방법은 CPU가 **MMIO로 한 워드씩 읽는 것**입니다(PIO, Programmed I/O).

```
CPU: for (i = 0; i < N; i++) buf[i] = *(bar0 + offset + i);
     → 읽기마다 MRd TLP → Completion 기다림 → 다음 읽기
```

이 방식에는 문제가 있습니다.

- 002에서 봤듯 **Memory Read는 Non-Posted**라서 매번 Completion을 기다리는 지연이 생깁니다.
- 한 번에 읽는 크기가 작아 TLP 헤더 오버헤드가 큽니다.
- 그 시간 동안 **CPU가 다른 일을 못 합니다.**

---

# DMA의 기본 흐름

DMA는 반대로 **장치가 직접 호스트 메모리에 읽고 씁니다.** CPU는 "무엇을, 어디서, 어디로, 얼마나" 옮길지만 알려 주고 빠집니다.

```
1. 드라이버: 호스트 메모리에 버퍼 준비
2. 드라이버: 전송 정보(주소, 길이)를 장치에 알림   ← Descriptor
3. 장치: Bus Master로 MRd/MWr TLP를 연속해서 발행
4. 장치: 완료를 알림 (인터럽트 또는 상태 기록)
5. 드라이버: 완료 확인 후 버퍼 정리
```

여기서 2번의 **전송 정보를 담는 구조**가 **Descriptor**이고, 다음 글의 주제입니다.

장치는 TLP를 최대 payload 크기(MPS) 단위로 연속해서 내보낼 수 있으므로, 한 워드씩 읽을 때보다 **훨씬 큰 대역폭**을 얻습니다.

## 방향: H2C와 C2H

DMA는 방향에 따라 이름이 정해져 있습니다. QDMA에서 쓰는 용어입니다.

| 용어 | 방향 | 장치 입장에서 | TLP |
|------|------|--------------|-----|
| **H2C** (Host to Card) | 호스트 → FPGA | 호스트 메모리를 **읽음** | MRd 후 CplD 수신 |
| **C2H** (Card to Host) | FPGA → 호스트 | 호스트 메모리에 **씀** | MWr 발행 |

H2C는 읽기라서 Completion을 받아야 하고, C2H는 쓰기라서 Posted로 보낼 수 있습니다.

---

# 왜 QDMA인가

## 기존 방식의 한계

데이터 흐름이 단순할 때는 "채널 몇 개"로 충분합니다. 하지만 NIC처럼 **패킷이 많고, 여러 CPU 코어와 여러 흐름이 동시에 접근**하는 시스템에서는 소수의 채널을 모두가 공유하게 되어 한계가 생깁니다.

## Queue 기반 DMA

PG302는 **큐(Queue)** 를 QDMA와 다른 DMA 제품을 구분하는 가장 큰 차이로 설명합니다.

- 큐 하나하나가 **독립적인 전송 통로**입니다.
- 큐마다 **인터페이스 종류와 동작 모드를 개별 설정**할 수 있습니다.
- 큐의 **설정과 갱신 오버헤드가 낮아** 많은 큐를 쓰기에 적합합니다.
- 큐를 여러 **PCIe Physical Function(PF)과 Virtual Function(VF)** 에 할당할 수 있습니다.
- **높은 대역폭과 많은 패킷 수**를 다루도록 설계되었습니다.

이를 NIC에 대입하면, 각 CPU 코어가 **자기만의 큐**를 가지고 서로 방해하지 않으면서 패킷을 주고받는 구조를 만들 수 있습니다.

---

# QDMA 개요

QDMA는 **UltraScale+ Integrated Block for PCI Express**(PG213) 위에 DMA와 브리지 기능을 얹은 서브시스템입니다.

```
  Host (드라이버)
      │  PCIe
┌─────┴───────────────────────────────┐
│ UltraScale+ Integrated PCIe Block   │
├─────────────────────────────────────┤
│            QDMA Subsystem           │
│  ┌────────────────────────────┐     │
│  │ Descriptor Engine          │     │
│  ├──────────┬─────────────────┤     │
│  │ H2C / C2H│ H2C / C2H       │     │
│  │ MM Engine│ Stream Engine   │     │
│  ├──────────┴─────────────────┤     │
│  │ Completion Engine          │     │
│  │ Interrupt Module           │     │
│  └────────────────────────────┘     │
│   AXI4-MM │ AXI4-Stream │ Bridge    │
└─────────────────────────────────────┘
     사용자 로직 (FPGA)
```

PG302의 구성 요소 목록에서 DMA 쪽의 주요 블록은 다음과 같습니다.

| 블록 | 역할 |
|------|------|
| **Descriptor Engine** | 호스트 메모리의 큐에서 Descriptor를 가져옴 |
| **H2C MM / C2H MM Engine** | 메모리 맵(MM) 방식 전송 |
| **H2C Stream / C2H Stream Engine** | 스트림 방식 전송 |
| **Completion Engine** | 완료 정보(Completion)를 호스트로 전달 |
| **Interrupt Module** | 인터럽트 처리 |

이 밖에도 선택적인 **AXI4 MM Master/Slave 브리지**가 있어서, DMA 엔진을 거치지 않고 PCIe 트래픽을 AXI로 바로 연결할 수도 있습니다.

---

# MM과 Stream

QDMA는 **큐마다** 사용자 로직과 연결되는 방식을 AXI4 Memory Mapped(MM)와 AXI4-Stream 중에서 고를 수 있습니다.

| | AXI4-MM | AXI4-Stream |
|---|---------|-------------|
| 개념 | **주소**를 가지고 읽고 쓰는 방식 | 주소 없이 **데이터가 흐르는** 방식 |
| Descriptor가 담는 것 | 호스트 주소, **카드 주소**, 길이 | 호스트 주소, 길이 |
| 사용자 로직 쪽 | DDR, HBM, BRAM 같은 메모리 | FIFO, 패킷 처리 파이프라인 |
| 어울리는 용도 | 큰 데이터 블록을 카드 메모리와 교환 | 패킷, 프레임 같은 **연속 데이터 흐름** |
| Descriptor 크기 | 32 B | H2C 16 B, C2H 8 B |

> Descriptor 크기는 PG302의 목차(AXI Memory Mapped Descriptor 32B, H2C Stream Descriptor 16B, C2H Stream Descriptor 8B)를 기준으로 정리했습니다.

Ethernet의 CMAC이 AXI4-Stream으로 프레임을 내보낸다는 것(005 글)을 떠올리면, **NIC 형태의 시스템은 Stream 모드**가 자연스럽습니다. CMAC에서 받은 프레임이 AXI4-Stream으로 QDMA에 들어가고, QDMA가 PCIe로 호스트 메모리에 씁니다(C2H).

```
네트워크 → CMAC → AXI4-Stream → QDMA (C2H Stream) → PCIe → 호스트 메모리
호스트 메모리 → PCIe → QDMA (H2C Stream) → AXI4-Stream → CMAC → 네트워크
```

## 큐별 완료 알림 방식

큐마다 **인터럽트 모드**와 **폴링 모드** 중에서도 고를 수 있습니다.

- **인터럽트**: 완료되면 MSI-X로 호스트에 알림(002에서 본 것처럼 Memory Write TLP)
- **폴링**: 드라이버가 메모리의 완료 정보를 주기적으로 확인

패킷이 매우 많을 때는 인터럽트가 병목이 될 수 있어서 폴링 모드를 쓰는 경우가 많습니다.

---

# 소프트웨어와의 관계

QDMA는 하드웨어만으로 동작하지 않고 **호스트 드라이버와 한 쌍**입니다. PG302는 AMD가 제공하는 **QDMA reference driver**로 IP를 시험할 수 있고, 이를 응용에 맞게 확장해서 쓸 수 있다고 설명합니다.

호스트 OS가 만든 Descriptor를 하드웨어가 읽어 가는 구조이므로, 하드웨어 설계자도 **드라이버가 무엇을 언제 어떻게 쓰는지** 이해해야 디버깅이 가능합니다.

---

# What I Learned

- PIO는 MRd를 반복하기 때문에 지연과 CPU 점유 문제가 있고, DMA는 장치가 Bus Master로 직접 옮겨 **CPU를 해방**한다.
- H2C는 호스트 메모리를 읽고(MRd), C2H는 호스트 메모리에 쓴다(MWr).
- QDMA의 핵심은 **큐**다. 큐마다 인터페이스(MM/Stream)와 완료 방식(인터럽트/폴링)을 개별 설정하고, PF/VF에 할당할 수 있다.
- **패킷 중심의 NIC 구조에는 AXI4-Stream 모드**가 자연스럽다.
- QDMA는 **하드웨어와 드라이버가 한 쌍**으로 동작한다.

---

# Next

QDMA에서 가장 중요한 구조인 **Descriptor와 Queue(Ring)** 를 살펴봅니다. 호스트와 하드웨어가 어떤 인덱스로 서로의 진행 상황을 알려 주는지가 핵심입니다.

> **PCIe / QDMA Study - 004: QDMA Descriptor와 Queue**

---

# References

- AMD PG302 – QDMA Subsystem for PCI Express (Overview, QDMA Architecture)
- AMD PG213 – UltraScale+ Devices Integrated Block for PCI Express
- PCI-SIG – PCI Express Base Specification
