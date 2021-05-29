# 16. in 을 사용하고 딕셔너리 키가 없을 때 KeyError를 처리하기 보다는 get 을 사용하라

## 1. 딕셔너리

- 키 혹은 키와 연관된 값에 접근, 대입, 수정, 삭제하는 것
- 딕셔너리 내용은 동적으로 생성 혹은 수정되므로 키가 항상 존재한다고 보장할 수 없음
- 그러므로 키가 존재하는 체크해야 함

```python
counters = {
    '품퍼니켈': 2,
    '사워도우': 1,
}
key = '밀'
if key in counters:
    count = counters[key]
else:
    count = 0
    
counters[key] = count + 1
```

- 다음 예제는 `counters` 딕셔너리에 `'밀'` 이라는 키가 존재하는지 확인하는 과정을 `if`, `else` 로 처리하였음

```python
counters = {
    '품퍼니켈': 2,
    '사워도우': 1,
}
key = '밀'
try:
    count = counters[key]
except KeyError:
    count = 0
    
counters[key] = count + 1
```

- 다음 예제는 조건문 대신 `try except` 문을 사용하여 키 존재 여부를 처리하였음
- 이 예제는 일단 키를 사용해 값을 접근하려는 시도를 하고 키가 없다면 예외 발생시켜서 처리하도록 하였음
- 이 예제는 키 접근이 1번만 이뤄지므로 조금 더 효율적

## 2. 딕셔너리의 `get` 메소드

- `get` 메소드는 인자를 키값을 받음
- 2번째 인자로 키가 딕셔너리에 존재하지 않는 경우 반환할 값을 넣을 수 있음

```python
counters = {
    '품퍼니켈': 2,
    '사워도우': 1,
}
key = '밀'
count = counters.get(key, 0) 
counters[key] = count + 1
```

- 딕셔너리의 value 가 복잡한 자료구조 타입인 경우는?

```python
votes = {
    '바게트': ['철수', '순이'],
    '치아바타': ['하니', '유리'],
}
key = '브리오슈'
who = '단이'

if key in votes:
    names = votes[key]
else:
    votes[key] = names = []
    
names.append(who)
print(votes)

>>>
{'바게트': ['철수', '순이'], '치아바타': ['하니', '유리'], '브리오슈': ['단이']}
```

- `if, else` 문 사용, key 접근 2번, 비효율적

```python
...
try:
    names = votes[key]
except KeyError:
    votes[key] = names = []
    
names.append(who)
```

- `try, except` 문 사용, key 접근 1번, `if, else` 문에 비해 효율적

```python
names = votes.get(key)
if names is None:
    votes[key] = names = []
    
names.append(who)

# More Better Way-------------------------------

if (names := votes.get(key)) is None:
    votes[key] = names = []
    
names.append(who)
```

- `get` 메소드 사용

## 3. `setdefault` 메소드

- 딕셔너리에서 키를 사용해 값을 가져오려 할 때 키가 없다면 제공받은 기본 값을 키에 연관시켜 대입 후 값 반환

```python
names = votes.setdefault(key, [])

names.append(who)
```

- **단점**
    - 가독성 별로
        - 메소드의 동작을 이름으로 드러내지 못함(get 동작인데 이름은 `set` 임)
    - 키가 없으면 `setdefault`에 전달된 기본 값이 복사되지 않고 딕셔너리에 직접 대입

        ```python
        data = {}
        key = 'foo'
        value = []
        data.setdefault(key, value)
        print('이전:', data)
        value.append('hello')
        print('이후:', data)

        >>>
        이전: {'foo': []}
        이후: {'foo': ['hello']}
        ```

        - 위 예제와 같이 value 리스트가 깊은 복사 처리되지 않으므로 reference화 됨
        - 결국 `setdefault` 함수를 호출할 때 마다 리스트 새로 생성해야 한다는 것

- 만일 `setdefault` 를 꼭 사용해야 하는 경우라면 `defaultdict`를 사용해도 되는지 고려할 것~!
    - `defaultdict` 는 `Better Way 17` 참고

## Note

- 만일 딕셔너리의 `value` 가 카운터로만 쓰인다면 `collections` 패키지의 `counter` 클래스 고려