# 61. 스레드에 사용한 I/O 를 어떻게 asyncio 로 포팅할 수 있는지 알아두라

## 1. 코루틴을 사용하기 위한 기존 코드 포팅

- 스레드와 블로킹 I/O 를 사용하는 코드를 코루틴과 비동기 I/O를 사용하는 코드로 포팅

- 숫자를 추측하는 게임을 실행해주는 TCP 기반 서버
    - lower, upper: 고려할 숫자의 범위를 표현
    - 클라이언트 요청 시 서버는 이 범위 안의 숫자 반환
    - 서버는 자신이 추측한 숫자가 클라이언트가 정한 비밀의 수에 가까운지 먼지 여부의 정보를 받음
- 블로킹 I/O 와 스레드를 사용
- 메세지 송수신 도우미 클래스

```python
class EOFError(Exception):
    pass
    
class ConnectionBase:
    def __init__(self, connection):
        self.connection = connection
        self.file = connection.makefile('rb')
        
    def send(self, command):
        line = command + '\n'
        data = line.encode()
        self.connection.send(data)
        
    def receive(self):
        line = self.file.readline()
        if not line:
            raise EOFError('Connection closed')
        return line[:-1].decode()
```

- 서버는 한 번에 하나씩 연결을 처리, 클라이언트의 세션 상태 유지 클래스 구현

```python
import random 

WARMER = '더 따뜻함'
COLDER = '더 차가움'
UNSURE = '잘 모르겠음'
CORRECT = '맞음'

class UnknownCommandError(Exception):
    pass

class Session(ConnectionBase):
    def __init__(self, *args):
        super().__init__(*args)
        self._clear_state(None, None)

    def _clear_state(self, lower, upper):
        self.lower = lower
        self.upper = upper
        self.secret = None
        self.guesses = []
```

- 클라이언트에서 들어오는 메시지를 처리하여 명령에 맞는 메소드 호출

```python
    def loop(self):
        while command := self.receive():
            parts = command.split(' ')
            if parts[0] == 'PARAMS':
                self.set_params(parts)
            elif parts[0] == 'NUMBER':
                self.send_number()
            elif parts[0] == 'REPORT':
                self.receive_report(parts)
            else:
                raise UnknownCommandError(command)
```

- 첫번째 명령은 서버가 추측할 값의 상한과 하한 설정

```python
    def set_params(self, parts):
        assert len(parts) == 3
        lower = int(parts[1])
        upper = int(parts[2])
        self._clear_state(lower, upper)
```

- 두번째 명령은 클라이언트에 해당하는 `Session` 인스턴스에 저장된 이전 상태를 바탕으로 새로운 수를 추측
- 서버가 파라미터 설정된 시점 이후에 같은 수를 두 번 반복하여 추측하지 않도록 보장

```python
    def next_guess(self):
        if self.secret is not None:
            return self.secret
            
        while True:
            guess = random.randint(self.lower, self.upper)
            if guess not in self.guesses:
                return guess
                
    def send_number(self):
        guess = self.next_guess()
        self.guesses.append(guess)
        self.send(format(guess))
```

- 세번째 명령은 서버의 추측이 따뜻한지 차가운지에 대해 클라이언트가 보낸 결과를 받은 후 `Session` 상태 변경

```python
    def receive_report(self, parts):
        assert len(parts) == 2
        decision = parts[1]
        
        last = self.guesses[-1]
        if decision == CORRECT:
            self.secret = last

        print(f'서버: {last}는 {decision}')
```

- 클라이언트 코드

```python
import contextlib
import math

class Client(ConnectionBase):
    def __init__(self, *args):
        super().__init__(*args)
        self._clear_state()
        
    def _clear_state(self):
        self.secret = None
        self.last_distance = None

    @contextlib.contextmanager
    def session(self, lower, upper, secret):
        print(f'\n{lower}와 {upper} 사이의 숫자를 맞춰보세요!'
              f' 쉿! 그 숫자는 {secret} 입니다.')
        self.secret = secret
        self.send(f'PARAMS {lower} {upper}')
        try:
            yield
        finally:
            self._clear_state()
            self.send('PARAMS 0 -1')

    def request_numbers(self, count):
        for _ in range(count):
            self.send('NUMBER')
            data = self.receive()
            yield int(data)
            if self.last_distance == 0:
                return

def report_outcome(self, number):
    new_distance = math.fabs(number - self.secret)
    decision = UNSURE
    
    if new_distance == 0:
        decision = CORRECT
    elif self.last_distance is None:
        pass
    elif new_distance < self.last_distance:
        decision = WARMER
    elif new_distance > self.last_distance:
        decision = COLDER
        
    self.last_distance = new_distance
    
    self.send(f'REPORT {decision}')
    return decision
```

- 소켓에 listen 하는 스레드 사용, 새 연결이 들어올 때마다 스레드를 추가로 시작
- 클라이언트는 주 스레드에서 실행, 추측 게임의 결과를 호출한 쪽에 돌려줌

```python
import socket
from threading import Thread

def handle_connection(connection):
    with connection:
        session = Session(connection)
        try:
            session.loop()
        except EOFError:
            pass
            
def run_server(address):
    with socket.socket() as listener:
        listener.bind(address)
        listener.listen()
        while True:
            connection, _ = listener.accept()
            thread = Thread(target=handle_connection,
                            args=(connection,),
                            daemon=True)
            thread.start()

def run_client(address):
    with socket.create_connection(address) as connection:
        client = Client(connection)
        
        with client.session(1, 5, 3):
            results = [(x, client.report_outcome(x))
                       for x in client.request_numbers(5)]
                        
        with client.session(10, 15, 12):
            for number in client.request_numbers(5):
                outcome = client.report_outcome(number)

def main():
    address = ('127.0.0.1', 1234)
    server_thread = Thread(
        target=run_server, args=(address,), daemon=True)
    server_thread.start()
    
    results = run_client(address)
    for number, outcome in results:
        print(f'클라이언트: {number}는 {outcome}')
                                
main()

>>>
1와 5 사이의 숫자를 맞춰보세요! 쉿! 그 숫자는 3 입니다.
서버: 1는 잘 모르겠음
서버: 2는 더 따뜻함
서버: 5는 더 차가움
서버: 3는 맞음
10와 15 사이의 숫자를 맞춰보세요! 쉿! 그 숫자는 12 입니다.
서버: 14는 잘 모르겠음
서버: 10는 잘 모르겠음
서버: 15는 더 차가움
서버: 13는 더 따뜻함
서버: 12는 맞음
클라이언트: 1는 잘 모르겠음
클라이언트: 2는 더 따뜻함
클라이언트: 5는 더 차가움
클라이언트: 3는 맞음
클라이언트: 14는 잘 모르겠음
클라이언트: 10는 잘 모르겠음
클라이언트: 15는 더 차가움
클라이언트: 13는 더 따뜻함
클라이언트: 12는 맞음
```

## 2. 포팅

- `ConnectionBase` 클래스가 블로킹 I/O 대신 `send` 와 `receive` 라는 코루틴을 제공하게 변경 필요

```python
class AsyncConnectionBase:
    def __init__(self, reader, writer):      # 변경됨
        self.reader = reader                 # 변경됨
        self.writer = writer                 # 변경됨
        
    async def send(self, command):
        line = command + '\n'
        data = line.encode()
        self.writer.write(data)              # 변경됨
        await self.writer.drain()            # 변경됨
        
    async def receive(self):
        line = await self.reader.readline()  # 변경됨
        if not line:
            raise EOFError('연결 닫힘')
        return line[:-1].decode()
```

- 단일 연결의 세션 상태 표현 위해 상태를 저장하는 클래스 필요

```python
class AsyncSession(AsyncConnectionBase):     # 변경됨
    def __init__(self, *args):
        ...
        
    def _clear_values(self, lower, upper):
        ...
```

- 서버 명령 처리 루프의 주 진입점 메소드 변경

```python
    async def loop(self):  # 변경됨
        while command := await self.receive():  # 변경됨
            parts = command.split(' ')
            if parts[0] == 'PARAMS':
                self.set_params(parts)
            elif parts[0] == 'NUMBER':  
                await self.send_number()        # 변경됨
            elif parts[0] == 'REPORT':
                self.receive_report(parts)
            else:
                raise UnknownCommandError(command)
```

- 두번째 명령 메소드 변경
    - 추측한 값을 클라이언트에게 송신할 때 비동기 I/O 쓰도록 변경

```python
    def next_guess(self):
        ...
    async def send_number(self):               # 변경됨
        guess = self.next_guess()
        self.guesses.append(guess)
        await self.send(format(guess))         # 변경됨
```

- 클라이언트 코드 변경
    - `AsyncConnectionBase` 로 변경
    - `session` 메소드에서 `async`, `await` 키워드 추가
    - `contextlib` 내장 모듈에서 `async contextmanager` 도우미 함수 사용
    - 두번째, 세번째 명령에도 `async`, `await`

```python
class AsyncClient(AsyncConnectionBase):                     # 변경됨
    def __init__(self, *args):
        ...
        
    def _clear_state(self):
        ...

@contextlib.asynccontextmanager                             # 변경됨
    async def session(self, lower, upper, secret):          # 변경됨
        print(f'\n{lower}와 {upper} 사이의 숫자를 맞춰보세요!'
              f' 쉿! 그 숫자는 {secret} 입니다.')
        self.secret = secret
        await self.send(f'PARAMS {lower} {upper}')          # 변경됨
        try:
            yield
        finally:
            self._clear_state()
            await self.send('PARAMS 0 -1')                   # 변경됨

    async def request_numbers(self, count):         # 변경됨
        for _ in range(count):
            await self.send('NUMBER')               # 변경됨
            data = await self.receive()             # 변경됨
            yield int(data)
            if self.last_distance == 0:
                return

    async def report_outcome(self, number):         # 변경됨
        ...
        await self.send(f'REPORT {decision}')       # 변경됨
        ...

```

- 서버 실행 코드 변경
    - `asyncio` 내장 모듈 사용, 해당 모듈의 `start_server` 함수 사용

```python
import asyncio

async def handle_async_connection(reader, writer):
    session = AsyncSession(reader, writer)
    try:
        await session.loop()
    except EOFError:
        pass
        
async def run_async_server(address):
    server = await asyncio.start_server(handle_async_connection, *address)
    async with server:
        await server.serve_forever()
```

- 클라이언트 실행 코드 변경
    - 블로킹 `socket` 인스턴스와 상호작용하던 모든 부분 변경 필요
        - `asyncio` 사용하도록
        - 코루틴 상호작용 부분 `async`, `await` 키워드 사용
    - 기존 함수를 코루틴으로 포팅하기 위해 `AsyncClient` 와 상호작용하는 대부분의 코드 구조를 바꿀 필요 없음

```python
async def run_async_client(address):
    streams = await asyncio.open_connection(*address)          # 새 기능
    client = AsyncClient(*streams)                             # 새 기능
    
    async with client.session(1, 5, 3):
        results = [(x, await client.report_outcome(x))
                    async for x in client.request_numbers(5)]
                    
    async with client.session(10, 15, 12):
        async for number in client.request_numbers(5):
            outcome = await client.report_outcome(number)
            results.append((number, outcome))
            
    _, writer = streams                                        # 새 기능
    writer.close()                                             # 새 기능
    await writer.wait_closed()                                 # 새 기능
    
    return results

async def main_async():
    address = ('127.0.0.1', 4321)

    server = run_async_server(address)
    asyncio.create_task(server)

    results = await run_async_client(address)
    for number, outcome in results:
        print(f'클라이언트: {number}는 {outcome}')

asyncio.run(main_async())

>>>
1와 5 사이의 숫자를 맞춰보세요! 쉿! 그 숫자는 3 입니다.             
서버: 5는 잘 모르겠음
서버: 1는 잘 모르겠음
서버: 4는 더 따뜻함
10와 15 사이의 숫자를 맞춰보세요! 쉿! 그 숫자는 12 입니다.
서버: 3는 맞음
서버: 15는 잘 모르겠음
서버: 11는 잘 모르겠음
서버: 10는 더 차가움
클라이언트: 5는 잘 모르겠음
클라이언트: 1는 잘 모르겠음
클라이언트: 4는 더 따뜻함
클라이언트: 3는 맞음
클라이언트: 15는 잘 모르겠음
클라이언트: 11는 잘 모르겠음
클라이언트: 10는 더 차가움
```

- 과연 포팅은 수월할 것인가?
    - `next` 와 `iter` 내장 함수에 대응하는 비동기 함수 없음
        - 직접 `_anext_` 나 `_aiter_` 메소드에 대해 `await` 해야 함
    - `yield from` 대응 비동기 버전 없음

- 문서 참고
    - [https://docs.python.org/3/library/asyncio.html](https://docs.python.org/3/library/asyncio.html)