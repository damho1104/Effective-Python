# 6. 인덱스를 사용하는 대신 대입을 사용해 데이터를 언패킹하라

## 1. 언패킹

- 한 문장 안에 여러 값 대입 가능

## 2. 언패킹 장점

- 언패킹은 튜플 인덱스를 사용하는 것 보다 직관적
- 예제 1. 값 switch

```python
# 인덱스 사용 예제
def bubble_sort(a):
    for _ in range(len(a)):
        for i in range(1, len(a)):
            if a[i] < a[i-1]:
                temp = a[i]
                a[i] = a[i-1]
                a[i-1] = temp

names = ['프레즐', '당근', '쑥갓', '베이컨']
bubble_sort(names)
print(names)

>>>
['당근', '베이컨', '쑥갓', '프레즐']
```

```python
# 언패킹 사용 예제
def bubble_sort(a):
    for _ in range(len(a)):
        for i in range(1, len(a)):
            if a[i] < a[i-1]:
                a[i-1], a[i] = a[i], a[i-1] # 맞바꾸기

names = ['프레즐', '당근', '쑥갓', '베이컨']
bubble_sort(names)
print(names)

>>>
['당근', '베이컨', '쑥갓', '프레즐']
```

- 예제 2. 복잡한 자료구조 순회

```python
# snacks 내부 depth 탐색
snacks = [('베이컨', 350), ('도넛', 240), ('머핀', 190)]
for i in range(len(snacks)):
    item = snacks[i]
    name = item[0]
    calories = item[1]
    print(f'#{i+1}: {name} 은 {calories} 칼로리입니다.')

>>>
#1: 베이컨 은 350 칼로리입니다.
#2: 도넛 은 240 칼로리입니다.
#3: 머핀 은 190 칼로리입니다.
```

```python
# enumerate 사용 예제
for rank, (name, calories) in enumerate(snacks, 1):
    print(f'#{rank}: {name} 은 {calories} 칼로리입니다.')

>>>
#1: 베이컨 은 350 칼로리입니다.
#2: 도넛 은 240 칼로리입니다.
#3: 머핀 은 190 칼로리입니다.
```

## 3. 정리

- 언패킹을 효율적으로 사용하면 인덱스 사용을 피할 수 있어 개발자가 보기에 더 직관적이고 명확하게 됨