# 87. 호출자를 API로부터 보호하기 위해 최상위 Exception 을 정의하라

## 1. 모듈 API 에서의 예외

- 모듈에서 최상위 Exception 을 정의
    - 모듈이 발생시키는 다른 모든 예외가 이 최상위 예외를 상속
    - API에서 발생하는 예외 계층 구조 생성 가능

```python
# my_module.py
class Error(Exception):
    """이 모듈에서 발생할 모든 예외의 상위 클래스."""

class InvalidDensityError(Error):
    """밀도 값이 잘못된 경우."""

class InvalidVolumeError(Error):
    """부피 값이 잘못된 경우."""

def determine_weight(volume, density):
    if density < 0:
        raise InvalidDensityError('밀도는 0보다 커야 합니다')
    if volume < 0:
        raise InvalidVolumeError('부피는 0보다 커야 합니다')
    if volume == 0:
        density / volume
```

- 어떤 모듈 안에 최상위 예외가 있으면 API 사용자들이 이 모듈에서 발생한 오류를 쉽게 캐치 가능
    - 함수 호출에 `try/except` 문을 사용하여 최상위 예외 잡아낼 수 있음

```python
try:
    weight = my_module.determine_weight(1, -1)
except my_module.Error:
    logging.exception('예상치 못한 오류')

>>>
ERROR:root:예상치 못한 오류
Traceback (most recent call last):
  File "main.py", line 5, in <module>
    weight = my_module.determine_weight(1, -1)
  File "...my_module.py", line 14 in determine_weight
    raise InvalidDensityError('밀도는 0보다 커야 합니다')
my_module.InvalidDensityError: 밀도는 0보다 커야 합니다
```

- `logging.exception` 함수가 잡아낸 예외의 전체 스택 trace 를 출력하므로 더 쉽게 디버깅 가능
    - `try/except` 문 사용하여 프로그램 깨지는 상황 방지

- 최상위 예외로 API 로부터 호출하는 코드 보호 가능
- 보호로 인한 3가지 효과
    - **효과 1.** 최상위 예외가 있으면 API를 호출하는 사용자가 API를 잘못 사용한 경우를 더 쉽게 이해할 수 있음
        - 사용자가 API 에서 의도적으로 발생시킨 여러 예외를 잡아내지 않으면 모듈의 최상위 예외를 잡아내는 방어적인 `except` 블록까지 예외 전달됨
            - 이 블록은 API 사용자의 주의 환기 가능
            - 사용자가 깜빡한 예외 타입 제대로 처리할 수 있는 기회 제공
        
        ```python
        try:
            weight = my_module.determine_weight(-1, 1)
        except my_module.InvalidDensityError:
            weight = 0
        except my_module.Error:
            logging.exception('호출 코드에 버그가 있음')
        
        >>>
        ERROR:root:호출 코드에 버그가 있음
        Traceback (most recent call last):
          File "main2.py", line 5, in <module>
            weight = my_module.determine_weight(-1, 1)
          File "...", line 16, in determine_weight
            raise InvalidVolumeError('부피는 0보다 커야 합니다')
        my_module.InvalidVolumeError: 부피는 0보다 커야 합니다
        ```
        
    - **효과 2.** API 모듈 코드의 버그를 발견할 때 도움될 수 있음
        - 모듈에서 예외 계층에 속하지 않는 다른 타입의 예외가 발생하면 구현한 API 코드에 버그가 있다는 뜻
        - 최상위 예외를 장바내는 `try/except` 문이 버그로부터 API 소비자들을 보호하지 못함
        - 호출하는 쪽에서 `Except` 클래스를 잡아내는 다른 `except` 블록 추가해야 함
        - 2 가지 `except` 문 사용하면 API 소비자가 API 모듈에 수정해야 할 버그가 있는 경우를 쉽게 탐색 가능
        
        ```python
        try:
            weight = my_module.determine_weight(0, 1)
        except my_module.InvalidDensityError:
            weight = 0
        except my_module.Error:
            logging.exception('호출 코드에 버그가 있음')
        except Exception:
            logging.exception('API 코드에 버그가 있음!')
            raise  # 예외를 호출자 쪽으로 다시 발생시킴
        
        >>>
        ERROR:root:API 코드에 버그가 있음!
        Traceback (most recent call last):
          File "example.py", line 3, in <module>
            weight = my_module.determine_weight(0, 1)
          File ".../my_module.py", line 14, in determine_weight
            density / volume
        ZeroDivisionError: division by zero
        Traceback ...
        ZeroDivisionError: division by zero
        ```
        
    - **효과 3.** 미래의 API 보호 가능
        - 시간이 지남에 따라 API 를 확장해 특정 상황에서 더 구체적인 예외를 제공하고 싶을 수 있음
        - 아래 예제는 밀도가 음수인 경우를 오류 조건으로 표시하는 `Exception` 하위 클래스 추가
        
        ```python
        # my_module.py
        ...
        
        class NegativeDensityError(InvalidDensityError):
            """밀도가 음수인 경우."""
        ...
        
        def determine_weight(volume, density):
            if density < 0:
                raise NegativeDensityError('밀도는 0보다 커야 합니다')
            ...
        ```
        
        - 모듈을 호출하는 코드는 변경하지 않아도 예전과 동일함
        - `InvalidDensityError` 예외를 이미 처리하기 때문
        - 추후 새로운 타입의 예외를 더 처리하기로 결정하면 그에 따라 동작 적절히 수정 가능
        
        ```python
        try:
            weight = my_module.determine_weight(1, -1)
        except my_module2.NegativeDensityError as exc:
            raise ValueError('밀도로 음수가 아닌 값을 제공해야 합니다') from exc
        except my_module.InvalidDensityError:
            weight = 0
        except my_module.Error:
            logging.exception('호출 코드에 버그가 있음')
        except Exception:
            logging.exception('API 코드에 버그가 있음!')
            raise
        
        >>>
        Traceback ...
        NegativeDensityError: 밀도는 0보다 커야 합니다
        
        The above exception was the direct cause of the following exception:
        
        Traceback ...
        ValueError: 밀도로 음수가 아닌 값을 제공해야 합니다
        ```
        
    
- 최상위 예외 바로 알애 폭넓은 예외 상황을 표현하는 다양한 오류를 제공
    - 미래의 코드 변경에 대한 보호를 더 강화할 수 있음
- 무게 계산 관련 예외, 부피 계산 관련 예외, 밀도 계산 관련 예외 추가 예제

```python
# my_module.py
class Error(Exception):
    """이 모듈에서 발생할 모든 예외의 상위 클래스."""

class WeightError(Error):
    """무게 계산 관련 예외의 상위 클래스."""

class VolumeError(Error):
    """부피 계산 관련 예외의 상위 클래스."""

class DensityError(Error):
    """밀도 계산 관련 예외의 상위 클래스."""
...
```

## 2. 정리

- 구체적인 예외는 일반적인 예외를 상속
- 각각의 중간 단계 예외는 각각 최상위 예외 역할 수행
    - API 코드로부터 API 를 호출하는 코드를 보호하는 계층 쉽게 추가 가능
- 모든 호출 코드가 구체적인 `Exception` 하위 클래스 예외를 일일이 처리하게 하는 것보다 예외 계층 구조를 채택하는 것이 나음