---
title: "OpenNIC Study - 003: Linux 드라이버와 시험"
date: 2026-10-24 19:20:00 +0900
last_modified_at: 2026-10-27 11:00:00 +0900

categories:
  - Networking
  - OpenNIC Study

tags:
  - OpenNIC
  - Linux Driver
  - DPDK
  - ethtool
  - RSS

toc: true

comments: true
---

# Introduction

Shell(하드웨어)과 한 쌍인 **호스트 드라이버**를 살펴봅니다. PCIe / QDMA Study 003에서 QDMA가 "하드웨어와 드라이버가 한 쌍"이라고 한 것이 OpenNIC에서는 구체적으로 어떤 모습인지 정리합니다.

> 이 글은 OpenNIC **Linux 드라이버의 공개 README**(2026년 10월 확인한 `main` 브랜치)를 기준으로 정리했습니다. 장치 점검 체크리스트와 DPDK 시험 흐름은 README가 아니라 이전 스터디(PCIe / QDMA Study 007)의 내용입니다. 드라이버 버전과 지원 환경은 README마다 다를 수 있으므로, 사용하는 버전의 문서를 확인해야 합니다.

---

# 드라이버가 하는 일

OpenNIC의 Linux 커널 드라이버는 Shell이 제공하는 여러 PF와 큐를 **일반 네트워크 인터페이스**로 보이게 합니다.

- 여러 **PCIe PF**를 지원하고, PF마다 **여러 TX/RX 큐**를 가진다.
- 한 카드의 **100Gbps 포트 최대 2개**를 다룬다.
- PF마다 **하나의 net device**를 등록한다.
- 카드에 올라간 비트스트림의 **CMAC 개수를 감지**해서 링크를 관리한다.

PF 2개, CMAC 2개로 구성한 비트스트림에서는 **PF0이 port0, PF1이 port1**에 대응합니다. 포트의 링크를 켜고 끄는 것은 **마스터 PF인 PF0만** 할 수 있습니다.

---

# 큐 개수는 어떻게 정해지나

큐 개수는 드라이버가 임의로 정하지 않고 **PF의 MSI-X 벡터 수**에서 정해집니다.

| PF | 큐 개수 | 빠지는 벡터 |
|----|---------|-------------|
| PF0 (마스터) | MSI-X 벡터 수 − 2 | 카드 수준 에러 인터럽트 1개 + function 수준 사용자 인터럽트 1개 |
| 그 외 PF | MSI-X 벡터 수 − 1 | function 수준 사용자 인터럽트 1개 |

- 한 net device에서 **TX 큐와 RX 큐의 수는 같습니다.**
- MSI-X는 PCIe / QDMA Study 002에서 본 대로 장치가 **Memory Write TLP로 인터럽트를 보내는 방식**입니다. 큐마다 벡터를 하나씩 쓰고, 나머지를 위 용도로 예약하기 때문에 큐 수가 벡터 수보다 작아집니다.
- 카드 전체의 에러를 받는 벡터가 PF0에만 있다는 점이, PF0이 마스터 역할(링크 제어)을 한다는 것과 맞물립니다.

---

# 빌드와 로드

README가 안내하는 순서는 다음과 같습니다.

```bash
make                       # onic.ko 컴파일
sudo insmod onic.ko        # 모듈 로드 (선택 파라미터 RS_FEC_ENABLED=0|1)
dmesg                      # 오류 확인
ifconfig                   # 새 인터페이스가 보이는지 확인
```

주의할 점이 두 가지 있습니다.

- **케이블(또는 루프백 어댑터)을 먼저 연결한 뒤 모듈을 삽입**해야 합니다. 현재 드라이버는 링크 상태 변화를 감지하지 못하므로, 드라이버를 올리기 전에 링크가 준비되어 있어야 합니다.
- RS-FEC 사용 여부를 모듈 파라미터로 줄 수 있습니다. Project 01에서 CMAC 설정으로 RS-FEC를 고른 것과 이어지는 부분이며, **상대 장비와 같은 설정**이어야 링크가 올라옵니다.

---

# 확인하고 시험하는 방법

README가 설명하는 도구입니다.

| 도구 | 용도 |
|------|------|
| `dmesg` | 모듈 로드 오류 확인 |
| `ifconfig` / `ip` | 인터페이스가 보이는지 |
| `ethtool` | 링크 상태, 드라이버 정보(`-i`), MAC 통계 카운터(`-S`) |
| `pcimem` + BAR2 레지스터 | **루프백 모드** 설정 |
| `lm-sensors` | 전압, 전류, 온도 등 센서 (디자인에 CMS IP 필요, 소스의 `CMS_SUPPORT` 매크로 활성화) |

## 루프백 시험

**루프백 모드**를 BAR2의 레지스터로 켤 수 있다는 점이 유용합니다. 외부 장비 없이 **카드 안에서 TX를 RX로 되돌려** 볼 수 있기 때문입니다. Project 01에서 CMAC 루프백으로 첫 시험을 한 것과 같은 방식의 확인입니다.

| 포트 | 루프백 레지스터 (BAR2 오프셋) |
|------|-------------------------------|
| port0 | `0x8090` |
| port1 | `0xC090` |

`0x1`을 쓰면 켜지고 `0x0`을 쓰면 꺼집니다. 레지스터 접근에는 [pcimem](https://github.com/billfarrow/pcimem)을 씁니다.

```bash
# port0 루프백 켜기 (resource2 = BAR2, 경로의 BDF는 장치에 맞게)
sudo ./pcimem /sys/bus/pci/devices/0000:<BDF>/resource2 0x8090 w 0x1

# 끄기
sudo ./pcimem /sys/bus/pci/devices/0000:<BDF>/resource2 0x8090 w 0x0
```

README의 시험 시나리오는 간단합니다.

1. 인터페이스에 `tcpdump`를 걸어 둔다. (`sudo tcpdump -i <IF> -xx`)
2. 그 인터페이스의 **자기 IP로 `ping`** 을 보낸다.
3. 캡처된 패킷이 **항상 두 번씩** 보이면 루프백이 동작하는 것이다. 나간 패킷이 CMAC에서 되돌아와 다시 호스트로 올라오기 때문이다.

케이블 없이도 PCIe → QDMA → Shell 데이터 경로 → CMAC → 다시 호스트까지 **전체 경로를 한 번에 확인**할 수 있다는 점이 핵심입니다.

> 이 오프셋이 CMAC 내부의 어느 레지스터에 해당하는지는 README에 나오지 않습니다. 궁금하면 Shell의 `cmac_subsystem` 소스와 PG203의 레지스터 맵을 함께 확인해야 합니다.

## 장치 점검 순서

호스트 쪽에서 장치를 점검하는 순서는 PCIe / QDMA Study 007의 체크리스트와 같습니다.

1. `lspci`로 장치가 보이는가
2. `lspci -k`로 드라이버가 붙었는가
3. `dmesg`에 오류가 없는가
4. 인터페이스가 보이고 링크가 올라왔는가 (`ethtool <IF>`)
5. 통계 카운터가 증가하는가 (`ethtool -S <IF>`)

---

# RSS와 드라이버

PCIe / QDMA Study 006에서 RSS의 해시 키와 Indirection Table을 `ethtool`로 확인·변경할 수 있다고 했습니다. 다만 **드라이버가 지원해야** 합니다.

OpenNIC 드라이버 README는 **RSS 해시 키와 Indirection Table을 바꾸는 ethtool 루틴이 버전 1.0에서는 구현되지 않았다**고 밝힙니다. RSS 자체는 하드웨어(Shell)가 수행하지만, 호스트에서 그 설정을 바꾸는 경로는 드라이버 버전에 따라 제한될 수 있다는 뜻입니다.

---

# 알려진 문제

README에 적힌 주의사항입니다.

- 모듈을 삽입한 직후 **DHCP 클라이언트가 커널 패닉**을 일으킬 수 있습니다(`dmesg`에 kernel paging request 관련 BUG 메시지). 정적 IP를 지정하거나 DHCP를 끄라고 안내합니다.
- 모듈 설치 중 **시스템이 멈추는** 현상도 같은 DHCP 문제와 관련된 것으로 보이며, 역시 DHCP를 끄라고 권장합니다.

새로운 NIC가 생기면 OS가 자동으로 네트워크 설정을 시도하므로, 그 동작이 드라이버와 부딪히는 경우를 조심해야 한다는 일반적인 교훈으로 볼 수 있습니다.

---

# 지원 환경

README 기준으로 **Ubuntu 18.04, 20.04, 22.04**의 여러 커널에서 시험되었다고 합니다. 다른 배포판이나 더 새로운 커널에서는 빌드 오류가 날 수 있으므로, 커널 버전을 먼저 확인하는 것이 좋습니다.

또 Shell README에 따르면 Vivado 2022.2로 빌드하면 QDMA v5.0이 들어가므로 드라이버 업그레이드가 필요할 수 있습니다. **Shell 빌드 버전과 드라이버 버전도 한 쌍**으로 맞춰야 합니다.

---

# DPDK 드라이버

OpenNIC은 커널을 거치지 않는 **DPDK 드라이버**도 따로 제공합니다. 큐마다 전용 코어를 배정해서 폴링으로 패킷을 처리하는 방식이라 고성능 시험에 쓰입니다. PCIe / QDMA Study 007에서 본 DPDK 시험의 일반 흐름(hugepage → 드라이버 바인딩 → `testpmd` → 포트 통계)이 여기에 적용됩니다. 구체적인 사용법은 DPDK 저장소의 문서를 따라야 합니다.

---

# What I Learned

- OpenNIC 드라이버는 **PF마다 net device**를 만들고, 큐 수는 **MSI-X 벡터 수**에서 인터럽트용 벡터(PF0은 2개, 나머지는 1개)를 뺀 값이다.
- **케이블을 먼저 연결한 뒤** 모듈을 삽입해야 하고, RS-FEC 설정은 상대 장비와 맞아야 한다.
- BAR2의 `0x8090`(port0) / `0xC090`(port1) 레지스터로 **루프백 모드**를 켜면, 자기 IP로 ping을 보내 패킷이 두 번 잡히는지로 전체 경로를 확인할 수 있다.
- RSS 설정을 `ethtool`로 바꾸는 기능은 **드라이버 버전에 따라 제한**된다.
- 새 NIC에 대한 OS의 자동 네트워크 설정(DHCP)이 시험을 방해할 수 있다.

---

# Next

OpenNIC Study는 여기까지입니다. 이 구조를 실제 프로젝트에 적용한 내용은 공개 가능한 범위를 정한 뒤에 따로 정리할 예정입니다.

---

# References

- AMD OpenNIC Driver README – https://github.com/Xilinx/open-nic-driver
- AMD OpenNIC Shell README – https://github.com/Xilinx/open-nic-shell
- AMD OpenNIC – https://github.com/Xilinx/open-nic
- pcimem – https://github.com/billfarrow/pcimem
- AMD PG302 – QDMA Subsystem for PCI Express
- AMD PG203 – UltraScale+ Devices Integrated 100G Ethernet Subsystem
