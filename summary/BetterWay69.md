# 69. 정확도가 매우 중요한 경우에는 decimal 을 사용하라

## 1. 부동소수점

- 2배 정밀도 부동소수점 타입은 IEEE 754 표준을 따름
- 파이선은 허수 값을 포함하는 표준 복소수 타입 제공

- 국제 전화 고객에게 부과할 통화료를 계산 가정
    - 고객이 통화한 시간을 분, 초 단위로 알고 있으며 미국과 남극 사이의 도수(분)당 통화료가 1.45달러/분 이라 가정
    
    ```python
    rate = 1.45
    seconds = 3*60 + 42
    cost = rate * seconds / 60
    print(cost)
    
    >>>
    5.364999999999999
    ```
    
    ```python
    print(round(cost, 2))
    
    >>>
    5.36
    ```
    
    - IEEE 754 부동소수점 수의 내부(이진) 표현법으로 인해 결과는 5.365보다 0.00000000000001 만큼 작음
    - 부동소수점 수의 오류로 인해 이 값을 가장 가까운 센트 단위로 반올림하면 최종 요금이 늘어나지 않고 줄어들게 됨
    

## 2. `decimal` 내장 모듈

- `decimal` 내장 모듈에 있는 `Decimal` 클래스 사용
- `Decimal` 클래스는 기본값으로 소수점 이하 28번째 자리까지 고정소수점 수 연산 제공
    - 자리수 늘리기 가능

```python
from decimal import Decimal

rate = Decimal('1.45')
seconds = Decimal(3*60 + 42)
cost = rate * seconds / Decimal(60)
print(cost)

>>>
5.365
```

- `Decimal` 클래스를 사용하여 근사치가 아닌 정확한 값 표현 가능
- `Decimal` 인스턴스에 값을 지정하는 방법
    - 숫자가 들어있는 str 타입 문자열을 `Decimal` 생성자에 전달
        - 파이선 부동소수점 수의 근본적인 특성으로 인해 발생하는 정밀도 손실을 막을 수 있음
    - int나 float 타입 인스턴스를 `Decimal` 생성자에 전달
    - 정수를 넘기는 경우 문제 안됨
    
    - 정확한 답을 요구하는 경우 문자열로 전달하는 것이 중요
    
    ```python
    print(Decimal('1.45'))
    print(Decimal(1.45))
    
    >>>
    1.45
    1.4499999999999999555910790149937383830547332763671875
    ```
    

- 연결 비용이 훨씬 저렴한 지역 사이의 아주 짧은 통화도 지원하고 싳은 경우
    - 통화 시간 5초, 통화료 0.05원
- 계산한 값이 너무 작아 0 이 나옴

```python
rate = Decimal('0.05')
seconds = Decimal('5')
small_cost = rate * seconds / Decimal(60)
print(small_cost)

>>>
0.004166666666666666666666666667
```

```python
print(round(small_cost, 2))

>>>
0.00
```

- `Decimal` 클래스의 원하는 소수점 이하 자리까지 원하는 방식으로 근삿값 계산 내장 함수 사용
    - 반올림 방식 사용

```python
from decimal import ROUND_UP
rounded = cost.quantize(Decimal('0.01'), rounding=ROUND_UP)
print(f'반올림 전: {cost} 반올림 후: {rounded}')

>>>
반올림 전: 5.365 반올림 후: 5.37
```

```python
rounded = small_cost.quantize(Decimal('0.01'),
                              rounding=ROUND_UP)
print(f'반올림 전: {small_cost} 반올림 후: {rounded}')

>>>
반올림 전: 0.004166666666666666666666666667 반올림 후: 0.01
```

## 3. 정리

- `Decimal` 은 고정소수점 표현에 대해서 잘 작동함
- 여전히 정밀도에 한계는 존재
- 정밀도 제한 없이 유리수를 사용하고 싶은 경우 `fractions` 내장 모듈의 `Fraction` 클래스 사용할 것