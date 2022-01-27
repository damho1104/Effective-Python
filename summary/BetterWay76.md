# 76. TestCase 하위 클래스를 사용해 프로그램에서 연관된 행동 방식을 검증하라

## 1. 파이선 테스트 모듈

- `unittest` 내장 모듈

```python
def to_string(data):
    if isinstance(data, str):
        return data
    elif isinstance(data, bytes):
        return data.decode('utf-8')
    else:
        raise TypeError(f'string 으로 변환할 수 있는 객체가 아닙니다 (Value: %r).' % data)
```

- 위 코드에 대해 테스트 추가
    - 테스트 실패 시 자세한 출력까지 표시 가능

```python
from unittest import TestCase, main
from utils import to_str

class UtilsTestCase(TestCase):
    def test_to_string_for_bytes(self):
        self.assertEqual('test1', to_string(b'test1'))
        
    def test_to_string_for_str(self):
        self.assertEqual('test2', to_string('test2'))
        
    def test_fail(self):
        self.assertEqual('test3', to_string('test4'))
        
if __name__ == '__main__':
    main()
```

```python
$ python3 better_way_76_test.py
F..
===============================================================
FAIL: test_fail (_ _main__.UtilsTestCase)
---------------------------------------------------------------
Traceback (most recent call last):
  File "better_way_76_test.py", line 15, in test_fail
    self.assertEqual('test3', to_string('test4'))
AssertionError: 'test3' != 'test4'
- test3
+ test4

---------------------------------------------------------------
Ran 3 tests in 0.002s

FAILED (failures=1)
```

- 테스트는 `TestCase` 의 하위 클래스로 구성
- 각 테스트 케이스는 이름이 `test` 라는 단어로 시작하는 메소드
- 테스트 메소드가 어떤 Exception 도 발생시키지 않고 실행이 끝나면 테스트가 성공한 것으로 간주
    - 주로 `AssertionError` 로 Exception 발생시킴
- 테스트 메소드가 실패해도 그 이후의 테스트는 계속 진행시킴
- 특정 테스트 메소드에 대한 실행

```python
$ python3 better_way_76_test.py UtilsTestCase.test_to_string_for_bytes
.       
---------------------------------------------------------------
Ran 1 test in 0.000s 

OK
```

## 2. TestCase 클래스의 도우미 메소드

- Assertion 을 만들 때 도움이 되는 도우미 메소드
    - `assertEqual`
        - 두 값이 같은지 비교
    - `assertTrue`
        - 주어진 bool 식이 참인지 검증
    - 파이선 내장 `assert` 문과의 차이 예제
    
    ```python
    from unittest import TestCase, main
    from utils import to_str
    
    class AssertTestCase(TestCase):
        def test_assert_helper(self):
            expected = 14
            found = 2 * 6
            self.assertEqual(expected, found)
            
        def test_assert_statement(self):
            expected = 14
            found = 2 * 6
            assert expected == found
            
    if __name__ == '__main__':
        main()
    ```
    
    ```python
    $ python3 assert_test.py
    FF
    ===============================================================
    FAIL: test_assert_helper (_ _main__.AssertTestCase)
    ---------------------------------------------------------------
    Traceback (most recent call last):
      File "assert_test.py", line 16, in test_assert_helper
        self.assertEqual(expected, found)
    AssertionError: 14 != 12
    ===============================================================
    FAIL: test_assert_statement (_ _main__.AssertTestCase)
    ---------------------------------------------------------------
    Traceback (most recent call last):
      File "assert_test.py", line 11, in test_assert_statement
        assert expected == found
    AssertionError
    ---------------------------------------------------------------
    Ran 2 tests in 0.001s
    
    FAILED (failures=2)
    ```
    
- `assertRaises`
    - 예외가 발생하는지 검증하기 위해 with 문 안에서 컨텍스트 매니저로 사용 가능
    - 이 메소드를 사용한 코드는 `try/except` 문과 비슷
        - 테스트 케이스의 해당 부분에서 예외가 발생할 것으로 예상한다는 점을 명확히 파악 가능
    
    ```python
    from unittest import TestCase, main
    from utils import to_str
    class UtilsErrorTestCase(TestCase):
        def test_to_string_for_bad_case(self):
            with self.assertRaises(TypeError):
                to_string(object())
            
        def test_to_string_for_bad_encoding_case(self):
            with self.assertRaises(UnicodeDecodeError):
                to_string(b'\xfa\xfa')
                
    if __name__ == '__main__':
        main()
    ```
    
- `TestCase` 하위 클래스 안에 복잡한 로직이 들어가는 유저 커스텀 메소드 정의 가능
    - 가독성 증가
    - 도우미 메소드 이름이 test 로 시작하지 않아야 함
    
    ```python
    from unittest import TestCase, main
    
    def sum_squares(values):
        cumulative = 0
        for value in values:
            cumulative += value ** 2
            yield cumulative
    
    class HelperTestCase(TestCase):
        def verify_complex_case(self, values, expected):
            expect_it = iter(expected)
            found_it = iter(sum_squares(values))
            test_it = zip(expect_it, found_it)
    
            for i, (expect, found) in enumerate(test_it):
                self.assertEqual(
                    expect,
                    found,
                    f'잘못된 인덱스: {i}')
    
            # 두 제너레이터를 모두 소진했는지 검증
            try:
                next(expect_it)
            except StopIteration:
                pass
            else:
                self.fail('실제보다 예상한 제너레이터가 더 깁니다')
                
            try:
                next(found_it)
            except StopIteration:
                pass
            else:
                self.fail('예상한 제너레이터보다 실제가 더 깁니다')
    
        def test_wrong_lengths(self):
            values = [1.1, 2.2, 3.3]
            expected = [
                1.1 ** 2,
            ]
            self.verify_complex_case(values, expected)
    
        def test_wrong_results(self):
            values = [1.1, 2.2, 3.3]
            expected = [
                1.1 ** 2,
                1.1 ** 2 + 2.2 ** 2,
                1.1 ** 2 + 2.2 ** 2 + 3.3 ** 2 + 4.4 ** 2,
            ]
            self.verify_complex_case(values, expected)
    
    if __name__ == '__main__':
        main()
    ```
    
    ```python
    $ python3 helper_test.py
    FF
    ======================================================================|
    FAIL: test_wrong_lengths (_ _main__.HelperTestCase)
    ----------------------------------------------------------------------
    Traceback (most recent call last):
      File "helper_test.py", line 41, in test_wrong_lengths
        self.verify_complex_case(values, expected)
      File "helper_test.py", line 34, in verify_complex_case
        self.fail('예상한 제너레이터보다 실제가 더 깁니다')
    AssertionError: 예상한 제너레이터보다 실제가 더 깁니다
    
    ======================================================================
    FAIL: test_wrong_results (_ _main__.HelperTestCase)
    ----------------------------------------------------------------------
    Traceback (most recent call last):
      File "helper_test.py", line 50, in test_wrong_results
        self.verify_complex_case(values, expected)
      File "helper_test.py", line 17, in verify_complex_case
        self.assertEqual(
    AssertionError: 36.3 != 16.939999999999998 : 잘못된 인덱스: 2
    
    ----------------------------------------------------------------------
    Ran 2 tests in 0.001s
    
    FAILED (failures=2)
    ```
    
- 일반적으로 한 `TestCase` 하위 클래스 안에 연관된 일련의 테스트를 함께 정의
- `TestCase` 클래스가 제공하는 `subTest` 도우미 메소드를 사용하면 한 테스트 메소드 안에 여러 테스트 정의 가능
    - 준비 코드를 작성하지 않아도 됨
    - 데이터 기반 테스트를 작성할 때 `subTest` 유리
    - 하위 테스트 케이스 중 하나가 실패해도 다른 테스트 케이스 계속 진행 가능
    
    ```python
    # data_driven_test.py
    from unittest import TestCase, main
    from utils import to_str
    class DataDrivenTestCase(TestCase):
        def test_good(self):
            good_cases = [
                (b'my bytes', 'my bytes'),
                ('no error', b'no error'),  # 이 부분에서 실패함
                ('other str', 'other str'),
                ...
            ]
            for value, expected in good_cases:
                with self.subTest(value):
                    self.assertEqual(expected, to_str(value))
    
        def test_bad(self):
            bad_cases = [
                (object(), TypeError),
                (b'\xfa\xfa', UnicodeDecodeError),
                ...
            ]
            for value, exception in bad_cases:
                with self.subTest(value):
                    with self.assertRaises(exception):
                        to_str(value)
    
    if __name__ == '__main__':
        main()
    ```
    
    ```python
    $ python3 data_driven_test.py
    .
    ===============================================================
    FAIL: test_good (_ _main__.DataDrivenTestCase) [no error]
    ---------------------------------------------------------------
    Traceback (most recent call last):
      File "testing/data_driven_test.py", line 18, in test_good
        self.assertEqual(expected, to_str(value))
    AssertionError: b'no error' != 'no error'
    
    ---------------------------------------------------------------
    Ran 2 tests in 0.001s
    
    FAILED (failures=1)
    ```
    
- 프로젝트의 복잡성과 테스트 요구사항에 따라 `pytest` 오픈 소스 패키지 혹은 다양한 플러그인 사용 가능