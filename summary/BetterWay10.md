# 10. 대입식을 사용해 반복을 피하라

## 1. 왈러스 연산자(walrus operator)

- python 3.8부터 도입
- 왈러스 연산자는 대입문이 쓰일 수 없는 위치에서 변수 값을 대입할 수 있음
    - ex. if 문에서 사용 가능

```python
def make_lemonade(count):
    ...

def out_of_stock():
    ...

count = fresh_fruit.get('레몬', 0)
if count:
    make_lemonade(count)
else:
    out_of_stock()
```

- count 변수는 if 문 안에서만 사용됨
- 이럴 때 왈러스 연산자를 쓰면 유용

```python
if count := fresh_fruit.get('레몬', 0):
    make_lemonade(count)
else:
    out_of_stock()
```

- 이렇게 사용하면 count 변수는 if문에서만 의미 있다는 것이 명확히 보이므로 코드 가독성이 증가함

## 2. 왈러스 연산자 사용 권장 케이스

- python 에는 switch/case 와 do/while 이 존재하지 않음
- 대체제로 사용하기 좋음

```python
count = fresh_fruit.get('바나나', 0)
if count >= 2:
    pieces = slice_bananas(count)
    to_enjoy = make_smoothies(pieces)
else:
    count = fresh_fruit.get('사과', 0)
    if count >= 4:
        to_enjoy = make_cider(count)
    else:
        count = fresh_fruit.get('레몬', 0)
        if count:
            to_enjoy = make_lemonade(count)
        else:
            to_enjoy'= '아무것도 없음
```

- switch/case 문의 if/else 화

```python
if (count := fresh_fruit.get('바나나', 0)) >= 2:
    pieces = slice_bananas(count)
    to_enjoy = make_smoothies(pieces)
elif (count := fresh_fruit.get('사과', 0)) >= 4:
    to_enjoy = make_cider(count)
elif count := fresh_fruit.get('레몬', 0):
    to_enjoy = make_lemonade(count)
else:
    to_enjoy = '아무것도 없음
```

- 왈러스 연산자를 사용한 방법

```python
bottles = []
while True:             # 무한 루프
    fresh_fruit = pick_fruit()
    if not fresh_fruit: # 중간에서 끝내기
        break

    for fruit, count in fresh_fruit.items():
        batch = make_juice(fruit, count)
        bottles.extend(batch)
```

- do/while 문의 while 문 화

```python
bottles = []
while fresh_fruit := pick_fruit():
    for fruit, count in fresh_fruit.items():
        batch = make_juice(fruit, count)
        bottles.extend(batch)
```

- 왈러스 연산자를 사용한 방법