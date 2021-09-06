# 38. 간단한 인터페이스의 경우 클래스 대신 함수를 받아라

## 1. 훅(hook)

- API 가 실행되는 과정에서 전달한 함수를 실행하는 경우
- `sort` 메소드에서 `key` 를 훅으로 `len` 내장 함수를 전달하는 예제

```python
names = ['소크라테스', '아르키메데스', '플라톤', '아리스토텔레스']
names.sort(key=len)
print(names)

>>>
['플라톤', '소크라테스', '아르키메데스', '아리스토텔레스']
```

- `defaultdict` 를 사용하여 훅 사용하는 예제
    - 훅 함수: `log_missing`

```python
from collections import defaultdict

def log_missing():
    print('키 추가됨')
    return 0

current = {'초록': 12, '파랑': 3}
increments = [
    ('빨강', 5),
    ('파랑', 17),
    ('주황', 9),
]
result = defaultdict(log_missing, current)
print('이전:', dict(result))
for key, amount in increments:
    result[key] += amount
print('이후:', dict(result))

>>>
이전: {'초록': 12, '파랑': 3}
키 추가됨
키 추가됨
이후: {'초록': 12, '파랑': 20, '빨강': 5, '주황': 9}
```

## 2. 훅 함수에서 상태를 처리할 수 있도록 확장한 예제

- `missing` 훅 함수에 `added_count` 변수에 상태를 기록

```python
def increment_with_report(current, increments):
    added_count = 0

    def missing():
        nonlocal added_count  # 상태가 있는 클로저
        added_count += 1
        return 0

    result = defaultdict(missing, current)
    for key, amount in increments:
        result[key] += amount

    return result, added_count

result, count = increment_with_report(current, increments)
assert count == 2
```

- 간단한 클래스로 상태를 관리하는 예제

```python
class CountMissing:
    def __init__(self):
        self.added = 0
        
    def missing(self):
        self.added += 1
        return 0

counter = CountMissing()
result = defaultdict(counter.missing, current)  # 메서드 참조
for key, amount in increments:
    result[key] += amount
assert counter.added == 2

```

- 헬퍼 클래스로 상태가 있는 클로저와 같은 동작 구현이 함수 사용한 구현보다 깔끔함
- 그러나 클래스만 보면 해당 클래스 정의 목적이 드러나지 않음
    - 누가 만들지?
    - 누가 호출하지?
    - 추후 공개 메소드가 더 추가될 가능성은?
    - ...
- `__call__` 특별 메소드 정의한 예제

```python
class BetterCountMissing:
    def __init__(self):
        self.added = 0
        
    def __call__(self):
        self.added += 1
        return 0
        
counter = BetterCountMissing()
assert counter() == 0
assert callable(counter)
```

- **`__call__`, callable 객체**
    - 객체를 함수처럼 호출 가능함
    - `__call__` 이 정의된 클래스 인스턴스에 대해 `callable` 내장 함수를 호출하면 `True` 반환됨

```python
counter = BetterCountMissing()
result = defaultdict(counter, current)  # __call__에 의존함
for key, amount in increments:
    result[key] += amount
assert counter.added == 2
```

## 3. 정리

- 여러 컴포넌트 사이 간단한 인터페이스가 필요할 때는 클래스 인스턴스화하는 대신 함수를 사용하는게 이득
- `callable` 객체를 사용하면 함수처럼 호출 가능
- 상태를 관리하는 함수가 필요한 경우 `callable` 객체를 사용해야 할지를 고민할 것