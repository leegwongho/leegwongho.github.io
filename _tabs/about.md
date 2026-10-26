---
icon: fas fa-user
order: 1
---

# About

안녕하세요. FPGA Engineer입니다.

FPGA로 **고속 Ethernet / PCIe 데이터 경로**를 설계하고 디버깅하는 일을 합니다. 개발하면서 배운 내용을 나중에 다시 읽고 도움을 얻을 수 있도록, **일반 원칙과 확인 방법** 중심으로 기록합니다.

> 이 블로그에는 소속 회사의 정보(제품, 설계, 설정값, 측정 결과)를 싣지 않습니다. 개인 학습과 공개 자료를 기반으로 한 내용만 다룹니다.

## 시리즈

{% include series_list.html %}

{% assign portfolio_posts = site.categories['Portfolio'] %}
{% if portfolio_posts and portfolio_posts.size > 0 %}
정리한 내용을 한 번에 보려면 포트폴리오 글을 참고해 주세요.

{% for p in portfolio_posts %}
- [{{ p.title }}]({{ p.url | relative_url }})
{% endfor %}
{% endif %}

## 관심 분야

- FPGA / RTL 설계
- Ethernet (MAC, CMAC)
- PCIe, DMA (QDMA)
- AXI4-Stream
- 타이밍 클로저와 디버깅
- Linux 드라이버, DPDK 기초

## 사용한 도구

- Vivado, ILA
- Verilog
- Linux 도구(lspci, ethtool, DPDK testpmd)
- 네트워크 계측기(트래픽 발생/분석)
- Python (검증용 스크립트)
