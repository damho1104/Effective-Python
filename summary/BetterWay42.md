# 42. 비공개 애트리뷰트보다는 공개 애트리뷰트를 사용하라

## 1. private attribute

```python
class MyObject:
    def __init__(self):
        self.public_field = 5
        self.__private_field = 10
        
    def get_private_field(self):
        return self.__private_field

foo = MyObject()
foo.__private_field

>>>
Traceback ...
AttributeError: 'MyObject' object has no attribute '__private_field’
```

- 밑줄 2개로 시작하는 애트리뷰트는 private
- 외부에서 private attribute 접근은 예외 발생

```python
class MyParentObject:
    def __init__(self):
        self.__private_field = 71
        
class MyChildObject(MyParentObject):
    def get_private_field(self):
        return self.__private_field
        
baz = MyChildObject()
baz.get_private_field()

>>>
Traceback ...   
AttributeError: 'MyChildObject' object has no attribute '_MyChildObject__private_field’
```

- 하위 클래스에서 부모 클래스의 private attribute 는 접근할 수 없음

- python 에서의 private attribute 처리 방식
    - 이름을 변경함
        - ex. `MyChildObject` 의 `__private_field` 는 `_MyChildObject__private_field` 로 변경됨
    - 그러므로 클래스가 다른 위치에서 접근하게 되면 알 수 없는 이름의 애트리뷰트로 변환되어 예외가 발생함
- 다음과 같이 접근 시 예외 발생 안함

```python
    def __init__(self):
        self.__private_field = 71
        
class MyChildObject(MyParentObject):
    def get_private_field(self):
        return self.__private_field
        
baz = MyChildObject()
assert baz._MyParentObject__private_field == 71
```

## 2. private 애트리뷰트를 쓰지 말아야 하는 이유

- python 에서 private 애트리뷰트를 엄격하게 제한하지 않음
    - python 모토

        개발자가 하고 싶은 일을 언어가 제한하면 안된다

        특정 기능을 확장할 지의 여부는 개발자가 고려해야 할 일, 이로 인한 책임을 다하면 됨

    - 클래스를 작성한 사람이 예기치 못하게 확장할 수 있도록 열어둠으로써 얻을 수 있는 장점이 그로 인해 발생할 수 있는 단점보다 크다고 생각
    - 애트리뷰트에 접근할 수 있는 언어 기능에 대한 훅을 제공함
        - `__getattr__`
        - `__getattribute__`
        - `__setattr__`
- python 은 내부에 몰래 접근함으로써 생길 수 있는 피해를 줄이는데 있어서는 스타일 가이드의 명명 규약을 지킬 것을 당부함

- 다음과 같이 중간에 `MyStringClass` 클래스가 추가되었고 `MyIntegerSubclass` 의 부모가 변경되었다고 하면 `get_value` 의 반환 표현식은 틀린 표현이 되버림

```python
class MyBaseClass:
    def __init__(self, value):
        self._ _value = value
    def get_value(self):
        return self._ _value
        
class MyStringClass(MyBaseClass):
    def get_value(self):
        return str(super().get_value())         # 변경됨
        
class MyIntegerSubclass(MyStringClass):
    def get_value(self):
        return int(self._MyStringClass__value)  # 변경되지 않음
```

- 이 경우 부모 클래스쪽에서 protected 애트리뷰트를 사용하고 예외를 발생시키는 편이 더 나음
- 모든 protected 필드에 설명을 추가하고 어떤 필드를 하위 클래스에서 변경할 수 있고 어떤 필드를 사용하지 말아야 할지 명시해야 함

```python
class MyStringClass:
    def __init__(self, value):
        # 여기서 객체에게 사용자가 제공한 값을 저장한다
        # 사용자가 제공하는 값은 문자열로 타입 변환이 가능해야 하며
        # 일단 한번 객체 내부에 설정되고 나면 
        # 불변 값으로 취급돼야 한다
        self._value = value
    ...
```

## 3. private 애트리뷰트를 써야하는 경우는?

- 자식 클래스가 실수로 부모 클래스가 이미 정의한 애트리뷰트를 정의하였을 경우 충돌 발생

```python
class ApiClass:
    def __init__(self):
        self._value = 5
    def get(self):
        return self._value

class Child(ApiClass):
    def __init__(self):
        super().__init__()
        self._value = 'hello'  # 충돌

a = Child()
print(f'{a.get()} 와 {a._value} 는 달라야 합니다.')

>>>
hello 와 hello 는 달라야 합니다.
```

- 이런 문제 위험성을 줄이기 위해 부모 클래스에서 애트리뷰트 이름 겹침을 해결하기 위해 비공개 애트리뷰트를 사용할 수 있음

```python
class ApiClass:
    def __init__(self):
        self.__value = 5      # 밑줄 두 개#!

    def get(self):
        return self.__value   # 밑줄 두 개#!

class Child(ApiClass):
    def __init__(self):
        super().__init__()
        self._value = 'hello'  # OK!

a = Child()
print(f'{a.get()} 와 {a._value} 는 달라야 합니다.')

>>>
5 와 hello 는 달라야 합니다.
```

## 4. 정리

- python 에서는 private 애트리뷰트를 자식 클래스나 클래스 외부에서 사용하지 못하도록 엄격하게 금지하진 않음
- private 애트리뷰트를 사용하여 하위 클래스에서 해당 애트리뷰트를 사용하지 못하도록 막기보다는 해당 애트리뷰트를 재활용할 수 있도록 허용
- 만일 하위 클래스에서 접근을 막아야 한다면 private 애트리뷰트를 사용하기 보다 protected 애트리뷰트를 사용하고 적절한 문서화를 할 것
- private 애트리뷰트를 꼭 사용하야 하는 경우는 **하위 클래스에서 이름 충돌이 일어나는 경우를 막고 싶을 때 이**다