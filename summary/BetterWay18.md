# 18. __missing__ 을 사용해 키에 따라 다른 디폴트 값을 생성하는 방법을 알아두라

## 1. `setdefault` 와 `defaultdict` 를 모두 사용하기 적당하지 않은 경우

```python
pictures = {}
path = 'profile_1234.png'

if (handle := pictures.get(path)) is None:
    try:
        handle = open(path, 'a+b')
    except OSError:
        print(f'경로를 열 수 없습니다: {path}')
        raise
    else:
        pictures[path] = handle

handle.seek(0)
image_data = handle.read()
```

- 위 예제에서 키가 없으면 `path` 를 가지고 `open` 을 시도함
- 예외 처리는 잘 되어 있으나 딕셔너리를 더 많이 읽고 내포되는 블록 깊이가 더 깊어지는 단점 존재

```python
try:
    handle = pictures.setdefault(path, open(path, 'a+b'))
except OSError:
    print(f'경로를 열 수 없습니다: {path}')
    raise
else:
    handle.seek(0)
    image_data = handle.read()
```

- 위 예제는 `setdefault` 를 사용한 예제
- 단점
    - 경로에 파일이 존재하는지 여부 확인 없이 open 시도함
    - 예외가 `open` 에서 `throw` 되었는지 `setdefault` 에서 throw 되었는지 애매

```python
from collections import defaultdict

def open_picture(profile_path):
    try:
        return open(profile_path, 'a+b')
    except OSError:
        print(f'경로를 열 수 없습니다: {profile_path}')
        raise
        
pictures = defaultdict(open_picture)
handle = pictures[path]
handle.seek(0)
image_data = handle.read()
```

- 위 예제는 `defaultdict` 를 사용한 예제
- 단점
    - `defaultdict` 에 전달한 함수의 인자를 전달할 수 있는 방법이 없음

## 2. `__missing__` 메소드

- `dict` 를 상속받은 클래스에서 사용
- 키가 없는 경우에 대해 처리하는 로직을 커스텀하게 구현 가능

```python
class Pictures(dict):
    def __missing__(self, key):
        value = open_picture(key)
        self[key] = value
        return value
        
pictures = Pictures()
handle = pictures[path]
handle.seek(0)
image_data = handle.read()

```

- `pictures[path]` 에서 `path` 키가 없는 경우 `__missing__` 메소드 호출
- `__missing__` 메소드에서는 키에 대한 값을 삽입하므로 이후 키로 접근하면 값을 반환받을 수 있음