---
layout: single
title:  "TLS 프로토콜과 쿠버네티스 클러스터에서의 TLS"
categories: Kubernetes
tag: [쿠버네티스, Kubernetes]
author_profile: false
sidebar:
    nav: "docs"
---

인터넷을 사용하다 보면 도메인 주소에 https가 붙어 있는 것을 흔히 확인할 수 있습니다. 일반적으로 웹 통신 프로토콜은 80번 포트를 사용하는 http로 알려져 있는데 https와는 어떤 차이가 있을까요? 443번 포트를 사용하는 https 프로토콜은 웹 서버에 전달되는 민감한 데이터가 제 3자에 노출되지 않도록 **전송 중 암호화** 방식을 사용합니다. 이러한 전송 중 암호화 방식을 TLS(Transport Layer Security)라고 하고 통상적으로 SSL(Secure Socket Layer) 이라고 알려져 있죠.

TLS는 웹 통신 뿐만 아니라 쿠버네티스 클러스터 내부에서도 사용되는데요. 쿠버네티스를 구성하는 컴포넌트들은 종류가 다양하고 API 통신을 기반으로 민감한 정보를 주고받는 경우도 있기 때문에 서로에게 자신을 증명하고 암호화된 통신을 할 수 있어야 합니다. 그렇다면 TLS는 어떤 방식으로 통신하고 쿠버네티스에서는 어떻게 적용될까요? 본 포스팅에서는 TLS 암호화 통신 방식을 먼저 알아보고 쿠버네티스에서 어떻게 TLS 통신이 이루어지는지 살펴보려고 합니다.

## TLS

### 비 TLS 통신

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/68d6323a9a8e6085e4c1987edb9d7e29826a21fd.png" alt="loading-ag-1021" data-align="center">

TLS가 적용되지 않은 웹 통신에서는 어떤 일이 발생하게 될까요? 먼저 사용자가 접속하고자 하는 웹 사이트가 있고, 이러한 웹 사이트를 서비스하는 웹 서버가 있고 미상의 제 3자(Hacker)가 사용자와 웹 서버가 통신하는 네트워크 중간에 있다고 가정해 보겠습니다. 사용자와 웹 서버와의 세션은 성립되었고, 제 3자는 사용자가 패스워드나 주민등록번호 같은 민감 정보를 입력해서 웹 서버에 전달하길 기다리고 있습니다.

<img title="" src="../../../images/2024-12-26-ssl_tls_kubernetes/c72ced2267f6385adf81757987079776f043b2f5.png" alt="loading-ag-1035" data-align="center">

사용자가 로그인을 하기 위해 아이디와 패스워드를 입력하고 네트워크를 통해 웹 서버에 전달됩니다. 세션이 암호화를 지원하지 않기 때문에 아이디를 포함하여 패스워드는 평문 그대로 제 3자에게 탈취됩니다. 그래서 대부분의 웹 서버는 TLS가 적용되지 않은 일반 http 프로토콜로 접근하는 세션을 저절로 https로 리다이렉트되도록 설정합니다. 이는 ISMS 인증 심사의 요구 사항 중 하나이기도 합니다.

### TLS 통신

#### SSL/TLS Handshake

그럼 TLS 통신 방식을 본격적으로 살펴볼께요. 사용자가 도메인을 입력하고 웹 사이트에 접속하게 되면 TCP 3-way handshake를 먼저 수행하고 그 다음 TLS handshake를 수행합니다. TLS handshake 과정에서 사용자의 웹 브라우저가 지원하는 Cipher Suite(암호화 방식)를 서버 측에 제시합니다. 이 때 서버 측에서는 제시한 방식 중 하나를 선정하여 사용자에게 알리게 됩니다.

#### 클라이언트에 공개 키 전달

TLS Handshake가 성사되면 서버는 사용자에게 자신의 인증서에 공개 키를 포함하여 전달합니다. 일반적인 암호화 통신에서는 키를 사용하여 민감 정보를 암호화하여 전달하는데요. 웹 서버는 암호화에 사용할 키를 사용자에게 전달하여 아이디와 패스워드를 암호화하도록 할 수 있습니다. 하지만 키를 사용자에게 전달하는 과정에서 제 3자가 키를 탈취하게 되면 이러한 방식은 전혀 의미가 없게 됩니다. 따라서 TLS는 **암호화할 수 있는 키(Public key)** 와 **복호화할 수 있는 키(Private key)** 를 따로 두는 **비대칭 암호화(Asymmetric Encryption) 방식**을 사용합니다. 제 3자는 서버가 사용자에게 전송한 공개 키를 탈취한다 해도 암호화하는데만 사용할 수 있습니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/ec3d1150f4828ed5cbb35ec7fff87106495ceed2.png" alt="loading-ag-1080" data-align="center">

#### 공개 키를 사용하여 민감 정보 암호화

공개 키의 민감 정보 암호화가 마치 함부로 열람하지 못하도록 잠그는 것과 같다고 하여 자물쇠로 표현했는데요. 공개 키를 전달받은 사용자는 자신의 아이디와 패스워드를 암호화하여 서버에 전송합니다. 제 3자는 암호화된 아이디와 패스워드를 탈취할 수 있지만 암호화만을 취급하는 공개 키로 이러한 정보들을 복호화할 수 없습니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/fc144c274ffad26daa5ab402bb32dc08dbbb2cf2.png" alt="loading-ag-1105" data-align="center">

#### 개인 키를 사용하여 민감 정보 복호화

암호화된 정보를 수신한 서버는 내용을 확인하기 위해 개인 키를 사용하여 복호화하게 됩니다. 이렇게 TLS 키를 사용하여 안전한 웹 통신이 가능하도록 할 수 있습니다. 개인 키는 네트워크를 통해 주고받는 일 없이 서버에만 저장되므로 서버 관리자는 방화벽과 접근 제어, 개인 키에 대한 permission을 철저하게 관리합니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/3bd31c8afc92bb3511f376c296e07610e30a10ab.png" alt="loading-ag-1121" data-align="center">

### 신뢰할 수 있는 사이트임을 입증하는 인증서 체인

TLS handshake 이후 서버는 공개 키를 자신의 인증서에 포함하여 사용자에 전달한다고 했는데요. 이 인증서는 어떤 역할을 하게 될까요?

<img title="" src="../../../images/2024-12-26-ssl_tls_kubernetes/2a2ff8f5db3c7cc3ca1229c8849ff95d186ff9b0.png" alt="loading-ag-1157" data-align="center">

그림과 같이 제 3자는 사용자가 접속하고자 하는 웹 사이트를 그대로 모방한 사이트를 개설하고 자신만의 TLS 키 페어를 갖게 된다고 가정해볼께요. 그리고 자신만의 특수한 방법으로 사용자가 도메인을 입력하면 자신의 가짜 서버로 라우팅하도록 DNS를 조작할 수도 있습니다. 그렇게 되면 사용자는 제 3자의 가짜 서버에 접속하여 TLS 통신을 하게 되고 제 3자는 자신의 공개 키로 암호화된 사용자의 민감 정보를 자신의 개인 키로 복호화할 수 있게 됩니다. 이를 방지하기 위해 서버는 자신이 웹 사이트 도메인의 소유자임을 **CA(Certificate Authority)** 라는 기관으로부터 인증받고 인증서를 발급받아 저장해야 합니다. 그리고 사용자에게 보여주어 자신이 신뢰할 수 있는 서버임을 입증해야 하는 것이죠. 제 3자는 인증서를 발급받으려 해도 정식으로 도메인을 구입하고 소유한 것이 확인되지 않는 이상 인증서를 발급받을 수 없습니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/1d73ab7a53964318f064918f6eda8df6b7df9638.png" alt="loading-ag-1166" data-align="center">

TLS 인증서는 **서버 인증서, 체인 인증서, 루트 인증서** 체인으로 구성되어 있습니다. 서버 인증서는 도메인 자체에 대한 인증서를 의미하고 루트 인증서는 도메인을 공식적으로 인증하는 최상위 기관인 CA의 인증서입니다. 그리고 체인 인증서는 이러한 서버 인증서와 루트 인증서의 중개 역할을 하여 서버가 CA로부터 공식적으로 인증받았음을 증명합니다. 이 세 가지 인증서 체인을 모두 갖고 있어야 비로소 신뢰할 수 있는 사이트가 되는 것이죠.

서버가 인증서를 발급받아 저장하기 위해서는 먼저 자신의 도메인 이름으로 한 **CSR(Certificate Signing Request)** 을 생성해야 합니다. 이 CSR을 CA에 전달하면 CA는 서버와 도메인에 대한 확인 절차를 거치고 서명합니다. 그리고 서명된 서버 인증서와 체인 인증서, 루트 인증서 체인을 서버에 발급합니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/2025-01-06-13-46-17-image.png" alt="loading-ag-1174" data-align="center">{: .align-center}

제가 기술블로그를 작성하는 github.io 도메인의 경우 그림과 같이 DigiCert라는 CA의 인증을 통한 인증서 체인이 구성되어 있습니다. CA는 이 외에도 Symantec, GlobalSign 등이 있는데 우리가 사용하는 웹 브라우저는 웬만한 CA의 루트 인증서 정보를 모두 갖고 있습니다. 그래서 서버가 제시하는 인증서가 CA로부터 서명된 신뢰할 수 있는 인증서인지 확인할 수 있습니다. 이렇게 검증했을 때 신뢰할 수 있는 사이트일 경우 TLS 세션이 성사되고 그렇지 않을 경우 악의적인 제 3자로 간주되어 신뢰할 수 없는 사이트임을 표시하게 됩니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/fffbdbc6f879acefed3e282ade983fb082941c8d.png" alt="loading-ag-1186" data-align="center">

## 쿠버네티스 클러스터에서의 TLS

### mTLS

지금까지 SSL로 알려진 TLS에 대해 살펴봤는데요. 이번에는 쿠버네티스 클러스터를 구성하는 컴포넌트 간의 TLS 통신에 대해 알아보려고 합니다. 쿠버네티스 컴포넌트는 TLS 중에서도 좀 더 Zero Trust에 가까운 mTLS를 사용하는데요. 일반 TLS와는 어떤 차이가 있을까요?

앞서 살펴본 https는 서버만 자신이 신뢰할 수 있다는 것을 인증서를 통해 증명했습니다. 하지만 만약에 서버에 접속하려고 하는 클라이언트가 서버에 악의적인 공격을 감행하려는 의도를 가진 사용자라면 서버는 이를 어떻게 거부할 수 있을까요? 그리고 신뢰할 수 있는 클라이언트를 어떻게 식별할 수 있을까요? mTLS는 **클라이언트와 서버가 각자 자신의 신뢰할 수 있는 CA에서 인증한 인증서를 갖고 서로를 인증**한다는 차이점이 있습니다. 클라이언트는 서버가 제시한 인증서를 확인함으로써 신뢰할 수 있는 사이트임을 확인하고, 서버는 클라이언트가 제시한 인증서를 확인하여 신뢰할 수 있는 접속자임을 확인할 수 있는 것이죠. 쿠버네티스 컴포넌트는 이러한 mTLS를 기반으로 상호 간의 통신을 성사시킵니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/050a21f5160793b0e51195a3f253da2eb219be9c.png" alt="loading-ag-1209" data-align="center">

그림은 쿠버네티스를 구성하는 컴포넌트와 사용자가 인증서와 키를 갖고 상호 통신하는 형태를 도식화한 것입니다. 자신이 안전한 클라이언트임을 증명할 수 있는 클라이언트 인증서를 보유한 컴포넌트가 있고 자신이 안전한 서버임을 증명할 수 있는 인증서를 보유한 컴포넌트도 있습니다. 가장 핵심적인 역할을 하는 API 서버는 두 가지 다 갖고 있네요.

kubeadm을 이용하여 쿠버네티스 클러스터를 구축하면 kubeadm은 각 컴포넌트를 생성하고 인증서와 키를 한 쌍씩 생성합니다. 이 때 **클러스터 자신의 이름으로 직접 서명한 CA 루트 인증서**가 먼저 생성되고 이 루트 인증서는 kubelet 서버 인증서와 etcd를 제외한 다른 컴포넌트의 인증서들을 서명하고 발급하는데 사용됩니다.

### 관리자, 사용자

클러스터가 생성되고 나면 관리자용 인증서 데이터가 포함된 kubeconfig가 생성되고, 관리자는 이 설정 파일을 갖고 API 서버와 통신하며 클러스터를 제어할 수 있습니다. 사용자는 openssl을 이용하여 자신의 자격 증명을 위한 키와 CSR을 생성하고 관리자에게 자신의 kubeconfig 생성을 요청할 수 있습니다. 

관리자는 직접 openssl을 통해 CSR을 클러스터 CA로 서명할 수 있지만, 많은 사용자들의 CSR을 일일이 이러한 방식으로 처리하는데는 많은 번거로움이 있을 수 있습니다. 그래서 쿠버네티스 Controller Manager는 사용자 CSR을 더욱 편리하게 서명할 수 있도록 **인증서 API**를 제공하는데요. 덕분에 관리자는 kubectl 명령어로 인증서 API에 접근하여 CSR을 쉽고 빠르게 승인할 수 있습니다.

CSR을 승인하고 생성된 사용자에게 RBAC를 통해 쿠버네티스를 사용할 수 있는 적절한 권한을 부여할 수 있습니다.

자세한 내용을 아래 공식 문서 링크에서 살펴보실 수 있습니다.

[Certificates and Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#normal-user)

### Controller Manager

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/2025-01-06-19-10-29-image.png" alt="loading-ag-1618" data-align="center">Controller Manager는 정해진 개수만큼 파드를 실행시키기 위해 API 서버에 요청하는 클라이언트 컴포넌트입니다. 그래서 지정된 kubeconfig 파일에 클러스터 CA를 통해 서명된 인증서 데이터와 CA 루트 인증서 데이터가 저장되어 있고 이 설정 파일을 갖고 API 서버에 접근할 수 있습니다. 인증서 API를 제공하는 역할을 하기 때문에 CSR을 서명하기 위한 CA 루트 인증서 경로 또한 명시되어 있습니다.

### Scheduler

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/2025-01-06-19-13-58-image.png" alt="loading-ag-1634" data-align="center">{: .align-center}

마찬가지로 Scheduler는 어떤 노드에 파드를 배치하면 좋은지 API 서버에 알리는 역할을 하는 클라이언트 컴포넌트입니다. 그래서 지정된 kubeconfig 파일에 클러스터 CA로 서명된 인증서 데이터와 CA 루트 인증서 데이터가 저장되어 있고 이 설정 파일을 갖고 API 서버에 접근할 수 있습니다.

### kube-proxy

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/6b377cef142d6272daec9bb5e8f2695671b53967.png" alt="loading-ag-1684" data-align="center">{: .align-center}

kube-proxy는 서비스를 비롯한 쿠버네티스 네트워크 동작을 관리하는 역할로 역시 API 서버 호출이 필요한 클라이언트 컴포넌트입니다. 역시 지정된 kubeconfig 파일에 클러스터 CA로 서명된 인증서 데이터와 CA 루트 인증서 데이터가 저장되어 있고 이 설정 파일을 갖고 API 서버에 접근할 수 있습니다.

### kubelet

kubelet은 노드에서 파드를 실행시키는 데몬으로 상태 정보를 API 서버에 전달하는 클라이언트 역할을 하기도 합니다. 따라서 지정된 kubeconfig 파일에 클러스터 CA 루트 인증서 데이터와 서버 인증서 path가 저장되어 있습니다.

**kubelet 서버 인증서는 초기 kubeadm 구성 시 노드에서 자체적으로 발급하고 서명하는 Self-Signed 인증서로 생성됩니다.** 따라서 발급자와 Subject가 노드 이름으로 되어 있는 것을 확인할 수 있습니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/2025-01-07-15-35-25-image.png" alt="loading-ag-406" data-align="center">{: .align-center}

클라이언트 인증서를 살펴보면 쿠버네티스 클러스터로부터 발급되었다는 점과 Subject가 kubelet 데몬이 실행되는 노드 그룹으로 되어있는 것을 확인할 수 있습니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/2025-01-06-19-59-36-image.png" alt="loading-ag-1737" data-align="center">{: .align-center}

## ETCD의 인증서

etcd는 쿠버네티스 클러스터에 대한 데이터를 key=value 형태로 저장하는 메모리 기반 데이터베이스로 Control-Plane에 스태틱 파드 형태로 배치되거나 클러스터 외부에 별도로 구성할 수 있습니다. 따라서 **자체 CA 루트 인증서와 서버 인증서를 갖고 있습니다.** 이들 모두 etcd CA에 의해 서명되어 있는 것을 확인할 수 있습니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/2025-01-06-20-06-31-image.png" alt="loading-ag-1749" data-align="center">{: .align-center}

## APIServer의 인증서

API 서버는 다른 컴포넌트로부터 접근 받는 서버 역할, 그리고 kubelet 데몬에 파드 실행을 명령하거나 etcd에 데이터를 저장하는 클라이언트 역할을 함께 수행합니다. 그래서 서버 인증서와 kubelet 접근을 위한 클라이언트 인증서, etcd 접근을 위한 클라이언트 인증서를 하나씩 보유하고 있습니다. 서버 인증서와 kubelet 클라이언트 인증서는 클러스터 CA에 의해 서명되어 있고 etcd 클라이언트 인증서는 etcd CA에 의해 서명되어 있습니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/2025-01-06-20-09-26-image.png" alt="loading-ag-1759" data-align="center">{: .align-center}

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/2025-01-06-20-10-33-image.png" alt="loading-ag-1763" data-align="center">{: .align-center}

쿠버네티스 API 서버를 호출할 때 사용하는 이름은 각 컴포넌트와 객체별로 다릅니다. 따라서 API 서버의 서버 인증서 SANs(Subject Alternative Name)는 API 서버로써 호출될 수 있는 여러 가지 이름과 IP 주소를 포함하고 있는 것이 특징입니다.

<img title="" src="../../images/2024-12-26-ssl_tls_kubernetes/2025-01-07-15-11-52-image.png" alt="loading-ag-396" data-align="center">{: .align-center}
