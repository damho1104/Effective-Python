# 34. send 로 제너레이터에 데이터를 주입하지 말라

## 1. `yield` 표현식

- 제너레이터 함수가 이터레이션이 가능한 출력 값 생성 가능
- **단점:** 생성된 데이터 채널은 단방향
- 아래 예제는 `wave` 제너레이터를 이터레이션하면서 진폭이 고정된 파형 신호 송신 가능
    - 별도의 입력을 사용해 진폭을 변경하려는 경우는 어떻게???

```python
import math

def wave(amplitude, steps):
    step_size = 2 * math.pi / steps   # 2라디안/단계 수
    for step in range(steps):
        radians = step * step_size
        fraction = math.sin(radians)
        output = amplitude * fraction
        yield output

def transmit(output):
    if output is None:
        print(f'출력: None')
    else:
        print(f'출력: {output:>5.1f}')

def run(it):
    for output in it:
        transmit(output)

run(wave(3.0, 8))

>>>
출력:   0.0
출력:   2.1
출력:   3.0
출력:   2.1
출력:   0.0
출력:  -2.1
출력:  -3.0
출력:  -2.1
```

## 2. `send` 메소드

- `yield` 식을 양방향 채널로 격상시켜줌
- 입력을 제너레이터에 스트리밍하는 동시에 출력을 내보낼 수 있음
- `yield` 식 기본 반환값은 `None`

```python
def my_generator():
    received = yield 1
    print(f'받은 값 = {received}')

it = iter(my_generator())
output = next(it)            # 첫 번째 제너레이터 출력을 얻는다
print(f'출력값 = {output}')

try:
    next(it)                 # 종료될 때까지 제너레이터를 실행한다
except StopIteration:
    pass

>>>
출력값 = 1
받은 값 = None
```

- 시작 제너레이터는 `yield` 식에 도달하지 못했기 때문에 최초로 `send` 를 호출할 때 인자로 전달할 수 있는 값은 `None`
    - 시작 시점에 다른 값 전달하면 `TypeError` 발생

```python
def my_generator():
    received = yield 1
    print(f'받은 값 = {received}')

it = iter(my_generator())
output = it.send(None)     # 첫 번째 제너레이터 출력을 얻는다
print(f'출력값 = {output}')

try:
    it.send('안녕!')        # 값을 제너레이터에 넣는다
except StopIteration:
    pass

>>>
출력값 = 1
받은 값 = 안녕!
```

- `send` 메소드를 적용한 진폭 예제

```python
def wave_modulating(steps):
    step_size = 2 * math.pi / steps
    amplitude = yield                # 초기 진폭을 받는다
    for step in range(steps):
        radians = step * step_size
        fraction = math.sin(radians)
        output = amplitude * fraction
        amplitude = yield output     # 다음 진폭을 받는다

def run_modulating(it):
    amplitudes = [
        None, 7, 7, 7, 2, 2, 2, 2, 10, 10, 10, 10, 10]
    for amplitude in amplitudes:
        output = it.send(amplitude)
        transmit(output)

def transmit(output):
    if output is None:
        print(f'출력: None')
    else:
        print(f'출력: {output:>5.1f}')
        
run_modulating(wave_modulating(12))

>>>
출력: None
출력:   0.0
출력:   3.5
출력:   6.1
출력:   2.0
출력:   1.7
출력:   1.0
출력:   0.0
출력:  -5.0
출력:  -8.7
출력: -10.0
출력:  -8.7
출력:  -5.0
```

- 코드의 가독성 떨어짐
    - `send` 와 `yield` 연결을 알아보기 쉽지 않음

## 3. `yield from` 을 사용한 방법

- `yield from` 으로 변경한 예제

```python
def wave_modulating(steps):
    step_size = 2 * math.pi / steps
    amplitude = yield                # 초기 진폭을 받는다
    for step in range(steps):
        radians = step * step_size
        fraction = math.sin(radians)
        output = amplitude * fraction
        amplitude = yield output     # 다음 진폭을 받는다

def complx_wave_modulating():
    yield from wave_modulating(3)
    yield from wave_modulating(4)
    yield from wave_modulating(5)

def run_modulating(it):
    amplitudes = [
        None, 7, 7, 7, 2, 2, 2, 2, 10, 10, 10, 10, 10]
    for amplitude in amplitudes:
        output = it.send(amplitude)
        transmit(output)

def transmit(output):
    if output is None:
        print(f'출력: None')
    else:
        print(f'출력: {output:>5.1f}')

run_modulating(complex_wave_modulating())

>>>
출력: None
출력:   0.0
출력:   6.1
출력:  -6.1
출력: None
출력:   0.0
출력:   2.0
출력:   0.0
출력: -10.0
출력: None
출력:   0.0
출력:   9.5
출력:   5.9
```

- 출력이 `None` 인 경우 발생
    - 처음 `wave_modulating` 함수 실행 시 아무 값도 생성하지 않은 `yield` 로 시작
    - 이로 인해 부모 제너레이터가 자식 제너레이터로 실행 흐름이 이동할 때마다 `None` 이 출력됨
- 방안
    - `run_modulating` 함수의 복잡도를 증가시켜 `None` 문제 우회시키기
        - `run_modulating` 함수에서 `None` 일때의 처리를 추가하는 것
        - 비추
    - `wave_cascading` 함수에 이터레이터 전달
        - **단점 :** thread-safe 하지 않음, **제너레이터는 항상 thread-safe 하지 않다는 점 주의!!**
            - 이땐 `async` 함수로 비동기 처리해야 할듯!

        ```python
        def wave_cascading(amplitude_it, steps):
            step_size = 2 * math.pi / steps
            for step in range(steps):
                radians = step * step_size
                fraction = math.sin(radians)
                amplitude = next(amplitude_it)  # 다음 입력 받기
                output = amplitude * fraction
                yield output

        def complex_wave_cascading(amplitude_it):
            yield from wave_cascading(amplitude_it, 3)
            yield from wave_cascading(amplitude_it, 4)
            yield from wave_cascading(amplitude_it, 5)

        def transmit(output):
            if output is None:
                print(f'출력: None')
            else:
                print(f'출력: {output:>5.1f}')

        def run_cascading():
            amplitudes = [7, 7, 7, 2, 2, 2, 2, 10, 10, 10, 10, 10]
            it = complex_wave_cascading(iter(amplitudes))
            for amplitude in amplitudes:
                output = next(it)
                transmit(output)
                
        run_cascading()

        >>>
        출력:   0.0
        출력:   6.1
        출력:  -6.1
        출력:   0.0
        출력:   2.0
        출력:   0.0
        출력:  -2.0
        출력:   0.0
        출력:   9.5
        출력:   5.9
        출력:  -5.9
        출력:  -9.5
        ```

## 4. 정리

- 제너레이터의 입력과 출력을 동시에 하는 방법 존재
- 그러나 send 메소드를 사용하면 코드 가독성이 떨어짐, send 메소드는 가급적 사용하지 말 것
- yield from 으로 우회할 수 있으나 None 처리의 어려움 존재, 우회시켜야 함