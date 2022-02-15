# 80. pdb 를 사용해 대화형으로 디버깅하라

## 1. 버그 찾기

- 출력을 통해 버그 찾기
- 테스트를 작성하여 버그 찾기
- 파이선 내장 대화형 디버거를 사용하여 버그 찾기

## 2. 파이선 내장 대화형 디버거 pdb

- 문제를 조사하기에 가장 적합한 지점에서 프로그램이 디버거를 초기화할 수 있게 프로그램 변경
    - `breakpoint` 내장 함수 호출
        - `pdb` 내장 모듈 임포트
        - `set_trace` 함수 실행
- breakpoint 함수 실행되자마자, 다음 줄로 실행이 옮겨지기 전에 프로그램 실행이 일시 중단
- 대화형 파이선 쉘 실행

```python
# always_breakpoint.py
import math

def compute_rmse(observed, ideal):
    total_err_2 = 0
    count = 0
    for got, wanted in zip(observed, ideal):
        err_2 = (got - wanted) ** 2
        breakpoint()  # 여기서 디버거를 시작함
        total_err_2 += err_2
        count += 1
        
    mean_err = total_err_2 / count
    rmse = math.sqrt(mean_err)
    return rmse
    
result = compute_rmse(
    [1.8, 1.7, 3.2, 6],
    [2, 1.5, 3, 5])
print(result)
```

```python
$ python3 always_breakpoint.py
> always_breakpoint.py(12)compute_rmse()
-> total_err_2 += err_2
(Pdb)
```

- `(Pdb)` 프롬프트에서 `p` 명령을 통해 지역 변수 이름을 입력하면 변수에 저장된 값 출력
- `locals` 내장 함수 호출하면 모든 지역 변수 목록을 볼 수 있음
- `(Pdb)` 프롬프트에서 모듈 임포트, 새 객체 생성, `help` 내장함수 실행, 프로그램 일부 변경 가능
- `Pdb` 명령
    - `where`
        - 현재 실행 중인 프로그램의 호출 스택 출력
        - 프로그램의 현재 위치 파악 가능, `breakpoint` 트리거 발동 위치 파악 가능
    - `up`, `u`
        - 실행 호출 스택에서 현재 관찰 중인 함수를 호출한 쪽으로 호출 스택 영역을 이동해서 해당 함수의 지역 변수 관찰 가능
    - `down`, `d`
        - 실행 호출 스택에서 한 수준 아래로 호출 스택 영역 이동
    - `step`
        - 다음 줄까지 실행한 다음 제어를 디버거로 돌려서 디버거 프롬프트 표시
        - 소스 코드 다음 줄에 함수를 호출하는 부분이 있다면 해당 함수의 첫 줄에서 디버거로 제어가 돌아옴
    - `next`
        - `step` 과 유사, 차이점은 함수 호출 부분에서 해당 함수 반환 후 제어가 디버거로 돌아옴
    - `return`
        - 현재 함수에서 반환될 때까지 프로그램을 실행 한 후 제어가 디버거로 돌아옴
    - `continue`
        - 다음 중단점에 도달할 때까지 프로그램을 계속 실행
            - breakpoint 호출이나 디버거에서 설정한 중단점에서 제어가 디버거에게 돌아옴
        - 실행하는 중에 중단점을 만나지 못하면 프로그램 계속 실행
    - `quit`
        - 디버거에서 나가면서 프로그램 중단
    
- `breakpoint` 함수는 프로그램 어디서든 호출 가능

```python
# conditional_breakpoint.py
def compute_rmse(observed, ideal):
    ...
    for got, wanted in zip(observed, ideal):
        err_2 = (got - wanted) ** 2
        if err_2 >= 1:  # True인 경우에만 디버거를 실행함
            breakpoint()
        total_err_2 += err_2
        count += 1
    ...
result = compute_rmse(
    [1.8, 1.7, 3.2, 7],
    [2, 1.5, 3, 5])
print(result)
```

```python
$ python3 conditional_breakpoint.py
> conditional_breakpoint.py(14)compute_rmse()
-> total_err_2 += err_2
(Pdb) wanted
5
(Pdb) got
7
(Pdb) err_2
4
```

- 사후 디버깅(post-mortem debugging)
    - 예외 발생이나 프로그램 비정상 종류 이후 디버깅 가능
- `python3 -m pdb -c continue [프로그램 경로]`
    - `pdb` 모듈이 프로그램 실행을 제어하게 할 수 있음
    - `continue` 명령은 `pdb`에게 프로그램을 즉시 시작하라고 명령
    - 프로그램 실행 후 문제가 발생하면 대화형 디버거로 제어 이전 그 시점부터 코드 상태 관찰 가능

```python
# postmortem_breakpoint.py
import math

# 평균 제곱근 오차(root# mean square error)를 구함
def compute_rmse(observed, ideal):
   ...
   
result = compute_rmse(
   [1.8, 1.7, 3.2, 7j],  # 잘못된 입력
   [2, 1.5, 3, 5])
print(result)
```

```python
$ python3 -m pdb -c continue postmortem_breakpoint.py
Traceback (most recent call last):
  File ".../pdb.py", line 1697, in main
    pdb._runscript(mainpyfile)
  File ".../pdb.py", line 1566, in _runscript
    self.run(statement)
  File ".../bdb.py", line 585, in run
    exec(cmd, globals, locals)
  File "<string>", line 1, in <module>
  File "postmortem_breakpoint.py", line 4, in <module>
    import math
  File "postmortem_breakpoint.py", line 16, in compute_rmse
    rmse = math.sqrt(mean_err)
TypeError: can't convert complex to float
Uncaught exception. Entering post mortem debugging
Running 'cont' or 'step' will restart the program
> postmortem_breakpoint.py(16)compute_rmse()
-> rmse = math.sqrt(mean_err)
(Pdb) mean_err
(-5.97-17.5j)
```

- 프로그램을 실행하는 중에 예외가 발생했는데 처리되지 않은 경우
    - `pdb` 모듈의 `pm` 함수를 호출하면 사후 디버깅 사용 가능

```python
$ python3
>>> import my_module
>>> my_module.compute_stddev([5])
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "my_module.py", line 17, in compute_stddev
    variance = compute_variance(data)
  File "my_module.py", line 13, in compute_variance
    variance = err_2_sum / (len(data) - 1)
ZeroDivisionError: float division by zero
>>> import pdb; pdb.pm()
> my_module.py(13)compute_variance()
-> variance = err_2_sum / (len(data) - 1)
(Pdb) err_2_sum
0.0
(Pdb) len(data)
1
```