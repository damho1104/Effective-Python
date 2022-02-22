# 85. 패키지를 사용해 모듈을 체계화하고 안정적인 API를 제공하라

## 1. 패키지

- 프로그램의 코드베이스 크기가 늘어나면 자연스럽게 코드 구조를 체계화시킴
    - 큰 함수를 여러 작은 함수로
    - 데이터 구조를 도우미 클래스로
    - 기능을 나눠 서로 의존적인 여러 모듈로 분산화
    - 계층을 추가로 도입해서 코드를 좀 더 이해하기 쉽도록 변경
- 패키지
    - 다른 모듈들을 포함하는 모듈
    - 디렉토리에 `__init__.py` 를 정의하여 패키지 정의
    - `__init__.py` 가 존재하는 디렉토리를 기준으로 상대적인 경로를 통해 임포트해서 사용 가능
- 아래와 같은 구조로 코드가 존재한다면 `utils` 모듈을 임포트하기 위해 패키지 디렉토리 이름이 포함된 절대적인 모듈 이름을 사용

```python
main.py
mypackage/_ _init__.py
mypackage/models.py
mypackage/utils.py
```

```python
# main.py
from mypackage import utils
```

`

- 패키지 역할
    - 이름 공간(namespace)
    - 안정적인 API
    

## 2. 이름 공간 (namespace)

- 모듈을 별도의 이름 공간(namespace) 으로 분리
- 패키지를 사용하면 파일이름은 같지만 서로 다른 절대 유일한 경로를 통해 접근할 수 있는 모듈을 여럿 정의 가능

```python
# main.py
from analysis.utils import log_base2_bucket
from frontend.utils import stringify

bucket = stringify(log_base2_bucket(33))
```

- 패키지 내 정의된 함수, 클래스, 하위 모듈의 이름이 같으면 이런 접근 방법을 사용할 수 없음
    - `analysis.utils` 와 `frontend.utils` 에 있는 `inspect` 함수를 함께 사용하고 싶다고 가정
    - 이 애트리뷰트를 직접 임포트하면 두번째 `import` 문이 현재 영역의 `inspect` 값을 덮어 씀
    - `as` 절 사용하여 임포트 대상 이름 변경하여 처리
    
    ```python
    # main#2#.py
    from analysis.utils import inspect
    from frontend.utils import inspect  # 앞 줄에서 임포트한# inspect를 덮어 씀!
    ```
    
    ```python
    # main#3#.py
    from analysis.utils import inspect as analysis_inspect
    from frontend.utils import inspect as frontend_inspect
    
    value = 33
    if analysis_inspect(value) == frontend_inspect(value):
        print('인스펙션 결과가 같음!')
    ```
    
- `as` 절을 사용하면 `import` 로 가져온 대상이 무엇이든 관계없이 이름 변경 가능
    - 이 기능으로 이름 공간에 들어 있는 코드에 편하게 접근 가능
    - 이름 공간에 속한 대상을 사용할 때 어떤 것에 접근하는지 더 쉽게 식별 가능

- 임포트한 이름이 충돌하지 않게 막는 다른 방법
    - 최상위 모듈 이름을 항상 붙여서 사용
        - `as` 절 필요 없음
        - 이 방법은 코드를 처음 읽는 사람도 명확하게 확인 가능
    
    ```python
    # main4.py
    import analysis.utils
    import frontend.utils
    
    value = 33
    if (analysis.utils.inspect(value) ==
        frontend.utils.inspect(value)):
        print('인스펙션 결과가 같음!')
    ```
    

## 3. 안정적인 API

- 패키지 2번째 역할
    - 엄격하고 안정적인 API 를 제공
- 외부에 공개될 널리 사용될 API 를 작성할 경우 릴리스할 때 변경되지 않는 안정적인 기능 제공해야 함
    - 외부 사용자로부터 내부 코드 조직을 감춰야 함
    - 그렇게 해야 외부 사용자의 코드를 깨지 않고 패키지의 내부 모듈을 리팩터링, 개선 가능
- `__all__` 애트리뷰트를 통해 API 소비자에게 노출할 표면적을 제한할 수 있음
    - 모듈에서 외부로 공개된 API
    - 익스포트할 모든 이름이 들어 있는 리스트
    - `foo import *`를 실행한 소비자 코드는 `foo` 로부터 `foo.__all__` 에 있는 애트리뷰트만 임포트할 수 있음
    - `foo` 에 `__all__` 정의가 들어 있지 않으면 공개 애트리뷰트만 임포트

```python
# models.py
__all__ = ['Projectile']

class Projectile:
    def __init__(self, mass, velocity):
        self.mass = mass
        self.velocity = velocity
```

```python
# utils.py
from . models import Projectile

__all__ = ['simulate_collision']

def _dot_product(a, b):
    ...
                        
def simulate_collision(a, b):
    ...
```

- 아래와 같은 조건 가정
    - `mypackage` 의 `models` 모델에 발사체에 대한 표현을 정의
    - `Projectile` 인스턴스에 대한 연산을 `mypackage` 밑 `utils` 모듈에 정의
    - 이 API 를 사용하는 사용자들이 `mypackage.models` 나 `mypackage.utils` 를 임포트하지 않고 `mypackage` 에서 직접 필요한 요소를 임포트할 수 있음
    - `mypackage` 내부 구성을 변경해도 외부 사용자 코드는 문제없이 작동 가능
- 이 경우 `mypackage` 디렉토리의 `__init__.py` 변경
    - `mypackage` 임포트할 때 실제 패키지 내용으로 인식되는 파일
    - `__init__.py` 안에 외부 공개하고 싶은 이름만 제한적으로 임포트하면 `mypackage` 의 외부 API 를 명시적으로 지정 가능

```python
# __init__.py
__all__ = []

from . models import *
__all__ += models.__all__

from . utils import *
__all__ += utils.__all__
```

```python
# api_consumer.py
from mypackage import *

a = Projectile(1.5, 3)
b = Projectile(4, 1.7)
after_a, after_b = simulate_collision(a, b)
```

- `mypackage.utils._dot_product` 와 같은 내부 전용 함수는 `__all__` 에 없기 때문에 사용 불가