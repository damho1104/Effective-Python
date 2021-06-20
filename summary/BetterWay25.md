# 25. 위치로만 인자를 지정하게 하거나 키워드로만 인자를 지정하게 해서 함수 호출을 명확하게 만들라

## 1. 위치 인자와 키워드 인자 혼합 사용의 문제점

- 위치 인자

    ```python
    def safe_division(number, divisor,
                      ignore_overflow,
                      ignore_zero_division):
        try:
            return number / divisor
        except OverflowError:
            if ignore_overflow:
                return 0
            else:
                raise
        except ZeroDivisionError:
            if ignore_zero_division:
                return float('inf')
            else:
                raise

    # Case 1(** 거듭제곱)
    result = safe_division(1.0, 10**500, True, False)
    print(result)

    >>>
    0

    # Case 2
    result = safe_division(1.0, 0, False, True)
    print(result)

    >>>
    inf
    ```

    - 해당 코드 단점
        - 어떤 예외를 무시할 지 결정하는 `bool` 변수 위치 혼동 가능성 증가
    - 방안
        - 키워드 인자 사용

    ```python
    def safe_division_b(number, divisor,
                        ignore_overflow=False,        # 변경
                        ignore_zero_division=False):  # 변경
        ...

    result = safe_division_b(1.0, 10**500, ignore_overflow=True)
    print(result)

    result = safe_division_b(1.0, 0, ignore_zero_division=True)
    print(result)

    >>>
    0
    inf
    ```

    - 단점
        - 키워드 인자만 사용하도록 강요 불가

    ## 2. 키워드 인자만 사용하도록 히는 방법

    ```python
    def safe_division_c(number, divisor, *,           # 변경
                        ignore_overflow=False,
                        ignore_zero_division=False):
        ...

    safe_division_c(1.0, 10**500, True, False)

    >>>
    Traceback ...
    TypeError: safe_division_c() takes 2 positional arguments but 4 were given

    -------------------------------------------

    result = safe_division_c(1.0, 0, ignore_zero_division=True)
    assert result == float('inf')

    try:
        result = safe_division_c(1.0, 0)
    except ZeroDivisionError:
        pass  # 예상대로 작동함
    ```

    - `safe_division_c` 인자의 `*` 은 위치 인자의 마지막과 키워드만 사용하는 인자의 시작을 구분하는 구분자
    - 키워드 인자를 사용하지 않고 위치 인자를 사용하면 예외 발생
    - 디폴트 인자는 잘 작동함
    - 문제점
        - number, divisor 인자를 혼동하는 경우
        - 앞 두 인자를 키워드 인자로 줬으나 두 인자 이름이 변경된 경우
    - 해결 방안(`python 3.8` 이상)
        - 위치로만 지정하는 인자를 구분할 수 있는 구분자 `/` 사용

    ## 3. 위치 인자만 사용할 수 있도록 구분하는 방법

    ```python
    def safe_division_d(numerator, denominator, /, *,     # 변경
                        ignore_overflow=False,
                        ignore_zero_division=False):
        ...

    # Case 1
    assert safe_division_d(2, 5) == 0.4

    # Case 2. 위치로만 지정하는 인자에 키워드 사용한 경우 예외
    safe_division_d(numerator=2, denominator=5)

    >>>
    Traceback ...
    TypeError: safe_division_d() got some positional-only arguments passed as keyword arguments: 'numerator, denominator’
    ```

    - 구분자 `/` 와 `*` 를 혼합하여 위치로만 혹은 키워드로만 사용하도록 하는 인자 구분

    ```python
    def safe_division_e(numerator, denominator, /,
                        ndigits=10, *,                 # 변경
                        ignore_overflow=False,
                        ignore_zero_division=False):
        try:
            fraction = numerator / denominator         # 변경
            return round(fraction, ndigits)            # 변경
        except OverflowError:
            if ignore_overflow:
                return 0
            else:
                raise
        except ZeroDivisionError:
            if ignore_zero_division:
                return float('inf')
            else:
                raise

    result = safe_division_e(22, 7)
    print(result)

    result = safe_division_e(22, 7, 5)
    print(result)

    result = safe_division_e(22, 7, ndigits=2)
    print(result)

    >>>
    3.1428571429
    3.14286
    3.14
    ```

    - 구분자 `/` 와 `*` 사이에 있는 모든 인자는 위치를 사용하거나 키워드를 사용하여 인자 전달 가능
    - 각자 원하는 API 스타일에 맞춰 개발하여 가독성을 높이고 잡음을 줄일 수 있도록 함

    ## 4. 내 의견

    - 구분자 `*` 만 사용하여 개발하는게 명확하지 않을까 싶음
    - 요즘 lint 에서 위치 인자에 필요한 인자가 뭔지 잘 가이드하므로 큰 문제 생기진 않을 것으로 보임
    - 키워드 인자만 강요하도록 하여 해당 API 의 혼동을 줄일 수 있도록 하는게 어떨까?