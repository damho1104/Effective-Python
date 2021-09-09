# 41. 기능을 합성할 때는 믹스인 클래스를 사용하라

## 1. 믹스인(mix-in) 클래스

[믹스인 - 위키백과, 우리 모두의 백과사전](https://ko.wikipedia.org/wiki/%EB%AF%B9%EC%8A%A4%EC%9D%B8)

- 자식 클래스가 사용할 메소드 몇 개만 정의하는 클래스
- 자체 애트리뷰트 정의가 없음
    - 생성자 메소드 호출 필요 없음
- 믹스인으로 합성하거나 계층화하여 반복적인 코드 최소화, 재사용 가능
- 개인적인 생각으로 믹스인 클래스는 상속을 이용한 포함 관계를 정의하려는 느낌!!
- 메모리 내 파이선 객체를 직렬화할 수 있도록 딕셔너리 변환 기능이 담겨있는 믹스인 클래스 예제

```python
class ToDictMixin:
    def to_dict(self):
        return self._traverse_dict(self._ _dict__)

    def _traverse_dict(self, instance_dict):
        output = {}
        for key, value in instance_dict.items():
            output[key] = self._traverse(key, value)
        return output
    
    def _traverse(self, key, value):
        if isinstance(value, ToDictMixin):
            return value.to_dict()
        elif isinstance(value, dict):
            return self._traverse_dict(value)
        elif isinstance(value, list):
            return [self._traverse(key, i) for i in value]
        elif hasattr(value, '__dict__'):
            return self._traverse_dict(value.__dict__)
        else:
            return value
```

- 믹스인을 사용해 이진 트리를 딕셔너리 표현으로 변경하는 예제

```python
class BinaryTree(ToDictMixin):
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right

tree = BinaryTree(10,
                  left=BinaryTree(7, right=BinaryTree(9)),
                  right=BinaryTree(13, left=BinaryTree(11)))
print(tree.to_dict())

>>> 
{'value': 10,
 'left': {'value': 7,
          'left': None,
          'right': {'value': 9, 'left': None, 'right': None}},
 'right': {'value': 13,
           'left': {'value': 11, 'left': None, 'right': None},
           'right': None}}
```

- 만일 `BinaryTree` 하위 클래스가 존재하는 경우 `ToDictMixin.to_dict` 구현은 무한 루프가 생김
- 그러므로 `_traverse` 오버라이드 하여 문제가 되는 부분을 수정하여 무한 루프 방지

```python
class BinaryTreeWithParent(BinaryTree):
    def __init__(self, value, left=None,
                  right=None, parent=None):
        super().__init__(value, left=left, right=right)
        self.parent = parent

    def _traverse(self, key, value):
        if (isinstance(value, BinaryTreeWithParent) and key == 'parent'):
            return value.value  # 순환 참조 방지
        else:
            return super()._traverse(key, value)

root = BinaryTreeWithParent(10)
root.left = BinaryTreeWithParent(7, parent=root)
root.left.right = BinaryTreeWithParent(9, parent=root.left)
print(root.to_dict())

>>>
{'value': 10,
 'left': {'value': 7,
          'left': None,
          'right': {'value': 9, 'left': None, 'right': None, 'parent': 7},
          'parent': 10},
 'right': None,
 'parent': None}
```

## 2. 믹스인 합성

- `to_dict` 를 `ToDictMixin` 클래스를 통해 제공한다고 가정
- 임의의 클래스를 JSON 으로 직렬화하는 제너릭 미스인 생성 예제

```python
import json

class JsonMixin:
    @classmethod
    def from_json(cls, data):
        kwargs = json.loads(data)
        return cls(**kwargs)
        
    def to_json(self):
        return json.dumps(self.to_dict())
```

- `JsonMixin` 클래스에는 클래스 메소드와 인스턴스 메소드가 혼재되어 있음
- 믹스인 클래스는 메소드의 동작에 대해 가리지 않고 사용 가능함

```python
class DatacenterRack(ToDictMixin, JsonMixin):
    def __init__(self, switch=None, machines=None):
        self.switch = Switch(**switch)
        self.machines = [
            Machine(**kwargs) for kwargs in machines]

class Switch(ToDictMixin, JsonMixin):
    def __init__(self, ports=None, speed=None):
        self.ports = ports
        self.speed = speed
        
class Machine(ToDictMixin, JsonMixin):
    def __init__(self, cores=None, ram=None, disk=None):
        self.cores = cores
        self.ram = ram
        self.disk = disk

serialized = """{
    "switch": {"ports": 5, "speed": 1e9},
    "machines": [
        {"cores": 8, "ram": 32e9, "disk": 5e12},
        {"cores": 4, "ram": 16e9, "disk": 1e12},
        {"cores": 2, "ram": 4e9, "disk": 500e9}
    ]
}"""
deserialized = DatacenterRack.from_json(serialized)
roundtrip = deserialized.to_json()
assert json.loads(serialized) == json.loads(roundtrip)
```

## 3. 느낌

- 믹스인 클래스는 상속 관계를 활용한 포함 관계를 나타낼 때 사용하는 것이라 이해함
- 그러나 이렇게 되면 상속 관계로 착각할 가능성이 높아지는 것이 아닌가 싶음