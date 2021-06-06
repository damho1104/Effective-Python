# 20. None 을 반환하기보다는 예외를 발생시켜라

## 1. `None` 반환을 피해야 하는 이유

```python
def careful_divide(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        return None

x, y = 1, 0
result = careful_divide(x, y)
if result is None:
    print('잘못된 입력')
```

- `careful_divide` 함수 반환 결과를 if 문에서 평가할 때 bool 타입으로 검사하는 경우 0 값이 문제될 수 있음

```python
x, y = 0, 5
result = careful_divide(x, y)
if not result:
    print('잘못된 입력')  # 이 코드가 실행되는데, 사실 이 코드가 실행되면 안 된다!

>>>
잘못된 입력
```

## 2. 실수할 가능성을 줄이기 위한 방안

- 반환 값을 2-튜플로 분리하는 것
    - 튜플의 첫번째는 연산 결과를 반환
    - 튜플의 두번째는 실제 결과를 반환
    - 호출부에서 첫번 째 인자를 무시하는 경우 문제 발생(`_` 사용)
    - `None` 을 쓰나 이 방법을 쓰나 실수할 가능성은 존재함

    ```python
    def careful_divide(a, b):
        try:
            return True, a / b
        except ZeroDivisionError:
            return False, None

    _, result = careful_divide(x, y)
    if not success:
        print('잘못된 입력')
    ```

- `None` 값을 반환하지 않는 것
    - Exception을 호출한 쪽으로 발생시켜 호출자가 이를 처리할 수 있도록 함

    ```python
    def careful_divide(a, b):
        try:
            return a / b
        except ZeroDivisionError as e:
            raise ValueError('잘못된 입력')

    x, y = 5, 2
    try:
        result = careful_divide(x, y)
    except ValueError:
        print('잘못된 입력')
    else:
        print('결과는 %.1f 입니다' % result)

    >>>
    결과는 2.5 입니다
    ```

    - type annotation 을 사용하는 코드의 적용
        - **Python 의 gradual typing 에서는 함수의 인터페이스에 예외가 포함되는지 표현하는 방법(Checked Exception)이 의도적으로 제외되었음**
        - 호출자에서 어떤 exception 을 잡아야 하는지 문서에 작성해야 함

    ```python
    def careful_divide(a: float, b: float) -> float:
        """a를 b로 나눈다.
        Raises:
            ValueError: b가 0이어서 나눗셈을 할 수 없을 때
        """
        try:
            return a / b
        except ZeroDivisionError as e:
            raise ValueError('잘못된 입력')
    ```