# 19. 함수가 여러 값을 반환하는 경우 절대로 4 값 이상을 언패킹하지 말라

## 1. 여러 값을 반환하는 함수의 문제점

```python
def get_stats(numbers):
    minimum = min(numbers)
    maximum = max(numbers)
    count = len(numbers)
    average = sum(numbers) / count

    sorted_numbers = sorted(numbers)
    middle = count // 2
    if count % 2 == 0:
        lower = sorted_numbers[middle - 1]
        upper = sorted_numbers[middle]
        median = (lower + upper) / 2
    else:
        median = sorted_numbers[middle]
    
    return minimum, maximum, average, median, count

minimum, maximum, average, median, count = get_stats(lengths)

print(f'최소 길이: {minimum}, 최대 길이: {maximum}')
print(f'평균: {average}, 중앙값: {median}, 개수: {count}')

>>>
최소 길이: 60, 최대 길이: 73
평균: 67.5, 중앙값: 68.5, 개수: 10
```

- 단점
    - 모든 반환값이 `int` 라 순서를 혼동하기 쉬움
    - 가독성 떨어짐
        - 함수 호출과 반환 값을 언패킹하는 부분이 길다
        - 여러 가지 방법으로 줄을 바꿀 수 있음

## 2. 그럼 왜 4개 이상부터 하지 말라고 하는 걸까?

- 3개까지는 통용되는 순서가 있음
    - 따로 변수에 넣는 구문일 수도
    - 두 값을 변수에 넣고 나머지 모든 값을 한 변수에 넣는 구문일 수도 있음

## 3. 그러면 4개 이상 반환해야 할 경우는 어떻게?

- 경량 클래스(lightweight class)
- `namedtuple` 사용(값 초기화 이후 변경 불가, immutable)

    ```python
    from collections import namedtuple

    Color = namedtuple("Color", ["red", "green", "blue"])

    color = Color(55, 155, 255)
    print(color.red)

    >>>
    55
    ```