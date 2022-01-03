# 68. copyreg 를 사용해 pickle을 더 신뢰성 있게 만들라

## 1. `pickle` 내장 모듈

- 파이선 객체를 바이트 스트림으로 직렬화하거나 바이트 스트림을 파이선 객체로 역직렬화 가능
- 피클된 바이트 스트림은 서로 신뢰할 수 없는 당사자 사이의 통신에 사용 X
- 목적
    - 개발자가 제어하는 프로그램들이 이진 채널을 통해 서로 파이선 객체를 넘기는데 있음
- **pickle 로 사용하는 것 보다 JSON 포맷을 사용 권장 (참고, Note)**
    - 설계상 `pickle` 모듈의 직렬화 형식은 unsafe
        - 악의적인 `pickle` 데이터가 자신을 역직렬화하는 파이선 프로그램의 일부를 취약하게 만들 수 있음
            - 직렬화한 데이터에는 원본 파이선 객체를 복원하는 방법을 표현하는 데이터 존재
            - 해당 데이터는 프로그램과 동일
    - JSON 모듈은 설계상 안전
        - 직렬화한 JSON 데이터에는 객체 계층 구조를 간단한게 묘사한 값 존재
        - JSON 데이터를 역직렬화해도 파이선 프로그램이 추가적인 위험에 노출되는 일은 없음

```python
class GameState:
    def __init__(self):
        self.level = 0
        self.lives = 4

state = GameState()
state.level += 1     # 플레이어가 레벨을 깼다
state.lives -= 1     # 플레이어가 재시도해야 한다

print(state.__dict__)

>>>
{'level': 1, 'lives': 3}
```

```python
import pickle

state_path = 'game_state.bin'
with open(state_path, 'wb') as f:
    pickle.dump(state, f)
```

- 게임 중인 플레이어의 진행 상태를 표현
- 진행 상태는 플레이어의 레벨과 남은 생명 개수
- 사용자가 게임을 그만두면 나중에 이어서 진행 가능하도록 프로그램이 게임 상태를 파일에 저장
- `pickle` 모듈을 사용하여 상태 저장 가능

```python
with open(state_path, 'rb') as f:
    state_after = pickle.load(f)
    
print(state_after.__dict__)

>>>
{'level': 1, 'lives': 3}
```

- 파일에 대해 `load` 함수를 호출하면 직렬화한 적이 전혀 없었던 것처럼 다시 `GameState` 객체를 돌려받을 수 있음

- 단점
    - 해당 접근 방법은 시간이 지나면서 게임 기능을 확장할 때 문제 발생 가능성 높음
    
    - 플레이어가 최고점을 목표로 점수를 얻을 수 있게 게임 변경한다고 가정
    
    ```python
    class GameState:
        def __init__(self):
            self.level = 0
            self.lives = 4
            self.points = 0   # 새로운 필드 
    ```
    
    - `pickle` 을 사용해 새로운 `GameState` 를 직렬화하는 코드는 동일
    - 다음 코드는 `dumps` 를 사용해 데이터를 직렬화한 후 `loads` 를 사용하여 문자열에서 객체로 역직렬화하는 과정
    
    ```python
    state = GameState()
    serialized = pickle.dumps(state)
    state_after = pickle.loads(serialized)
    print(state_after.__dict__)
    
    >>>
    {'level': 0, 'lives': 4, 'points': 0}
    ```
    
    - 사용자가 예전 버전에서 저장한 `GameState` 객체를 사용해 게임을 이어서 진행하길 원한다면?
    
    ```python
    with open(state_path, 'rb') as f:
        state_after = pickle.load(f)
        
    print(state_after.__dict__)
    
    >>>
    {'level': 1, 'lives': 3}
    ```
    
    - `points` 필드 사라짐
    - `pickle` 작동 방식의 side-effect
    

## 2. `copyreg` 내장 모듈

- 객체를 직렬화하고 역직렬화할 때 사용하는 함수 등록 가능
    - `pickle` 동작 제어 가능
    - `pickle` 동작 신뢰성 높일 수 있음
    
- 디폴트 애트리뷰트 값
    - 간단한 방법, 디폴트 인자가 있는 생성자를 사용하면 GameState 객체를 unpickle했을 때도 항상 필요한 모든 애트리뷰트를 포함시킬 수 있음
    
    ```python
    class GameState:
        def __init__(self, level=0, lives=4, points=0):
            self.level = level
            self.lives = lives
            self.points = points
    ```
    
    - 생성자를 pickling 에 사용하려면 `GameState` 객체를 받아서 `copyreg` 모듈이 사용할 수 있는 튜플 파라미터로 변환하는 helper 함수 필요
    - 반대 helper 함수 정의 필요
    - `copyreg` 를 통해 등록
    
    ```python
    def pickle_game_state(game_state):
        kwargs = game_state.__dict__
        return unpickle_game_state, (kwargs,)
    
    def unpickle_game_state(kwargs):
        return GameState(**kwargs)
    ```
    
    ```python
    import copyreg
    
    copyreg.pickle(GameState, pickle_game_state)
    ...
    state = GameState()
    state.points += 1000
    serialized = pickle.dumps(state)
    state_after = pickle.loads(serialized)
    print(state_after.__dict__)
    
    >>>
    {'level': 0, 'lives': 4, 'points': 1000}
    ```
    
    ```python
    # 변경
    class GameState:
        def __init__(self, level=0, lives=4, points=0, magic=5):
            self.level = level
            self.lives = lives
            self.points = points
            self.magic = magic   # 추가한 필드
    
    print('이전:', state.__dict__)
    state_after = pickle.loads(serialized)
    print('이후:', state_after.__dict__)
    
    >>>
    이전: {'level': 0, 'lives': 4, 'points': 1000}
    이후: {'level': 0, 'lives': 4, 'points': 1000, 'magic': 5}
    ```
    
    - unpickle 할 때 `GameState` 생성자를 호출하므로 디폴트값에 의해 잘 채워짐
    
- 클래스 버전 지정
    - 파이선 객체의 필드를 제거해 예전 버전 객체와의 하위 호환성이 없어지는 경우도 존재
    - 이런 경우 사용 가능한 방법
    
    - 게임에서 생명이라는 개념을 없애고 싶다고 가정
    - 역직렬화 불가
    
    ```python
    class GameState:
        def __init__(self, level=0, points=0, magic=5):
            self.level = level
            self.points = points
            self.magic = magic
    
    pickle.loads(serialized)
    
    >>>
    Traceback ...           
    TypeError: __init__() got an unexpected keyword argument 'lives’
    ```
    
    - `copyreg` 함수에게 전달하는 함수에 버전 파라미터를 추가하여 해결 가능
    - 새로운 `GameState` 객체를 피클링할 때 직렬화한 데이터의 버전이 2로 설정
    - 이전 버전 데이터에는 `version` 인자가 없음, 이에 맞춰 GameState 생성자에 전달할 인자를 적절히 변경
    
    ```python
    def pickle_game_state(game_state):
        kwargs = game_state.__dict__
        kwargs['version'] = 2
        return unpickle_game_state, (kwargs,)
    
    def unpickle_game_state(kwargs):
        version = kwargs.pop('version', 1)
        if version == 1:
            del kwargs['lives']
        return GameState(**kwargs)
    
    copyreg.pickle(GameState, pickle_game_state)
    print('이전:', state.__dict__)
    state_after = pickle.loads(serialized)
    print('이후:', state_after.__dict__)
    
    >>>
    이전: {'level': 0, 'lives': 4, 'points': 1000}
    이후: {'level': 0, 'points': 1000, 'magic': 5}
    ```
    
- 안정적인 임포트 경로
    - pickle 할 떄 마주칠 수 있는 문제
        - 클래스 이름이 변경되어 코드가 깨지는 경우
    
    - `GameState` 클래스를 `BetterGameState` 로 이름 변경
    - 역직렬화 시도 시 예외 발생
        - 피클된 데이터에 직렬화한 클래스의 임포트 경로가 존재
    
    ```python
    class BetterGameState:
        def __init__(self, level=0, points=0, magic=5):
            self.level = level
            self.points = points
            self.magic = magic
    
    pickle.loads(serialized)
    
    >>>
    Traceback ...           
    AttributeError: Can't get attribute 'GameState' on <module __main__' from 'my_code.py'>
    ```
    
    - `copyreg` 를 사용하여 언피클할 때 사용할 함수에 식별자 지정하여 해결 가능
    - 이 기능은 한번 더 간접 계층을 추가함
    - `copyreg` 를 쓰면 `BetterGameState` 대신 `unpickle_game_state` 에 대한 임포트 경로가 인코딩된다는 사실 확인 가능
    
    ```python
    copyreg.pickle(BetterGameState, pickle_game_state)
    
    state = BetterGameState()
    serialized = pickle.dumps(state)
    print(serialized)
    
    >>>
    b'\x80\x04\x95W\x00\x00\x00\x00\x00\x00\x00\x8c\x08__main__\x94\x8c\x13unpickle_game_state\x94\x93\x94}\x94(\x8c\x05level\x94K\x00\x8c\x06points\x94K\x00\x8c\x05magic\x94K\x05\x8c\x07version\x94K\x02u\x85\x94R\x94.
    ```
    
    - 단점
        - unpickle_game_state 함수가 위치하는 모듈의 경로를 바꿀 수 없음