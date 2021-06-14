# 23. 키워드 인자로 선택적인 기능을 제공하라

## 1. 키워드 인자

- 이름이 정해진 인자
- 필요한 위치 기반 인자가 모두 제공되는 한, 키워드 인자를 넘기는 순서는 관계없음
- 위치 기반 인자는 키워드 인자보다 앞에 지정해야 함
- 각 인자는 단 한 번만 지정

```python
def remainder(number, divisor):
    return number % divisor
    
assert remainder(20, 7) == 6

remainder(20, 7)
remainder(20, divisor=7)
remainder(number=20, divisor=7)
remainder(divisor=7, number=20)
```

- 딕셔너리를 사용하여 `remainder` 함수를 만드는 경우
    - `**` 사용

```python
my_kwargs = {
    'number': 20,
    'divisor': 7,
}
assert remainder(**my_kwargs) == 6

----------------------------------------------------

my_kwargs = {
    'number': 20,
}
other_kwargs = {
    'divisor': 7,
}
assert remainder(**my_kwargs, **other_kwargs) == 6
```

- 아무 키워드 인자를 받는 함수 생성의 경우, 모든 키워드 인자를 `dict`에 모아주는 `**kwargs`  파라미터 사용(26장 데코레이터 참고)

```python
def print_parameters(**kwargs):
    for key, value in kwargs.items():
        print(f'{key} = {value}')
        
print_parameters(alpha=1.5, beta=9, 감마=4)  # 한글 파라미터 이름도 잘 작동함

>>>
alpha = 1.5
beta = 9
감마 = 4
```

## 2. 키워드 인자 사용 이점

- 코드를 처음 보는 사람들에게 함수 호출의 의미를 명확히 알려줄 수 있음
- 함수 정의에서 디폴트 값 지정 가능
- 기존 호출자에게 하위 호환성 제공하면서 함수 파라미터 확장 가능
    - 선택적인 인자를 위치 인자로 지정하면 혼동 야기
    - 키워드 인자 사용, 위치 인자 사용 X

```python
def flow_rate(weight_diff, time_diff, period=1):
    return (weight_diff / time_diff) * period

flow_per_second = flow_rate(weight_diff, time_diff)
flow_per_hour = flow_rate(weight_diff, time_diff, period=3600)
```

```python
def flow_rate(weight_diff, time_diff,
              period=1, units_per_kg=1):
    return ((weight_diff * units_per_kg) / time_diff) * period

pounds_per_hour = flow_rate(weight_diff, time_diff,
                            period=3600, units_per_kg=2.2)
```