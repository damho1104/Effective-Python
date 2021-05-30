# 17. 내부 상태에서 원소가 없는 경우를 처리할 때는 setdefault보다 defaultdict를 사용하라

## 1. `setdefault`

- 키가 없는 딕셔너리의 값을 처리할 때 setdefault 로 값을 줄 수 있음
- `setdefault` 를 사용하는 것이 `get` 메소드와 대입식(`:=`) 사용보다 더 간결

```python
visits.setdefault('프랑스', set()).add('칸')    # 짧다

if (japan := visits.get('일본')) is None:       # 길다
    visits['일본'] = japan = set()
japan.add('교토')
print(visits)

>>>
{'미국': {'로스엔젤레스', '뉴욕'}, '일본': {'하코네', '교토'}, '프랑스': {'칸'}}
```

- 아래 코드는 `add` helper 함수를 구현하여 키가 없는 경우 새로운 값을 추가할 수 있도록 `setdefault` 를 사용하였음

```python
class Visits:
    def __init__(self):
        self.data = {}
        
    def add(self, country, city):
        city_set = self.data.setdefault(country, set())
        city_set.add(city)

visits = Visits()
visits.add('러시아', '예카테린부르크')
visits.add('탄자니아', '잔지바르')
print(visits.data)

>>>
{'러시아': {'예카테린부르크'}, '탄자니아': {'잔지바르'}}
```

- `setdefault` 는 이름이 여전히 헷갈릴 수 있기 때문에 코드 동작을 바로 이해하기 어려움
- 해당 코드는 주어진 키가 있던 없던 간에 무조건 set 인스턴스를 생성하므로 효율적이지 않음

## 2. `defaultdict`

- `collections` 내장 모듈에 존재
- 키가 없을 때 자동으로 기본 값을 저장하여 간단히 처리할 수 있도록 함

```python
from collections import defaultdict

class Visits:
    def __init__(self):
        self.data = defaultdict(set)

    def add(self, country, city):
        self.data[country].add(city)

visits = Visits()
visits.add('영국', '바스')
```

- 이름도 명확
- `add` 메소드 호출 시 의미없는 인스턴스가 생성되지 않으므로 앞선 코드보다 효율적

## 결론

- 기본적으로 `setdefault` 보단 `defaultdict` 를 사용하는 것이 나은지 고려가 필요함
- 물론 `setdefault` 를 꼭 사용해야 할 수도 있으므로 신중히 판단해야 함