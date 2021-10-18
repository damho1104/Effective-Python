# 48. \_\_init_subclass\_\_ 를 사용해 하위 클래스를 검증하라

## 1. 메타클래스 활용

- 클래스 구현 검증
    - 복잡한 클래스 계층을 설계할 때의 요구사항을 구현 단계에서 검증 코드를 수행 가능
    - 검증에 메타클래스를 사용하면 프로그램 시작 시 클래스가 정의된 모듈을 청믕 미포트할 때와 같은 시점에 검증
        - 예외가 훨씬 더 빨리 발생
        
- 메타클래스는 `type` 을 상속하여 정의
    - `__new__` 메소드를 통해 자신과 연관된 클래스의 내용을 받음
- 어떤 타입이 실제로 구성되기 전에 클래스 정보를 살펴보고 변경하는 코드

```python
class Meta(type):
    def __new__(meta, name, bases, class_dict):
        print(f'* 실행: {name}의 메타 {meta}._ _new__')
        print('기반 클래스들:', bases)
        print(class_dict)
        return type.__new__(meta, name, bases, class_dict)

class MyClass(metaclass=Meta):
    stuff = 123

    def foo(self):
        pass

class MySubclass(MyClass):
    other = 567

    def bar(self):
        pass

>>>
* 실행: MyClass의 메타 <class '__main__.Meta'>._ _new__
기반 클래스들: ()
{'__module__': '__main__', '__qualname__': 'MyClass', 'stuff': 123, 'foo': <function MyClass.foo at 0x0000022AD6BDF700>}
* 실행: MySubclass의 메타 <class '__main__.Meta'>._ _new__
기반 클래스들: (<class '__main__.MyClass'>,)
{'__module__': '__main__', '__qualname__': 'MySubclass', 'other': 567, 'bar': <function MySubclass.bar at 0x0000022AD9D288B0>}
```

- 메타클래스는 클래스 이름, 클래스가 상속하는 부모 클래스, 클래스 본문에 정의된 모든 클래스 애트리뷰트에 접근 가능
- 메타클래스가 받는 부모 클래스의 튜플 안에는 object 가 명시적으로 들어 있지 않음
    - 모든 클래스는 object 상속함

- 연관된 클래스가 정의되기 전에 이 클래스의 모든 파라미터를 검증하려면 `Meta.__new__` 에 기능 추가 필요

```python
class ValidatePolygon(type):
    def __new__(meta, name, bases, class_dict):
        # Polygon 클래스의 하위 클래스만 검증한다
        if bases:
            if class_dict['sides'] < 3:
                raise ValueError('다각형 변은 3개 이상이어야 함')
        return type.__new__(meta, name, bases, class_dict)

class Polygon(metaclass=ValidatePolygon):
    sides = None  # 하위 클래스는 이 애트리뷰트에 값을 지정해야 한다
    
    @classmethod
    def interior_angles(cls):
        return (cls.sides - 2) * 180

class Triangle(Polygon):
    sides = 3

class Rectangle(Polygon):
    sides = 4

class Nonagon(Polygon):
    sides = 9

assert Triangle.interior_angles() == 180
assert Rectangle.interior_angles() == 360
```

- `Polygon` 클래스의 하위 클래스만 검증
    - 변 개수가 3보다 작은 경우 class 정의문의 본문이 실행된 직후 예외 발생
        - 변이 2개 이하인 클래스를 정의하면 프로그램이 아예 시작되지도 않는다는 뜻
        - 동적으로 import 되는 모듈에서 이런 클래스를 정의하면 예외 발생 안함 (Better way 88)

```python
print('class 이전')

class Line(Polygon):
    print('sides 이전')
    sides = 2
    print('sides 이후')

print('class 이후')

>>>
class 이전
sides 이전
sides 이후
Traceback ...
ValueError: 다각형 변은 3개 이상이어야 함
```

- **단점**
    - 검증해야 할 조건들이 많아지면 코드의 복잡성 증가
    - 클래스 정의마다 메타클래스를 단 하나만 지정 가능

## 2. `__init_subclass__`

- python 3.6 이상에서 지원

```python
class BetterPolygon:
    sides = None  # 하위 클래스에서 이 애트리뷰트의 값을 지정해야 함

    def __init_subclass__(cls):
        super().__init_subclass__()
        if cls.sides < 3:
            raise ValueError('다각형 변은 3개 이상이어야 함')

    @classmethod
    def interior_angles(cls):
        return (cls.sides - 2) * 180

class Hexagon(BetterPolygon):
    sides = 6

assert Hexagon.interior_angles() == 720

print('class 이전')

class Point(BetterPolygon):
    sides = 1

print('class 이후')

>>>
class 이전
Traceback ...
ValueError: 다각형 변은 3개 이상이어야 함
```

- 코드 간결, `ValidatePolygon` 클래스 삭제되었음
- `side` 애트리뷰트 가져올 때 딕셔너리에서 꺼낼 필요 없어짐

```python
class ValidateFilled(type):
    def __new__(meta, name, bases, class_dict):
        # Filled 클래스의 하위 클래스만 검증한다
        if bases:
            if class_dict['color'] not in ('red', 'green'):
                raise ValueError('지원하지 않는 color 값')
        return type.__new__(meta, name, bases, class_dict)

class Filled(metaclass=ValidateFilled):
    color = None  # 모든 하위 클래스에서 이 애트리뷰트의 값을 지정해야 한다

class RedPentagon(Filled, Polygon):
    color = 'red'
    sides = 5

>>>
Traceback ...
TypeError: metaclass conflict: the metaclass of a derived class must be a (non-strict) subclass of the metaclasses of all its bases
```

- `Polygon` 메타클래스와 `Filled` 메타클래스를 함께 사용하려 시도하면 이해하기 힘든 오류 메시지 발생
- 해결 방안
    - 검증을 여러 단계로 하기 위해 메타클래스 type 정의를 복잡한 계층으로 설계하는 방법 (**단점**)
        
        ```python
        class ValidatePolygon(type):
            def __new__(meta, name, bases, class_dict):
                # 루트 클래스가 아닌 경우만 검증한다
                if not class_dict.get('is_root'):
                    if class_dict['sides'] < 3:
                        raise ValueError('다각형 변은 3개 이상이어야 함')
                return type.__new__(meta, name, bases, class_dict)
        
        class Polygon(metaclass=ValidatePolygon):
            is_root = True
            sides = None  # 하위 클래스에서 이 애트리뷰트 값을 지정해야 한다
        
        class ValidateFilledPolygon(ValidatePolygon):
            def __new__(meta, name, bases, class_dict):
                # 루트 클래스가 아닌 경우만 검증한다
                if not class_dict.get('is_root'):
                    if class_dict['color'] not in ('red', 'green'):
                        raise ValueError('지원하지 않는 color 값')
                return super().__new__(meta, name, bases, class_dict)
        
        class FilledPolygon(Polygon, metaclass=ValidateFilledPolygon):
            is_root = True
            color = None  # 하위 클래스에서 이 애트리뷰트 값을 지정해야 한다
        
        class GreenPentagon(FilledPolygon):
            color = 'green'
            sides = 5
        
        greenie = GreenPentagon()
        assert isinstance(greenie, Polygon)
        
        class OrangePentagon(FilledPolygon):
            color = 'orange'
            sides = 5
        
        >>>
        Traceback ...
        ValueError: 지원하지 않는 color 값
        
        class RedLine(FilledPolygon):
            color = 'red'
            sides = 2
        
        >>>
        Traceback ...
        ValueError: 다각형 변은 3개 이상이어야 함
        ```
        
        - `FilledPolygon` 은 `Polygon`의 인스턴스
        - 검증은 OK
        - 단점
            - Compsability(합성성) 를 해침
            - 색 검증 로직을 다른 클래스 계층 구조에 적용하려면 모든 로직 중복 정의해야 함
    - `__init_subclass__` 메소드 사용
        - `super` 내장 함수를 사용하여 base, sibling 클래스의 `__init_subclass__` 메소드를 호출해주면 여러 단계로 이뤄진 클래스 계층 구조 정의 가능
        
        ```python
        class Filled:
            color = None  # 하위 클래스에서 이 애트리뷰트 값을 지정해야 한다
        
            def __init_subclass__(cls):
                super().__init_subclass__()
                if cls.color not in ('red', 'green', 'blue'):
                    raise ValueError('지원하지 않는 color 값')
        
        class RedTriangle(Filled, Polygon):
            color = 'red'
            sides = 3
            
        ruddy = RedTriangle()
        assert isinstance(ruddy, Filled)
        assert isinstance(ruddy, Polygon)
        
        print('class 이전')
        
        class BlueLine(Filled, Polygon):
            color = 'blue'
            sides = 2
        
        print('class 이후')
        
        >>>
        class 이전
        Traceback ...
        ValueError: 다각형 변은 3개 이상이어야 함
        
        print('class 이전')
        
        class BeigeSquare(Filled, Polygon):
            color = 'beige'
            sides = 4
        
        print('class 이후')
        
        >>>
        class 이전
        Traceback ...
        ValueError: 지원하지 않는 color 값
        ```
        
        - 새로운 클래스에서 `BetterPolygon`, `Filled` 클래스 모두 상속 가능
        - 각 클래스에서 `__init_subclass__` 메소드 호출되므로 각각의 검증 로직 실행 가능
        - 다이아몬드 상속에서도 사용 가능
        
        ```python
        class Top:
            def __init_subclass__(cls):
                super().__init_subclass__()
                print(f'{cls}의 Top')
        
        class Left(Top):
            def __init_subclass__(cls):
                super().__init_subclass__()
                print(f'{cls}의 Left')
        
        class Right(Top):
            def __init_subclass__(cls):
                super().__init_subclass__()
                print(f'{cls}의 Right')
        
        class Bottom(Left, Right):
            def __init_subclass__(cls):
                super().__init_subclass__()
                print(f'{cls}의 Bottom')
        
        >>>
        <class '__main__.Left'>의 Top
        <class '__main__.Right'>의 Top
        <class '__main__.Bottom'>의 Top
        <class '__main__.Bottom'>의 Right
        <class '__main__.Bottom'>의 Left
        ```