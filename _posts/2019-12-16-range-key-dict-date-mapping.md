---
layout: post
title: "날짜 구간에 값을 매핑하는 방법 (그리고 RangeKeyDict)"
author: 박연오
description: "날짜 구간에 값을 매핑하는 방법과 RangeKeyDict의 사용법"
date: 2019-12-16 12:00 +0900
tags: [python, mapping, date]
comments: true
---

일정한 날짜 구간에 값을 매핑하는 방법을 알아봅시다.

**한줄 요약: 날짜 구간에 값을 매핑하고자 할 때, 외부 라이브러리를 사용할 수 있다면 RangeKeyDict를 이용하면 깔끔합니다.**

특정 일자에 따라 값을 다르게 설정해야 하는 경우가 자주 있습니다.

예를 들어, 회사의 포인트 적립 정책이 2020년 1월 1일부터는 1 퍼센트, 2020년 2월 2일부터는 2 퍼센트, 2020년 8월 8일부터는 8퍼센트라고 합시다.

이것은 if 문을 이용해 간단히 코드로 옮길 수 있습니다.

```
from datetime import date
from decimal import Decimal

def date_to_point_rate(base_date):
    if base_date < date(2020, 1, 1):
        return None
    if base_date < date(2020, 2, 2):
        return Decimal('0.01')
    if base_date < date(2020, 8, 8):
        return Decimal('0.02')
    return Decimal('0.08')

assert date_to_point_rate(date(2019, 12, 31)) == None
assert date_to_point_rate(date(2020, 1, 1)) == Decimal('0.01')
assert date_to_point_rate(date(2020, 2, 2)) == Decimal('0.02')
assert date_to_point_rate(date(2020, 8, 8)) == Decimal('0.08')
```

하지만 이런 방식은 날짜 구간이 변경될 때마다 비교식을 수정해야 하는 부담이 있습니다. 작성한 코드도 한눈에 쉽게 들어오지 않습니다. 자세히 보지 않으면 2019년 8월 8일의 포인트 적립율이 2퍼센트인지 8퍼센트인지 헷갈릴 수 있다고 생각합니다.

날짜 구간에 값을 매핑해야 하는 경우가 자주 있어서 더 좋은 방법을 생각해 보았습니다. 날짜와 값을 쌍으로 지정해 두고 그 안에서 날짜를 맞추면 어떨까요?

```
# 주의: 적용 날짜를 내림차순으로 나열해야 한다.
applying_date_to_point_rate_pairs = (
    (date(2020, 8, 8), Decimal('0.08')),
    (date(2020, 2, 2), Decimal('0.02')),
    (date(2020, 1, 1), Decimal('0.01')),
    (date.min, None),
)

def date_to_point_rate_v2(base_date):
    for applying_date, point_rate in applying_date_to_point_rate_pairs:
        if applying_date <= base_date:
            return point_rate

assert date_to_point_rate_v2(date(2019, 12, 31)) == None
assert date_to_point_rate_v2(date(2020, 1, 1)) == Decimal('0.01')
assert date_to_point_rate_v2(date(2020, 2, 2)) == Decimal('0.02')
assert date_to_point_rate_v2(date(2020, 8, 8)) == Decimal('0.08')
```

이렇게 하면 날짜 매핑을 로직이 아니라 값으로 설정할 수 있습니다. 이제 날짜 구간을 변경할 때 `applying_date_to_point_rate_pairs`에서 날짜와 값 쌍을 수정하면 됩니다. 하지만 여전히 단점이 있습니다. 적용 날짜를 내림차순으로 입력해야 한다는 점이 자연스럽지 않은 것 같습니다.(일반적으로 오름차순이 기본이므로) 그리고 `date_to_point_rate_v2` 함수를 이해하기가 약간은 어려운 것 같습니다.

이 코드를 보고 직장 동료 김남홍 님이 좋은 방법을 귀뜸해 줬습니다. "RangeKeyDict 한 번 써봐!"

[RangeKeyDict](https://github.com/albertmenglongli/range-key-dict)는 파이썬에 이런 게 있으면 좋겠다고 한 번쯤 생각했던 겁니다. 사전(dict)의 키로 range를 사용할 수 있는 매핑 컬렉션입니다.

RangeKeyDict를 사용하려면 range-key-dict 패키지를 설치해야 합니다.

```
pip install range-key-dict
```

이제 `range_key_dict.RangeKeyDict`를 임포트하여 `RangeKeyDict({(start, end): value})`와 같은 형태로 매핑을 작성할 수 있습니다. 포인트 정책을 RangeKeyDict로 수정해 봅시다.

```
from range_key_dict import RangeKeyDict

applying_date_to_point_rate_mapping = RangeKeyDict({
    (date.min, date(2020, 1, 1)): None,
    (date(2020, 1, 1), date(2020, 2, 2)): Decimal('0.01'),
    (date(2020, 2, 2), date(2020, 8, 8)): Decimal('0.02'),
    (date(2020, 8, 8), date.max): Decimal('0.08'),
})

def date_to_point_rate_v3(base_date):
    return applying_date_to_point_rate_mapping[base_date]

assert date_to_point_rate_v3(date(2019, 12, 31)) == None
assert date_to_point_rate_v3(date(2020, 1, 1)) == Decimal('0.01')
assert date_to_point_rate_v3(date(2020, 2, 2)) == Decimal('0.02')
assert date_to_point_rate_v3(date(2020, 8, 8)) == Decimal('0.08')
```

코드가 꽤 직관적입니다. RangeKeyDict는 꼭 날짜가 아니더라도 어떤 범위에 값을 매핑할 때 편리하게 사용할 수 있습니다. 자주 바뀌는 법령, 정책 등을 반영할 일이 있으면 RangeKeyDict를 활용해보시면 좋을 것입니다.


