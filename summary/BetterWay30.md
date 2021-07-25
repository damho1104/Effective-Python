# 30. 리스트를 반환하기보다는 제너레이터를 사용하라

## 1. 리스트 반환 코드 문제점

```python
def index_words(text):
    result = []
    if text:
        result.append(0)
    for index, letter in enumerate(text):
        if letter == ' ':
            result.append(index + 1)
    return result

address = '컴퓨터(영어: Computer, 문화어: 콤퓨터 , 순화어: 전산기)는 진공관'
result = index_words(address)
print(result[:10])

>>>
[0, 8, 18, 23, 28, 38]
```

- `index_words` 함수 문제점
    - 코드 가독성 떨어짐, 핵심 파악 어려움
        - 리스트에 append 되는 값이 무엇을 뜻하는지 파악하기 어려움
    - 반환하기 전 리스트에 모든 결과를 다 저장함
        - 리스트에 저장될 데이터가 너무 커서 메모리 초과될 수 있음

## 2. 제너레이터

- yield 를 사용하는 함수에 의해 만들어짐

```python
def index_words_iter(text):
    if text:
        yield 0
    for index, letter in enumerate(text):
        if letter == ' ':
            yield index + 1

it = index_words_iter(address)
print(next(it))
print(next(it))

>>>
0
8
```

- 함수 실행이 아닌 `iterator` 반환
- 이터레이터를 가지고 `next` 내장 함수를 호출할 때 실제 함수가 `yield`까지 실행됨
- 이전 `yield` 를 재사용 할 수 없는 단점 존재

## 3. 제너레이터 장점

- 반환하는 리스트와의 상호작용하는 코드가 사라졌으므로 가독성 높아짐
- 실제 사용하는 메모리 크기 제한 가능