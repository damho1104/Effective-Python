# 15. 딕셔너리 삽입 순서에 의존할 때는 조심하라

## 1. Dictionary python 3.5 vs python 3.6 이상

- python 3.5
    - dictionary 삽입 순서와 이터레이션 순서 일치하지 않음
    - 내장 hash 함수와 seed 사용한 해시 알고리즘 사용
- python 3.6
    - 삽입 순서 보존
    - python 3.7 부터 언어 명세에 포함되었음

```python
python 3.6 이상

baby_names = {
    'cat': 'kitten',
    'dog': 'puppy',
}
print(baby_names)

>>>    
{'cat': 'kitten', 'dog': 'puppy'}
```

## 2. dictionary 변경으로 인한 side-effect

- 함수의 키워드 인자, `keys`, `values`, `items`, `popitems` 가 순서에 의존하도록 코드 작성 가능
- `class` 도 인스턴스 dictionary에 `dict` 타입 사용

```python
# Python 3.5
class MyClass:
    def __init__(self):
        self.alligator = 'hatchling'
        self.elephant = 'calf'
        
a = MyClass()
for key, value in a.__dict__.items():
    print('%s = %s' % (key, value))

>>>
elephant = calf
alligator = hatchling

---------------------------------------------

# Python 3.6
class MyClass:
    def __init__(self):
        self.alligator = 'hatchling'
        self.elephant = 'calf'
        
a = MyClass()
for key, value in a.__dict__.items():
    print(f'{key} = {value}')

>>>
alligator = hatchling
elephant = calf
```

## 3. 주의해야 할 점

- 커스텀 컨테이너를 사용하여 dictionary 를 비슷하게 구현한 경우 삽입 순서는 유지되지 않을 수 있음
- 개발자의 커스텀에 따라 정렬 순서가 정해질 수 있으므로 삽입 순서에 의존하면 안됨
- 해결방안
    1. 어떤 특정 순서로 이터레이션된다고 가정하지 말고 구현
    2. dict 인지 커스텀 타입인지 타입 체킹
    3. `type annotation` 사용하여 타입 명시

        ```python
        from typing import Dict, MutableMapping

        def populate_ranks(votes: Dict[str, int],
                           ranks: Dict[str, int]) -> None:
            names = list(votes.keys())
            names.sort(key=votes.get, reverse=True)
            for i, name in enumerate(names, 1):
                ranks[name] = i
                
        def get_winner(ranks: Dict[str, int]) -> str:
            return next(iter(ranks))
            
        class SortedDict(MutableMapping[str, int]):
            ...
            
        votes = {
            'otter': 1281,
            'polar bear': 587,
            'fox': 863,
        }

        sorted_ranks = SortedDict()
        populate_ranks(votes, sorted_ranks)
        print(sorted_ranks.data)
        winner = get_winner(sorted_ranks)
        print(winner)

        $ python3 -m mypy --strict example.py
        .../example.py:48: error: Argument 2 to "populate_ranks" has incompatible type "SortedDict"; expected "Dict[str, int]"
        .../example.py:50: error: Argument 1 to "get_winner" has incompatible type "SortedDict"; expected "Dict[str, int]
        ```

## Note.

- collections 내장 모듈의 OrderedDict 의 경우 key 삽입과 popitem 을 자주 사용하는 경우 사용