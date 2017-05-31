---
layout: post
title: "Django timezone 문제 파헤치기"
description: "Django timezone 문제를 파헤쳐보자"
date: 2017-05-31
tags: [python,django,timezone]
comments: true
---

Python django 에서 날짜와 시간을 다룰 때 발생할 수 있는 timezone 문제를 겪으면서 알게된 점들을 공유해보려고 합니다.
이 포스팅은 python 3.5 django 1.9에 PostgreSQL 사용을 기준으로 합니다.

### 프로젝트 설정

한국 시간(UTC+9)을 사용하기 위해 django 프로젝트의 settings.py 에 아래와 같이 설정을 합니다.

```python
USE_TZ = True
TIME_ZONE = 'Asia/Seoul'
```

### 설정에 따른 영향

1. datetime.datetime.now() datetime.datetime.today() 등으로 현재 날짜/시간을 가져오는 경우 TIME_ZONE 설정에 따라 +9 기준의 값이 나오지만 TIME_ZONE 정보가 없는 naive datetime 객체입니다.
2. 위의 시간값을 사용해서 DB 에 저장하면 UTC 기준 시간값으로 저장이 됩니다. (time-zone-aware datetime 객체)
3. django 모델을 통해 DB 에서 읽은 DateTimeField 타입의 컬럼 값은 TIME_ZONE 이 UTC 인 time-zone-aware datetime 객체입니다.

### 문제가 발생할 수 있는 부분

1. 위의 3번처럼 읽은 시간값에 .date() 를 호출해서 날짜만 얻으면 UTC 기준의 날짜가 나오므로 한국 시간 기준의 날짜와 맞지 않을 수 있습니다. 예를 들어 한국 시간으로 9시 이전이면 UTC 기준으로 하면 하루 전이 되어버립니다.
2. string 을 datetime.datetime.strptime() 함수를 통해 날짜/시간값을 얻거나 datetime.datetime.now() 를 등을 통해 현재 날짜/시간값을 얻은 naive datetime 객체를 time-zone-aware datetime 객체와 비교시 exception 이 발생합니다.
3. django 모델의 DateField 의 경우는 UTC 기준이 아닌 한국 날짜로 저장이 되고 읽어집니다.

### 문제 해결 방법

1. naive datetime 개체인 datetime.datetime.now() 를 사용하는 대신 django.utils.timezone.now() 를 사용해서 time-zone-aware datetime 객체를 사용하도록 합니다.
2. 한국 시간이 필요한 경우 아래와 같은 코드를 사용해서 변환합니다.

    ```python
    from django.utils import timezone

    now = timezone.localtime(timezone.now())
    ```

3. naive datetime 객체를 어쩔 수 없이 사용해야 하는 경우 아래와 같은 코드를 사용해서 time-zone-aware datetime 객체로 변환합니다.

    ```python
    from datetime import datetime
    from django.utils import timezone

    repay_datetime = datetime.strptime('2016-10-01 14:00:00', '%Y-%m-%d %H:%M:%S')
    repay_datetime_aware = timezone.make_aware(repay_datetime)
    if investment.created_datetime > repay_datetime_aware:
        do_something()
    ```

### 맺음말

한국 시간대에 맞는 한국향 서비스를 개발하게 되면 timezone 문제를 겪을 수 있습니다.
전세계의 기준 시간대가 한국 시간대면 좋겠지만 아닌게 현실이기 때문에 문제가 발생하지 않도록 잘 개발할 수 밖에 없는게 현실입니다.
한국의 많은 django 개발자들에게 도움이 되길 바라면서 글을 마칩니다.

### 참고

[Django 1.9 timezones](https://docs.djangoproject.com/en/1.9/topics/i18n/timezones/)
