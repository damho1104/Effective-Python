# 31. 인자에 대해 이터레이션할 때는 방어적이 돼라

## 1. 이터레이터 사용 시 주의점

```python
def normalize(numbers):
    total = sum(numbers)
    result = []
    for value in numbers:
        percent = 100 * value / total
        result.append(percent)
    return result

visits = [15, 35, 80]
percentages = normalize(visits)
print(percentages)
assert sum(percentages) == 100.0

>>>
[11.538461538461538, 26.923076923076923, 61.53846153846154]
```

⇒

```python
def read_visits(data_path):
    with open(data_path) as f:
        for line in f:
            yield int(line)

def normalize(numbers):
    total = sum(numbers)
    result = []
    for value in numbers:
        percent = 100 * value / total
        result.append(percent)
    return result

it = read_visits('my_numbers.txt')
percentages = normalize(it)
print(percentages)

>>>
[]
```

- 이터레이터가 결과를 한번만 만들어내서 `stopIteration` 예외 발생했거나 **제너레이터를 다시 이터레이션**하면 결과가 없음
- 이터레이터 없어진 예

    ```python
    it = read_visits('my_numbers.txt')
    print(list(it))
    print(list(it))  # 이미 모든 원소를 다 소진했다

    >>>
    [15, 35, 80]
    []
    ```

## 2. 이터레이터 사용 시 문제 해결 방안

- 이터레이터를 명시적으로 소진시키기
    - 예(이터레이터 복사)

    ```python
    def normalize_copy(numbers):
        numbers_copy = list(numbers)  # 이터레이터 복사 
        total = sum(numbers_copy)
        result = []
        for value in numbers_copy:
            percent = 100 * value / total
            result.append(percent)
        return result
    ```

    - 해당 예제 **단점**
        - 이터레이터를 복사하여 사용하므로 **메모리 공간 낭비 발생 가능성이 높음**
    - 단점 우회 방안
        1. 호출될 때마다 새로 이터레이터 반환하는 함수를 받도록 하기

        ```python
        def normalize_func(get_iter):
            total = sum(**get_iter()**)   # 새 이터레이터 
            result = []
            for value in **get_iter()**:  # 새 이터레이터
                percent = 100 * value / total
                result.append(percent)
            return result

        path = 'my_numbers.txt'
        percentages = normalize_func(lambda: read_visits(path))
        print(percentages)
        assert sum(percentages) == 100.0

        >>>
        [11.538461538461538, 26.923076923076923, 61.53846153846154]
        ```

        1. 이터레이터 프로토콜을 구현한 새로운 컨테이너 클래스 제공

        ```python
        class ReadVisits:
            def __init__(self, data_path):
                self.data_path = data_path
                
            def __iter__(self):
                with open(self.data_path) as f:
                    for line in f:
                        yield int(line)

        def normalize(numbers):
            total = sum(numbers)
            result = []
            for value in numbers:
                percent = 100 * value / total
                result.append(percent)
            return result

        visits = ReadVisits(path)
        percentages = normalize(visits)
        print(percentages)
        assert sum(percentages) == 100.0

        >>>
        [11.538461538461538, 26.923076923076923, 61.53846153846154]
        ```

        - `sum` 메소드가 `ReadVisits` 의 `__iter__` 를 호출하여 새로운 이터레이터 객체 할당
        - `for` 루프에서도 이전 이터레이터와 다른 이터레이터 새로 할당
            - iter 내장 함수 VS 컨테이너 타입
                - **iter 내장 함수**: 전달 받은 이터레이터 그대로 반환
                - **컨테이너 타입**: 새로운 이터레이터 객체 반환
        - 입력 인자로부터 전달 받은 이터레이션을 검사하여 이터레이션할 수 없는 인자인 경우 TypeError 발생시킬 수 있음

        ```python
        def normalize_defensive(numbers):
            if iter(numbers) is numbers:  # 이터레이터# -- 나쁨! 
                raise TypeError('컨테이너를 제공해야 합니다')
            total = sum(numbers)
            result = []
            for value in numbers:
                percent = 100 * value / total
                result.append(percent)
            return result
        ```

        - `collections.abc` 내장 모듈 사용한 검사 방법

        ```python
        from collections.abc import Iterator

        def normalize_defensive(numbers):
            if isinstance(numbers, Iterator): # 반복 가능한 이터레이터인지 검사하는 다른 방법
                raise TypeError('컨테이너를 제공해야 합니다')
            total = sum(numbers)
            result = []
            for value in numbers:
                percent = 100 * value / total
                result.append(percent)
            return result
        ```