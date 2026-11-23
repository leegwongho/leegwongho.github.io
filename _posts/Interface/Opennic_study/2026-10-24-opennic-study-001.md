---
title: "OpenNIC Study - 001: OpenNIC 개요와 구성 요소"
date: 2026-10-24 19:00:00 +0900
last_modified_at: 2026-10-27 11:00:00 +0900

categories:
  - Networking
  - OpenNIC Study

tags:
  - OpenNIC
  - NIC
  - FPGA
  - QDMA
  - CMAC

toc: true

comments: true
---

# Introduction

Project 01에서 100G Ethernet CMAC을, PCIe / QDMA Study에서 PCIe와 QDMA를 정리했습니다. 이 두 가지가 한 구조로 묶인 공개 프로젝트가 **OpenNIC**입니다.

이 시리즈는 OpenNIC의 **공개 저장소와 문서를 읽고 구조를 분석한 학습 글**입니다.

> 이 글은 공개된 저장소(README)의 내용만 근거로 정리했습니다. 업무 프로젝트와 관련된 설계, 수정, 측정 결과는 포함하지 않습니다. 기준은 2026년 10월에 확인한 각 저장소 `main` 브랜치의 README입니다. 저장소는 버전에 따라 내용이 바뀔 수 있으므로(지원 보드, Vivado 버전, 주소 맵 등), 실제로 사용하는 버전의 문서를 확인해야 합니다.

---

# OpenNIC이란

OpenNIC은 AMD(Xilinx)가 오픈소스 커뮤니티를 위해 공개한 **FPGA 기반 NIC 플랫폼**입니다. 목적은 **하드웨어 가속 네트워크 응용을 빠르게 프로토타이핑**하는 것이며, 저장소는 이것이 완성된 SmartNIC 솔루션은 아니라고 밝히고 있습니다.

FPGA에 올린 NIC가 서버에서는 **일반 네트워크 카드처럼** 보이고, 그 사이에 사용자가 자기 로직을 끼워 넣을 수 있는 구조입니다.

---

# 세 가지 구성 요소

OpenNIC은 세 개의 저장소로 나뉩니다.

| 구성 요소 | 역할 |
|-----------|------|
| **OpenNIC Shell** | FPGA에 올라가는 하드웨어(RTL). PCIe, QDMA, CMAC, 사용자 로직 연결부 |
| **OpenNIC Driver** | 호스트의 Linux 커널 드라이버 |
| **OpenNIC DPDK** | DPDK 기반 드라이버 (커널을 거치지 않는 고성능 경로) |

```
┌──────────────────────────────────────┐
│ 호스트                                │
│   Linux 커널 드라이버  /  DPDK 드라이버│  ← 소프트웨어
└───────────────┬──────────────────────┘
                │ PCIe
┌───────────────┴──────────────────────┐
│ FPGA : OpenNIC Shell                  │  ← 하드웨어
│   QDMA ⇄ (사용자 로직) ⇄ CMAC         │
└───────────────┬──────────────────────┘
                │ 100G Ethernet
              네트워크
```

호스트 쪽 소프트웨어와 FPGA 쪽 하드웨어가 **한 쌍**이라는 점은 PCIe / QDMA Study 003에서 본 내용과 같습니다. Shell의 큐 설정과 드라이버가 서로 맞아야 동작합니다.

---

# Shell이 제공하는 것

공개된 설명을 기준으로 하면 Shell은 다음을 제공합니다.

- 최대 **4개의 PCIe Physical Function(PF)**
- 최대 **2개의 100Gbps Ethernet 포트**
- 사용자 로직을 붙일 수 있는 **정의된 데이터 인터페이스와 제어 인터페이스**
- 드라이버 쪽에서는 PF마다 **여러 개의 TX/RX 큐**

PF가 여러 개라는 것은 서버에서 **네트워크 인터페이스가 여러 개**로 보인다는 뜻입니다. PF와 포트의 대응은 빌드 구성에 따라 달라집니다. 드라이버 README에 따르면 **PF 2개, CMAC 2개로 구성한 비트스트림**에서는 PF0이 port0, PF1이 port1에 대응합니다. (빌드 옵션의 기본값 자체는 PF 1개, 포트 1개입니다.)

---

# 지원 보드와 빌드 도구

현재 저장소 README 기준으로 지원 보드는 AMD Alveo 계열의 **U50, U55N, U55C, U200, U250, U280, U45N**입니다. 빌드 옵션의 보드 이름으로는 `au50`, `au55n`, `au55c`, `au200`, `au250`, `au280`, `U45N`이 나옵니다. Bittware SoC-250은 OpenNIC 1.0부터 지원이 더 이상 유지되지 않는다고 적혀 있습니다.

빌드 도구는 **Vivado 2020.x, 2021.x, 2022.1**을 기준으로 합니다. Vivado 2022.2는 QDMA v5.0을 쓰기 때문에 드라이버 쪽 업그레이드가 필요할 수 있다고 안내합니다.

> U50은 Vivado가 MGT 전원 레일 마진과 관련된 critical warning을 낼 수 있다고 README가 안내합니다. 실제 전류는 레일 한계보다 약간 낮지만 보드 파일이 정한 10% 마진을 벗어나며, U50 버전은 시험이 제한적이었다고 적혀 있습니다.

> 이전 버전 README는 지원 보드를 U50/U200/U250/U280으로, 빌드 도구를 Vivado 2020.2로 설명했습니다. 이처럼 버전에 따라 내용이 다르므로, 글을 읽을 때 기준 버전을 확인해야 합니다.

---

# 빌드 파라미터

Shell은 Vivado의 Tcl 스크립트(`build.tcl`)로 빌드하고, 구성은 **옵션**으로 정합니다. 기능과 직접 관련된 것만 정리하면 다음과 같습니다.

| 옵션 | 의미 | 범위 / 기본값 |
|------|------|---------------|
| `-board` | 대상 보드 | `au50`, `au250`, `au280` 등 |
| `-num_phys_func` | **QDMA 서브시스템당** PF 개수 | 1 ~ 4 (기본 1) |
| `-num_qdma` | QDMA 서브시스템 개수 | 1 (기본) 또는 2 (보드에 따라, 예: U45N) |
| `-num_queue` | QDMA 큐 개수 | 1 ~ 2048 (기본 512) |
| `-num_cmac_port` | CMAC 포트 수 | 1 (기본) 또는 2 (보드에 따라) |
| `-user_plugin` | 사용자 로직(플러그인) 경로 | |
| `-tag` | 빌드 이름 | 출력은 `build/[BOARD]_[TAG]` |

PCIe / QDMA Study 004에서 본 **큐(링)와 컨텍스트 구조**의 개수가, 여기서는 **빌드 시점의 옵션**으로 정해지는 것입니다. 큐가 많을수록 호스트와 하드웨어가 더 많은 컨텍스트를 관리해야 합니다.

빌드는 크게 네 단계입니다: IP 생성 → 프로젝트 구성 → 합성/구현/비트스트림 → 후처리. 기본으로는 앞의 두 단계(프로젝트 생성까지)만 수행하고, `-impl 1`, `-post_impl 1` 옵션으로 뒤 단계를 이어서 수행합니다.

---

# 이 시리즈에서 다루는 것

| 글 | 내용 |
|----|------|
| 001 | OpenNIC 개요와 구성 요소 (이 글) |
| 002 | Shell의 데이터 경로와 클럭 도메인 |
| 003 | Linux 드라이버와 시험 |

---

# What I Learned

- OpenNIC은 **Shell(FPGA) + Linux 드라이버 + DPDK 드라이버**로 구성된 공개 FPGA NIC 플랫폼이다.
- Shell은 최대 4개 PF, 최대 2개의 100G 포트, 빌드 시점에 정하는 큐 개수(최대 2048)를 지원한다.
- PF와 포트의 대응은 빌드 구성에 따라 다르며, 2PF/2포트 구성에서는 PF0→port0, PF1→port1이다.
- 앞서 정리한 CMAC(Project 01), PCIe와 QDMA(PCIe / QDMA Study)가 이 구조 안에서 한 경로로 이어진다.
- 지원 보드와 Vivado 버전은 README 버전마다 다르므로, 사용하는 버전의 README를 확인해야 한다.

---

# Next

다음 글에서는 Shell 안의 **데이터가 어떤 블록들을 거쳐 흐르는지**, 그리고 왜 클럭이 250MHz와 322MHz로 나뉘는지를 살펴봅니다.

> **OpenNIC Study - 002: Shell의 데이터 경로와 클럭 도메인**

---

# References

- AMD OpenNIC – https://github.com/Xilinx/open-nic
- AMD OpenNIC Shell README – https://github.com/Xilinx/open-nic-shell
- AMD OpenNIC Driver README – https://github.com/Xilinx/open-nic-driver
