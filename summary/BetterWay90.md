# 90. typing 과 정적 분석을 통해 버그를 없애라

## 1. 컴파일 시점 타입 안정성

- API를 올바르게 사용하고 하위 의존 관계를 올바른 방법으로 활용하는지 검사하는 매커니즘 필요
    - 문서는 API 를 제대로 사용하는 방법을 알려주는 훌륭한 방법이지만 충분하지 않고 잘못 사용하면 여전히 버그가 생김
- 파이선은 역사적으로 동적인 기능이 초점을 맞춰 컴파일 시점 타입 안정성을 제공하지 않았음
    - 여러 프로그래밍 언어가 컴파일 시점 타입 검사 제공
    - 최근들어 특별한 구문과 `typing` 모듈이 도입되어 변수, 클래스 필드, 함수, 메소드에 타입 애너테이션을 붙일 수 있게 되었음
    - 타입 힌트를 사용하면 코드베이스를 점진적으로 변경하는 점진적 타입 지정 가능

## 2. 정적 분석 도구

- 타입 애너테이션을 추가하면 정적 분석 도구로 프로그램 소스 코드를 검사해서 버그가 생긴 가능성이 높은 부분을 식별할 수 있다는 장점 존재
    - `typing` 내장 모듈은 실제 그 자체로는 어떠한 타입 검사 기능도 제공하지 않음
    - 파이선 코드에 적용할 수 있고 별도의 도구가 소비하는 generics 를 포함한 타입 정의 시 공통 라이브러리 제공
        - generics
            - 여러 타입에 대해 작동가능한 일반적인 코드를 작성할 수 있게 해주는 기능
    - 파이선 정적 분석 도구
        - mypy
        - pytype
        - pyright
        - pyre
        
        ```bash
        $ python3 -m mypy --strict example.py
        ```
        
        - 프로그램을 실행하기 전 수많은 오류 감지 가능
    
- 다음 코드는 컴파일은 성공하지만 실행 시점에 예외가 발생하는 버그 예제
    
    ```python
    def subtract(a, b):
        return a - b
    
    subtract(10, '5')
    
    >>>
    Traceback ...
    TypeError: unsupported operand type(s) for -: 'int' and 'str'
    ```
    
    - 위 예제에 타입 애너테이션을 붙이면 아래와 같음
        - 파아미터와 변수 타입 애너테이션 사이는 콜론으로 구분
        - 반환값 타입은 함수 인자 목록 뒤에 `-> 타입` 형태
        - 타입 애너테이션과 mypy 를 사용하면 버그 탐지 가능
    
    ```python
    def subtract(a: int, b: int) -> int:  # 함수에 타입 애너테이션을 붙임
        return a - b
        
    subtract(10, '5')    # 아이고! 문자열 값을 넘김
    
    $ python3 -m mypy --strict example.py
    .../example.py:4: error: Argument 2 to "subtract" has incompatible type "str"; expected "int"
    ```
    
- 다른 실수는 python3 로 넘어오면서 `bytes` 와 `str` 인스턴스를 섞어쓰는 점
    - 타입 힌트와 mypy 를 사용하면 정적으로 문제 탐지 가능
    
    ```python
    def concat(a, b):
        return a + b
    
    concat('first', b'second')
    
    >>>
    Traceback ...
    TypeError: can only concatenate str (not "bytes") to str
    ```
    
    ```python
    def concat(a: str, b: str) -> str:
        return a + b
    
    concat('첫째', b'second')  # 아이고! bytes 값을 넘김
    $ python3 -m mypy --strict example.py
    .../example.py:4: error: Argument 2 to "concat" has incompatible type "bytes"; expected "str"
    ```
    
- 타입 애너테이션을 클래스에도 적용 가능
    
    ```python
    class Counter:
        def __init__(self):
            self.value = 0
            
        def add(self, offset):
            value += offset
            
        def get(self) -> int:
            self.value
    ```
    
    ```python
    counter = Counter()
    counter.add(5)
    
    >>>
    Traceback ...           
    UnboundLocalError: local variable 'value' referenced before assignment
    ```
    
    ```python
    counter = Counter()
    found = counter.get()
    assert found == 0, found
    
    >>>
    Traceback ...
    AssertionError: None
    ```
    
    - mypy 를 사용하면 위 2개 문제 탐지 가능
    
    ```python
    class Counter:
        def __init__(self) -> None:
            self.value: int = 0  # 필드/변수 애너테이션
            
        def add(self, offset: int) -> None:
            value += offset      # 아이고! 'self.'를 안 씀
            
        def get(self) -> int:
            self.value           # 아이고! 'return'을 안 씀
            
    counter = Counter()
    counter.add(5)
    counter.add(3)
    assert counter.get() == 8
    
    $ python3 -m mypy --strict example.py
    .../example.py:6: error: Name 'value' is not defined
    .../example.py:8: error: Missing return statement
    ```
    

- 동적으로 작동하는 파이선 강점은 덕 타입에 대해 작동하는 제너릭 가능을 작성하기 쉽다는 점
    - 덕 타입에 대한 제너릭 기능을 사용하면 한 구현으로 다양한 타입 처리 가능
        - 반복적인 수고를 줄일 수 있음
        - 테스트 단순화 가능
- 리스트와 값을 모두 조합하는 덕 타입을 지원하는 제너릭 함수 정의 예제
    - 그러나 마지막 단언문 실패
    
    ```python
    def combine(func, values):
        assert len(values) > 0
        
        result = values[0]
        for next_value in values[1:]:
            result = func(result, next_value)
            
        return result
        
    def add(x, y):
        return x + y
    
    inputs = [1, 2, 3, 4j]
    result = combine(add, inputs)
    assert result == 10, result  # 실패함
    
    >>>
    Traceback ...
    AssertionError: (6+4j)
    ```
    
    - `typing` 모듈의 제너릭 지원을 사용하면 함수에 애너테이션 추가 가능, 문제 발견 가능
    
    ```python
    from typing import Callable, List, TypeVar
    
    Value = TypeVar('Value')
    Func = Callable[[Value, Value], Value]
    
    def combine(func: Func[Value], values: List[Value]) -> Value:
        assert len(values) > 0
    
        result = values[0]
        for next_value in values[1:]:
            result = func(result, next_value)
    
        return result
    
    Real = TypeVar('Real', int, float)
    
    def add(x: Real, y: Real) -> Real:
        return x + y
        
    inputs = [1, 2, 3, 4j]  # 아이고!: 복소수를 넣었다
    result = combine(add, inputs)
    assert result == 10
    
    $ python3 -m mypy --strict example.py
    .../example.py:21: error: Argument 1 to "combine" has incompatible type "Callable[[Real, Real], Real]"; expected "Callable[[complex, complex], complex]"
    ```
    
- 객체의 `None` 여부 문제
    
    ```python
    def get_or_default(value, default):
        if value is not None:
            return value
        return value
        
    found = get_or_default(3, 5)
    assert found == 3
    
    found = get_or_default(None, 5)
    assert found == 5, found  # 실패함
    
    >>>
    Traceback ...
    AssertionError: None
    ```
    
    - `typing` 모듈은 선택적인 타입 지원
    - 선택적인 타입은 프로그램이 널 검사를 제대로 수행한 경우에만 값을 다룰 수 있게 강제
    
    ```python
    from typing import Optional
    
    def get_or_default(value: Optional[int],
                       default: int) -> int:
        if value is not None:
            return value
        return value  # 아이고!: 'default'를 반환해야 하는데 'value'를 반환했다
        
    $ python3 -m mypy --strict example.py
    .../example.py:7: error: Incompatible return value type (got "None", expected "int")
    ```
    

- `typing` 모듈은 이외에도 다양한 기능 제공
    - [https://docs.python.org/3.8/library/typing](https://docs.python.org/3.8/library/typing)
    - 예외를 인터페이스 정의의 일부분으로 간주하지 않음
    - 예외를 제대로 발생시키고 잡아내는지 검증이 필요하면 테스트를 작성해야 함

## 3. `typing` 모듈 사용 시 빠지게 되는 함정

- 전방 참조를 처리할 때 생길 수 있는 문제

```python
class FirstClass:
    def __init__(self, value):
        self.value = value
        
class SecondClass:
    def __init__(self, value):
        self.value = value
        
second = SecondClass(5)
first = FirstClass(second)
```

- 해당 프로그램에 타입 힌트를 추가하고 mypy 를 실행해도 아무 문제 없다고 보고

```python
class FirstClass:
    def __init__(self, value: SecondClass) -> None:
        self.value = value
        
class SecondClass:
    def __init__(self, value: int) -> None:
        self.value = value
        
second = SecondClass(5)
first = FirstClass(second)

$ python3 -m mypy --strict example.py
```

- 그러나 실제로 이 코드를 실행하면 `SecondClass` 가 실제로 정의되기 전에 `FirstClass.__init__` 메소드의 파라미터에서 `SecondClass` 를 사용하므로 실패

```python
class FirstClass:
    def __init__(self, value: SecondClass) -> None:  # 깨짐
        self.value = value
        
class SecondClass:
    def __init__(self, value: int) -> None:
        self.value = value
        
second = SecondClass(5)
first = FirstClass(second)

>>>
Traceback ...
NameError: name 'SecondClass' is not defined
```

- 정적 분석 도구가 지원하는 방법 중 하나는 전방 참조가 포함된 타입 애너테이션을 표현할 때 문자열을 쓰는 것
    - 분석 도구는 이 문자열을 구문 분석해서 체크할 타입 정보 추출
    
    ```python
    class FirstClass:
        def __init__(self, value: 'SecondClass') -> None:  # OK
            self.value = value
            
    class SecondClass:
        def __init__(self, value: int) -> None:
            self.value = value
            
    second = SecondClass(5)
    first = FirstClass(second)
    ```
    
- `from __future__ import annotations` 사용
    - python 3.7 이상
    - 해당 임포트는 파이선 인터프리터가 프로그램을 실행할 때 타입 애너테이션에 지정된 값을 무시하라 지시
        - 전방 참조 문제 해결, 프로그램 시작할 때 성능 향상 가능
    
    ```python
    from __future__ import annotations
    
    class FirstClass:
        def __init__(self, value: SecondClass) -> None:  # OK
            self.value = value
            
    class SecondClass:
        def __init__(self, value: int) -> None:
            self.value = value
            
    second = SecondClass(5)
    first = FirstClass(second)
    ```
    

## 4. 정리

- 다음은 염두에 둘 만한 모범적인 사용법
    - 일반적인 전략은 아무 타입 애너테이션도 사용하지 않으면서 최초 버전을 작성하고, 이어서 테스트를 작성한 다음, 타입 정보가 가장 유용하게 쓰일 수 있는 곳에 타입 정보를 추가하는 것이다.
        - 새로운 코드를 작성하면서 처음부터 타입 애너테이션을 사용하려고 하면 개발 과정이 느려진다.
    - 타입 힌트는 여러분의 코드에 의존하는 많은 호출자(따라서 다른 사람들)에게 기능을 제공하는 API와 같이 코드베이스의 경계에서 가장 중요하다.
        - 타입 힌트는 API를 변경해도 API를 호출하는 사람들이 예기치 못한 오류를 보거나 코드가 깨지는 일이 없도록 하기 위해 통합 테스트([Better way 77](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay77.md))나 경고([Better way 89](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay89.md))를 보완한다.
    - API의 일부분이 아니지만 코드베이스에서 가장 복잡하고 오류가 발생하기 쉬운 부분에 타입 힌트를 적용해도 유용할 수 있다.
        - 타입 힌트를 코드의 모든 부분에 100% 적용하는 것은 바람직하지 않다.
            - 타입을 추가하다 보면, 타입을 추가해서 얻을 수 있는 이익(한계 이익)이 점점 줄어들기 마련이다(한계수확 체감 법칙(low of diminishing returns)).
    - 가능하면 여러분의 자동 빌드와 테스트 시스템의 일부분으로 정적 분석을 포함시켜서 코드베이스에 커밋할 때마다 오류가 없는지 검사해야 한다.
        - 추가로 타입 검사에 사용할 설정을 저장소에 유지해서 여러분이 협업하는 모든 사람이 똑같은 규칙을 사용하게 해야 한다.
    - 코드에 타입 정보를 추가해나갈 때는 타입을 추가하면서 타입 검사기를 실행하는 일이 중요하다.
        - 타입을 추가하면서 타입 검사기를 실행하지 않으면, 타입 힌트를 여기저기 흩뿌려 놓으면서 타입 검사 도구가 엄청나게 많은 오류를 표시하는 것을 보게 된다.
        - 이런 일이 발생하면, 낙담해서 결국 타입 힌트를 아예 사용하지 않게 될 수도 있다.