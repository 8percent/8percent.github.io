---
layout: post
title: "[장고 ORM] 역방향 참조 외래 키 찾기"
author: bakyeono
description: "[장고 ORM] 역방향 참조 외래 키 찾기"
date: 2019-11-29 20:00 +0900
tags: [django, orm]
comments: true
---

이 포스팅은 8퍼센트의 CTO 이호성 님으로부터 치킨 두 마리를 지원받아 주관적으로 작성되었습니다. (아~ 글감 고르기 힘들다.)

## 개요 (TL; DR)

장고에서 외래 키 값을 수정할 때, 순방향 참조를 수정하는 경우(수정할 모델이 다른 모델을 가리킬 때)에는 단순히 외래 키 필드에 저장된 값을 수정하면 됩니다.

하지만 반대로, 역방향 참조를 수정하는 경우(다른 모델이 수정할 모델을 가리킬 때)에는 수정할 모델을 가리키는 모든 모델을 수정해야 합니다. 어디서 이 모델을 가르키는지 다 모른다고요? 장고 모델 클래스의 `_meta.related_objects` 속성에서 찾을 수 있습니다.

```
for related_object in Deal._meta.related_objects:
    print(
        related_object.related_model.__name__,
        related_object.remote_field.name,
        related_object.get_accessor_name(),
        sep='\t',
    )

# => Purchase           product   purchases
# => Sell               product   sells
# => ProductStatistics  product   statistics
```

## 예제: 외래 키로 연결한 데이터 모델

설명을 위해 단순한 상품 판매 모델을 예로 들겠습니다.

![예제의 ERD](images/ookup-reversed-foreign-keys-1.png)

```
class ProductCategory(Model):
    code = CharField(...)

class Product(Model):
    category = Foreignkey(ProductCategory, related_name='products', ...)
    name = CharField(...)
    ...

class Purchase(Model):
    buyer = ForeignKey(User, related_name='purchases', ...)
    product = ForeignKey(Product, related_name='purchases', ...)
    ...

class Sell(Model):
    seller = ForeignKey(User, related_name='sells', ...)
    product = ForeignKey(Product, related_name='purchases', ...)
    ...
```

상품분류(`ProductCategory`)에 속한 상품(`Product`)들을 사용자(`User`)가 구매(`Purcase`)하거나 판매(`Sell`)할 수 있는 구조입니다. 모델의 세부 내용과 `on_delete` 같은 필수 매개변수 등은 생략했습니다. 예제를 간단히 하기 위한 것이니 신경쓰지 않으셔도 됩니다.

## 순방향 참조 수정하기

순방향 외래 키 참조를 수정하는 것부터 해봅시다. 상품분류와 상품을 다음과 같이 등록해 두었다고 합시다.

```
category_book = ProductCategory.objects.create(code='BOOK', ...)
category_magazine = ProductCategory.objects.create(code='MAGAZINE', ...)

Product.objects.create(category=category_book, name='월간 8퍼센트 소식 2020년 10월호', ...)
Product.objects.create(category=category_book, name='월간 8퍼센트 소식 2020년 11월호', ...)
```

상품 분류로 'BOOK', 'MAGAZINE'이 있고, BOOK으로 분류된 '월간 8퍼센트 소식지'라는 상품들이 입력되어 있습니다. 그런데 소식지 상품들의 상품분류가 잘못 입력되었다고 합니다. BOOK이 아니라 MAGAZINE이라는군요. 이걸 수정하는 건 어렵지 않습니다. 대상 상품들의 `category` 필드만 수정하면 됩니다.

```
Product.objects.filter(
    category=category_book,
    name__startswith='월간 8퍼센트 소식지',
).update(
    category=category_magazine,
)
```

순방향 외래 키 참조를 수정하는 건 간단하군요.

## 역방향 참조 수정하기

역방향 외래 키 참조를 수정하는 건 조금 까다롭습니다. 구매와 판매가 다음과 같이 기록되어 있다고 합시다.

```
product_1 = Product.objects.create(name='월간 8퍼센트 소식 2020년 12월호', ...)
product_2 = Product.objects.create(name='월간 8퍼센트 소식 2020년 12월호', ...)

Purchase.objects.create(product=product_1, ...)
Purchase.objects.create(product=product_2, ...)
Sell.objects.create(product=product_1, ...)
Sell.objects.create(product=product_2, ...)
...
```

데이터 정리를 하다가 `product_1`과 `product_2` 가 동일한 제품이라는 사실을 발견했습니다. 제품 담당자가 실수로 동일한 제품을 두 번 입력했던 것입니다.

데이터 이상이 발생한 상황이므로 수정해야 합니다. `product_2`는 삭제하고 `product_1`만 남기는 방식으로 둘을 합치기로 결정했습니다. `product_2`는 다른 모델이 참조하고 있기 때문에 그냥 삭제하면 안 됩니다. 먼저 `product_2`를 참조하는 모델들을 찾아  `product_1`로 수정해야 합니다.

```
Purchase.objects.filter(product=product_2).update(product=product_1)
Sell.objects.filter(product=product_2).update(product=product_1)
```

앞서 작성한 모델 정의 코드를 확인해보면 `Product` 모델을 참조하는 모델은 `Purchase`와 `Sell` 뿐입니다. 위 소스코드 같이 모델을 직접 찾아 수정하는 것도 한 방법입니다.

그런데 이 방법에는 잠재적인 위험이 있습니다. 만약 `Product` 모델을 참조하는 모델이  `Purchase`와 `Sell`  뿐이라고 확신할 수 있을까요? 장고 앱은 다른 장고 앱에서 가져와 함께 사용할 수도 있습니다. 그렇기 때문에 모델을 참조하는 부분이 같은 앱의 소스코드에 없더라도 잠재적으로 다른 곳에서 참조될 가능성이 있습니다. 따라서 이 경우처럼 역방향 참조를 수정할 때는 이 모델을 가리키는 역방향 외래 키를 모두 조회해야 합니다.

## 역방향 참조를 모두 찾아 수정하기

장고 모델 클래스에는 `_meta` 라는 속성이 정의되어 있습니다. 이 속성은 `Options` 클래스의 인스턴스로, 모델의 여러 가지 부가 정보가 정의되어 있는데 그 중에는 참고 관계 정보도 있습니다. `Options` 인스턴스의 다양한 정보 중 `related_objects` 속성이 바로 모델의 참조 관계를 담은 시퀀스입니다. for 문으로 확인해봅시다.

```
for related_object in Deal._meta.related_objects:
    print(related_object)

# => <ManyToOneRel: shopping.purchase>
# => <ManyToOneRel: shopping.sell>
# => <ManyToOneRel: statistics.productstatistics>
```

예제에서 작성한 모델은 `shopping`이라는 앱에 정의해 두었나 봅니다. `shopping` 앱의 `Purchase` 모델과 `Sell` 모델이 다대일 관계(`ManyToOneRel`)로 연결된 것을 확인할 수 있습니다. 그런데 `statistics` 앱에서도 `Product` 모델과 다대일 관계로 연결된 모델이 있나 보군요.

모델과 외래 키 필드의 이름을 정확하게 확인해봅시다.

```
for related_object in Deal._meta.related_objects:
    print(
        related_object.related_model.__name__,
        related_object.remote_field.name,
        related_object.get_accessor_name(),
        sep='\t',
    )

# => Purchase           product   purchases
# => Sell               product   sells
# => ProductStatistics  product   statistics
```

위 코드에서 확인한 속성의 의미는 다음과 같습니다.

- `related_object.related_model`: 참조한 모델
- `related_object.remote_field.name`: 참조한 필드의 이름
- `related_object.get_accessor_name()`: 역참조 모델의 역참조 필드의 이름

역참조 필드의 이름은 `ForeignKey` 필드를 정의하는 모델에서 `related_name`에 정의한 이름입니다. 직접 정의하지 않으면 모델 이름 뒤에 접미사 `_set` 을 붙인 형태로 정의됩니다. 예를 들어, `Sell` 모델의 `product` 필드에서는 `sells`, 라고 정의했는데, 직접 정의하지 않았다면 `sell_set`이 되었을 겁니다.

이를 활용해 `Purchase` 모델이 역참조하는 모든 외래 키를 수정할 수 있습니다.

```
from django.db.models.fields.reverse_related import ManyToOneRel

for related_object in Product._meta.related_objects:
    if type(related_object) != ManyToOneRel:
        continue
    model = related_object.related_model
    field_name = related_object.remote_field.name
    model.objects.filter(
        **{field_name: product_2},
    ).update(
        **{field_name: product_1},
    )
```

`_meta.related_objects` 에는 `ManyToOneRel`외에도 `OneToOneRel`, `ManyToManyRel` 등이 있습니다. 이것들은 수정하는 방식이 `ForeignKey` 와 약간 달라서 `type(related_object)`를 확인해서 제외했습니다. (그것들이 존재한다면 예제와 같이 그냥 넘어가는 게 아니라 알맞은 방법으로 수정하셔야 합니다.)

이걸로 `Product` 가 역참조하는 모델들을 빠짐없이 수정했습니다. 마지막으로 `product_2` 를 삭제하여 작업을 마무리하면 됩니다.

```
product2.delete()
```

## SQL으로 역방향 참조를 확인하고 싶나요?

SQL으로도 역방향 참조를 확인할 수 있습니다. Dataedo의 문서([Dataedo의 문서](https://dataedo.com/kb/query/postgresql/list-foreign-keys))을 참고하세요.

## 참고 자료

장고 Options 클래스 소스코드: [장고 Options 클래스 소스코드](https://docs.djangoproject.com/en/2.2/_modules/django/db/models/options/)

## 장고를 좋아하시나요?

기술 블로그를 잘 써 놓으면 회사에 훌륭한 개발자 분들이 입사지원을 한다는 도시전설이 있더군요. 주식회사 8퍼센트는 한국에서 파이썬과 장고로 금융 서비스를 개발하는 많지 않은 회사 중 하나입니다. 금융에 관심이 있고 파이썬과 장고를 좋아하시는 분들을 찾고 있습니다. 입사지원 방법이 아래에 나와 있습니다. 다닐만한 회사를 찾고 계신다면 부담 느끼지 마시고 연락주시기 바랍니다!
