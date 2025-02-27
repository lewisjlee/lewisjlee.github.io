---
layout: single
title:  "커널 파라미터, 어떻게 설정해야 할까?"
categories: Linux
tag: [Linux]
author_profile: false
sidebar:
    nav: "docs"
---

이번 포스팅에서는 컴퓨터 친화적인 영역인 커널과 관련된 주제로 한 번 작성해 보려고 합니다. 자동차 운전을 예로 들자면, 그냥 차를 사서 몰고 싶을 뿐인데 엔진과 기어의 원리를 세세하게 알려고 할 필요가 없겠죠. 설령 알려고 한다 해도 전공자가 아닌 이상 굉장히 머리가 아플 겁니다. 마찬가지로 리눅스나 윈도우 등 OS 위에서 서버를 운영하다 보면 프로세스 구동이나 성능 확인 등 주로 사용자로부터 가까운 기능들을 먼저 익히고 사용하는데요. 때문에 커널 이야기를 하려고 하면 벌써부터 복잡하고 머리가 아픈 것처럼 느껴질 수 있습니다. 하지만 이러한 커널이 직접적으로 참조할 수 있는 설정값, 즉 커널 파라미터를 지정하는 일은 생각보다 그렇게 어렵게 느껴지지 않을 것 같습니다.

## 커널? 그게 뭐야?

<img title="" src="../../images/2025-01-20-linux-kernel-parameter/92b5b16e0d366d13cf84dffea056ac11e72e3de6.png" alt="loading-ag-130" data-align="center">{: .align-center}

커널은 운영체제의 기본 기능을 제공하고 하드웨어를 관리하고 자원을 분배하는 등 핵심 역할을 수행하는 **운영체제의 실체**입니다. 실제로 운영체제에서 메모리에 계속 상주하고 있는 영역이죠. 사용자나 애플리케이션은 커널이 제공하는 **시스템 콜(System call)** 을 통해 컴퓨터 자원에 접근할 수 있고 그 종류에는 read(), write(), exec(), fork() 등 여러가지가 있습니다. 그리고 사용자는 커널을 둘러싸고 있는 **쉘(Shell)** 이라는 인터페이스를 통해 시스템 콜을 호출하고 커널에 접근할 수 있는데요. 이 때 커널 서브 시스템과 하드웨어 장치 드라이버는 내장된 **자료 구조**에 데이터를 담아 서로 통신하게 됩니다.

## 커널 파라미터

커널 파라미터(kernel parameter)는 리눅스 OS에서 커널 동작을 제어할 수 있는 설정값을 의미하는데요. 사용자가 직접 설정하여 OS 동작을 최적화할 수 있습니다. 모든 리소스와 프로세스를 파일 시스템으로 관리하는 리눅스는 커널 파라미터 값을 /proc/sys 하위 다양한 경로에 저장합니다.

### sysctl

사용자는 RHEL 계열 OS에서 기본으로 제공되는 procps 패키지에 포함된 **sysctl**이라는 유틸리티를 통해 커널 파라미터를 설정하고 관리할 수 있습니다. 그리고 **/etc/sysctl.conf** 라는 설정 파일에 시스템 부팅 시 자동으로 적용될 커널 파라미터를 지정할 수 있습니다.

주로 사용되는 sysctl command는 다음과 같습니다.

* **sysctl -a** : 현재 설정된 모든 커널 파라미터를 출력

* **sysctl -p** : /etc/sysctl.conf에 설정된 값들을 재부팅 없이 즉시 적용

* **sysctl -w** : 특정 커널 파라미터 값을 일시적으로 적용할 때 사용할 수 있습니다. 설정한 값은 시스템 재부팅 시 초기화되며, 중지 및 재부팅이 가능한 환경인 전제 하에 원하는 커널 파라미터 값을 테스트할 때 유용하게 사용할 수 있습니다.

## 어떤 파라미터를 설정해야 할까?

이제부터 제가 실제로 실무에서 적용했던 커널 파라미터를 살펴보려고 하는데요.

```
kernel.sysrq = 0    #   매직 키 비활성화
kernel.core_uses_pid = 1    #   Core Dump 생성 활성화
```

먼저 kernel session의 파라미터입니다.

**매직 키(Magic System Request Key)**는 커널이 어떤 워크로드를 바쁘게 진행하고 있더라도 항상 요청에 반응할 수 있도록 할 수 있는 기능입니다. System hang이 걸렸을 때 긴급 조치가 가능하도록 할 수 있는 기능으로 유용하게 쓰일 것 같지만, <span style="color:red">물리적으로 서버에 연결할 수만 있다면 계정 권한을 타지 않고 시스템을 악의적으로 제어할 수 있기 때문에</span> 일반적으로 0을 설정하여 비활성화 합니다.

**코어 덤프(Core Dump)**는 장애 등의 이유로 프로세스가 비정상적으로 종료되었을 때 프로세스와 관련된 메모리를 dump 백업할 수 있는 기능입니다. 일부 애플리케이션에서는 관련 프로세스가 죽었을 때 log를 남기지 않는 경우가 있는데요. 이 때 코어 덤프를 활성화 시키면 dump된 파일을 갖고 문제를 분석할 수 있습니다. 기본값이 0으로 비활성화 되어 있기 때문에 1로 설정하여 활성화 해줍니다.
### 웹 서비스 운영을 위한 파라미터
```
net.core.somaxconn=65535    #   Connection 대기 최대 수
net.ipv4.tcp_max_syn_backlog = 8192 #   SYN 수신 Backlog 최대 크기
net.ipv4.tcp_fin_timeout = 10   #   FIN Segment 대기 시간
net.ipv4.tcp_keepalive_time = 10    #    TCP KeepAlive 주기
net.ipv4.tcp_tw_reuse = 0   #   TIME_WAIT 상태 로컬 포트 재사용 비활성화
net.ipv4.ip_local_port_range = 15000    64000   #   클라이언트 로컬 포트 범위
net.ipv4.tcp_syn_retries = 2    #   Active SYN 재전송 횟수
net.ipv4.tcp_retries1 = 2   #   TCP 연결 자체에 문제가 있을 때 재시도 횟수
```
이번에는 **메모리 64G** 기준 WEB/WAS 서버에 설정할 수 있는 네트워크 관련 파라미터를 살펴보겠습니다. 웹 서비스 인프라 특성 상 외부에서 많은 TCP 연결 요청을 수신하고 WAS나 DB 등 내부 컴포넌트와 많은 Connection을 생성하게 되는데요. 그래서 Connection 대기 최대 수와 SYN Backlog를 넉넉하게 설정하고 FIN 대기 시간과 KeepAlive 시간을 짧게 설정하여 서버가 동시 접속자 수와 신규 접속을 원활하게 처리할 수 있도록 해줬습니다. 그리고 상품 주문과 결제 등 트랜잭션을 데이터 유실 없이 정확한 처리와 응답을 보장하기 위해 4-way handshake가 완전히 종료되지 않은 TIME_WAIT 상태의 소켓의 재사용을 비활성화 해줬습니다.

대부분의 리눅스 커널의 로컬 포트 번호 범위는 32768 ~ 60999로 지정되어 있는데요. 접속이 늘어날수록 워크로드가 많아지면서 WEB에서 WAS로, WAS에서 DB로 가는 세션 수 또한 증가하기 때문에 15000 ~ 64000번으로 늘려 주었습니다. 그리고 TCP 연결 재시도 횟수는 새로 생성되는 Connection에 대한 지연을 고려하여 기본값보다 낮게 각각 2로 설정하였습니다.

추가적으로 데이터베이스는 다른 서버에 주로 먼저 연결 요청을 하지 않지만 애플리케이션 서버로부터 많은 요청을 수신하기 때문에 somaxconn와 max_syn_backlog 정도는 꼭 설정해 주는 것이 좋습니다.
### 악성 공격 대응을 위한 파라미터
```
net.ipv4.tcp_syncookies = 1 #   보내는 SYN Segment에 Cookie값 추가
net.ipv4.tcp_synack_retries = 2 #   Passive SYN/ACK 재전송 횟수
```
SYN Flooding 공격을 방지하고 정상적인 TCP 연결을 보장하기 위한 설정입니다. <span style="color:red">SYN Flooding은 다수의 좀비 노드를 통해 SYN 세그먼트를 동시다발적으로 전송하고 돌아오는 ACK/SYN에 대한 응답을 제공하지 않아 시스템을 장시간 대기 상태로 만드는 공격</span>인데요. ACK/SYN를 응답할 때 쿠킷값을 추가하여 같은 Source로부터 다시 수신된 SYN를 무시하고 SYN/ACK 응답 횟수 제한을 설정하여 SYN Flooding 공격에 효과적으로 대응할 수 있습니다.
```
net.ipv4.conf.all.arp_notify = 1    #   IPv4 주소나 장비 변경 시 알림
net.ipv4.conf.default.accept_redirects = 0  #   redirect된 ICMP 수신 비활성화
net.ipv4.conf.default.send_redirects = 0    #   ICMP Packet redirect 전송 비활성화
net.ipv4.conf.default.accept_source_route = 0   #   Source Routing 수신 비활성화
net.ipv4.conf.default.secure_redirects = 0  #   게이트웨이로부터의 redirect 비활성화
net.ipv4.icmp_echo_ignore_broadcasts = 1    #   Broadcast로부터의 ICMP 수신 비활성화
net.ipv4.icmp_ignore_bogus_error_responses = 1  #   비정상 TCP/IP Header의 ICMP Packet 무시
```
서버가 스푸핑(spoofing)과 스머프(smurf) 공격에 악용되지 않도록 하는 설정입니다. 자신의 서버를 경유하여 변조된 악성 패킷을 공격 대상으로 리다이렉트하는 것을 방지하고 반대로 자신의 서버가 리다이렉트된 악성 패킷을 수신하지 못하도록 설정할 수 있습니다. 특히 외부에 공인 IP로 개방된 서버의 경우 이러한 파라미터들이 필수적으로 적용될 것으로 생각되네요.
### 메모리 할당 및 Swap
```
vm.swappiness = 20  #   SWAP 메모리 사용 범위
vm.overcommit_memory = 1    #   메모리 공간 할당이 항상 성공하도록 메모리 overcommit을 허용
```
Swap 및 메모리 할당에 대한 파라미터입니다. swappiness 값은 보통 10이 권장되지만 메모리 성능을 더욱 넉넉하게 유지하기 위해 20으로 설정하고, overcomit_memory 값을 1로 설정하여 메모리 overcommit을 허용해 주었습니다.

메모리 commit은 프로세스가 메모리 공간을 할당받는 것을 의미하는데요. 애플리케이션을 구동하다 보면 자식 프로세스가 부모 프로세스의 메모리 주소 공간을 그대로 복사하는 fork() 시스템 콜 호출이 발생할 수 있습니다. 그렇게 되면 **실제 메모리 공간을 초과하는 메모리 공간 할당이 필요하게 되죠. 이것을 overcommit이라고 합니다.** 하지만 대부분 fork()에서 exec() 시스템 콜로 전환되어 결과적으로 초기 할당받은 메모리를 모두 사용하지 않기 때문에 일시적 fork()를 위한 overcommit 과정을 허용하여 프로세스 생성과 진행을 원활하게 할 수 있습니다.

### Docker, 쿠버네티스 운영에서의 커널 파라미터
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```
번외로 컨테이너 운영 환경에서 필수적으로 설정해야 할 커널 파라미터가 있습니다. 리눅스 Host는 **iptables** 정책을 통해 네임스페이스 기반으로 격리되어 동작하는 컨테이너에 트래픽을 라우팅하는데요. 이를 위해 호스트에 수신된 패킷을 네임스페이스로 Forwarding하고 **브릿지(Bridge)** 네트워크가 iptables 정책을 통해 패킷을 주고받을 수 있도록 위의 파라미터 값들을 1로 활성화해야 합니다.