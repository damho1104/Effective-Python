# 74. bytes 를 복사하지 않고 다루려면 memoryview와 bytearray를 사용하라

## 1. 파이선에서의 적절한 I/O 지원 선택 중요성

- 파이선이 CPU 위주의 계산 작업을 추가적인 노력없이 병렬화해줄 수는 없으나 스루풋이 높은 병렬 I/O 를 다양한 방식으로 지원 가능
    - I/O 지원을 잘못 사용하여 파이선이 I/O 위주의 부하에 대해서도 느리다는 결론으로 이어짐
    
- 네트워크를 통해 스트리밍하는 미디어 서버 예
    - 플레이 중인 비디오 앞, 뒤로 이동 기능
    - 반복 기능

```python
def timecode_to_index(video_id, timecode):
    ...
    # 비디오 데이터의 바이트 오프셋을 반환한다
    
def request_chunk(video_id, byte_offset, size):
    ...
    # video_id에 대한 비디오 데이터 중에서 바이트 오프셋부터# size만큼을 반환한다

video_id = ...
timecode = '01:09:14:28
byte_offset = timecode_to_index(video_id, timecode)
size = 20 * 1024 * 1024
video_data = request_chunk(video_id, byte_offset, size)
```

- `request_chunk` 요청을 받아서 요청에 해당하는 데이터를 도렬주는 서버 측 핸들러 구현
    - 요청 받은 데이터 덩어리를 메모리 캐시에 들어 있는 수 기가바이트 크기의 비디오 정보에서 꺼낸 후 소켓을 통해 클라이언트에게 돌려주는 과정에 집중
- 해당 코드의 지연 시간과 스루풋
    1. `video_data` 에서 20MB 의 비디오 덩어리를 가져오는 데 걸리는 시간
    2. 데이터를 클라이언트에게 송신하는 데 걸리는 시간
    - 2번이 무한히 빠르다고 가정하면 1번에만 집중
    - 데이터 덩어리를 만들기 위해 bytes 인스턴스를 슬라이싱하는 방법 시간 측정으로 확인 가능

```python
socket = ...             # 클라이언트가 연결한 소켓
video_data = ...         # video_id에 해당하는 데이터가 들어 있는# bytes
byte_offset = ...        # 요청받은 시작 위치
size = 20 * 1024 * 1024  # 요청받은 데이터 크기

chunk = video_data[byte_offset:byte_offset + size]
socket.send(chunk)
```

- 대략 5밀리초
- 서버의 최대 스루풋
    - 이론적으로 20MB / 5밀리초 = 7.3GB
- 병렬로 새 데이터 덩어리를 요청할 수 있는 클라이언트의 최대 개수
    - 1 CPU 초 / 5밀리초 = 200
- **문제**
    - 기반 데이터를 `bytes` 인스턴스로 슬라이싱하려면 메모리를 복사해야 하는데 이 과정이 CPU 시간 점유

```python
import timeit

def run_test():
    chunk = video_data[byte_offset:byte_offset + size]
    # socket.send(chunk)를 호출해야 하지만 벤치마크를 위해 무시한다

result = timeit.timeit(
    stmt='run_test()',
    globals=globals(),
    number=100) / 100

print(f'{result:0.9f} 초')

>>>
0.004925669 초
```

## 2. `memoryview` 내장 타입 사용

- CPython 의 고성능 버퍼 프로토콜을 프로그램에 노출
- 버퍼 프로토콜은 파이선 런타임과 C 확장이 bytes 와 같은 객체를 통하지 않고 하부 데이터 버퍼에 접근할 수 있게 해주는 저수준 C API
- 슬라이싱에서 데이터를 복사하지 않고 새로운 `memoryview` 인스턴스를 생성함

```python
data = '동해물과 abc 백두산이 마르고 닳도록'.encode("utf8")
view = memoryview(data)
chunk = view[12:19]
print(chunk)
print('크기:', chunk.nbytes)
print('뷰의 데이터:', chunk.tobytes())
print('내부 데이터:', chunk.obj)

>>>
<memory at 0x000002245F779D00>
크기: 7
뷰의 데이터: b' abc \xeb\xb0'
내부 데이터: b'\xeb\x8f\x99\xed\x95\xb4\xeb\xac\xbc\xea\xb3\xbc abc\xeb\xb0\xb1\xeb\x91\x90\xec\x82\xb0\xec\x9d\xb4 \xeb\xa7\x88\xeb\xa5\xb4\xea\xb3\xa0 \xeb\x8b\xb3\xeb\x8f\x84\xeb\xa1\x9d
```

- zero-copy 연산을 활성화하여 NumPy 같은 수치 계산 C 확장이나 예제 프로그램 같은 I/O 위주 프로그램이 커다란 메모리를 빠르게 처리해야 할 경우 성능 향상 가능

- `bytes` 슬라이스를 `memoryview` 로 변경한 후 벤치마크 수행한 결과

```python
video_view = memoryview(video_data)
def run_test():
    chunk = video_view[byte_offset:byte_offset + size]
    # socket.send(chunk)를 호출해야 하지만 벤치마크를 위해 무시한다

result = timeit.timeit(
    stmt='run_test()',
    globals=globals(),
    number=100) / 100

print(f'{result:0.9f} 초')

>>>
0.000000250 초
```

- 성능 개선으로 인해 프로그램의 성능은 CPU bound 보단 소켓 연결 성능에 따라 제한

- 일부 클라이언트가 여러 사용자에게 방송하기 위해 서버러 라이브 비디오 스트림 보내는 예
    - 사용자가 가장 최근에 보낸 비디오 데이터를 캐시에 삽입, 다른 클라이언트가 캐시에 있는 비디오 데이터를 읽도록 해야 함
- `socket.recv` 메소드는 bytes 인스턴스 반환
- 슬라이스 연산과 `bytes.join` 메소드를 사용하여 `byte_offset` 에 있는 기존 캐시 데이터를 새로운 데이터로 splice 가능
- 벤치마크 수행
    - 1MB 데이터를 받아 비디오 캐시를 갱신하는데 걸린 시간
        - 33밀리초
    - 수신 시 최대 스루풋
        - 1MB / 33밀리초 = 31MB/s
    - 스트리밍 방송 클라이언트는 31개로 제한

```python
socket = ...        # 클라이언트가 연결한 소켓
video_cache = ...   # 서버로 들어오는 비디오 스트림의 캐시
byte_offset = ...   # 데이터 버퍼 위치
size = 1024 * 1024  # 데이터 덩어리 크기
chunk = socket.recv(size)
video_view = memoryview(video_cache)
before = video_view[:byte_offset]
after = video_view[byte_offset + size:]
new_cache = b''.join([before, chunk, after])
```

```python
def run_test():
    chunk = socket.recv(size)
    before = video_view[:byte_offset]
    after = video_view[byte_offset + size:]
    new_cache = b''.join([before, chunk, after])

result = timeit.timeit(
    stmt='run_test()',
    globals=globals(),
    number=100) / 100

print(f'{result:0.9f} 초')

>>>
0.033520550 초
```

- `bytes` 인스턴스는 read-only 이므로 인덱스를 사용해 값을 변경할 수 없으므로 `bytearrary` 사용
- `bytearray`
    - `bytes` 에서 원하는 위치에 있는 값 변경 가능한 가변 버전

```python
my_bytes = b'hello'
my_bytes[0] = b'\x79'

>>>
Traceback ...
TypeError: 'bytes' object does not support item assignment
```

```python
my_array = bytearray('hello 안녕'.encode("utf8"))  # b''가 아니라# '' 문자열
my_array[0] = 0x79
print(my_array)

>>>
bytearray(b'yello \xec\x95\x88\xeb\x85\x95')
```

- `memoryview` 를 이용한 개선책
    - `socket.recv_info` 나 `RawIOBase.readInto` 와 같은 라이브러리 메소드가 버퍼 프로토콜을 사용해 데이터를 빠르게 받아들이거나 읽을 수 있음

```python
video_array = bytearray(video_cache)
write_view = memoryview(video_array)
chunk = write_view[byte_offset:byte_offset + size]

socket.recv_into(chunk)
```

- 벤치마크 수행 결과

```python
def run_test():
    chunk = write_view[byte_offset:byte_offset + size]
    socket.recv_into(chunk)

result = timeit.timeit(
    stmt='run_test()',
    globals=globals(),
    number=100) / 100

print(f'{result:0.9f} 초')

>>>
0.000033925 초
```