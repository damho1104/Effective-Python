# 45. 애트리뷰트를 리팩터링하는 대신 @property 를 사용하라

## 1. `@property` 데코레이터

- 기존 클래스를 호출하는 코드를 전혀 바꾸지 않고도 클래스 애트리뷰트의 기존 동작 변경 가능

```python
from datetime import datetime, timedelta

class Bucket:
    def __init__(self, period):
        self.period_delta = timedelta(seconds=period)
        self.reset_time = datetime.now()
        self.quota = 0
        
    def __repr__(self):
        return f'Bucket(quota={self.quota})'

def fill(bucket, amount):
    now = datetime.now()
    if (now - bucket.reset_time) > bucket.period_delta:
        bucket.quota = 0
        bucket.reset_time = now
    bucket.quota += amount

def deduct(bucket, amount):
    now = datetime.now()
    if (now - bucket.reset_time) > bucket.period_delta:
        return False # 새 주기가 시작됐는데 아직 버킷 할당량이 재설정되지 않았다
    if bucket.quota - amount < 0:
        return False # 버킷의 가용 용량이 충분하지 못하다
    else:
        bucket.quota -= amount
        return True  # 버킷의 가용 용량이 충분하므로 필요한 분량을 사용한다

bucket = Bucket(60)
fill(bucket, 100)
print(bucket)

if deduct(bucket, 99):
    print('99 용량 사용')
else:
    print('가용 용량이 작아서 99 용량을 처리할 수 없음')
print(bucket)

if deduct(bucket, 3):
    print('3 용량 사용')
else:
    print('가용 용량이 작아서 3 용량을 처리할 수 없음')
print(bucket)

>>>
Bucket(quota=100)
99 용량 사용
Bucket(quota=1)
‘가용 용량이 작아서 3 용량을 처리할 수 없음
Bucket(quota=1)
```

- 리키 버킷 알고리즘은 시간을 일정한 간격으로 구분하고(주기) 가용 용량을 소비할 때마다 시간을 검사
- 주기가 달라질 경우 이전 주기에 미사용한 가용 용량이 새로운 주기로 넘어오지 못하게 막음
- 가용 용량을 소비하는 쪽
    - 어떤 작업을 하고 싶을 때마다 버킷으로부터 자신의 작업에 필요한 용량을 할당 받아야 함
- 어느 순간이 되면 버킷의 가용 용량이 데이터 처리에 필요한 용량보다 작아지면서 작업 진행 못함

- **구현의 문제점**
    - 버킷이 시작할 때 가용 용량이 얼마인지 알 수 없음
    - 한 주기 안에서는 가용 용량이 0이 될 때까지 감소할 것
        - 그러나 새로운 가용 용량을 할당받기 전까지 deduct 는 항상 `False` 를 반환할 것
        - `deduct` 호출하는 쪽에서 할당받지 못한 원인을 알 수 있는 방법 확인 필요
            - 가용 용량이 소진되었기 때문?
            - 이번 주기에 아직 버킷에 매 주기마다 재설정하도록 미리 정해진 가용 용량을 추가받지 못했기 때문?

```python
class NewBucket:
    def __init__(self, period):
        self.period_delta = timedelta(seconds=period)
        self.reset_time = datetime.now()
        self.max_quota = 0
        self.quota_consumed = 0
        
    def __repr__(self):
        return (f'NewBucket(max_quota={self.max_quota}, '
                f'quota_consumed={self.quota_consumed})')

    @property
    def quota(self):
        return self.max_quota - self.quota_consumed

    @quota.setter
    def quota(self, amount):
        delta = self.max_quota - amount
        if amount == 0:
            # 새로운 주기가 되고 가용 용량을 재설정하는 경우
            self.quota_consumed = 0
            self.max_quota = 0
        elif delta < 0:
            # 새로운 주기가 되고 가용 용량을 추가하는 경우 
            assert self.quota_consumed == 0
            self.max_quota = amount
        else:
            # 어떤 주기 안에서 가용 용량을 소비하는 경우
            assert self.max_quota >= self.quota_consumed
            self.quota_consumed += delta
```

- `max_quota`, `quota_consumed` 추가
    - `max_quota` : 가용 용량
    - `quota_comsumed` : 이번 주기에 버킷에서 소비한 용량의 합계
- `Bucket` 클래스와 인터페이스를 동일하게 제공하기 위해 `@property` 데코레이터가 붙은 메소드를 사용
    - 현재 가용 용량 수준을 그때그때 계산하도록 설정
- `fill`, `deduct` 에서 `quota` 를 사용할 때 `NewBucket` 클래스에 맞게 `setter` 수정

```python
bucket = NewBucket(60)
print('최초', bucket)
fill(bucket, 100)
print('보충 후', bucket)

if deduct(bucket, 99):
    print('99 용량 사용')
else:
    print('가용 용량이 작아서 99 용량을 처리할 수 없음')
print('사용 후', bucket)

if deduct(bucket, 3):
    print('3 용량 사용')
else:
    print('가용 용량이 작아서 3 용량을 처리할 수 없음')

print('여전히', bucket)

>>>
최초 NewBucket(max_quota=0, quota_consumed=0)
보충 후 NewBucket(max_quota=100, quota_consumed=0)
```

## 2. `@property` 데코레이터 사용 특징

- 장점
    - 해당 예제에서의 장점
        - 기존 `Bucket.quota` 를 사용하는 코드를 변경할 필요 없음
        - 클래스 구현이 변경되었음을 알 필요 또한 없음
        - 사용 로직은 그대로 두고 클래스 타입만 변경하여 새로운 기능(`max_quota`, `quota_comsumed`) 또한 추가
    - 데이터 모델 점진적으로 개선 가능
- 주의점
    - @property 데코레이터를 사용한 메소드를 너무 과하게 사용하고 있다면 의도치 않은 버그 발생 가능성이 증가할 수 있음
    - 이때는 리팩토링 고려해야 함