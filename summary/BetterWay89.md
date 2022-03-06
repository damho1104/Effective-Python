# 89. 리팩터링과 마이그레이션 방법을 알려주기 위해 warning 을 사용하라

## 1. API 변경 사항 전달

- 예전에 예측하지 못했던 새로운 요구 사항을 충족하기 위해 API 변경이 필요
- API가 작고 상위 의존 관계나 하위 의존 관계가 거의 없다면 API 를 변경하는 것도 단순한 과정
- 코드베이스가 커지면 API 변경과 호출 지점 변경을 함께 일관성 있게 수행하는 것이 어려움
    - API를 호출하는 지점 수가 너무 많아지거나
    - 여러 소스 코드 저장소에 호출 지점이 흩어지므로
- 사용자들에게 자신의 코드를 리팩터링하고 API 를 사용하는 부분을 최신 API 에 맞춰 변경하도록 하는 방법 찾아야 함
- 잘 작동함에도 불구하고 이 구현은 계산 단위가 암시적이므로 사용자가 오류를 저지르기 쉬움
    - 총알이 초당 1,000 미터 속도로 3초 동안 움직인 거리가 얼마인지 알고 싶다면 잘못된 결과 반환

```python
def print_distance(speed, duration):
    distance = speed * duration
    print(f'{distance} 마일')

print_distance(5, 2.5)

>>>
12.5 마일
```

```python
print_distance(1000, 3)

>>>
3000 마일
```

- `print_distance` 가 `speed` 와 `duration` 에 대한 단위와 계산한 값을 출력할 때 사용할 거리 단위를 선택적인 키워드 인자로 받게 하면 해결 가능

```python
# 키에 해당하는 단위로 돼 있는 값을 
# SI 단위계 단위로 바꿀 때 곱해야 하는 숫자를 저장하는 딕셔너리
CONVERSIONS = {
    'mph': 1.60934 / 3600 * 1000,  # 마일/초 -> 미터/초
    '시간': 3600,                  # 시간 -> 초
    '마일': 1.60934 * 1000,        # 마일 -> 미터
    '미터': 1,                     # 미터 -> 미터
    'm/s': 1,                      # 미터/초 -> 미터/초
    '초': 1,                       # 초 -> 초
}

def convert(value, units):
    rate = CONVERSIONS[units]
    return rate * value

def localize(value, units):
    rate = CONVERSIONS[units]
    return value / rate

def print_distance(speed, duration, *,
                   speed_units='mph',
                   time_units='시간',
                   distance_units='마일'):
    norm_speed = convert(speed, speed_units)
    norm_duration = convert(duration, time_units)
    norm_distance = norm_speed * norm_duration
    distance = localize(norm_distance, distance_units)
    print(f'{distance} {distance_units}')
```

```python
print_distance(1000, 3,
               speed_units='미터',
               time_units='초')

>>>
1.8641182099494205 마일
```

- 수정 방향은 좋으나 어떻게 기존 API 를 호출하는 모든 사용자가 항상 단위를 지정하게 할 수 있을까?
    - `print_distance` 에 의존하는 코드가 깨지는 경우를 최소화
    - 호출쪽 코드에 빨리 새로운 단위 인자를 포함할 수 있도록 장려하는 방법 필요

## 2. `warnings` 내장 모듈

- 다른 프로그래머에게 자신이 의존하는 모드가 변경되었으므로 각자의 코드를 변경하라고 안내 가능
- `print_distance` 를 변경하여 키워드 인자를 제공하지 않았다고 경고 발생 가능
- 인자 적용해 이 함수를 호출하고 `warnings` 의 `sys.stderr` 출력을 살펴보면 경고가 발생되었는지 확인 가능

```python
import warnings

def print_distance(speed, duration, *,
                   speed_units=None,
                   time_units=None,
                   distance_units=None):
    if speed_units is None:
        warnings.warn(
            'speed_units가 필요합니다', DeprecationWarning)
        speed_units = 'mph'

    if time_units is None:
        warnings.warn(
            'time_units가 필요합니다', DeprecationWarning)
        time_units = '시간'

    if distance_units is None:
        warnings.warn(
            'distance_units가 필요합니다', DeprecationWarning)
        distance_units = '마일'

    norm_speed = convert(speed, speed_units)
    norm_duration = convert(duration, time_units)
    norm_distance = norm_speed * norm_duration
    distance = localize(norm_distance, distance_units)
    print(f'{distance} {distance_units}')
```

```python
import contextlib
import io

fake_stderr = io.StringIO()
with contextlib.redirect_stderr(fake_stderr):
    print_distance(1000, 3,
                   speed_units='미터',
                   time_units='초')

print(fake_stderr.getvalue())

>>>
1.8641182099494205 miles
.../example.py:97: DeprecationWarning: distance_units가 필요합니다
warnings.warn(
```

- 문제
    - 경고 추가 시 반복적인 준비 코드가 필요하고 가독성 떨어짐
    - 경고 메시지는 `warnings.warn` 호출 위치를 표시함
    - 키워드 인자를 제공하지 않고 `print_distance` 를 호출한 위치를 가리키고 싶음
- `warnings.warn` 함수는 stacklevel 파라미터 지원
    - 호출 스택에서 경고를 발생시킨 위치를 제대로 보고 가능
    - stacklevel 를 활용하면 다른 코드를 대신해 경고를 표시하는 함수 쉽게 작성 가능

```python
def require(name, value, default):
    if value is not None:
        return value
    warnings.warn(
        f'{name}이(가) 곧 필수가 됩니다. 코드를 변경해 주세요',
        DeprecationWarning,
        stacklevel=3)
    return default

def print_distance(speed, duration, *,
                   speed_units=None,
                   time_units=None,
                   distance_units=None):
    speed_units = require('speed_units', speed_units, 'mph')
    time_units = require('time_units', time_units, '시간')
    distance_units = require('distance_units', distance_units, '마일')

    norm_speed = convert(speed, speed_units)
    norm_duration = convert(duration, time_units)
    norm_distance = norm_speed * norm_duration
    distance = localize(norm_distance, distance_units)
    print(f'{distance} {distance_units}')
```

```python
import contextlib
import io

fake_stderr = io.StringIO()
with contextlib.redirect_stderr(fake_stderr):
    print_distance(1000, 3,
                   speed_units='미터',
                   time_units='초')

print(fake_stderr.getvalue())

>>>
1.8641182099494205 마일
.../example.py:174: DeprecationWarning: distance_units이(가) 곧 필수가 됩니다. 코드를 변경해 주세요
  print_distance(1000, 3,
```

- `warnings` 모듈은 경고가 발생할 때 해야 할 작업을 설정할 수 있게 해줌
    - 한 가지 옵션은 모든 경고를 오류로 바꾸는 것
    - 경고할 일이 발생했을 때 `sys.stderr` 에 경고 출력 대신 예외가 발생

```python
warnings.simplefilter('error')
try:
    warnings.warn('이 사용법은 향후 금지될 예정입니다',
                  DeprecationWarning)
except DeprecationWarning:
    print("DeprecationWarning이 예외로 발생")
    pass  # 예외가 발생할 것으로 예상함

>>>
DeprecationWarning이 예외로 발생
```

- 예외를 발생시키는 동작 방식은 상위 의존 관계에 있는 변경을 감지해 적절히 실채하는 자동화된 테스트에서 유용
- `warnings.simplefilter('error')` 를 쓰지 않더라도 `-W error` 명령줄 인자를 파이선 인터프리터에 넘기거나 `PYTHONWARNINGS` 환경 변수를 설정해 사용 가능

```python
# ex6.py
import warnings
try:
    warnings.warn('이 사용법은 향후 금지될 예정입니다',
                  DeprecationWarning)
except DeprecationWarning:
    print("DeprecationWarning이 예외로 발생")
```

```python
$ python ex6.py
ex6.py:4: DeprecationWarning: 이 사용법은 향후 금지될 예정입니다
  warnings.warn('이 사용법은 향후 금지될 예정입니다',
```

```python
$ python -W error ex6.py
DeprecationWarning이 예외로 발생
```

- 앞으로 사용이 금지될 API에 의존하는 코드를 담당하는 사람들이 자신의 코드를 변경해야 한다는 사실을 알고 나면 `simplefilter` 와 `filterwarnings` 함수를 사용해 오류 무시 가능

```python
warnings.simplefilter('ignore')
warnings.warn('이 경고는 표준 오류(stderr)에 표시되지 않습니다')
```

- 배포 후 중요 시점에 프로그램이 중단될 수 있으므로 경고, 오류를 유발하는 것은 타당하지 않음
- 더 나은 접근 방법으로 경고를 `logging` 내장 모듈에 복제하는 것 고려 가능

```python
import logging

fake_stderr = io.StringIO()
handler = logging.StreamHandler(fake_stderr)
formatter = logging.Formatter(
    '%(asctime)-15s WARNING] %(message)s')
handler.setFormatter(formatter)

logging.captureWarnings(True)
logger = logging.getLogger('py.warnings')
logger.addHandler(handler)
logger.setLevel(logging.DEBUG)

warnings.resetwarnings()
warnings.simplefilter('default')
warnings.warn('이 경고는 로그 출력에 표시됩니다')

print(fake_stderr.getvalue())

>>>
2020-08-29 23:15:43,465 WARNING] ex8.py:17: UserWarning: 이 경고는 로그 출력에 표시됩니다
  warnings.warn('이 경고는 로그 출력에 표시됩니다')
```

- 로깅을 사용해 경고를 잡아내면 프로그램에 오류 보고 시스템이 설정된 경우 프로덕션 환경에서도 중요한 경고 통보 받기 가능

- API 라이브러리 관리자는 경고가 제대로 된 환경에서 명확하고 해결 방법을 제대로 알려주는 메시지와 함께 만들어지는 검증하는 단위 테스트 작성 필요

```python
with warnings.catch_warnings(record=True) as found_warnings:
    found = require('my_arg', None, '가짜 단위')
    expected = '가짜 단위'
    assert found == expected
```

```python
assert len(found_warnings) == 1
single_warning = found_warnings[0]
assert str(single_warning.message) == ('my_arg이(가) 곧 필수가 됩니다. 코드를 변경해 주세요')
assert single_warning.category == DeprecationWarning
```