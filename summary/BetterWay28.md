# 28. 컴프리헨션 내부에 제어 하위 식을 세 개 이상 사용하지 말라

## 1. 루프

```python
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
squared = [[x**2 for x in row] for row in matrix]
print(squared)

>>>
[[1, 4, 9], [16, 25, 36], [49, 64, 81]]
```

- 컴프리헨션에 리스트 표현(`[]`)이 추가되므로 조금 복잡해보이지만 그나마 읽기 쉬운 편
- 그러나 이 이상으로 루프나 제어식이 존재한다면 코드가 길어지므로 여러 줄로 표현해야 함

```python
my_lists = [
    [[1, 2, 3], [4, 5, 6]],
    ...
                        
]
flat = [x for sublist1 in my_lists
        for sublist2 in sublist1
        for x in sublist2]
```

- 다중 컴프리헨션 표현식이 다른 대안에 비해 더 길어보임

```python
flat = []
for sublist1 in my_lists:
    for sublist2 in sublist1:
        flat.extend(sublist2)
```

- 2중 `for` 루프로 표현하는 것이 더 명확

## 2. **조건문**

```python
a = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
b = [x for x in a if x > 4 if x % 2 == 0]
c = [x for x in a if x > 4 and x % 2 == 0]
```

- 여러 조건을 같은 수준의 루프에 사용하면 암시적으로 `and` 식을 의미
- `for` 하위 식의 바로 뒤에 `if` 를 추가함으로써 각 수준마다 조건을 지정 가능

```python
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
filtered = [[x for x in row if x % 3 == 0]
            for row in matrix if sum(row) >= 10]
print(filtered)

>>>
[[6], [9]]
```

- 복합적인 리스트, 집합, 딕셔너리 컴프리헨션은 사용하지 말 것
    - 코드 가독성 떨어짐

## 3. 방법

- 컴프리헨션에 들어가는 하위 식이 **세 개 이상** 되지 않게 제한하라는 규칙을 지킬 것
    - 조건문 두 개, 루프 두 개, 혹은 조건문 한 개와 루프 한 개를 사용 가능