# 29. 대입식을 사용해 컴프리헨션 안에서 반복 작업을 피해라

## 1. 컴프리헨션 사용 단점

```python
stock = {
    '못': 125,
    '나사못': 35,
    '나비너트': 8,
    '와셔': 24,
}

order = ['나사못', '나비너트', '클립']

def get_batches(count, size):
    return count // size

result = {}
for name in order:
    count = stock.get(name, 0)
    batches = get_batches(count, 8)
    if batches:
        result[name] = batches

print(result)

>>>
{'나사못': 4, '나비너트': 1}
```

- 딕셔너리 컴프리헨션 사용 예제

```python
found = {name: get_batches(stock.get(name, 0), 8)
         for name in order
         if get_batches(stock.get(name, 0), 8)}
print(found)

>>>
{'나사못': 4, '나비너트': 1}
```

- 컴프리헨션으로 변경했을 때 **단점**
    - `get_batches` 함수 반복으로 인해 가독성 떨어짐
    - 두 식을 항상 똑같이 수정해야 하는데 그렇지 못할 실수를 할 가능성이 존재

## 2. 왈러스 연산자를 사용한 컴프리헨션

```python
found = {name: batches for name in order
         if (batches := get_batches(stock.get(name, 0), 8))}
```

- 왈러스(:=) 연산자를 사용하여 `get_batches` 함수 단일 호출 가능
- 왈러스 연산자를 사용한 대입식을 컴프리헨션 값 쪽에 사용 가능
    - 그러나 컴프리헨션 평가 순서 고려 필요함

```python
result = {name: (tenth := count // 10)
          for name, count in stock.items() if tenth > 0}

>>>
Traceback ...
NameError: name 'tenth' is not defined
```

- 해당 예제는 컴프리헨션 값 쪽에 왈러스 연산자 사용
- 그러나 `tenth` 는 `if` 문 이후에 assign 되므로 정의되지 않았다고 에러 발생함

```python
result = {name: tenth for name, count in stock.items()
          if (tenth := count // 10) > 0}
print(result)

>>>
{'못': 12, '나사못': 3, '와셔': 2}

```

- 컴프리헨션 값 부분에 왈러스 연산자 사용하는 경우 루프 변수가 외부로 유출됨

```python
half = [(last := count // 2) for count in stock.values()]
print(f'{half}의 마지막 원소는 {last}') # 여기서 last 변수 사용 가능

>>>
[62, 17, 4, 12]의 마지막 원소는 12
```

- 컴프리헨션의 루프 변수는 외부로 유출되지 않음

```python
half = [count // 2 for count in stock.values()]
print(half)   # 작동함
print(count)  # 루프 변수가 누출되지 않기 때문에 예외가 발생함

>>>
[62, 17, 4, 12]
Traceback ...
NameError: name 'count' is not defined
```

## 3. 정리

- 왈러스 연산자를 사용하여 컴프리헨션을 사용하면 가독성이 좋아지고 중복 코드를 없앨 수 있음
- 조건 식 뿐만 아니라 값에 해당하는 위치에서도 왈러스 연산자를 사용한 대입식 사용 가능하나 루프 변수 외부 유출을 피하도록 하기 위해 최대한 조건 식에서만 사용할 것