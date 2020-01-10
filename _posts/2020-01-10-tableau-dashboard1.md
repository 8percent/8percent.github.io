---
layout: post
title: "Tableau로 사무실 대시보드 구축하기(1)"
author: bakyeono
description: "Tableau에 대한 간단한 소개와 사무실 대시보드를 구축하게 된 동기"
date: 2020-10-10 12:00 +0900
tags: [tableau, dashboard]
comments: true
---

최근에 Tableau 를 이용해서 회사 내에 TV 대시보드를 구축했습니다.

<center>
<figure>
<img src="/images/tableau-dashboard-1.jpg" alt="views">
<figcaption>사무실 TV 대시보드</figcaption>
</figure>
</center>

그 과정을 간단히 공유해 보려고 합니다. 글의 순서는 다음과 같이 진행 될 예정입니다.

1. Tableau 에 대한 간단한 소개와 구축 동기
2. Tableau Server 설치하기
3. Prep Builder 로 데이터 흐름 만들기
4. Tableau Desktop으로 Dashboard 만들기
5. 여러 개의 Dashboard를 자동으로 Rotation & Refresh 하기

위 글에서 기술적인 내용이 이미 문서로 잘 작성 되어 있는 경우에는 굳이 따로 설명을 하지 않고 링크를 걸 예정입니다. (해당 문서들은 잘 업데이트가 될 테니까요.) 자, 첫번째 부터 시작합니다.

----

### Tableau 에 대한 간단한 소개

8퍼센트는 BI(Business Intelligence) 도구로 Tableau를 3년째 사용하고 있습니다. 얼마전 라이센스를 갱신할 때 물어보니 국내에서 8퍼센트에만 적용되어 있는 라이센스가 있다고 하니 꽤 초기 부터 사용을 했나 봅니다. 그리고 최근 교육을 받으러 갔더니 큰 컨퍼런스룸을 가득 채울 정도로 사용자가 많아 졌습니다. 최근에는 국내에서 약 2,000개의 회사가 Tableau를 사용 하고 있다고 합니다.

Tableau는 간단하게 설명하면 여러 곳에 산재 해 있는 데이터를 잘 가져와서 효과적으로 시각화 하는 도구입니다. 다음과 같은 구성을 가지고 있습니다.

- Tableau Desktop: 가장 중심이 되는 도구로 데스크탑에서 데이터를 시각화 하는 역할을 합니다. 생각할 수 있는 대부분의 시각화를 의외로(?) 간단하게 구현할 수 있습니다. [Public Tableau Gallery](https://public.tableau.com/ko-kr/gallery/?tab=viz-of-the-day&type=viz-of-the-day)에 가보면 도대체 어떻게 만들었나 싶은 시각화 샘플들을 많이 만날 수 있습니다.

<center>
<figure>
<img src="/images/tableau-dashboard-2.png" alt="views">
<figcaption>Tableau Desktop</figcaption>
</figure>
</center>

- Prep builder: Tableau Desktop에 필요한 데이터 전처리를 담당합니다. 다양하게 분산되어 있는 데이터를 가공하고 조합해서 속칭 '원장' 데이터를 만들어 내는 일을 합니다.

<center>
<figure>
<img src="/images/tableau-dashboard-3.png" alt="views">
<figcaption>Prep builder</figcaption>
</figure>
</center>

- Tableau Server: Tableau Desktop과 Prep builder에서 생성된 파일들을 보관하고 웹서비스로 제공합니다. 추가로 스케쥴에 따라 데이터를 재생성 합니다.
- Tableau Online: Tableau Server가 SaaS 로 제공되는 형태입니다.



### TV 사무실 대시보드는 필요 한가?

8퍼센트에는 "통계왕"이라는 슬랙봇이 있습니다.  저녁 6시에는 각 조직별로 그 조직의 성과를 보여줄 수 있는 지표들을 보여주고, 저녁 9시에는 전체 채널에 회사의 성과를 보여줄 수 있는 지표를 전달합니다. 처음에는 가볍게 일별 취급액을 보여주는 것으로 시작 했지만 이제는 300개가 넘는 숫자가 매일 통계왕을 통해서 전달됩니다. 당일의 성과를 보여주는 것에는 여전히 훌륭한 방법입니다만 시간에 따른 변화를 보기에는 적절 하지는 않습니다.

웹을 통해서 제공되는 대시보드의 경우에는 적절한 시각화를 통해서 인싸이트를 전달할 수 있지만 정보를 Push 할 수는 없습니다. 정보는 제 때 전달 되어야 가치를 발휘하는데, 방문하지 않는 웹페이지를 만들어 두는 것만으로는 부족한 부분이 있었습니다. 그래서 회사에 주요 지표들을 다함께 보고 목표와 진행을 공유할 수 있도록 사무실 대시보드를 만들어야 겠다고 생각했습니다.

참고로 얼마전 당근마켓의 [인터뷰 글](http://www.ingray.net/2019/12/17/a-billion-dollar-advice-to-speed-up-your-team/)에서 운영 페이지에 관련 통계를 확인할 수 있는 기능을 같이 두는 것이 좋다는 글을 보았습니다.  이와 같은 문제를 해결할 수 있는 좋은 방법 중 하나라고 생각합니다.



### Tableau Server 를 설치하기 까지

초기에는 Tableau Desktop 만을 사용해서 분석하는 일들을 진행했습니다. 분석을 하면 이것을 공유하고자 하는 필요가 자연스레 생겨납니다. 이를 위해 처음에는 Tableau 파일을 공유 했습니다. 하지만 데이터를 보기만을 원하는 사람들에게는 Tableau는 너무 무겁고 사용하기 까지의 허들이 높았습니다.

다음으로 Tableau Online을 사용했습니다. Tableau Server를 깔게 되면 물론 비용도 들지만 서버를 유지관리 해 주어야 하는 일이 추가로 생기기 때문입니다. Tableau Online에 싸이트를 셋업 해 두고 자주 보는 사람은 직접 접속해서 보도록 하고, 그렇지 않은 사람들은 Confluence 에서 볼 수 있도록 [플러그인](https://marketplace.atlassian.com/apps/350103/tableau-for-confluence-pro?hosting=server&tab=overview](https://marketplace.atlassian.com/apps/350103/tableau-for-confluence-pro?hosting=server&tab=overview))을 설치해 두었습니다.  처음에는 문제 없이 사용했지만 데이터가 늘어나고, 더 다양한 대시보드를 요구하게 됨에 따라 속도가 문제가 되기 시작했습니다. 데이터를 확인해 보고 싶어서 방문을 했는데 데이터를 확인하기 까지 30초 이상이 걸리게 되니 사람들이 데이터에서 멀어지게 되었습니다. Tableau Online 의 경우 서버가 미국에 있기 때문에 데이터가 오가는 속도가 문제가 되었고, 우리가 필요한 만큼의 컴퓨팅 파워도 확보하지 못하였기 때문입니다.

또한 얼마전 모든 데이터베이스를 내부망으로 넣으면서 Tableau Online 서버에서 데이터베이스의 데이터를 가져가지 못하는 문제가 발생했습니다. 물론 이와 같은 경우를 대비해서 [Tableau Bridge](https://www.tableau.com/en-gb/products/tableau-bridge)가 제공되어서 이를 사용하였습니다. 하지만 고객의 거래내역과 같은 내부 민감 데이터를 외부로 보내서 분석할 수는 없었고, Prep Conductor를 사용해 Tableau로 데이터를 모으고자 하니, Tableau Online의 100GB의 데이터 용량이 한계로 느껴졌습니다.

이제는 Tableau Server 를 직접 설치해서 관리를 할 수 밖에 없구나라는 생각이 들어 삽을 들었습니다.

자, 다음글에는 Tableau Server 설치과정을 다뤄보겠습니다. 


