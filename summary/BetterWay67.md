# 67. 지역 시간에는 time 보다는 datetime 을 사용하라

## 1. `time` 모듈

- `time` 내장 모듈의 `localtime` 함수를 사용해 유닉스 타임스탬프를 호스트 컴퓨터의 시간대에 맞는 지역 시간으로 변환해줌
- 지역 시간은 `strftime` 함수를 사용하여 human-readable 한 표현으로 출력

```python
import time

now = 1598523184
local_tuple = time.localtime(now)
time_format = '%Y-%m-%d %H:%M:%S'
time_str = time.strftime(time_format, local_tuple)
print(time_str)

>>>
2020-08-27 19:13:04
```

- 반대로 변환할 경우
    - `strptime` 함수로 문자열을 입력받아 시간으로 변경
    - `mktime` 함수를 사용하여 유닉스 타임스탬프로 변환

```python
time_tuple = time.strptime(time_str, time_format)
utc_now = time.mktime(time_tuple)
print(utc_now)

>>>
1598523184.0
```

- 특정 시간대에 속한 시간을 다른 지역의 시간대로 변환헤야 하는 경우
    - 함수 반환값을 직접 조작하여 변경 가능하나 시간대 변환은 좋은 생각이 아님
        - 시간대는 각 지역의 법에 따라 정해짐, 복잡
    - OS 에서 시간대 변경을 자동으로 처리해주는 설정 파일 제공
    - 그러나 windows 의 경우 `time` 이 제공하는 시간대 관련 기능 중 몇 가지를 사용할 수 없음

```python
import os
if os.name == 'nt':
    print("이 예제는 윈도우에서 작동하지 않습니다.")
else:
    parse_format = '%Y-%m-%d %H:%M:%S %Z'    # %Z는 시간대를 뜻함
    depart_icn = '2020-08-27 19:13:04 KST'
    time_tuple = time.strptime(depart_icn, parse_format)
    time_str = time.strftime(time_format, time_tuple)
    print(time_str)

>>>
2020-08-27 19:13:04
```

```python
arrival_sfo = '2020-08-28 04:13:04 PDT'
time_tuple = time.strptime(arrival_sfo, time_format)

>>>
Traceback ...
ValueError: unconverted data remains: PDT’
```

- 예제에서 KST 로 설정한 시간은 제대로 작동
- `strptime` 에 PDT(미국 태평양 시간대)를 사용하면 에러 발생
- 호스트 OS 에 따라 time 내장 모듈의 일부 지원이 되지 않을 수 있음

## 2. `datetime` 모듈

- `datetime` 을 사용하면 UTC 나 지역 시간과 같이 여러 시간대에 속한 시간을 상호 변환할 수 있음

```python
from datetime import datetime, timezone

now = datetime(2020, 8, 27, 10, 13, 4)     # 시간대 설정이 안 된 시간을 만듦
now_utc = now.replace(tzinfo=timezone.utc) # 시간대를# UTC로 강제 지정
now_local = now_utc.astimezone()           # UTC 시간을 디폴트 시간대로 변환
print(now_local)

>>>
2020-08-27 19:13:04+09:00
```

- datetime 모듈로 UTC 로 쉽게 변경 가능

```python
time_str = '2020-08-27 19:13:04'
now = datetime.strptime(time_str, time_format)   # 시간대 설정이 안 된 시간으로 문자열을 구문 분석
time_tuple = now.timetuple()         # 유닉스 시간 구조체로 변환
utc_now = time.mktime(time_tuple)    # 구조체로부터 유닉스 타임스탬프 생성
print(utc_now)

>>>
1598523184.0
```

- `datetime` 은 자신의 `tzinfo` 클래스와 이 클래스 안에 있는 메소드에 대해서만 시간대 관련 기능 제공
- 파이선 기본 설치는 UTC 를 제외한 시간대 정의가 없음
- 파이선 패키지 관리 도구로 pytz 모듈을 다운로드 받아서 추가 가능
- pytz 를 효과적으로 사용하기 위해 항상 지역 시간을 UTC 로 설정 필요

```python
import pytz

arrival_sfo = '2020-08-28 04:13:04'
sfo_dt_naive = datetime.strptime(arrival_sfo, time_format)   # 시간대가 설정되지 않은 시간
eastern = pytz.timezone('US/Pacific')                        # 샌프란시스코의 시간대
sfo_dt = eastern.localize(sfo_dt_naive)                      # 시간대를 샌프란시스코 시간대로 변경
utc_dt = pytz.utc.normalize(sfo_dt.astimezone(pytz.utc))     # UTC로 변경
print(utc_dt)

>>>
2020-08-28 11:13:04+00:00
```

- 기본으로 UTC `datetime` 을 얻으면 다른 지역 시간대로 변경이 쉬움

```python
korea = pytz.timezone('Asia/Seoul')
korea_dt = korea.normalize(utc_dt.astimezone(korea))
print(korea_dt)

>>>
2020-08-28 20:13:04+09:00
```