# 21. 변수 영역과 클로저의 상호작용 방식을 이해하라

## 1. 클로저

- 자신이 정의된 영역 밖의 변수를 참조하는 함수(라고 책에 적혀있지만 기법이 맞는듯 하다...)
- inner function 에서 outer function에 있는 변수를 접근할 수 있는 기법을 뜻함

```python
def sort_priority(values, group):
    def helper(x):
        if x in group:
            return (0, x)
        return (1, x)
    values.sort(key=helper)

numbers = [8, 3, 1, 2, 5, 4, 7, 6]
group = {2, 3, 5, 7}
sort_priority(numbers, group)
print(numbers)

>>>
[2, 3, 5, 7, 1, 4, 6, 8]
```

- 위 예제에서 inner function `helper`는 outer function `sort_priority` 의 `group` 을 참조하고 있음

※ `first-class functions` : 함수를 직접 가리킬 수 있고 변수에 대입하거나 다른 함수의 인자로 전달 가능, 내가 정의를 하자면 함수의 변수화 가능

## 2. 변수 영역

- 4가지 순서로 변수 영역 탐색
    - 현재 함수의 영역
    - 현재 함수를 둘러싸고 있는 영역
    - 현재 코드가 들어있는 모듈 영역(global scope)
    - 내장 영역( built-in scope, ex) `len`, `str` )

```python
def sort_priority2(numbers, group):
    found = False
    def helper(x):
        if x in group:
                found = True  # 여기!!
                return (0, x)
        return (1, x)
    numbers.sort(key=helper)
    return found

found = sort_priority2(numbers, group)
print('발견:', found)
print(numbers)

>>>
발견: False
[2, 3, 5, 7, 1, 4, 6, 8]
```

- 위 예제는 `helper` 에서 outer function `sort_priority2` 의 `found` 변수를 참조하려 하였음
- 그러나 결과는 `false`
- 이유
    - 4가지 변수 영역 탐색 순서로 인해 `helper`에서의 `found` 는 ***현재 함수 영역에서 정의된 새 변수*** 로 취급하였기 때문
- 방법
    - `nonlocal` 사용

```python
def sort_priority2(numbers, group):
    found = False
    def helper(x):
        nonlocal found       # 추가함
        if x in group:
                found = True
                return (0, x)
        return (1, x)
    numbers.sort(key=helper)
    return found
```

- `nonlocal` 키워드를 사용하면 지정된 변수는 변수 영역 결정 규칙에 의거해 대입될 변수의 영역이 결정됨
- 한계
    - **global scope** 까지 탐색하지는 **않음**
- 사실 `nonlocal`, `global` 은 **안티 패턴**
    - 이유: 가독성을 해치기 때문
- `nonlocal` 대체 방안
    - 클래스(with `__call__`)로 대체

```python
class Sorter:
    def __init__(self, group):
        self.group = group
        self.found = False
        
    def __call__(self, x):
        if x in self.group:
            self.found = True
            return (0, x)
        return (1, x)
        
sorter = Sorter(group)
numbers.sort(key=sorter)
assert sorter.found is True
```