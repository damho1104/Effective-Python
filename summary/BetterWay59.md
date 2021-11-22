# 59. 동시성을 위해 스레드가 필요한 경우에는 ThreadpoolExecutor 를 사용하라

## 1. `concurrent.futures` 의 `ThreadpoolExecutor` 내장 모듈

- Thread 와 Queue 를 사용한 접근 방법들의 장점을 조합하여 병렬 I/O 문제 해결

```python
ALIVE = '*'
EMPTY = '-'

class Grid:
    ...
    
class LockingGrid(Grid):
    ...
    
def count_neighbors(y, x, get):
    ...
    
def game_logic(state, neighbors):
    ...
    # 여기서 블로킹# I/O를 수행한다
    data = my_socket.recv(100)
    ...
    
def step_cell(y, x, get, set):
    state = get(y, x)
    neighbors = count_neighbors(y, x, get)
    next_state = game_logic(state, neighbors)
    set(y, x, next_state)
```

- `Grid` 의 각 셀에 대해 새 `Thread` 인스턴스를 시작하는 대신 함수를 실행기(executor)에 전달하여 팬아웃
- 실행기는 전달받은 함수를 별도의 스레드에서 수행
- 나중에 팬인하기 위해 모든 작업의 결과를 기다림

```python
from concurrent.futures import ThreadPoolExecutor

def simulate_pool(pool, grid):
    next_grid = LockingGrid(grid.height, grid.width)
    futures = []
    for y in range(grid.height):
        for x in range(grid.width):
            args = (y, x, grid.get, next_grid.set)
            future = pool.submit(step_cell, *args) # 팬아웃
            futures.append(future)
            
    for future in futures:
        future.result()                            # 팬인
        
    return next_grid
```

```python
class ColumnPrinter:
    ...
                        
grid = LockingGrid(5, 9)
grid.set(0, 3, ALIVE)
grid.set(1, 4, ALIVE)
grid.set(2, 2, ALIVE)
grid.set(2, 3, ALIVE)
grid.set(2, 4, ALIVE)

columns = ColumnPrinter()
with ThreadPoolExecutor(max_workers=10) as pool:
    for i in range(5):
        columns.append(str(grid))
        grid = simulate_pool(pool, grid)
        
print(columns)

>>>
    0     |     1     |     2     |     3     |     4
---*----- | --------- | --------- | --------- | ---------
----*---- | --*-*---- | ----*---- | ---*----- | ----*----
--***---- | ---**---- | --*-*---- | ----**--- | -----*---
--------- | ---*----- | ---**---- | ---**---- | ---***---
--------- | --------- | --------- | --------- | ---------
```

- 실행기는 사용할 스레드를 미리 할당
    - `simulate_pool` 을 실행할 떄 마다 스레드 시작 비용 감소
- 병렬 I/O 문제 해결 위한 Thread 로 인한 메모리 부족 문제를 스레드 풀에 사용할 스레드 최대 개수를 지정하여 해결
    - `max_workers` 파라미터 사용
- `**ThreadPoolExecutor` 장점**
    - `submit` 메소드가 반환하는 `Future` 인스턴스에 대해 `result` 메소드를 호출하면 스레드를 실행하는 중에 발생한 예외 자동 propagation

```python
def game_logic(state, neighbors):
    ...
    raise OSError('I/O 문제 발생')
    ...
    
with ThreadPoolExecutor(max_workers=10) as pool:
    task = pool.submit(game_logic, ALIVE, 3)
    task.result()

>>>
Traceback ...
OSError: I/O 문제 발생
```

- `count_neighbors` 함수에 I/O 병렬성을 제공해야 하는 경우
    - `ThreadPoolExecutor` 가 `step_cell` 의 일부분으로 두 함수를 이미 동시에 실행하고 있으므로 변경 필요 없음
- **단점**
    - `ThreadPoolExecutor` 가 제한된 수의 I/O 병렬성만 제공함
        - `max_workers` 를 100으로 설정해도 동시 I/O 가 필요한 그리드에 10000개 이상의 셀을 넣을 경우 확장 불가
    

## 2. 정리

- `ThreadPoolExecutor` 는 비동기적인 해법이 존재하지 않은 상황을 처리하는 경우 좋은 방법
- 많은 경우 I/O 병렬성을 최대화할 수 있는 더 나은 방법 존재(Corutine)