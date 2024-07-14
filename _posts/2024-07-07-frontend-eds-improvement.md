---
layout: post
title: "디자인 시스템, 디자인과 코드의 간극 줄이기"
author: minjeong
date: 2024-07-07
tags: [frontend, design system, eds, figma]
comments: true
description: 디자인 시스템과 개발 및 적용 과정의 문제점 분석과 Figma Plugin API를 활용한 도구 소개
---

# 디자인 시스템, 디자인과 코드의 간극 줄이기

안녕하세요, 에잇퍼센트 Web Frontend팀의 김민정입니다. 프론트엔드팀은 EDS(Eight Design System)을 사용하여 효율적인 서비스 개발을 진행하고 있으며, 이를 위해 팀 내 UI 라이브러리 `eds-wc`를 제공하고 있습니다. 디자인팀과 프론트엔드팀이 협업하여 디자인 시스템을 구현하는 과정은 많은 고려 사항이 필요합니다. 특히, 디자이너와 개발자 간의 명세 이해 차이와 커뮤니케이션 비용 증가가 주요 문제로 지적됩니다.

이 글에서는 디자인 시스템의 개발 단계와 서비스 개발 단계에서 발생하는 구체적인 문제점을 분석하고, 이를 해결하기 위한 개선 방안을 공유합니다. 디자인 시스템의 핵심 구성 요소인 디자인 시스템 Figma 컴포넌트 라이브러리와 코드 UI 라이브러리 역할을 중심으로 문제점을 파악합니다. 또한, Figma Plugin API를 활용하여 디자인과 개발 간의 간극을 줄이고, 효율적인 워크플로우를 구현하기 위한 도구들을 소개합니다.

## 워크플로우에서 발생한 문제점

디자인 시스템 개발부터 이를 사용하는 서비스 개발까지의 워크플로우는 아래와 같습니다.

(TODO: 이미지 첨부)

> **용어 안내**
>
> - EDS Code: EDS 컴포넌트를 코드로 구현한 UI 라이브러리 (=eds-wc)
> - EDS Figma: EDS 컴포넌트를 Figma Component Set로 구현한 Figma 컴포넌트 라이브러리
> - Service Figma: EDS Figma 컴포넌트를 조합하여 서비스 UI를 구현한 Figma 파일

### 디자인 시스템 개발 단계

> **요약**
>
> 디자인 시스템 개발에서 커뮤니케이션 비용이 가장 많이 드는 단계는 개발 단계입니다. 이는 디자이너와 개발자가 컴포넌트의 인터페이스와 형태를 이해하는 방식이 다르기 때문입니다. 또한, 커스텀 속성의 일관성이 부족하면 명세 재확인과 수정 요청이 추가됩니다. 결국 개발자가 속성을 임의로 변경하여 EDS Figma와 EDS Code의 간극이 생길 수 있습니다.

디자인 시스템 개발 단계에서는 다음 단계를 거칩니다.

1. **디자인**: 디자이너가 EDS 컴포넌트의 명세를 생성합니다.
2. **개발**:

   1. 명세 변경이 필요한 경우 프론트엔드 개발자가 재요청합니다.
   2. 명세 변경이 필요 없는 경우 명세를 기반으로 컴포넌트를 개발합니다.

3. **리뷰**:

   1. 컴포넌트 리뷰를 통과하지 못하면 다시 개발 또는 디자인 단계로 돌아갑니다.
   2. 컴포넌트 리뷰를 통과하면 EDS Figma와 EDS Code가 각각 디자인팀과 프론트엔드팀에 공유됩니다.

위 단계 중 커뮤케이션 비용이 가장 많이 발생한 단계는 어디일까요? 실질적인 피드백을 하는 리뷰 단계가 아닌, 개발 단계입니다. 개발 단계에서 개발자는 전달 받은 컴포넌트 명세의 문제점 파악 한 후, 디자이너에게 반복하여 수정을 요청하기 때문입니다. 리뷰 단계에서는 이미 여러차례 수정이 된 명세와 개발 단계에서 자체적인 버그 픽스를 거친 상태였으므로 많은 비용이 발생하지 않았습니다.

명세가 반복되어 수정되는 이유는 컴포넌트의 인터페이스와 형태에 대한 디자이너와 개발자의 이해 차이 때문입니다.  초기 명세 작성 시 디자이너는 컴포넌트가 가질 상태를 모두 예측하기 어렵습니다.

(TODO: 이미지 첨부)

반면, 개발자는 실질적으로 필요한 UI 형태와 구조에 대해 예측하기 어렵습니다. 이 문제점에 대해서는 해당 영상에서 잘 설명하고 있어 추가로 참고하면 좋을 것 같습니다. (TODO: 링크 추가)

또한 Figma는 컴포넌트 속성에 대한 가이드라인이 없기 때문에, 디자인팀 내 명확한 명명 규칙이 없다면 동일한 속성이 일관되지 않게 정의될 수 있습니다.

(TODO: 이미지 첨부)

웹에서 표준화되어 있는 속성들은 개발자들이 쉽게 이해할 수 있습니다. 그러나 특정 컴포넌트에서만 사용되는 유사한 커스텀 속성들이 일관되지 않게 정의되면 추가적인 명세 재확인과 커뮤니케이션이 필요합니다.

(TODO: 이미지 첨부)

결국 커뮤니케이션 비용 증가로 인해 개발 단계에서 개발자가 임의로 속성을 변경하는 경우가 빈번히 발생하며, 최종 결과물인 EDS Figma과 EDS Code의 간극이 발생하였습니다.

### 서비스 개발 단계

> 요약
>
> 서비스 개발 과정에서 디자이너는 EDS Figma를 활용하여 Service Figma를 구축하고, 개발자는 이 명세를 기반으로 EDS Code를 사용하여 서비스를 개발합니다. 개발자는 이전 단계에서 발생한 차이로 인해 명세 확인에 더 많은 시간을 할애해야 합니다.

서비스 개발 단계에서는 다음 단계를 거칩니다.

1. **디자인**: 디자이너가 EDS Figma를 사용하여 서비스 개발을 위한 Service Figma를 구현합니다.
2. **개발**: 프론트엔드 개발자가 Service Figma 명세를 기반으로 EDS Code를 사용하여 서비스를 개발합니다.

이전 디자인 시스템 단계에서 발생한 간극으로 인해, 개발자가 각각의 명세를 확인해야 하므로 피로도가 증가합니다. 이는 디자인 시스템을 직접 개발하지 않은 개발자일수록 더욱 두드러집니다.

예시로, 하단의 버튼 컴포넌트의 `넓이`, `테마`, `높이` 속성이 EDS Code에서는 `fullWidth`, `variant`, `size` 속성으로 구현되었습니다. EDS Figma에서는 `size` 속성, `Primary라는 컴포넌트 명칭`, `Size별 Section 기능`으로 구별되었습니다.

```html
<eds-button
  fullWidth
  variant="primary"
  size="”l”"
  text="증명서 발급"
></eds-button>
```

(TODO: 이미지 첨부)

Figma Page, Section 명칭에 따른 계층화도 지원하는 Figma Component Library 환경의 디자이너 팀에서는 효율적인 방식이였습니다. 하지만 프론트엔드팀 권한의 Service Figma 환경은 Page, Section 명칭에 따른 계층화가 지원되지 않았습니다. 이로 인해 개발자는 해당 버튼의 size 속성을 알기위해서 EDS Figma를 반복적으로 확인해야하는 번거로움이 있었습니다.

## 문제점 개선 방안

워크플로우에서 파악된 문제점 개선을 위해, 다음 3가지 방안을 적용하였습니다.

1. **신규 EDS 컴포넌트 명세 작성 시 인터페이스를 협의합니다.**

초기 명세를 작성할 때 디자이너와 개발자가 함께 정확한 인터페이스를 협의합니다. 이는 컴포넌트의 기능, 상태, 속성을 명확히 이해하고 일관성 있게 문서화하는 데 중요합니다.

2. **EDS Figma에 대한 규약을 추가합니다.**

개발자는 EDS Figma로 부터 EDS Code 필요한 모든 정보를 얻을 수 있어야합니다. 그렇다면 디자이너는 EDS Figma로 어디까지 정보를 제공해야할까요? 그 기준을 Figma Plugin API로 접근 가능한 EDS [ComponentSetNode](https://www.figma.com/plugin-docs/api/ComponentSetNode/)와 [InstanceNode](https://www.figma.com/plugin-docs/api/InstanceNode/#getmaincomponentasync)로 부터 EDS Code 구현을 위한 데이터를 추출이 가능한 정도로 정의하였습니다. EDS Code 구현을 위한 두가지 최소 조건은 다음과 같습니다.

```
1. 생성되는 EDS Figma Component Set의 이름 내에 EDS 컴포넌트 명칭에 대한 데이터가 있어야합니다.
이는 다른 Figma Component Set와 구별을 위함입니다.

GOOD 🟢 ) EdsButton, EDS Button, <EdsButton>, <EDS Button>
BAD ❌ ) Primary, Atom, Button

- 명칭 규칙
  - {Compnent Set Name} (eds>{Component Name}>{Property Name}={Property Name}>...)

- 명칭 수정이 필요한 컴포넌트 리스트
  - Bigline -> bigline (eds>textfield)
  - header/toptabbar -> tab (eds)
// ...

2. 동일한 기능을 나타내기 위해 사용되는 property의 값은 통일되어야 합니다.

GOOD 🟢 )
<Button> + property(size: ['sm', 'md', 'lg'])
<Textfield> + property(size: ['sm', 'md', 'lg'])
BAD ❌ )
<Button> + property(size: ['small', 'medium', 'large'])
<Textfield> + property(size: ['sm', 'md', 'lg'])

GOOD 🟢 )
<Checkbox> + property(disabled: [true, false])
<Toggle> + property(disabled: [true, false])
BAD ❌ )
<Checkbox> + property(disabled: [true, false])
<Toggle> + property(usable: ['able', 'disable'])

- 통일되는 property 리스트
  - 🔷 VARIANT / state / [enabled(default), hovered, focused, pressed, disabled ]
  // ...
```

3. **Figma Plugin API를 통해 EDS Figma를 EDS Code 수준으로 추상화된 코드를 제공합니다.**

개발자는 추가적인 명세 확인 없이 EDS Figma를 EDS Code로 구현할 수 있어야합니다. EDS Figma에서 디자인 속성으로만 정보를 제공하는 것은 한계가 있습니다. 규격화된 EDS Figma를 기반으로 EDS Code 수준으로 추상화된 코드를 Figma 환경에서 제공합니다.

이 과정에서 중요한 역할을 하는 것이 바로 Figma Plugin API입니다. 디자인과 개발 간의 간극을 효과적으로 줄이기 위해 Figma Plugin API를 활용하여 EDS Lint, EDS Codegen과 같은 도구를 개발할 수 있었습니다. 사용 기술과 도구에 대해 간단히 소개합니다.

## 사용 기술

### Figma Plugin API

Figma는 개발자가 Figma 환경에서 기능을 확장하여 사용할 수 있도록 Plugin API를 제공하고 있습니다. (TODO: [링크](https://www.figma.com/plugin-docs/)) 이 글에서는 기술적인 세부 사항을 전문적으로 다루지 않지만, Figma Plugin API의 기본 개념을 간단하게 소개합니다. 기존에도 효율적인 디자인과 개발을 위해 다양한 플러그인들이 제공되고 있습니다. (TODO: [링크](https://www.figma.com/community/category/development?resource_type=plugins))

Figma Plugin은 플러그인이 안전하게 실행될 수 있도록 하기 위해 샌드박스 구조를 채택하고 있습니다. 이 두 환경은 분리되어 있으며, 메세지를 통해 정보를 주고 받게 됩니다.

(TODO: 이미지 첨부)

- main 스레드 환경: 플러그인 API를 통해 Figma의 주요 기능에 접근합니다. (ex. Figma SeneNode, Figma Selection 등)
- iframe 환경: 브라우저 API가 사용 가능하며 ui 구현을 담당합니다.

## 도구 소개

### EDS Lint

EDS Lint는 Figma 플러그인으로, 생성된 EDS Figma의 Component Set가 규약에 맞게 정의되었는지 검사합니다.

- 디자인 시스템 개발 단계에서 EDS Figma의 Component Set의 구조와 속성의 일관성을 유지할 수 있습니다.
- 디자이너들이 Figma에서 작업할 때 실시간으로 피드백을 제공하여, 문제를 사전에 방지하고 수정할 수 있도록 돕습니다.

프로토타입 버전으로 개발된 EDS Lint의 기능은 크게 다음과 같습니다.

1. **신규 EDS ComponentSet 및 Variant 생성 기능**

(TODO: 이미지 첨부)

- 신규 ComponentSet 명칭을 입력하고 Create 버튼을 클릭 시, ComponentSetNode가 생성됩니다.
- 규약된(defined) 공통 속성의 체크박스를 클릭 시, 해당 속성과 값을 가진 Variant가 자동 생성됩니다.

2. **기존 EDS ComponentSet Lint 기능**

(TODO: 이미지 첨부)

- 필수 속성이 사용되지 않았을 경우 에러를 표기합니다.
- 규약된(defined)속성의 정의된 값 외 사용에 대한 에러를 표기합니다.
- 미규약된(not defined) 속성 사용에 대한 정보를 제공합니다.

### EDS Codegen

EDS Codegen은 EDS Figma의 Component Set를 EDS Code로 변환해주는 도구입니다. 개발자는 추가적인 명세 확인 없이 EDS Figma로 부터 EDS Code를 제공받을 수 있습니다.

**EDS Figma ComponentSet Instance를 EDS Code로 변환하는 기능**

(TODO: 이미지 첨부)

- EDS Figma CompoentSet의 Instance를 클릭할 시, eds-wc에서 지원하는 HTML, React, Vue 문법으로 Code를 제공합니다.

## 마치며

이번 글에서는 디자인 시스템과 서비스 개발 단계에서 발생하는 문제점들을 분석하고, 문제를 해결하기 위한 여러 방안을 살펴보았습니다. 특히, Figma Plugin API를 활용한 도구들을 통해 더욱 효율적이고 일관된 작업 환경을 만들 수 있었습니다. 이러한 개선 방안들을 통해 앞으로 디자인과 개발간의 간극을 줄이기 위해 노력할 것입니다.
