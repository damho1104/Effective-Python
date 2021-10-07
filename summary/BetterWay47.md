# 47. 지연 계산 애트리뷰트가 필요하면 \_\_getattr\_\_, \_\_getattribute\_\_, \_\_setattr\_\_ 을 사용하라

## 1. object 훅

- object 훅을 사용하면 시스템을 서로 접합하는 제너릭 코드를 쉽게 작성 가능
    - ex. 데이터베이스 레코드를 파이선 객체로 표현 가능
    

## 2. `__getattr__` 훅

- 데이터베이스 스키마를 표현하는 클래스는 `__getattr__` 메소드를 통해 표현 가능
    - 이 객체의 인스턴스 딕셔너리에서 찾을 수 없는 애트리뷰트에 접근할 때마다 `__getattr__` 호출됨

```python
class LazyRecord:
    def __init__(self):
        self.exists = 5

    def __getattr__(self, name):
        value = f'{name}를 위한 값'
        setattr(self, name, value)
        return value

data = LazyRecord()
print('이전:', data.__dict__)
print('foo: ', data.foo)
print('이후:', data.__dict__)

>>>
이전: {'exists': 5}
foo:  foo를 위한 값
이후: {'exists': 5, 'foo': 'foo를 위한 값'}
```

- 2번째 `print` 에서 `data.foo` 를 통해 `LazyRecord` 클래스에 없는 애트리뷰트 `foo` 를 사용
- 이때 `LazyRecord` 클래스에 있는 `__getattr__` 메소드가 호출됨
- 이후 `foo` 애트리뷰트를 사용하려 하면 이땐 앞서서 `foo` 애트리뷰트가 생성되었으므로 `__getattr__` 를 다시 호출하지 않고 생성된 `foo` 를 사용

```python
class LoggingLazyRecord(LazyRecord):
    def __getattr__(self, name):
        print(f'* 호출: __getattr__({name!r}), '
              f'인스턴스 딕셔너리 채워 넣음')
        result = super().__getattr__(name)
        print(f'* 반환: {result!r}')
        return result

data = LoggingLazyRecord()
print('exists:', data.exists)
print('첫 번째 foo:', data.foo)
print('두 번째 foo:', data.foo)

>>>
exists: 5
* 호출: __getattr__('foo'), 인스턴스 딕셔너리 채워 넣음
* 반환: 'foo를 위한 값'
첫 번째 foo: foo를 위한 값
두 번째 foo: foo를 위한 값
```

- 이런 기능은 **스키마가 없는 데이터에 지연 계산으로 접근하는 것**과 같은 활용이 필요할 때 유용
    - lazy binding 가능

## 3. `__getattribute__` 훅

- 데이터베이스 시스템 안에서 트랜잭션이 필요한 상황 가정
    - 사용자가 프로퍼티에 접근할 때 상응하는 데이터베이스에 있는 레코드가 유효한지, 트랜잭션이 열려 있는지 판단 필요
- `__getattr__` 은 애트리뷰트가 생성된 이후 호출되지 않으므로 부적합
- 이런 경우 `__getattribute__` 훅 사용
    - 객체의 애트리뷰트에 접근할 때마다 호출
    - 애트리뷰트 딕셔너리에 존재하는 애트리뷰트에 접근할 때도 이 훅이 호출됨
    - 프로퍼티에 접근할 때마다 항상 전역 트랜잭션 상태를 검사하는 작업에 적용 가능

```python
class ValidatingRecord:
    def __init__(self):
        self.exists = 5

    def __getattribute__(self, name):
        print(f'* 호출: __getattr__({name!r})')
        try:
            value = super().__getattribute__(name)
            print(f'* {name!r} 찾음, {value!r} 반환')
            return value
        except AttributeError:
            value = f'{name}를 위한 값'
            print(f'* {name!r}를 {value!r}로 설정')
            setattr(self, name, value)
            return value

data = ValidatingRecord()
print('exists:', data.exists)
print('첫 번째 foo:', data.foo)
print('두 번째 foo:', data.foo)

>>>
* 호출: __getattr__('exists')
* 'exists' 찾음, 5 반환
exists: 5
* 호출: __getattr__('foo')
* 'foo'를 'foo를 위한 값'로 설정
첫 번째 foo: foo를 위한 값
* 호출: __getattr__('foo')
* 'foo' 찾음, 'foo를 위한 값' 반환
두 번째 foo: foo를 위한 값
```

- 해당 예제를 통해 `ValidatingRecord` 클래스의 애트리뷰트에 접근할 때마다 `__getattribute__` 메소드가 호출되는 것 확인 가능
- 존재하지 않는 애트리뷰트에 접근하는 경우 `AttributeError` 예외 발생
    - `__getattr__` 에도 동일 상황에서 동일 예외 발생

```python
class MissingPropertyRecord:
    def __getattr__(self, name):
        if name == 'bad_name':
            raise AttributeError(f'{name}을 찾을 수 없음')
        ...

data = MissingPropertyRecord()
data.bad_name

>>>
Traceback ...
AttributeError: bad_name을 찾을 수 없음
```

- `hasattr`, `getattr` 내장 함수를 통해 프로퍼티 존재 여부, 프로퍼티 값을 꺼내오는 작업을 수행할 수 있는데 이 두 함수도 `__getattr__` 를 호출하기 전에 애트리뷰트 이름을 인스턴스 딕셔너리에서 검색
    - `hasattr`, `getattr` 내장 함수를 호출하면 없던 애트리뷰트라도 `__getattr__` 로 인해 생성되는 것 같음

```python
data = LoggingLazyRecord() # __getattr__을 구현
print('이전:', data.__dict__)
print('최초에 foo가 있나:', hasattr(data, 'foo'))
print('이후:', data.__dict__)
print('다음에 foo가 있나:', hasattr(data, 'foo'))

>>>
이전: {'exists': 5}
* 호출: __getattr__('foo'), 인스턴스 딕셔너리 채워 넣음
* 반환: 'foo를 위한 값'
최초에 foo가 있나: True
이후: {'exists': 5, 'foo': 'foo를 위한 값'}
다음에 foo가 있나: True
```

```python
data = ValidatingRecord() # __getattribute__를 구현
print('최초에 foo가 있나:', hasattr(data, 'foo'))
print('다음에 foo가 있나:', hasattr(data, 'foo'))

>>>
* 호출: __getattr__('foo')
* 'foo'를 'foo를 위한 값'로 설정
최초에 foo가 있나: True
* 호출: __getattr__('foo')
* 'foo' 찾음, 'foo를 위한 값' 반환
다음에 foo가 있나: True
```

## 4. `__setattr__` 훅

- 파이선 객체에 값이 대입된 경우 이후 이 값을 데이터베이스에 다시 저장하고 싶다고 가정
- 이때 `__setattr__` 을 사용해 구현 가능
    - 인스턴스의 애트리뷰트에 대입이 이뤄질 때마다 항상 호출

```python
class SavingRecord:
    def __setattr__(self, name, value):
        # 데이터를 데이터베이스 레코드에 저장한다
        ...
        super().__setattr__(name, value)

class LoggingSavingRecord(SavingRecord):
    def __setattr__(self, name, value):
        print(f'* 호출: __setattr__({name!r}, {value!r})')
        super().__setattr__(name, value)

data = LoggingSavingRecord()
print('이전:', data.__dict__)
data.foo = 5
print('이후:', data.__dict__)
data.foo = 7
print('최후:', data.__dict__)

>>>
이전: {}
* 호출: __setattr__('foo', 5)
이후: {'foo': 5}
* 호출: __setattr__('foo', 7)
최후: {'foo': 7}
```

## 5. `__getattribute__`, `__setattr__` 문제점

- 원하든 원하지 않든 어떤 객체의 모든 애트리뷰트에 접근할 때마다 함수가 호출된다는 것

```python
class BrokenDictionaryRecord:
    def __init__(self, data):
        self._data = {}

    def __getattribute__(self, name):
        print(f'* 호출: __getattribute__({name!r})')
        return self._data[name]

data = Brokedata = BrokenDictionaryRecord({'foo': 3})
data.foo

>>>
* 호출: __getattribute__('foo')
* 호출: __getattribute__('_data')
* 호출: __getattribute__('_data')
* 호출: __getattribute__('_data')
...
Traceback ...                   
RecursionError: maximum recursion depth exceeded while calling a Python object
```

- 다음 코드는 재귀가 일어나다 스택을 다 소모해 죽음
    - `__getattribute__` 메소드에서 애트리뷰트에 다시 접근하기 때문에 `__getattribute__` 메소드가 다시 호출됨
- 해결 방안
    - `super().__getattribute__` 를 호출해 인스턴스 애트리뷰트 딕셔너리에서 값을 가져오도록 수정
    - `__setattr__` 메서드에서 애트리뷰트를 변경하는 경우에도 `super().__setattr__` 을 적절히 호출

```python
class DictionaryRecord:
    def __init__(self, data):
        self._data = data

    def __getattribute__(self, name):
        print(f'* 호출: __getattribute__({name!r})')
        data_dict = super().__getattribute__('_data')
        return data_dict[name]

data = DictionaryRecord({'foo': 3})
print('foo:', data.foo)

>>>
* 호출: __getattribute__('foo')
foo: 3

```