# 86. 배포 환경을 설정하기 위해 모듈 영역의 코드를 사용하라

## 1. 배포 환경에서 발생 가능한 문제

- 프로그램을 작성하는 궁극적인 목표
    - 프로덕션 환경에서 프로그램을 실행해 원하는 결과를 얻는 것
- 개발 환경은 프로덕션 환경과 아주 많이 다를 수 있음
- `venv` 같은 도구를 쓰면 모든 환경에 똑같은 파이선 패키지 설치 가능
- 문제
    - 프로덕션 환경의 경우 개발 환경에서 재현하기 힘든 외부 가정이 많이 있을 수 있음
    - 웹 서버 컨테이너 안에서 프로그램을 실행시키되 프로그램이 DB 에 접근할 수 있도록 하용하고 싶다고 가정
        - 프로그램 코드를 변경할 때마다 서버 컨테이너를 실행, DB 스키마 적절히 갱신 필요
        - DB 접근에 필요한 암호를 프로그램이 알아야 함
        - 프로그램에서 한 줄만 변경한 뒤 제대로 동작하는 검증하는데 이 모든 작업을 다시 해야 한다면 비용이 너무 비쌈

## 2. 배포 환경을 고려한 방법

- 프로그램 일부를 오버라이드하여 배포되는 환경에 따라 다른 기능을 제공
    - 두 파일의 차이
        - `TESTING` 상수 값이 다르다는 것
    - 다른 모듈은 `__main__` 모듈을 임포트해서 `TESTING` 값에 따라 자신이 정의하는 애트리뷰트 값 결정

```python
# dev_main.py
TESTING = True

import db_connection

db = db_connection.Database()

# prod_main.py
TESTING = False

import db_connection

db = db_connection.Database()
```

```python
# db_connection.py
import __main__

class TestingDatabase:
    ...
                        
class RealDatabase:
    ... 
    
if __main__.TESTING:
    Database = TestingDatabase
else:
    Database = RealDatabase
```

- 모듈 영역에서 실행되는 코드는 파이선 코드만 존재
- `if` 문을 모듈 수준에서 사용하면 모듈 안에서 이름이 정의되는 방식 결정 가능
    - 다양한 배포 환경에 맞춰 모듈 구성 가능
    - DB 설정과 같이 비용이 많이 드는 가정이 불필요한 배포 환경에서 설정 가능
- 이런 접근 방법은 외부 환경에 대한 가정을 우회하기 위한 용도 이상으로 사용될 수 있음
    - 프로그램이 호스트 플랫폼에 따라 다르게 작동되야 한다고 가정
    - 최상위 요소 정의 전 `sys` 모듈 사용하여 나눠보기
    - 비슷한 방식으로 `os.environ` 에서 얻은 환경 변수를 모듈 정의에 참조 가능
    
    ```python
    # db_connection.py
    import sys
    
    class Win32Database:
        ...
        
    class PosixDatabase:
        ...
        
    if sys.platform.startswith('win32'):
        Database = Win32Database
    else:
        Database = PosixDatabase
    ```