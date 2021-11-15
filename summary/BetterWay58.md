# 58. 동시성과 Queue 를 사용하기 위해 코드를 어떻게 리펙터링해야 하는지 이해하라

## 1. [Better way 57](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay57.md) 병렬 I/O 문제 해결 방안

- `queue` 내장 모듈의 `Queue` 클래스를 사용해 파이프라인을 스레드로 실행하는 방법
    - 생명 게임의 세대마다 셀당 하나씩 스레드를 생성하는 대신 필요한 병렬 I/O 숫자에 맞춰 미리 정해진 작업자 스레드 생성
    - 프로그램은 이를 통해 자원 사용 제어, 새로운 스레드를 자주 시작하며 생기는 side-effect 감소 가능
    
    - in_queue 에서 원소를 소비하는 스레드를 여러 개 시작할 수 있음
    - 각 스레드는 game_logic을 호출해 원소를 처리한 다음 out_queue 에 결과를 넣음
    - 각 스레드는 동시에 실행되며 병렬적으로 I/O 를 수행
        - 세대를 처리하는 지연 시간 감소
    
    ```python
    from queue import Queue
    
    class ClosableQueue(Queue):
        ...
                            
    in_queue = ClosableQueue()
    out_queue = ClosableQueue()
    ```
    
    ```python
    from threading import Thread
    
    class StoppableWorker(Thread):
        ...
    def game_logic(state, neighbors):
        ...
        # 여기서 블로킹# I/O를 수행한다
        data = my_socket.recv(100)
        ...
        
    def game_logic_thread(item):
        y, x, state, neighbors = item
        try:
            next_state = game_logic(state, neighbors)
        except Exception as e:
            next_state = e
        return (y, x, next_state)
        
    # 스레드를 미리 시작한다
    threads = []
    for _ in range(5):
        thread = StoppableWorker(
            game_logic_thread, in_queue, out_queue)
        thread.start()
        threads.append(thread)
    ```
    
    - 해당 큐와 상호작용하면서 상태 전이 정보 요청, 응답하도록 simulate 함수 재정의
    - `팬아웃`: 원소를 in_queue 에 추가하는 과정
    - `팬인`: 원소를 out_queue가 빈 큐가 될 때까지 원소를 소비하는 과정
    - `Grid.get` 과 `Grid.set` 호출은 모두 새로운 `simulate_pipeline` 함수에서만 발생
        - 동기화를 위해 `Lock` 인스턴스를 사용하는 `Grid` 구현을 대신 기존 단일 스레드 구현을 쓸 수 있다는 뜻
    
    ```python
    ALIVE = '*'
    EMPTY = '-'
    class SimulationError(Exception):
        pass
        
    class Grid:
        ...
        
    def count_neighbors(y, x, get):
        ...
        
    def simulate_pipeline(grid, in_queue, out_queue):
        for y in range(grid.height):
            for x in range(grid.width):
                state = grid.get(y, x)
                neighbors = count_neighbors(y, x, grid.get)
                in_queue.put((y, x, state, neighbors))  # 팬아웃
    
        in_queue.join()
        out_queue.close()
    
        next_grid = Grid(grid.height, grid.width)
        for item in out_queue:                          # 팬인
            y, x, next_state = item
            if isinstance(next_state, Exception):
                raise SimulationError(y, x) from next_state
            next_grid.set(y, x, next_state)
            
        return next_grid
    ```
    
    - `game_logic` 함수 안에서 I/O 를 하는 동안 발생하는 예외는 모두 잡혀 `out_queue` 에 전달되고 주 스레드에서 다시 던져짐
    
    ```python
    def game_logic(state, neighbors):
        ...
        raise OSError('게임 로직에서 I/O 문제 발생')
        ...
    
    simulate_pipeline(Grid(1, 1), in_queue, out_queue)
    
    >>>
    Traceback ...
                            
    OSError: 게임 로직에서 I/O 문제 발생
    
    The above exception was the direct cause of the following exception:
    
    Traceback ...
    SimulationError: (0, 0)
    ```
    
    - `simulate_pipeline` 을 루프 안에서 호출해 세대별 다중 스레드 파이프라인 구동 가능
    
    ```python
    class ColumnPrinter:
        ...
                            
    grid = Grid(5, 9)
    grid.set(0, 3, ALIVE)
    grid.set(1, 4, ALIVE)
    grid.set(2, 2, ALIVE)
    grid.set(2, 3, ALIVE)
    grid.set(2, 4, ALIVE)
    
    columns = ColumnPrinter()
    for i in range(5):
        columns.append(str(grid))
        grid = simulate_pipeline(grid, in_queue, out_queue)
        
    print(columns)
    
    for thread in threads:
        in_queue.close()
    for thread in threads:
        thread.join()
    
    >>>
        0     |     1     |     2     |     3     |     4
    ---*----- | --------- | --------- | --------- | ---------
    ----*---- | --*-*---- | ----*---- | ---*----- | ----*----
    --***---- | ---**---- | --*-*---- | ----**--- | -----*---
    --------- | ---*----- | ---**---- | ---**---- | ---***---
    --------- | --------- | --------- | --------- | ---------
    ```
    
    - 장점
        - 디버깅 쉬움
        - 메모리 폭발 문제 해결
        - 스레드 시작 비용 최소화
        

## 2. 해결 방안의 단점

- [Better way 56](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay56.md) 의 `simulate_thread` 방식보다 `simulate_pipeline` 어려움
- 코드 가독성 개선을 위해 `ClosableQueue`와 `StoppableWorker` 추가 지원 클래스 필요, 복잡도 증가
- 자동으로 시스템 규모가 확장되지 않음
    - 미리 부하를 예측해서 잠재적인 병렬성 수준을 미리 지정해야 함
- 디버깅하려면 발생한 예외를 작업 스레드에서 수동으로 잡아 `Queue` 전달시켜야 하므로 주 스레드에서 다시 발생시켜야 함
- 요구사항 변경 시 문제 발생

## 3. 요구사항 변경 시 발생하는 문제 및 해결 방안

- `count_neighbors` 에서도 I/O 수행해야 한다고 가정하면 또 별도의 스레드에서 실행하는 단계를 파이프라인에 추가해야 함
- 작업자 스레드 사이에서 예외 발생이 주 스레드까지 도달하는지 확인 필요
- 작업자 스레드 사이 동기화를 위해 `Lock` 사용 필수

```python
def count_neighbors(y, x, get):
    ...
    # 여기서 블로킹# I/O를 수행한다
    data = my_socket.recv(100)
    ...

def count_neighbors_thread(item):
    y, x, state, get = item
    try:
        neighbors = count_neighbors(y, x, get)
    except Exception as e:
        neighbors = e
    return (y, x, state, neighbors)
    
def game_logic_thread(item):
    y, x, state, neighbors = item
    if isinstance(neighbors, Exception):
        next_state = neighbors
    else:
        try:
            next_state = game_logic(state, neighbors)
        except Exception as e:
            next_state = e
    return (y, x, next_state)

class LockingGrid(Grid):
    ...
```

- `count_neighbors_thread` 작업자와 그에 해당하는 `Thread` 인스턴스를 위해 또 다른 `Queue` 인스턴스 집합 생성

```python
in_queue = ClosableQueue()
logic_queue = ClosableQueue()
out_queue = ClosableQueue()

threads = []

for _ in range(5):
    thread = StoppableWorker(
        count_neighbors_thread, in_queue, logic_queue)
    thread.start()
    threads.append(thread)
    
for _ in range(5):
    thread = StoppableWorker(
        game_logic_thread, logic_queue, out_queue)
    thread.start()
    threads.append(thread)
```

- 파이프라인의 여러 단계를 조율하고 팬아웃 혹은 팬인하도록 `simulate_pipeline` 변경

```python
def simulate_phased_pipeline(
        grid, in_queue, logic_queue, out_queue):
    for y in range(grid.height):
        for x in range(grid.width):
            state = grid.get(y, x)
            item = (y, x, state, grid.get)
            in_queue.put(item) # 팬아웃
            
    in_queue.join()
    logic_queue.join()         # 파이프라인을 순서대로 실행한다
    out_queue.close()
    
    next_grid = LockingGrid(grid.height, grid.width)
    for item in out_queue:     # 팬인
        y, x, next_state = item
        if isinstance(next_state, Exception):
            raise SimulationError(y, x) from next_state
        next_grid.set(y, x, next_state)
        
    return next_grid
```

```python
grid = LockingGrid(5, 9)
grid.set(0, 3, ALIVE)
grid.set(1, 4, ALIVE)
grid.set(2, 2, ALIVE)
grid.set(2, 3, ALIVE)
grid.set(2, 4, ALIVE)
columns = ColumnPrinter()
for i in range(5):
    columns.append(str(grid))
    grid = simulate_phased_pipeline(
        grid, in_queue, logic_queue, out_queue)
        
print(columns)
for thread in threads:
    in_queue.close()
for thread in threads:
    logic_queue.close()
for thread in threads:
    thread.join()

>>>
    0     |     1     |     2     |     3     |     4
---*----- | --------- | --------- | --------- | ---------
----*---- | --*-*---- | ----*---- | ---*----- | ----*----
--***---- | ---**---- | --*-*---- | ----**--- | -----*---
--------- | ---*----- | ---**---- | ---**---- | ---***---
--------- | --------- | --------- | --------- | ---------
```

- **단점**
    - 변경 부분 과다
    - 준비 코드 과다
    - `Queue` 로 팬아웃, 팬인 해결 가능하나 side-effect 비용이 높음
- `Thread` 만 사용하는 방식 보다는 `Queue` 사용 방식이 더 나으나 전체 I/O 병렬성의 정도를 제한함