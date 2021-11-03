# 52. 자식 프로세스를 관리하기 위해 subprocess 를 사용하라

## 1. 파이선에서 하위 프로세스를 실행하는 방법

- 하위 프로세스를 실행한느 방법
    - `os.popen`
    - `os.exec`
    - `subprocess`
- `subprocess` 모듈 사용 권장
    - 관리 편의성 증대

```python
import subprocess

result = subprocess.run(['echo', '자식 프로세스가 보내는 인사!'],
                        capture_output=True,
                        encoding='utf-8')

result.check_returncode()  # 예외가 발생하지 않으면 문제없이 잘 종료한 것이다
print(result.stdout)

>>>
자식 프로세스가 보내는 인사!
```

- `run` 함수로 프로세스 시작, 프로세스의 출력을 읽고 프로세스의 종료 여부 확인 가능
- `subprocess` 모듈을 통해 실행한 자식 프로세스는 부모 프로세스 파이선 인터프리터와 독립적으로 실행
- `run` 대신 `Popen` 클래스를 사용한 자식 프로세스 실행 및 상태 검사 예제

```python
proc = subprocess.Popen(['sleep', '1'])
while proc.poll() is None:
    print('작업 중...')
    # 시간이 걸리는 작업을 여기서 수행한다
    ...

print('종료 상태', proc.poll())
>>>
작업 중...
작업 중...
작업 중...
작업 중...
종료 상태 0

```

```python
import time

start = time.time()
sleep_procs = []
for _ in range(10):
    proc = subprocess.Popen(['sleep', '1'])
    sleep_procs.append(proc)

for proc in sleep_procs:
    proc.communicate()
end = time.time()
delta = end - start
print(f'{delta:.3} 초 만에 끝남')

>>>
1.04 초 만에 끝남
```

- PIPE 를 사용하여 하위 프로세스로 데이터를 보내거나 출력을 받을 수 있음

```python
import os
def run_encrypt(data):
    env = os.environ.copy()
    env['password'] = 'zf7ShyBhZOraQDdE/FiZpm/m/8f9X+M1'
    proc = subprocess.Popen(
        ['openssl', 'enc', '-des3', '-pass', 'env:password'],
        env=env,
        stdin=subprocess.PIPE,
        stdout=subprocess.PIPE)
    proc.stdin.write(data)
    proc.stdin.flush()  # 자식이 입력을 받도록 보장한다
    return proc

rocs = []
for _ in range(3):
    data = os.urandom(10)
    proc = run_encrypt(data)
    procs.append(proc)
for proc in procs:
    out, _ = proc.communicate()
    print(out[-10:])

>>>
b'\x8c(\xed\xc7m1\xf0F4\xe6'
b'\x0eD\x97\xe9>\x10h{\xbd\xf0'
b'g\x93)\x14U\xa9\xdc\xdd\x04\xd2'
```

- 자식 프로세스 관리를 위해 procs 리스트에 저장하였다가 마지막 출력을 가져옴
- 출력은 암호화된 바이트 문자열

- 유닉스 파이프라인처럼 한 자식 프로세스의 출력을 다음 프로세스의 입력으로 계속 연결시켜 병렬 프로세스를 연쇄적으로 연결 가능

```python
def run_hash(input_stdin):
    return subprocess.Popen(
        ['openssl', 'dgst', '-whirlpool', '-binary'],
        stdin=input_stdin,
        stdout=subprocess.PIPE)

encrypt_procs = []
hash_procs = []
for _ in range(3):
    data = os.urandom(100)
    
    encrypt_proc = run_encrypt(data)
    encrypt_procs.append(encrypt_proc)
    
    hash_proc = run_hash(encrypt_proc.stdout)
    hash_procs.append(hash_proc)
    
    # 자식이 입력 스트림에 들어오는 데이터를 소비하고# communicate() 메서드가 
    # 불필요하게 자식으로부터 오는 입력을 훔쳐가지 못하게 만든다
    # 또 다운스트림 프로세스가 죽으면# SIGPIPE를 업스트림 프로세스에 전달한다
    encrypt_proc.stdout.close()
    encrypt_proc.stdout = None

for proc in encrypt_procs:
    proc.communicate()
    assert proc.returncode == 0
for proc in hash_procs:
    out, _ = proc.communicate()
    print(out[-10:])
    
assert proc.returncode == 0

>>>
b'\xe2j\x98h\xfd\xec\xe7T\xd84'
b'\xf3.i\x01\xd74|\xf2\x94E'
b'5_n\xc3-\xe6j\xeb[i'
```

- 자식 프로세스가 끝나지 않는 경우, 블록될 수 있음
- timeout 파라미터 사용하여 끝날 수 있도록 하는 예제

```python
proc = subprocess.Popen(['sleep', '10'])
try:
    proc.communicate(timeout=0.1)
except subprocess.TimeoutExpired:
    proc.terminate()
    proc.wait() 

print('종료 상태', proc.poll())

>>>
종료 상태 -15
```