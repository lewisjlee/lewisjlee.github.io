---
layout: single
title:  "커널 파라미터, 그게 뭐야?"
categories: Linux
tag: [Linux]
author_profile: false
sidebar:
    nav: "docs"
---

이번 포스팅에서는 컴퓨터 친화적인 영역인 커널과 관련된 주제로 한 번 작성해 보려고 합니다. 커널은 리눅스의 기반이 되는 굉장히 방대한 시스템 소프트웨어이기 때문에 커널 이야기를 하려고 하면 벌써부터 머리가 복잡하고 아픈 것처럼 느껴질 수 있습니다. 하지만 이러한 커널이 참조할 수 있는 설정값, 즉 커널 파라미터를 지정하는 일은 생각보다 그렇게 어렵게 느껴지지 않을 것 같습니다.

## 커널? 그게 뭐야?

<img title="" src="../../images/2025-01-20-linux-kernel-parameter/92b5b16e0d366d13cf84dffea056ac11e72e3de6.png" alt="loading-ag-130" data-align="center">{: .align-center}

커널은 운영체제의 기본 기능을 제공하고 하드웨어를 관리하고 자원을 분배하는 등 핵심 역할을 수행하는 **운영체제의 실체**입니다. 실제로 운영체제에서 메모리에 계속 상주하고 있는 영역이죠. 사용자나 애플리케이션은 커널이 제공하는 **시스템 콜(System call)** 을 통해 컴퓨터 자원에 접근할 수 있고 그 종류에는 read(), write(), exec(), fork() 등 여러가지가 있습니다. 그리고 사용자는 커널을 둘러싸고 있는 **쉘(Shell)** 이라는 인터페이스를 통해 시스템 콜을 호출하고 커널에 접근할 수 있는데요. 이 때 커널 서브 시스템과 하드웨어 장치 드라이버는 내장된 **자료 구조**에 데이터를 담아 서로 통신하게 됩니다.

## 커널 파라미터

커널 파라미터(kernel parameter)는 리눅스 OS에서 커널 동작을 제어할 수 있는 설정값을 의미하는데요. 사용자가 직접 설정하여 OS 동작을 최적화할 수 있습니다. 모든 리소스와 프로세스를 파일 시스템으로 관리하는 리눅스는 커널 파라미터 값을 /proc/sys 하위 다양한 경로에 저장합니다.

### sysctl

사용자는 RHEL 계열 OS에서 기본으로 제공되는 procps 패키지에 포함된 **sysctl**이라는 유틸리티를 통해 커널 파라미터를 설정하고 관리할 수 있습니다. 그리고 **/etc/sysctl.conf** 라는 설정 파일에 시스템 부팅 시 자동으로 적용될 커널 파라미터를 지정할 수 있습니다.

주로 사용되는 sysctl command는 다음과 같습니다.

- **sysctl -a** : 현재 설정된 모든 커널 파라미터를 출력

- **sysctl -p** : /etc/sysctl.conf에 설정된 값들을 재부팅 없이 즉시 적용

- **sysctl -w** : 특정 커널 파라미터 값을 일시적으로 적용할 때 사용할 수 있습니다. 설정한 값은 시스템 재부팅 시 초기화되며, 중지 및 재부팅이 가능한 환경인 전제 하에 원하는 커널 파라미터 값을 테스트할 때 유용하게 사용할 수 있습니다.

## 주요 파라미터 종류

**어떤 커널 파라미터를 어떻게 설정해야 할지는 서버의 역할과 사양, 시스템 인프라 구조에 따라 모두 다르기 때문에 여러 테스트를 통해 가장 적합한 값을 찾아서 적용하는 것이 좋습니다.** 저 또한 모든 커널 파라미터 종류를 다 알지 못하고 이전에 설정했던 값들이 해당 서버에 가장 핏한 값이었는지 100% 확신하기 어렵기 때문에 보다 많은 추가 학습을 필요로 할 것 같습니다. 따라서 실제로 어떤 파라미터를 어떤 값으로 설정했는지 그대로 드러내기보단 제가 실제로 적용했던 주요 커널 파라미터의 종류를 그 의미와 함께 살펴보는 것이 좋을 것 같다고 판단했습니다.

먼저 kernel session의 파라미터입니다.

- `kernel.sysrq` - 매직 키 활성화 여부
- `kernel.core_uses_pid` - Core Dump 생성 활성화 여부

**매직 키(Magic System Request Key)**는 커널이 어떤 워크로드를 바쁘게 진행하고 있더라도 항상 요청에 반응할 수 있도록 할 수 있는 기능입니다. 온프레미스 환경에서 System hang이 걸렸을 때 긴급 조치가 가능하도록 할 수 있는 기능으로 유용하게 쓰일 것 같지만, <span style="color:red">물리적으로 서버에 연결할 수만 있다면 계정 권한을 타지 않고 시스템을 악의적으로 제어할 수 있기 때문에</span> 일반적으로 0을 설정합니다.

**코어 덤프(Core Dump)**는 장애 등의 이유로 프로세스가 비정상적으로 종료되었을 때 프로세스와 관련된 메모리를 dump 백업할 수 있는 기능입니다. 일부 애플리케이션에서는 관련 프로세스가 죽었을 때 log를 남기지 않는 경우가 있는데요. 이 때 코어 덤프를 활성화 시키면 dump된 파일을 갖고 문제를 분석할 수 있습니다.

### 웹 서비스 운영에 사용되는 네트워크 파라미터

- `net.core.somaxconn` - 최대 소켓 Connection 성사 수
- `net.ipv4.tcp_max_syn_backlog` - SYN 수신 대기열(Backlog) 최대 크기
- `net.ipv4.tcp_fin_timeout` - FIN Segment 대기 시간
- `net.ipv4.tcp_keepalive_time` - TCP KeepAlive 주기
- `net.ipv4.tcp_tw_reuse` - TIME_WAIT 상태 로컬 포트 재사용 활성화 여부
- `net.ipv4.ip_local_port_range` - 클라이언트 로컬 포트 범위
- `net.ipv4.tcp_syn_retries` - Active SYN 재전송 횟수
- `net.ipv4.tcp_retries1` - TCP 세션 연결 후 요청에 대한 ACK 응답을 받지 못할 때의 초기 재전송 횟수

`net.ipv4.tcp_max_syn_backlog`는 클라이언트로부터 SYN 세그먼트를 수신한 **SYN-RECV** 상태의 연결 최대 수를 나타내고, `net.core.somaxconn`은 최종적으로 ACK를 받고 **ESTABLISH** 상태가 된 소켓 연결의 최대 수를 나타냅니다. 따라서 **외부로부터 많은 연결 요청을 받는 웹 서버**에서는 이런 값들을 넉넉하게 늘려줘야 하는데요. 너무 낮게 설정하면 일부 TCP 연결이 거부되거나 정상적으로 성사되지 않을 수 있고, 너무 높게 설정하면 메모리 사용량이 증가하여 시스템 성능 저하를 불러올 수 있으므로 메모리 크기를 고려하여 설정해야 합니다.

 WAS와 같은 애플리케이션 서버와 많은 연결을 맺는 웹 서버 특성 상 `net.ipv4.tcp_tw_reuse`와 `net.ipv4.ip_local_port_range`도 유용하게 쓰일 수 있는 파라미터입니다. 대부분의 리눅스 커널의 로컬 포트 번호 범위는 32768 ~ 60999로 지정되어 있는데요. `net.ipv4.ip_local_port_range`를 조정하여 클라이언트에서 서버와의 세션이 늘어날 때 더 많은 로컬 포트를 사용할 수 있도록 범위를 늘려줄 수 있습니다. 여기서 `net.ipv4.tcp_tw_reuse`를 활성화하여 TIME_WAIT 상태의 소켓을 재사용할 수 있도록 훨씬 더 여유로운 로컬 포트 할당이 가능하게 됩니다.

### 악성 공격 대응을 위한 파라미터

- `net.ipv4.tcp_syncookies` - 보내는 SYN Segment에 Cookie값 추가
- `net.ipv4.tcp_synack_retries` - Passive SYN/ACK 재전송 횟수

SYN Flooding 공격을 방지하고 정상적인 TCP 연결을 보장하기 위한 파라미터입니다. <span style="color:red">SYN Flooding은 다수의 좀비 노드를 통해 SYN 세그먼트를 동시다발적으로 전송하고 돌아오는 ACK/SYN에 대한 응답을 제공하지 않아 시스템을 장시간 대기 상태로 만드는 공격인데요.</span> ACK/SYN 세그먼트를 응답할 때 쿠킷값을 추가하여 같은 Source로부터 다시 수신된 SYN를 무시하고 SYN/ACK 응답 횟수 제한을 설정하여 SYN Flooding 공격에 효과적으로 대응할 수 있습니다.

- `net.ipv4.conf.all.arp_notify`- IPv4 주소나 장비 변경 시 알림
- `net.ipv4.conf.default.accept_redirects` - redirect된 ICMP 수신 활성화 여부
- `net.ipv4.conf.default.send_redirects` - ICMP Packet redirect 전송 활성화 여부
- `net.ipv4.conf.default.accept_source_route` - Source Routing 수신 활성화 여부
- `net.ipv4.conf.default.secure_redirects` - 게이트웨이로부터의 redirect 활성화 여부
- `net.ipv4.icmp_echo_ignore_broadcasts` - Broadcast로부터의 ICMP 수신 활성화 여부
- `net.ipv4.icmp_ignore_bogus_error_responses` - 비정상 TCP/IP Header의 ICMP Packet 무시 여부

서버가 스푸핑(spoofing)과 스머프(smurf) 공격에 악용되지 않도록 하는 설정입니다. 자신의 서버를 경유하여 변조된 악성 패킷을 공격 대상으로 리다이렉트하는 것을 방지하고 반대로 자신의 서버가 리다이렉트된 악성 패킷을 수신하지 못하도록 설정할 수 있습니다. 특히 공인 IP를 갖고 인터넷에 개방된 서버의 경우 이러한 파라미터들이 필수적으로 적용될 것으로 생각되네요.

### 메모리 할당 및 Swap 관련 파라미터

- `vm.swappiness` - Swap 메모리 사용 정도, 기본값은 60
- `vm.overcommit_memory` - 메모리 overcommit 동작 방식 지정

Swap 및 메모리 할당에 대한 파라미터입니다. **`vm.swappiness`는 프로세스에 할당할 물리 메모리 공간이 부족할 때 캐시 메모리를 재할당할 것인지 아니면 Swap을 사용할 것인지의 비율을 조정할 수 있는 파라미터입니다.** 기본값보다 높게 설정하여 Page Cache 용량이 충분해도 Swap을 사용하도록 할 수 있죠.

Swap은 디스크의 일부분을 메모리처럼 사용하기 위한 공간이므로 프로세스가 Swap 영역을 참조할 경우 I/O가 발생하여 시스템 성능이 저하될 수 있습니다. 하지만 프로세스로부터 참조된 지 오래된 Inactive 메모리 영역이 주로 Swap 영역으로 이동하기 때문에 자주 사용하지 않는 프로세스가 많을 경우 `vm.swappiness`를 조금 더 높게 조정할 수 있습니다.

메모리 commit은 프로세스가 메모리 공간을 할당받는 것을 의미하는데요. 애플리케이션을 구동하다 보면 자식 프로세스가 부모 프로세스의 메모리 주소 공간을 그대로 복사하는 fork() 시스템 콜 호출이 발생할 수 있습니다. 그렇게 되면 **실제 메모리 공간을 초과하는 메모리 공간 할당이 필요하게 되죠. 이것을 overcommit이라고 합니다.** 하지만 대부분 fork()에서 exec() 시스템 콜로 전환되어 결과적으로 초기 할당받은 메모리를 모두 사용하지 않기 때문에 일시적인 fork()를 위한 overcommit을 통해 프로세스 생성과 진행을 원활하게 할 수 있습니다.

`vm.overcommit_memory` 값은 0,1,2 총 세 가지 값을 지정할 수 있는데 각각의 값이 가지는 의미는 다음과 같습니다.

- **0** - 기본값, page cache + swap 영역 + Slab reclaimable 값보다 작으면 계속 commit
- **1** - 가용 메모리에 상관없이 요청 온 모든 메모리에 대해 commit, <span style="color:red">메모리 누수를 일으키는 프로세스가 있다면 시스템 응답 불가 현상을 일으킬 수도 있으니 주의해야 한다.</span>
- **2** - `vm.overcommit_ratio`에 설정된 비율값과 swap 크기를 토대로 제한적인 commit

### Docker, 쿠버네티스 운영에서의 커널 파라미터

```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

컨테이너 운영 환경에서 필수적으로 설정해야 할 커널 파라미터가 있습니다. 리눅스 Host는 **iptables** 정책을 통해 네임스페이스 기반으로 격리되어 동작하는 컨테이너에 트래픽을 라우팅하는데요. 이를 위해 호스트에 수신된 패킷을 네임스페이스로 Forwarding하고 **브릿지(Bridge)** 네트워크가 iptables 정책을 통해 패킷을 주고받을 수 있도록 위의 파라미터 값들을 1로 활성화해야 합니다.

## 커널 파라미터 설정 시 주의사항

리눅스 커널에는 정말 많은 종류의 커널 파라미터가 존재합니다. 그래서 <span style="color:red">특정 파라미터에 대해 자신이 아는 바가 정확하지 않을 수 있고, 함부로 조정했다가 시스템이 예상치 못한 반응을 할 수도 있기 때문에 운영 환경에 무턱대고 적용하는 것은 매우 위험할 수 있습니다.</span> 따라서 시스템 성능 개선의 방편으로 커널 파라미터 튜닝을 고려하고 있다면 먼저 비슷한 환경을 구축하고 테스트한 다음 적용하는 식으로 안전하게 진행하는 것이 좋습니다.

#### References

[S-Core - 커널 옵션 대 방출! Kubernetes 환경을 최적화하라](https://s-core.co.kr/insight/view/%EC%BB%A4%EB%84%90-%EC%98%B5%EC%85%98-%EB%8C%80-%EB%B0%A9%EC%B6%9C-kubernetes-%ED%99%98%EA%B2%BD%EC%9D%84-%EC%B5%9C%EC%A0%81%ED%99%94%ED%95%98%EB%9D%BC/)

강진우. (2017). DevOps와 SE를 위한 리눅스 커널 이야기. 인사이트