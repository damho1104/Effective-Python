# 14. 복잡한 기준을 사용해 정렬할 때는 key 파라미터를 사용하라

## 1. `list` 내장 함수 `sort`

- 기본적으로 원소 타입의 순서를 사용해 오름차순으로 정렬

```python
numbers = [93, 86, 11, 68, 70]
numbers.sort()
print(numbers)

>>>
[11, 68, 70, 86, 93]
```

- 객체 순서 표현은 특별 메소드 정의(Better Way 73)하면 가능
- 객체의 특정 어트리뷰트를 이용하여 정렬 가능
    - `sort` 함수 `key` 파라미터
    - `key` 파라미터에 람다 함수 사용하여 특정 정렬 조건 표현 가능

```python
class Tool:
    def __init__(self, name, weight):
        self.name = name
        self.weight = weight

    def __repr__(self):
        return f'Tool({self.name!r}, {self.weight})'
        
tools = [
    Tool('수준계', 3.5),
    Tool('해머', 1.25),
    Tool('스크류드라이버', 0.5),
    Tool('끌', 0.25),
]

print('미정렬:', repr(tools))
tools.sort(key=lambda x: x.name)
print('\n정렬:', tools)

>>>
미정렬: [Tool('수준계', 3.5), Tool('해머', 1.25), Tool('스크류드라이버', 0.5), Tool('끌', 0.25)] 
정렬: [Tool('끌', 0.25), Tool('수준계', 3.5), Tool('스크류드라이버', 0.5), Tool('해머', 1.25)]
```

## 2. 내장함수 sort 활용

- 복합적인 기준을 사용해여 여러번 정렬하고 싶은 경우
    - `tuple` 사용
    - `tuple` 은 첫번째 위치에 있는 값 비교 후 같으면 두번째 위치 값 비교를 진행
    - 단점
        - 정렬 방향이 모두 같아야 함, 모두 오름차순 혹은 내림차순
    - 숫자의 경우
        - `-` 를 사용해 정렬 방향 변경 가능
    - 다른 타입의 경우
        - `sort` 의 `key` 를 다르게 하여 여러번 호출
        - `sort` 의 `key` 함수 결과가 같으면 원래 순서 유지
        - 정렬 기준의 우선순위가 점점 높아지는 순서로 `sort` 호출

```python
power_tools = [
    Tool('드릴', 4),
    Tool('원형 톱', 5),
    Tool('드릴', 4),
    Tool('원형 톱', 5),
    Tool('착암기', 40),
    Tool('연마기', 4),
]

power_tools.sort(key=lambda x: (x.weight, x.name))
print(power_tools)

power_tools.sort(key=lambda x: (-x.weight, x.name))
print(power_tools)

power_tools.sort(key=lambda x: x.name)    # name 기준 오름차순
power_tools.sort(key=lambda x: x.weight,  # weight 기준 내림차순
                 reverse=True)
print(power_tools)

>>>
[Tool('드릴', 4), Tool('연마기', 4), Tool('원형 톱', 5), Tool('착암기', 40)]
[Tool('착암기', 40), Tool('원형 톱', 5), Tool('드릴', 4), Tool('연마기', 4)]
[Tool('착암기', 40), Tool('원형 톱', 5), Tool('드릴', 4), Tool('연마기', 4)]
```