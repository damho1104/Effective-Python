# 46. 재사용 가능한 @property 메소드를 만들려면 디스크립터를 사용하라

## 1. `@property` 내장 기능의 단점

- 재사용성 떨어짐
- `@property`가 데코레이션하는 메소드를 같은 클래스에 속하는 여러 애트리뷰트로 사용 불가

```python
class Homework:
    def __init__(self):
        self._grade = 0

    @property
    def grade(self):
        return self._grade

    @grade.setter
    def grade(self, value):
        if not (0 <= value <= 100):
            raise ValueError('점수는 0과 100 사이입니다')
        self._grade = value

galileo = Homework()
galileo.grade = 95
```

```python
class Exam:
    def __init__(self):
        self._writing_grade = 0
        self._math_grade = 0

    @staticmethod
    def _check_grade(value):
        if not (0 <= value <= 100):
            raise ValueError('점수는 0과 100 사이입니다')

    @property
    def writing_grade(self):
        return self._writing_grade
        
    @writing_grade.setter
    def writing_grade(self, value):
        self._check_grade(value)
        self._writing_grade = value
        
    @property
    def math_grade(self):
        return self._math_grade
        
    @math_grade.setter
    def math_grade(self, value):
        self._check_grade(value)
        self._math_grade = value
```

- `Exam` 클래스와 같이 계속 확장하다보면 시험 과목을 이루는 각 부분마다 새로운 `@property` 를 지정하고 관련 검증 메소드를 작성해야 함
- 숙제나 시험 성적 이외의 부분에 백분율 검증을 하고 싶은 경우 똑같이 `검증대상_grade` 에 대한 setter 메소드를 다시 만들어야 함

## 2. 디스크립터

- 디스크립터 프로토콜은 파이선 언어에서 애트리뷰트 접근을 해석하는 방법을 정의
- 디스크립터 클래스는 `__get__`, `__set__` 메소드 제공
- 같은 로직을 한 클래스 안에 속한 여러 다른 애트리뷰트에 적용 가능
- 디스크립터가 [믹스인](https://www.notion.so/41-2b0d4463ce504bc5b11163965794e4c5) 보다 나음

```python
class Grade:
    def __get__(self, instance, instance_type):
        ...
    def __set__(self, instance, value):
        ...

class Exam:
    # 클래스 애트리뷰트
    math_grade = Grade()
    writing_grade = Grade()
    science_grade = Grade()

exam = Exam()
exam.writing_grade = 40
```

- `exam.writing_grade` 에 40 을 대입하게 되면 다음과 같이 해석됨

```python
Exam.__dict__['writing_grade'].__set__(exam, 40)
```

- `exam.writing_grade` 를 읽게되면 다음과 같이 해석됨

```python
Exam.__dict__['writing_grade'].__get__(exam, Exam)
```

- 해당 동작은 object 의 `__getattribute__` 메소드
    - `Exam` 인스턴스에 `writing_grade` 이름의 애트리뷰트가 없으면 python 에서 `Exam` 클래스의 애트리뷰트를 대신 사용
    - 클래스의 애트리뷰트가 `__get__`, `__set__` 메소드가 정의된 객체라면 python 은 디스크립터 프로토콜을 따르도록 결정함

- 현재 구현은 모든 `Exam` 인스턴스가 애트리뷰트를 공유하여 관리에 불편, 아래와 같이 수정

```python
class Grade:
    def __init__(self):
        self._values = {}

    def __get__(self, instance, instance_type):
        if instance is None:
            return self
        return self._values.get(instance, 0)

    def __set__(self, instance, value):
        if not (0 <= value <= 100):
            raise ValueError('점수는 0과 100 사이입니다')
        self._values[instance] = value
```

- `__set__` 메소드에서 프로그램이 실행되는 동안 모든 `Exam` 인스턴스에 대한 참조를 `_value` 가 저장
- **문제**
    - 인스턴스에 대한 참조 카운터가 절대로 0이 될 수 없음
    - garbage collector가 인스턴스 메모리를 재활용하지 못함
    - 누수 발생

- **해결방안**
    - `weakref` 내장 모듈 사용
    - `WeakKeyDictionary` 클래스 제공, `_value` 에 사용하도록 수정
        - 딕셔너리에 객체를 저장할 때 약한 참조(weak reference)를 사용함
        - garbage collector 는 약한 참조로만 참조된 객체가 사용 중인 메모리를 재활용할 수 있음
        - `WeakKeyDictionary` 에 저장된 `Exam` 인스턴스가 더 이상 쓰이지 않으면 garbage collector 가 해당 메모리 재활용 가능

```python
from weakref import WeakKeyDictionary

class Grade:
    def __init__(self):
        self._values = WeakKeyDictionary()
        
    def __get__(self, instance, instance_type):
        ...
                
    def __set__(self, instance, value):
        ...

class Exam:
    math_grade = Grade()
    writing_grade = Grade()
    science_grade = Grade()
    
first_exam = Exam()
first_exam.writing_grade = 82
second_exam = Exam()
second_exam.writing_grade = 75
print(f'첫 번째 쓰기 점수 {first_exam.writing_grade} 맞음')
print(f'두 번째 쓰기 점수 {second_exam.writing_grade} 맞음')

>>>
첫 번째 쓰기 점수 82 맞음
두 번째 쓰기 점수 75 맞음
```