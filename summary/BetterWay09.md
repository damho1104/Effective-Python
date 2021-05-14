# 9. for 나 while 루프 뒤에 else 블록을 사용하지 말라

## 1. 루프 다음 else 블록

```python
for i in range(3):
    print('Loop', i)
else:
    print('Else block!')

>>>
Loop 0
Loop 1
Loop 2
Else block!
```

- else 블록은 루프가 끝나고 수행
- **루프를 모두 순환하고 벗어난 경우에만 else 블록 실행**
- 만일 루프 내부에서 break 으로 루프 밖으로 나온다면??
    - else 블록 수행 안함
- 만일 빈 리스트나 빈 시퀀스로 루프를 수행한다면??
    - else 블록 수행

```python
for i in range(3):
    print('Loop', i)
    if i == 1:
        break
else:
    print('Else block!')

>>>
Loop 0
Loop 1
```

```python
for x in []:
    print('이 줄은 실행되지 않음')
else:
    print('For Else block!')

>>>
For Else block!
```

## 2. while else 블록 사용 권장하지 않는 이유?

- 저자가 말하고자 하는 핵심
    - 여러 사람들과 협업하는 환경에서 서로가 이해할 수 있는 혹은 명확한 방법이 아니라 판단한 것
    - else 블록을 사용함으로써 루프 동작 자체가 직관적이지 않고 혼란스럽다고 판단

- 나는?
    - **동의**
    - **이유**: 어떤 상황에서 써야 할 지 잘 모르겠음
    - 책에서는 서로소에 대한 예제로 while else 블록 필요 이유를 설명하지만 굳이 저렇게 써야 하나 싶긴 하다.