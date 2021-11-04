# 53. 블로킹 I/O의 경우 스레드를 사용하고 병렬성을 피하라

## 1. CPython

- 파이선의 표준 구현
- 2단계로 구성
    1. 소스 코드를 구문 분석해서 바이트코드로 변환
        - 8비트 명령어를 사용하는 저수준 프로그램 표현
            - 3.6 부터 16비트 명령어를 사용, wordcode
    2. 바이트코드를 스택 기반 인터프리터를 통해 실행
        - 바이트코드 인터프리터에 전역 인터프리터 락(GIL, Global Interpreter Lock) 사용하여 일관성 강제 유지
        - GIL
            - 상호 배제 락
            - 뮤텍스
            - 선점형 멀티스레드로 인해 영향을 받는 것을 방지
                - 한 스레드가 다른 스레드의 실행을 중간에 인터럽트시키고 제어를 가져올 수 있음
                - 인터럽트가 예기치 못한 때 발생하면 인터프리터의 상태가 오염될 수 있음
            - 역할
                - CPython 자체와 CPython이 사용하는 C 확장 모듈이 실행되면서 인터럽트가 함부로 발생하는 것을 방지
                - 인터프리터 상태가 제대로 유지되고 바이트코드 명령들이 제대로 실행되도록 만듬
            - side-effect
                - python 다중 스레드 지원하지만 GIL로 인해 여러 스레드 중 어느 하나만 진행 가능
                - 병렬 처리를 수행하고자 스레드를 사용하면 안됨
- 순차적 실행 예제
    
    ```python
    import time
    def factorize(number):
        for i in range(1, number + 1):
            if number % i == 0:
                yield i
    
    numbers = [2139079, 1214759, 1516637, 1852285]
    start = time.time()
    
    for number in numbers:
        list(factorize(number))
    
    end = time.time()
    delta = end - start
    print(f'총 {delta:.3f} 초 걸림')
    
    >>>
    총 0.256 초 걸림
    ```
    
- 스레드를 사용한 예제
    
    ```python
    from threading import Thread
    
    class FactorizeThread(Thread):
        def __init__(self, number):
            super().__init__()
            self.number = number
            
        def run(self):
            self.factors = list(factorize(self.number))
    
    start = time.time()
    threads = []
    for number in numbers:
        thread = FactorizeThread(number)
        thread.start()
        threads.append(thread)
    
    for thread in threads:
        thread.join()
    end = time.time()
    delta = end - start
    print(f'총 {delta:.3f} 초 걸림')
    
    >>>
    총 0.446 초 걸림
    ```
    
- 순차적 실행보다 스레드 실행이 더 오래 걸림
    - 해당 서적에서는 9900k 에서 WSL 상태에서 실행한 결과 근소하게 스레드 사용 예제의 시간이 더 빨랐으나 결국 성능향상은 없다고 강조함
    - 다른 언어에 비해 스레드를 하나씩 할당하면, 스레드 생성 및 조정에 따른 부가 비용이 발생
    - 해당 결과는 표준 CPython 인터프리터에서 GIL의 락 충돌과 스케줄링 부가 비용이 미치는 영향을 잘 보여주는 예
    

## 2. Python 에서 스레드를 지원하는 이유는?

- 다중 스레드를 사용하면 프로그램에 동시에 여러 일을 하는 것처럼 보이게 만들기 쉬움
    - 동시성 작업에 스레드를 사용하면 GIL에 의해 스레드 중 하나만 실행이 되지만 CPython 이 어느 정도 균일하게 각 스레드를 실행시키므로 다중 스레드를 통해 여러 함수를 동시에 실행 가능
- 블로킹 I/O 를 다루기 위함
    - python 이 특정 system call 을 사용할 때 발생
    - 파일 read/write, network interaction, hardware 간 통신과 같은 작업이 블로킹 I/O 에 속함
    - 스레드를 사용하면 운영체제가 system call 요청에 응답하는 데 걸리는 시간 동안 다른 작업 가능
- 직렬 포트를 통해 원격 제어 헬리콥터에 신호를 보내는 순차적 실행 예제
    
    ```python
    import select
    import socket
    
    def slow_systemcall():
        select.select([socket.socket()], [], [], 0.1)
    
    start = time.time()
    
    for _ in range(5):
        slow_systemcall()
    end = time.time()
    delta = end - start
    print(f'총 {delta:.3f} 초 걸림')
    
    >>>
    총 0.503 초 걸림
    ```
    
- 직렬 포트를 통해 원격 제어 헬리콥터에 신호를 보내는 스레드 실행 예제
    
    ```python
    start = time.time()
    threads = []
    for _ in range(5):
        thread = Thread(target=slow_systemcall)
        thread.start()
        threads.append(thread)
    
    def compute_helicopter_location(index):
        ...
    
    for i in range(5):
        compute_helicopter_location(i)
    
    for thread in threads:
        thread.join()
    
    end = time.time()
    delta = end - start
    print(f'총 {delta:.3f} 초 걸림')
    
    >>>
    총 0.102 초 걸림
    ```
    
- `slow_systemcall` 함수가 실행되는 동안 프로그램은 아무것도 작업을 진행하지 못함
    - `select` 시스템 콜에 의해 블록됨
    - 헬리콥터에 신호를 보내는 동안 헬리콥터가 다음에 어디로 이동할 지 계산 가능해야 하는데 순차적 실행은 이런 작업을 동시에 하지 못함
- `slow_systemcall` 을 여러 스레드에서 따로 호출하도록 하여 블록 작업과 python 작업을 동시에 가능하도록 구현

## 3. 정리

- GIL로 인해 생기는 한계가 있더라도 여러 스레드를 통해 시스템 콜을 병렬로 실행할 수 있음을 보여줬음
- 스레드 외에도 asyncio 내장 모듈과 같이 블로킹 I/O 를 처리하는 방법은 존재
- 대안을 사용하려면 각 실행 모드에 맞게 코드를 변경하는  추가 작업이 필요함
- 코드를 가급적 크게 변경하지 않고 블로킹 I/O 를 병렬로 실행하고 싶은 경우 스레드를 사용하는 것이 간편