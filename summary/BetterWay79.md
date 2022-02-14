# 79. 의존 관계를 캡슐화해 모킹과 테스트를 쉽게 만들라

## 1. 의존 관계 캡슐화

- `unittest.mock` 사용하는 이전 챕터 방식으로 만든 테스트 케이스는 준비 코드가 많이 들어가 가독성 떨어짐
- `DatabaseConnection` 객체를 인자로 직접 전달하는 대신 래퍼 객체를 사용해 데이터베이스 인터페이스를 캡슐화
- 추상화를 사용하면 목이나 테스트를 더 쉽게 만들수 있음

- 데이터베이스 도우미 함수를 개별 함수가 아니라 하나의 클래스 안에 있는 메소드로 다시 정의
- 이제 `do_rounds` 함수가 `ZooDatabase` 객체의 메소드를 호출하도록 변경

```python
class ZooDatabase:
    ...
                           
    def get_animals(self, species):
        ...
                               
    def get_food_period(self, species):
        ...
  
    def feed_animal(self, name, when):
        ...
```

```python
from datetime import datetime

def do_rounds(database, species, *, utcnow=datetime.utcnow):
    now = utcnow()
    feeding_timedelta = database.get_food_period(species)
    animals = database.get_animals(species)
    fed = 0
    
    for name, last_mealtime in animals:
        if (now - last_mealtime) >= feeding_timedelta:
            database.feed_animal(name, now)
            fed += 1

    return fed
```

- `unittest.mock.patch` 를 사용해 목을 테스트 대상 코드에 주입할 필요 없음
- `ZooDatabase` 표현 `Mock` 인스턴스를 생성해서 `do_rounds` 의 database로 넘길 수 있음
    - `Mock` 클래스는 자신의 애트리뷰트에 대해 이뤄지는 모든 접근에 대해 목 객체 반환
    - 애트리뷰트 메소드처럼 호출가능
    - 애트리뷰트로 반환될 예상 값 설정, 호출 여부 검증 가능

```python
from unittest.mock import Mock

database = Mock(spec=ZooDatabase)
print(database.feed_animal)
database.feed_animal()
database.feed_animal.assert_any_call()

>>>
<Mock name='mock.feed_animal' id='4384773408'>
```

- `ZooDatabase` 캡슐화를 사용하도록 `Mock` 설정 코드 다시 작성
- 테스트 대상 함수 실행, 함수가 의존하는 모든 메소드 검증 가능

```python
from datetime import timedelta
from unittest.mock import call

now_func = Mock(spec=datetime.utcnow)
now_func.return_value = datetime(2019, 6, 5, 15, 45)

database = Mock(spec=ZooDatabase)
database.get_food_period.return_value = timedelta(hours=3)
database.get_animals.return_value = [
    ('점박이', datetime(2019, 6, 5, 11, 15)),
    ('털보', datetime(2019, 6, 5, 12, 30)),
    ('조조', datetime(2019, 6, 5, 12, 55))
]
```

```python
result = do_rounds(database, '미어캣', utcnow=now_func)
assert result == 2

database.get_food_period.assert_called_once_with('미어캣')
database.get_animals.assert_called_once_with('미어캣')
database.feed_animal.assert_has_calls(
    [
       call('점박이', now_func.return_value),
       call('털보', now_func.return_value),
    ],
    any_order=True)
```

- 클래스를 모킹할 때 `spec` 파라미터를 `Mock` 에 사용하면 테스트 대상 코드가 실수로 메소드 이름을 잘못 사용하는 경우 발견 가능
    - 테스트 대상 코드와 단위 테스트에서 메소드 이름을 잘못 사용하는 오류를 사전에 발견 가능

```python
database.bad_method_name()

>>>
Traceback ...
AttributeError: Mock object has no attribute 'bad_method_name'
```

- 프로그램을 중간 수준의 통합 테스트와 함께 테스트를 원할 경우 프로그램에 ZooDatabase 를 주입할 방법이 필요
    - 의존 관계 주입의 연결점 역할을 하는 도우미 함수 생성하여 주입

```python
DATABASE = None

def get_database():
    global DATABASE
    if DATABASE is None:
        DATABASE = ZooDatabase()
    return DATABASE

def main(argv):
    database = get_database()
    species = argv[1]
    count = do_rounds(database, species)
    print(f'급양: {count} {species}')
    return 0
```

- patch를 사용해 ZooDatabase 목을 주입, 테스트 실행, 검증
- `datetime.utcnow` 목을 사용하지 않고 단위 테스트와 비슷한 결과를 낼 수 있도록 목이 반환하는 데이터베이스 레코드의 시간을 현재 시간에 대한 상대적인 값으로 설정

```python
import contextlib
import io
from unittest.mock import patch

with patch('__main__.DATABASE', spec=ZooDatabase):
    now = datetime.utcnow()

    DATABASE.get_food_period.return_value = timedelta(hours=3)
    DATABASE.get_animals.return_value = [
        ('점박이', now - timedelta(minutes=4.5)),
        ('털보', now - timedelta(hours=3.25)),
        ('조조', now - timedelta(hours=3)),
    ]

    fake_stdout = io.StringIO()
    with contextlib.redirect_stdout(fake_stdout):
        main(['프로그램 이름', '미어캣'])

    found = fake_stdout.getvalue()
    expected = '급양: 2 미어캣\n'

    assert found == expected
```