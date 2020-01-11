---
layout: post
title: "Tableau로 사무실 대시보드 구축하기(2)"
author: leclipse
description: "Tableau서버 설치하기"
date: 2020-01-11 12:00 +0900
tags: [tableau, dashboard]
comments: true
---

이 글은 "Tableau로 사무실 대시보드 구축하기" 시리즈의 두번째 글 입니다.

1. [Tableau 에 대한 간단한 소개와 구축 동기](https://8percent.github.io/2020-01-10/tableau-dashboard1)
2. Tableau Server 설치하기
3. Prep Builder 로 데이터 흐름 만들기
4. Tableau Desktop으로 Dashboard 만들기
5. 여러 개의 Dashboard를 자동으로 Rotation & Refresh 하기

------

글을 들어가기 전에 8퍼센트는 메인 인프라를 AWS에 두고 있습니다. 그리고 Tableau Server를 내부용으로 사용하기 때문에 Private Subnet에 설치했습니다. AWS에 Tableau Server를 설치에 대해서는 [Install Tableau Server on Amazon Web Services](https://help.tableau.com/current/server/en-us/ts_aws_welcome.htm) 를 참고하시면 좋습니다. 설치에는 3가지 방법이 있습니다.

- 직접 깔기
- AWS Quick Start 이용하기 - Cloud Formation Template으로 정해진 구성을 셋업하는 방식
- AWS Marketplace 이용하기 - EC2 AMI로 설치되는 방식

<center>
<figure>
<img src="/images/tableau-server-1.png" alt="views">
<figcaption>AWS Quick Start를 통해 설치되는 Tableau Server 구성</figcaption>
</figure>
</center>

이 중 AWS Quick Start는 단일 노드일 경우 Tableau Server를 Public subnet에 띄우기 때문에 탈락이고, AWS Marketplace의 경우 AMI가 최신으로 업데이트가 되고 있지 않았습니다. 따라서 직접 깔기로 결정하였습니다. 물론 Tableau Server가 어떻게 돌아가는지 궁금하기 때문에 어짜피 그냥 깔아볼 생각이었습니다.

### Instance의 생성

Instance를 생성하기 전에 [Selecting an AWS Instance Type and Size](https://help.tableau.com/current/server/en-us/ts_aws_virtual_machine_selection.htm)를 살펴봅시다. 요약하면 다음과 같습니다.

- T2 계열은 사용하지 마세요.
- 8 CPU cores (16 AWS vCPUs), 64GB RAM 을 권고합니다.

Instance Type 의 선택에 있어서는 [Tableau at the Speed of Amazon EC2](https://www.tableau.com/sites/default/files/whitepapers/tableau_whitepaper_aws_ec2_rvsd.pdf) 백서를 참고해 볼수 있습니다. 이 백서에서는 태블로 부하테스트 도구인 [TabJolt](https://github.com/tableau/tabjolt/)를 이용해서 다양한 AWS  Instance 들을 테스트 해두었습니다. 감사하게도 [public tableau](https://public.tableau.com/profile/tableau.core.product.marketing#!/vizhome/TableauattheSpeedofEC2-GettingtheMostoutofYourTableauServerinAWS_0/EC2InstanceComparison) 에서 직접 테스트 해 볼 수 있습니다. 저희는 사용자 수가 많지 않은 관계로 m5.2xlarge를 선택했습니다. Instance 타입이야 언제든지 바꿀 수 있으니 일단 가볍게 시작합니다.

주의할 점: EBS 용량을 충분히 확보하세요. EBS 용량을 충분히 확보하지 않고 Tableau Server를 설치한 이후에 동적으로 EBS 용량을 확장했더니, Tableau Server가 바보가 되어서 기술지원을 받았지만 해결 못하고 재설치를 하였습니다.

**Domain 설정**

태블로 서버와는 많은 중요한 데이터가 오가기 때문에 도메인을 할당하고 SSL설정을 하는 것이 좋습니다. 저는 Route53 에서 도메인 설정을 해주었습니다.

<center>
<figure>
<img src="/images/tableau-server-2.png" alt="views">
<figcaption>Route53 도메인 설정</figcaption>
</figure>
</center>

**Tableau Server 설치**

Ubuntu 18.04 에 설치합니다. 설치에 관련된 문서는 [Linux 기반 Tableau Server: 모든 사용자를 위한 설치 가이드](https://help.tableau.com/current/guides/everybody-install-linux/ko-kr/everybody_admin_intro.htm](https://help.tableau.com/current/guides/everybody-install-linux/ko-kr/everybody_admin_intro.htm)) 를 참고 하세요. 설치 후 약 15일 정도의 테스트 라이센스를 사용할 수 있습니다. 모든 기능들이 사용 가능하고, 충분한 사용자를 추가할 수 있으니 서버를 처음 설치하는 분들은 일단 설치를 해서 사용 해보고 라이센스 구매를 하시는 것도 방법이 되겠습니다.

설치 파일을 다운로드 받고 설치합니다.

<center>
<figure>
<img src="/images/tableau-server-3.png" alt="views">
<figcaption>설치파일 다운로드</figcaption>
</figure>
</center>

<center>
<figure>
<img src="/images/tableau-server-4.png" alt="views">
<figcaption>태블로 설치</figcaption>
</figure>
</center>

설치가 완료되면 tsm(Tableau Services Manager)를 초기화 합니다.

<center>
<figure>
<img src="/images/tableau-server-5.png" alt="views">
<figcaption>태블로 초기화</figcaption>
</figure>
</center>

이제 할당된 주소로 접속을 할 수 있습니다. Tableau Server는 기본적으로 80, 443 포트로 서비스가 제공되며 TSM 관리자 페이지는 8850 포트를 사용합니다. 웹페이지의 안내에 따라 설치를 완료하고 나면 다음과 같이 정상적으로 노드가 동작하는 것을 볼 수 있습니다!

<center>
<figure>
<img src="/images/tableau-server-6.png" alt="views">
<figcaption>태블로 서버 상태</figcaption>
</figure>
</center>

Tableau Server는 다양한 [프로세스](Tableau Server Processes) 를 사용하고 있습니다. 각각의 기능에 대해 간단하게 알아두면 문제가 생겼을 때 보다 손쉽게 대응할 수 있습니다.

- 게이트웨이: 태블로 서버의 웹인터페이스 입니다. 내부적으로는 apache를 띄웁니다.

- 응용 프로그램 서버(VizPortal): 웹페이지의 요청, REST API 요청등을 처리합니다.

- 대화형 마이크로서비스 컨테이너 / 비 대화형 마이크로서비스 컨테이너: Tableau Server가 내부적으로 사용하는 작은 서비스들을 관리하는 역할을 합니다.

  - tsm status -v 명령을 쳐보면 어떤것들을 관리하고 있는지 알 수 있습니다.

    <center>
    <figure>
    <img src="/images/tableau-server-7.png" alt="views">
    <figcaption>Interactive Microservice Container</figcaption>
    </figure>
    </center>

    <center>
    <figure>
    <img src="/images/tableau-server-8.png" alt="views">
    <figcaption>Non-Interactive Microservice Container</figcaption>
    </figure>
    </center>

- VizQL 서버: 쿼리를 실행하고 뷰를 그리는 핵심적인 역할을 합니다.

- 캐시 서버: 쿼리 캐시입니다. 내부적으로는 [redis](https://redis.io/) 를 사용합니다.

- 클러스터 컨트롤러: 주요 기능들을 모니터링하고 실패를 감지하고 재실행을 합니다.

- 검색 및 찾아보기: 서버에서 워크북을 검색하거나 필터를 거는 등의 검색에서 사용합니다. 내부적으로는 [solr](https://lucene.apache.org/solr/) 를 사용합니다.

- 백그라운더: 추출을 업데이트 하는 등 여러가지 종류의 Task 들을 백그라운드에서 처리해 줍니다.

- 데이터 서버: Tableau Server 가 보관하고 있는 데이터들을 제공하는 일을 합니다.

- 데이터 엔진: [Hyper](https://www.tableau.com/products/new-features/hyper) 라는 메모리 데이터엔진을 사용합니다. 이전에 비해 훨씬 빠르게 쿼리와 추출이 된다고 합니다. 이종 데이터베이스 간의 조인시에도 사용 됩니다.

- 파일저장소: 추출된 파일을 보관합니다.

- 리포지토리: 사용자정보, 퍼미션, 프로젝트, 메타데이터 등을 보관합니다. 내부적으로는 [postgresql](https://www.postgresql.org/)을 사용합니다.

- Tableau Prep 컨덕터: Tableau Prep Builder 에서 생성되어 Tableau Server에 올라간 흐름을 실행 시켜주는 역할을 합니다. Tableau Prep 컨덕터를 사용하려면 Data Management 라이센스가 필요합니다.

- 데이터에 질문: Tableau Server에서 제공하는 "데이터에 질문하기" 기능을 제공합니다.

- 탄력적 서버: "데이터에 질문하기"를 위한 인덱스를 제공 합니다. 내부적으로는 [elasticsearch](https://www.elastic.co/kr/)를 사용합니다.

- 메시징 서비스: Tableau Server 와 마이크로서비스간의 통신을 위해 사용됩니다. 내부적으로는 [ActiveMQ](https://activemq.apache.org/) 를 사용합니다.

- TSM 컨트롤러: Tableau  Server를 컨트롤할 수 있는 REST API를 제공합니다.

- 라이선스 서버: 태블로 라이센스를 관리합니다.

이렇게 많은 프로세스를 하나의  node에 띄우려다 보니 높은 사양이 필요할 수 밖에 없겠습니다. VizQL, Hyper 와 같이 태블로의 핵심적인 기능을 제외하면 오픈 소스를 적극적으로 활용하고 있습니다. 나머지 웹 기반으로 제공되는 서비스들은 Java 기반으로 작성이 되어 있습니다.

이제 서버의 설치와 구경을 완료했으니 tableau 서버에 접속해 봅니다.

<center>
<figure>
<img src="/images/tableau-server-9.png" alt="views">
<figcaption>Tableau Server 최초 접속하기</figcaption>
</figure>
</center>

admin 계정의 설정이 필요합니다. admin 에 대한 설정을 진행하면 정상적으로 접속이 가능합니다.

<center>
<figure>
<img src="/images/tableau-server-10.png" alt="views">
<figcaption>어드민 계정 생성하기</figcaption>
</figure>
</center>

Tableau Desktop 은 Tableau Server 와 수많은 데이터를 주고 받기 때문에 SSL 설정은 필수적입니다. TSM 관리자 페이지에서 SSL 설정을 할 수 있습니다.

<center>
<figure>
<img src="/images/tableau-server-11.png" alt="views">
<figcaption>SSL 설정하기</figcaption>
</figure>
</center>

정상적으로 완료되면 드디어 크롬에서 Tableau Server에 정상적으로 접속할 수 있습니다.

<center>
<figure>
<img src="/images/tableau-server-12.png" alt="views">
<figcaption>접속 성공!</figcaption>
</figure>
</center>

마지막으로 Tableau Server에서 특정 데이터베이스에 접근을 하기 위해서는 필요한 드라이버를 설치해 주어야 합니다. 저희는 내부적으로 PostgreSQL을 사용하기 때문에 관련 파일을 [다운로드](https://www.tableau.com/ko-kr/support/drivers) 받아 설치 해 주었습니다.

<center>
<figure>
<img src="/images/tableau-server-13.png" alt="views">
<figcaption>PostgreSQL 드라이버 설치하기</figcaption>
</figure>
</center>

이것으로 Tableau Server의 설치가 완료되었습니다. HA, 부하 분산등을 목적으로 node를 늘려야 한다면 추가 과정이 필요합니다. 하지만 저희는 크리티컬한 환경에서 사용하는것은 아니라서 single node로 작업을 완료하였습니다. 

다음은 Tableau Prep Builder을 이용해서 데이터 전처리 과정을 진행해 보겠습니다.


