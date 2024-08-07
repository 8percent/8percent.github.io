---
layout: post
title: "디자인 시스템, 디자인과 코드의 간극 줄이기"
author: minjeong
date: 2024-07-15
tags: [frontend, design system, eds, figma]
comments: true
description: 디자인 시스템과 개발 및 적용 과정의 문제점 분석과 Figma Plugin API를 활용한 도구 소개
---

안녕하세요, 에잇퍼센트 웹 프론트엔드 팀의 김민정입니다. 프론트엔드 팀은 사내 EDS(Eight Design System)를 적용한 웹 서비스를 효율적으로 구현하기 위해 팀 내 UI 라이브러리인 `eds-wc`를 제공하고 있습니다. 디자인 팀과 프론트엔드 팀이 협업하여 디자인 시스템을 구현하는 과정에서는 고려할 사항이 매우 많습니다. 특히, 명세에 대해 디자이너와 개발자 간 이해의 차이로 인한 반복적인 커뮤니케이션은 대부분 경험하였을 것입니다. 또한 UI 라이브러리와 디자인이 완벽히 일치하지 않는 문제는 한 측에서 노력해서 해결되기 어렵습니다.

이 글에서는 다음과 같은 내용을 소개합니다.

1. 디자인 시스템 개발과 서비스 개발 단계에서 발생하는 문제점

   - 디자이너와 개발자의 커뮤니케이션 비용이 증가하는 이유
   - 디자인 시스템 Figma와 Code의 간극으로 인한 피로도

2. 문제점 개선 방안

   - 디자이너와 개발자의 협업을 통한 명확한 인터페이스 정의
   - 디자인 시스템 Figma 규약 추가: 디자인 시스템 Figma로부터 모든 정보 얻기
   - 디자인 시스템 Figma를 Code로 추상화하기

3. 사용 기술과 도구 소개

   - Figma Plugin API
   - EDS Lint와 EDS Codegen

## 문제점 파악

디자인 시스템 개발부터 이를 사용하는 서비스 개발까지의 워크플로우는 다음과 같습니다.

<img width="920" alt="워크플로우" src="/images/frontend-eds-improvement-1.png">

> **용어 안내**
>
> - **EDS Code**: EDS 컴포넌트를 코드로 구현한 UI 라이브러리 (`eds-wc`)
> - **EDS Figma**: EDS 컴포넌트를 Figma 컴포넌트 세트로 구현한 Figma 컴포넌트 라이브러리
> - **Service Figma**: EDS Figma 컴포넌트를 조합하여 서비스 UI를 구현한 Figma 파일

### 디자인 시스템 개발 단계

> **요약**
>
> _디자인 시스템 개발에서 커뮤니케이션 비용이 가장 많이 드는 단계는 개발 단계입니다. 이는 디자이너와 개발자가 컴포넌트의 인터페이스와 형태를 이해하는 배경이 다르기 때문입니다. 또한, 커스텀 속성의 일관성이 부족하면 명세 재확인과 수정 요청이 추가됩니다. 결국 개발자가 속성을 임의로 변경하여 EDS Figma와 EDS Code의 간극이 생길 수 있습니다._

디자인 시스템 개발은 다음 단계를 거칩니다.

1. **디자인**: 디자이너가 EDS 컴포넌트의 명세를 생성합니다.
2. **개발**:

   1. 명세 변경이 필요한 경우 프론트엔드 개발자가 재요청합니다.
   2. 명세 변경이 필요 없는 경우 명세를 기반으로 프론트엔드 개발자가 컴포넌트를 개발합니다.

3. **리뷰**:

   1. 컴포넌트 리뷰를 통과하지 못하면 다시 개발 또는 디자인 단계로 돌아갑니다.
   2. 컴포넌트 리뷰를 통과하면 EDS Figma와 EDS Code가 각각 디자인 팀과 프론트엔드 팀에 공유됩니다.

위 단계 중 커뮤케이션 비용이 가장 많이 발생한 단계는 어디일까요? 실질적인 피드백을 하는 리뷰 단계가 아닌, 개발 단계입니다. 개발 단계에서 개발자는 전달받은 컴포넌트 명세의 문제점 파악한 후, 디자이너에게 반복하여 수정을 요청하기 때문입니다. 리뷰 단계에서는 이미 여러 차례 수정이 된 명세와 개발 단계에서 자체적인 버그 픽스를 거친 상태였으므로, 상대적으로 많은 비용이 발생하지 않았습니다.

명세가 반복되어 수정되는 이유는 컴포넌트의 인터페이스와 형태에 대한 디자이너와 개발자의 이해 차이 때문입니다. 초기 명세 작성 시 디자이너는 컴포넌트가 가질 속성을 모두 예측하기 어렵습니다.

<div style="display: flex; justify-content: center; flex-wrap: wrap;">
  <figure>
    <img height="280" alt="예시 이미지" src="/images/frontend-eds-improvement-2.png">
    <figcaption style="text-align: center; font-size: 16px">
      <code>disabled</code> + <code>checked</code> 속성 조합이 있음
    </figcaption>
  </figure>
  <figure>
    <img height="280" alt="예시 이미지" src="/images/frontend-eds-improvement-3.png">
    <figcaption style="text-align: center; font-size: 16px">
      <code>disabled</code> + <code>checked</code> 속성 조합이 누락
    </figcaption>
  </figure>
</div>

반면, 개발자는 필요한 UI 형태와 구조에 대해 예측하기 어렵습니다. 이 문제점에 대해서는 [2023 FEConf에서 하태영님이 발표하신 "크로스 플랫폼 디자인 시스템, 1.5년의 기록" 영상 중 "의사소통의 순환참조"](https://youtu.be/obQvttzgSzY?feature=shared&t=667) 내용을 참고하였으며, 해당 영상에서 잘 설명하고 있어 참고하시면 좋을 것 같습니다.

또한 Figma는 컴포넌트 속성에 대한 가이드라인이 없기 때문에, 디자인 팀 내 명확한 명명 규칙이 없다면 동일한 속성이 일관되지 않게 정의될 수 있습니다.

<div style="display: flex; justify-content: center; flex-wrap: wrap;">
  <figure>
    <img height="280" alt="예시 이미지" src="/images/frontend-eds-improvement-2.png">
    <figcaption style="text-align: center; font-size: 16px">
      비활성화 상태로 <code>usable="unable"</code> 속성을 사용
    </figcaption>
  </figure>
  <figure>
    <img height="280" alt="예시 이미지" src="/images/frontend-eds-improvement-3.png">
    <figcaption style="text-align: center; font-size: 16px">
      비활성화 상태로 <code>state="disabled"</code> 속성을 사용
    </figcaption>
  </figure>
</div>

웹에서 표준화되어 있는 속성들은 개발자들이 쉽게 이해할 수 있습니다. 그러나 커스텀 컴포넌트에서만 사용하는 속성들을 일관되지 않게 정의하면 추가적인 명세 재확인과 커뮤니케이션이 필요합니다.

<div style="display: flex; justify-content: center; flex-wrap: wrap;">
  <figure>
    <img width="330" alt="예시 이미지" src="/images/frontend-eds-improvement-4.png">
    <figcaption style="text-align: center; font-size: 16px">
      icon 관련 <code>variant</code>를 사용
    </figcaption>
  </figure>
  <figure style="display: flex; flex-direction: column; justify-content: center;">
    <img width="330" alt="예시 이미지" src="/images/frontend-eds-improvement-5.png">
    <figcaption style="text-align: center; font-size: 16px">
      <code>icon</code>, <code>icon 위치</code>속성을 사용
    </figcaption>
  </figure>
</div>

결국 커뮤니케이션 비용 증가로 인해 개발 단계에서 개발자가 임의로 속성을 변경하는 경우가 빈번히 발생하며, 최종 결과물인 EDS Figma과 EDS Code의 간극이 발생하였습니다.

### 서비스 개발 단계

> **요약**
>
> _서비스 개발 과정에서 디자이너는 EDS Figma를 활용하여 Service Figma를 구축하고, 개발자는 이 명세를 기반으로 EDS Code를 사용하여 서비스를 개발합니다. 개발자는 이전 단계에서 발생한 차이로 인해 명세 확인에 더 많은 시간을 할애해야 합니다._

서비스 개발 단계에서는 다음 단계를 거칩니다.

1. **디자인**: 디자이너가 EDS Figma를 사용하여 서비스 개발을 위한 Service Figma를 구현합니다.
2. **개발**: 프론트엔드 개발자가 Service Figma 명세를 기반으로 EDS Code를 사용하여 서비스를 개발합니다.

이전 디자인 시스템 단계에서 발생한 간극으로 인해, 개발자가 각각의 명세를 확인해야 하므로 피로도가 증가합니다. 이는 디자인 시스템을 직접 개발하지 않은 개발자일수록 더욱 두드러집니다.

예시로, 하단의 EDS Button 컴포넌트는 `넓이 확장 유무 [true, false]`, `테마 ["primary", "secondary", "tertiary"]`, `크기 ["xs", "s", "m", "l"]`를 설정할 수 있는 속성을 가지고 있습니다. EDS Code에서는 `fullWidth`, `variant`, `size` 속성으로 구현되었습니다. EDS Figma에서는 `size` 속성, `Primary라는 컴포넌트 명칭`, `size별 Section 기능`으로 구별되었습니다.

<figure style="display: flex; flex-direction: column; align-items: center;">
  <img width="600" alt="예시 이미지" src="/images/frontend-eds-improvement-6.png">
  <figcaption style="text-align: center; font-size: 16px">
    디자인 시스템 비개발자: <code>size=”custom”</code>...?
  </figcaption>
</figure>

```html
<!-- 실제 구현해야하는 코드 -->
<eds-button
  fullWidth
  variant="primary"
  size="l"
  text="증명서 발급"
></eds-button>
```

Figma 컴포넌트 라이브러리는 Figma Page, Section 명칭에 따른 계층화를 지원하며, 해당 기능에 접근이 가능한 디자인 팀에서는 효율적이며 선호하는 방식이었습니다. 하지만 Figma 컴포넌트 라이브러리에 접근 권한이 없으면 Page, Section 명칭에 따른 계층화가 지원되지 않았습니다. 접근 권한이 없는 개발자는 Button의 Section 기능으로 나눠진 `size`를 알 수 없습니다. EDS Figma의 `size="custom"` 속성과 값이 `크기` 속성을 나타내지 않는다는 것을 인지한 후, EDS Figma 상으로 정보가 제공되지 않는 속성 정보를 알기 위해 별도의 EDS Figma 텍스트 문서를 반복적으로 확인해야 하는 번거로움이 있었습니다.

## 문제점 개선 방안

워크플로우에서 파악된 문제점을 개선하기 위해, 다음 3가지 방안을 적용하였습니다.

1.  **신규 EDS 컴포넌트 명세 작성 시 인터페이스를 협의합니다.**

    초기 명세를 작성할 때 디자이너와 개발자가 함께 정확한 인터페이스를 협의합니다. 이는 컴포넌트의 기능, 상태, 속성을 명확히 이해하고 일관성 있게 문서화하는 데 중요합니다.

2.  **EDS Figma에 대한 규약을 추가합니다.**

    개발자는 EDS Figma로부터 EDS Code 필요한 모든 정보를 얻을 수 있어야 합니다. 그렇다면 디자이너는 EDS Figma로 어디까지 정보를 제공해야할까요? 그 기준을 Figma Plugin API로 접근할 수 있는 EDS [ComponentSetNode](https://www.figma.com/plugin-docs/api/ComponentSetNode/)와 [InstanceNode](https://www.figma.com/plugin-docs/api/InstanceNode/#getmaincomponentasync)로 부터 EDS Code 구현을 위한 데이터를 추출이 가능한 정도로 정의하였습니다. ComponentSetNode 생성 시 EDS Code 구현을 위해 필요한 세 가지 최소 조건은 다음과 같습니다.

    1. 컴포넌트의 기능은 EDS Figma ComponentSetNode의 이름 또는 property 기능으로 구분되어야합니다.

       ```
        GOOD 🟢 ) <EdsButton> + property(size: ['sm', 'md', 'lg'])
        BAD ❌ ) Figma Section 별로 size 분리
       ```

    2. 생성되는 EDS Figma ComponentSetNode의 이름 내에 EDS 컴포넌트 명칭에 대한 데이터가 있어야 합니다.
       이는 다른 Figma ComponentSetNode와 구별을 위함입니다.

       ```md
       GOOD 🟢 ) EdsButton, EDS Button, <EdsButton>, <EDS Button>
       BAD ❌ ) Primary, Atom, Button
       ```

    3. 동일한 기능을 나타내기 위해 사용되는 property의 값은 통일되어야 합니다. (property의 enum화)

       ```md
       GOOD 🟢 )
       <EdsButton> + property(size: ['sm', 'md', 'lg'])
       <EdsTextfield> + property(size: ['sm', 'md', 'lg'])
       BAD ❌ )
       <EdsButton> + property(size: ['small', 'medium', 'large'])
       <EdsTextfield> + property(size: ['sm', 'md', 'lg'])

       GOOD 🟢 )
       <EdsCheckbox> + property(disabled: [true, false])
       <EdsToggle> + property(disabled: [true, false])
       BAD ❌ )
       <EdsCheckbox> + property(disabled: [true, false])
       <EdsToggle> + property(usable: ['able', 'disable'])
       ```

3.  **Figma Plugin API를 통해 EDS Figma를 EDS Code 수준으로 추상화된 코드를 제공합니다.**

    개발자는 추가적인 명세 확인 없이 EDS Figma를 EDS Code로 구현할 수 있어야 합니다. EDS Figma에서 디자인 속성으로만 정보를 제공하는 것은 한계가 있습니다. 규격화된 EDS Figma를 기반으로 EDS Code 수준으로 추상화된 코드를 Figma 환경에서 제공합니다.

이 과정에서 중요한 역할을 하는 것이 바로 **Figma Plugin API**입니다. 디자인과 개발 간의 간극을 효과적으로 줄이기 위해 Figma Plugin API를 활용하여 **EDS Lint**, **EDS Codegen**과 같은 도구를 개발할 수 있었습니다. 사용 기술과 도구에 대해 간단히 소개합니다.

## 사용 기술

### Figma Plugin API

Figma는 개발자가 Figma 환경에서 기능을 확장하여 사용할 수 있도록 [Plugin API](https://www.figma.com/plugin-docs/)를 제공하고 있습니다. 이 글에서는 기술적인 세부 사항을 전문적으로 다루지 않지만, Figma Plugin API의 기본 개념을 간단하게 소개합니다. 기존에도 효율적인 디자인과 개발을 위해 [다양한 플러그인](https://www.figma.com/community/category/development?resource_type=plugins)들이 제공되고 있습니다.

Figma Plugin은 플러그인이 안전하게 실행될 수 있도록 하기 위해 샌드박스 구조를 채택하고 있습니다. 이 두 환경은 분리되어 있으며, 메세지를 통해 정보를 주고 받게 됩니다.

<figure style="display: flex; flex-direction: column; align-items: center;">
  <img width="800" alt="예시 이미지" src="/images/frontend-eds-improvement-7.png">
  <figcaption style="text-align: center; font-size: 16px">
    Figma Plugin API 구조
  </figcaption>
</figure>

- **main 스레드 환경**: 플러그인 API를 통해 Figma의 주요 기능에 접근합니다. (ex. Figma SeneNode, Figma Selection 등)
- **iframe 환경**: 브라우저 API를 사용할 수 있으며 ui 구현을 담당합니다.

## 도구 소개

### EDS Lint

EDS Lint는 Figma 플러그인으로, 생성된 EDS Figma의 ComponentSetNode가 규약에 맞게 정의되었는지 검사합니다.

- 디자인 시스템 개발 단계에서 EDS Figma의 ComponentSetNode의 구조와 속성의 일관성을 유지할 수 있습니다.
- 디자이너들이 Figma에서 작업할 때 실시간으로 피드백을 제공하여, 문제를 사전에 방지하고 수정할 수 있도록 돕습니다.

프로토타입 버전으로 개발한 EDS Lint의 기능은 크게 다음과 같습니다.

1. **신규 EDS ComponentSetNode 및 Variant 생성 기능**

   <div style="display: flex; justify-content: center; flex-wrap: wrap;">
     <figure style="margin-left: 14px; margin-right: 14px">
       <img height="320" alt="예시 이미지" src="/images/frontend-eds-improvement-8.png">
     </figure>
     <figure style="margin-left: 14px; margin-right: 14px">
       <img height="320" alt="예시 이미지" src="/images/frontend-eds-improvement-9.png">
     </figure>
   </div>

   - 신규 ComponentSetNode 명칭을 입력하고 Create 버튼을 클릭 시, ComponentSetNode가 생성됩니다.
   - 사전 정의 된(defined) 공통 속성의 체크박스를 클릭 시, 해당 속성과 값을 가진 Variant가 자동 생성됩니다.

2. **기존 EDS ComponentSetNode Lint 기능**

   <div style="display: flex; justify-content: center; flex-wrap: wrap;">
       <img width="400" alt="예시 이미지" src="/images/frontend-eds-improvement-10.png">
   </div>

   - 필수 속성이 사용되지 않았을 경우 에러를 표기합니다.
   - 사전 정의 된(defined) 속성값 외 사용에 대한 에러를 표기합니다.
   - 사전 정의 되지 않은(not defined) 속성 사용에 대한 정보를 제공합니다.

### EDS Codegen

EDS Codegen은 EDS Figma의 ComponentSet를 EDS Code로 변환해 주는 도구입니다. 개발자는 추가적인 명세 확인 없이 EDS Figma로부터 EDS Code를 제공받을 수 있습니다.

**EDS Figma ComponentSet Instance를 EDS Code로 변환하는 기능**

   <div style="display: flex; justify-content: center; flex-wrap: wrap;">
     <figure style="margin-left: 14px; margin-right: 14px">
       <img height="620" alt="예시 이미지" src="/images/frontend-eds-improvement-11.png">
     </figure>
     <figure style="margin-left: 14px; margin-right: 14px">
       <img height="620" alt="예시 이미지" src="/images/frontend-eds-improvement-12.png">
     </figure>
   </div>

- EDS Figma CompoentSet의 Instance를 클릭할 시, `eds-wc`에서 지원하는 HTML, React, Vue 문법으로 Code를 제공합니다.

## 마치며

이번 글에서는 디자인 시스템과 서비스 개발 단계에서 발생하는 문제점들을 분석하고, 문제를 해결하기 위한 여러 방안을 살펴보았습니다. 규격화된 Figma ComponentSetNode를 기반으로 Figma Plugin API를 통해 더 효율적이고 일관된 작업 환경을 위한 도구들을 만들 수 있었습니다. 결론적으로, 디자인 시스템의 성공적인 구현을 위해서는 팀 간의 원활한 커뮤니케이션과 명확한 명세 정의가 필수적입니다. 에잇퍼센트 프론트엔드 팀은 앞으로도 디자인과 개발 간의 간극을 줄여 나가기 위해 최선을 다하겠습니다.
