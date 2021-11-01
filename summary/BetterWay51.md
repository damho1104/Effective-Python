# 51. 합성 가능한 클래스 확장이 필요하면 메타클래스보다는 클래스 데코레이터를 사용하라

## 1. 데코레이터

- 어떤 클래스의 모든 메소드를 감싸서 메소드에 전달되는 인자, 반환 값, 예외를 모두 출력
    
    ```python
    from functools import wraps
    
    def trace_func(func):
        if hasattr(func, 'tracing'):  # 단 한 번만 데코레이터를 적용한다
            return func
    
        @wraps(func)
        def wrapper(*args, **kwargs):
            result = None
            try:
                result = func(*args, **kwargs)
                return result
            except Exception as e:
                result = e
                raise
            finally:
                print(f'{func._ _name__}({args!r}, {kwargs!r}) -> '
                      f'{result!r}')
    
        wrapper.tracing = True
        return wrapper
    
    class TraceDict(dict):
        @trace_func
        def __init__(self, *args, **kwargs):
            super().__init__(*args, **kwargs)
            
        @trace_func
        def __setitem__(self, *args, **kwargs):
            return super().__setitem__(*args, **kwargs)
            
        @trace_func
        def __getitem__(self, *args, **kwargs):
            return super().__getitem__(*args, **kwargs)
            
        ...
    
    trace_dict = TraceDict([('안녕', 1)])
    trace_dict['거기'] = 2
    trace_dict['안녕']
    try:
        trace_dict['존재하지 않음']
    except KeyError:
        pass # 키 오류가 발생할 것으로 예상함
    
    >>>
    __init__(({'안녕': 1}, [('안녕', 1)]), {}) -> None
    __setitem__(({'안녕': 1, '거기': 2}, '거기', 2), {}) -> None
    __getitem__(({'안녕': 1, '거기': 2}, '안녕'), {}) -> 1
    __getitem__(({'안녕': 1, '거기': 2}, '존재하지 않음'), {}) -> KeyError('존재하지 않음')
    ```
    
- **단점**
    - 해당 데코레이터를 사용해야 하는 모든 메소드에 `@trace_func` 데코레이터를 써서 재정의해야 함
        - 불필요한 중복으로 가독성 떨어짐
        - 실수 가능성 존재
    - `dict` 상위 클래스에 메소드를 추가하면 `TraceDict` 에서 해당 메소드를 재정의하기 전까지는 데코레이터 적용이 되지 않음
    

## 2. 메타클래스 활용

- 메타클래스를 사용해 클래스에 속한 모든 메소드를 자동으로 감싸는 것

```python
import types

trace_types = (
    types.MethodType,
    types.FunctionType,
    types.BuiltinFunctionType,
    types.BuiltinMethodType,
    types.MethodDescriptorType,
    types.ClassMethodDescriptorType)
    
class TraceMeta(type):
    def __new__(meta, name, bases, class_dict):
        klass = super().__new__(meta, name, bases, class_dict)
        
        for key in dir(klass):
            value = getattr(klass, key)
            if isinstance(value, trace_types):
                wrapped = trace_func(value)
                setattr(klass, key, wrapped)
                
        return klass

class TraceDict(dict, metaclass=TraceMeta):
    pass
trace_dict = TraceDict([('안녕', 1)])
trace_dict['거기'] = 2
trace_dict['안녕']
try:
    trace_dict['존재하지 않음']
except KeyError:
    pass  # 키 오류가 발생할 것으로 예상함

>>>
__new__((<class '__main__.TraceDict'>, [('안녕', 1)]), {}) -> {}
__getitem__(({'안녕': 1, '거기': 2}, '안녕'), {}) -> 1
__getitem__(({'안녕': 1, '거기': 2}, '존재하지 않음'), {}) -> KeyError('존재하지 않음')
```

- 이전 예제에서 재정의를 누락한 `__new__` 호출 또한 잘 적용됨
- **문제**
    - 만일 상위 클래스에서 이미 메타클래스를 사용하였을 경우엔 어떤 일이 발생할까??

```python
class OtherMeta(type):
    pass
    
class SimpleDict(dict, metaclass=OtherMeta):
    pass
    
class TraceDict(SimpleDict, metaclass=TraceMeta):
    pass

>>>
Traceback ...   
TypeError: metaclass conflict: the metaclass of a derived class must be a (non-strict) subclass of the metaclasses of all its bases
```

- `TraceMeta` 를 `OtherMeta` 가 상속하지 않아서 에러 발생
- 메타클래스 상속을 활용하여 TraceMeta 를 OtherMeta 가 상속하도록 하면 해결 가능
    - 제약 사항이 많음
        - 라이브러리에 있는 메타클래스를 사용하는 경우는 불가
        - `TraceMeta` 같은 유틸리티 메타클래스를 여럿 사용하고 싶은 경우에도 사용 불가
        

## 3. 클래스 데코레이터

- 클래스 데코레이터 예제
    - 클래스 선언 앞에 @ 기호와 데코레이터 함수 사용
    - 인자로 받은 클래스를 적절히 변경해서 재생성

```python
def my_class_decorator(klass):
    klass.extra_param = '안녕'
    return klass

@my_class_decorator
class MyClass:
    pass

print(MyClass)
print(MyClass.extra_param)

>>>
<class '__main__.MyClass'>
안녕
```

- `TraceMeta.__new__` 메소드의 핵심 부분만을 별도의 함수로 옮겨서 어떤 클래스에 속한 모든 메소드와 함수에 `trace_func` 를 적용하는 클래스 데코레이터

```python
def trace(klass):
    for key in dir(klass):
        value = getattr(klass, key)
        if isinstance(value, trace_types):
            wrapped = trace_func(value)
            setattr(klass, key, wrapped)
    return klass

@trace
class TraceDict(dict):
    pass

trace_dict = TraceDict([('안녕', 1)])
trace_dict['거기'] = 2
trace_dict['안녕']
try:
    trace_dict['존재하지 않음']
except KeyError:
    pass # 키 오류가 발생할 것으로 예상함

>>>
__new__((<class '__main__.TraceDict'>, [('안녕', 1)]), {}) -> {}
__getitem__(({'안녕': 1, '거기': 2}, '안녕'), {}) -> 1
__getitem__(({'안녕': 1, '거기': 2}, '존재하지 않음'), {}) -> KeyError('존재하지 않음')
```

- 데코레이션을 적용할 클래스에 이미 메타클래스가 있어서 데코레이터 적용 가능

```python
class OtherMeta(type):
    pass

@trace
class TraceDict(dict, metaclass=OtherMeta):
    pass

trace_dict = TraceDict([('안녕', 1)])
trace_dict['거기'] = 2
trace_dict['안녕']
try:
    trace_dict['존재하지 않음']
except KeyError:
    pass  # 키 오류가 발생할 것으로 예상함

>>>
__new__((<class '__main__.TraceDict'>, [('안녕', 1)]), {}) -> {}
__getitem__(({'안녕': 1, '거기': 2}, '안녕'), {}) -> 1
__getitem__(({'안녕': 1, '거기': 2}, '존재하지 않음'), {}) -> KeyError('존재하지 않음')
```