# 35. 제너레이터 안에서 throw 로 상태를 변화시키지 말라

## 1. 제너레이터 `throw`

- 제너레이터에 대해 `throw` 호출되면 제너레이터는 `throw` 가 제공한 `Exception`을 다시 던짐

```python
class MyError(Exception):
    pass
    
def my_generator():
    yield 1
    yield 2
    yield 3

it = my_generator()
print(next(it))  # 1을 내놓음
print(next(it))  # 2를 내놓음
print(it.throw(MyError('test error')))

>>>
1
2
Traceback ...
MyError: test error
```

```python
def my_generator():
    yield 1
    try:
        yield 2

    except MyError:
        print('MyError 발생!')

    else:
        yield 3
    yield 4

it = my_generator()
print(next(it))  # 1을 내놓음
print(next(it))  # 2를 내놓음
print(it.throw(MyError('test error')))

>>>
1
2
MyError 발생!
4
```

## 2. 제너레이터 `throw` 를 사용한 예제

```python
class Reset(Exception):
    pass
    
def timer(period):
    current = period
    while current:
        current -= 1
        try:
            yield current
        except Reset:
            current = period

def check_for_reset():
    # 외부 이벤트를 폴링한다
    ...

def announce(remaining):
    print(f'{remaining} 틱 남음')

def run():
    it = timer(4)
    while True:
        try:
            if check_for_reset():
                current = it.throw(Reset())
            else:
                current = next(it)
        except StopIteration:
            break
        else:
            announce(current)

run()

>>>
3 틱 남음
2 틱 남음
1 틱 남음
3 틱 남음
2 틱 남음
3 틱 남음
2 틱 남음
1 틱 남음
0 틱 남음
```

- `throw` 에 의존하는 제너레이터를 통해 타이머를 구현한 코드
- `timer` 제너레이터에서 `Reset` 예외가 발생할 때마다 카운터가 period 로 재설정됨
- **단점**
    - 가독성이 떨어짐
    - `run` 함수에서도 `StopIteration` 예외를 잡아야 할지 `throw` 해야 할지에 따라 방식이 또 나눠짐
        - 코드에 잡음이 너무 많음

- 이터러블 컨테이너 객체 사용한 예제

```python
class Timer:
    def __init__(self, period):
        self.current = period
        self.period = period
        
    def reset(self):
        self.current = self.period
        
    def __iter__(self):
        while self.current:
            self.current -= 1
            yield self.current

def run():
    timer = Timer(4)
    for current in timer:
        if check_for_reset():
            timer.reset()
        announce(current)

run()

>>>
3 틱 남음
2 틱 남음
1 틱 남음
3 틱 남음
2 틱 남음
3 틱 남음
2 틱 남음
1 틱 남음
0 틱 남음
```

- 상태가 있는 클로저를 정의한 형태
- `run` 함수에서는 루프를 사용해 단순하게 이터레이션 수행 가능
- condition depth 가 줄어서 코드의 가독성 높아짐

## 3. 정리

- `throw` 메소드를 사용하면 제너레이터가 마지막으로 실행한 `yield` 식의 위치에서 예외를 다시 발생시킬 수 있음
- 그러나 가독성이 떨어지는 단점이 존재
- 그러므로 이터러블 컨테이너 객체를 사용하는 클래스 생성하여 구현하는 방법으로 진행할 것