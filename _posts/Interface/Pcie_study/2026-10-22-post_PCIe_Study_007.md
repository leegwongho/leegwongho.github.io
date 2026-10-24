---
title: "PCIe / QDMA Study - 007: 시리즈 정리와 디버깅 체크리스트"
date: 2026-10-22 16:00:00 +0900

categories:
  - Interface
  - PCIe Study

tags:
  - PCIe
  - QDMA
  - DPDK
  - Debugging
  - Checklist

toc: true

comments: true
---

# Introduction

001부터 006까지 PCIe의 구조, TLP, DMA와 QDMA의 큐, 멀티 큐와 RSS를 정리했습니다. 마지막 글에서는 시리즈 전체를 한 장의 지도로 묶고, **문제가 생겼을 때 어디부터 볼지**를 체크리스트로 정리합니다.

---

# 시리즈 지도

| 글 | 주제 | 한 줄 요약 |
|----|------|-----------|
| 001 | PCIe란 무엇인가 | 직렬 point-to-point, Lane과 Gen, 세 계층 |
| 002 | TLP와 Transaction | MRd/MWr/Cpl, BAR와 MMIO, Bus Master |
| 003 | DMA와 QDMA | PIO 대신 DMA, H2C/C2H, 큐 기반 DMA |
| 004 | Descriptor와 Queue | 링, PIDX/CIDX, Status Descriptor, CMPT |
| 005 | 링크 확인과 시험 | ILA와 `lspci`, 오픈소스 NIC 구조 시험 |
| 006 | 멀티 큐와 RSS | Toeplitz 해시와 Indirection Table |

전체 데이터 경로는 다음과 같이 이어집니다.

```
호스트 앱/드라이버
   │  Descriptor 작성, PIDX 갱신 (MMIO = MWr TLP)       ← 002, 004
   ▼
PCIe 링크                                               ← 001
   ▼
QDMA (큐, Descriptor Engine, H2C/C2H, Completion)       ← 003, 004
   ▼
AXI4-Stream ⇄ MAC(CMAC) ⇄ 네트워크                      ← Project 01
        ▲
     RSS로 수신 큐 선택                                  ← 006
```

---

# 문제를 계층별로 나누어 본다

PCIe 기반 NIC에서 "안 된다"는 말은 여러 곳을 뜻합니다. **아래에서 위로** 한 단계씩 확인하면 원인이 있는 층을 좁힐 수 있습니다.

| 단계 | 확인 대상 | 확인 방법 |
|------|-----------|-----------|
| ① | FPGA 쪽 링크 | ILA의 link up, 클럭과 리셋 |
| ② | 호스트가 장치를 인식하는가 | `lspci`에 표시, `LnkSta`의 속도와 Lane 수 |
| ③ | 드라이버가 장치에 붙었는가 | `lspci -k`의 Kernel driver in use, DPDK는 `dpdk-devbind.py --status` |
| ④ | 레지스터 접근 (MMIO) | BAR 읽기/쓰기, 응답 여부 |
| ⑤ | 큐와 DMA | 큐 설정, PIDX 갱신, Descriptor, 완료 처리 |
| ⑥ | 데이터 경로 | 패킷 카운터, 통계, 손실 |

**위 단계가 확인되지 않으면 아래 단계는 의미가 없습니다.** 예를 들어 ②에서 장치가 안 보이는데 ⑤의 큐 설정을 의심하는 것은 낭비입니다.

---

# 증상별 의심 위치

## ① ILA에서 link up이 안 올라온다

- 참조 클럭이 들어오는가, 리셋이 풀렸는가
- 슬롯과 보드 설정, 링크 폭과 속도 설정
- 001에서 본 **링크 트레이닝**이 끝나지 않는 것이므로, 한쪽 장치의 문제만이 아닐 수 있다.

## ② link up인데 `lspci`에 안 보인다

- 장치 인식은 **호스트 부팅 시 Enumeration**에서 이루어집니다. FPGA를 나중에 프로그래밍했다면 호스트 **재부팅이나 PCIe rescan**이 필요할 수 있습니다.
- 슬롯, BIOS의 PCIe 설정, 전원 순서도 확인합니다.
- **주의**: 장치가 `lspci`에 보이는 것은 원칙적으로 **드라이버와 무관**합니다. 드라이버가 필요한 것은 장치를 *사용*하는 단계(③)입니다. "드라이버를 설치했더니 보였다"라는 경험이 있다면, 그사이에 재부팅이나 rescan이 함께 있지 않았는지 되짚어 봐야 합니다.

```bash
sudo lspci -s <BDF> -vv                  # 인식된 속도, Lane 수
echo 1 | sudo tee /sys/bus/pci/rescan    # 재스캔 (장치가 있다면)
dmesg | grep -i pci                      # 커널 로그
```

## ③ `lspci`에는 보이는데 인터페이스나 DPDK 포트가 안 보인다

- `lspci -k`에서 **Kernel driver in use**가 비어 있으면 드라이버가 붙지 않은 상태입니다.
- 장치가 요구하는 **드라이버와 FPGA 설계의 버전이 맞는지** 확인합니다. 드라이버와 하드웨어가 서로 다른 레지스터 구조를 기대하면 장치는 보여도 동작하지 않습니다.
- DPDK는 보통 장치를 커널 드라이버에서 떼어 **DPDK용 드라이버(VFIO 등)에 바인딩**해야 합니다.

```bash
dpdk-devbind.py --status                 # 어떤 드라이버에 붙어 있는지
sudo dpdk-devbind.py -b <driver> <BDF>   # 바인딩
```

## ④ BAR 접근이 안 되거나 시스템이 멈춘다

- 링크가 불안정하거나 클럭이 흔들리면 MMIO 응답이 돌아오지 않을 수 있습니다. 002에서 본 대로 **MMIO 읽기는 Completion을 기다리는 Non-Posted 요청**이기 때문입니다.
- BAR 크기, 주소 구간, 접근 폭이 설계와 맞는지 확인합니다.

## ⑤ 장치는 보이는데 DMA가 안 된다

**링크와 장치 인식이 정상이라고 DMA가 정상이라는 뜻은 아닙니다.** (005)

- 큐 컨텍스트가 올바르게 설정되었는가 (링 크기, 베이스 주소, 모드)
- 드라이버가 **PIDX를 갱신**했는가
- Descriptor가 가리키는 주소가 장치가 접근 가능한 주소인가
- C2H에 **받을 Descriptor를 충분히 올렸는가** (없으면 하드웨어는 쓸 곳이 없어 멈춘다)
- 완료는 CIDX/Status 또는 CMPT로 확인하고 있는가 (004)

## ⑥ 패킷이 손실되거나 비정상이다

- 수신 큐가 가득 차거나 Descriptor가 부족해서 버려지는가
- 큐 사이에 분배가 한쪽으로 몰리는가 (006의 RSS 한계)
- 네트워크 쪽 손실은 Project 01 008에서 본 **클럭 오차와 IPG 여유** 같은 원인도 있습니다.
- 통계 신호가 증분인지 누적인지, 센 지점이 실제 이벤트인지 먼저 확인합니다. (Project 01 006, 007)

---

# DPDK로 시험하는 일반적인 흐름

오픈소스 NIC 구조(OpenNIC)로 시험할 때는 DPDK 드라이버로 데이터를 송수신했습니다. 처음에는 DPDK 명령어를 잘 몰라서 도움을 받아 가며 실행했습니다. 모르는 도구는 **전체 흐름을 먼저 잡고 각 명령이 무엇을 하는지 이해한 뒤** 쓰는 것이 안전합니다.

DPDK 시험의 일반적인 흐름은 다음과 같습니다.

| 단계 | 내용 |
|------|------|
| 1 | **Hugepage** 확보 (DPDK는 큰 페이지의 메모리를 사용) |
| 2 | 장치를 **DPDK용 드라이버에 바인딩** |
| 3 | 응용(예: `testpmd`)을 실행, 사용할 코어와 장치 지정 |
| 4 | 포트 시작, 송수신 |
| 5 | **포트 통계**로 송신·수신·오류 확인 |

```bash
# 일반적인 형태의 예 (옵션과 이름은 DPDK 버전과 설계에 따라 다릅니다)
sudo ./dpdk-testpmd -l 0-3 -n 4 -a <BDF> -- -i
testpmd> start
testpmd> show port stats all
```

- `-l`은 사용할 CPU 코어, `-a`는 사용할 장치의 허용 목록입니다.
- `-i`는 대화형 모드이고, `start`로 포워딩을 시작하고 `show port stats all`로 통계를 봅니다.
- OpenNIC의 DPDK 드라이버에는 전용 설정과 요구사항이 있으므로, 실제 실행 방법은 **해당 저장소의 문서**를 따라야 합니다.

멀티 큐를 쓰는 DPDK 응용에서는 **큐마다 코어를 배정**합니다. 006에서 본 "큐 = 코어" 구조가 그대로 나타나는 부분입니다.

---

# 이번 시리즈에서 배운 것

- 장치가 보이는 것(링크, 인식)과 데이터가 오가는 것(DMA)은 **다른 확인 단계**다.
- 문제는 **아래 계층에서 위 계층으로** 확인한다. 층을 건너뛰면 원인을 놓친다.
- QDMA의 모든 동작은 **TLP**로 이루어진다. Descriptor 읽기, PIDX 쓰기, 데이터 전송 모두 PCIe 트랜잭션이다.
- 큐는 하드웨어만의 문제가 아니라 **드라이버와 한 쌍**이다.
- 기억과 기록이 다를 수 있다. 어떤 조치가 어떤 결과를 만들었는지(드라이버 설치인지, 재부팅인지)는 **그 시점에 기록해야** 한다.

---

# 체크리스트 모음

- [ ] ILA의 link up과 `lspci`의 `LnkSta`(속도, Lane 수)를 모두 확인했는가
- [ ] FPGA를 프로그래밍한 뒤 호스트가 장치를 다시 인식하도록 재부팅 또는 rescan을 했는가
- [ ] `lspci -k`로 드라이버가 붙었는지 확인했는가 (DPDK는 `dpdk-devbind.py --status`)
- [ ] 드라이버와 FPGA 설계의 버전이 맞는가
- [ ] BAR 접근이 되는가 (MMIO 읽기/쓰기)
- [ ] 큐 설정, PIDX 갱신, Descriptor 주소를 확인했는가
- [ ] C2H에 받을 Descriptor를 충분히 올렸는가
- [ ] 완료를 CIDX/Status 또는 CMPT로 확인하는가
- [ ] 포트 통계의 카운터가 증분인지 누적인지 확인했는가
- [ ] 문제가 풀린 시점에 **증상 / 원인 / 조치**를 한 줄씩 기록했는가

---

# Next

PCIe / QDMA 이론 정리는 여기까지입니다. 다음 글에서는 지금까지 정리한 지식을 활용한 **다음 프로젝트**로 이어갑니다.

---

# References

- AMD PG302 – QDMA Subsystem for PCI Express
- AMD PG213 – UltraScale+ Devices Integrated Block for PCI Express
- AMD OpenNIC – https://github.com/Xilinx/open-nic
- DPDK Documentation – Linux Drivers (dpdk-devbind), testpmd Application User Guide
- PCI-SIG – PCI Express Base Specification
