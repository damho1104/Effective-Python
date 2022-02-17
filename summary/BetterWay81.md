# 81. 프로그램이 메모리를 사용하는 방식과 메모리 누수를 이해하기 위해 tracemalloc을 사용하라

## 1. 메모리 관리

- CPython 은 메모리 관리를 위해 reference counting 사용
    - 객체를 가리키는 참조가 모두 없어지면 참조된 객체도 메모리에서 삭제되고 메모리 공간을 다른 데이터에 내어줄 수 있음
    - cycle detector 존재, 자기 자신을 참조하는 객체의 메모리도 garbage collect 됨
- CPython 런타임이 알아서 메모리 관리를 해주지만 더 이상 사용하지 않는 쓸모없는 참조를 유지하려 하므로 메모리를 수진하게 되는 경우 존재

- 메모리 사용 디버깅하는 방법
    - `gc` 내장 모듈을 사용해 현재 garbage collector 가 알고 있는 모든 객체 나열
    - `gc` 내장 모듈을 사용해 실행 중 생성한 객체 개수와 생성한 객체 중 일부를 출력
    
    ```python
    # waste_memory.py
    import os
    
    class MyObject:
        def __init__(self):
            self.data = os.urandom(100)
            
    def get_data():
        values = []
        for _ in range(100):
            obj = MyObject()
            values.append(obj)
        return values
        
    def run():
        deep_values = []
        for _ in range(100):
            deep_values.append(get_data())
        return deep_values
    ```
    
    ```python
    # using_gc.py
    import gc
    
    found_objects = gc.get_objects()
    print('이전:', len(found_objects))
    
    import waste_memory
    
    hold_reference = waste_memory.run()
    
    found_objects = gc.get_objects()
    print('이후:', len(found_objects))
    
    for obj in found_objects[:3]:
        print(repr(obj)[:100])
    
    >>>
    이전: 6207
    이후: 16801
    <waste_memory.MyObject object at 0x10390aeb8>
    <waste_memory.MyObject object at 0x10390aef0>
    <waste_memory.MyObject object at 0x10390af28>
    ...
    ```
    
    - 문제
        - `gc.get_objects` 는 객체 어떻게 할당됐는지를 알려주지 않음
            - 특정 클래스에 속하는 객체가 여러 다른 방식에 의해 할당될 수 있음
        - 할당된 객체 개수를 아는 것 보단 메모리 누수시키는 객체에 대한 정보를 아는게 중요
        

## 2. `tracemalloc` 내장 모듈

- python 3.4 부터 도입
- 객체를 자신이 할당된 장소와 연결
- 메모리 사용의 이전과 이후 스냅샷을 만들어 비교

```python
# top_n.py
import tracemalloc

tracemalloc.start(10)                     # 스택 깊이 설정
time1 = tracemalloc.take_snapshot()       # 이전 스냅샷

import waste_memory

x = waste_memory.run()                    # 이 부분의 메모리 사용을 디버깅함
time2 = tracemalloc.take_snapshot()       # 이후 스냅샷

stats = time2.compare_to(time1, 'lineno') # 두 스냅샷을 비교

for stat in stats[:3]:
    print(stat)

>>>
waste_memory.py:5: size=2392 KiB (+2392 KiB), count=29994 (+29994), average=82 B
waste_memory.py:10: size=547 KiB (+547 KiB), count=10001 (+10001), average=56 B
waste_memory.py:11: size=82.8 KiB (+82.8 KiB), count=100 (+100), average=848 B
```

- 출력을 통해 크기와 카운트 레이블 확인해보면 프로그램에서 메모리를 주로 사용하는 객체와 이런 객체를 할당한 소스 코드를 명확히 알 수 있음

- `tracemalloc` 모듈은 각 할당의 전체 stacktrace 제공

```python
# with_trace.py
import tracemalloc

tracemalloc.start(10)
time1 = tracemalloc.take_snapshot()

import waste_memory

x = waste_memory.run()
time2 = tracemalloc.take_snapshot()

stats = time2.compare_to(time1, 'traceback')
top = stats[0]
print('가장 많이 사용하는 부분은:')
print('\n'.join(top.traceback.format()))

>>>
가장 많이 사용하는 부분은:
  File "with_trace.py", line 9
    x = waste_memory.run()
  File "waste_memory.py", line 17
    deep_values.append(get_data())
  File "waste_memory.py", line 10
    obj = MyObject()
  File "waste_memory.py", line 5
    self.data = os.urandom(100)
```