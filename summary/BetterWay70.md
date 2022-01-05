# 70. 최적화하기 전에 프로파일링을 하라

## 1. 파이선 성능

- 파이선 동적 특성으로 인해 실행 시간 성능이 예상과 다를 때 있음
    - 느릴 것으로 예상한 연산이 실제로 빠름
        - 문자열 조작
        - 제너레이터
    - 빠를 것으로 예상한 연산이 실제로 느림
        - 애트리뷰트 접근
        - 함수 호출

## 2. 프로파일링 체크

- 프로그램 최적화 전 프로그램 성능 측정 필요
- 문제가 되는 부분을 집중적으로 최적화
- 파이선은 프로파일러 제공
    - profile
        - pure python written
        - side-effect 가 커서 결과 왜곡 가능성 증가
    - **cProfile(권장)**
        - C Language based
        - 프로파일 대상 프로그램의 성능에 최소로 영향을 끼침
- **프로파일링 시 주의할 점**
    - 외부 시스템의 성능이 아니라 코드 자체의 성능을 측정하도록 유의할 것
    - 외부 시스템의 변수를 모두 제외한 python 코드 자체의 성능을 테스트하여 문제점을 찾기 위함

- 삽입 정렬 예제, 느린 이유를 알아보고 싶다고 가정

```python
def insertion_sort(data):
    result = []
    for value in data:
        insert_value(result, value)
    return result
```

```python
def insert_value(array, value):
    for i, existing in enumerate(array):
        if existing > value:
            array.insert(i, value)
            return
    array.append(value)
```

- `insert_sort` 와 `insert_value` 를 프로파일링
- 난수로 데이터 집합 생성 및 프로파일러에 넘길 test 함수 정의
- `cProfile` 모듈에 있는 `Profile` 객체를 인스턴스화하고 인스턴스의 `runcall` 메소드를 사용
- `pstats` 내장 모듈의 `Stats` 클래스를 사용해 성능 통계 추출
- `runcall` 메소드가 실행되면서 프로파일러가 활성화되어 있는 동안에만 샘플링

```python
from random import randint
from cProfile import Profile
from pstats import Stats

max_size = 10**4
data = [randint(0, max_size) for _ in range(max_size)]
test = lambda: insertion_sort(data)

profiler = Profile()
profiler.runcall(test)

stats = Stats(profiler)
stats.strip_dirs()
stats.sort_stats('cumulative')   # 누적 통계
stats.print_stats()

>>>
         20003 function calls in 1.151 seconds
   Ordered by: cumulative time
   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    1.151    1.151 scratch.py:18(<lambda>)
        1    0.002    0.002    1.151    1.151 scratch.py:1(insertion_sort)
    10000    1.141    0.000    1.149    0.000 scratch.py:7(insert_value)
     9991    0.008    0.000    0.008    0.000 {method 'insert' of 'list' objects}
        9    0.000    0.000    0.000    0.000 {method 'append' of 'list' objects}
    ...   
```

- 통계에서 각 열의 의미
    - ncalls
        - 프로파일링 기간 동안 함수가 몇 번 호출했는지
    - tottime
        - 프로파일링 기간 동안 대상 함수를 실행하는 데 걸린 시간의 총합
        - 대상 함수가 다른 함수를 호출한 경우 이 다른 함수를 실행하는 데 걸린 시간은 제외
    - tottime precall
        - 프로파일링 기간 동안 함수가 호출될 때마다 걸린 시간의 평균
        - 대상 함수가 다른 함수를 호출한 경우 이 다른 함수를 실행하기 위해 걸린 시간은 제외
        - = tottime / ncalls
    - cumtime
        - 함수를 실행할 때 걸린 누적 시간
        - 대상 함수가 호출한 다른 함수를 실행하는 데 걸린 시간 포함
    - cumtime precall
        - 프로파일링 기간 동안 함수가 호출될 때마다 걸린 누적 시간의 평균
        - 대상 함수가 호출한 다른 함수를 실행하는 데 걸린 시간 포함
        - = cumtime / ncalls
- 결과를 통해 누적 시간으로 CPU를 가장 많이 사용한 함수는 `insert_value` 함수

- `bisect` 내장 모듈(이진 탐색)을 사용해 다시 구현한 `insert_value` 함수 재프로파일링 결과

```python
from bisect import bisect_left

def insert_value(array, value):
    i = bisect_left(array, value)
    array.insert(i, value)

...

>>>
         30003 function calls in 0.014 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.014    0.014 scratch.py:17(<lambda>)
        1    0.001    0.001    0.014    0.014 scratch.py:1(insertion_ort)
    10000    0.003    0.000    0.012    0.000 scratch.py:9(insert_value)
    10000    0.007    0.000    0.007    0.000 {method 'insert' of 'list' objects}
    10000    0.003    0.000    0.003    0.000 {built-in method _bisect.bisect_left

    ...
```

- 이전 `insert_value` 함수에 비해 새로 정의한 함수의 누적 실행 시간이 100배 가까이 줄어들었음을 확인 가능

- 전체 프로그램 프로파일링 시 공통 유틸리티 함수가 대부분 실행 시간을 차지한다는 사실 발견하는 예제
    - 프로파일링 결과를 제대로 이해하기 어려움

```python
def my_utility(a, b):
    c = 1
    for i in range(100):
    c += a * b
    
def first_func():
    for _ in range(1000):
        my_utility(4, 5)
        
def second_func():
    for _ in range(10):
        my_utility(1, 3)
        
def my_program():
    for _ in range(20):
        first_func()
        second_func()

...

>>>
         20242 function calls in 0.100 seconds

   Ordered by: cumulative time

   ncalls  tottime  percall  cumtime  percall filename:lineno(function)
        1    0.000    0.000    0.100    0.100 scratch.py:14(my_program)
       20    0.003    0.000    0.099    0.005 scratch.py:6(first_func)
    20200    0.097    0.000    0.097    0.000 scratch.py:1(my_utility)
       20    0.000    0.000    0.001    0.000 scratch.py:10(second_func)
     ...
```

- `print_stats` 디폴트 옵션으로 통계 출력 결과
- `my_utility` 함수가 실행 시간의 대부분을 차지하는 것은 확실
    - 이 함수가 왜 많이 호출되었는지 즉시 확인 어려움

- `print_callers` 메소드를 통해 caller-callee 관계를 통한 호출 횟수 확인 가능

```python
...
stats.print_callers()
...

>>>
   Ordered by: cumulative time

Function                            was called by...
                                        ncalls  tottime  cumtime
scratch.py:14(my_program)           <- 
scratch.py:6(first_func)            <-      20    0.003    0.098 scatch.py:14(my_program)
scratch.py:1(my_utility)            <-   20000    0.095    0.095 scatch.py:6(first_func)
                                           200    0.001    0.001 scatch.py:10(second_func)
scratch.py:10(second_func)          <-      20    0.000    0.001 scratch.py:14(my_program)
```

- 왼쪽에 호출된 함수(callee), 오른쪽에 함수를 호출한 함수(caller)