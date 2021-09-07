# 39. 객체를 제너릭하게 구성하려면 @classmethod를 통한 다형성을 활용하라

## 1. 다형성을 사용하지 않은 맵리듀스 예제

- `InputData`: 추상 인터페이스,  입력 데이터
    - `PathInputData`
- `Worker`: 추상 인터페이스, 입력 데이터를 소비하는 공통 방법을 제공하는 맵리듀스 worker
    - `LineCountWorker`
- 클래스 객체 생성 및 연동 함수
    - `generate_inputs`
    - `create_workers`
    - `execute`
    - `mapreduce`

```python
class InputData:
    def read(self):
        raise NotImplementedError

class PathInputData(InputData):
    def __init__(self, path):
        super().__init__()
        self.path = path
        
    def read(self):
        with open(self.path) as f:
            return f.read()

class Worker:
    def __init__(self, input_data):
        self.input_data = input_data
        self.result = None
        
    def map(self):
        raise NotImplementedError
        
    def reduce(self, other):
        raise NotImplementedError

class LineCountWorker(Worker):
    def map(self):
        data = self.input_data.read()
        self.result = data.count('\n')
        
    def reduce(self, other):
        self.result += other.result

import os

def generate_inputs(data_dir):
    for name in os.listdir(data_dir):
        yield PathInputData(os.path.join(data_dir, name))

def create_workers(input_list):
    workers = []
    for input_data in input_list:
        workers.append(LineCountWorker(input_data))
    return workers

from threading import Thread

def execute(workers):
    threads = [Thread(target=w.map) for w in workers]
    for thread in threads: thread.start()
    for thread in threads: thread.join()
    
    first, *rest = workers
    for worker in rest:
        first.reduce(worker)
    return first.result

def mapreduce(data_dir):
    inputs = generate_inputs(data_dir)
    workers = create_workers(inputs)
    return execute(workers)

import os
import random

def write_test_files(tmpdir):
    os.makedirs(tmpdir)
    for i in range(100):
        with open(os.path.join(tmpdir, str(i)), 'w') as f:
            f.write('\n' * random.randint(0, 100))

tmpdir = 'test_inputs'
write_test_files(tmpdir)

result = mapreduce(tmpdir)
print(f'총 {result} 줄이 있습니다.')

>>>
총 5474 줄이 있습니다.
```

- 단점
    - generic 하지 않음
        - 다른 `InputData` 나 `Worker` 하위 클래스를 사용하고 싶다면 연결 관련 함수 재작성해야 함
    - 다형성을 고려해 클래스 계층 관계를 만든다고 해도 **파이선에서는 생성자는 하나만 정의 가능함**

- 해결방안
    - 클래스 메소드 다형성 사용
        - 클래스에 다른 생성자를 우회하여 정의 가능

    ```python
    class GenericInputData:
        def read(self):
            raise NotImplementedError
            
        @classmethod
        def generate_inputs(cls, config):
            raise NotImplementedError

    class PathInputData(GenericInputData):
        ...
       
        @classmethod
        def generate_inputs(cls, config):
            data_dir = config['data_dir']
            for name in os.listdir(data_dir):
                yield cls(os.path.join(data_dir, name))

    class GenericWorker:
        def __init__(self, input_data):
            self.input_data = input_data
            self.result = None
            
        def map(self):
            raise NotImplementedError
            
        def reduce(self, other):
            raise NotImplementedError
            
        @classmethod
        def create_workers(cls, input_class, config):
            workers = []
            for input_data in input_class.generate_inputs(config):
                workers.append(cls(input_data))
            return workers

    class LineCountWorker(GenericWorker):
        ...

    def mapreduce(worker_class, input_class, config):
        workers = worker_class.create_workers(input_class, config)
        return execute(workers)

    config = {'data_dir': tmpdir}
    result = mapreduce(LineCountWorker, PathInputData, config)
    print(f'총 {result} 줄이 있습니다.')

    >>>
    총 5474 줄이 있습니다.
    ```