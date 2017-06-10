---
layout: post
title: "8퍼센트 Test case 작성 가이드"
author: seba
description: "8퍼센트 개발팀의 Test case 작성 가이드를 공유합니다."
date: 2017-05-31
tags: [python,django,TDD,test,guide]
comments: true
---

8퍼센트에서 Python Django 코드에 대한  Test case 작성시 사용하는 가이드를 공유해보려고 합니다.

## 클래스명

- 일반적으로 TestCase 를 상속 받는 클래스일 경우 class 명의 마지막에 TestCase 를 붙입니다.
- 예제: SimpleTestCase(TestCase)

## 함수명

테스트 함수명의 경우 test_ 로만 시작하면 동작하는데 문제가 없고 테스트 코드에까지 주석을 다는 것은 번거로우므로 함수명의 test_ 뒷부분을 한글로 하여 설명을 대신하도록 합니다.

```python
class IUPaginationMethodTestCase(TestCase):
    @classmethod
    def setUpTestData(cls):
        cls.request = Mock()
        cls.request.GET = {'page': 1, 'items_per_page': 1}
        cls.pagination = IUPagination(cls.request)
 
    def test_page_url_기본(self):
        expected = '?{}=1'.format(self.pagination.page_key)
        self.assertEqual(self.pagination.page_url(), expected)
 
    def test_page_url_쿼리스트링_없는경우_물음표_붙인다(self):
        expected = '/?{}=1'.format(self.pagination.page_key)
        self.pagination.url_prefix = '/'
        self.assertEqual(self.pagination.page_url(), expected)
 
    def test_page_url_쿼리스트링_있는경우_엠퍼센드로_붙인다(self):
        expected = '{}&{}=1'.format(
            self.pagination.url_prefix, self.pagination.page_key
        ))

        self.pagination.url_prefix = '?utm=source'
        self.assertEqual(self.pagination.page_url(), expected)
```

## factory_boy

fixture 를 대신해서 가급적 factory_boy 를 사용합니다.

### signals 끄기

- factory boy로 모델 객체 생성시 signal 이 호출되는데 signal에 대한 테스트가 아니라면 대부분 실행할 필요가 없습니다.
- 이 때 factory.django.mute_signals를 사용해서 끄면 됩니다.
- decorator, context manager 둘 다 사용 가능합니다.
  - decorator
    ```python
    @mute_signals(signals.post_save)
    def test_some_code(self):
        some = SomeFactory()
    ```
  - context manager
    ```python
    with mute_signals(signals.post_save):
        some = SomeFactory()
    ```

### 참고 링크

- [factory_boy](https://factoryboy.readthedocs.io/en/latest/)
- [Disabling signals](http://factoryboy.readthedocs.io/en/latest/orms.html#disabling-signals)

## setUpTestData vs setUp

- fixture를 사용하면 fixture로 정의한 모델 객체가 모든 테스트 시작 전에 생성이 되는데 유사하게 setUp 에서 factory 생성을 하게 되면 매번 객체 생성을 하게 되므로 느립니다.
- 테스트에서 read only 로만 사용하는 객체의 경우 class method인 setUpTestData 에서 생성하면 1번만 생성이 되므로 빨라집니다.
- 가급적 setUp 에서 매번 객체를 생성하는 것을 지양하고 테스트 함수 내에서 필요한 객체만 생성하는 것이 효율적이고 빠릅니다.

## method mock

메소드를 mock 하는 경우 unittest.mock.patch() 를 사용합니다.

### decorator

보통 테스트 메소드에 대한 decorator 로 사용합니다.

### 직접 호출

- class 내의 여러 테스트 메소드 혹은 모든 테스트 메소드에서 동일한 함수를 mock 하는 경우에는 start, stop 을 활용하면 편합니다.
- 예제 코드
  ```python
  from unittest import mock

  class MyTest(TestCase):
      def setUp(self):
          self.mock_method1 = mock.patch('package.module.method1').start()
          self.mock_method1 = mock.patch('package.module.method2').start()
  
      def tearDown(self):
          mock.patch.stopall()

      def test_something(self):
          something()
          self.assertTrue(self.mock_method1.called)
    ```
- 참고 링크: [patch methods start and stop](https://docs.python.org/3/library/unittest.mock.html#patch-methods-start-and-stop)

## timezone

- datetime.datetime.now() datetime.datetime.strptime() 등을 사용해서 naive datetime 객체를 django 모델의 DateTimeField 에 할당할 필요가 있는 경우 반드시 django.utils.timezone.make_aware() 를 사용해서 time-zone-aware datetime 객체로 변환한 후에 합니다.
- 참고 링크: [Django timezone 문제 파헤치기](http://localhost:4000/2017-05-31/django-timezone-problem/)

## freezegun

- 특정 시점에서의 테스트가 필요한 경우 freezegun 을 사용해서 현재 시간값을 고정합니다.
- 가급적 decorator 나 context manager 를 사용해서 특정 클래스나 메소드, 혹은 코드 블럭에만 적용하도록 하는 것이 좋습니다.
  - decorator 예제
    ```python
    from freezegun import freeze_time
    import datetime
    import unittest
     
    @freeze_time("2012-01-14")
    def test():
	assert datetime.datetime.now() == datetime.datetime(2012, 1, 14)
    ```
  - context manager 예제
    ```python
    from freezegun import freeze_time
     
    def test():
        assert datetime.datetime.now() != datetime.datetime(2012, 1, 14)
        with freeze_time("2012-01-14"):
	    assert datetime.datetime.now() == datetime.datetime(2012, 1, 14)
        assert datetime.datetime.now() != datetime.datetime(2012, 1, 14)
    ```
- 특정 테스트 케이스 전체에 적용을 하기 위해 start(), stop() 메소드를 사용하기도 하는데 이 경우 반드시 stop() 을 해주어야 다른 테스트 케이스의 시간 값에 영향을 주지 않습니다.
  - 예제
    ```python
    from django.test import TestCase
    from freezegun import freeze_time
     
    class SomeTestCase(TestCase):
        def setUp(self):
            self.freezer = freeze_time("2016-01-05 00:00:00")
            self.freezer.start()
     
        def tearDown(self):
            self.freezer.stop()
    ```
- 참고 링크: [freezegun](https://github.com/spulec/freezegun)

## 맺음말

Python Django 개발시 Test case 작성을 잘 하기 위한 8퍼센트 개발팀의 가이드를 공유해 보았습니다.
Python Django 개발자들이 Test case 작성을 효율적으로 잘 해서 서비스의 안정성을 높이는데 도움이 되기를 기대해 봅니다.

