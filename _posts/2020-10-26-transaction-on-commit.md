---
layout: post
title: "for 루프에서 transaction.on_commit을 사용할 때 콜백 함수에 인자를 올바르게 넘기는 방법"
author: anohk
description: "for 루프에서 transaction.on_commit을 사용할 때 콜백 함수에 인자를 올바르게 넘기는 방법"
date: 2020-10-26 12:00 +0900
tags: [python, django, transaction]
comments: true
---

transaction 블록 안에서 for 루프와 `on_commit`을 사용할 때 발생할 수 있는 이슈를 공유합니다.

트랜잭션이 성공적으로 커밋 된 경우에만 특정 동작을 실행해야 할 때가 있습니다. 이 때 장고의 [transaction.on_commit](https://docs.djangoproject.com/en/3.1/topics/db/transactions/#django.db.transaction.on_commit) 함수를 사용하여 콜백 함수를 등록하면 트랜잭션이 커밋된 후 등록된 순서대로 콜백 함수가 실행됩니다.


## 문제 상황

예를 들어, 8퍼센트 추첨 이벤트에서 10명의 사용자를 추첨하고 당첨된 사용자에게 안내를 해야 하는 상황이라고 가정해 봅시다.

```python
with transaction.atomic():
    event.draw_lots(10) 

    for user in event.won_users:
        transaction.on_commit(lambda send_notification(user))
```

기대한 바는 event.won_users를 순회하며 각 당첨자마다 알림을 보내는 것입니다.

하지만 문제가 있습니다. 코드를 실행하면 won_users 의 사용자들에게 알림을 한번씩 발송하는 것이 아니라, 마지막 사용자 한명에게만 알림을 10번 발송하는 무서운 일이 발생합니다.


## 원인

콜백 함수에 전달한 인자가 언제 평가되는지를 고려하지 못한 것이 원인입니다.

문제가 되는 코드에서는 for 루프를 돌면서 당첨자 수만큼 콜백 함수를 등록합니다.

등록된 함수들을 실행하려고 할 때에는 모든 당첨자에 대해 순회를 마치고 마지막 사용자가 user에 할당된 상태이기 때문에 마지막 사용자만 10번의 알림을 받게 됩니다.


## 문제 해결 방법

익명 함수의 인자에 기본 값을 설정하는 방법으로 해결할 수 있습니다.

```python
with transaction.atomic():
    event.draw_lots(10)

    for user in event.won_users:
        transaction.on_commit(lambda user=user: send_notification(user))
```

위와 같이 작성하면 won_users에 포함된 사용자들이 각각 차례대로 익명 함수의 매개변수의 기본값으로 정의되며, 의도한 대로 모든 당첨자에게 알림을 발송할 수 있습니다.


## 자세히

`transaction.on_commit()`은 매개변수가 없는 함수를 인자로 받습니다.

on_commit은 콜백 함수를 호출할 때 콜백 함수에 별도의 인자를 전달하지 않습니다. 따라서 아래와 같이 필수 매개변수가 정의된 익명 함수를 등록하는 방식은 사용할 수 없습니다.

```python
transaction.on_commit(lambda user: send_notification(user))
```

처음 작성한 코드에서 커밋 후 실행될 익명 함수들을 풀어쓰면 아래와 같이 표현해 볼 수 있습니다.

```python
def call_back_0():
    send_notification(user)

def call_back_1():
    send_notification(user)

...

def call_back_9():
    send_notification(user)
```

여기에서 등록하려는 콜백 함수는 user를 인자로 받는 함수를 실행합니다. 콜백 함수에 매개변수가 없으며 인자도 전달받지 않으므로 실행시간에 어딘가 다른 곳에 정의되어 있는 user 변수의 값을 평가하여 사용합니다. 그리고 위 함수들을 차례로 실행할 때 user는 모두 동일한 값으로 평가될 것입니다.

하지만 익명 함수의 매개변수에 기본값을 정의해두면 인자를 전달하지 않고 함수를 호출해도 됩니다.

```python
for user in event.won_users:
    transaction.on_commit(lambda user=user: send_notification(user))
```

위의 익명 함수를 풀어쓰면 아래와 같습니다.

```python
def call_back_0(user=첫번째 사용자):
    send_notification(user)

def call_back_1(user=두번째 사용자):
    send_notification(user)

...

def call_back_9(user=열번째 사용자):
    send_notification(user)
```

for 루프 안에서 user를 평가하여 콜백 함수의 매개변수의 기본값으로 정의했기 때문에, on_commit이 인자를 전달하지 않고 함수를 호출하더라도, 콜백 함수는 user 매개변수의 기본값을 사용할 수 있으며, 의도한 대로 함수가 실행됩니다.
