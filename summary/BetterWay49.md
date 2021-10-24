# 49. __init_subclass__ 를 사용해 클래스 확장을 등록하라

## 1. 클래스 확장

- object 를 JSON 으로 직렬화하는 표현 방식 구현

```python
import json

class Serializable:
    def __init__(self, *args):
        self.args = args
        
    def serialize(self):
        return json.dumps({'args': self.args})

class Deserializable(Serializable):
    @classmethod
    def deserialize(cls, json_data):
        params = json.loads(json_data)
        return cls(*params['args'])

class Point2D(Serializable):
    def __init__(self, x, y):
        super().__init__(x, y)
        self.x = x
        self.y = y
    
    def __repr__(self):
        return f'Point2D({self.x}, {self.y})'

class BetterPoint2D(Deserializable):
       ...

before = BetterPoint2D(5, 3)
print('객체:', before)

data = before.serialize()
print('직렬화한 값:', data)

after = BetterPoint2D.deserialize(data)
print('이후:', after)

>>>
객체: Point2D(5, 3)
직렬화한 값: {"args": [5, 3]}
이후: Point2D(5, 3)
```

- 해당 접근 방식은 직렬화할 데이터의 타입(`Point2D`, `BetterPoint2D`) 을 미리 알고있는 경우만 사용 가능

- 공통 함수화

```python
class BetterSerializable:
    def __init__(self, *args):
        self.args = args
        
    def serialize(self):
        return json.dumps({
            'class': self._ _class__.__name__,
            'args': self.args,
        })
        
    def __repr__(self):
        name = self._ _class__.__name__
        args_str = ', '.join(str(x) for x in self.args)
        return f'{name}({args_str})'

registry = {}
def register_class(target_class):
    registry[target_class.__name__] = target_class

def deserialize(data):
    params = json.loads(data)
    name = params['class']
    target_class = registry[name]
    return target_class(*params['args'])

class EvenBetterPoint2D(BetterSerializable):
    def __init__(self, x, y):
        super().__init__(x, y)
        self.x = x
        self.y = y

register_class(EvenBetterPoint2D)

before = EvenBetterPoint2D(5, 3)
print('이전:', before)

data = before.serialize()
print('직렬화한 값:', data)

after = deserialize(data)
print('이후:', after)

>>>
이전: EvenBetterPoint2D(5, 3)
직렬화한 값: {"class": "EvenBetterPoint2D", "args": [5, 3]}
이후: EvenBetterPoint2D(5, 3)
```

- serialize 할 때 클래스의 타입 정보를 남겨놓음
    - 이후 deserialize 시 `register_class` 를 통해 등록된 모든 클래스에 대해 확인 및 변환 가능
- deserialize 가 제대로 작동하기 위해선 나중에 역직렬화할 모든 클래스에서 `register_class` 를 호출해야 함
- **단점**
    - `register_class` 호출을 깜빡할 수 있음

## 2. 메타 클래스를 사용한 클래스 확장

- 메타 클래스는 하위 클래스가 정의될 때 class 문을 가로채서 적절한 동작을 수행하게 가능함([Better way 48](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay48.md))

```python
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        cls = type.__new__(meta, name, bases, class_dict)
        register_class(cls)
        return cls
        
class RegisteredSerializable(BetterSerializable, metaclass=Meta):
    pass

class Vector3D(RegisteredSerializable):
    def __init__(self, x, y, z):
        super().__init__(x, y, z)
        self.x, self.y, self.z = x, y, z

before = Vector3D(10, -7, 3)
print('이전:', before)

data = before.serialize()
print('직렬화한 값:', data)
print('이후:', deserialize(data))

>>>
이전: Vector3D(10, -7, 3)
직렬화한 값: {"class": "Vector3D", "args": [10, -7, 3]}
이후: Vector3D(10, -7, 3)
```

- `RegisteredSerializable` 의 하위 클래스를 정의할 때 `register_class` 호출시키고 deserialize 가능한 방법

## 3. `__init_subclass__` 를 사용한 클래스 확장

- `__init_subclass__` 를 사용하면 클래스를 정의할 때 커스텀 로직을 제공할 수 있으므로 코드 가독성 증가
    - 상속 트리가 제대로 되어 있기만 한다면 사용 가능

```python
class BetterRegisteredSerializable(BetterSerializable):
    def __init_subclass__(cls):
        super().__init_subclass__()
        register_class(cls)

class Vector1D(BetterRegisteredSerializable):
    def __init__(self, magnitude):
        super().__init__(magnitude)
        self.magnitude = magnitude

before = Vector1D(6)
print('이전:', before)
data = before.serialize()
print('직렬화한 값:', data)
print('이후:', deserialize(data))

>>>
이전: Vector1D(6)
직렬화한 값: {"class": "Vector1D", "args": [6]}
이후: Vector1D(6)
```