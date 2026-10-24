---
title: "PCIe / QDMA Study - 004: QDMA Descriptor와 Queue"
date: 2026-10-22 14:30:00 +0900

categories:
  - Interface

tags:
  - QDMA
  - Descriptor
  - Queue
  - PIDX
  - CIDX

toc: true

comments: true
---

# Introduction

003에서 QDMA의 핵심이 **큐(Queue)** 라는 것을 봤습니다. 이번 글은 큐가 실제로 어떤 구조이고, **호스트 소프트웨어와 하드웨어가 어떻게 서로의 진행 상황을 주고받는지**를 정리합니다.

> 이 글은 **AMD PG302 v4.0**의 "General Design of Queues"와 "H2C and C2H Queues" 설명을 기준으로 했습니다. 컨텍스트 개수 같은 세부 수치는 IP 버전에 따라 다를 수 있으므로, 사용하는 버전의 문서로 확인해야 합니다.

---

# Queue는 Ring이다

QDMA의 큐는 **호스트 메모리에 있는 원형 버퍼(Ring)** 입니다. 큐 하나의 세트는 세 개의 링으로 구성됩니다.

| Ring | 누가 쓰나 | 내용 |
|------|-----------|------|
| **H2C** | 소프트웨어가 씀, 하드웨어가 읽음 | 호스트 메모리에서 **읽어 갈** 데이터에 대한 Descriptor |
| **C2H** | 소프트웨어가 씀, 하드웨어가 읽음 | 호스트 메모리에 **써 줄** 데이터를 받을 버퍼에 대한 Descriptor |
| **CMPT** (Completion) | **하드웨어가 씀** | C2H Stream 패킷의 완료 정보, 또는 사용자 정의 완료/알림 |

H2C와 C2H 링에서는 **소프트웨어가 Producer, Descriptor Engine이 Consumer**입니다. CMPT 링은 반대로 하드웨어가 채우고 소프트웨어가 읽습니다.

---

# Descriptor

**Descriptor**는 "한 번의 전송에 필요한 정보"를 담은 항목입니다. 링의 각 칸이 Descriptor 하나입니다.

| 큐 종류 | Descriptor가 설명하는 것 |
|---------|-------------------------|
| **H2C** | 호스트가 준비한 **payload 버퍼** (이 데이터를 카드로 가져가라) |
| **C2H** | 호스트가 마련한 **수신 공간** (카드에서 오는 데이터를 여기에 써라) |

담기는 필드는 큐의 **컨텍스트 설정**에 따라 달라집니다.

| 모드 | Descriptor 필드 |
|------|-----------------|
| **Memory Mapped** | 호스트 주소, 카드 주소, 길이 |
| **Stream** | 호스트 주소, 길이 |
| Stream, C2H (internal cached mode) | 호스트 주소만 |

- **Internal 모드**의 H2C Descriptor는 gather descriptor라고 부르며, 주소·길이에 **32비트 메타데이터**가 추가됩니다.
- **버퍼 크기는 큐마다 고정**이지만, 전송이 버퍼를 꽉 채울 필요는 없습니다.
- **Descriptor Bypass 모드**에서는 임의의 형식을 쓸 수 있습니다. Descriptor가 DMA 엔진이 아닌 사용자 로직으로 전달되기 때문입니다.

---

# PIDX와 CIDX

링에서 어디까지 채웠고 어디까지 처리했는지는 두 개의 **인덱스**로 표현합니다.

| 인덱스 | 이름 | 누가 갱신 | 의미 |
|--------|------|-----------|------|
| **PIDX** | Producer Index | 소프트웨어 | "여기까지 유효한 Descriptor를 채웠다" |
| **CIDX** | Consumer Index | 하드웨어 | "여기까지 처리했다" |

각자 상대 인덱스의 **사본**을 들고 있어서 서로 침범하지 않습니다.

- **소프트웨어**는 자신의 PIDX와 하드웨어 CIDX의 사본을 보고, 아직 읽히지 않은 Descriptor를 덮어쓰지 않습니다.
- **엔진**은 자신의 CIDX와 소프트웨어 PIDX의 사본을 보고, 아직 쓰이지 않은 Descriptor를 읽지 않습니다.

```
         ┌───┬───┬───┬───┬───┬───┬───┬────────┐
  Ring   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ Status │
         └───┴───┴───┴───┴───┴───┴───┴────────┘
                ▲           ▲
              CIDX         PIDX
         (하드웨어 처리 위치)  (소프트웨어 채운 위치)
```

## 링의 크기와 마지막 칸

- 링 크기는 컨텍스트 프로그래밍으로 고르며, **16가지 선택지**가 있습니다.
- **링의 마지막 칸은 Status Descriptor 용으로 예약**되어 있어서, 실제로 쓸 수 있는 칸은 **링 크기 − 1** 입니다.
- 링의 베이스 주소는 **4KB 정렬**이어야 합니다.

문서의 예시입니다. 링 크기가 8이면 CIDX가 0일 때 쓸 수 있는 인덱스는 0~6이고, 그래서 PIDX를 최대 6까지만 갱신할 수 있습니다. CIDX가 4라면 PIDX는 최대 3까지입니다. 링이 가득 차면 PIDX가 CIDX를 따라잡아 덮어쓰는 일이 생기지 않도록 **하나는 항상 비워 둔다**고 이해하면 됩니다.

---

# 전송이 일어나는 순서

H2C와 C2H 큐에서 하나의 전송은 다음 순서로 진행됩니다.

1. **소프트웨어**가 Descriptor를 링의 PIDX 위치에 씁니다.
   - H2C: 먼저 payload를 호스트 버퍼에 쓰고, 그 버퍼를 설명하는 Descriptor를 링에 둡니다.
   - C2H: 데이터를 받을 버퍼를 설명하는 Descriptor를 둡니다.
2. 소프트웨어가 하드웨어의 **PIDX 레지스터에 해당 큐(QID)로 쓰기**를 합니다. 새 PIDX 값을 전달합니다.
   - 이 레지스터 쓰기는 002에서 본 **BAR 주소로 가는 MWr TLP**입니다.
3. 엔진은 레지스터 주소 오프셋과 Function ID로 **절대 QID**를 구하고, 그 큐의 **컨텍스트**를 가져옵니다.
4. 컨텍스트를 보고 **몇 개의 Descriptor를 가져와도 되는지** 계산합니다. 그리고 **BADDR, CIDX, Descriptor 크기**로 각 Descriptor의 주소를 구해 **호스트 메모리에 DMA 읽기(MRd)** 를 합니다.
5. Descriptor가 돌아오면 **Internal 모드**에서는 H2C 또는 C2H 엔진으로 전달해 실제 데이터를 옮깁니다. **Bypass 모드**에서는 Descriptor가 **사용자 로직으로 가는 Bypass 출력 인터페이스**로 나갑니다.

Descriptor를 가져오는 것도 **PCIe 읽기 TLP**라는 점이 핵심입니다. 데이터 전송 전에 Descriptor 읽기가 먼저 일어납니다.

---

# 완료는 어떻게 알리나

## Status Descriptor

링의 마지막 칸이 **Status Descriptor**입니다. 엔진이 가져온 Descriptor의 처리를 마치면 **하드웨어 CIDX**와 그 밖의 상태를 여기에 씁니다.

- **Memory Mapped 큐**와 **Internal 모드의 H2C Stream 큐**는 처리 후 CIDX를 Status Descriptor에 씁니다.
- **Bypass 모드** 큐는 사용자 로직이 Bypass 입력 인터페이스로 Writeback을 제어합니다.
- 컨텍스트 설정으로 Status Descriptor를 **얼마나 자주 쓸지(moderation)** 조절할 수 있습니다.

소프트웨어는 이 CIDX를 보고 **어떤 Descriptor를 회수할 수 있고, 어떤 호스트 버퍼를 해제할 수 있는지** 판단합니다.

## C2H Stream의 완료: CMPT Ring

C2H Stream 큐는 방식이 다릅니다. **항상 CMPT 링으로 완료를 알립니다.**

- 보통 **C2H 패킷 하나당 CMPT 항목 하나**가 만들어지고, 그 안에 **몇 개의 C2H Descriptor를 소비했는지** 등이 들어 있습니다.
- 소프트웨어는 **Color bit** 또는 CMPT 링 끝의 Status Descriptor로 새 항목이 도착했는지 확인합니다.
- C2H Stream 링은 CIDX가 아니라 **CMPT 항목**을 보고 Descriptor를 회수합니다.

Stream으로 패킷이 들어오면 패킷의 **길이가 미리 정해져 있지 않기** 때문에, 하드웨어가 "이 패킷은 이만큼이다"라고 알려 주는 별도의 통로(CMPT)가 필요한 것으로 이해할 수 있습니다.

---

# Queue Context

큐 하나의 속성은 **컨텍스트(Context)** 에 저장됩니다.

- 엔진은 컨텍스트를 **로컬 메모리**에 가지고 있습니다. v4.0 문서 기준으로 **H2C 2048개, C2H 2048개**입니다.
- 컨텍스트에는 링 베이스 주소(BADDR), 소프트웨어 PIDX, CIDX, 큐 크기(depth) 같은 큐의 속성이 들어갑니다.
- MM인지 Stream인지, 링 크기, Bypass 여부 같은 동작도 컨텍스트 프로그래밍으로 정해집니다.

PG302에는 컨텍스트 관리를 다루는 절(Function Map Table, Context Programming, Queue Setup/Teardown)이 따로 있습니다. 이번 글에서는 개념만 다루고, 설정 절차는 실제 사용 경험과 함께 다시 정리하겠습니다.

---

# 한눈에 정리

```
소프트웨어(드라이버)                          하드웨어(QDMA)
 ① Descriptor를 링에 씀
 ② PIDX 레지스터 갱신  ────── MWr TLP ──────►  ③ QID로 컨텍스트 조회
                                               ④ Descriptor를 호스트에서 읽음 (MRd)
                                               ⑤ 데이터 전송 (H2C: MRd / C2H: MWr)
 ⑥ CIDX(Status) 또는 CMPT로 완료 확인  ◄─────── ⑥ 완료 기록 (MWr)
 ⑦ 버퍼 회수
```

| 항목 | 내용 |
|------|------|
| 큐 구성 | H2C 링 + C2H 링 + CMPT 링 |
| 링 위치 | 호스트 메모리 |
| 진행 표시 | PIDX(SW가 갱신), CIDX(HW가 갱신) |
| 링 마지막 칸 | Status Descriptor 용 예약 → 사용 가능 = 크기 − 1 |
| H2C / C2H(MM) 완료 | CIDX를 Status Descriptor에 기록 |
| C2H Stream 완료 | CMPT 링 항목 |
| 모드 | Internal(엔진이 처리) / Bypass(사용자 로직이 처리) |

---

# What I Learned

- QDMA 큐는 **호스트 메모리의 링**이고, 소프트웨어가 Descriptor를 채우고(PIDX) 하드웨어가 처리한다(CIDX).
- 전송은 **PIDX 레지스터 쓰기 → 컨텍스트 조회 → Descriptor 읽기 → 데이터 전송 → 완료 기록** 순서다.
- Descriptor 자체를 읽는 것도 PCIe 읽기이고, PIDX 갱신도 PCIe 쓰기다.
- 링은 **한 칸을 항상 비워 둔다**(마지막 칸은 Status Descriptor).
- **C2H Stream은 CMPT 링**으로 패킷 단위 완료를 알린다.
- Internal 모드와 Bypass 모드의 차이는 **Descriptor를 누가 처리하는가**이다.

---

# Next

다음 글에서는 Ethernet CMAC에서 들어온 데이터가 QDMA를 거쳐 호스트까지 전달되는 **전체 경로**를 하나로 연결합니다. AXI4-Stream 데이터 경로, 데이터 폭, 클럭 도메인이 중심입니다.

> **PCIe / QDMA Study - 005: FPGA Ethernet과 QDMA 연결**

---

# References

- AMD PG302 – QDMA Subsystem for PCI Express (General Design of Queues, H2C and C2H Queues)
- AMD PG213 – UltraScale+ Devices Integrated Block for PCI Express
