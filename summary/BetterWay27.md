# 27. map과 filter 대신 컴프리헨션을 사용하라

## 1. 리스트 컴프리헨션

- 다른 시퀀스나 이터러블에서 새 리스트를 만들어내는 간결한 구문
- 예제

    ```python
    a = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10] 
    squares = []
    for x in a:
        squares.append(x**2)
    print(squares)

    >>>
    [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
    ```

    - `for` 루프를 사용하여 `square` 리스트 생성

    ```python
    squares = [x**2 for x in a]  # 리스트 컴프리헨션 
    print(squares)

    >>>
    [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
    ```

    - 리스트 컴프리헨션 사용하여 `square` 리스트 생성

    ```python
    alt = map(lambda x: x**2, filter(lambda x: x % 2 == 0, a))
    assert even_squares == list(alt)
    ```

    - `map` 과 `filter` 를 사용하여 리스트 생성

    ```python
    even_squares = [x**2 for x in a if x % 2 == 0]
    print(even_squares)

    >>>
    [4, 16, 36, 64, 100]
    ```

    - 리스트 컴프리헨션과 `if` 문을 사용하여 리스트 생성

- map 과 filter 를 사용하는 방법은 같은 결과를 얻지만 코드의 가독성이 떨어짐

## 2. 딕셔너리와 집합 컴프리헨션

```python
even_squares_dict = {x: x**2 for x in a if x % 2 == 0}
threes_cubed_set = {x**3 for x in a if x % 3 == 0}
print(even_squares_dict)
print(threes_cubed_set)

>>>
{2: 4, 4: 16, 6: 36, 8: 64, 10: 100}
{216, 729, 27}
```