# 72. 정렬된 시퀀스를 검색할 때는 bisect 를 사용하라

## 1. 정렬된 리스트로 존재하는 큰 데이터 검색

- 리스트에서 `index` 함수를 사용해 특정 값을 찾아내려면 리스트 길이에 선형으로 비례하는 시간 필요

```python
data = list(range(10**5))
index = data.index(91234)
assert index == 91234
```

- 찾는 값이 리스트 안에 들어 있는지 모르는 경우, 원하는 값과 같거나 그보다 큰 값의 인덱스 중 가장 작은 인덱스를 찾고 싶을 것
- 리스트를 앞에서부터 선형으로 읽으면서 각 원소를 찾는 값과 비교

```python
def find_closest(sequence, goal):
    for index, value in enumerate(sequence):
        if goal < value:
            return index
    raise ValueError(f'범위를 벗어남: {goal}')
    
index = find_closest(data, 91234.56)
assert index == 91235
```

## 2. 내장 `bisect` 모듈

- 이진 검색 모듈
- 순서가 정해져 있는 리스트에 대한 검색을 더 효과적으로 수행
- 리스트 타입 뿐만 아니라 시퀀스처럼 작동([Better way 43](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay43.md))하는 모든 파이선 객체에 대해 적용 가능
- `bisect_left` 를 사용하면 정렬된 원소 시퀀스에 대해 이진 검색 가능
- 로그 복잡도

```python
from bisect import bisect_left

index = bisect_left(data, 91234)    # 정확히 일치
assert index == 91234

index = bisect_left(data, 91234.56) # 근접한 값과 일치
assert index == 91235

index = bisect_left(data, 91234.23) # 근접한 값과 일치(찾는 값 이상의 값 중에서 근접한 값을 찾음)
assert index == 91235
```

- 벤치마크 결과

```python
선형 검색: 3.892842초
이진 검색: 0.004266초
선형 검색이 912.4배 더 걸림
```