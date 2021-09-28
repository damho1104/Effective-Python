# 43. 커스텀 컨테이너 타입은 collections.abc 를 상속하라

## 1. 커스텀 컨테이너 타입

- 파이선 프로그래밍을 하게 되면 주로 데이터를 관리하기 위해 특정 클래스를 정의하고 해당 객체들이 서로 상호작용하는 방법을 기술함
- 모든 파이선 클래스는 일종의 컨테이너
- 파이선은 데이터를 관리할 때 사용할 수 있도록 리스트, 튜플, 집합, 딕셔너리와 같은 내장 컨테이너 타입 제공

```python
class FrequencyList(list):
    def __init__(self, members):
        super().__init__(members)
        
    def frequency(self):
        counts = {}
				for item in self:
            counts[item] = counts.get(item, 0) + 1
        return counts

foo = FrequencyList(['a', 'b', 'a', 'c', 'b', 'a', 'd'])
print('길이:', len(foo))

foo.pop()
print('pop한 다음:', repr(foo))
print('빈도:', foo.frequency())

>>>
길이: 7
pop한 다음: ['a', 'b', 'a', 'c', 'b', 'a']
빈도: {'a': 3, 'b': 2, 'c': 1}
```

- `FrequencyList` 를 리스트의 하위 클래스로 설정하므로써 리스트가 제공하는 모든 함수를 `FrequenctList` 에서 사용할 수 있도록 설정

```python
class BinaryNode:
    def __init__(self, value, left=None, right=None):
        self.value = value
        self.left = left
        self.right = right

class IndexableNode(BinaryNode):
    def _traverse(self):
        if self.left is not None:
            yield from self.left._traverse()
        yield self
        if self.right is not None:
            yield from self.right._traverse()
            
    def __getitem__(self, index):
        for i, item in enumerate(self._traverse()):
            if i == index:
                return item.value
        raise IndexError(f'인덱스 범위 초과: {index}')

tree = IndexableNode(
    10,
    left=IndexableNode(
        5,
        left=IndexableNode(2),
        right=IndexableNode(
            6,
            right=IndexableNode(7))),
    right=IndexableNode(
        15,
        left=IndexableNode(11)))

print('LRR:', tree.left.right.right.value)
print('인덱스 0:', tree[0])
print('인덱스 1:', tree[1])
print('11이 트리 안에 있나?', 11 in tree)
print('17이 트리 안에 있나?', 17 in tree)
print('트리:', list(tree))

>>>
LRR: 7
인덱스 0: 2
인덱스 1: 5
11이 트리 안에 있나? True
17이 트리 안에 있나? False
트리: [2, 5, 6, 7, 10, 11, 15]

===========================================================

len(tree)
>>>
Traceback ...
TypeError: object of type 'IndexableNode' has no len()’
```

- 이진 트리 클래스를 시퀀스의 의미 구조를 사용해 다룰 수 있는 클래스
    - `__getitem__` 함수를 재구현
    - left 나 right 애트리뷰트를 사용해 순회 가능
    - 리스트처럼 접근도 가능
    
- 문제 : 리스트 인스턴스에 대해 모든 시퀀스의 의미 구조를 제공할 수 없음
    - 리스트에서 제공하는 내장 함수 사용 불가(구현하지 않았음!!)

## 2. `collections.abc` 내장 모듈

- 컨테이너 타입에 정의해야 하는 전형적인 메소드를 모두 제공하는 추상 기반 클래스 정의가 들어 있음
- 해당 추상 클래스의 하위 클래스를 만들고 필요한 메소드 구현을 하지 않으면 에러 발생

```python
from collections.abc import Sequence

class BadType(Sequence):
    pass

foo = BadType()

>>>
Traceback ...   
TypeError: Can't instantiate abstract class BadType with abstract methods __getitem__, __len__
```

- 위 예제를 `collections.abc` 내장 모듈에서 가져온 추상 기반 클래스가 요구하는 모든 메소드를 구현하면 해당 문제 해결 가능
    - 개발자 실수 최소화 가능
    - `index` 나 `count` 와 같은 추가 메서드 구현 안해도 거져 얻을 수 있음

```python
class SequenceNode(IndexableNode):
    def __len__(self):
        for count, _ in enumerate(self._traverse(), 1):
            pass
        return count

class BetterNode(SequenceNode, Sequence):
    pass

tree = BetterNode(
    10,
    left=BetterNode(
        5,
        left=BetterNode(2),
        right=BetterNode(
            6,
            right=BetterNode(7))),
    right=BetterNode(
        15,
        left=BetterNode(11))
)

print('7의 인덱스:', tree.index(7))
print('10의 개수:', tree.count(10))

>>>
7의 인덱스: 3
10의 개수: 1
```

- `set` 이나 `MutableMapping`  과 같이 파이선 관례에 맞춰 구현해야 하는 특별 메소드가 많은 복잡한 컨테이너 타입을 구현할 때 `collection.abc` 모듈을 사용하는 것은 장점이 많음