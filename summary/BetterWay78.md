# 78. 목을 사용해 의존 관계가 복잡한 코드를 테스트하라

## 1. 목(mock) 을 사용한 테스트

- 테스트를 작성할 떄 필요한 공통 기능
- 사용하기 너무 느리거나 어려운 함수와 클래스에 대해 대체 사용
    - ex. 데이터베이스 실제 연결 후 테스트
- 자신이 흉내 내려는 대상에 의존하는 다른 함수들이 어떤 요청을 보내면 어떤 응답을 보내야 할지 알고 요청에 따라 적절한 응답 반환
- 목 vs 페이크
    - 예제에서 페이크는 DatabaseConnection의 기능을 대부분 제공하지만 더 단순한 단일 스레드 인메모리 데이터베이스 사용
    

## 2. `unittest.mock`

- `Mock` 클래스는 목 함수 생성
- `return_value` 애트리뷰트는 목이 호출되었을 때 반환할 값
- `spec` 인자는 목이 작동을 흉내내야 하는 대상

```python
from datetime import datetime
from unittest.mock import Mock

mock = Mock(spec=get_animals)
expected = [
    ('점박이', datetime(2020, 6, 5, 11, 15)),
    ('털보', datetime(2020, 6, 5, 12, 30)),
    ('조조', datetime(2020, 6, 5, 12, 45)),
]
mock.return_value = expected
```

- 목은 대상에 대한 잘못된 요청이 들어오면 오류 발생

```python
mock.does_not_exist

>>>
Traceback ...
AttributeError: Mock object has no attribute 'does_not_exist'
```

- 목이 생기면 이 목을 호출하고 반환 값을 받고 반환 값이 예상 값인지 검증 가능
- 목이 `database` 를 실제로 사용하지 않기 떄문에 고유한 object 값을 database 의 인자로 넘김
- `DatabaseConnection` 인스턴스를 필요로 하는 객체들이 제대로 작동하기 위해 공급받은 `database` 파라미터를 제대로 연결해 사용하는지에 관심

```python
database = object()
result = mock(database, '미어캣')
assert result == expected
```

- `Mock` 클래스의 `assert_called_once_with` 메소드를 통해 목을 호출한 코드가 제대로 인자를 목에게 전달했는지 확인 가능
    - 잘못된 파라미터를 전달하면 예외 발생
    - `assert_called_once_with` 사용한 테스트 케이스 실패

```python
mock.assert_called_once_with(database, '미어캣')
```

```python
mock.assert_called_once_with(database, '기린')

>>>
Traceback ...
AssertionError: expected call not found.                
Expected: mock(<object object at 0x109038790>, '기린') 
Actual: mock(<object object at 0x109038790>, '미어캣')
```

- 목에 전달되는 개별 파라미터에 관심이 없다면 `unittest.mock.ANY` 상수 사용
- `Mock` 의 `assert_called_with` 메소드를 사용하면 가장 최근에 목을 호출할 때 어떤 인자가 전달돼었는지 확인 가능

```python
from unittest.mock import ANY

mock = Mock(spec=get_animals)
mock('database 1', '토끼')
mock('database 2', '들소')
mock('database 3', '미어캣')

mock.assert_called_with(ANY, '미어캣')
```

- `Mock` 클래스는 예외 발생을 쉽게 모킹할 수 있는 도구 제공

```python
class MyError(Exception):
    pass

mock = Mock(spec=get_animals)
mock.side_effect = MyError('에구머니나! 큰 문제 발생')
result = mock(database, '미어캣')

>>>
Traceback ...
MyError: 에구머니나! 큰 문제 발생
```

## 3. 목 효율적으로 사용하는 방법

- 데이터베이스와 상호작용하기 위한 몇 가지 함수를 사용해 동물원의 여러 동물에게 먹이를 여러 차례 급양하는 함수 정의

```python
def get_food_period(database, species):
    # 데이터베이스에 질의한다
    ...
    # 주기를 반환한다
    
def feed_animal(database, name, when):
    # 데이터베이스에 기록한다
    ...
    
def do_rounds(database, species):
    now = datetime.datetime.utcnow()
    feeding_timedelta = get_food_period(database, species)
    animals = get_animals(database, species)
    fed = 0
    
    for name, last_mealtime in animals:
        if (now - last_mealtime) > feeding_timedelta:
            feed_animal(database, name, now)
            fed += 1
            
    return fed
```

- 테스트 목적
    - 검증
        - `do_rounds` 실행될 때 원하는 동물에게 먹이가 주어졌는지
        - 데이터베이스에 최종 급양 시간이 기록되는지
        - 함수가 반환한 전체 급양 횟수가 제대로인지
- `datetime.utcnow` 모킹
    - 테스트를 실행하는 시간이 서머타임이나 다른 일시적인 변화에 영향을 받지 않도록 함
- `get_food_preiod`, `get_animals` 모킹
    - 데이터베이스에서 값을 가져오는 함수들
- `feed_animal` 모킹
    - 데이터베이스에 값 기록

- **문제**
    - `do_rounds` 함수가 실제 함수가 아닌 목 함수를 쓰게 바꾸는 방법은?

- **방법**
    - 모든 요소를 키워드 방식으로 지정
        
        ```python
        def do_rounds(database, species, *,
                      now_func=datetime.utcnow,
                      food_func=get_food_period,
                      animals_func=get_animals,
                      feed_func=feed_animal):
            now = now_func()
            feeding_timedelta = food_func(database, species)
            animals = animals_func(database, species)
            fed = 0
            
            for name, last_mealtime in animals:
                if (now - last_mealtime) > feeding_timedelta:
                    feed_func(database, name, now)
                    fed += 1
                    
            return fed
        ```
        
        - 모든 `Mock` 인스턴스를 미리 만들고 각각의 예상 반환 값 설정
        - 목을 `do_rounds` 함수에 전달하여 기본 동작 오버라이드하면 테스트 가능
        - `do_rounds` 의존 함수가 예상하는 값 전달받았는지 검증 가능
        
        ```python
        from datetime import timedelta
        
        now_func = Mock(spec=datetime.utcnow)
        now_func.return_value = datetime(2020, 6, 5, 15, 45)
        
        food_func = Mock(spec=get_food_period)
        food_func.return_value = timedelta(hours=3)
        
        animals_func = Mock(spec=get_animals)
        animals_func.return_value = [
            ('점박이', datetime(2020, 6, 5, 11, 15)),
            ('털보', datetime(2020, 6, 5, 12, 30)),
            ('조조', datetime(2020, 6, 5, 12, 45)),
        ]
        
        feed_func = Mock(spec=feed_animal)
        ```
        
        ```python
        result = do_rounds(
            database,
            'Meerkat',
            now_func=now_func,
            food_func=food_func,
            animals_func=animals_func,
            feed_func=feed_func)
        
        assert result == 2
        ```
        
        ```python
        from unittest.mock import call
        
        food_func.assert_called_once_with(database, '미어캣')
        
        animals_func.assert_called_once_with(database, '미어캣')
        
        feed_func.assert_has_calls(
            [
                call(database, '점박이', now_func.return_value),
                call(database, '털보', now_func.return_value),
            ],
            any_order=True)
        ```
        
        - ***단점***
            - 코드가 장황
            - 테스트 대상 함수를 모두 변경해야 함
    - `unittest.mock.patch` 사용
        - `patch` 함수는 임시로 모듈이나 클래스 애트리뷰트에 다른 값 대입해줌
        - `patch` 를 사용하면 데이터베이스에 접근하는 함수들을 임시로 다른 함수로 대치 가능
        
        ```python
        from unittest.mock import patch
        
        print('패치 외부:', get_animals)
        
        with patch('__main__.get_animals'):
           print('패치 내부:', get_animals)
        
        print('다시 외부:', get_animals)
        
        >>>
        패치 외부: <function get_animals at 0x7fe7dd5d3820>
        패치 내부: <MagicMock name='get_animals' id='140633827659680'>
        다시 외부: <function get_animals at 0x7fe7dd5d3820>
        ```
        
        - 모듈이나 클래스, 애트리뷰트에 대해 `patch` 사용 가능
        - `patch` 활용법
            - `with` 문 내
            - 함수 데코레이터
            - `setUp`, `tearDown` 메소드 내
        - `patch` 사용 불가한 경우
            - `do_rounds` 테스트를 위해 현재 시간을 돌려주는 datetime.utcnow 클래스 메소드 모킹 필요
            - `datetime` 클래스가 C 확장 모듈, 아래 코드와 같이 변경은 불가
        
        ```python
        fake_now = datetime(2020, 6, 5, 15, 45)
        
        with patch('datetime.datetime.utcnow'):
            datetime.utcnow.return_value = fake_now
        
        >>>
        Traceback ...           
        TypeError: can't set attributes of built-in/extension type 'datetime.datetime'
        ```
        
        - 방법 1. 우회하려면 `patch` 를 적용할 수 있는 다른 도우미 함수로 시간을 얻어야 함
        
        ```python
        def get_do_rounds_time():
            return datetime.datetime.utcnow()
            
        def do_rounds(database, species):
            now = get_do_rounds_time()
            ...
                                
         with patch('__main__.get_do_rounds_time'):
            ...
        ```
        
        - 방법 2. `datetime.utcnow` 목에 대해서는 키워드로 호출해야 하는 인자로 사용 방법 수정
        
        ```python
        def do_rounds(database, species, *, utcnow=datetime.utcnow):
            now = utcnow()
            feeding_timedelta = get_food_period(database, species)
            animals = get_animals(database, species)
            fed = 0
            
            for name, last_mealtime in animals:
                if (now - last_mealtime) > feeding_timedelta:
                    feed_func(database, name, now)
                    fed += 1
                    
            return fed
        ```
        
        - 방법 2를 사용, `patch.multiple` 함수를 사용해 여러 목을 만들고 각각의 예상 값 설정
        - `patch.multiple` 사용한 `with` 문 내에서 제대로 목 호출이 되었는지 검증 가능
        - `patch.multiple` 키워드 인자들은 `__main__` 모듈에 있는 이름 중에서 테스트하는 동안에만 변경하고 싶은 이름에 해당
        - `DEFAULT` 값은 각 이름에 대해 표준 `Mock` 인스턴스를 생성하고 싶다는 뜻
        - `autospec=True`
            - 만들어진 목은 각각이 시뮬레이션하기로 되어 있는 객체의 명세를 따름
        
        ```python
        from unittest.mock import DEFAULT
        with patch.multiple('__main__',
                            autospec=True,
                            get_food_period=DEFAULT,
                            get_animals=DEFAULT,
                            feed_animal=DEFAULT):
            now_func = Mock(spec=datetime.utcnow)
            now_func.return_value = datetime(2020, 6, 5, 15, 45)
            get_food_period.return_value = timedelta(hours=3)
            get_animals.return_value = [
                ('점박이', datetime(2020, 6, 5, 11, 15)),
                ('털보', datetime(2020, 6, 5, 12, 30)),
                ('조조', datetime(2020, 6, 5, 12, 45))
            ]
        ```
        
        ```python
        result = do_rounds(database, '미어캣', utcnow=now_func)
        assert result == 2
        
        food_func.assert_called_once_with(database, '미어캣')
        animals_func.assert_called_once_with(database, '미어캣')
        
        feed_func.assert_has_calls(
            [
                call(database, '점박이', now_func.return_value),
                call(database, '털보', now_func.return_value),
            ],
            any_order=True)
        ```