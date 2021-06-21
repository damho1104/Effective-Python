# 26. functools.wrap 을 사용해 함수 데코레이터를 정의하라

## 1. 데코레이터

- 감싸고 있는 함수의 실행 전과 후의 실행을 제어할 수 있도록 해줌
- 감싸고 있는 함수의 인자, 반환 값, 예외에 대해 조작 및 처리 가능

```python
def trace(func):
    def wrapper(*args, **kwargs):
        result = func(*args, **kwargs)
        print(f'{func._ _name__}({args!r}, {kwargs!r}) '
              f'-> {result!r}')
        return result
    return wrapper

@trace
def fibonacci(n):
    """n번째 피보나치 수를 반환한다."""
    if n in (0, 1):
        return n
    return (fibonacci(n - 2) + fibonacci(n - 1))

fibonacci(4)

>>>
fibonacci((0,), {}) -> 0
wrapper((0,), {}) -> 0
fibonacci((1,), {}) -> 1
wrapper((1,), {}) -> 1
fibonacci((2,), {}) -> 1
wrapper((2,), {}) -> 1
fibonacci((1,), {}) -> 1
wrapper((1,), {}) -> 1
fibonacci((0,), {}) -> 0
wrapper((0,), {}) -> 0
fibonacci((1,), {}) -> 1
wrapper((1,), {}) -> 1
fibonacci((2,), {}) -> 1
wrapper((2,), {}) -> 1
fibonacci((3,), {}) -> 2
wrapper((3,), {}) -> 2
fibonacci((4,), {}) -> 3
wrapper((4,), {}) -> 3

-----------------------------------------
print(fibonacci)

>>>
<function trace.<locals>.wrapper at 0x108955dc0>
```

- 데코레이터 함수: `trace`
- 감싼 함수: `fibonacci`
- `fibonacci` 를 실행한 후 결과를 출력하는 작업은 `trace` 에서 진행됨
- 그러나 fibonacci 함수를 호출했으나 결국 trace 함수 데코레이터가 반환하는 꼴
    - 이런 동작은 디버거와 같이 인트로스펙션2을 하는 도구에서 문제가 됨(Better way 80)

```python
# Case 1
help(fibonacci)

>>>
Help on function wrapper in module __main__:

wrapper(*args, **kwargs)

-----------------------------------------
# Case 2
import pickle

pickle.dumps(fibonacci)

>>>
Traceback ...
AttributeError: Can't pickle local object 'trace.<locals>.wrapper'
```

- Case 1: `help` 내장 함수를 사용하면 `fibonacci` 에 있는 독스트링 출력해야 하나 그렇지 않음
- Case 2: 데코레이터가 감싸고 있는 원래 함수의 위치를 찾을 수 없으므로 객체 직렬화도 깨짐

## 2. functools 내장 패키지의 wraps 함수

- `wraps` 를 `wrapper` 함수에 적용
    - 데코레이터 내부 함수의 메타데이터를 복사해 wrapper 함수에 적용

    ```python

    #from functools import wraps

    def trace(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            ...

        return wrapper
        
    @trace
    def fibonacci(n):
        ...

    # Case 1
    help(fibonacci)

    >>>
    Help on function fibonacci in module __main__:
    fibonacci(n)
        n번째 피보나치 수를 반환한다.

    ----------------------------------------
    # Case 2
    print(pickle.dumps(fibonacci))

    >>>
    b'\x80\x04\x95\x1a\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main__\x94\x8c\tfibonacci\x94\x93\x94.'
    ```

    - Case 1: `help` 함수 정상 작동
    - Case 2: 객체 직렬화 정상 작동
    - `wraps` 를 사용하면 함수 애트리뷰트도 복사하여 적용해줌

    ## 3. 내 의견

    - 데코레이터의 인자를 사용해야 하는 경우 `wraps` 붙이는게 잘 안되었음
    - 이 경우도 `wraps` 를 붙일 수 있는지 확인해봐야 겠음