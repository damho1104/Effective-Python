# 7. range 보다는 enumerate를 사용하라

## 1. range 내장 함수

- 정수 집합을 순회하는 루프가 필요할 때 사용
- 특정 정수 범위를 지정할 때 사용

```python
from random import randint

random_bits = 0
for i in range(32):
    if randint(0, 1):
        random_bits |= 1 << i

print(bin(random_bits))

>>>
0b11101000100100000111000010000001
```

- 리스트를 순회할 때 인덱스를 사용하고 싶은 경우에도 range 사용 가능

```python
flavor_list = ['바닐라', '초콜릿', '피칸', '딸기']
for i in range(len(flavor_list)):
    flavor = flavor_list[i]
    print(f'{i + 1}: {flavor}')

>>>
1: 바닐라
2: 초콜릿
3: 피칸
4: 딸기
```

- **단점**
    - list 의 길이 파악 필요
    - 인덱스 사용하여 접근 필요함
    - 단계가 여러개로 구성되므로 가독성 떨어짐

## 2. enumerate 내장 함수

- 이터레이터를 반환하며 next 내장 함수나 루프에 사용하여 인덱스, 값 튜플을 반환받을 수 있음

```python
it = enumerate(flavor_list)
print(next(it))
print(next(it))

>>>
(0, '바닐라')
(1, '초콜릿')
```

```python
for i, flavor in enumerate(flavor_list):
    print(f'{i + 1}: {flavor}')

>>>
1: 바닐라
2: 초콜릿
3: 피칸
4: 딸기
```

- enumerate 두번째 인자는 시작 숫자를 정의함(기본값: 0)

```python
# i 는 1부터 시작함
for i, flavor in enumerate(flavor_list, 1):
    print(f'{i}: {flavor}')
```