# 54. 스레드에서 데이터 경합을 피하기 위해 Lock 을 사용하라

## 1. GIL 의 오해

- '전역 인터프리터 락(GIL) 으로 인해 더 이상 상호 배제 락을 사용하지 않아도 된다' 는 표현은 잘못되었음
    - 파이선 스레드는 한 번에 하나만 실행될 수 있음
    - 인터프리터에서 어떤 스레드가 데이터 구조에 대해 수행하는 연산은 연속된 두 바이트코드 사이에 언제든 인터럽트될 수 있음
    - **결론**: 스레드가 같은 데이터 구조에 동시에 접근하면 위험
    
- 병렬적으로 여러 가지 개수를 세는 프로그램 예제
    
    ```python
    class Counter:
        def __init__(self):
            self.count = 0
            
        def increment(self, offset):
            self.count += offset
    
    def worker(sensor_index, how_many, counter):
        for _ in range(how_many):
            # 센서를 읽는다
            ....
            counter.increment(1) 
    
    from threading import Thread
    
    how_many = 10**5
    counter = Counter()
    
    threads = []
    for i in range(5):
        thread = Thread(target=worker, args=(i, how_many, counter))
        threads.append(thread)
        thread.start()
    
    for thread in threads:
        thread.join()
    
    expected = how_many * 5
    found = counter.count
    print(f'카운터 값은 {expected}여야 하는데, 실제로는 {found} 입니다')
    
    >>>
    카운터 값은 500000여야 하는데, 실제로는 481384 입니다
    ```
    
    - 센서를 읽을 때 블로킹 I/O 수행, 센서마다 작업자 스레드 할당
    - 물론 해당 실행 결과는 바뀔 수 있음(운 좋으면 500000)
    
- 파이선 인터프리터는 실행되는 모든 스레드를 강제로 공평하게 취급하여 각 스레드의 실행 시간을 비슷하게 동작하도록 설정
- 실행 중인 스레드를 일시 중단시키고 다른 스레드를 실행시키는 작업 반복
- 스레드 중단 시점을 알 수 없음
- atomic 인 것처럼 보이는 연산 수행 도중에도 스레드 일시 중단 가능

## 2. threading 내장 모듈 Lock

- 상호 배제 락(뮤텍스)

```python
from threading import Lock
class LockingCounter:
    def __init__(self):
        self.lock = Lock()
        self.count = 0
        
    def increment(self, offset):
        with self.lock:
            self.count += offset

counter = LockingCounter()

threads = []
for i in range(5):
    thread = Thread(target=worker, args=(i, how_many, counter))
    threads.append(thread)
    thread.start()

for thread in threads:
    thread.join()

expected = how_many * 5
found = counter.count
print(f'카운터 값은 {expected}여야 하는데, 실제로는 {found} 입니다')

>>>
카운터 값은 500000여야 하는데, 실제로는 500000 입니다
```

- worker 스레드를 사용하되 `LockingCounter` 를 사용하여 `lock` 상황일 때만 `count` 를 증가시키도록 하였음
    - `with` 문 사용하여 lock 활성화 하였음