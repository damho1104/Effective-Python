# 8. 여러 이터레이터에 대해 나란히 루프를 수행하려면 zip을 사용해라

## 1. zip 내장함수

- `zip` 은 lazy generator 로 2개 이상의 이터레이터를 묶을 수 있음

```python
# names 리스트와 counts 리스트를 동시 순회
names = ['Cecilia', '남궁민수', '毛泽东']
counts = [len(n) for n in names]
for i, name in enumerate(names):
    count = counts[i]
    if count > max_count:
        longest_name = name
        max_count = count
```

```python
# zip 내장함수로 두 리스트를 감싸면 각 요소의 tuple 을 반환함
for name, count in zip(names, counts):
    if count > max_count:
        longest_name = name
        max_count = count
```

- 두 리스트의 원소 개수가 다른 경우는?

```python
names = ['Cecilia', '남궁민수', '毛泽东']
counts = [len(n) for n in names]
names.append('Rosalind')
for name, count in zip(names, counts):
    print(name)
```

- 출력은 다음과 같다.

    ```python
    >>>
    Cecilia
    남궁민수
    毛泽东
    ```

    - zip 내장 함수는 원소의 개수가 작은 리스트를 기준으로 각 원소를 tuple 로 묶는다.

## 2. zip_longest

- itertools 모듈에 존재
- `zip` 내장 함수와는 달리 가장 긴 리스트를 기준으로 각 원소를 tuple 로 묶어 반환
- 값이 없는 리스트로부터 반환되는 건 `fillvalue` 에 넣은 값을 반환, 기본값: `None`

```python
import itertools

for name, count in itertools.zip_longest(names, counts):
    print(f'{name}: {count}')

>>>
Cecilia: 7
남궁민수: 4
毛泽东: 3
Rosalind: None
```