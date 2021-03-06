---
layout: post
title: Django - Admin의 Query 최적화
tags: [python, django, query, select_related, only ]
---

### Django Admin

#### 1) 개요
> 특정 Admin에 접근하는데 걸리는 시간이 너무 비약적으로 상승.  
(Local에선 20초...Production에선 6초 정도...)

> Admin에서만 정보를 확인할 수 있고,  
또 그 정보를 사용하는 유저들이 있음.(관리자?)

#### 2) Select_Related의 등판.

> ForeignKey Field가 보이면 일단 select_related를 써야지~  
라고 마음을 먹어야함.

> select_related를 사용하면 좋은 이점을 설명하자면.

```
class School(models.Model):
    name = models.CharField(max_length=200)
    

class Person(models.Model):
    school = models.ForeignKey(School)
    name = models.CharField(max_length=100)
    
class Car(models.Model):
    owner = models.ForeignKey(Person)
    name = models.CharField(max_length=100) 
```

> 이 상황에서 Car모델의 id가 1인 instance의 person과 school을 가져온다면
1. 첫번째 방법 
    ```
        car = Car.objects.get(id=1)
        person = car.owner
        school = person.school
    ```

2. 두번째 방법 
    ```
        car = Car.objects.select_related('owner__school').get(id=1)
        person = car.owner
        school = person.school
    ```
> 이 후 출력결과에서는 두개의 차이점을 확인할 수 어렵지만,  
DB에 접근하는 방식을 보면 차이가 나는 것을 확인할 수 있다.

첫번째 방법은 DB에 총 세번을 접근하게 된다.

    1. Car모델에서 id가 1인 car를 가져오는데 1번
    2. car instance에서 person을 가져오는데 1번
    3. person에서 school을 가져오는데 1번
    
두번째는 어떨까?
  
    1. Car모델에 접근한채로 owner와 school에 관해서 cache에 다 저장해둔다.  
    2. 끝
    
항상 select_related를 사용하는게 좋을까?????
#### 3) Only

> 말그대로 지정한 column만을 접근해서 처리해온다.  

> Django Admin에서는 list를 가져올때  
 어떤 부분들만 보여줄 것인지를 선택할 수 있는데,  
 그 부분들 중에서 보여줘야하는 한정적인 자원만을  
 가져오도록하는게 최선일거라고 생각해서 사용하게 되었다.
 
 
#### 4) get_queryset

> django admin에서 model의 list를 불러올때 사용되는 함수이다.

> override해서 사용하는게 좀 더 최적화에 유용하다.


기본적으로,
```
def get_queryset(self, request):
    qs = self.model._default_manager.get_queryset()
    return qs
```
라면... 수정해서는
```
def get_queryset(self, request):
    queryset = super(SomethingAdmin, self).get_queryset(request). \
    select_related("blah").prefetch_related("blahblah"). \
    only("blah", "blahblah", "somthing_name")
    return queryset
```
이런 느낌이랄까??
 
#### 5) Fieldset
> 이 부분은 사실 최적화를 위한 부분이라기보다는  
사용유저가 좀 더 편하게 보기위해서 적용시켜본 부분이다.


> list로 구성해서 그 안에 튜플을 심으면 된다.

```
fieldsets = [
    (None, {'fields': ['name']}),
    ("Info", {'fields': ['school']}), 
]
```

이렇게되면 영역이 분리되서 화면에 보여진다.

    


