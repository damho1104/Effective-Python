# 33. yield from 을 사용해 여러 제너레이터를 합성하라

## 1. 여러 제너레이터를 사용할 시 단점

- 제너레이터를 사용해 화면의 이미지를 움직이게 하는 그래픽 프로그램의 일부 코드라 가정
- 화면상 이동 변위를 만들어낼 때 사용할 2가지 제너레이터를 정의한 코드
- 최종 애니메이션을 만드려면 move 와 pause 를 합성하여 변위 시퀀스 하나만 만들어야 함

```python
def move(period, speed):
    for _ in range(period):
        yield speed

def pause(delay):
    for _ in range(delay):
        yield 0

def animate():
    for delta in move(4, 5.0):
        yield delta
    for delta in pause(3):
        yield delta
    for delta in move(2, 3.0):
        yield delta

def render(delta):
    print(f'Delta: {delta:.1f}')
    # 화면에서 이미지를 이동시킨다
    ...
def run(func):
    for delta in func():
        render(delta)

run(animate)

>>>
Delta: 5.0
Delta: 5.0
Delta: 5.0
Delta: 5.0
Delta: 0.0
Delta: 0.0
Delta: 0.0
Delta: 3.0
Delta: 3.0
```

- `for`, `yield` 식이 반복되면서 가독성이 떨어짐

## 2. `yield from` 식 사용한 제너레이터 합성

- 제어를 부모 제너레이터에게 전달하기 전 내포된 제너레이터가 모든 값을 내보냄
- 이전 코드와 동일한 행동 수행, 가독성 증가
- 인터프리터에서 대신 `for` 루프를 내포시켜 처리하도록 함(이로 인해 최적화 됨)

```python
def animate_composed():
    yield from move(4, 5.0)
    yield from pause(3)
    yield from move(2, 3.0)
    
run(animate_composed)

>>>
Delta: 5.0
Delta: 5.0
Delta: 5.0
Delta: 5.0
Delta: 0.0
Delta: 0.0
Delta: 0.0
Delta: 3.0
Delta: 3.0
```

- 벤치마크 결과 코드

```python
import timeit

def child():
    for i in range(1_000_000):
        yield i

def slow():
    for i in child():
        yield i

def fast():
    yield from child()

baseline = timeit.timeit(
    stmt='for _ in slow(): pass',
    globals=globals(),
    number=50)
print(f'수동 내포: {baseline:.2f}s')
comparison = timeit.timeit(
    stmt='for _ in fast(): pass',
    globals=globals(),
    number=50)
print(f'합성 사용: {comparison:.2f}s')

reduction = -(comparison - baseline) / baseline
print(f'{reduction:.1%} 시간이 적게 듦')

>>>
수동 내포: 2.81s
합성 사용: 2.56s
8.8% 시간이 적게 듦
```