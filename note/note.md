# BADASS 디펜스 스크립트 (한국어)

> 평가 항목 순서 그대로 정리한 발표용 대본입니다.
> 각 항목은 **말할 내용 → 보여줄 명령어** 순서로 되어 있습니다.

---

## 0. 사전 준비 (평가 시작 전 체크리스트)

- [ ] 평가자 스테이션에서 `git clone git@github.com:PfClaKr/Ecole42Paris_BADASS.git`
- [ ] 저장소 루트에 `P1/`, `P2/`, `P3/` 폴더가 있는지 함께 확인 (`ls`)
- [ ] Docker 이미지 2개 빌드 확인: `docker images | grep ychun`
- [ ] GNS3 실행, Docker 템플릿 등록 상태 확인
- [ ] Wireshark 캡처가 되는지 미리 1회 테스트 (패킷 인스펙션 필수 항목)

### 파일 ↔ 노드 매핑 (평가자가 "이 설정 파일은 뭐냐"고 물을 때 바로 답할 것)

| 파일 | 용도 |
|---|---|
| `P1/_ychun-2` | 라우터 이미지 Dockerfile (FRR 기반) → `routeur_ychun:latest` |
| `P1/_ychun-1_host` | 호스트 이미지 Dockerfile (Alpine/BusyBox) → `host_ychun:latest` |
| `P1/daemons` | FRR 데몬 활성화 설정 (bgpd, ospfd, isisd = yes) |
| `P2/_ychun-1_s`, `P2/_ychun-2_s` | Part2 **static** (유니캐스트 VXLAN) 라우터 설정 |
| `P2/_ychun-1_g`, `P2/_ychun-2_g` | Part2 **group** (멀티캐스트 VXLAN) 라우터 설정 |
| `P2/_ychun-1_host`, `P2/_ychun-2_host` | Part2 호스트 IP 설정 (30.1.1.1/24, 30.1.1.2/24) |
| `P3/_ychun-1` | Spine = Route Reflector 설정 |
| `P3/_ychun-2`, `P3/_ychun-3`, `P3/_ychun-4` | Leaf = VTEP 설정 |
| `P3/_ychun-N_host` | Part3 호스트 IP 설정 (20.1.1.N/24) |

---

## 1. 프로젝트 개요 (Project overview)

### 1-1. GNS3의 기본 동작

> "GNS3는 네트워크 토폴로지를 실제 소프트웨어로 에뮬레이션하는 도구입니다.
> 시뮬레이터처럼 '흉내'만 내는 게 아니라, 진짜 라우팅 소프트웨어(FRR)와 진짜 리눅스 커널이
> 각 노드 안에서 돌아갑니다.
>
> 구조는 GUI와 서버로 나뉩니다. GUI는 그림을 그리는 역할이고, 실제 노드를 띄우는 건
> gns3-server 입니다. 우리 프로젝트에서는 노드가 전부 **Docker 컨테이너**이기 때문에,
> gns3-server가 도커 데몬에게 컨테이너 생성을 요청하고,
> 노드 사이의 케이블은 **ubridge**라는 컴포넌트가 각 컨테이너의 네트워크 네임스페이스에
> veth 페어를 만들어 연결해 줍니다.
>
> 그래서 GNS3가 도커 그룹 권한이 없으면 노드를 못 띄우고,
> 링크를 우클릭하면 그 veth 위에서 바로 Wireshark 캡처가 가능합니다."

### 1-2. BGP의 전반적 동작과 존재 이유

> "BGP는 **Path Vector 프로토콜**이고, AS(Autonomous System) 단위로 경로 정보를 교환합니다.
> 인터넷 자체가 BGP로 굴러갑니다.
>
> - OSPF 같은 IGP는 'AS 내부'에서 최단 경로를 계산합니다. 링크 상태를 전부 공유하고 SPF를 돌립니다.
> - BGP는 'AS 사이'에서 **정책**으로 경로를 고릅니다. 최단 거리가 목적이 아니라,
>   '어느 이웃을 통해 보낼 것인가'를 관리자가 통제하는 게 목적입니다.
>
> 동작은 이렇습니다: TCP 179번 포트로 **명시적으로 지정한 neighbor**와 세션을 맺고
> (OSPF처럼 자동 발견하지 않습니다), OPEN → KEEPALIVE로 세션을 유지하며,
> 경로가 바뀔 때만 UPDATE를 보냅니다. 즉 **증분(incremental) 갱신**입니다.
>
> 존재 이유는 세 가지입니다:
> 1. 수십만 개 경로를 감당하는 확장성 (IGP는 이 규모에서 죽습니다)
> 2. AS_PATH 기반 루프 방지
> 3. 정책 제어 (Local Preference, MED, communities)
>
> 그리고 이 프로젝트에서 중요한 점 — BGP는 IPv4 경로만 나르는 게 아니라
> **MP-BGP** 확장으로 임의의 주소 체계를 실을 수 있습니다.
> Part 3에서 우리가 쓰는 `l2vpn evpn` address-family가 바로 그것이고,
> BGP로 **MAC 주소**를 광고합니다."

### 1-3. L2와 L3의 차이

> "**L2(데이터 링크 계층)** 는 같은 브로드캐스트 도메인 안에서 **MAC 주소**로 프레임을 전달합니다.
> 스위치가 여기 속하고, 목적지를 모르면 flooding하고, 응답을 보고 MAC 테이블을 학습합니다.
> 주소가 평평(flat)해서 요약이 불가능하고, TTL이 없어서 루프가 생기면 무한 순환합니다
> (그래서 STP가 필요합니다).
>
> **L3(네트워크 계층)** 는 **IP 주소**로 서로 다른 네트워크 사이를 라우팅합니다.
> IP는 계층적이라 요약이 되고, TTL이 있어서 루프가 생겨도 패킷이 소멸합니다.
> 브로드캐스트 도메인을 나누는 경계가 바로 L3입니다.
>
> 이 프로젝트의 핵심은 **'L3 네트워크 위에 L2를 얹는 것'** 입니다.
> 물리적으로는 라우팅된 L3 망(OSPF)인데, 호스트들은 마치 같은 스위치에 꽂힌 것처럼
> 같은 L2 세그먼트(20.1.1.0/24)에 있다고 착각합니다. 그 다리 역할을 VXLAN이 합니다."

---

## 2. Part 1 - 이론

### 2-1. 패킷 라우팅 소프트웨어란

> "전용 하드웨어 라우터 대신, 범용 리눅스에서 라우팅 프로토콜을 돌려주는 소프트웨어입니다.
> 우리는 **FRRouting(FRR)** 을 씁니다. Quagga의 후속이고, Cumulus/NVIDIA, Netflix 등에서
> 실제 프로덕션에 쓰입니다.
>
> 구조가 중요한데, FRR은 **하나의 프로그램이 아니라 데몬 묶음**입니다.
> 프로토콜별로 프로세스가 분리되어 있고(bgpd, ospfd, isisd...),
> 각자 계산한 최적 경로를 **zebra**에게 넘기면, zebra가 그것들을 종합해서
> 실제 **리눅스 커널의 라우팅 테이블**에 넣습니다.
> 즉 실제 패킷 포워딩은 커널이 하고, FRR은 '어디로 보낼지'를 결정하는 두뇌입니다.
> 관리 인터페이스는 `vtysh`이고, Cisco IOS와 거의 같은 문법을 씁니다."

### 2-2. bgpd

> "FRR에서 BGP를 담당하는 데몬입니다. TCP 179 포트로 neighbor와 세션을 맺고,
> BGP 테이블(Adj-RIB-In / Loc-RIB / Adj-RIB-Out)을 관리하며 best path를 선출해서 zebra에 넘깁니다.
> 우리 프로젝트에서는 Part 3에서 `address-family l2vpn evpn`을 활성화해서
> IP 경로가 아니라 MAC/IP 정보를 나르는 용도로 씁니다."

### 2-3. ospfd

> "OSPF를 담당하는 데몬입니다. **Link-State** 프로토콜이라,
> Hello 패킷(224.0.0.5 멀티캐스트)으로 이웃을 자동 발견하고,
> LSA를 area 내에 flooding해서 모든 라우터가 **동일한 토폴로지 DB**를 갖게 한 뒤,
> 각자 Dijkstra(SPF)를 돌려 최단 경로를 계산합니다. 메트릭은 대역폭 기반 cost입니다.
>
> Part 3에서 OSPF의 역할은 딱 하나입니다: **언더레이(underlay) 도달성**.
> 각 라우터의 루프백(1.1.1.x/32)끼리 서로 통신 가능해야 iBGP 세션이 맺어지는데,
> 그 루프백 경로를 배포해 주는 게 OSPF입니다."

### 2-4. 라우팅 엔진 (routing engine)

> "**zebra**입니다. FRR의 커널이자 중재자 역할입니다.
> - 각 프로토콜 데몬(bgpd/ospfd/isisd)에게서 경로를 받습니다.
> - 같은 목적지에 여러 프로토콜이 경로를 주면 **AD(Administrative Distance)** 로 우선순위를 매깁니다.
>   (connected 0 < static 1 < eBGP 20 < OSPF 110 < iBGP 200)
> - 선택된 경로만 실제 커널 FIB(`ip route`)에 설치합니다.
> - 반대로 인터페이스 up/down, IP 변경 같은 커널 이벤트를 각 데몬에게 알려 줍니다.
> - EVPN에서는 추가로 리눅스 브리지의 **MAC 학습 정보를 bgpd에게 전달**하는 역할도 합니다.
>   Part 3에서 호스트를 켰을 때 Type-2 경로가 생기는 게 바로 zebra 덕분입니다.
>
> 그래서 `daemons` 파일에도 "watchfrr, zebra, staticd는 항상 켜진다"고 적혀 있습니다."

### 2-5. BusyBox

> "리눅스 기본 유틸리티(ls, cp, ip, ping, sh 등) 수백 개를 **단일 실행 파일 하나**로 합친 것입니다.
> 심볼릭 링크로 호출되는 이름(argv[0])을 보고 어떤 기능을 실행할지 정합니다.
> '임베디드 리눅스의 맥가이버 칼'이라고 불리고, 크기가 몇 MB 수준이라
> 라우터/IoT/컨테이너처럼 용량이 중요한 환경에 쓰입니다.
>
> 우리 호스트 이미지는 `alpine:latest` 기반인데, **Alpine의 유저랜드가 곧 BusyBox**입니다.
> 그래서 이미지가 5MB 정도밖에 안 되고, 호스트를 여러 대 띄워도 부담이 없습니다.
> 컨테이너 안에서 `busybox` 를 치면 지원 명령 목록이 나오고,
> `ls -l /bin/ls` 를 보면 busybox로 심링크된 걸 확인할 수 있습니다."

**보여줄 것**
```sh
docker run --rm host_ychun:latest ls -l /bin/ls
docker run --rm host_ychun:latest busybox | head -20
```

---

## 3. Part 1 - 실습

### 3-1. Docker 이미지 구성 설명

**라우터 이미지** — `P1/_ychun-2`

```dockerfile
FROM  frrouting/frr:latest
ENV DAEMONS="zebra bgpd ospfd isisd"
COPY daemons /etc/frr/daemons
RUN chown frr:frr /etc/frr/daemons
CMD ["bash", "-c", "/usr/lib/frr/docker-start & exec bash"]
```

> "공식 FRR 이미지를 베이스로 씁니다.
> - `DAEMONS` 환경변수와 `daemons` 파일로 **어떤 라우팅 데몬을 켤지** 지정합니다.
>   과목에서 요구한 bgpd, ospfd, isisd를 yes로 켰고 zebra는 항상 켜집니다.
> - `chown frr:frr` 이 중요한데, FRR은 설정 파일 소유자가 frr이 아니면 데몬을 시작하지 않습니다.
>   `daemons` 파일 주석에도 그 경고가 그대로 적혀 있습니다.
> - `CMD`는 FRR 기동 스크립트를 백그라운드로 돌리면서 동시에 bash 셸을 PID 1로 유지합니다.
>   이렇게 해야 GNS3에서 콘솔로 붙었을 때 대화형 셸을 쓸 수 있습니다."

**호스트 이미지** — `P1/_ychun-1_host`

```dockerfile
FROM alpine:latest
RUN apk add vim iperf socat tcpdump
```

> "BusyBox 기반의 초경량 호스트입니다. 라우팅 기능은 없고 트래픽을 만들고 검증하는 용도입니다.
> tcpdump는 패킷 확인, iperf/socat은 트래픽 생성, vim은 설정 편집용입니다."

**빌드 명령**
```sh
cd P1
docker build -t routeur_ychun:latest -f _ychun-2 .
docker build -t host_ychun:latest -f _ychun-1_host .
docker images | grep ychun
```

> "`daemons` 파일을 COPY하기 때문에 반드시 P1 디렉토리를 빌드 컨텍스트로 잡아야 합니다."

### 3-2. GNS3에 Docker 이미지 임포트하는 방법

> "GNS3는 로컬 도커 데몬을 직접 조회합니다. 별도 import/export 과정이 없습니다."

1. `Edit → Preferences → Docker → Docker containers → New`
2. `Existing image` 선택 → 드롭다운에서 `routeur_ychun:latest` 선택
3. 이름 지정 → **Adapters 개수 지정** (라우터 3개, 호스트 2개 — P3에서 3포트까지 씁니다)
4. Console type: **telnet**
5. 호스트 이미지도 동일하게 반복

> "권한 관련해서 — GNS3 프로세스가 도커 소켓에 접근해야 하므로
> `sudo usermod -aG docker $USER` 후 재로그인이 필수입니다. README에도 적어 두었습니다."

```sh
groups     # docker, ubridge 가 보여야 함
```

### 3-3. 임포트한 머신 실행 / 접속

> "노드를 캔버스로 드래그해서 배치하고, 툴바의 ▶(Start all) 또는 우클릭 → Start 로 켭니다.
> 초록불이 켜지면 컨테이너가 떠 있는 겁니다."

- 접속: 노드 **더블클릭** 또는 우클릭 → **Console** (telnet으로 컨테이너 셸에 붙음)
- 호스트에서 확인
- 밖에서 검증: `docker ps` 로 GNS3가 띄운 컨테이너가 실제로 보임

### 3-4. 요구된 서비스가 살아 있는지 증명

라우터 콘솔에서:

```sh
ps aux | grep -E 'zebra|bgpd|ospfd|isisd'
vtysh -c "show daemons"
vtysh -c "show version"
```

vtysh 안에서:
```
show running-config
show ip route
```

> "zebra, bgpd, ospfd, isisd 네 개의 프로세스가 각각 떠 있는 것이 보입니다.
> `show daemons`는 vtysh가 실제로 통신에 성공한 데몬만 나열하므로,
> 여기 나온다는 건 유닉스 소켓 연결까지 정상이라는 뜻입니다."

---

## 4. Part 2 - 이론

### 4-1. VXLAN이란, VLAN과의 차이

> "VXLAN = Virtual eXtensible LAN. **L3 UDP 위에 L2 프레임을 캡슐화**하는 오버레이 터널입니다.
> 원본 이더넷 프레임 전체를 [Outer IP][UDP:4789][VXLAN 헤더(VNI 24bit)] 로 감싸서 보냅니다.
>
> VLAN과의 차이는 네 가지입니다:
>
> | | VLAN | VXLAN |
> |---|---|---|
> | 식별자 | 12비트 → **4094개** | 24비트 → **약 1670만개** |
> | 동작 계층 | L2 태그 (802.1Q) | L3/UDP 캡슐화 |
> | 확장 범위 | 같은 L2 도메인 안 | **라우팅된 L3 망을 넘어서** |
> | 필요 조건 | 경로상 모든 스위치가 VLAN 인지 | 중간 장비는 그냥 IP만 라우팅하면 됨 |
>
> 핵심 장점: 중간 네트워크는 **VXLAN의 존재를 전혀 몰라도 됩니다.**
> 그냥 UDP 패킷으로 보이니까요. 그래서 데이터센터 사이,
> 심지어 인터넷을 건너서도 같은 L2 세그먼트를 확장할 수 있습니다.
> 클라우드에서 멀티테넌시를 구현하는 표준 방식이 이것입니다.
>
> 대가는 오버헤드 50바이트 — 그래서 언더레이 MTU를 키워야 합니다."

### 4-2. 이 과제에서의 switch

> "P2 토폴로지 가운데의 `Switch_ychun`은 GNS3 내장 **이더넷 스위치**입니다.
> 순수 L2 장비이고, 두 라우터의 eth0을 같은 브로드캐스트 도메인
> (언더레이 10.1.1.0/24)에 묶어 주는 역할만 합니다.
>
> 동작 원리: 프레임이 들어오면 출발지 MAC과 들어온 포트를 MAC 테이블에 학습하고,
> 목적지 MAC이 테이블에 있으면 해당 포트로만 보내고, 없으면 나머지 전 포트로 flooding 합니다.
> 브로드캐스트/멀티캐스트도 flooding 됩니다 — 이게 P2 group 파트에서
> 멀티캐스트 239.1.1.1이 상대편에 전달되는 이유입니다."

### 4-3. 이 과제에서의 bridge

> "`br0`는 **리눅스 커널 안의 소프트웨어 스위치**입니다. `ip link add br0 type bridge` 로 만듭니다.
> 기능은 물리 스위치와 동일합니다 — MAC 학습, 포워딩, flooding.
>
> 우리가 br0를 쓰는 이유가 이 프로젝트의 핵심입니다.
> br0에 두 개의 포트를 꽂습니다:
> - `eth1` → 호스트로 가는 실제 물리(가상) 인터페이스
> - `vxlan10` → VXLAN 터널 인터페이스
>
> 이 둘을 같은 브리지에 넣으면, **호스트에서 온 프레임이 자동으로 VXLAN 터널로 흘러갑니다.**
> 브리지 입장에서 vxlan10은 그냥 '다른 포트'일 뿐이거든요.
> 즉 브리지가 로컬 세그먼트와 오버레이 터널을 접합(stitching)하는 지점입니다.
>
> 스위치 vs 브리지 차이를 굳이 말하자면: 브리지는 개념/소프트웨어 구현,
> 스위치는 그걸 ASIC으로 구현한 전용 하드웨어 — 논리는 같습니다."

### 4-4. 브로드캐스트 vs 멀티캐스트

> "둘 다 1:N 전송이지만 수신자 범위가 다릅니다.
>
> - **브로드캐스트**: 세그먼트의 **모든** 장비에게. 목적지 MAC `ff:ff:ff:ff:ff:ff`,
>   IP는 255.255.255.255. 관심 없는 장비도 인터럽트를 받고 처리해야 하므로 낭비가 큽니다.
>   L3를 넘지 못합니다. ARP가 대표적입니다.
> - **멀티캐스트**: **가입한 그룹만**. IPv4는 224.0.0.0/4 (D클래스),
>   IGMP로 가입/탈퇴를 관리하고, MAC은 `01:00:5e`로 시작하는 주소로 매핑됩니다.
>   관심 있는 장비만 받으니 효율적이고, 라우터를 넘어갈 수 있습니다(PIM 필요).
>
> P2에서 이 차이가 그대로 드러납니다:
> static 방식은 상대 VTEP을 하나하나 적어서 **유니캐스트 복제**를 하고,
> group 방식은 239.1.1.1 그룹으로 한 번만 보내면 그룹 멤버가 다 받습니다.
> 참고로 BUM = Broadcast, Unknown-unicast, Multicast — VXLAN이 특별히 처리해야 하는 트래픽 3종입니다."

### 4-5. P2 토폴로지의 기대 동작

```
  host_ychun-1                                        host_ychun-2
   30.1.1.1/24                                         30.1.1.2/24
       │ eth1                                              eth1 │
       │                                                        │
 ┌─────┴──────┐                                        ┌────────┴───┐
 │ routeur-1  │  br0 = eth1 + vxlan10                  │ routeur-2  │
 │  eth0      │  10.1.1.1/24                           │   eth0     │  10.1.1.2/24
 └─────┬──────┘                                        └────────┬───┘
       │                    ┌───────────────┐                   │
       └────────────────────│ Switch_ychun  │───────────────────┘
                            └───────────────┘
                        언더레이: 10.1.1.0/24
                        오버레이: 30.1.1.0/24 (VNI 10)
```

> "두 호스트는 30.1.1.0/24 라는 **같은 서브넷**에 있고, 서로를 같은 L2에 있다고 인식합니다.
> 실제로는 그 사이에 라우터 2대와 스위치 1대, 그리고 VXLAN 터널이 있습니다.
> host1이 host2에 ping 하면 ARP 브로드캐스트가 br0로 들어가고 → vxlan10으로 복제되어
> 10.1.1.1 → 10.1.1.2 UDP 4789로 캡슐화되어 건너가고 → 반대편에서 디캡슐화되어
> host2에 도달합니다. 호스트는 이 모든 과정을 전혀 모릅니다."

---

## 5. Part 2 - 실습 (static)

### 5-1. 설정 파일 설명

`P2/_ychun-1_s` (routeur_ychun-1):
```sh
ip link add br0 type bridge                 # 소프트웨어 스위치 생성
ip link set dev br0 up
ip link add name vxlan10 type vxlan id 10 dev eth0 \
        remote 10.1.1.2 local 10.1.1.1 dstport 4789
ip link set dev vxlan10 up

ip addr add 10.1.1.1/24 dev eth0            # 언더레이 (VTEP 주소)
ip addr add 20.1.1.1/24 dev vxlan10

brctl addif br0 eth1                        # 호스트 쪽 포트를 브리지에
brctl addif br0 vxlan10                     # 터널 포트를 브리지에
```

> "한 줄씩 설명하면:
> - `id 10` → **VNI = 10** (과목 요구사항)
> - `dev eth0` → 캡슐화된 패킷이 나갈 언더레이 인터페이스
> - `local 10.1.1.1` / `remote 10.1.1.2` → **내 VTEP과 상대 VTEP을 직접 명시**.
>   이게 'static' 방식의 정의입니다. 상대가 늘어나면 여기에 계속 추가해야 합니다.
> - `dstport 4789` → VXLAN 표준 UDP 포트 (IANA 할당)
> - 마지막 두 줄이 접합 지점: 호스트 포트와 터널 포트를 같은 브리지에 넣습니다.
>
> `_ychun-2_s`는 local/remote와 IP만 서로 뒤바뀐 대칭 설정입니다."

> `_ychun-2_s`는 local/remote와 IP만 서로 뒤바뀐 대칭 설정입니다."

> ⚠️ **평가자가 `ip addr add 20.1.1.1/24 dev vxlan10` 을 물어볼 경우**
> 솔직하게 답하는 게 낫습니다: "이건 라우터 자신도 오버레이에 참여시키려고 넣었던 줄인데,
> 바로 다음 줄에서 vxlan10을 br0에 enslave 하기 때문에 실제로는 동작하지 않습니다.
> enslave된 인터페이스의 IP는 커널이 사용하지 않고 IP는 브리지(br0)에 붙어야 합니다.
> 호스트 간 통신은 30.1.1.0/24로 이루어지므로 이 줄이 없어도 결과는 동일합니다."
> 필요하면 `ip addr add 20.1.1.1/24 dev br0` 로 옮겨서 라우터에서도 오버레이 통신이
> 되는 걸 보여 줄 수 있습니다.


`P2/_ychun-1_host`, `_ychun-2_host`:
```sh
ip addr add 30.1.1.1/24 dev eth1      # host-1
ip addr add 30.1.1.2/24 dev eth1      # host-2
```

### 5-2. 프로젝트 임포트 & 실행

1. `File → Import portable project` → `P2/P2.gns3project` 선택
2. 프로젝트 이름/경로 확인 후 OK
3. 툴바 ▶ (Start all nodes) 또는 각 노드 우클릭 → Start

> "포터블 프로젝트에는 토폴로지, 링크, 각 노드의 이미지 참조가 들어 있습니다.
> 이미지 자체는 안 들어 있으므로 `routeur_ychun`, `host_ychun`이
> 로컬 도커에 미리 빌드되어 있어야 합니다."

### 5-3. 각 머신 설정

각 노드 콘솔에 붙어서 해당 설정 파일 내용을 붙여넣습니다.

```sh
# routeur_ychun-1 콘솔
(P2/_ychun-1_s 내용 붙여넣기)

# routeur_ychun-2 콘솔
(P2/_ychun-2_s 내용 붙여넣기)

# host_ychun-1 콘솔
ip addr add 30.1.1.1/24 dev eth1

# host_ychun-2 콘솔
ip addr add 30.1.1.2/24 dev eth1
```

검증:
```sh
ip -br addr            # 인터페이스 상태
ip -d link show vxlan10    # vxlan id 10, remote/local 확인
brctl show             # br0 에 eth1, vxlan10 두 개가 붙어 있는지
bridge fdb show dev vxlan10
```

### 5-4. Ping + 패킷 인스펙션 (필수)

```sh
# host_ychun-1 에서
ping 30.1.1.2
```

**패킷 캡처 방법**: GNS3 캔버스에서 `routeur_ychun-1 ↔ Switch_ychun` 링크를 우클릭 → **Start capture** → Wireshark 자동 실행

> "여기서 봐야 할 것은 **이중 헤더**입니다. Wireshark에서 패킷 하나를 펼치면:
>
> 1. **Outer Ethernet** — 라우터 간 MAC
> 2. **Outer IP** — 10.1.1.1 → 10.1.1.2 (VTEP 주소)
> 3. **UDP** — destination port **4789**
> 4. **VXLAN** — VNI = **10** ← 과목 요구사항, 여기서 확인시켜 드립니다
> 5. **Inner Ethernet** — 호스트 간 MAC
> 6. **Inner IP** — 30.1.1.1 → 30.1.1.2
> 7. **ICMP** — Echo request / reply
>
> 즉 원본 ping 패킷이 통째로 UDP 안에 들어가 있는 걸 직접 볼 수 있습니다.
> 처음 몇 패킷은 ARP인데, 그것도 마찬가지로 VXLAN에 감싸여 있습니다.
> Wireshark 필터: `vxlan` 또는 `udp.port == 4789`"

라우터 안에서 tcpdump로도 가능:
```sh
tcpdump -i eth0 -n udp port 4789 -vv
```

---

## 6. Part 2 - 실습 (group / 멀티캐스트)

### 6-1. 설정 파일 설명

`P2/_ychun-1_g`:
```sh
ip link add name vxlan10 type vxlan id 10 dev eth0 \
        group 239.1.1.1 dstport 4789
```

> "static 버전과 **딱 한 부분**만 다릅니다.
> `local ... remote ...` 가 사라지고 `group 239.1.1.1` 이 들어갔습니다.
> 239.0.0.0/8은 조직 내부용(administratively scoped) 멀티캐스트 대역입니다.
>
> 이제 이 VTEP은 상대 주소를 모릅니다. 대신 239.1.1.1 그룹에 가입하고,
> BUM 트래픽이 생기면 그 그룹 주소로 **딱 한 번** 보냅니다.
> 유니캐스트 응답을 받으면 그때 상대 VTEP 주소를 학습해서
> 이후 유니캐스트 트래픽은 직접 보냅니다(data-plane learning)."

나머지(br0, 호스트 IP)는 static과 동일합니다.

### 6-2. 실행 & 검증

```sh
# 라우터에서 설정 적용 후
ip -d link show vxlan10        # group 239.1.1.1 확인
# host-1 에서
ping 30.1.1.2
```

캡처에서 확인할 것:
```
# Wireshark 필터
ip.dst == 239.1.1.1
```

> "ARP 같은 BUM 트래픽이 목적지 IP **239.1.1.1**로 나가는 것이 보입니다.
> 목적지 MAC도 `01:00:5e:01:01:01` 형태로 멀티캐스트 매핑되어 있습니다.
> 그 안에는 VNI 10과 함께 원본 ARP 프레임이 들어 있습니다.
> 이후 학습이 끝나면 유니캐스트 10.1.1.1 → 10.1.1.2로 전환됩니다."

### 6-3. 두 방식의 차이와 멀티캐스트의 장점 (평가 질문)

> "**차이**
>
> | | static (유니캐스트) | group (멀티캐스트) |
> |---|---|---|
> | 상대 VTEP | 설정에 직접 명시 | 명시 안 함, 그룹만 지정 |
> | BUM 처리 | VTEP 개수만큼 **복제해서 각각 전송** (head-end replication) | **한 번만** 전송, 망이 복제 |
> | 설정 복잡도 | N개 VTEP → N-1개 remote 필요 | 모든 VTEP이 **동일한 설정** |
> | VTEP 추가 시 | **기존 장비 전부 수정** | 신규 장비만 설정 |
> | 요구 사항 | 없음 | 언더레이가 멀티캐스트 지원 (IGMP/PIM) |
>
> **장점**
> 1. **확장성** — VTEP 10대면 static은 각 노드에 9개 remote(총 90줄), 멀티캐스트는 전부 같은 한 줄
> 2. **대역폭 절약** — BUM 트래픽을 소스가 N번 복제하지 않음. 라우터 CPU 부담도 감소
> 3. **운영 편의** — 설정이 동일하니 자동화/템플릿화가 쉽고, VTEP 추가가 무중단
>
> **한계** — 언더레이 전체가 멀티캐스트를 지원해야 합니다(PIM 설계 필요).
> 퍼블릭 클라우드나 인터넷 구간에서는 대부분 멀티캐스트가 안 됩니다.
> 그래서 실무의 해답이 Part 3입니다: **컨트롤 플레인(BGP-EVPN)으로 문제를 해결**해서
> 멀티캐스트 없이도 확장성을 얻습니다."

---

## 7. Part 3 - 이론

### 7-1. BGP-EVPN이란

> "EVPN = Ethernet VPN. **MP-BGP에 L2 정보를 실어 나르는 컨트롤 플레인**입니다
> (RFC 7432, VXLAN 적용은 RFC 8365). BGP의 새로운 address-family `l2vpn evpn` 을 씁니다.
>
> Part 2의 근본적 문제는 **컨트롤 플레인이 없다는 것**입니다.
> VTEP이 서로를 모르고, MAC도 모릅니다. 그래서 모르면 일단 flooding하고
> 응답을 보고 배우는 방식(flood-and-learn)에 의존합니다. 이건 확장이 안 됩니다.
>
> EVPN은 이걸 뒤집습니다:
> - VTEP이 로컬에서 학습한 MAC/IP를 **BGP UPDATE로 미리 광고**합니다
> - 다른 VTEP은 트래픽이 오기 전에 이미 '그 MAC은 저쪽 VTEP에 있다'는 걸 압니다
> - 그래서 **flooding이 거의 필요 없어지고, 멀티캐스트 언더레이도 필요 없어집니다**
> - VXLAN 터널도 수동 설정 없이 EVPN이 알려준 VTEP 주소로 **자동 생성**됩니다
>
> 요약: 데이터 플레인은 여전히 VXLAN, 컨트롤 플레인만 BGP로 교체한 것입니다."

### 7-2. Route Reflection(경로 반사)의 원리

> "iBGP에는 **split-horizon 규칙**이 있습니다:
> *'iBGP 피어에게 배운 경로는 다른 iBGP 피어에게 전달하지 않는다.'*
> AS_PATH가 iBGP 안에서는 변하지 않아 루프를 감지할 수 없기 때문입니다.
>
> 그 결과 iBGP는 원래 **풀 메시**를 요구합니다. N대면 N(N-1)/2개 세션 —
> 100대면 4950개입니다. 관리 불가능하죠.
>
> **Route Reflector**는 이 규칙에 예외를 부여받은 라우터입니다.
> 클라이언트에게 배운 경로를 다른 클라이언트에게 **반사(reflect)** 해 줄 수 있습니다.
> 그러면 각 클라이언트는 RR 하나와만 세션을 맺으면 되고, 세션 수가 N-1개로 줄어듭니다.
>
> 루프 방지는 두 개의 새 속성으로 합니다:
> - **ORIGINATOR_ID** — 최초 광고자의 router-id. 자기 것이 돌아오면 무시
> - **CLUSTER_LIST** — 거쳐 온 RR 목록. 자기 cluster-id가 있으면 무시
>
> 우리 P3에서는 Spine(`_ychun-1`)이 RR이고, Leaf 3대가 클라이언트입니다.
> 중요한 점 — **RR은 VTEP이 아닙니다.** 데이터 트래픽은 RR을 통과하지 않고
> Leaf ↔ Leaf 사이에서 직접 VXLAN 터널로 흐릅니다.
> RR은 오직 컨트롤 플레인 정보만 중계합니다."

설정 근거 (`P3/_ychun-1`):
```
router bgp 1
    neighbor ibgp peer-group
    neighbor ibgp remote-as 1              ← 같은 AS 1 = iBGP
    neighbor ibgp update-source lo         ← 루프백으로 세션
    bgp listen range 1.1.1.0/29 peer-group ibgp   ← 동적 네이버
    address-family l2vpn evpn
        neighbor ibgp activate
        neighbor ibgp route-reflector-client       ← RR 선언
```

> "`bgp listen range`도 설명할 만합니다. 각 Leaf를 일일이 neighbor로 적는 대신,
> **1.1.1.0/29 대역에서 오는 접속을 동적으로 수락**합니다.
> Leaf를 추가해도 Spine 설정을 안 바꿔도 됩니다. peer-group으로 정책은 공유합니다.
> `update-source lo`는 물리 인터페이스가 하나 죽어도 세션이 유지되게 하기 위함입니다 —
> 그래서 루프백 도달성을 제공하는 OSPF가 필수입니다."

### 7-3. VTEP

> "VTEP = **VXLAN Tunnel EndPoint**. VXLAN 캡슐화/디캡슐화를 수행하는 지점입니다.
>
> - **인그레스 VTEP**: 로컬 호스트의 이더넷 프레임을 받아 VXLAN 헤더를 씌워 보냄
> - **이그레스 VTEP**: 받아서 헤더를 벗기고 로컬 호스트에게 원본 프레임을 전달
>
> VTEP은 **IP 주소로 식별**되고, 보통 **루프백 주소**를 씁니다.
> 물리 인터페이스에 종속되지 않아서 링크 하나가 죽어도 VTEP 정체성이 유지되기 때문입니다.
>
> P3에서는 Leaf 3대(`_ychun-2/3/4`)가 VTEP이고, 각각 1.1.1.2 / 1.1.1.3 / 1.1.1.4 입니다.
> Spine은 VTEP이 아닙니다 — 그냥 IP 라우팅과 BGP 반사만 합니다.
> EVPN Type-3 경로에 이 VTEP 주소가 실려서 서로에게 알려집니다."

### 7-4. VNI

> "VNI = **VXLAN Network Identifier**. VXLAN 헤더의 **24비트** 필드입니다.
>
> - VLAN ID(12비트, 4094개)에 대응하는 개념이지만 약 **1670만개**까지 가능합니다
> - 하나의 **테넌트/세그먼트**를 구분합니다. VNI가 다르면 완전히 격리됩니다
> - 같은 물리 인프라 위에서 수많은 논리 네트워크를 겹쳐 돌릴 수 있게 해 주는 핵심입니다
>
> 우리는 과목 요구대로 **VNI = 10** 을 씁니다.
> 디캡슐화할 때 VTEP은 VNI를 보고 '이 프레임을 어느 브리지로 보낼지' 결정합니다.
> 우리 설정에서는 VNI 10 ↔ br0 매핑입니다.
> FRR의 `advertise-all-vni`는 '로컬에 있는 모든 VNI를 EVPN으로 광고하라'는 뜻입니다."

### 7-5. Type 2 vs Type 3 경로

> "EVPN에는 여러 경로 타입(1~5)이 있고, 우리는 2번과 3번을 씁니다.
>
> **Type 3 — Inclusive Multicast Ethernet Tag (IMET)**
> - 의미: *'나는 이 VNI에 참여하는 VTEP이고, 내 주소는 X다'*
> - **VTEP이 VNI를 활성화하는 순간 즉시** 생성됩니다. 호스트가 하나도 없어도 존재합니다
> - 용도: **BUM 트래픽 복제 리스트**를 구성. 브로드캐스트를 누구에게 보내야 하는지 알려 줍니다
> - 결과: 멀티캐스트 언더레이 없이 **유니캐스트 복제 리스트**를 컨트롤 플레인으로 만듭니다
> - 담는 정보: VNI + 발신 VTEP IP (MAC 정보 없음)
>
> **Type 2 — MAC/IP Advertisement**
> - 의미: *'MAC 주소 Y (와 IP Z)는 내 뒤에 있다'*
> - **호스트가 실제로 트래픽을 보내서 MAC이 학습된 후에** 생성됩니다
> - 용도: 원격 VTEP이 MAC 테이블을 미리 채워서 flooding을 없앰. ARP suppression도 가능
> - 담는 정보: MAC, (선택적) IP, VNI, VTEP IP, MAC/IP용 RT
>
> **핵심 차이 한 줄 요약**
> - Type 3 = **VTEP의 존재**를 알림 → 인프라 정보, 호스트와 무관
> - Type 2 = **엔드포인트의 위치**를 알림 → 호스트가 생겨야 나타남
>
> 그리고 Type 2는 두 단계로 나타납니다:
> - 호스트가 IP 없이 트래픽만 보내면 → **MAC-only Type 2**
> - IP까지 설정되면 → **MAC+IP Type 2**
>
> 평가에서 이걸 단계별로 시연합니다."

### 7-6. P3 토폴로지의 기대 동작

```
                         ┌──────────────────────┐
                         │  _ychun-1 (Spine/RR) │
                         │      lo 1.1.1.1      │
                         │ eth0 10.1.1.1/30     │
                         │ eth1 10.1.1.5/30     │
                         │ eth2 10.1.1.9/30     │
                         └───┬──────┬───────┬───┘
              10.1.1.2/30    │      │       │   10.1.1.10/30
                     ┌───────┘      │       └────────┐
                     │       10.1.1.6/30             │
             ┌───────┴────┐  ┌──────┴─────┐  ┌───────┴────┐
             │ _ychun-2   │  │ _ychun-3   │  │ _ychun-4   │
             │ lo 1.1.1.2 │  │ lo 1.1.1.3 │  │ lo 1.1.1.4 │
             │  (Leaf/VTEP)│ │ (Leaf/VTEP)│  │ (Leaf/VTEP)│
             └───────┬────┘  └──────┬─────┘  └───────┬────┘
                 eth1│            eth0│           eth0│
              host-1 │           host-2│         host-3│
             20.1.1.1/24        20.1.1.2/24     20.1.1.3/24
                        오버레이 VNI 10 / 20.1.1.0/24
```

> "동작 순서로 설명하면:
> 1. **OSPF (area 0)** 가 언더레이 링크와 루프백(1.1.1.x/32) 경로를 배포 → Leaf가 Spine 루프백에 도달 가능
> 2. 그 루프백 위로 **iBGP EVPN 세션**이 Leaf ↔ Spine(RR) 사이에 맺어짐 (AS 1, 모두 동일 AS)
> 3. 각 Leaf가 `advertise-all-vni` 로 VNI 10 참여를 광고 → **Type-3 경로**가 RR을 통해 전 Leaf에 반사됨
> 4. 이 시점에 **VXLAN 터널이 자동 생성**됨. 우리 설정에 remote 주소가 한 줄도 없다는 게 증거입니다
> 5. 호스트가 트래픽을 보내면 Leaf의 br0가 MAC 학습 → zebra가 bgpd에 전달 → **Type-2 경로** 광고
> 6. 다른 Leaf들은 트래픽이 오기 전에 이미 MAC 위치를 알고 있으므로 flooding 없이 바로 유니캐스트 전송
>
> 데이터 트래픽은 Leaf → (Spine을 단순 IP 라우팅으로 통과) → Leaf 로 흐릅니다.
> Spine은 캡슐화된 UDP 패킷을 그냥 라우팅할 뿐, 안에 뭐가 들었는지 모릅니다."

---

## 8. Part 3 - 실습

### 8-1. 설정 파일 설명

**Spine/RR** — `P3/_ychun-1`
```
vtysh
conf t
interface eth0
    ip address 10.1.1.1/30
    ip ospf area 0
interface eth1
    ip address 10.1.1.5/30
    ip ospf area 0
interface eth2
    ip address 10.1.1.9/30
    ip ospf area 0
interface lo
    ip address 1.1.1.1/32
    ip ospf area 0
router bgp 1
    neighbor ibgp peer-group
    neighbor ibgp remote-as 1
    neighbor ibgp update-source lo
    bgp listen range 1.1.1.0/29 peer-group ibgp
    address-family l2vpn evpn
        neighbor ibgp activate
        neighbor ibgp route-reflector-client
        exit-address-family
router ospf
```

> "포인트 3가지:
> - **/30 서브넷** — 라우터 간 P2P 링크에는 주소 2개면 충분하므로 낭비를 없앱니다
> - **`ip ospf area 0`** — 인터페이스 단위로 OSPF를 켜는 최신 문법. 루프백도 포함해야
>   루프백 경로가 광고되고, 그래야 iBGP 세션이 성립합니다
> - **RR에는 `advertise-all-vni`가 없습니다** — Spine은 VTEP이 아니니까요. 의도된 설계입니다"

**Leaf/VTEP** — `P3/_ychun-2` (나머지도 IP만 다르고 동일 구조)
```sh
ip link add br0 type bridge
ip link set dev br0 up
ip link add vxlan10 type vxlan id 10 dstport 4789
ip link set dev vxlan10 up
brctl addif br0 vxlan10
brctl addif br0 eth1
```
```
vtysh
conf t
interface eth0
    ip address 10.1.1.2/30
    ip ospf area 0
interface lo
    ip address 1.1.1.2/32
    ip ospf area 0
router bgp 1
    neighbor 1.1.1.1 remote-as 1
    neighbor 1.1.1.1 update-source lo
    address-family l2vpn evpn
        neighbor 1.1.1.1 activate
        advertise-all-vni
        exit-address-family
router ospf
```

> "**Part 2와 비교해서 강조할 점**:
> VXLAN 인터페이스 생성 줄에 `remote`도 `group`도 **없습니다**.
> VNI와 포트만 있습니다. 상대 VTEP 정보는 전부 EVPN이 알려 주기 때문입니다.
> 이게 Part 3의 본질입니다.
>
> `advertise-all-vni`가 zebra에게 '로컬 VNI를 전부 EVPN으로 광고하라'고 지시하고,
> Leaf는 RR(1.1.1.1) 하나와만 세션을 맺습니다."

**호스트** — `P3/_ychun-N_host`
```sh
ip addr add 20.1.1.1/24 dev eth1     # host-1 (leaf-2 의 eth1 에 연결)
ip addr add 20.1.1.2/24 dev eth0     # host-2
ip addr add 20.1.1.3/24 dev eth0     # host-3
```

### 8-2. 임포트 & 실행 & 설정

1. `File → Import portable project` → `P3/P3.gns3project`
2. **호스트는 아직 켜지 않고**, 라우터 4대만 Start
3. 각 라우터 콘솔에 해당 설정 붙여넣기

언더레이 검증:
```sh
vtysh -c "show ip ospf neighbor"      # Leaf 3개가 Full 상태
vtysh -c "show ip route ospf"         # 1.1.1.x/32 루프백들이 보임
vtysh -c "show bgp l2vpn evpn summary"  # Leaf 3개 세션 Established
vtysh -c "show evpn vni"              # VNI 10, type L2, VTEP IP
vtysh -c "show evpn vni detail"
```

### 8-3. 모든 HOST 끄기 → Type 3만 존재하는지 확인

> "호스트를 전부 끕니다 (GNS3에서 각 host 노드 우클릭 → **Stop**, 또는 Ctrl 다중선택 후 Stop).
> 이 상태에서는 MAC을 학습할 대상이 없으므로 Type-2가 존재할 수 없습니다."

```sh
vtysh -c "show bgp l2vpn evpn"
vtysh -c "show bgp l2vpn evpn route type multicast"   # Type 3 → 존재
vtysh -c "show bgp l2vpn evpn route type macip"       # Type 2 → 비어 있음
```

> "출력에서 `[3]:[0]:[32]:[1.1.1.2]` 형태의 prefix가 보입니다.
> 앞의 `[3]`이 route type 3이고, 마지막이 광고한 VTEP의 IP입니다.
> Leaf 3대분이 다 보이는데, **호스트가 하나도 없는데도 존재합니다.**
> 이게 Type 3의 성격입니다 — VTEP의 존재 자체를 알리는 인프라 정보입니다.
>
> 그리고 여기서 이미 VXLAN 터널이 만들어져 있습니다:"

```sh
bridge fdb show dev vxlan10 | grep 00:00:00:00:00:00
# → 00:00:00:00:00:00 dst 1.1.1.3 / dst 1.1.1.4  ← BUM 복제 리스트
```

> "all-zero MAC 엔트리가 바로 **BUM 트래픽 복제 목적지 리스트**입니다.
> Part 2에서 멀티캐스트 그룹이 하던 일을 EVPN Type-3가 대신한 결과물입니다."

### 8-4. 호스트 1대만 켜기 (IP 없이) → 학습 과정 증명

> "이제 host_ychun-1 하나만 Start 합니다. **IP는 설정하지 않습니다.**"

```sh
# host-1 콘솔에서 (IP 없음)
ip link set eth1 up
ip link show eth1        # MAC 주소 확인 (메모)
ping -c 2 20.1.1.99      # 아무 트래픽이나 발생시켜 MAC 학습 유도 (선택)
```

```sh
# Leaf _ychun-2 에서
vtysh -c "show evpn mac vni 10"
vtysh -c "show bgp l2vpn evpn route type macip"
```

```sh
# 다른 Leaf (_ychun-3) 에서 — 원격에서 보이는지가 핵심
vtysh -c "show bgp l2vpn evpn"
vtysh -c "show evpn mac vni 10"
```

> "이제 **Type-2 경로가 나타났습니다**. prefix가 `[2]:[0]:[48]:[aa:bb:cc:...]` 형태입니다.
> 앞의 `[2]`가 route type 2, `[48]`은 MAC 길이 48비트를 뜻합니다.
> **IP 부분이 없습니다** — 호스트에 IP를 안 줬으니까요. MAC-only Type 2입니다.
>
> 여기서 증명되는 것:
> 1. Leaf-2의 br0가 로컬에서 MAC을 학습했고
> 2. zebra가 그걸 bgpd에 전달했고
> 3. bgpd가 RR에 광고했고
> 4. RR이 다른 Leaf들에게 반사했고
> 5. Leaf-3/4가 **한 번도 이 호스트의 트래픽을 본 적이 없는데도** 그 위치를 알고 있습니다
>
> 이것이 컨트롤 플레인 기반 학습입니다. Part 2의 flood-and-learn과 근본적으로 다릅니다."

### 8-5. 모든 호스트 켜고 IP 설정

```sh
# 각 호스트 Start 후
ip addr add 20.1.1.1/24 dev eth1      # host-1
ip addr add 20.1.1.2/24 dev eth0      # host-2
ip addr add 20.1.1.3/24 dev eth0      # host-3
```

```sh
# Leaf 에서 다시 확인
vtysh -c "show bgp l2vpn evpn route type macip"
vtysh -c "show evpn arp-cache vni 10"
```

> "이제 Type-2 prefix에 **MAC 뒤에 IP가 붙었습니다**: `[2]:[0]:[48]:[MAC]:[32]:[20.1.1.1]`.
> `[32]`는 IPv4 프리픽스 길이입니다.
> MAC-only에서 MAC+IP로 진화한 것을 직접 비교해서 보여 드릴 수 있습니다.
> 이 IP 정보 덕분에 ARP suppression(Leaf가 ARP에 대신 응답)도 가능해집니다."

### 8-6. Ping + 패킷 인스펙션

```sh
# host-1 에서
ping 20.1.1.2
ping 20.1.1.3
```

캡처 지점: `_ychun-2 ↔ _ychun-1` 링크 우클릭 → **Start capture**

> "확인할 것:
> - Outer IP: **1.1.1.2 → 1.1.1.3** ← 루프백끼리, 즉 **VTEP 간 직접 터널**입니다.
>   RR(1.1.1.1)을 목적지로 하지 않습니다. 컨트롤 플레인과 데이터 플레인이 분리된 증거입니다.
> - UDP dst port **4789**
> - VXLAN **VNI = 10**
> - Inner: 20.1.1.1 → 20.1.1.2 ICMP
>
> Part 2와 비교하면, 초기 ARP flooding이 거의 없습니다.
> MAC을 이미 알고 있어서 첫 패킷부터 유니캐스트로 갑니다."

Wireshark 필터:
```
vxlan.vni == 10
```

### 8-7. OSPF 패킷 확인

같은 캡처 창에서:
```
ospf
```

> "**Hello 패킷**이 목적지 **224.0.0.5** (AllSPFRouters 멀티캐스트)로 10초마다 나가는 것이 보입니다.
> IP 프로토콜 번호는 89입니다.
> 안에는 Router ID, Area ID (0.0.0.0), 이미 알고 있는 이웃 목록이 들어 있습니다.
>
> 초기 인접(adjacency) 형성 과정을 보여 주려면 인터페이스를 한 번 내렸다 올리면
> Hello → DB Description → LS Request → LS Update → LS Ack 순서를 그대로 볼 수 있습니다.
>
> 그리고 BGP도 같이 보여 드릴 수 있습니다 (필터 `bgp`).
> TCP 179 포트로 UPDATE가 오가고, 그 안에 MP_REACH_NLRI 속성으로
> EVPN 경로가 실려 있는 것을 확인할 수 있습니다."

인접 재형성 시연 (선택):
```sh
vtysh -c "clear ip ospf process"
vtysh -c "show ip ospf neighbor"
```

---

## 9. 예상 추가 질문 대비

**Q. 왜 루프백을 BGP update-source로 쓰나?**
> 물리 링크 하나가 죽어도 다른 경로로 세션이 유지됩니다. 인터페이스 IP를 쓰면 그 링크가 죽는 순간 세션도 죽습니다. VTEP 식별자로도 같은 이유로 루프백을 씁니다.

**Q. Spine이 죽으면 데이터 트래픽도 끊기나?**
> 컨트롤 플레인은 끊깁니다(새 MAC 학습 불가). 하지만 P3 토폴로지에서는 Spine이 물리 경로이기도 해서 데이터도 끊깁니다. 실무에서는 Spine을 2대 이상 두고 RR도 이중화합니다.

**Q. eBGP가 아니라 왜 iBGP인가?**
> 데이터센터 EVPN 설계에는 두 가지 유파가 있습니다. iBGP+RR 방식(우리 방식)은 AS 번호가 하나라 관리가 단순하고 RR로 세션을 줄입니다. eBGP 방식(AS per leaf)은 RR이 필요 없고 AS_PATH로 루프를 막습니다. 우리는 과목 요구에 맞춰 RR을 시연하기 위해 iBGP를 택했습니다.

**Q. MTU 문제는?**
> VXLAN이 50바이트를 추가하므로 언더레이 MTU를 1550 이상으로 올리거나 오버레이 MTU를 1450으로 낮춰야 합니다. GNS3 가상 환경에서는 큰 패킷을 안 보내서 드러나지 않지만, 실무에서는 반드시 처리해야 할 부분입니다. `ping -s 1500` 으로 재현 가능합니다.

**Q. VXLAN이 UDP를 쓰는 이유는?**
> ① TCP처럼 재전송/순서보장을 하면 이중 재전송 문제가 생깁니다. 터널은 투명해야 합니다.
> ② UDP source port에 **inner 헤더의 해시**를 넣어서, 중간 라우터가 ECMP로 부하분산할 때 플로우가 골고루 퍼지게 합니다. 이게 큰 장점입니다.

**Q. Part 2와 Part 3, 실무에서는 뭘 쓰나?**
> Part 3(EVPN)입니다. Part 2 static은 소규모/PoC, group은 멀티캐스트가 가능한 사내망에서만 씁니다. 퍼블릭 클라우드나 대규모 DC는 전부 EVPN입니다.

---

## 10. 자주 생기는 문제와 대응

| 증상 | 원인 / 해결 |
|---|---|
| GNS3에서 노드 시작 시 permission denied | `sudo usermod -aG docker $USER` 후 재로그인, `groups`로 확인 |
| `brctl: not found` | FRR 이미지가 Alpine 기반. `apk add bridge-utils` 또는 `ip link set dev X master br0` 사용 |
| OSPF 이웃이 안 붙음 | `/30` 서브넷 양쪽이 맞는지, `router ospf` 가 선언됐는지, 인터페이스 up 상태인지 |
| BGP 세션 Active/Idle에서 멈춤 | 루프백 도달성 확인 (`ping -I lo 1.1.1.1`). OSPF가 루프백을 광고하는지 확인 |
| `show evpn vni` 가 비어 있음 | `advertise-all-vni` 누락, 또는 vxlan 인터페이스가 br0에 안 붙어 있음 |
| Type-2가 안 생김 | vxlan 디바이스에 로컬 VTEP IP 명시 필요할 수 있음: `ip link add vxlan10 type vxlan id 10 dstport 4789 local 1.1.1.2 nolearning` |
| 호스트끼리 ping 실패 | `brctl show` 로 br0에 두 포트가 다 붙었는지, 인터페이스가 전부 up인지 확인 |

---

## 11. 마지막 정리 한 문장

> "Part 1에서 라우팅 소프트웨어를 컨테이너로 만들었고,
> Part 2에서 VXLAN으로 L3 위에 L2 오버레이를 수동으로 만들었으며,
> Part 3에서 BGP-EVPN이라는 컨트롤 플레인을 도입해서
> 그 오버레이를 **자동으로, 확장 가능하게** 만들었습니다.
> 이것이 현대 데이터센터 네트워크(Spine-Leaf + EVPN/VXLAN)의 표준 구조입니다."
