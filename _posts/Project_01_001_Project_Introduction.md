---
title: "Project 01 - 001. Project Introduction"
date: 2026-07-08 20:00:00 +0900

categories:
  - Networking

tags:
  - 100G
  - Ethernet
  - CMAC
  - FPGA
  - VCU128

pin: true

toc: true

comments: true
---

# Overview

입사 후 처음 수행한 프로젝트는 **100G Ethernet CMAC Bring-up** 프로젝트였습니다.

본 프로젝트의 목표는 Xilinx에서 제공하는 **100G Ethernet CMAC IP**를 이용하여 100Gb Ethernet Link를 정상적으로 구성하고, 패킷을 송수신할 수 있는 환경을 구축하는 것이었습니다.

프로젝트를 진행하면서 단순히 CMAC IP만 사용하는 것이 아니라 Ethernet 시스템을 구성하는 다양한 요소들을 함께 학습하게 되었습니다.

- GT Transceiver
- CMAC
- PCS/PMA
- Ethernet Frame
- AXI4-Stream Interface
- Clock & Reset
- Link Bring-up
- Debug 및 ILA 활용

이 프로젝트는 이후 진행하게 되는 OpenNIC 기반 10G Ethernet 개발과 SmartNIC 기능 구현의 기초가 되었습니다.

---

# Project Goal

프로젝트의 주요 목표는 다음과 같습니다.

- 100G Ethernet CMAC IP 생성 및 설정
- GT Transceiver 연결
- QSFP28 인터페이스 구성
- 100Gb Ethernet Link Up 확인
- Packet 송신
- Packet 수신
- Loopback Test
- 동작 검증 및 Debug

---

# Hardware Platform

| Item | Description |
|------|-------------|
| Development Board | Xilinx VCU128 Evaluation Board |
| FPGA Device | Virtex UltraScale+ XCVU37P |
| Optical Interface | QSFP28 |
| Test Equipment | Spirent Traffic Generator |

---

# Software Environment

| Item | Version |
|------|---------|
| Vivado | 2023.2 |
| Hardware Manager | Vivado Hardware Manager |
| Debug Tool | Integrated Logic Analyzer (ILA) |

---

# Development Flow

프로젝트는 다음과 같은 순서로 진행되었습니다.

1. 개발 환경 구축
2. 100G Ethernet 구조 이해
3. CMAC IP 분석
4. GT Wizard 분석
5. CMAC Bring-up
6. Packet Generator 구현
7. Packet 송신
8. Packet 수신
9. Loopback Test
10. Debug 및 성능 검증

이후 작성되는 Project 01의 글들은 위 순서에 맞추어 하나씩 정리할 예정입니다.

---

# What I Learned

이번 프로젝트를 통해 처음으로 FPGA 기반 Ethernet 시스템을 직접 구성해 보았습니다.

특히 아래 항목들에 대한 이해를 크게 높일 수 있었습니다.

- FPGA에서 Ethernet이 동작하는 전체 구조
- GT Transceiver의 역할
- CMAC IP의 내부 구성
- AXI4-Stream 인터페이스
- Clock Domain Crossing(CDC)
- Link Bring-up 과정
- Hardware Debug 방법

이 경험은 이후 진행한 OpenNIC 기반 10G Ethernet 프로젝트와 SmartNIC 개발 프로젝트의 기반이 되었습니다.

---

# Next

다음 글에서는 프로젝트를 진행했던 개발 환경을 소개합니다.

> **Project 01 - 002. Development Environment**
