---
layout: single
title:  "비용 효율적인 프라이빗 서브넷 Outbound 트래픽 처리(Feat. NAT 게이트웨이, VPC 엔드포인트)"
categories: AWS
tag: [AWS]
author_profile: false
sidebar:
    nav: "docs"
---

최근 Terraform을 사용하여 EKS 기반 서비스 인프라 모듈을 직접 개발해보던 중 프라이빗 클러스터가 외부와 통신할 수 있는 비용 효율적인 방식을 고민하게 되었습니다. 이 때 AWS Load Balancer Controller나 다른 서드 파티 도구를 설치하는 등의 경우 NAT 게이트웨이를 사용하고, EKS API, S3 등 AWS 서비스 트래픽을 처리하기 위해 VPC 엔드포인트를 사용할 수 있는데요. **VPC 엔드포인트가 NAT 게이트웨이보다 과금이 5배 이상 저렴하기 때문에** VPC 엔드포인트만 사용할 수 있는 게 가장 이상적이겠지만 인터넷을 통해 외부 Source와 통신해야 하는 경우가 분명히 있을 것이기 때문에 웬만한 인스턴스 기반 서비스 아키텍쳐에서는 NAT 게이트웨이 사용이 필연적이지 않을까 생각합니다.

## NAT 게이트웨이

NAT 게이트웨이는 프라이빗 서브넷에 NAT 서비스를 제공할 수 있는 AWS 관리형 게이트웨이 리소스입니다. <u>물리적으로 Public Subnet에 생성되고 특정 가용 영역에 종속되기 때문에 고가용성을 위해 가용 영역별로 하나씩 배치할 수 있습니다.</u> 라우팅 테이블을 통해 프라이빗 서브넷을 NAT 게이트웨이에 연결할 수 있습니다.

요금은 포스팅 작성일 기준으로 미국 동부 버지니아 북부(us-east-1) 기준 시간당 0.045 USD, 1GB당 0.045 USD이고, 서울 리전(ap-northeast-2)을 기준으로는 시간당 0.059USD, 1GB당 0.059 USD로 조금 더 비싼 편입니다.

NAT 게이트웨이는 NAT 서비스를 통해 **AWS 서비스 접근을 포함한 모든 Outbound 트래픽을 외부 인터넷을 통해 처리할 수 있습니다.**

## VPC 엔드포인트

VPC 엔드포인트는 프라이빗 서브넷에서 AWS 서비스에 접근할 수 있도록 하는 엔드포인트로, **NAT 게이트웨이와 달리 외부 인터넷을 거치지 않고 사설 네트워크를 통해 접근할 수 있다**는 특징이 있습니다. VPC 엔드포인트는 과금의 유무와 접근 가능한 서비스 유형에 따라 인터페이스 엔드포인트와 게이트웨이 엔드포인트로 나뉩니다.

### 인터페이스 엔드포인트

인터페이스 엔드포인트는 AWS PrivateLink라는 서비스를 기반으로 동작하며 프라이빗 서브넷에 별도의 ENI를 생성하여 AWS 서비스에 접근할 수 있도록 합니다. <u>특정 프라이빗 서브넷에 종속되기 때문에 프라이빗 서브넷이 존재하는 가용 영역 당 하나씩 생성해야 하고 보안 그룹 적용이 필수입니다.</u> Kinesis나 SNS, EKS API, Lambda 등 대부분의 AWS 서비스에 대한 접근을 지원하고 <u>Site to Site VPN이나 Direct Connect를 통한 하이브리드 클라우드 방식과 다른 VPC나 리전에서의 접근 엔드포인트로 사용할 수도 있습니다.</u>

요금은 포스팅 작성일 기준으로 미국 동부 버지니아 북부(us-east-1) 기준 시간당 0.01 USD, 1GB당 0.01 USD이고, 서울 리전(ap-northeast-2)을 기준으로는 시간당 0.013 USD, 1GB당 0.01 USD로 시간당 요금이 조금 더 비싼 편입니다.

### 게이트웨이 엔드포인트

게이트웨이 엔드포인트는 라우팅 테이블에 타겟으로 설정하여 접근할 수 있는 VPC 엔드포인트로 **S3와 DynamoDB 접근에 사용합니다.** 게이트웨이 엔드포인트는 **별도의 과금 없이 무료로 사용할 수 있습니다.**

## 비용 효율성 비교

앞서 서술했다시피 게이트웨이 엔드포인트는 과금이 없기 때문에 꼭 사용하여 S3와 DynamoDB 트래픽을 처리하도록 하고, EKS 프라이빗 노드 그룹이 NAT 게이트웨이만 사용하는 방식과 NAT 게이트웨이 + 인터페이스 엔드포인트 조합을 사용하는 방식을 다음과 같은 가정을 바탕으로 비교해 보겠습니다.

- 월간 비용 측정

- 서울 리전(ap-northeast-2)을 기준으로 요금 책정

- 최소한의 고가용성을 위해 2개의 가용 영역(AZ)에 프라이빗 서브넷을 생성

- NAT 게이트웨이와 인터페이스 엔드포인트를 함께 사용할 경우 다음과 같이 트래픽을 나눠 처리
  
  <img title="" src="../../images/2025-07-13-eks_nat_enp/523f06708e6263c617f26e22d25a6c6a8fb9ae21.png" alt="loading-ag-2673" data-align="center">{: .align-center}

- 공식 문서에서 권장하는 EKS 프라이빗 클러스터를 위한 인터페이스 엔드포인트 중 다음 8가지 사용
  
  - com.amazonaws.`region-code`.ec2
  
  - com.amazonaws.`region-code`.ecr.api
  
  - com.amazonaws.`region-code`.ecr.dkr
  
  - com.amazonaws.`region-code`.elasticloadbalancing
  
  - com.amazonaws.`region-code`.logs
  
  - com.amazonaws.`region-code`.sts
  
  - com.amazonaws.`region-code`.eks-auth
  
  - com.amazonaws.`region-code`.eks

## 1. 월간 총 8GB

**AWS 요금 계산기(Pricing calculator)** 를 사용하여 월간 총 8GB의 트래픽을 2개의 NAT 게이트웨이로 나눠 사용해서 처리할 때의 비용을 계산해보면

<img title="" src="../../images/2025-07-13-eks_nat_enp/66e518e0dedec8b6a754f030b2dbfecafc2ff68f.png" alt="loading-ag-2760" data-align="center">

NAT 게이트웨이가 4GB, 인터페이스 엔드포인트가 4GB를 처리하면

<img title="" src="../../images/2025-07-13-eks_nat_enp/694b6721910d59fcf9994d96cf8bd7e7885f1f8b.png" alt="loading-ag-2771" data-align="center">

<img title="" src="../../images/2025-07-13-eks_nat_enp/e6012fb778097cdddcf740af0d2581476508909a.png" alt="loading-ag-2778" data-align="center">

NAT 게이트웨이만 사용하면 **86.62 USD**, 

NAT 게이트웨이 + 인터페이스 엔드포인트 조합에서는 **86.38 USD + 113.92 USD = 200.30 USD**로

NAT 게이트웨이만 사용했을 때의 비용이 훨씬 저렴한 것을 확인할 수 있습니다.

일부 트래픽을 인터페이스 엔드포인트로 분산했음에도 불구하고 비용이 더 비싼 이유는 **NAT 게이트웨이와 인터페이스 엔드포인트의 누적되는 시간당 비용이 많은 부분을 차지**하기 때문입니다. 두 가지 다 사용 시간당 비용은 각각 86.14 USD, 113.88 USD가 나왔지만 GB당 과금되는 트래픽 처리 비용은 각각 0.24 USD, 0.04 USD밖에 나오지 않았죠.

## 2. 월간 총 800GB

월간 총 800GB의 트래픽을 NAT 게이트웨이만 사용하여 처리하면

<img title="" src="../../images/2025-07-13-eks_nat_enp/e1173cdb8669d3ea99c287b899c13e9c277412e3.png" alt="loading-ag-2911" data-align="center">

NAT 게이트웨이가 400GB, 인터페이스 엔드포인트가 400GB를 처리하면

<img title="" src="../../images/2025-07-13-eks_nat_enp/f41664df3e71488d8674e95a8ff4f074b4d657e5.png" alt="loading-ag-2920" data-align="center">

<img title="" src="../../images/2025-07-13-eks_nat_enp/e0b74f5e6e0caa83e2ef2c59c64264f4837a4646.png" alt="loading-ag-2927" data-align="center">

NAT 게이트웨이만 사용하면 **133.94 USD**,

NAT 게이트웨이 + 인터페이스 엔드포인트 조합에서는 **109.74 USD + 117.88 USD = 227.62 USD**로

Outbound 트래픽 규모가 수백 GB 수준으로 증가해도 여전히 NAT 게이트웨이만 사용했을 때의 비용이 훨씬 저렴합니다.

## 3. 월간 총 8TB

이번에는 TB 단위까지 규모가 증가했다고 가정하고 월간 총 8TB의 트래픽을 NAT 게이트웨이만 사용하여 처리하면

<img title="" src="../../images/2025-07-13-eks_nat_enp/dbf9932cc13ba93754e018c856e64e77c4fa5589.png" alt="loading-ag-2952" data-align="center">

NAT 게이트웨이가 4TB, 인터페이스 엔드포인트가 4TB를 처리하면

<img title="" src="../../images/2025-07-13-eks_nat_enp/a9581537917802c6b37759dc6a771c3826e5cb54.png" alt="loading-ag-2963" data-align="center">

<img title="" src="../../images/2025-07-13-eks_nat_enp/618f83bb5df65d74e2da2f1041c54d74ad461b0e.png" alt="loading-ag-2970" data-align="center">

NAT 게이트웨이만 사용하면 **569.46 USD**,

NAT 게이트웨이 + 인터페이스 엔드포인트 조합에서는 **327.80 USD + 154.84 USD = 482.64 USD**로

<u>이번에는 NAT 게이트웨이와 인터페이스 엔드포인트를 함께 사용했을 때의 비용이 좀 더 저렴한 것을 확인할 수 있습니다.</u>

이처럼 월간 Outbound 트래픽 처리량이 TB 수준으로 증가하면 트래픽 처리 비용이 시간당 비용을 초과하여 다른 결과가 나타난 것을 확인할 수 있습니다.

## 결론

이번 테스트를 계기로 **두 유형의 리소스의 시간당 과금을 능가하는 대규모 Outbound 트래픽이 발생하지 않는 이상 NAT 게이트웨이만 사용하는 방식이 더 비용 효율적임을 알 수 있었습니다.** 

운영 서비스의 성격에 따라 본 포스팅에서의 가정과 다르게 NAT 게이트웨이를 통한 인터넷 트래픽이 더 많거나 VPC 엔드포인트를 통한 AWS 서비스 트래픽이 더 많은 비중을 차지할 수 있고, 가용 영역 수와 사용하는 인터페이스 엔드포인트 수도 다를 수 있습니다. 그래서 **정확한 예상 과금을 확인하고 비교하려면 AWS 요금 계산기를 적극 활용해볼 수 있습니다.**

NAT 게이트웨이만 사용하는 방식이 더 비용 효율적일 순 있지만 **민감 정보를 주고받는 등 보안이 중요한 운영 환경에서는 비용 효율성과 관계 없이 사설 네트워크를 거쳐 통신하는 인터페이스 엔드포인트를 함께 사용해야 할 수 있습니다.**

마지막으로 **S3와 DynamoDB로의 트래픽은 게이트웨이 엔드포인트를 통해 처리하여 보안을 강화함과 동시에 비용을 절감해야 합니다.** S3와 DynamoDB는 대부분의 서비스 아키텍쳐에서 사용하며 많은 트래픽이 발생하기 때문에 게이트웨이 엔드포인트 사용에 따르는 비용 절감 효과가 굉장히 클 것입니다.

#### References

[인터넷 액세스가 제한된 프라이빗 클러스터 배포 - Amazon EKS](https://docs.aws.amazon.com/ko_kr/eks/latest/userguide/private-clusters.html)

[AWS 요금 계산기(AWS Pricing Calculator)](https://calculator.aws/#/)
