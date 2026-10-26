---
title: "Portfolio: 고속 데이터 경로 설계와 디버깅 정리"
date: 2026-10-26 13:50:00 +0900

categories:
  - ETC
  - Portfolio

tags:
  - Portfolio
  - FPGA
  - Ethernet
  - PCIe
  - QDMA
  - Debugging

featured: true
featured_order: 1
description: "100G CMAC, PCIe/QDMA, OpenNIC 구조를 직접 시험하며 정리한 기록과 문제 해결 방식을 한 번에 보는 글"

toc: true

comments: true
---

# Introduction

이 블로그에 쓴 네 개의 시리즈를 **한 번에 볼 수 있도록 묶은 글**입니다. 각 시리즈에서 무엇을 했고, 어떻게 확인했고, 무엇을 배웠는지를 짧게 정리합니다.

> 이 글에는 소속 회사의 정보(제품, 설계, 설정값, 측정 결과)를 싣지 않았습니다. 개인 학습과 공개 자료, 그리고 일반 원칙으로 정리할 수 있는 내용만 다룹니다.

---

# 시리즈 바로가기

{% include series_list.html %}

---

# 한눈에 보기

| 영역 | 내용 | 확인 방법 |
|------|------|-----------|
| **100G Ethernet MAC** | CMAC 설정, 프레임 생성기(TPG), 수신 경로, FIFO | ILA, 통계 카운터, 계측기 |
| **PCIe / QDMA** | 링크, TLP, 큐와 Descriptor 구조, 멀티 큐 | ILA, `lspci`, 드라이버 통계 |
| **OpenNIC 구조** | Shell, 클럭 도메인, Packet Adapter, 드라이버 | 공개 저장소와 문서 분석 |
| **디버깅 원칙** | 카운터 배치, 타이밍 리포트 읽기 | 카운터 비교, 타이밍 리포트 |

---

# 1. Project 01 — 100G CMAC Bring-up

**목표:** VCU128 보드에서 100G Ethernet MAC(CMAC)을 직접 구성하고, 외부 계측기와 연결해 송수신을 확인한다.

**한 일**

- CAUI-4, 참조 클럭, RS-FEC 사용 여부 등 **IP 설정**과 링크 정렬 상태 확인
- Ethernet 프레임 구조를 이해하고 **TPG(Test Pattern Generator)** 를 설계
- RX 경로가 나오지 않는 문제를 **ILA로 추적**
- 클럭과 FIFO가 필요한 이유 분석 (필요한 것은 CDC가 아니라 속도 정합)
- 계측기와 연동해 **장시간 시험**

**기억에 남는 문제**

- RX가 나오지 않던 원인은 **`tkeep` 신호가 1비트로 잘못 선언된 것**이었습니다. 64비트가 필요한 신호가 1비트 wire로 암묵 선언되어 동작이 어긋났고, `default_nettype none`을 쓰면 컴파일 단계에서 잡을 수 있습니다.
- ILA 프로브는 **IP 포트에 연결된 네트**에 붙여야 실제 값이 보였습니다.

**남아 있는 것**

- 계측기에서 **0.0001% 수준의 미수신**이 있었고, 클럭 ppm 차이와 TX 프레임 간격(IPG) 여유를 원인 후보로 봤습니다. 그러나 **확정하지는 못했습니다.**

**배운 점**

- 증상이 아니라 **기록된 신호와 설정**으로 원인을 좁혀야 한다.
- 추정과 확정은 구분해서 적어야 한다.

---

# 2. PCIe / QDMA Study

**목표:** PCIe와 QDMA를 **구조 → 동작 → 실제 확인** 순서로 이해한다.

**한 일**

- PCIe의 Lane, Gen, 대역폭, **계층 구조(Transaction / Data Link / Physical)**, TLP 종류, Posted / Non-posted, MPS / MRRS 정리
- DMA의 개념과 **QDMA 큐 구조**(H2C / C2H / Completion 링, PIDX / CIDX, 컨텍스트)를 PG302 기준으로 정리
- 예제 디자인으로 **ILA의 link up과 `lspci`의 `LnkSta`** 를 함께 확인하는 방법 정리
- **멀티 큐와 RSS**(Toeplitz 해시, Indirection Table)를 이론으로 정리하고, 해시 값을 직접 계산해 **공개된 검증 벡터와 비교**

**배운 점**

- **장치가 보이는 것과 데이터가 오가는 것은 다른 단계**다.
- 해시처럼 결과만 보고 맞는지 알 수 없는 로직은 **알려진 정답과 비교**해서 검증한다.

---

# 3. OpenNIC Study

**목표:** 오픈소스 NIC 플랫폼인 OpenNIC의 구조를 **공개 저장소 문서를 읽고 분석**한다.

**한 일**

- Shell, Linux 드라이버, DPDK 드라이버로 이루어진 **구성 요소와 빌드 파라미터** 정리
- **250MHz(QDMA)와 322MHz(CMAC)** 로 나뉜 클럭 도메인과 Packet Adapter의 역할 분석
- 드라이버가 PF와 큐를 어떻게 다루는지, 시험 방법 정리

**배운 점**

- 서로 다른 IP 사이에서는 **클럭이 다르고 CDC가 필요**하다는 점이 Project 01의 CMAC 내부와 대조된다.
- CMAC RX에 없는 **역압을 중간 FIFO가 복원**하는 구조가 어디서 필요한지 이해했다.

---

# 4. Project 02 — 고속 데이터 경로 디버깅 원칙

**목표:** 오픈소스 NIC 구조를 다른 환경에 적용할 때 일반적으로 쓰는 **확인 방법**을 정리한다.

- **001** 폭과 클럭이 바뀔 때 확인할 것, 파서와 해시의 단계별 검증, 타이밍과 분산 문제의 접근법
- **002** 경로 곳곳의 **카운터를 한 줄로 읽어 멈춘 위치를 찾는 방법** (정지와 손실 구분, `tvalid`/`tready` 해석)
- **003** **타이밍 리포트**를 읽는 순서와 원인별 대응

---

# 문제를 푸는 방식

네 시리즈에 공통으로 쓴 방식입니다.

1. **현상과 원인을 구분한다.** 카운터가 멈춘 것은 현상이고, 원인은 마지막으로 증가한 곳과 처음으로 증가하지 않은 곳 사이에 있다.
2. **나누어서 확인한다.** 인터페이스, 프로토콜 처리, 타이밍처럼 서로 다른 종류의 문제를 한꺼번에 보지 않는다.
3. **알려진 값과 비교한다.** 해시나 레지스터처럼 맞는지 알기 어려운 결과는 공개된 검증 값이나 소프트웨어 결과와 대조한다.
4. **확정한 것과 후보를 구분해서 기록한다.** 시간이 지나면 후보가 확정처럼 기억되기 때문이다.

---

# 사용한 도구

- **설계/디버깅:** Vivado, ILA, Verilog
- **호스트:** `lspci`, `ethtool`, DPDK `testpmd`
- **시험:** 네트워크 계측기(트래픽 발생과 분석)
- **검증:** Python 스크립트(해시 값 계산 등)

---

# 앞으로

- 타이밍 클로저와 메모리 인터페이스(DDR) 주제를 추가로 정리할 예정입니다.
- 시리즈를 읽다가 **틀린 내용**이나 **더 정확한 근거**가 있다면 댓글로 알려 주세요.

---

# References

- AMD PG203 – UltraScale+ Devices Integrated 100G Ethernet Subsystem
- AMD PG302 – QDMA Subsystem for PCI Express
- AMD OpenNIC – https://github.com/Xilinx/open-nic
