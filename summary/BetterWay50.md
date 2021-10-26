# 50. __set_name__ 으로 클래스 애트리뷰트를 표시하라

## 1. 메타클래스를 활용한 중복 제거

```python
class Field:
    def __init__(self, name):
        self.name = name
        self.internal_name = '_' + self.name
        
    def __get__(self, instance, instance_type):
        if instance is None:
            return self
        return getattr(instance, self.internal_name, '')
        
    def __set__(self, instance, value):
        setattr(instance, self.internal_name, value)

class Customer:
    # 클래스 애트리뷰트
    first_name = Field('first_name')
    last_name = Field('last_name')
    prefix = Field('prefix')
    suffix = Field('suffix')

cust = Customer()
print(f'이전: {cust.first_name!r} {cust._ _dict__}')
cust.first_name = '유클리드'
print(f'이후: {cust.first_name!r} {cust._ _dict__}')

>>>
이전: '' {}
이후: '유클리드' {'_first_name': '유클리드'}
```

- 데이터베이스 테이블의 각 컬럼에 해당하는 프로퍼티를 클래스 `Field` 로 정의
- 컬럼 이름을 `Field` 디스크립터에 저장한 후 `setattr` 내장 함수를 사용해 인스턴스별 상태를 직접 인스턴스 딕셔너리에 저장 가능, `getattr` 로 인스턴스의 상태 read 가능
- 로우를 표현하는 클래스 `Customer` 정의
- **단점**
    - 중복이 많음
        - `first_name = Field('first_name')`
            - 이름이 중복 표현됨
- 해결방안
    - 메타클래스 활용

```python
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        for key, value in class_dict.items():
            if isinstance(value, Field):
                value.name = key
                value.internal_name = '_' + key
        cls = type.__new__(meta, name, bases, class_dict)
        return cls

class DatabaseRow(metaclass=Meta):
    pass

class Field:
    def __init__(self):
        # 이 두 정보를 메타클래스가 채워준다
        self.name = None
        self.internal_name = None
        
    def __get__(self, instance, instance_type):
        if instance is None:
            return self
        return getattr(instance, self.internal_name, '')
        
    def __set__(self, instance, value):
        setattr(instance, self.internal_name, value)

class BetterCustomer(DatabaseRow):
    first_name = Field()
    last_name = Field()
    prefix = Field()
    suffix = Field()

cust = BetterCustomer()
print(f'이전: {cust.first_name!r} {cust._ _dict__}')
cust.first_name = '오일러'
print(f'이후: {cust.first_name!r} {cust._ _dict__}')

>>>
이전: '' {}
이후: '오일러' {'_first_name': '오일러'}
```

- class `BetterCustomer` 문에 훅을 걸어 class 본문이 끝나자마자 필요한 동작 수행
    - 디스크립터의 `Field.name` 과 `Field.internal_name` 을 자동으로 대입 가능
- `Meta` 의 `__new__` 메소드 사용하여 애트리뷰트 설정
- **단점**
    - `DatabaseRow` 상속받는 것을 까먹거나 클래스 계층 구조로 인한 제약으로 인해 `DatabaseRow` 를 상속받지 못할 경우 사용 불가

## 2. `__set_name__` 특별 메소드 활용

```python
class Field:
    def __init__(self):
        self.name = None
        self.internal_name = None
        
    def __set_name__(self, owner, name):
        # 클래스가 생성될 때 모든 스크립터에 대해 이 메서드가 호출된다
        self.name = name
        self.internal_name = '_' + name
        
    def __get__(self, instance, instance_type):
        if instance is None:
            return self
        return getattr(instance, self.internal_name, '')
        
    def __set__(self, instance, value):
        setattr(instance, self.internal_name, value)

class FixedCustomer:
    first_name = Field()
    last_name = Field()
    prefix = Field()
    suffix = Field()

cust = FixedCustomer()
print(f'이전: {cust.first_name!r} {cust._ _dict__}')
cust.first_name = '메르센'
print(f'이후: {cust.first_name!r} {cust._ _dict__}')

>>>
이전: '' {}
이후: '메르센' {'_first_name': '메르센'}
```

- python 3.6 부터 도입된 `__set_name__` 특별 메소드 사용
- 클래스가 정의될 때마다 해당 클래스 안에 있는 디스크립터 인스턴스의 `__set_name__` 을 호출
- `__set_name__` 은 디스크립터 인스턴스를 소유 중인 클래스와 디스크립터 인스턴스가 대입될 애트리뷰트 이름을 인자로 받음
- 해당 예제는 `Meta.__new__` 가 하던 작업을 `__set_name__` 에서 처리
- **장점**
    - 특정 메타클래스 기반 클래스를 상속받지 않아도 `Field` 디스크립터가 제공하는 기능 활용 가능