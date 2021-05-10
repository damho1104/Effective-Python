# 5. 복잡한 식을 쓰는 대신 도우미 함수를 작성하라

## 1. 도우미 함수를 사용할 것

```python
# 질의 문자열이 '빨강=5&파랑=0&초록='인 경우
red = my_values.get('빨강', [''])[0] or 0
green = my_values.get('초록', [''])[0] or 0
opacity = my_values.get('투명도', [''])[0] or 0
print(f'빨강: {red!r}')
print(f'초록: {green!r}')
print(f'투명도: {opacity!r}')

>>>
빨강: '5'
초록: 0
투명도: 0
```

- 파라미터가 없거나 비어있을 경우 조건문을 통해 검증 가능
- 그러나 위 예제 처럼 조건문이 직관적으로 보이지 않음

```python
red = int(my_values.get('빨강', [''])[0] or 0)
```

- 위 예제처럼 모든 파라미터를 정수로 변환해서 즉시 수식에 활용할 수 있음
- 그러나 이 예제도 직관적이지 않음

```python
red_str = my_values.get('빨강', [''])
red = int(red_str[0]) if red_str[0] else 0
```

- 조건문이 아닌 조건식을 통해 한 줄로 표현
- 조건문보다는 덜 명확

```python
green_str = my_values.get('초록', [''])
if green_str[0]:
    green = int(green_str[0])
else:
    green = 0
```

- 훨씬 명확해짐

```python
def get_first_int(values, key, default=0):
    found = values.get(key, [''])
    if found[0]:
        return int(found[0])
    return default

green = get_first_int(my_values, '초록')
```

- 이 로직을 하나의 함수(도우미 함수)로 정의하면 중복 제거 가능하고 훨씬 직관적임

## 2. 정리

- 코드의 간결성보다는 가독성을 좋게 하는 훈련을 할 수 있도록 할 것!