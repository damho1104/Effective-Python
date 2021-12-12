# 64. 진정한 병렬성을 살리려면 concurrent.futures 를 사용하라

## 1. 성능 개선

- 연산을 병렬로 적용
- 그러나 전역 인터프리터 락(GIL) 로 인해 파이선 스레드는 진정한 병렬 실행 불가
- 성능에 가장 결정적인 영향을 미치는 부분을 C 언어를 사용한 확장 모듈로 작성
    - C 를 사용하면 로우 레벨에 가깝게 적용 가능하여 파이선에 비해 빠름
    - 파이선의 GIL과도 상관없이 CPU 코어 활용 가능
    - SWIG([https://github.com/swig/swig](https://github.com/swig/swig)), CLIF([https://github.com/google/clif](https://github.com/google/clif))
- C로 코드 재작성하는 것의 단점
    - 파이선에서 짧고 이해하기 쉬운 코드가 C 에서는 장황하고 복잡할 수 있음
    - 포팅 시 버그 여부 확인 필요
    - 코드의 한 부분만 C 로 바꾸면 되는 경우가 드물다는 점
    - 거의 대부분의 코드를 포팅해야 함
    - 그로 인해 테스트의 필요성 증대, 코드의 위험성 증대
- 파이선에서 C로 편하게 전환시켜주는 오픈 소스 도구
    - CPtyhon ([https://cython.org/](https://cython.org/))
    - Numba ([https://numba.pydata.org/](https://numba.pydata.org/))

## 2. `concurrent.futures` 내장 모듈

- `multiprocessing` 내장 모듈 사용
- 자식 프로세스를 다른 파이선 인터프리터를 실행하여 파이선에서 여러 CPU 코어를 활용할 수 있음
- 자식 프로세스는 주 인터프리터와 별도로 실행되므로 GIL 분리됨

```python
# mymodule.py
def gcd(pair):
    a, b = pair
    low = min(a, b)
    for i in range(low, 0, -1):
        if a % i == 0 and b % i == 0:
            return i
    assert False, '도달할 수 없음
```

```python
# run_serial.py
import my_module
import time

NUMBERS = [
    (1963309, 2265973), (2030677, 3814172),
    (1551645, 2229620), (2039045, 2020802),
    (1823712, 1924928), (2293129, 1020491),
    (1281238, 2273782), (3823812, 4237281),
    (3812741, 4729139), (1292391, 2123811),
]

def main():
    start = time.time()
    results = list(map(my_module.gcd, NUMBERS))
    end = time.time()
    delta = end - start
    print(f'총 {delta:.3f} 초 걸림')

if __name__ == '__main__':
    main() 

>>>
총 0.911 초 걸림
```

- 위 예제는 두 수의 최대공약수를 구하는 구현
- 순차 실행 시 연산 시간 선형적으로 증가
- GIL로 인해 파이썬이 여러 스레드를 다중 CPU 코어에서 병렬 실행할 수 없으므로 스레드를 활용해도 속도 향상 거의 없음

```python
# run_threads.py
import my_module
from concurrent.futures import ThreadPoolExecutor
import time

NUMBERS = [
   ...
]

def main():
    start = time.time()
    pool = ThreadPoolExecutor(max_workers=2)
    results = list(pool.map(my_module.gcd, NUMBERS))
    end = time.time()
    delta = end - start
    print(f'총 {delta:.3f} 초 걸림')

if __name__ == '__main__':
    main()

>>>
총 1.436 초 걸림
```

- 위 예제는 `concurrent.futures` 모듈에 있는 `ThreadPoolExecutor` 클래스를 활용하여 두 개의 작업자 스레드로 수행
- 스레드 풀 시작, 풀과 통신하는 로직으로 인해 실행 시간이 좀 더 증가

```python
# run_parallel.py
import my_module
from concurrent.futures import ProcessPoolExecutor
import time

NUMBERS = [
...
]

def main():
    start = time.time()
    pool = ProcessPoolExecutor(max_workers=2)     # 이 부분만 바꿈
    results = list(pool.map(my_module.gcd, NUMBERS))
    end = time.time()
    delta = end - start
    print(f'총 {delta:.3f} 초 걸림')

if __name__ == '__main__':
    main()

>>>
총 0.683 초 걸림
```

- `concurrent.futures` 에 있는 `ProcessPoolExecutor` 사용
- `ProcessPoolExecutor` 작업
    - **[parent]** 이 객체는 입력 데이터로 들어온 `map` 메소드에 전달된 `NUMBERS`의 각 원소를 취함
    - **[parent]** 이 객체는 전달되어 얻은 원소를 `pickle` 모듈을 사용하여 이진 데이터로 직렬화
    - **[parent, child]** 이 객체는 로컬 소켓을 통해 주 인터프리터 프로세스로부터 자식 인터프리터 프로세스에게 직렬화한 데이터를 복사
    - **[child]** 이 객체는 다시 역직렬화
    - **[child]** 이 객체는 `gcd` 함수가 들어 있는 모듈 import
    - **[child]** 이 객체는 입력 데이터에 대해 `gcd` 함수를 실행, 다른 자식 인터프리터 프로세스와 병렬로 실행
    - **[child]** 이 객체는 `gcd` 함수의 결과를 이진 데이터로 직렬화
    - **[parent, child]** 이 객체는 로컬 소켓을 통해 자식 인터프리터 프로세스로부터 부모 인터프리터 프로세스에게 직렬화한 결과 데이터 전달
    - **[parent]** 이 객체는 다시 역직렬화
    - **[parent]** 여러 자식 프로세스가 반환한 결과를 병합하여 list 로 생성
- 부모와 자식 프로세스 사이 데이터를 주고받을 때 마다 직렬화 및 역직렬화 발생, 추가비용이 큼

- 이 방식은 코드 간 의존성이 없고 레버리지가 큰 유형의 작업에 적합
    - 레버리지 : 주고받아야 하는 데이터의 크기는 작으나 자식 프로세스가 연산해야 할 양은 상당히 큰 것

## 3. 정리

- 처음에는 `multiprocessing` 모듈을 사용하지 않고 순차적으로 코드를 작성
- `ThreadPoolExecutor` 를 통해 확인
- 그래도 성능 문제가 있으면 `ProcessPoolExecutor` 를 사용할 것을 권장