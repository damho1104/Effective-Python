# 22. 변수 위치 인자를 사용해 시각적인 잡음을 줄여라

## 1. 가변 인자(=스타 인자)

- 위치 인자를 가변적으로 받는 방법

```python
def log(message, *values):  # 달라진 유일한 부분
    if not values:
        print(message)
    else:
        values_str = ', '.join(str(x) for x in values)
        print(f'{message}: {values_str}')
log('내 숫자는', 1, 2)
log('안녕')  # 훨씬 좋다

>>>
내 숫자는: 1, 2
안녕
```

- `*` 를 붙여 인자의 위치를 고정시키지 않고 인자의 개수도 가변적으로 쓸 수 있음
- 가변 인자 함수에 시퀀스(ex. 리스트) 를 사용하고 싶은 경우

```python
favorites = [7, 33, 99]
log('좋아하는 숫자는', *favorites)
>>>
좋아하는 숫자는: 7, 33, 99
```

## 2. 가변 인자 문제점

- 함수로 전달되기 전 튜플로 변환됨
    - 예로 제너레이터에 `*` 를 붙이면 제너레이터 모든 원소를 얻기 위해 반복함
    - 이로 인해 메모리를 많이 소비함

    ```python
    def my_generator():
        for i in range(10):
            yield i
            
    def my_func(*args):
        print(args)
        
    it = my_generator()
    my_func(*it)

    >>>
    (0, 1, 2, 3, 4, 5, 6, 7, 8, 9)
    ```

- 함수에 새로운 위치 인자를 추가하면 해당 함수를 호출하는 모든 코드를 변경해야 함
    - 이미 가변 인자가 존재하는 함수에서 새로운 위치 인자를 추가하면 호출 코드 전체를 다 수정해야 하므로 문제가 생길 수 있음

    ```python
    def log(sequence, message, *values):
        if not values:
            print(f'{sequence} - {message}')
        else:
            values_str = ', '.join(str(x) for x in values)
            print(f'{sequence} - {message}: {values_str}')
            
    log(1, '좋아하는 숫자는', 7, 33)   # 새 코드에서 가변 인자 사용. 문제없음
    log(1, '안녕')                     # 새 코드에서 가변 인자 없이 메시지만 사용
                                       # 문제없음
    log('좋아하는 숫자는', 7, 33)      # 예전 방식 코드는 깨짐

    >>>
    1 - 좋아하는 숫자는: 7, 33
    1 - 안녕
    좋아하는 숫자는 - 7: 33
    ```

    - 3번째 `log` 함수 호출은 시퀀스를 전달하지 않았기 때문에 7 을 `message` 인자로 사용하므로 문제

## 3. 해결 방안

- 가변 인자를 받아들이는 함수를 확장할 때는 키워드 기반의 인자만 사용
- 더 방어적으로 프로그래밍하려면 타입 애너테이션 사용