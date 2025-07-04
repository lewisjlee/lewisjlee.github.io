---
layout: single
title:  "TCP 재전송과 Timeout"
categories: Linux
tag: [Linux, Network]
author_profile: false
sidebar:
    nav: "docs"
---

TCP는 OSI 7 Layer 중 전송 계층(L4)의 프로토콜로, 데이터의 신뢰성과 순서를 보장하는 연결 지향형 프로토콜입니다. 단순히 데이터를 보내는 것 뿐만 아니라 수신 여부를 확인하고 누락되거나 손상된 데이터는 재전송하며, 연결이 안전하게 유지될 수 있도록 합니다. 그렇다면 만약 TCP 통신에서 전송한 특정 패킷에 대해 상대가 응답하지 않는다면 어떤 일이 일어날까요? TCP 통신에서 최초의 연결을 맺기 위해 수행하는 TCP 3-way handshake와 이후 데이터를 주고받는 과정에서 발생할 수 있는 재전송과 Timeout에 대해 알아보도록 하겠습니다.

## TCP 3-way handshake와 데이터 통신

<img title="" src="../../images/2025-06-25-tcp_retries/bb58f2325c98b32e1129c59d9c76ef1cdefefb87.png" alt="loading-ag-487" data-align="center">

TCP 세션을 맺기 위해 가장 먼저 수행하는 것은 3-way handshake입니다. 3번의 단방향 패킷 통신을 하기 때문에 3-way라고 이름을 붙이는데요. 다음과 같은 순서로 클라이언트와 서버 간의 3-way handshake가 수행됩니다.

#### 1) SYN

먼저 클라이언트는 서버에 연결하기 위해 SYN 세그먼트를 전송한다. 이 때 순서를 보장하기 위해 전송되는 세그먼트에는 **Sequence Number**가 포함된다. 소켓은 **SYN-SENT** 상태가 된다.

#### 2) SYN + ACK

SYN를 수신한 서버는 SYN와 ACK 컨트롤 플래그를 포함하는 세그먼트를 전송한다. 전송되는 세그먼트에는 **(클라이언트로부터 수신한 Sequence Number + 1)의 값인 Acknowledgement Number와 서버의 Sequence Number**가 포함된다. 이 때 소켓은 **SYN-RECV** 상태가 된다.

#### 3) ACK

서버로부터 SYN+ACK 세그먼트를 수신한 클라이언트는 정상적으로 수신받았다는 의미로 ACK 세그먼트를 전송한다. 전송되는 세그먼트에는 **(서버로부터 전달받은 Sequence Number + 1)의 값인 Acknowledgement Number**가 포함된다. 비로소 3-way handshake를 완료하고 세션이 연결되어 각 소켓은 **ESTABLISHED** 상태가 된다.

이렇게 3-way handshake를 통해 세션이 만들어지면 Active-closer가 먼저 연결을 종료할 때까지 데이터를 주고받을 수 있게 됩니다.

## TCP 재전송

만약 여기서 클라이언트가 전송한 SYN 세그먼트나 요청을 서버가 정상적으로 수신하지 못한다면 다시 전송하여 세션을 무사히 연결할 수 있도록 해야 합니다. 그래서 **클라이언트는 보낸 SYN나 요청에 대해 ACK를 받지 못한다면 패킷을 다시 보내게 되는데요.** <span style="color:red">이미 보냈던 패킷을 다시 보내기 때문에 네트워크 성능 저하가 발생할 수 있지만</span> TCP는 신뢰성을 보장하는 연결 지향형 프로토콜이기 때문에 이러한 재전송은 꼭 필요한 단계입니다.

### RTO와 RTT

그렇다면 클라이언트는 전송한 패킷에 대한 ACK를 받기 위해 얼마나 기다려야 할까요? 리눅스 커널은 이를 결정하기 위해 RTO와 RTT라는 기능을 갖고 있는데요. RTO와 RTT는 각각 다음을 의미합니다.

- **RTO(Retransmission Timeout)** : ACK 세그먼트를 기다리는 시간, 이 시간동안 ACK가 오지 않으면 재전송

- **RTT(RoundTripTime)** : 두 종단(클라이언트 및 서버) 간 패킷 전송과 ACK 응답까지 소요되는 시간

**RTT는 재전송되지 않은 패킷에 대해서만 측정하며, 패킷 송신 시점을 저장해 두었다가 ACK가 올 때까지의 시간을 측정하여 표본(SampleRTT)으로 사용합니다.** 그래서 RTO는 RTT를 기반으로 동적으로 계산되는데요. 커널 로직에 따라 다음과 같이 계산됩니다.

```
1. 초기 측정
RTO = SRTT + max(G, 4 * RTTVAR)

RTTVAR = SampleRTT / 2
SRTT = SampleRTT
G : clock granularity (Linux에서는 일반적으로 1ms)
```

```
2. 이후 다음과 같이 측정
RTO = SRTT + max(G, 4 * RTTVAR)

RTTVAR = (1 - β) * RTTVAR + β * |SRTT - SampleRTT|
SRTT = (1 - α) * SRTT + α * SampleRTT
α = 1/8 (0.125), β = 1/4 (0.25)
```

하지만 무조건 이러한 방식으로 RTO가 계산되어 적용되면 데이터 통신이 조금이라도 지연될 경우 패킷 재전송이 더 많이 발생하여 TCP 통신 성능이 저하될 수 있습니다. 그래서 커널은 이러한 RTO가 너무 작아지지 않도록 **TCP_RTO_MIN**이라는 값을 통해 하한선을 두는데요. 이 값은 기본적으로 **200ms**로 설정되어 있습니다.

리눅스 환경에서 RTO와 RTT를 직접 확인해 보겠습니다. 퍼블릭 서브넷에 AWS EC2 한 대를 생성하고 nginx를 설치한 다음 로컬 VM에서 telnet을 통해 http로 접속하여 TCP 통신을 수행해 보겠습니다.

아래 명령어를 사용하여 연결된 세션의 RTO와 RTT 값을 확인할 수 있습니다.

```bash
ss -i
```

<img title="" src="../../images/2025-06-25-tcp_retries/bf3bb41743333dc5070247fa6043683e1c28d969.png" alt="loading-ag-598" data-align="center">

<img title="" src="../../images/2025-06-25-tcp_retries/54955f9460780581d7f5a78ebefa7793ebd5a740.png" alt="loading-ag-607" data-align="center">

여기 붉은색 밑줄에서 rtt 필드의 값은 SRTT/RTTVAR를 의미합니다. 이 값들을 초기 RTO 계산식에 대입하면 TCP_RTO_MIN인 200ms보다 훨씬 낮은 결과값이 나오기 때문에 자동으로 200ms로 치환됩니다.

그러나 이미지에서 200ms보다 조금 높은 값이 표시되는 이유는 <u>ms 단위의 RTO 값을 커널 내부적으로 시간 경과를 측정하는 단위인 jiffies로 변환하고 올림 처리한 다음 다시 ms 단위로 환산하기 때문입니다.</u>

```
GET / HTTP/1.1
Host: [서버 주소]
```

<img title="" src="../../images/2025-06-25-tcp_retries/f63bb7549d31ee6ef28133516c2bb18f25e86b10.png" alt="loading-ag-697" data-align="center">

<img title="" src="../../images/2025-06-25-tcp_retries/403989ed82233a11e91cddd68249e4dd1e0aa9a1.png" alt="loading-ag-699" data-align="center">

세션 연결 후 GET 요청을 하면 초기 측정 때와 다른 계산식으로 RTO와 RTT 값이 다시 측정 및 계산된 것을 확인할 수 있습니다.

```bash
tcpdump -A -vvv -nn port 80 -w tcp_client.pcap
```

```bash
tcpdump -A -vvv -nn port 80 -w tcp_server.pcap
```

<img title="" src="../../images/2025-06-25-tcp_retries/4b435b6c7b432147878c2ad200c4b1e0cb1ddc9c.png" alt="loading-ag-800" data-align="center">

<img title="" src="../../images/2025-06-25-tcp_retries/f46ceb470630e6739ef84ed126cb8b1b742985d9.png" alt="loading-ag-802" data-align="center">

세션을 연결하고 GET 요청 및 응답까지의 패킷을 dump하고 와이어샤크로 열어봤습니다. SYN -> SYN + ACK -> ACK의 흐름을 통해 정상적으로 3-way handshake를 수행한 다음 요청과 응답을 주고 받은 것을 확인할 수 있고, Time 컬럼에서 확인할 수 있는 각 과정별 시간 경과도 앞서 확인한 RTT 측정값과 비슷하게 나온 것을 확인할 수 있습니다.

### InitRTO

TCP 세션 연결 이후 RTO 계산에 대해 살펴봤습니다. 양측이 3-way handshake를 수행하면서 처음 전송하는 SYN, SYN + ACK 세그먼트에 대해 상대가 응답하지 못했을 때, 이를 재전송하기 위한 RTO는 어떻게 될까요?

**InitRTO는 두 종단 간 최초의 TCP 연결을 시작할 때, 즉 TCP 3-way handshake가 일어나는 양측의 첫 번째 패킷에 대한 RTO를 의미하며, 커널 소스 코드에 1초로 설정되어 있습니다.** InitRTO를 테스트하기 위해 생성한 EC2의 보안 그룹에서 로컬 VM의 접속을 허용하는 인바운드 규칙을 삭제하고 연결을 시도해 보겠습니다.

## Connection Timeout

<img title="" src="../../images/2025-06-25-tcp_retries/125841be23c9a72ebc26568eb6c0af0165c5c178.png" alt="loading-ag-1016" data-align="center">

<img title="" src="../../images/2025-06-25-tcp_retries/f8621f2b0d72f7d2b595bca5b426e2c1e150b6dd.png" alt="loading-ag-1014" data-align="center">

최초 SYN 세그먼트를 전송하고 InitRTO 값인 1초에서 시작하여 RTO가 2배씩 증가하는 것을 확인할 수 있는데요. 이렇게 리눅스 커널은 **ACK 응답을 받지 못하고 RTO가 만료되면 다음 RTO를 2배씩 증가시키는 기능을 갖고 있습니다.**

```bash
sysctl -w net.ipv4.tcp_syn_retries=3
```

그리고 위의 커널 파라미터를 통해 **최초 SYN 재전송 횟수**를 지정할 수 있는데요. 저의 로컬 VM 환경에서 이 값을 3으로 설정했기 때문에 3번째 재전송에서 Timeout이 발생하는 것을 확인할 수 있었습니다.

이렇게 **3-way handshake 과정 중 SYN, SYN + ACK 세그먼트가 유실되거나 끝내 응답을 받지 못하여 발생하는 Timeout을 Connection Timeout이라고 합니다.** 그래서 InitRTO와 `net.ipv4.tcp_syn_retries` 파라미터는 Connection Timeout의 발생에 영향을 주는 요소들이 되는데요. 

**사실 클라이언트와 서버를 이어주는 이 네트워크는 구성 요소가 굉장히 방대하고 복잡하기 때문에 항상 완벽하게 동작하지만은 않습니다. 그래서 꼭 장애 상황이 아니더라도 일시적으로 네트워크가 불안정으로 패킷 유실이 발생하는 경우가 있을 수 있기 때문에 시스템은 한두번 정도의 패킷 유실은 용인하고 재전송 하도록 하여 세션이 무사히 연결될 수 있도록 해야 합니다.** 

따라서 InitRTO는 커널 소스에 고정된 상수값이기 때문에 변경할 수 없더라도 `net.ipv4.tcp_syn_retries` 값은 네트워크 성능과 환경에 따라 동적으로 조정하여 <u>Connection Timeout이 발생하기까지 적어도 3초 이상은 보장하는 것이 좋습니다.</u> 이 점을 감안하여 보통의 리눅스 환경에서는 Default로 이 값이 넉넉하게 6 정도로 설정되어 있습니다.

## Read Timeout

반면 Read Timeout은 **이미 연결되어 있는 TCP 세션을 통해 데이터를 요청(Read)했는데 응답이 늦어 발생하는 Timeout입니다.** 주로 애플리케이션에서 Connection Pool과 Keepalive를 이용하여 세션을 유지하는 상태에서 발생하는 Timeout이죠. 클라이언트가 연결된 TCP 세션을 통해 데이터를 요청하면 서버는 요청을 정상적으로 수신받았다는 의미로 ACK를 먼저 응답한 다음 요청 데이터를 응답합니다. 

따라서 **Read Timeout이 발생하기까지 최소 (RTT를 기반 RTO + 요청 데이터 응답 시간)의 시간을 보장해야 하는데 한두번 정도의 패킷 유실로 인한 재전송을 감안하여 이보다 조금 더 넉넉한 시간을 보장하는 것이 좋습니다.**

Read Timeout을 간단히 체험해보기 위해 다시 생성한 EC2의 보안 그룹을 원래대로 되돌리고 응답 지연이 있는 nginx를 구성해 보겠습니다.

```bash
apt-get install -y nginx-extras
vi /etc/nginx/sites-enabled/default
```

```
location / {
         try_files $uri $uri/ =404;
         echo_sleep 0.5;
 }
```

```bash
nginx -t
systemctl reload nginx
```

nginx-extras 모듈을 설치하고 location 블록에 sleep 구문을 추가하여 nginx가 500ms 동안의 sleep 후에 요청에 대한 응답을 제공하도록 설정했습니다.

여기서 클라이언트가 데이터를 요청하고 Read Timeout까지 보통의 RTO보다 조금 큰  300ms만을 보장한다면 어떻게 될까요? curl을 이용하여 서버에 요청을 전송해 보도록 하겠습니다.

```bash
curl --max-time 0.3 http://[서버 주소]
```

<img title="" src="../../images/2025-06-25-tcp_retries/8c5ec7318460e9da250c984dae5fa66c1f98a4d7.png" alt="loading-ag-411" data-align="center">

<img title="" src="../../images/2025-06-25-tcp_retries/a55f2c7ebf43c5d71ab72d1987822fa44ece4781.png" alt="loading-ag-418" data-align="center">

서버에서 요청 데이터를 응답하는데 500ms가 필요한데 클라이언트가 300ms만에 Timeout을 선언하고 4-way handshake로 세션을 종료해 버립니다. 서버는 500ms 후 요청 데이터를 응답하고 4-way handshake를 수행하려 하지만 이미 세션이 종료되었기 때문에 전송한 FIN 세그먼트에 대해 RST 세그먼트가 되돌아옵니다.

조금 더 시간을 넉넉하게 잡아 이번엔 1초의 시간을 보장하고 요청을 보내보도록 하겠습니다.

```bash
curl --max-time 1 http://[서버 주소]
```

<img title="" src="../../images/2025-06-25-tcp_retries/da0760930a2ddd750a75c6536dc855afa38ba2ee.png" alt="loading-ag-440" data-align="center">

Read Timeout까지 세션의 RTO와 서버의 요청 응답에 필요한 시간보다 더 넉넉한 시간을 보장하니 클라이언트의 응답 수신과 양측의 4-way handshake가 정상적으로 수행된 것을 확인할 수 있습니다.

## TCP 세션 연결 이후의 재전송 횟수

TCP 세션 연결 이후에는 RTT를 기반으로 계산된 RTO를 기준으로 재전송이 발생한다고 했는데요. RTO를 2배씩 증가시키면서 재전송을 수행하는 횟수는 다음의 커널 파라미터를 통해 설정됩니다.

- `net.ipv4.tcp_retries1` - 초기 재전송 횟수, 일시적인 네트워크 불안정에 빠르게 대응하여 응답을 받기 위한 용도로 사용된다.
- `net.ipv4.tcp_retries2` - `tcp_retries1` 이후에도 응답이 없을 때 이 횟수까지 재전송해도 응답이 없으면 세션 종료 

Default로 `tcp_retries1`은 3, `tcp_retries2`는 15로 설정되어 있는데요. 이는 <u>TCP 세션 연결 후 데이터 요청에 대해 ACK 응답이 오지 않을 때 RTO를 2배씩 증가시키며 우선적으로 3번 요청을 재전송하고, 그래도 응답이 없으면 최대 15번까지 재전송하여 결국 응답을 받지 못했을 때 세션을 종료하는 식으로 동작하게 됩니다.</u>

## 마치며

지금까지 리눅스 시스템에서의 TCP 재전송과 Timeout에 대해 자세히 살펴봤는데요. TCP 재전송은 필요한 세션을 반드시 연결하기 위한 기능이고 Timeout은 연결이 불가능한 세션을 빠르게 인지하고 포기하기 위한 기능이라고 볼 수 있습니다. 그러나 재전송 주기를 짧게 하면서 횟수를 많게 하면 그만큼 네트워크 성능에 영향을 주게 되고, 재전송 횟수를 너무 낮게 하고 Timeout 시간을 너무 짧게 하면 충분히 연결될 수 있는 세션을 고작 일시적인 이슈 때문에 연결하지 못하는 경우가 발생할 수 있습니다. 그래서 자신이 운영하는 시스템과 인프라의 특성을 잘 이해하는 것이 중요하며, 예상치 못한 Timeout이 발생했을 때 tcpdump와 와이어샤크 같은 도구들을 이용하여 패킷의 흐름을 살펴보고 재전송 횟수와 Timeout까지의 시간을 유동적으로 조정해볼 수 있겠습니다.