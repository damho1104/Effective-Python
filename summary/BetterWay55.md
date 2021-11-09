# 55. Queue 를 사용해 스레드 사이의 작업을 조율하라

## 1. 파이프라인

- 동시성 작업을 처리할 때 가장 유용한 방식
- 순차적으로 실행해야 하는 여러 단계가 있고 각 단계마다 실행할 구체적인 함수가 정해짐
- 파이프라인의 한쪽 끝에는 새로운 작업이 추가
- 각 함수는 동시 실행될 수 있으며 처리해야 할 일을 담당
- 작업은 매 단계 함수가 완료될 때마다 다음 단계로 전달, 실행할 단계가 없을 때 끝
- 장점
    - 각 작업에 블로킹 I/O 나 하위 프로세스가 포함되는 경우 좋음

- 각 단계를 처리하는 함수 `download`, `resize`, `upload` 가 존재하는 예제

```python
def download(item):
    ...

def resize(item):
    ...

def upload(item):
    ...
```

- 세 함수를 사용해 동시성 파이프라인을 적용한 예제
    
    ```python
    from collections import deque
    from threading import Lock
    
    class MyQueue:
        def __init__(self):
            self.items = deque()
            self.lock = Lock()
    
        def put(self, item):
            with self.lock:
                self.items.append(item)
    
        def get(self):
            with self.lock:
                return self.items.popleft()
    ```
    
    - 생산자: deque 사용, 각 단계 추가는 deque 끝
    - 소비자: deque 의 맨 앞 아이템 꺼내오기
    
    ```python
    from threading import Thread
    import time
    
    class Worker(Thread):
        def __init__(self, func, in_queue, out_queue):
            super().__init__()
            self.func = func
            self.in_queue = in_queue
            self.out_queue = out_queue
            self.polled_count = 0
            self.work_done = 0
    
        def run(self):
            while True:
                self.polled_count += 1
                try:
                    item = self.in_queue.get()
                except IndexError:
                    time.sleep(0.01)  # 할 일이 없음
                else:
                    result = self.func(item)
                    self.out_queue.put(result)
                    self.work_done += 1
    ```
    
    - `MyQueue` 에서 가져온 작업에 함수를 적용, 그 결과를 다른 큐에 넣는 스레드를 통해 파이프라인의 각 단계 구현
    - 각 작업자가 얼마나 많이 새로운 입력을 검사(폴링)했고 많은 작업을 완료헀는지 추적
    - `MyQueue` 가 비어있는 경우 이전 단계가 아직 작업을 완료하지 못했다는 것
        - `IndexError` 예외를 잡아내서 sleep
    
    ```python
    download_queue = MyQueue()
    resize_queue = MyQueue()
    upload_queue = MyQueue()
    
    done_queue = MyQueue()
    threads = [
        Worker(download, download_queue, resize_queue),
        Worker(resize, resize_queue, upload_queue),
        Worker(upload, upload_queue, done_queue),
    ]
    
    for thread in threads:
        thread.start()
        
    for _ in range(1000):
        download_queue.put(object())
    
    while len(done_queue.items) < 1000:
        # 기다리는 동안 유용한 작업을 수행한다
        ...
    
    processed = len(done_queue.items)
    polled = sum(t.polled_count for t in threads)
    print(f'{processed} 개의 아이템을 처리했습니다, '
          f'이때 폴링을 {polled} 번 했습니다.')
    
    >>>
    1000 개의 아이템을 처리했습니다, 이때 폴링을 3009 번 했습니다.
    ```
    
    - 각 단계별 큐 생성, 각 단계에 맞는 작업 스레드를 만들어서 연결
    - 각 단계를 처리하기 위해 세 가지 스레드를 시작, 첫 번째 단계에 원하는 만큼 작업을 삽입
    - download 함수의 입력은 `object()` 로 표현
    - done_queue 를 지켜보면서 파이프라인이 모든 원소를 처리할 때까지 wait
    - **단점**
        - 스레드들이 새로운 작업을 기다리면서 큐를 폴링함
        - `run` 메소드 내 `IndexError` 예외를 잡아내는 부분이 꽤  많이 실행됨
        - 작업자 함수의 속도가 달라지면 앞에 있는 단계가 그보다 더 뒤에 있는 단계의 진행을 방해하면서 파이프라인을 막을 수 있음
            - starvation 발생, while 루프를 돌며 입력 큐를 계속 검사하게 됨
        - 모든 작업이 다 끝났는지 검사하기 위해 done_queue 에 대한 busy waiting
        - Worker 의 run 메소드가 루프를 무한 반복
        - 작업자 스레드에게 루프를 중단한 시점임을 알려줄 방법 없음
        - 파이프라인 진행이 막히면 프로그램이 중단될 수 있음
            - 두번째 진행 단계가 오래 걸리면 큐의 크기가 계속 늘어나고 시간과 입력 데이터가 쌓이게 되면 메모리 초과로 프로그램 비정상 종료
    
    ## 2. 대안: `Queue`
    
    - queue 내장 모듈에 있는 `Queue` 클래스 사용
        - 새로운 데이터가 나타날 때까지 `get` 메소드가 블록
            - 작업자의 busy wait 문제 해결
        - `Queue` 클래스에서 두 단계 사이에 허용할 수 있는 미완성 작업의 최대 개수 지정 가능
            - 버퍼 크기를 정하면 큐가 이미 가득 찬 경우 `put` 이 블록
            - 파이프라인 중간이 막히는 경우 해결
        - `Queue` 클래스의 `task_done` 메소드 사용
            - 작업의 진행 추적 가능
            - 어떤 단계의 입력 큐가 소진될 때까지 기다릴 수 있음
            - 파이프라인의 마지막 단계를 폴링할 필요 없어짐
        - 모든 동작을 `Queue` 하위 클래스에 넣고 처리가 끝났음을 작업자 스레드에게 알리는 기능 추가 가능
    
    ```python
    from queue import Queue
    
    my_queue = Queue()
    def consumer():
        print('소비자 대기')
        my_queue.get()  # 다음에 보여줄 put()이 실행된 다음에 실행된다
        print('소비자 완료')
    
    thread = Thread(target=consumer)
    thread.start()
    
    print('생산자 데이터 추가')
    my_queue.put(object())     # 앞에서 본 get()이 실행되기 전에 실행된다
    print('생산자 완료')
    thread.join()
    
    >>>
    소비자 대기
    생산자 데이터 추가
    생산자 완료
    소비자 완료
    ```
    
    - 스레드가 먼저 실행되지만 `Queue` 인스턴스에 원소가 `put` 돼서 `get` 메소드가 반환할 원소가 생기기 전까지 끝나지 않음
    
    ```python
    my_queue = Queue(1)  # 버퍼 크기 1
    
    def consumer():
        time.sleep(0.1)  # 대기
        my_queue.get()   # 두 번째로 실행됨
        print('소비자 1')
        my_queue.get()   # 네 번째로 실행됨
        print('소비자 2')
        print('소비자 완료')
    
    thread = Thread(target=consumer)
    thread.start()
    
    my_queue.put(object())  # 첫 번째로 실행됨
    print('생산자 1')
    my_queue.put(object())  # 세 번째로 실행됨
    print('생산자 2')
    print('생산자 완료')
    thread.join()
    
    >>>
    생산자 1
    소비자 1
    생산자 2
    생산자 완료
    소비자 2
    소비자 완료
    ```
    
    - 큐 원소가 소비될 때까지 대기하는 스레드 정의
    - 큐 원소가 없을 경우 소비자 스레드가 대기, 생산자 스레드는 소비자 스레드가 `get` 을 호출했는지 여부와 관계없이 `put` 을 두 번 호출해 객체를 큐에 추가할 수 있음
    - 생산자가 두 번째로 후출한 `put` 이 큐에 두 번째 원소로 넣으려면 소비자가 최소 한 번이라도 `get` 을 호출할 때까지 기다려야 함
    
    ```python
    in_queue = Queue()
    def consumer():
        print('소비자 대기')
        work = in_queue.get()  # 두 번째로 실행됨
        print('소비자 작업 중')
        # 작업 진행
        ...
        print('소비자 완료')
        in_queue.task_done()  # 세 번째로 실행됨
    
    thread = Thread(target=consumer)
    thread.start()
    
    print('생산자 데이터 추가')
    in_queue.put(object())    # 첫 번째로 실행됨
    print('생산자 대기')
    in_queue.join()           # 네 번째로 실행됨
    print('생산자 완료')
    thread.join()
    
    >>>
    소비자 대기
    생산자 데이터 추가
    생산자 대기
    소비자 작업 중
    소비자 완료
    생산자 완료
    ```
    
    - 생산자 코드가 소비자 스레드를 join 하거나 폴링할 필요가 없음
    - 생산자는 `Queue` 인스턴스의 `join` 메소드를 호출함으로써 `in_queue` 가 끝나기를 기다릴 수 있음
    - `in_queue` 가 비어 있더라도 지금까지 이 큐에 들어간 모든 원소에 대해 `task_done` 이 호출되기 전까지 join 이 끝나지 않음
    
    ```python
    class ClosableQueue(Queue):
        SENTINEL = object()
        
        def close(self):
            self.put(self.SENTINEL)
    
        def __iter__(self):
            while True:
                item = self.get()
                try:
                    if item is self.SENTINEL:
                        return   # 스레드를 종료시킨다
                    yield item
                finally:
                    self.task_done()
    ```
    
    - 큐에 더 이상 다른 입력이 없음을 표시하는 특별한 센티넬 원소를 추가하는 `close` 메소드 정의
    - 큐를 이터레이션하다 센티널 원소를 찾으면 이터레이션을 끝냄
    - 이터레이션 중에 큐의 작업 진행을 감시할 수 있게 `task_done` 을 적당한 횟수만큼 호출
    
    ```python
    class StoppableWorker(Thread):
        def __init__(self, func, in_queue, out_queue):
            super().__init__()
            self.func = func
            self.in_queue = in_queue
            self.out_queue = out_queue
            
        def run(self):
            for item in self.in_queue:
                result = self.func(item)
                self.out_queue.put(result)
    ```
    
    - 작업자 스레드가 `ClosableQueue` 클래스의 동작 활용 가능
    
    ```python
    download_queue = ClosableQueue()
    resize_queue = ClosableQueue()
    upload_queue = ClosableQueue()
    done_queue = ClosableQueue()
    threads = [
        StoppableWorker(download, download_queue, resize_queue),
        StoppableWorker(resize, resize_queue, upload_queue),
        StoppableWorker(upload, upload_queue, done_queue),
    ]
    
    for thread in threads:
        thread.start()
    
    for _ in range(1000):
        download_queue.put(object())
        
    download_queue.close()
    
    download_queue.join()
    resize_queue.close()
    resize_queue.join()
    upload_queue.close()
    upload_queue.join()
    print(done_queue.qsize(), '개의 원소가 처리됨')
    
    for thread in threads:
        thread.join()
    
    >>>
    1000 개의 원소가 처리됨
    ```
    
    - `ClosableQueue` 로 대체
    - 작업자 스레드를 실행, 첫 번째 단계의 입력 큐에 모든 입력 작업을 추가한  후, 입력이 모드 끝났음을 표시하는 신호 추가
    - 각 단계를 연결하는 큐를 `join`, 작업 완료 기다림
    - 각 단계가 끝날 때마다 다음 단계의 입력 큐 `close` 를 호출하여 작업이 더 이상 없음을 통지
    - 마지막 `done_queue` 에는 모든 출력이 들어있음
    
    ```python
    def start_threads(count, *args):
        threads = [StoppableWorker(*args) for _ in range(count)]
        for thread in threads:
            thread.start()
        return threads
        
    def stop_threads(closable_queue, threads):
        for _ in threads:
            closable_queue.close()
        closable_queue.join()
        
        for thread in threads:
            thread.join()
    ```
    
    - 앞선 접근 방법을 사용해 단계마다 여러 작업자 사용
        - I/O 병렬성을 높일 수 있음
        
    
    ```python
    download_queue = ClosableQueue()
    resize_queue = ClosableQueue()
    upload_queue = ClosableQueue()
    done_queue = ClosableQueue()
    
    download_threads = start_threads(
        3, download, download_queue, resize_queue)
    resize_threads = start_threads(
        4, resize, resize_queue, upload_queue)
    upload_threads = start_threads(
        5, upload, upload_queue, done_queue)
    
    for _ in range(1000):
        download_queue.put(object())
    
    stop_threads(download_queue, download_threads)
    stop_threads(resize_queue, resize_threads)
    stop_threads(upload_queue, upload_threads)
    
    print(done_queue.qsize(), '개의 원소가 처리됨')
    
    >>>
    1000 개의 원소가 처리됨
    ```
    
    - 각 코드 조각을 서로 연결해 파이프라인에 객체를 넣고 그 과정에서 큐와 스레드의 완료를 `join` 을 통해 기다리고 최종 결과 소비
- 선형적인 파이프라인의 경우 Queue 가 잘 작동, 다른 도구가 더 나은 상황도 존재함, 상황에 따라 맞게 설정할 것!