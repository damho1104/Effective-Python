# 88. 순환 의존성을 깨는 방법을 알아두라

## 1. 순환 의존성

- 모듈들이 상호 의존하는 경우가 생김
- 문제
    - 아래 예제에서 prefs 객체가 들어있는 `app` 모듈이 프로그램 시작 시 대화창을 표시하고자 앞에서 정의한 `Dialog` 클래스를 임포트 함
    - 순환 의존 관계가 생김
    - `app` 모듈을 메인 프로그램에서 임포트 시도화면 예외 발생

```python
# dialog.py
import app

class Dialog:
    def __init__(self, save_dir):
        self.save_dir = save_dir
    ...
                            
save_dialog = Dialog(app.prefs.get('save_dir'))
    
def show():
    ...
```

```python
# app.py
import dialog

class Prefs:
    ... 
    def get(self, name):
        ...
                        
prefs = Prefs()
dialog.show()
```

```python
# main.py
import app

...

>>>
$ python3 main.py
Traceback (most recent call last):
  File ".../main.py", line 17, in <module>
    import app
  File ".../app.py", line 17, in <module>
    import dialog
  File ".../dialog.py", line 23, in <module>
    save_dialog = Dialog(app.prefs.get('save_dir'))
AttributeError: partially initialized module 'app' has no attribute 'prefs' (most likely due to a circular import)
```

- 임포트 기능 작동 원리 파악이 필요함
    - 자세한 사항을 살펴보려면 `importlib` 내장 패키지로 확인 가능
    - 모듈이 임포트되면 어떤 일을 하는지 깊이 우선 순위(depth first order)로 표현
        1. `sys.path` 에서 모듈 위치를 검색
        2. 모듈의 코드를 로딩하고 컴파일되는지 확인
        3. 임포트할 모듈에 상응하는 빈 모듈 객체 생성
        4. 모듈을 `sys.modules` 에 삽입
        5. 모듈 객체에 있는 코드를 실행해서 모듈 내용 정의
- 순환 의존 관계에서 문제
    - 어떤 모듈의 애트리뷰트를 정의하는 코드(5 단계)가 실제로 수행되기 전까지는 모듈 애트리뷰트가 정의되지 않음
    - 모듈 자체는 `import` 문을 사용해서 `sys.modules` 에 추가되자마자(4 단계) `import` 문을 사용해 로드 가능
- 위 예제에서의 문제
    - `app` 모듈이 다른 모든 내용 정의 전 `dialog` 모듈 임포트
    - 그 후 `dialog` 모듈은 `app` 모듈 임포트
    - `app` 이 아직 실행되지 않았기 떄문에 `app` 모듈은 비어 있음
    - 따라서 `prefs` 애트리뷰트를 정의하는 코드가 아직 실행되지 못했기 때문에 `AttributeError` 발생
- 해결 방안
    - 코드 리팩터링
        - `prefs` 데이터 구조를 의존 관계 트리 맨 밑바닥으로 보내는 것
        - 리팩터링이 어려워서 노력할 만한 가치가 없거나 명확한 구분 불가

## 2. 순환 의존성 해결 방법

### 2-1. **임포트 순서 바꾸기**

- `app` 모듈의 다른 내용이 모두 실행된 다음 맨 뒤에서 `dialog` 모듈을 임포트하면 `AttributeError` 사라짐
    - dialog 모듈이 나중에 로딩될 때 dialog 안에서 재귀적으로 임포트한 app 에 app.prefs 가 이미 정의되어 있기 때문

```python
# app.py
class Prefs:
    ...
    
prefs = Prefs()
import dialog  # 위치 바꿈
dialog.show()
```

- `AttributeError` 가 사라지지만 PEP 8 스타일 가이드에 위배
    - 스타일 가이드에서는 항상 파이선 파일 맨 위에 임포트를 넣으라고 제안
- 임포트가 맨 앞에 있어야 의존하는 모듈이 개발자 모든 영역에서 항상 사용 가능할 것이라 확신 가능
    - 뒷부분에 임포트 넣으면 깨지기 쉽고 코드 순서를 약간만 바꿔도 모듈 망가질 수 있음

### 2-2. **임포트, 설정, 실행**

- 임포트 시점에 부작용을 최소화한 모듈 사용
    - 모듈이 함수, 클래스, 상수만 정의하게 하고 임포트 시점에 실제로 함수를 전혀 실행하지 않도록 함
    - 그 후 다른 모듈이 모두 임포트를 끝낸 후 호출할 수 있는 `configure` 함수 제공
        - `configure` 함수 목적
            - 다른 모듈들의 애트리뷰트에 접근해 모듈 상태를 준비하는 것
        - 다른 모든 모듈을 임포트한 다음 `configure` 를 실행하므로 `configure` 실행 시점에는 항상 모든 애트리뷰트가 정의되어 있음
- 다음 예제는 `configure` 호출될 때만 `prefs` 객체에 접근하도록 `dialog` 모듈 재정의
- `app` 모듈도 임포트 시 동작을 수행하지 않게 재정의
- `main` 모듈은 모든 것을 임포트, `configure` 하고 프로그램 첫 동작을 실행하는 3 가지 단계를 수행

```python
# dialog.py
import app
class Dialog:
    ...
                        
save_dialog = Dialog()

def show():
    ...
                        
def configure():
    save_dialog.save_dir = app.prefs.get('save_dir')
```

```python
# app.py
import dialog

class Prefs:
    ...
                        
prefs = Prefs()

def configure():
    ...
```

```python
# main.py
import app
import dialog
app.configure()
dialog.configure()

dialog.show()
```

- 문제
    - 코드 구조를 변경해서 명시적인 configure 단계를 분리할 수 없을 상황인 경우 문제
    - 모듈 안에 서로 다른 단계가 2개 이상 있으면 객체를 정의하는 부분과 설정하는 부분이 분리되어 있어 코드 가독성 떨어짐

### 2-3. **동적 임포트**

- `import` 문을 함수나 메소드 안에서 사용하는 것
    - 프로그램 모듈 초기화 시점이 아닌 실행되는 동안 모듈 임포트가 발생함
- `dialog` 모듈이 초기화될 때 `app` 을 임포트하는 것 대신 `dialog.show` 함수 실행 시점에 임포트
    - `app` 모듈은 맨 처음 예제와 같으며 `dialog` 를 맨 위에서 임포트하고

```python
# dialog.py
class Dialog:
    ...
    
save_dialog = Dialog()

def show():
    import app  # 동적 임포트
    save_dialog.save_dir = app.prefs.get('save_dir')
    ...
```

```python
# app.py
import dialog

class Prefs:
    ...
    
prefs = Prefs()
dialog.show()
```

- 해당 접근 방법은 임포트, 설정, 실행 단계를 사용하는 방식과 비슷한 효과를 가짐
    - 동적 임포트 방식은 모듈을 정의하고 임포트하는 방식을 구조적으로 변경하지 않아도 됨
    - 순환적인 임포트를 다른 모듈에 접근해야하만 하는 시점으로 지연시킨 것
        - 해당 시점은 모든 다른 모듈이 이미 초기화한 후임
- 동적인 임포트는 피할 수 있으면 피하는 것을 권장
    - `import` 문 비용이 꽤 크며 자주 빠르게 반복되는 루프에서 사용하면 악영향이 커짐
    - 임포트 실행이 미뤄지는 것이므로 실행 시점에 예기치 못한 오류 발생 가능
        - ex. `SyntaxError`
    - 어느 정도 타협점을 찾는 방법이라 생각하면 될듯