ㅠ# 63. 응답성을 최대로 높이려면 asyncio 이벤트 루프를 블록하지 말라

## 1. [Better way 62](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay62.md) 문제

```python
import asyncio
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
```

- 문제
    - 출력 파일 핸들에 대한 `open`, `close`, `write` 호출이 주 이벤트 루프에서 발생
        - 시스템 콜 이므로 블록될 수 있음
        - 다른 코루틴이 진행되지 못함
        - 응답성 저하, 동시성이 강한 서버에서는 응답 시간 증대
- 해당 문제 발생인지 확인하고자 하는 경우
    - `asyncio.run` 함수에 `debug=True` 전달
    
    ```python
    import time
    
    async def slow_coroutine():
        time.sleep(0.5)  # 느린 I/O를 시뮬레이션함
    asyncio.run(slow_coroutine(), debug=True)
    
    >>>
    Executing <Task finished name='Task-1' coro=<slow_coroutine() done, defined at example.py:29> result=None created at .../asyncio/base_events.py:487> took 0.503 seconds
    ...
    ```
    

## 2. 해결 방안

- 이벤트 루프에서 시스템 콜이 이뤄질 잠재적 가능성을 최소화

```python
from threading import Thread

class WriteThread(Thread):
    def __init__(self, output_path):
        super().__init__()
        self.output_path = output_path
        self.output = None
        self.loop = asyncio.new_event_loop()

    def run(self):
        asyncio.set_event_loop(self.loop)
        with open(self.output_path, 'wb') as self.output:
            self.loop.run_forever()
            
        # 맨 마지막에 한 번 더 이벤트 루프를 실행해서 
        # 다른 이벤트 루프가# stop()에# await하는 경우를 해결한다
        self.loop.run_until_complete(asyncio.sleep(0))

    async def real_write(self, data):
        self.output.write(data)
        
    async def write(self, data):
        coro = self.real_write(data)
        future = asyncio.run_coroutine_threadsafe(coro, self.loop)
        await asyncio.wrap_future(future)

    async def real_stop(self):
        self.loop.stop()
        
    async def stop(self):
        coro = self.real_stop()
        future = asyncio.run_coroutine_threadsafe(coro, self.loop)
        await asyncio.wrap_future(future)

    async def __aenter__(self):
        loop = asyncio.get_event_loop()
        await loop.run_in_executor(None, self.start)
        return self
        
    async def __aexit__(self, *_):
        await self.stop()
```

- `write` 메소드는 실제 I/O 발생시키는 `real_write` 메소드가 thread-safety 하게 래핑
- 다른 스레드에서 실행되는 코루틴은 해당 클래스의 `write` 메소드를 실행하면서 `await`
    - `Lock` 필요 없음
- 다른 코루틴은 `stop` 메소드를 사용해 작업자 스레드에게 실행을 중단하라고 전달 가능
    - `write` 와 비슷하게 thread-safety 하게 작성
- `__acenter__`, `__aexit__` 를 사용하여 `with` 문과 함께 사용가능하도록 작성
    - 작업자 스레드가 주 이벤트 루프 스레드를 느리게 만들지 않으면서 제시간이 시작/종료 가능

```python
def readline(handle):
    ...                 

async def tail_async(handle, interval, write_func):
    ...         
    
async def run_fully_async(handles, interval, output_path):
    async with WriteThread(output_path) as output:
        tasks = []
        for handle in handles:
            coro = tail_async(handle, interval, output.write)
            task = asyncio.create_task(coro)
            tasks.append(task)
            
        await asyncio.gather(*tasks)

def confirm_merge(input_paths, output_path):
    ...
                        
input_paths = ...                    
handles = ...                        
output_path = ...                    

asyncio.run(run_fully_async(handles, 0.1, output_path))
confirm_merge(input_paths, output_path)
```

- `WriteThread` 클래스를 사용하여 `run_tasks` 를 완벽히 비동기적으로 변경
- 주어진 입력 핸들과 출력 파일 경로에 대해 작업자가 예상대로 작동하는지 검증 가능