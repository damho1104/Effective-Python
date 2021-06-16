# 24. None 과 독스트링을 사용해 동적인 디폴트 인자를 지정하라

## 1. 독스트링

- Documentation String
- 보통 함수나 모듈 설명을 기록할 때 사용
- 사용법
    - """ """
    - ''' '''

```python
def log(message, when=None):
    """메시지와 타임스탬프를 로그에 남긴다.

    Args:
        message: 출력할 메시지.
        when: 메시지가 발생한 시각(datetime).
              디폴트 값은 현재 시간이다.
    """
    if when is None:
        when = datetime.now()
    print(f'{when}: {message}')
```

## 2. 잘못된 디폴트 인자 사용법

```python
from time import sleep
from datetime import datetime

def log(message, when=datetime.now()):
    print(f'{when}: {message}')

log('안녕!')
sleep(0.1)
log('다시 안녕!')

>>>
2020-08-18 11:29:12.857588: 안녕!
2020-08-18 11:29:12.857588: 다시 안녕!
```

- 원래 의도는 `log` 함수 호출 시 `when` 값에 항상 `datetime.now()` 가 호출되어 동적으로 할당하는 것 이었음
- 그러나 디폴트 인자는 load될 때 한 번만 `evaluation` 됨
- 해당 예제의 시간 값은 항상 같음

## 3. 개선 방법

- 디폴트 값으로 None 지정
- 실제 동작을 독스트링으로 기록
- 변수가 `None`인지를 체크하여 값 할당하면 됨

```python
def log(message, when=None):
    """메시지와 타임스탬프를 로그에 남긴다.

    Args:
        message: 출력할 메시지.
        when: 메시지가 발생한 시각(datetime).
              디폴트 값은 현재 시간이다.
    """
    if when is None:
        when = datetime.now()
    print(f'{when}: {message}')

log('안녕!')
sleep(0.1)
log('다시 안녕!')

>>>
2020-08-18 12:06:27.168336: 안녕!
2020-08-18 12:06:27.274338: 다시 안녕!
```

- `when` 변수는 디폴트 값이 `None` 이며 독스트링에 `when` 디폴트 값의 할당을 설명하였음
- 또한 `if` 절을 통해 함수 호출로 할당하도록 함

```python
import json
def decode(data, default={}):
    try:
        return json.loads(data)
    except ValueError:
        return default

foo = decode('잘못된 데이터')
foo['stuff'] = 5
bar = decode('또 잘못된 데이터')
bar['meep'] = 1
print('Foo:', foo)
print('Bar:', bar)

>>>
Foo: {'stuff': 5, 'meep': 1}
Bar: {'stuff': 5, 'meep': 1}
```

- `decode` 함수의 `default` 변수의 디폴트 값은 비어있는 `dictionary` 임
- 첫 `decode` 호출로 `foo` 엔 비어있는 `dictionary` 를 할당하였고 그후 `stuff` 키에 따른 값 5 가 추가되었음
- 두번째 `decode` 호출로 비어 있는 `dictionary` 를 할당하고 이후 `meep` 키에 따른 값 1 이 추가되었을까?
    - **아니다**
    - `dictionary` 는 `call by reference` 이므로 `foo` 와 `bar` 는 같은 객체를 가리킴
    - 그러므로 `dictionary` 는 값이 업데이트 되며 공유됨
- **해결방안**
    - `None` 디폴트 값, 독스트링, 함수 실행될 때마다 `dictionary` 새로 할당

```python
def decode(data, default=None):
    """문자열로부터 JSON 데이터를 읽어온다

    Args:
        data: 디코딩할 JSON 데이터.
        default: 디코딩 실패 시 반환할 값이다.
            디폴트 값은 빈 딕셔너리다.
    """
    try:
        return json.loads(data)
    except ValueError:
        if default is None:
            default = {}
        return default

foo = decode('잘못된 데이터')
foo['stuff'] = 5
bar = decode('또 잘못된 데이터')
bar['meep'] = 1
print('Foo:', foo)
print('Bar:', bar)
assert foo is not bar

>>>
Foo: {'stuff': 5}
Bar: {'meep': 1}
```

- `type annotation` 을 사용한 방법
    - `Optional` 을 사용했으므로 when의 값은 `datetime` 혹은 `None`

```python
from typing import Optional

def log_typed(message: str,
              when: Optional[datetime]=None) -> None:
    """메시지와 타임스탬프를 로그에 남긴다.

    Args:
        message: 출력할 메시지.
        when: 메시지가 발생한 시각(datetime).
            디폴트 값은 현재 시간이다.

    """
    if when is None:
        when = datetime.now()
 print(f'{when}: {message}')
```