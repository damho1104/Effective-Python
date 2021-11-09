# 56. 언제 동시성이 필요할지 인식하는 방법을 알아두라

## 1. 콘웨이 생명 게임을 통한 sequencial 프로그램 예제

```python
ALIVE = '*'
EMPTY = '-'

0      1       2       3       4
-----   -----   -----   -----   -----
-*---   --*--   --**-   --*--   -----
--**-   --**-   -*---   -*---   -**--
---*-   --**-   --**-   --*--   -----
-----   -----   -----   -----   -----
```

- 콘웨이의 생명 게임
    - 유한 상태 오토마타를 보여주는 예제
    - 임의의 크기인 2차원 그리드가 있으며 이 그리드의 각 셀은 비어 있거나 살아 있을 수 있음
        - ALIVE
        - EMPTY
    - 클럭이 한 번 틱할 때마다 게임 진행
    - 틱마다 각 셀은 자신의 주변 여덟 개 셀이 살아 있는지 살펴보고 주변 셀 중에서 살아 있는 셀의 개수으ㅔ 따라 계속 살아남을 지, 죽을지, 재생성할지 결정
    - 위 예제는 5x5 그리드

```python
class Grid:
    def __init__(self, height, width):
        self.height = height
        self.width = width
        self.rows = []
        for _ in range(self.height):
            self.rows.append([EMPTY] * self.width)

    def get(self, y, x):
        return self.rows[y % self.height][x % self.width]
        
    def set(self, y, x, state):
        self.rows[y % self.height][x % self.width] = state
        
    def __str__(self):
        ...

grid = Grid(5, 9)
grid.set(0, 3, ALIVE)
grid.set(1, 4, ALIVE)
grid.set(2, 2, ALIVE)
grid.set(2, 3, ALIVE)
grid.set(2, 4, ALIVE)
print(grid)

>>>
---*-----
----*----
--***----
---------
---------
```

```python
def count_neighbors(y, x, get):
    n_ = get(y - 1, x + 0) # 북(N)
    ne = get(y - 1, x + 1) # 북동(NE)
    e_ = get(y + 0, x + 1) # 동(E)
    se = get(y + 1, x + 1) # 남동(SE)
    s_ = get(y + 1, x + 0) # 남(S)
    sw = get(y + 1, x - 1) # 남서(SW)
    w_ = get(y + 0, x - 1) # 서(W)
    nw = get(y - 1, x - 1) # 북서(NW)
    neighbor_states = [n_, ne, e_, se, s_, sw, w_, nw]
    count = 0
    for state in neighbor_states:
        if state == ALIVE:
            count += 1
    return count    
```

- `count_neighbors`: 그리드에 대해 질의를 수행해 살아 있는 주변 셀 수를 반환하는 도우미 함수
- `Grid` 인스턴스를 넘기는 대신 `get` 함수를 파라미터로 받음
    - 코드 결합성 감소

```python
def game_logic(state, neighbors):
    if state == ALIVE:
        if neighbors < 2:
            return EMPTY  # 살아 있는 이웃이 너무 적음: 죽음
        elif neighbors > 3:
            return EMPTY  # 살아 있는 이웃이 너무 많음: 죽음
    else:
        if neighbors == 3:
            return ALIVE  # 다시 생성됨
    return state
```

- 콘웨이 생명 게임 3가지 규칙
    - 이웃한 셀 중 2개 이하가 살아있으면 가운데 셀이 죽음
    - 4개 이상이 살아있으면 가운데 셀이 죽음
    - 3개가 살아있으면 계속 살아남음, 빈 셀이면 살아 있는 상태로 변경

```python
def step_cell(y, x, get, set):
    state = get(y, x)
    neighbors = count_neighbors(y, x, get)
    next_state = game_logic(state, neighbors)
    set(y, x, next_state)
```

- `count_neighbors` 와 `game_logic` 을 사용하여 연결
- 각 틱마다 셀 상태를 변화시키는 함수를 한 그리드 셀에 대해 호출하여 현재 셀 상태를 알아내고 이웃 셀의 상태를 살펴본 후 다음 상태를 결정한 다음 결과 그리드의 셀의 상태를 갱신

```python
def simulate(grid):
    next_grid = Grid(grid.height, grid.width)
    for y in range(grid.height):
        for x in range(grid.width):
            step_cell(y, x, grid.get, next_grid.set)
    return next_grid
```

- 셀로 이루어진 전체 그리드를 한 단계 진행시켜서 다음 세대의 상태가 담긴 그리드를 반환하는 함수 정의

```python
class ColumnPrinter:
    ...
        
columns = ColumnPrinter()
for i in range(5):
    columns.append(str(grid))
    grid = simulate(grid)
        
print(columns)

>>>
    0     |     1     |     2     |     3     |     4
---*----- | --------- | --------- | --------- | ---------
----*---- | --*-*---- | ----*---- | ---*----- | ----*----
--***---- | ---**---- | --*-*---- | ----**--- | -----*---
--------- | ---*----- | ---**---- | ---**---- | ---***---
--------- | --------- | --------- | --------- | ---------
```

- sequencial 프로그램의 결과

## 2. 요구사항 변경

- 요구 사항 변경
    - `game_logic` 함수에서 I/O(소켓 통신) 이 필요
    - 그리드 상태와 인터넷을 통한 사용자 간 통신에 의해 상태 전이가 결정되는 MMO 게임
- 블로킹 I/O 를 `game_logic` 에 직접 추가

```python
def game_logic(state, neighbors):
    ...
    # 블로킹# I/O를 여기서 수행한다
    data = my_socket.recv(100)
    ...
```

- 단점
    - 블로킹 I/O 로 인해 프로그램 느려짐
- 해결 방안
    - I/O 병렬 처리

## 3. 정리

- `팬아웃(fan-out)`: 각 작업 단위에 대해 동시 실행되는 여러 실행 흐름을 만들어내는 과정
- `팬인(fan-in)`: 전체를 조율하는 프로세스 안에서 다음 단계로 진행하기 전에 동시 작업 단위의 작업이 모두 끝날 때까지 기다리는 과정
- python 은 팬아웃과 팬인을 지원하는 여러 내장 도구를 제공
- 각 접근 방식의 장단점을 이해하여 상황에 따라 잘 사용해야 함
- 다음 챕터부터 활용 방식에 대해 설명