# 66. 재사용 가능한 try/finally 동작을 원한다면 contextlib 과 with 문을 사용하라

## 1. `with` 문

- 코드가 특별한 컨텍스트 안에서 실행되는 경우

```python
from threading import Lock
lock = Lock()
with lock:
    # 어떤 불변 조건을 유지하면서 작업을 수행한다
    ...
```

- `lock` 인스턴스가 `with` 문을 적절히 활성화해주므로 아래 예제와 동일한 효과를 지님

```python
lock.acquire()
try:
    # 어떤 불변 조건을 유지하면서 작업을 수행한다
    ...
finally:
    lock.release()
```

## 2. contextlib 내장 모듈

- 사용자가 만든 객체나 함수를 `with` 문에서 쉽게 사용 가능
- `contextmanager` 데코레이터 제공
- 이 데코레이터를 사용하는 방법이 `__enter__` 와 `__exit__` 특별 메소드를 사용하여 클래스 정의하는 방법보다 쉬움

```python
import logging

def my_function():
    logging.debug('디버깅 데이터')
    logging.error('이 부분은 오류 로그')
    logging.debug('추가 디버깅 데이터')

my_function()

>>>
ERROR:root:이 부분은 오류 로그
```

- 프로그램의 기본 로그 수준은 warning 이므로 위 예제는 오류만 출력됨

- `contextmanager` 데코레이터를 사용한 예제

```python
from contextlib import contextmanager

@contextmanager
def debug_logging(level):
    logger = logging.getLogger()
    old_level = logger.getEffectiveLevel()
    logger.setLevel(level)
    try:
        yield
    finally:
        logger.setLevel(old_level)

with debug_logging(logging.DEBUG):
    print('* 내부:')
    my_function()

print('* 외부:')
my_function()

>>>
* 내부:
DEBUG:root:디버깅 데이터
ERROR:root:이 부분은 오류 로그
DEBUG:root:추가 디버깅 데이터
* 외부:
ERROR:root:이 부분은 오류 로그
```

- 이 함수의 로그 수준을 높일 수 있음
- `with` 블록을 실행시키기 전 로그 심각성 수준을 높이고 블록 실행 후 심각성 수준을 이전 수준으로 복구
- `yield` 를 통해 `with` 블록의 코드가 실행됨, `with` 블록의 코드 실행이 끝나면 `finally` 로 넘어와서 진행

## 3. `with` 와 대상 변수 함께 사용하기

- `with` 문에 전달된 컨텍스트 매니저가 객체를 반환할 수도 있음
- 반환된 객체는 `with` 복합문의 일부로 지정된 지역 변수에 대입
- `with` 블록 안에서 실행되는 코드가 직접 컨텍스트 객체와 상호작용 가능

- 파일을 작성하고 이 파일이 제대로 닫혔는지 확인하고 싶은 경우
    - `with` 문에서 `as` 를 통해 대상으로 지정된 변수에게 파일 핸들을 전달, `with` 블록에서 나갈 때 핸들을 닫음

```python
with open('my_output.txt', 'w') as handle:
    handle.write('데이터입니다!')
```

- `as` 대상 변수에 값을 제공하기 위한 방법
    - `yield` 식에서 반환

```python
@contextmanager
def log_level(level, name):
    logger = logging.getLogger(name)
    old_level = logger.getEffectiveLevel()
    logger.setLevel(level)
    try:
        yield logger
    finally:
        logger.setLevel(old_level)

with log_level(logging.DEBUG, 'my-log') as logger:
    logger.debug(f'대상: {logger.name}!')
    logging.debug('이 메시지는 출력되지 않습니다')

>>>
DEBUG:my-log:대상: my-log!
```

- `as` 대상 변수로 얻은 로그 객체에 `debug` 와 같은 메소드 호출 시 `with` 블록 내 로그 심각성 수준이 낮게 설정되어 있으므로 디버깅 메시지 출력
- 기본 로그 심각성 수준이 warning 이므로 `logging` 모듈을 직접 사용해 debug 로그 메소드를 호출하면 출력 안됨

## 4. 정리

- `with` 문과 `contextmanager` 데코레이터를 사용의 장점
    - 상태를 격리 가능
    - 컨텍스트를 만드는 부분과 안에서 실행되는 코드를 서로 분리 가능