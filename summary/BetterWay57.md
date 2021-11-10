# 57. 요구에 따라 팬아웃을 진행하려면 새로운 스레드를 생성하지 말라

## 1. 팬아웃 진행

- 각 작업 단위에 대해 동시 실행되는 여러 실행 흐름을 만들어내는 과정
- 여러 동시 실행 흐름을 만들어내는 팬아웃을 수행하고자 스레드를 사용할 경우 중요한 **단점** 존재
    - 단점 확인을 휘애 앞 챕터의 예제 사용

```python
from threading import Lock

ALIVE = '*'
EMPTY = '-'
class Grid:
    ...
                        
class LockingGrid(Grid):
    def __init__(self, height, width):
        super().__init__(height, width)
        self.lock = Lock()
        
    def __str__(self):
        with self.lock:
            return super().__str__()
        
    def get(self, y, x):
        with self.lock:
            return super().get(y, x)
            
    def set(self, y, x, state):
        with self.lock:
            return super().set(y, x, state)
```

- 스레드를 사용해 `game_logic` 메소드 안에서 I/O 를 수행함으로써 생기는 지연 시간 해결함
- `Grid` 클래스에 락 관련 동작을 추가하면 여러 스레드에서 인스턴스를 동시에 사용해도 안전한 하위 클래스 정의 가능

```python
from threading import Thread

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
    
def simulate_threaded(grid):
    next_grid = LockingGrid(grid.height, grid.width)
    
    threads = []
    for y in range(grid.height):
        for x in range(grid.width):
            args = (y, x, grid.get, next_grid.set)
            thread = Thread(target=step_cell, args=args)
            thread.start()  # 팬아웃
            threads.append(thread)
            
    for thread in threads:
        thread.join()       # 팬인        

    return next_grid
```

- 각 `step_cell` 호출마다 스레드를 정의해 팬아웃되도록 `simulate` 함수를 다시 정의
- 스레드는 병렬로 실행, 다른 I/O 가 끝날 때까지 기다리지 않아도 됨
- 다음 세대로 진행하기 전에 모든 스레드가 작업을 마칠 때까지 기다림, 팬인 가능

```python
class ColumnPrinter:
    ...
    
grid = LockingGrid(5, 9)            # 바뀐 부분
grid.set(0, 3, ALIVE)
grid.set(1, 4, ALIVE)
grid.set(2, 2, ALIVE)
grid.set(2, 3, ALIVE)
grid.set(2, 4, ALIVE)

columns = ColumnPrinter()
for i in range(5):
    columns.append(str(grid))
    grid = simulate_threaded(grid)  # 바뀐 부분
    
print(columns)

>>>
    0     |     1     |     2     |     3     |     4
---*----- | --------- | --------- | --------- | ---------
----*---- | --*-*---- | ----*---- | ---*----- | ----*----
--***---- | ---**---- | --*-*---- | ----**--- | -----*---
--------- | ---*----- | ---**---- | ---**---- | ---***---
--------- | --------- | --------- | --------- | ---------
```

- `step_cell` 을 그대로 사용하고 구동 코드가 `LockingGrid` 와 `simulate_threaded` 구현을 사용하도록 변경하여 방금 만든 그리드 클래스 코드를 구동할 수 있음

## 2. **위 예제 단점**

- 이로 인해 [Better way 56](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay56.md) 에서 확인한 순차적인 단일 스레드 코드보다 스레드를 사용하는 코드 가독성이 훨씬 떨어짐
    - `Thread` 인스턴스를 서로 안전하게 조율하려면 특별한 도구 필요(`Lock`)
    - 복잡도 떄문에 시간이 지남에 따라 스레드 사용 코드 확장 및 유지 보수가 어려움
- 스레드 사용은 메모리를 많이 소비하며 스레드 하나당 약 8MB 더 필요함
    - 실제 프로그램에서 스레드를 만들게 되면 메모리 감당이 어려움
- 스레드 시작 비용이 비싸며 context switching 비용으로 성능 하락
- 디버깅 어려움

## 3. 그렇다면 대체 방법은?

- 지속적으로 새로운 동시성 함수를 시작하고 끝내야 하는 경우 스레드는 적절한 해결책이 아님
- python 의 경우 더 나은 해결책 제공
    - 다음 챕터부터 더 나은 해결책에 대한 내용을 다룸