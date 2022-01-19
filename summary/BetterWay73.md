# 73. 우선순위 큐로 heapq 를 사용하는 방법을 알아두라

## 1. 선입선출 큐의 한계

- 다른 큐의 제약
    - 선입선출(FIFO)
    - 원소를 받은 순서대로 정렬할 수 있음
- 원소를 받은 순서가 아닌 원소 간 상대적인 중요도에 따라 정렬하고 싶은 경우 적합

- 도서관에서 대출한 책을 관리하는 프로그램

```python
class Book:
    def __init__(self, title, due_date):
        self.title = title
        self.due_date = due_date
```

- 만기일을 넘긴 경우 연체 사실을 통지하는 메시지를 보내는 시스템 필요
- FIFO 큐 사용 불가
    - 책의 최대 대출 기간이 얼마나 최근에 발간된 책인지, 얼마나 유명한 책인지 요소에 따라 달라짐
- 표준 리스트를 사용하고 새로운 책이 도착할 때마다 원소 정렬하도록 구현

```python
def add_book(queue, book):
    queue.append(book)
    queue.sort(key=lambda x: x.due_date, reverse=True)
    
queue = []
add_book(queue, Book('돈키호테', '2020-06-07'))
add_book(queue, Book('프랑켄슈타인', '2020-06-05'))
add_book(queue, Book('레미제라블', '2020-06-08'))
add_book(queue, Book('전쟁과 평화', '2020-06-03'))
```

- 대출한 책의 목록은 항상 정렬되어 있다고 가정
    - 연체된 책을 검사하기 위해 할 일은 리스트의 마지막 원소 검사하는 것
    - 리스트에서 연체된 책이 있으면 해당 책을 한 권 찾아 큐에서 제거, 반환 함수 정의
- 이 함수를 반복 호출하여 연체된 책 탐색, 연체 기간이 긴 순으로 통지
- 책이 만기일 이전에 반환되면 리스트에서 반납된 책을 제거하여 연체 통지 예정 목록에서 해당 책 제거
- 모든 책이 반납되었는지 확인 후 `return_book` 함수는 정해진 예외 발생

```python
class NoOverdueBooks(Exception):
    pass
    
def next_overdue_book(queue, now):
    if queue:
        book = queue[-1]
        if book.due_date < now:
            queue.pop()
            return book

    raise NoOverdueBooks
```

```python
now = '2020-06-10'

found = next_overdue_book(queue, now)
print(found.title)

found = next_overdue_book(queue, now)
print(found.title)

>>>
전쟁과 평화
프랑켄슈타인
```

```python
def return_book(queue, book):
    queue.remove(book)

queue = []
book = Book('보물섬', '2020-06-04')

add_book(queue, book)
print('반납 전:', [x.title for x in queue])
return_book(queue, book)
print('반납 후:', [x.title for x in queue])

>>>
반납 전: ['보물섬']
반납 후: []
```

```python
try:
    next_overdue_book(queue, now)
except NoOverdueBooks:
    pass          # 이 문장이 실행될 것으로 예상함
else:
    assert False  # 이 문장은 결코 실행되지 않음
```

- **단점**
    - 계산 복잡도가 비쌈
    - 연체된 책을 검사, 제거하는 비용은 상수 시간
    - 책을 추가할 때마다 전체 리스트를 다시 정렬하는 추가 비용 발생
    - 책을 모두 추가하는 데 드는 비용
        - 책 개수 * 정렬 비용
        - `len(queue) * (len(queue) * math.log(len(queue)))`
    
    ```python
    원소 수: 500 걸린 시간: 0.000844초
    
    원소 수: 1,000 걸린 시간: 0.002584초
    데이터 크기  2.0배, 걸린 시간  3.1배
    
    원소 수: 1,500 걸린 시간: 0.005161초
    데이터 크기  3.0배, 걸린 시간  6.1배
    
    원소 수: 2,000 걸린 시간: 0.008700초
    데이터 크기  4.0배, 걸린 시간  10.3배
    ```
    
    - 책 만기일 이전에 반납된 경우 큐에서 선형 검색을 통해 찾은 후 제거
    - 이때 모든 원소 하나씩 뒤로 이동
        - 해당 비용도 선형보다 커짐
    
    ```python
    원소 수: 500 걸린 시간: 0.000682초
    
    원소 수: 1,000 걸린 시간: 0.002725초
    데이터 크기  2.0배, 걸린 시간  4.0배
    
    원소 수: 1,500 걸린 시간: 0.006290초
    데이터 크기  3.0배, 걸린 시간  9.2배
    
    원소 수: 2,000 걸린 시간: 0.011128초
    데이터 크기  4.0배, 걸린 시간  16.3배
    ```
    

## 2. 우선순위 큐(Priority Queue)

- 내장 `heapq` 모듈
    - 여러 아이템을 유지하되 새로운 원소를 추가하거나 가장 작은 원소를 제거할 때 로그 복잡도가 드는 구조
    - 도서관 예제에서 가장 작은 원소라는 말은 만기가 가장 빠른 원소

```python
from heapq import heappush

def add_book(queue, book):
    heappush(queue, book)
```

- 기존 정의한 `Book` 클래스를 그대로 사용하는 경우 예외 발생
    - 우선순위 큐에 들어갈 원소들이 서로 비교 가능해야 하고 원소 사이에 자연스러운 정렬 순서가 존재해야 함
        - `Book` 클래스에 비교 기능과 자연스러운 정렬 순서 제공
            - [Better way 14 참고](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay14.md)
            - `functools` 내장 모듈이 제공하는 `total_ordering` 클래스 데코레이터 사용([Better way 51](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay61.md))
            - `__It__` 특별 메소드([Better way 43](https://github.com/damho1104/Effective-Python/blob/master/summary/BetterWay43.md))를 구현

```python
queue = []
add_book(queue, Book('작은 아씨들', '2020-06-05'))
add_book(queue, Book('타임 머신', '2020-05-30'))

>>>
Traceback ...   
TypeError: '<' not supported between instances of 'Book' and 'Book'
```

```python
import functools

@functools.total_ordering
class Book:
    def __init__(self, title, due_date):
        self.title = title
        self.due_date = due_date
        
    def __lt__(self, other):
        return self.due_date < other.due_date
```

- `heapq.heappush` 함수를 사용해도 우선순위 큐에 문제 없이 등록 가능

```python
queue = []
add_book(queue, Book('오만과 편견', '2020-06-01'))
add_book(queue, Book('타임 머신', '2020-05-30'))
add_book(queue, Book('죄와 벌', '2020-06-06'))
add_book(queue, Book('폭풍의 언덕', '2020-06-12'))
```

- `heapq.heapify` 함수를 사용하면 선형 시간에 힙을 만들 수 있음
    - `sort` 를 사용하는 것에 비해 훨씬 빠름

```python
from heapq import heapify

queue = [
    Book('오만과 편견', '2020-06-01'),
    Book('타임 머신', '2020-05-30'),
    Book('죄와 벌', '2020-06-06'),
    Book('폭풍의 언덕', '2020-06-12'),
]
heapify(queue)
```

- 대출 만기를 넘긴 책을 검사하려면 리스트의 마지막 원소가 아니라 첫 번째 원소를 확인, 제거
- 현재 시간보다 만기가 이른 책을 모두 찾아 제거

```python
from heapq import heappop

def next_overdue_book(queue, now):
    if queue:
        book = queue[0]          # 만기가 가장 이른 책이 맨 앞에 있다
        if book.due_date < now:
            heappop(queue)       # 연체된 책을 제거한다
            return book            
    raise NoOverdueBooks
```

```python
now = '2020-06-02'

book = next_overdue_book(queue, now)
print(book.title)

book = next_overdue_book(queue, now)
print(book.title)

try:
    next_overdue_book(queue, now)
except NoOverdueBooks:
    pass              # 이 문장이 실행될 것으로 예상함
else:
    assert False      # 이 문장은 결코 실행되지 않음

>>>
타임 머신
오만과 편견
```

- 벤치마크 수행 결과

```python
원소 수: 500 걸린 시간: 0.000116초

원소 수: 1,000 걸린 시간: 0.000244초
데이터 크기  2.0배, 걸린 시간  2.1배

원소 수: 1,500 걸린 시간: 0.000374초
데이터 크기  3.0배, 걸린 시간  3.2배

원소 수: 2,000 걸린 시간: 0.000510초
데이터 크기  4.0배, 걸린 시간  4.4배
```

## 3. 정리

- `heapq` 를 사용하면 힙 연산을 빨라지나 저장소 부가 비용으로 인해 메모리 사용량 증가
- 해당 단점에도 불구하고 튼튼한 시스템을 구축하려 한다면 최악의 경우를 가정하고 계획해야 함
- `heapq` 모듈은 우선순위 큐 외에도 대응 기능 추가 제공, 많이 찾아보고 사용해볼 것