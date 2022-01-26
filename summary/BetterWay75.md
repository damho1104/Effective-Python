# 75. 디버깅 출력에는 repr 문자열을 사용하라

## 1. 파이선 출력

- format string 을 사용하거나 `logging` 내장 모듈을 사용하여 출력을 만들 수 있음
    - 아주 긴 출력이 생길 수 있음
- 프로그램이 실행되는 동안 `print` 메소드를 호출하여 상태가 어떻게 바뀌는지 알아내고 무엇이 잘못되었는지 확인 가능
- `print` 메소드에 다양한 방법으로 출력 가능

```python
my_value = 'foo 뭐시기'
print(str(my_value))
print('%s' % my_value)
print(f'{my_value}')
print(format(my_value))
print(my_value._ _format__('s'))
print(my_value.__str__())

>>>
foo 뭐시기
foo 뭐시기
foo 뭐시기
foo 뭐시기
foo 뭐시기
foo 뭐시기
```

- **단점**
    - 어떤 값을 사람이 읽을 수 있는 형식의 문자열로 바꿔도 이 값의 실제 타입과 구체적인 구성을 명확히 알기 어려움
        
        ```python
        print(5)
        print('5')
        
        int_value = 5
        str_value = '5'
        print(f'{int_value} == {str_value} ?')
        
        >>>
        5
        5
        5 == 5 ?
        ```
        
    

## 2. `repr` 내장 함수

- 객체의 출력 가능한 표현을 반환
- 출력 가능한 표현은 반드시 객체를 가장 명확하게 이해할 수 있는 문자열 표현

```python
a = '\x07'
print(repr(a))

>>>
'\x07’
```

- repr 이 돌려준 값을 eval 내장 함수에 넘기면 repr에 넘겼던 객체와 같은 객체가 생겨야 함
    - 실전에서 eval 호출은 조심

```python
b = eval(repr(a))
assert a == b
```

```python
print(repr(5))
print(repr('5'))

>>>
5
'5’
```

- `repr` 호출은 % 연산자에 %r format string 사용 혹은 f-문자열에 !r 타입 변환을 사용하는 것과 같음

```python
print('%r' % 5)
print('%r' % '5')

int_value = 5
str_value = '5'
print(f'{int_value!r} != {str_value!r}')

>>>
5
'5'
5 != '5'
```

- 파이선 클래스의 경우 사람이 읽을 수 있는 문자열 값은 `repr` 값과 같음
    - 인스턴스를 `print` 에 넘기면 원하는 출력이 나오기 때문에 `repr` 호출할 필요가 없음
    - `eval` 함수에 사용 불가, 객체 인스턴스 필드 정보 없음

```python
class OpaqueClass:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        
obj = OpaqueClass(1, 'foo')
print(obj)

>>>
<_ _main__.OpaqueClass object at 0x10963d6d0>
```

- 방법 1. `__repr__` 특별 메소드 사용(클래스 정의 수정 가능할 경우)
    - 객체를 다시 만들어내는 파이선 식을 포함하는 문자열을 반환하도록 정의

```python
class BetterClass:
    def __init__(self, x, y):
        self.x = x
        self.y = y
        
    def __repr__(self):
        return f'BetterClass({self.x!r}, {self.y!r})'
```

```python
obj = BetterClass(2, '뭐시기')
print(obj)

>>>
BetterClass(2, '뭐시기')
```

- 방법 2. `__dict__` 애트리뷰트를 통해 객체의 인스턴스 딕셔너리 접근(클래스 정의 수정 불가능할 경우)

```python
obj = OpaqueClass(4, 'baz')
print(obj.__dict__)

>>>
{'x': 4, 'y': 'baz'}
```