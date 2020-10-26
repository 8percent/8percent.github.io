---
layout: post
title: "for loop에서 transaction.on_commit 사용하기"
author: anohk
description: "for loop와 transaction.on_commit"
date: 2020-10-26 12:00 +0900
tags: [python, django, transaction]
comments: true
---

transaction block 안에서 for loop와 `on_commit`을 사용할 때 발생할 수 있는 이슈를 공유합니다.

데이터베이스 트랜잭션과 관련된 작업을 할 때, 트랜잭션이 성공적으로 커밋 된 경우에만 특정 동작을 실행시켜야 할 때가 있습니다. Django에서 제공하는 on_commit 을 사용하면 정의된 콜백 함수를 등록 해두고 트랜잭션이 커밋된 후에 등록된 순서대로 콜백 함수를 실행할 수 있습니다.


## 문제 상황

예를 들어, 8퍼센트 추첨 이벤트에서 10명의 사용자를 추첨하고 당첨된 사용자에게 안내를 해야 하는 상황이라고 가정해 봅시다.

```python
with transaction.atomic():
    event.draw_lots(10) 

    for user in event.won_users:
        transaction.on_commit(lambda send_notification(user))
```

기대한 바는 event.won_users를 순회하며 당첨자 마다 알림을 보내는 것입니다.

하지만 위 코드에는 문제가 있습니다. 이 코드를 실행하면 won_users의 마지막 사용자에게 10번의 알림을 전송하는 무서운 일이 발생하게 됩니다.


## 원인

콜백 함수가 실행될 때 함수에 전달한 인자가 언제 평가 되는지 고려하지 못한 것이 원인입니다.

문제가 되는 코드에서는 for loop를 돌면서 당첨자 수 만큼 콜백 함수를 등록합니다.

등록된 함수들을 실행하려고 할 때에는 모든 당첨자에 대해 순회를 마치고 마지막 사용자가 user에 할당된 상태이기 때문에 마지막 사용자만 10번의 알림을 받게 됩니다.


## 문제 해결 방법

익명 함수의 인자에 기본 값을 설정하는 방법으로 해결할 수 있습니다.

```python
with transaction.atomic():
    event.draw_lots(10)

    for user in event.won_users:
        transaction.on_commit(lambda user=user: send_notification(user))
```

위와 같이 작성하면 won_users에 담긴 사용자가 차례대로 익명 함수의 인자에 기본 값으로 할당되어 의도한 대로 모든 당첨자에게 알림을 전송할 수 있습니다.


## 자세히

`transaction.on_commit()`은 인자가 없는 함수를 인자로 받습니다.

on_commit은 인자 없이 콜백 함수를 호출하기 때문에 아래와 같이 익명 함수에 인자를 넘겨야 하는 방식은 사용할 수 없습니다.

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

여기에서 등록하려는 콜백 함수는 user를 인자로 받는 함수를 실행합니다. 콜백 함수에는 인자를 넘기지 않기 때문에, 인자는 어디에선가 정의되있어야 합니다. (위 함수들을 차례로 실행할 때 user는 변하지 않고, 실행 시점에 평가되어 있는 값을 사용하게 됩니다.)

하지만 익명 함수의 인자에 기본 값을 할당한다면, 인자 없이 함수를 호출하는 것이 가능합니다.

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

콜백 함수에 for loop내에서 평가된 user를 인자의 기본 값으로 정의했기 때문에, 인자 없이 함수를 호출한 경우 user는 기본 값을 사용하여 의도한 대로 함수를 실행할 수 있습니다.
