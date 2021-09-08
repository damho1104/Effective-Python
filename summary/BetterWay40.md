# 40. super 로 부모 클래스를 초기화하라

## 1. 부모 클래스 초기화

```python
class MyBaseClass:
    def __init__(self, value):
        self.value = value
        
class MyChildClass(MyBaseClass):
    def __init__(self):
        MyBaseClass.__init__(self, 5)
```

- 자식 클래스 생성자에서 부모 클래스의 생성자를 직접 호출함
- 해당 코드는 결함이 존재
    - 다중 상속에 의해 영향을 받는 경우 상위 클래스의 생성자를 직접 호출하면 예측할 수 없는 방식으로 작동 가능함
        - 다이아몬드 상속

```python
class TimesTwo:
    def __init__(self):
        self.value *= 2

class PlusFive:
    def __init__(self):
        self.value += 5

class OneWay(MyBaseClass, TimesTwo, PlusFive):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        TimesTwo.__init__(self)
        PlusFive.__init__(self)

class AnotherWay(MyBaseClass, PlusFive, TimesTwo):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        TimesTwo.__init__(self)
        PlusFive.__init__(self)

foo = OneWay(5)
print('첫 번째 부모 클래스 순서에 따른 값은 (5 * 2) + 5 =', foo.value)

bar = AnotherWay(5)
print('두 번째 부모 클래스 순서에 따른 값은', bar.value)

>>>
첫 번째 부모 클래스 순서에 따른 값은 (5 * 2) + 5 = 15
두 번째 부모 클래스 순서에 따른 값은 15
```

- 부모 클래스의 상속 순서를 다르게 주었을 때 결과
    - 변화 없음
    - 상속 순서와 부모 클래스 생성 순서가 일치하지 않음

```python
class TimesSeven(MyBaseClass):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        self.value *= 7

class PlusNine(MyBaseClass):
    def __init__(self, value):
        MyBaseClass.__init__(self, value)
        self.value += 9

class ThisWay(TimesSeven, PlusNine):
    def __init__(self, value):
        TimesSeven.__init__(self, value)
        PlusNine.__init__(self, value)

foo = ThisWay(5)
print('(5 * 7) + 9 = 44가 나와야 하지만 실제로는', foo.value)

>>>
(5 * 7) + 9 = 44가 나와야 하지만 실제로는 14
```

- 다이아몬드 상속의 경우
    - `TimeSeven` 생성자부터 실행 → `PlusNine` 순서로 실행되어 (5 * 7) + 9 순서로 계산될꺼라 예상
    - 그러나 실제로는 `PlusNine` 생성자 실행 시 `value` 가 5로 초기화됨
    - 그래서 14가 나옴

## 2. `super` 내장 함수

- `super` 내장 함수 통해 부모 클래스 생성자 호출
    - 표준 메소드 결정 순서(Method Resolution OMRO) 적용됨

```python
class TimesSevenCorrect(MyBaseClass):
    def __init__(self, value):
        super().__init__(value)
        self.value *= 7
        
class PlusNineCorrect(MyBaseClass):
    def __init__(self, value):
        super().__init__(value)
        self.value += 9

class GoodWay(TimesSevenCorrect, PlusNineCorrect):
    def __init__(self, value):
        super().__init__(value)

foo = GoodWay(5)
print('7 * (5 + 9) = 98이 나와야 하고 실제로도', foo.value)

>>>
7 * (5 + 9) = 98이 나와야 하고 실제로도 98
```

- 호출 순서는 클래스에 대한 MRO 정의를 따름

```python
mro_str = '\n'.join(repr(cls) for cls in GoodWay.mro())
print(mro_str)

>>>
<class '__main__.GoodWay'>
<class '__main__.TimesSevenCorrect'>
<class '__main__.PlusNineCorrect'>
<class '__main__.MyBaseClass'>
<class 'object'>
```

- 클래스 계층 구조를 따라 root 클래스의 생성자를 찾아들어감
- 그럼 결국 생성자 호출의 역순으로 작업 수행
    - `MyBaseClass` → `PlusNineCorrect` → `TImesSevenCorrect` → `GoodWay`

- `super` 함수에 넘길 수 있는 2가지 파라미터
    - 1번째 파라미터 : MRO 뷰를 제공할 부모 타입
    - 2번째 파라미터 : 1 번째 파라미터로 지정한 타입의 MRO 뷰에 접근할 때 사용할 인스턴스
- object 인스턴스를 초기화할 때 2개 파라미터 지정할 필요 없음
- 파라미터 없이 super 를 호출해도 파이선 인터프리터가 알아서 채워줌

```python
class ExplicitTrisect(MyBaseClass):
    def __init__(self, value):
        super(ExplicitTrisect, self).__init__(value)
        self.value /= 3

```

```python
class AutomaticTrisect(MyBaseClass):
    def __init__(self, value):
        super(__class__, self).__init__(value)
        self.value /= 3

class ImplicitTrisect(MyBaseClass):
    def __init__(self, value):
        super().__init__(value)
        self.value /= 3

assert ExplicitTrisect(9).value == 3
assert AutomaticTrisect(9).value == 3
assert ImplicitTrisect(9).value == 3
```