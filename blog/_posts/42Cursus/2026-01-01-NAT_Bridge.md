---
layout: post
title: "NAT와 Bridge Adapter"
date: 2026-01-01 16:00:00 +0900
description: >
  NAT와 Bridge Adapter가 무엇인지 알아봅시다.
categories: [42, C, Network]
tags: [C, Network]
---

## NAT (Network Address Translation)

NAT란, 라우터가 패킷을 지나가게 하면서 IP 주소를 다른 주소로 바꿔주는 기술로,
IP 주소를 변환해서 프라이빗 네트워크 <-> 외부 네트워크(인터넷)을 중계하는 역할을 한다.

### OSI 계층 관점

- 위치와 역할

NAT는 네트워크 계층(L3)기능으로 분류되며, 라우터나 L3 스위치에서 동작한다.

IP 헤더의 출발지/목적지 주소를 다른 주소로 매핑하여, 사실 IP <-> 공인 IP 번역을 수행한다.

- 어떤 PDU/필드를 건드리나?

L3 PDU(패킷)의 IP 헤더 필드(src/dest IP)를 수정하고, PAT(Port Address Translation)까지 포함하면 L4 헤더의 포트 번호도 함께 매핑한다.

이 과정에서 연결 상태를 NAT 테이블로 유지하여, 외부에서 돌아오는 패킷을 다시 내부 호스트로 되돌릴 수 있게 한다.

```mermaid
sequenceDiagram
    autonumber
    participant H as Host (내부 192.168.0.10:5000)
    participant SW as L2 Switch
    participant R as NAT Router (공인 203.0.113.5)
    participant S as Server (8.8.8.8:53)

    %% 1. 호스트에서 패킷 생성 (L3/L4)
    note over H: IP src=192.168.0.10:5000<br/>IP dst=8.8.8.8:53

    H->>SW: L2 Frame<br/>srcMAC=H-MAC dstMAC=R-MAC<br/>IP src=192.168.0.10 dst=8.8.8.8
    note over SW: L2에서 MAC 기반 포워딩만 수행<br/>IP/포트는 변경 없음

    SW->>R: L2 Frame<br/>srcMAC=H-MAC dstMAC=R-MAC<br/>IP src=192.168.0.10 dst=8.8.8.8

    %% 2. NAT 수행 (L3/L4 변경)
    note over R: NAT 테이블에<br/>(192.168.0.10:5000) -> (203.0.113.5:40001)<br/>매핑 생성
    R->>S: L2 Frame<br/>srcMAC=R-MAC dstMAC=S-MAC<br/>IP src=203.0.113.5:40001 dst=8.8.8.8:53

    %% 3. 서버 응답
    S-->>R: L2 Frame<br/>srcMAC=S-MAC dstMAC=R-MAC<br/>IP src=8.8.8.8:53 dst=203.0.113.5:40001

    note over R: NAT 테이블을 보고<br/>(203.0.113.5:40001) -> (192.168.0.10:5000) 역변환
    R-->>SW: L2 Frame<br/>srcMAC=R-MAC dstMAC=H-MAC<br/>IP src=8.8.8.8:53 dst=192.168.0.10:5000

    SW-->>H: L2 Frame<br/>srcMAC=R-MAC dstMAC=H-MAC<br/>IP src=8.8.8.8:53 dst=192.168.0.10:5000
```

- NAT 전/후 패킷 헤더 비교

```mermaid
packet
title NAT 이전 IP 헤더
0-3: "Version"
4-7: "IHL"
8-15: "TOS"
16-31: "Total Length"
32-63: "Identification+Flags+Fragment"
64-71: "TTL"
72-79: "Protocol"
80-95: "Header Checksum"
96-127: "Src IP=192.168.0.10"
128-159: "Dst IP=8.8.8.8"
```

```mermaid
packet
title NAT 이후 IP 헤더
0-3: "Version"
4-7: "IHL"
8-15: "TOS"
16-31: "Total Length"
32-63: "Identification+Flags+Fragment"
64-71: "TTL"
72-79: "Protocol"
80-95: "Header Checksum (recalc)"
96-127: "Src IP=203.0.113.5"
128-159: "Dst IP=8.8.8.8"
```

### 가상 머신에서의 NAT

![alt text](/assets/img/blog/image-14.png)

가상머신에서 네트워크 설정을 NAT로 설정하면, VM이 호스트와는 다른 사설 서브넷에 있고, 호스트가 라우터+NAT 역할을 해서 외부 네트워크에 나간다. 

따라서 VM은 인터넷에 나갈 수 있지만, 외부에서 VM으로 직접 접속하기 위해서는 별도 포트포워딩이 필요하다.
