# 13. 슬라이싱보다는 나머지를 모두 잡아내는 언패킹을 사용하라

## 1. 슬라이싱+인덱싱으로 인한 혼란

- 다음 코드는 인덱스와 슬라이스가 섞여 있어 가독성이 떨어짐

```python
oldest = car_ages_descending[0]
second_oldest = car_ages_descending[1]
others = car_ages_descending[2:]
print(oldest, second_oldest, others)

>>>
20 19 [15, 9, 8, 7, 6, 4, 1, 0]
```

- 이런 경우 언패킹 방식과 star 사용으로 가독성을 높일 수 있음

```python
oldest, second_oldest, *others = car_ages_descending
print(oldest, second_oldest, others)

>>>
20 19 [15, 9, 8, 7, 6, 4, 1, 0]
```

## 2. star 표현을 사용한 언패킹

- star 표현을 사용한 언패킹을 사용하는 경우 적어도 필수 대입 변수가 1개 이상 필요함
- 한 수준 언패킹 패턴에 star 표현을 2개 이상 불가

```python
*others = car_ages_descending

>>>
Traceback ...
SyntaxError: starred assignment target must be in a list or tuple
```

- 아래 예제와 같이 여러 계층 구조에서 서로 다른 부분에서는 star 표현을 여러개 사용해도 괜찮음

```python
car_inventory = {
    '시내': ('그랜저', '아반떼', '티코'),
    '공항': ('제네시스 쿠페', '소나타', 'K5', '엑센트'),
}
((loc1, (best1, *rest1)),
 (loc2, (best2, *rest2))) = car_inventory.items()
print(f'{loc1} 최고는 {best1}, 나머지는 {len(rest1)} 종')
print(f'{loc2} 최고는 {best2}, 나머지는 {len(rest2)} 종')

>>>
시내 최고는 그랜저, 나머지는 2 종
공항 최고는 제네시스 쿠페, 나머지는 3 종
```

- 또한 star 표현 타입은 list 이므로 star 표현 변수에 실제 할당되는 원소가 없다면 빈 리스트가 됨

- 제너레이터(yield 를 사용한 함수) 의 결과를 인덱스와 슬라이스를 사용하여 처리한 코드

```python
def generate_csv():
    yield ('날짜', '제조사' , '모델', '연식', '가격')
    ...

all_csv_rows = list(generate_csv())
header = all_csv_rows[0]
rows = all_csv_rows[1:]
print('CSV 헤더:', header)
print('행 수:', len(rows))

>>>
CSV 헤더: ('날짜', '제조사', '모델', '연식', '가격')
행 수: 200
```

- 해당 예제는 인덱스와 슬라이스를 사용하여 가독성이 떨어짐

- 언패킹과 이터레이터를 사용한 코드

```python
it = generate_csv()
header, *rows = it
print('CSV 헤더:', header)
print('행 수:', len(rows))

>>>
CSV 헤더: ('날짜', '제조사', '모델', '연식', '가격')
행 수: 200
```

- 해당 코드는 첫번째(`header`) 와 그 외 나머지(`row`) 로 처리하는 것이 명확하게 보이므로 가독성을 높일 수 있음
- star 표현은 항상 리스트 타입을 받으므로 iterator 를 언패킹하면 메모리를 한번에 과다하게 사용할 수 있으므로 조심해서 써야 할 필요 있음