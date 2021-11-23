# 60. I/O 를 할 때는 코루틴을 사용해 동시성을 높여라

## 1. 코루틴

- 파이선 프로그램 안에서 동시에 실행되는 것처럼 보이는 함수를 많이 사용 가능
- `async` 와 `await` 키워드를 사용해 구현
- 제너레이터를 실행하기 위한 인프라 사용
- 코루틴 시작 비용은 함수 호출뿐
- 활성화된 코루틴은 종료될 때까지 1kb 미만 메모리 사용
- 매 `await` 식에서 일시 중단, 일시 중단된 대기 가능성이 해결된 후 `async` 함수로부터 실행을 재개
- 여러 분리된 `async` 함수가 서로 실행되면 마치 모든 `async` 함수가 동시에 실행되는 것처럼 보임
    - 파이선 스레드의 동시성 동작 흉내 가능
- 스레드와 달리 메모리 부가 비용, 시작 비용, 컨텍스트 전환 비용이 들지 않음
- 복잡한 락과 동기화 코드 필요 없음
- 매커니즘: 이벤트 루프
    - 다수의 I/O 를 효율적으로 동시에 실행할 수 있고 이벤트 루프에 맞춰 작성된 함수들을 빠르게 전환해가며 실행 가능

## 2. 코루틴을 적용한 생명 게임 구현

- Queue 를 사용한 방법에서 나타난 문제를 극복하면서 `game_logic` 함수 I/O 를 발생시키기
- `game_logic` 함수 시작 시 `def` 대신 `async def` 를 사용하여 코루틴으로 변경
- `async def` 함수 내에서 `await` 구문 사용 가능

```python
ALIVE = '*'
EMPTY = '-'

class Grid:
    ...
def count_neighbors(y, x, get):
    ...
    
async def game_logic(state, neighbors):
    ...
    # 여기서# I/O를 수행한다
    data = await my_socket.read(50)
    ...
```

- `step_cell` 함수에도 `async def` 적용, `game_logic` 함수 호출 앞에 `await` 을 붙이면 `step_cell` 을 코루틴으로 적용 가능

```python
async def step_cell(y, x, get, set):
    state = get(y, x)
    neighbors = count_neighbors(y, x, get)
    next_state = await game_logic(state, neighbors)
    set(y, x, next_state)
```

- `simulate` 함수도 코루틴으로 변경
    - `step_cell` 을 호출해도 해당 함수가 즉시 호출되지 않음
    - `step_cell` 호출은 `await` 식에 사용할 수 있는 `coroutine` 인스턴스를 반환
    - `asyncio` 내장 라이브러리가 제공하는 `gather` 함수는 팬인을 수행
    - `gather` 에 대해 적용한 `await` 식은 이벤트 루프가 `step_cell` 코루틴을 동시에 실행하면서 `step_cell` 코루틴이 완료될 때마다 `simulate` 코루틴 실행을 재개하라고 요청
    - 모든 실행이 단일 스레드에서 이뤄짐, `Grid` 인스턴스에 락 사용할 필요 없음
    - I/O 는 `asyncio` 가 제공하는 이벤트 루프의 일부분으로 병렬화

```python
import asyncio
async def simulate(grid):
    next_grid = Grid(grid.height, grid.width)
    
    tasks = []
    for y in range(grid.height):
        for x in range(grid.width):
            task = step_cell(
                y, x, grid.get, next_grid.set) # 팬아웃
            tasks.append(task)
            
    await asyncio.gather(*tasks)               # 팬인
    
    return next_grid
```

- `asyncio.run` 함수를 사용해 `simulate` 코루틴을 이벤트 루프상에서 실행, 각 함수가 의존하는 I/O 실행

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
    grid = asyncio.run(simulate(grid)) # 이벤트 루프를 실행한다

print(columns)

>>>
    0     |     1     |     2     |     3     |     4
---*----- | --------- | --------- | --------- | ---------
----*---- | --*-*---- | ----*---- | ---*----- | ----*----
--***---- | ---**---- | --*-*---- | ----**--- | -----*---
--------- | ---*----- | ---**---- | ---**---- | ---***---
--------- | --------- | --------- | --------- | ---------
```

```python
async def game_logic(state, neighbors):
    ...
    raise OSError('I/O 문제 발생')
    ...
    
asyncio.run(game_logic(ALIVE, 3))

>>>
Traceback ...
OSError: I/O 문제 발생
```

- **결과**
    - 이전과 동일
    - 스레드와 관련된 모든 부가 비용 제거
    - 대화형 디버거를 사용해 코드를 한 줄씩 실행시켜볼 수 있음
        - `Queue` 혹은 `ThreadPoolExecutor` 접근 방법은 예외 처리에 한계 존재

- 추후 요구 사항 변경으로 `count_neighbors` 함수에서 I/O 수행이 필요하다면 기존 함수와 함수 호출 부분에 `async` 와 `await` 사용

```python
async def count_neighbors(y, x, get):
    ...
                        
async def step_cell(y, x, get, set):
    state = get(y, x)
    neighbors = await count_neighbors(y, x, get)
    next_state = await game_logic(state, neighbors)
    set(y, x, next_state)
    
grid = Grid(5, 9)
grid.set(0, 3, ALIVE)
grid.set(1, 4, ALIVE)
grid.set(2, 2, ALIVE)
grid.set(2, 3, ALIVE)
grid.set(2, 4, ALIVE)

columns = ColumnPrinter()
for i in range(5):
    columns.append(str(grid))
    grid = asyncio.run(simulate(grid))
    
print(columns)

>>>
    0     |     1     |     2     |     3     |     4
---*----- | --------- | --------- | --------- | ---------
----*---- | --*-*---- | ----*---- | ---*----- | ----*----
--***---- | ---**---- | --*-*---- | ----**--- | -----*---
--------- | ---*----- | ---**---- | ---**---- | ---***---
--------- | --------- | --------- | --------- | ---------
```

## 3. 정리

- 코루틴은 코드에서 외부 환경에 대한 명령과 원하는 명령을 수행하는 방법을 구현하는 것을 분리시켜줌
- 코루틴은 수만 개의 함수가 동시에 실행되는 것처럼 보이게 만드는 효과적인 방법을 제공