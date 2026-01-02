---
layout: post
title: "ft_traceroute 환경 설정"
date: 2026-01-01 16:00:00 +0900
description: >
  과제 요구 사항을 충족하기 위한 환경 설정 과정
categories: [42, C]
tags: [C]
---

이 글은 ft_traceroute 과제를 구현하면서 **“왜 내 코드의 출력이 traceroute(리눅스)와 다르게 보일 수 있는가?”** 를 설명하고,
제약된 환경에서도 **로직이 정상 동작함을 어떻게 검증했는지**를 정리한 기록이다.

과제의 평가 환경은 `Linux kernel >= 4.0`이며, raw socket을 사용하려면 `sudo` 권한이 필요하다.
클러스터 PC에서는 권한/네트워크 제약이 있어 가상 머신(VM)에서 테스트를 진행했는데,
VM 네트워크 모드에 따라 traceroute의 동작(또는 관측 결과)이 크게 달라졌다.

결론적으로는 **NAT 모드에서 테스트를 지속**하되,
출력 결과가 `* * *`로 보이거나 1홉 만에 끝나는 경우에도
Wireshark/tshark로 패킷을 직접 확인해 “TTL 조작 기반 경로 추적 로직”이 정상임을 입증하는 방향을 선택했다.

## (Reference) traceroute의 동작 원리 요약

traceroute는 목적지까지 가는 경로를 “직접 알아내는 프로토콜”이 아니라,
**TTL(Time To Live)** 값을 1부터 점차 증가시키며 보낸 프로브(probe)에 대해
중간 라우터가 반환하는 **ICMP 에러 응답**을 관측해 홉(hop)을 추정하는 도구이다.

핵심 메커니즘은 다음과 같다.

- 송신 측은 TTL=1인 프로브를 전송한다.
- 첫 번째 라우터는 TTL을 1 감소시키고(→0), 패킷을 폐기한 뒤 ICMP Time Exceeded(Type 11, Code 0)를 송신 측으로 되돌린다.
- 송신 측은 ICMP 응답의 **source IP(outer IP header)** 를 “1번째 홉 주소”로 기록한다.
- TTL을 2, 3, ...으로 늘려 동일한 과정을 반복하면 각 TTL에 대응하는 홉 주소를 얻을 수 있다.

리눅스의 기본 traceroute는 전통적으로 다음 방식을 사용한다.

- **UDP 프로브**를 목적지의 특정 포트(예: 33434+)로 보낸다.
- 목적지에 도달하면(더 이상 TTL로 막히지 않으면) 목적지는 해당 포트가 열려있지 않은 경우가 많아
  **ICMP Destination Unreachable(Type 3, Code 3: Port Unreachable)** 로 응답한다.
- 이 응답을 “도달”의 신호로 사용해 traceroute를 종료한다.

> 참고: UDP 기반 traceroute에서 “어떤 프로브에 대한 ICMP 응답인지”를 매칭할 때는
> ICMP payload 내부에 포함된 **원본 UDP destination port**가 대표적인 식별자로 사용된다.

## 1) VM 네트워크 모드에 따른 동작 차이

이 섹션은 “같은 traceroute라도 VM 네트워크 구성에 따라 관측 결과가 달라질 수 있음”을 정리한다.
특히 NAT 환경에서는 중간 장치가 패킷을 변형/대리 처리할 수 있으므로, 화면 출력만으로 로직의 정상 여부를 판단하기 어렵다.

> 관련 레퍼런스: [NAT와 Bridge Adapter 정리]({% post_url 42Cursus/2026-01-01-NAT_Bridge %})

- NAT 모드와 '가로채기' 현상
  - 증상: `traceroute -I 8.8.8.8` 실행 시 TTL=1에서 바로 목적지에 도달한 것처럼 보이는 경우가 있었다.
  - 원인(개념): NAT(Network Address Translation)는 사설 네트워크의 트래픽을 대표 주소로 변환해 외부로 내보내는 방식이다.
    구현체(VirtualBox NAT 등)에 따라서는 ICMP 처리를 “엔드포인트 대리 응답”처럼 수행하거나,
    VM 밖(호스트/NAT 엔진)에서 프로토콜 처리가 끝나 VM 내부에서는 경로가 짧게 관측될 수 있다.
  - 결론: 실제 경로 추적 결과는 왜곡될 수 있으나, 외부 인터넷 연결(패킷 송수신) 자체는 가장 안정적이었다.

- 브릿지 모드(Bridged Adapter) 시도와 한계
  - 목적: NAT 간섭 없이 VM이 공유기로부터 직접 IP를 받아 “실제 경로”를 확인하고자 했다.
  - 개념: 브릿지 모드는 VM NIC를 호스트의 물리 NIC에 L2(이더넷) 레벨로 연결해,
    VM이 동일한 L2 세그먼트의 “독립 호스트”처럼 동작하도록 만든다.
  - 문제: 클러스터의 보안 정책(예: 미인가 MAC 차단, 802.1X, 포트 보안 등)로 인해
    VM 트래픽이 차단될 수 있으며, 그 결과 VM 인터페이스가 `NO-CARRIER` 또는 외부 통신 불가 상태가 될 수 있다.
  - 정리: 이 환경에서는 브릿지 모드를 안정적인 테스트 경로로 사용하기 어려웠다.

## 2) 패킷 분석으로 로직 검증하기 (tshark)

NAT 모드에서는 출력 결과가 `* * *`로 보이거나 1홉 만에 종료되는 등 “도구와 동일한 출력”을 얻기 어렵다.
이 경우에도 코드의 핵심 로직(= TTL을 조작해 경로를 추적하는 방식)이 정상인지 판단하기 위해,
`tshark`로 UDP/ICMP 패킷을 직접 캡처해 검증했다.

- 검증 명령어

```bash
sudo tshark -i enp0s3 -f "udp or icmp" -T fields \
-e ip.src -e ip.dst -e ip.ttl -e _ws.col.Info

# -f "udp or icmp": UDP 프로브와 그에 대한 ICMP 응답을 함께 캡처
# -e ip.ttl: 발신 패킷의 TTL이 의도대로 설정되었는지 확인(예: setsockopt(IP_TTL))
```

- 해석 포인트
  - 발신(UDP): TTL 값이 `1, 1, 1, 2, 2, 2, ...`처럼 `probe_per_hop`만큼 반복되며 증가하는지 확인한다.
  - 수신(ICMP): 외부 경로가 막혀 있더라도, 최소한 게이트웨이(예: `10.0.2.2`)에서 `Time-to-live exceeded (Type 11)`가 관측되는지 확인한다.

- (Reference) 캡처에서 확인할 체크리스트
  - **송신 TTL 반영**: 발신 패킷의 `ip.ttl`이 코드에서 설정한 값과 일치해야 한다.
  - **ICMP 타입 구분**
    - 중간 홉: `Time-to-live exceeded (Type 11)`
    - 목적지 도달(UDP 방식): `Destination unreachable (Type 3)` 중 `Port unreachable (Code 3)`
  - **프로브 매칭 근거**: ICMP payload에 포함된 원본 IP/UDP 헤더에서
    원본 UDP destination port(예: 33434+)를 읽어 “어느 TTL/프로브의 결과인지”를 매칭할 수 있어야 한다.
  - **관측 왜곡 가능성**: NAT/방화벽/보안 정책에 따라 ICMP 응답이 차단·변형될 수 있으므로,
    출력(`* * *`)만으로 실패를 단정하지 않고 캡처로 근거를 확보한다.

- 결론
  - 출력이 일반적인 환경에서의 리눅스 `traceroute`와 동일하지 않더라도,
    패킷 레벨에서 TTL이 의도대로 적용되고 ICMP 응답이 관측되면 traceroute의 핵심 로직은 정상 동작으로 판단할 수 있다.

## 3) 평가/디버깅을 위한 체크리스트

- 커널 버전
  - `Linux kernel >= 4.0` 조건을 만족하는 환경(예: Debian 최신)을 유지한다.

- 출력 형식
  - 홉 번호, IP 주소, RTT 표시 형식을 실제 `traceroute`와 일치시키면 비교가 쉽다.

- 권한/소켓
  - raw socket 사용이 필요하므로 `sudo` 실행을 전제로 한다.
  - UDP 프로브 송신과 ICMP 응답 수신 흐름은 캡처 로그로 함께 설명할 수 있다.
