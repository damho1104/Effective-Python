# 62. asyncio 로 쉽게 옮겨갈 수 있도록 스레드와 코루틴을 함께 사용하라

## 1. 마이그레이션

- 블로킹 I/O 에 스레드를 사용하는 부분, 비동기 I/O 에 코루틴을 사용하는 부분이 서로 호환되면서 공존해야 함
    - 서로 변경 가능해야 함
- `asyncio` 내장 모듈에서 지원

- 로그 파일을 한 출력 스트림으로 병합하는 프로그램
    - 파인 핸들이 주어지면 새로운 데이터가 도착했는지 감시, 다음 줄 반환
    - `tell` 메소드를 사용하면 현재 읽기 중인 위치가 파일의 길이와 일치하는지 확인 가능
    - `while` 루프로 감싸서 작업자 스레드 생성
    - 데이터가 없는 경우 CPU 연산 시간 줄이기 위해 스레드를 일시 정지 상태로 변환, 입력 파일 핸들이 닫히면 스레드 종료

```python
class NoNewData(Exception):
    pass
    
def readline(handle):
    offset = handle.tell()
    handle.seek(0, 2)
    length = handle.tell()

    if length == offset:
        raise NoNewData
    
    handle.seek(offset, 0)
    return handle.readline()
```

```python
import time
 
def tail_file(handle, interval, write_func):
    while not handle.closed:
        try:
            line = readline(handle)
        except NoNewData:
            time.sleep(interval)
        else:
            write_func(line)
```

- `write` 헬퍼 함수
    - `Lock` 인스턴스를 사용해 출력 스트림에 데이터를 쓰는 순서 직렬화
    - 각 줄이 중간에 충돌하여 서로 섞이는 일이 없도록 함
- 입력 핸들이 열려 있는 한 작업자 스레드도 살아 있음

```python
from threading import Lock, Thread

def run_threads(handles, interval, output_path):
    with open(output_path, 'wb') as output:
        lock = Lock()
        def write(data):
            with lock:
                output.write(data)
                
        threads = []
        for handle in handles:
            args = (handle, interval, write)
            thread = Thread(target=tail_file, args=args)
            thread.start()
            threads.append(thread)
            
        for thread in threads:
            thread.join()
```

```python
def confirm_merge(input_paths, output_path):
    ...
                        
input_paths = ...
handles = ...        
output_path = ...
run_threads(handles, 0.1, output_path)

confirm_merge(input_paths, output_path)
```

## 2. 점진적인 마이그레이션

- 하향식
    - `main` 진입점과 같이 코드베이스에서 가장 높은 구성 요소로부터 시작해 점차 호출 계층의 리프 부분에 위치한 개별 함수와 클래스로 내려가면서 작업
    - 다른 프로그램에 사용하는 공통 모듈이 많은 경우 유용
    - 진입점부터 차례로 포팅, 공통 모듈 포팅이 끝나면 모든 곳에서 코루틴 사용
    - 단계
        1. 최상위 함수가 `def` 대신 `async def` 를 사용하게 변경
        2. 최상위 함수가 I/O 를 호출하는 모든 부분(이벤트 루프가 블록될 가능성 존재) 을 `asyncio.run_in_executor` 로 감싸라
        3. `asyncio.run_in_executor` 호출이 사용하는 자원이나 콜백이 제대로 동기화되었는지 확인하라
            - `Lock`
            - `asyncio.run_coroutine_threadsafe` 함수
        4. 호출 계층의 리프로 따라가면서 중간에 있는 함수와 메소드를 코루틴으로 변환하며 `get_event_loop` 와 `run_in_executor` 호출을 없애려고 시도하라
    - `run_thread` 함수에 대해 적용
        
        ```python
        import asyncio
        
        async def run_tasks_mixed(handles, interval, output_path):
            loop = asyncio.get_event_loop()
            
            with open(output_path, 'wb') as output:
                async def write_async(data):
                    output.write(data)
                    
                def write(data):
                    coro = write_async(data)
                    future = asyncio.run_coroutine_threadsafe(
                        coro, loop)
                    future.result()
                    
                tasks = []
                for handle in handles:
                    task = loop.run_in_executor(
                        None, tail_file, handle, interval, write)
                    tasks.append(task)
                    
                await asyncio.gather(*tasks)
        ```
        
        - `run_in_executor` 메소드는 이벤트 루프가 특정 `ThreadPoolExecutor` 나 디폴트 실행기 인스턴스를 사용해 주어진 함수(`test_fail`)를 실행하게 만듬
        - `run_in_executor` 함수를 이에 대응하는 `await` 식 없이 여러 번 호출하여 `run_tasks_mixed` 코루틴은 각 입력 파일마다 파일을 한 줄씩 처리하는 작업 팬아웃
        - `asyncio.gather` 함수와 `await` 식으로 `tail_file` 이 모두 종료되도록 팬인
        - `asyncio.run_coroutine_threadsafe` 를 사용하므로 `Lock` 혹은 `writer` 헬퍼 사용 필요 없음
            - `asyncio.run_coroutine_threadsafe` 를 사용하면 작업자 스레드가 코루틴(`write_sync`)를 호출하여 주 스레드에서 실행되는 이벤트 루프를 통해 실행
            - 스레드 간 동기화 효과
            - 출력 파일에 기록하는 작업이 모두 이벤트 루프에 의해 주 스레드에서 수행되도록 보장 가능
            - `asyncio.gather` 대기가 끝나면 출력 파일에 대한 기록도 끝났음을 가정할 수 있으므로 `with` 문 통해 close
        
        ```python
        input_paths = ...
        handles = ...
        output_path = ...
        asyncio.run(run_tasks_mixed(handles, 0.1, output_path))
        confirm_merge(input_paths, output_path)
        ```
        
    
    - `run_tasks_mixed` 함수에 4단계 적용
        
        ```python
        async def tail_async(handle, interval, write_func):
            loop = asyncio.get_event_loop()
            
            while not handle.closed:
                try:
                    line = await loop.run_in_executor(None, readline, handle)
                except NoNewData:
                    await asyncio.sleep(interval)
                else:
                    await write_func(line)
        ```
        
    
    - `tail_async` 사용 시 `get_event_loop` 와 `run_in_executor` 를 `run_tasks_mixed` 함수에서 완전히 제거해 호출 계층의 한 단계 아래로 내려보낼 수 있음
        
        ```python
        async def run_tasks(handles, interval, output_path):
            with open(output_path, 'wb') as output:
                async def write_async(data):
                    output.write(data)
                    
                tasks = []
                for handle in handles:
                    coro = tail_async(handle, interval, write_async)
                    task = asyncio.create_task(coro)
                    tasks.append(task)
        
                await asyncio.gather(*tasks)
        
        input_paths = ...
        handles = ...        
        output_path = ...
        asyncio.run(run_tasks(handles, 0.1, output_path))
        
        confirm_merge(input_paths, output_path)
        ```
        
- 상향식
    - 하향식과 비슷
    - 변환 과정에서 호출 계층을 반대 방향으로 옮겨간다는 점이 다름
    - 단계
        1. 프로그램에서 앞 부분에 있는 포팅하려는 함수의 비동기 코루틴 버전을 새로 만들기
        2. 기존 동기 함수를 변경하여 코루틴 버전을 호출하고 실제 동작을 구현하는 대신 이벤트 루프를 실행
        3. 호출 계층을 한 단계 올려서 다른 코루틴 계층을 만들고 기존에 동기적 함수를 호출하던 부분을 1단계에서 정의한 코루틴 호출로 변경
        4. 비동기 부분을 결합하기 위해 2단계에서 만든 동기 래퍼 삭제
    
    - `tail_file` 변환
        - `tail_async` 래핑
    
    ```python
    def tail_file(handle, interval, write_func):
        loop = asyncio.new_event_loop()
        asyncio.set_event_loop(loop)
        
        async def write_async(data):
            write_func(data)
            
        coro = tail_async(handle, interval, write_async)
        loop.run_until_complete(coro)
    ```