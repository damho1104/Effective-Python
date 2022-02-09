# 77. setUp, tearDown, setUpModule, tearDownModule 을 사용해 각각의 테스트를 격리하라

## 1. 테스트 하네스(Test Harness)

- 테스트 메소드 실행 전 테스트 환경 구축을 위해 `setUp`, `tearDown` 메소드 오버라이드
    - `setUp`
        - 테스트 메소드를 실행하기 전에 호출
    - `tearDown`
        - 테스트 메소드 실행되고 난 이후 호출
        

```python
# environment_test.py
from pathlib import Path
from tempfile import TemporaryDirectory
from unittest import TestCase, main

class EnvironmentTest(TestCase):
    def setUp(self):
        self.test_dir = TemporaryDirectory()
        self.test_path = Path(self.test_dir.name)
        
    def tearDown(self):
        self.test_dir.cleanup()
        
    def test_modify_file(self):
        with open(self.test_path / 'data.bin', 'w') as f:
            ...
                                    
if __name__ == '__main__':
    main()
```

- 실행 순서
    - `main` → ... → `EnvironmentTest` → `setUp` → `test_modify_file` → `tearDown`
- 프로그램이 복잡해지면 코드를 독립적으로 실행하는 대신 여러 모듈 사이의 end-to-end 상호작용을 검증하는 테스트 필요 (통합 테스트, integration test)
- 통합 테스트에 필요한 환경 구축 시 계산 비용이 너무 비싸거나 시간이 오래 소요될 수 있음
    - ***예.*** 통합 테스트를 위해 데이터베이스 프로세스 시작, 데이터베이스 모든 인덱스를 메모리에 적재되어야 함
    - 이런 경우 `setUp`, `tearDown` 메소드에서 준비하고 정리하는 과정이 실용적이지 않음

## 2. 모듈 단위 테스트 하네스

- 비싼 자원을 단 한 번만 초기화 가능
- 초기화를 반복하지 않고도 모든 `TestCase` 클래스와 테스트 메소드 실행 가능
- 모듈 내 모든 테스트가 끝나면 테스트 하네드 1번만 정리

- 아래 예제를 통해 `setUpModule` 이 1번 만 실행되었음
- 모든 `setUp` 메소드가 호출되기 전 `setUpModule` 이 호출
- `tearDownModule` 은 모든 `tearDown` 메소드 호출된 이후 1번 호출

```python
# integration_test.py
from unittest import TestCase, main

def setUpModule():
    print('* 모듈 설정')

def tearDownModule():
    print('* 모듈 정리')

class IntegrationTest(TestCase):
    def setUp(self):
        print('* 테스트 설정')

    def tearDown(self):
        print('* 테스트 정리')

    def test_end_to_end1(self):
        print('* 테스트 1')

    def test_end_to_end2(self):
        print('* 테스트 2')

if __name__ == '__main__':
    main()

$ python3 integration_test.py
* 모듈 설정
* 테스트 설정
* 테스트 1
* 테스트 정리
.* 테스트 설정
* 테스트 2
* 테스트 정리
.* 모듈 정리

----------------------------------------------------------------------
Ran 2 tests in 0.000s

OK
```